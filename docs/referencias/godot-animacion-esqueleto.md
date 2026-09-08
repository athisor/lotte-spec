# Auditoría: sistema de animación y esqueleto 2D de Godot

- **Repo fuente**: `references/godot/` (solo lectura, sparse checkout: `scene/gui`, `scene/animation`, `scene/2d`, `scene/resources`)
- **Commit**: `6a0f6f32cfb2ce4cc5bad6641d0afda413b62a9d` (medido con `git log -1 --format="%H %ci"` desde el checkout de referencia: `6a0f6f32cfb2ce4cc5bad6641d0afda413b62a9d 2026-09-07 16:16:21 -0500`)
- **Licencia**: MIT (Godot Engine, copyright Juan Linietsky, Ariel Manzur y colaboradores; ver cabecera de cada archivo citado)
- **Fecha del informe**: 2026-09-08
- **Estrategia de ingesta**: traducción C++ → Rust con atribución MIT. Ningún fragmento se copia tal
  cual a `lotte-*`; cada patrón portado se reimplementa en Rust y el commit que lo introduce cita
  este documento y el archivo:línea de origen en su cuerpo.

Convención de esta auditoría: cada afirmación de estructura o comportamiento cita `archivo:línea`
medido con `grep -n` o `Read` sobre el checkout de referencia el 2026-09-08. Donde el brief original
asumía una ubicación y el archivo estaba en otro lado, se anota explícitamente (caso de
`SkeletonModification2D*`, ver sección 6).

---

## 1. `Animation` — tipos de pista, direccionamiento, claves, interpolación y loop

### 1.1 Tipos de pista

`enum TrackType : uint8_t` en `scene/resources/animation.h:48-58`:

```cpp
enum TrackType : uint8_t {
    TYPE_VALUE, // Set a value in a property, can be interpolated.
    TYPE_POSITION_3D, // Position 3D track, can be compressed.
    TYPE_ROTATION_3D, // Rotation 3D track, can be compressed.
    TYPE_SCALE_3D, // Scale 3D track, can be compressed.
    TYPE_BLEND_SHAPE, // Blend Shape track, can be compressed.
    TYPE_METHOD, // Call any method on a specific node.
    TYPE_BEZIER, // Bezier curve.
    TYPE_AUDIO,
    TYPE_ANIMATION,
};
```

No hay tipos separados de posición/rotación/escala **2D**: los tracks 3D (`TYPE_POSITION_3D`,
`TYPE_ROTATION_3D`, `TYPE_SCALE_3D`) son los únicos tracks de transformación, y en 2D (`Node2D`,
`Bone2D`) se animan casi siempre vía `TYPE_VALUE` sobre las propiedades `position`, `rotation`,
`scale` del nodo — no existe un `TYPE_TRANSFORM_2D` dedicado (confirmado por ausencia del literal
`2D` en el enum de arriba).

### 1.2 Direccionamiento del objetivo

Cada `Track` (`scene/resources/animation.h:107-123`) guarda:

```cpp
struct Track {
    TrackType type = TrackType::TYPE_ANIMATION;
    InterpolationType interpolation = INTERPOLATION_LINEAR;
    bool loop_wrap = true;
    NodePath path; // Path to something.
    StringName concatenated_path;
    bool imported = false;
    bool enabled = true;
    ...
};
```

`path` es un `NodePath` de Godot, que admite la forma `"Path/To/Node:property"` (ruta de nodo +
subnombre de propiedad separado por `:`). El mixer resuelve esa ruta en dos partes — nodo/recurso y
subcamino de propiedad — vía `get_node_and_resource(path, resource, leftover_path)`
(`scene/animation/animation_mixer.cpp:733`); `leftover_path` (el subnombre tras `:`) se guarda como
`TrackCacheValue::subpath` (`animation_mixer.h:226`) y se usa después para escribir con
`Object::set_indexed(subpath, value)` (`scene/animation/animation_mixer.cpp:2020`, ver §3). `path`
en sí no separa nodo de propiedad — es `track_set_path`/`track_get_path`
(`scene/resources/animation.cpp:1034-1044`) el que guarda el `NodePath` completo, con subnombres
incluidos; `concatenated_path` (`String(path)`, línea 1037) es solo una caché de comparación.

### 1.3 Estructuras de clave por tipo de pista

Base común, `scene/resources/animation.h:126-135`:

```cpp
struct Key {
    real_t transition = 1.0;
    double time = 0.0; // Time in secs.
};

template <typename T>
struct TKey : public Key { T value; };
```

Por tipo (todas en `scene/resources/animation.h`):

| Track | Clave | Línea |
|---|---|---|
| `PositionTrack` | `TKey<Vector3>` | 144-148 |
| `RotationTrack` | `TKey<Quaternion>` | 152-156 |
| `ScaleTrack` | `TKey<Vector3>` | 160-164 |
| `BlendShapeTrack` | `TKey<float>` | 168-172 |
| `ValueTrack` | `TKey<Variant>` + `UpdateMode update_mode` | 176-183 |
| `MethodTrack` | `MethodKey : Key { StringName method; Vector<Variant> params; }` | 187-195 |
| `BezierTrack` | `TKey<BezierKey>` | 208-214 |
| `AudioTrack` | `TKey<AudioKey>` (`stream`, `start_offset`, `end_offset`) | 218-233 |
| `AnimationTrack` | `TKey<StringName>` (nombre de sub-animación a disparar) | 237-243 |

`BezierKey` (`animation.h:199-206`):

```cpp
struct BezierKey {
    Vector2 in_handle; // Relative (x always <0)
    Vector2 out_handle; // Relative (x always >0)
    real_t value = 0.0;
#ifdef TOOLS_ENABLED
    HandleMode handle_mode = HANDLE_MODE_FREE;
#endif
};
```

Cada `Key` trae `transition` (exponente de easing, no una tangente) además de `time`; `transition`
se usa en `_interpolate` (§1.4) para curvar el factor `c` con `Math::ease(c, tr)`
(`scene/resources/animation.cpp:2664-2666`) antes de aplicar la interpolación lineal/cúbica — es
una perilla de "easing" por-clave que Lotte no tiene hoy en `FCurve`.

### 1.4 Modos de interpolación y de actualización

`InterpolationType` (`animation.h:60-66`): `INTERPOLATION_NEAREST`, `INTERPOLATION_LINEAR`,
`INTERPOLATION_CUBIC`, `INTERPOLATION_LINEAR_ANGLE`, `INTERPOLATION_CUBIC_ANGLE`.

`UpdateMode` (`animation.h:68-72`): `UPDATE_CONTINUOUS`, `UPDATE_DISCRETE`, `UPDATE_CAPTURE`. Solo
aplica a `ValueTrack` (campo `update_mode` en `ValueTrack`, línea 177); ver uso en §3.

`LoopMode` (`animation.h:74-78`): `LOOP_NONE`, `LOOP_LINEAR`, `LOOP_PINGPONG`. Es una propiedad de
la `Animation` completa (no por track): `set_loop_mode` (`animation.cpp:3859-3861`).

### 1.5 Evaluación en `t`

`value_track_interpolate` (`animation.cpp:2723-2738`) delega en el template interno `_interpolate`
(`animation.cpp:2519-2721`, el núcleo real de evaluación de todos los tracks TKey-based). Con
`vt->update_mode == UPDATE_DISCRETE` fuerza `INTERPOLATION_NEAREST` en vez del tipo declarado en la
pista (línea 2731) — es decir, "discreto" no es un tipo de interpolación aparte sino un *override*
de interpolación a "vecino más cercano" en tiempo de evaluación.

`_interpolate` (líneas 2519-2721): encuentra el índice `idx` con `_find` (búsqueda binaria por
tiempo), calcula vecino `next` (y `pre`/`post` si la interpolación es cúbica), resuelve los tres
regímenes de `LoopMode` con lógica distinta para wrap de `delta`/`from` en cada uno
(`LOOP_NONE`: clamp a los bordes, líneas 2558-2566; `LOOP_LINEAR`: wrap con `Math::posmod`, líneas
2567-2600; `LOOP_PINGPONG`: wrap con `Math::pingpong`, líneas 2601-2636), aplica `transition` como
easing (líneas 2658-2666) y finalmente hace `switch(p_interp)` sobre
`NEAREST/LINEAR/LINEAR_ANGLE/CUBIC/CUBIC_ANGLE` (líneas 2668-2718).

Wrap de ángulos: `_interpolate_angle` (`animation.cpp:2468-2479`) solo aplica el wrap si el valor es
`INT`/`FLOAT` (chequeo de `Variant::Type` por máscara de bits, líneas 2471-2473):

```cpp
if (vformat == ((1 << Variant::INT) | (1 << Variant::FLOAT)) || vformat == (1 << Variant::FLOAT)) {
    real_t a = p_a;
    real_t b = p_b;
    return Math::fposmod((float)Math::lerp_angle(a, b, p_c), (float)Math::TAU);
}
return _interpolate(p_a, p_b, p_c);
```

Es decir: interpola por el camino angular más corto (`lerp_angle`) y renormaliza a `[0, TAU)` con
`fposmod`. Si el valor no es numérico cae al lerp genérico (`_interpolate`, sin wrap).

`bezier_track_interpolate` es un algoritmo separado, no pasa por `_interpolate`
(`animation.cpp:3607`, comentario propio: "this uses a different interpolation scheme") — ver §2.

---

## 2. Bezier tracks

### 2.1 Definición de manijas

`in_handle`/`out_handle` son **vectores relativos** en espacio (tiempo, valor) respecto a la propia
clave (`animation.h:200-201`), no puntos absolutos. `bezier_track_insert_key`
(`animation.cpp:3341-3365`) fuerza el signo: `in_handle.x` se clampa a `≤ 0` (línea 3352-3354) y
`out_handle.x` a `≥ 0` (línea 3356-3358) — la manija de entrada siempre apunta hacia atrás en el
tiempo y la de salida hacia adelante, evitando curvas Bezier con `x` no monótono dentro de un
segmento.

### 2.2 Resolución X→t: bisección con 10 iteraciones fijas

`bezier_track_interpolate` (`animation.cpp:3607-3670`). Tras ubicar el segmento `[idx, idx+1]` y
construir los cuatro puntos de control en espacio (tiempo relativo, valor):

```cpp
Vector2 start(0, bt->values[idx].value.value);
Vector2 start_out = start + bt->values[idx].value.out_handle;
Vector2 end(duration, bt->values[idx + 1].value.value);
Vector2 end_in = end + bt->values[idx + 1].value.in_handle;

for (int i = 0; i < iterations; i++) {          // iterations = 10, línea 3640
    real_t middle = (low + high) / 2;
    Vector2 interp = start.bezier_interpolate(start_out, end_in, end, middle);
    if (interp.x < t) { low = middle; } else { high = middle; }
}

Vector2 low_pos = start.bezier_interpolate(start_out, end_in, end, low);
Vector2 high_pos = start.bezier_interpolate(start_out, end_in, end, high);
real_t c = (t - low_pos.x) / (high_pos.x - low_pos.x);
return low_pos.lerp(high_pos, c).y;
```

Es **bisección pura sobre el parámetro `middle` de la curva** (no Newton, no subdivisión de De
Casteljau adaptativa): 10 iteraciones fijas (línea 3640) que acotan `[low, high]` comparando la
coordenada `x` de la curva contra el tiempo objetivo `t`, y un paso final de interpolación lineal
entre las dos `y` obtenidas para afinar el resultado dentro del intervalo final. El comentario
explícito en el código (línea 3608, "this uses a different interpolation scheme") confirma que es
un camino de evaluación separado de `_interpolate` (§1.5): no hay loop en bezier tracks
("there really is no looping interpolation on bezier", línea 3628) — fuera de rango devuelve el
valor de la primera o última clave (líneas 3630-3636).

### 2.3 Modo de manija (`HandleMode`, solo en editor)

`animation.h:94-104`, bajo `#ifdef TOOLS_ENABLED` (no existe en build de runtime):

```cpp
enum HandleMode { HANDLE_MODE_FREE, HANDLE_MODE_LINEAR, HANDLE_MODE_BALANCED, HANDLE_MODE_MIRRORED };
enum HandleSetMode { HANDLE_SET_MODE_NONE, HANDLE_SET_MODE_RESET, HANDLE_SET_MODE_AUTO };
```

`bezier_track_calculate_handles` (`animation.cpp:3543-3603`) computa in/out según modo:
- `LINEAR`: ambas manijas en `(0,0)` (línea 3550-3552, la curva se vuelve un segmento recto local).
- `BALANCED`: longitud fija `1/3` del intervalo de tiempo hacia cada vecino en `RESET`
  (línea 3554-3559), o tangente `(next_value - prev_value) / (next_time - prev_time)` con longitud
  `1/6` en `AUTO` (líneas 3560-3567) — la manija se ajusta a la pendiente entre vecinos, no a la
  posición real de las manijas del usuario en el otro lado (asimetría posible en longitud, no en
  dirección).
- `MIRRORED`: longitud simétrica (`min(prev_interval, next_interval) / 4`, línea 3569-3579) con
  igual magnitud a ambos lados; en `AUTO` usa la misma fórmula de tangente que `BALANCED` pero
  normalizada por `min_time` (línea 3585-3591).

Además, al editar una manija a mano (`bezier_track_set_key_in_handle`/`_out_handle`,
`animation.cpp:3381-3449`), si el modo es `LINEAR` ambas manijas se colapsan a `(0,0)`
(líneas 3397-3399, 3432-3434); si es `BALANCED` se reescala la manija opuesta manteniendo la razón
valor/tiempo (`Transform2D` de escala, líneas 3400-3407); si es `MIRRORED` la manija opuesta se
fija a `-p_handle` (líneas 3408-3409, 3443-3444).

---

## 3. `AnimationMixer` / `AnimationPlayer`

### 3.1 Modelo de blending

`PlaybackInfo` (`scene/animation/animation_mixer.h:84-94`):

```cpp
struct PlaybackInfo {
    double time = 0.0;
    double delta = 0.0;
    double start = 0.0;
    double end = 0.0;
    bool seeked = false;
    bool is_external_seeking = false;
    Animation::LoopedFlag looped_flag = Animation::LOOPED_FLAG_NONE;
    real_t weight = 0.0;
    LocalVector<real_t> *track_weights = nullptr;
};
```

Cada `Animation` activa se empuja como `AnimationInstance{animation, playback_info}` en
`animation_instances` (`animation_mixer.h:324`, alimentado por
`make_animation_instance`, `animation_mixer.cpp:2108-2116`). `_blend_process`
(`animation_mixer.cpp:1234-1289`) recorre todas las instancias activas y, por track, calcula:

```cpp
if (!track_weights.is_empty() && blend_idx < static_cast<int>(track_weights.size())) {
    blend = track_weights[blend_idx] * weight;   // peso por-track * peso de la animación
} else {
    blend = weight;
}
if (!deterministic) {
    if (Math::is_zero_approx(track->total_weight)) { continue; }
    blend = blend / track->total_weight;          // normalización (modo no-determinista)
}
```

(`animation_mixer.cpp:1273-1288`). Es decir: el peso efectivo de cada `Animation` sobre un track es
`peso_de_la_animación × peso_por_track`, y en el modo por defecto (no determinista) se normaliza
dividiendo por la suma total de pesos que tocan ese track (`total_weight`, acumulado antes en
`_blend_calc_total_weight`, `animation_mixer.cpp:1173-1173+`). No hay jerarquía explícita de
"capas" con prioridad — el orden de mezcla lo da el orden de inserción en `animation_instances`
más los pesos, todo resuelto en un único paso de acumulación por track.

`AnimationPlayer` (una subclase de `AnimationMixer`, `scene/animation/animation_player.h:36`) NO
reimplementa el blending: modela un *crossfade* entre "la animación actual" y una cola de
animaciones salientes con peso decreciente. `Blend{ PlaybackData data; double blend_time;
double blend_left; }` (`animation_player.h:87-91`) vive en `playback.blend`
(`LocalVector<Blend>`, línea 99); cada tick resta `blend_left` proporcional a
`Math::abs(speed_scale * p_delta) / b.blend_time` (`animation_player.cpp:293`), y el peso
resultante se pasa como `pi.weight = p_blend` (`animation_player.cpp:255`) a
`make_animation_instance` — es decir, `AnimationPlayer` es un *scheduler* de crossfades por
encima del mezclador genérico de pesos por-track de `AnimationMixer`. `blend_times`
(`HashMap<BlendKey, double>`, `animation_player.h:120`) permite declarar duraciones de blend
específicas por par `(from, to)` vía `set_blend_time` (línea 187).

### 3.2 Pistas discretas vs. continuas

`TrackCacheValue` (`animation_mixer.h:223-255`) trae `use_continuous`/`use_discrete` (líneas
230-231) y `is_using_angle` (línea 232, derivado de si la interpolación del track es
`*_ANGLE`, ver `animation_mixer.cpp:759`). En `_blend_apply` (`animation_mixer.cpp:1913` en
adelante, bloque `TYPE_VALUE` en 1980-2020):

```cpp
if (callback_mode_discrete == ANIMATION_CALLBACK_MODE_DISCRETE_FORCE_CONTINUOUS) {
    t->is_init = false;
} else if (!t->use_continuous && (t->use_discrete || !deterministic)) {
    t->is_init = true; // sin valor continuo aplicado, o recién empezado: no hacer RESET
}
if ((t->is_init && (is_zero_amount || !t->use_continuous)) ||
        (callback_mode_discrete != ANIMATION_CALLBACK_MODE_DISCRETE_FORCE_CONTINUOUS &&
                !is_zero_amount &&
                callback_mode_discrete == ANIMATION_CALLBACK_MODE_DISCRETE_DOMINANT &&
                t->use_discrete)) {
    break; // no pisar el valor puesto por UPDATE_DISCRETE
}
```

`AnimationCallbackModeDiscrete` (`animation_mixer.h:65-69`) tiene tres modos:
`DISCRETE_DOMINANT` (un salto discreto gana sobre valores continuos mezclados),
`DISCRETE_RECESSIVE` (el continuo gana) y `FORCE_CONTINUOUS` (fuerza interpolación aunque el track
sea `UPDATE_DISCRETE`). El valor final se escribe con
`t_obj->set_indexed(t->subpath, Animation::cast_from_blendwise(...))`
(`animation_mixer.cpp:2020`), donde `subpath` es el resto de la `NodePath` tras `:` (§1.2).

### 3.3 "RESET animation" y `capture`

Si existe una animación llamada `RESET` (`has_animation(SceneStringName(RESET))`,
`animation_mixer.cpp:699`), sus valores en `t=0` reemplazan el valor inicial (`init_value`) de cada
track cacheado — "toma precedencia sobreescribiendo" (comentario en línea 775):

```cpp
if (has_reset_anim) {
    int rt = reset_anim->find_track(path, track_src_type);
    if (rt >= 0 && reset_anim->track_is_enabled(rt) && reset_anim->track_get_key_count(rt) > 0) {
        track_value->init_value = is_value ? reset_anim->track_get_key_value(rt, 0)
                                            : (reset_anim->track_get_key_value(rt, 0).operator Array())[0];
    }
}
```

(`animation_mixer.cpp:776-784`). Es decir: `RESET` no es una animación especial en tiempo de
ejecución — es una convención de nombre que el mixer busca para establecer el "estado de reposo"
contra el que se blendean todas las demás animaciones.

`capture` (API pública `AnimationMixer::capture`, `animation_mixer.cpp:2355`) crea una
pseudo-animación (`capture_cache`) que hace *crossfade* desde el valor actual hacia la primera clave
de la animación objetivo usando una curva de easing de `Tween`
(`Tween::run_equation(trans_type, ease_type, remain, 0.0, 1.0, 1.0)`, `blend_capture`,
`animation_mixer.cpp:1140-1171`, específicamente línea 1154). Mientras dura la captura, reduce el
peso de las demás instancias activas por `(1 - weight_de_captura)` (líneas 1157-1160) — un
crossfade explícito, no una interpolación dentro de un track.

### 3.4 Resolución de `NodePath` y nodo inexistente

`_update_caches` (`animation_mixer.cpp:662-785`) resuelve cada track vía
`parent->get_node_and_resource(path, resource, leftover_path)` (línea 733). Si el nodo no existe:

```cpp
if (!child) {
    if (check_path) {
        WARN_PRINT_ED(mixer_name + ": '" + String(E) + "', couldn't resolve track: '" + String(path) + "'. ...");
    }
    continue;
}
```

(líneas 734-739): emite un warning (silenciable por proyecto vía
`animation/warnings/check_invalid_track_paths`, línea 675) y **salta ese track por completo** — no
hay error fatal ni excepción; el resto de la animación se sigue procesando. La caché de tracks
(`track_cache`, `AHashMap<TrackCacheID, TrackCache*>`, línea 307 en el header) se invalida
completa cuando cambia `setup_pass` y se reconstruye por completo en la próxima llamada — no hay
resolución incremental.

---

## 4. `Skeleton2D` / `Bone2D`

### 4.1 Qué guarda un hueso

`Bone2D` (`scene/2d/skeleton_2d.h:38-98`) guarda:

```cpp
Bone2D *parent_bone = nullptr;
Skeleton2D *skeleton = nullptr;
Transform2D rest;
bool autocalculate_length_and_angle = true;
real_t length = 16;
real_t bone_angle = 0;
int skeleton_index = -1;
```

(líneas 46-54). El **transform actual** del hueso es simplemente el `Transform2D` heredado de
`Node2D` (posición/rotación/escala del propio nodo en la jerarquía de escena) — `Bone2D` no separa
"transform actual" de "transform de nodo": son lo mismo, y `rest` es un campo aparte que guarda la
pose de referencia.

`Skeleton2D::Bone` (estructura interna, no la clase `Bone2D`; `skeleton_2d.h:110-123`):

```cpp
struct Bone {
    Bone2D *bone = nullptr;
    int parent_index = 0;
    Transform2D accum_transform;
    Transform2D rest_inverse;
    Transform2D local_pose_override;
    real_t local_pose_override_amount = 0;
    bool local_pose_override_persistent = false;
};
```

`rest_inverse` es la inversa de la pose de reposo acumulada (bind pose inversa, comentario
"//bind pose" en `skeleton_2d.cpp:563`); `accum_transform` es el transform global (dentro del
espacio del `Skeleton2D`) acumulado por la jerarquía de huesos en la pose actual.

### 4.2 Cálculo del rest

`Bone2D::set_rest`/`get_rest` son simples setters/getters del campo `rest`
(`skeleton_2d.cpp:387-398`). `get_skeleton_rest` compone recursivamente con el padre:

```cpp
Transform2D Bone2D::get_skeleton_rest() const {
    if (parent_bone) {
        return parent_bone->get_skeleton_rest() * rest;
    } else {
        return rest;
    }
}
```

(`skeleton_2d.cpp:400-406`) — el rest de un hueso hijo es local a su padre, y `get_skeleton_rest`
lo lleva a espacio del `Skeleton2D` multiplicando la cadena de rests locales.
`apply_rest` (`skeleton_2d.cpp:408-410`) simplemente hace `set_transform(rest)`: fuerza el
transform actual del nodo de vuelta a su pose de reposo local.

`calculate_length_and_rotation` (`skeleton_2d.cpp:435-453`) deriva `length`/`bone_angle`
automáticamente **de la posición del primer hijo `Bone2D`** (si existe): transforma la posición
global del hijo al espacio local del hueso (`global_inv.xform(child->get_global_position())`,
línea 444) y toma su longitud y ángulo. Si no hay hijos `Bone2D`, emite un warning y usa
`get_transform().get_rotation()` como ángulo (línea 452) — el largo no se puede inferir sin un
hijo.

### 4.3 "Skeleton base transform" y transformaciones globales

`Skeleton2D::_update_bone_setup` (`skeleton_2d.cpp:552-578`) recorre los huesos **ordenados**
(`bones.sort()`, línea 560 — por comparación en `Bone::operator<`, que usa `is_greater_than` de
`Bone2D`, i.e. orden de profundidad en el árbol de escena) y para cada uno computa:

```cpp
bones[i].rest_inverse = bones[i].bone->get_skeleton_rest().affine_inverse(); //bind pose
```

(línea 563). `Skeleton2D::_update_transform` (`skeleton_2d.cpp:590-614`) hace dos pasadas: primero
acumula el transform global de cada hueso en espacio del esqueleto encadenando con su padre
(`accum_transform = padre.accum_transform * hueso.get_transform()`, línea 604), y luego, para
cada hueso, calcula el transform final que se envía al renderer:

```cpp
Transform2D final_xform = bones[i].accum_transform * bones[i].rest_inverse;
RS::get_singleton()->skeleton_bone_set_transform_2d(skeleton, i, final_xform);
```

(líneas 611-612) — el patrón clásico `pose_actual × inversa(bind_pose)` que da la transformación de
deformación por hueso, subida a un recurso `RID skeleton` (`skeleton_2d.h:135`) que gestiona el
`RenderingServer`.

Aparte de eso, el propio `Skeleton2D` (como `Node2D`) tiene su transform global de escena, que se
sube **por separado** al `RenderingServer` cada vez que cambia (`NOTIFICATION_TRANSFORM_CHANGED`,
`skeleton_2d.cpp:679-699`, específicamente línea 686):

```cpp
RS::get_singleton()->skeleton_set_base_transform_2d(skeleton, get_global_transform());
```

Esa es la "skeleton base transform" del enunciado: la posición/rotación/escala del propio nodo
`Skeleton2D` en el mundo, que el renderer combina con las matrices por-hueso (`final_xform`) al
momento de deformar la malla — los `final_xform` están en espacio local del esqueleto, no en
espacio de mundo.

---

## 5. `Polygon2D`

### 5.1 Datos

`scene/2d/polygon_2d.h:41-68`:

```cpp
Vector<Vector2> polygon;
Vector<Vector2> uv;
Vector<Color> vertex_colors;
Array polygons;                 // sub-polígonos (huecos, islas) sobre el mismo array de puntos
int internal_vertices = 0;      // vértices auxiliares no dibujados (usados por Delaunay interno)

struct Bone {
    NodePath path;
    Vector<float> weights;      // un peso por CADA vértice del polígono
};
Vector<Bone> bone_weights;

NodePath skeleton;
ObjectID current_skeleton_id;
```

Nótese la forma de `bone_weights`: es una lista **por-hueso**, cada entrada con un array de pesos
tan largo como `polygon` (un peso por vértice, potencialmente 0). No es una lista por-vértice de
`(bone_id, weight)` como el `SkinnedVertex.weights: Vec<BoneWeight>` de `lotte-rig` — es la
transposición: por-hueso × todos-los-vértices.

### 5.2 Conexión al `Skeleton2D`

`_skeleton_bone_setup_changed` (`polygon_2d.cpp:108-145`) resuelve `skeleton` (el `NodePath`) a un
nodo `Skeleton2D*` y llama:

```cpp
RS::get_singleton()->canvas_item_attach_skeleton(get_canvas_item(), skeleton_node->get_skeleton());
```

(línea 132; línea 135 hace lo mismo con `RID()` para desconectar, y el destructor en línea 765
también desconecta). El `CanvasItem` de `Polygon2D` queda así asociado al `RID` de esqueleto del
`Skeleton2D` (el mismo `RID skeleton` de §4.3) directamente en el `RenderingServer` — no hay
referencia C++ directa mantenida por `Polygon2D` a los `Transform2D` de cada hueso.

### 5.3 Empaquetado CPU de bones/weights (hasta 4 por vértice)

En la reconstrucción de la malla (`polygon_2d.cpp:246-309`), si hay `skeleton_node` y
`bone_weights`:

```cpp
bones.resize(vc * 4);
weights.resize(vc * 4);
...
for (int i = 0; i < bone_weights.size(); i++) {
    ...
    int bone_index = bone->get_index_in_skeleton();
    for (int j = 0; j < vc; j++) {
        if (r[j] == 0.0) continue; // peso sin pintar, se salta
        for (int k = 0; k < 4; k++) {
            if (weightsw[j * 4 + k] < r[j]) {
                // insertion sort: desplaza los 3 restantes y mete este peso
                ...
                weightsw[j * 4 + k] = r[j];
                bonesw[j * 4 + k] = bone_index;
                break;
            }
        }
    }
}
// normaliza para que los 4 pesos de cada vértice sumen 1
for (int i = 0; i < vc; i++) {
    real_t tw = 0.0;
    for (int j = 0; j < 4; j++) tw += weightsw[i * 4 + j];
    if (tw == 0) continue;
    for (int j = 0; j < 4; j++) weightsw[i * 4 + j] /= tw;
}
```

(líneas 249-308). Este paso — **transponer de por-hueso a por-vértice, quedarse con los 4 pesos
más grandes por vértice (insertion sort, líneas 279-290) y normalizar (líneas 294-308)** — corre
en CPU cada vez que la malla se reconstruye (no cada frame; solo si cambia el polígono o el setup
de huesos).

### 5.4 CPU vs. GPU en el skinning real

El array final se sube al `RenderingServer` como parte de los arrays estándar de malla:

```cpp
arr[RSE::ARRAY_BONES] = bones;
arr[RSE::ARRAY_WEIGHTS] = weights;
...
RS::get_singleton()->mesh_create_surface_data_from_arrays(&sd, RSE::PRIMITIVE_TRIANGLES, arr, ...);
```

(`polygon_2d.cpp:387-388`, superficie subida en 411-425), y el dibujo real ocurre con:

```cpp
RS::get_singleton()->canvas_item_add_mesh(get_canvas_item(), mesh, Transform2D(), Color(1, 1, 1),
        texture.is_valid() ? texture->get_scaled_rid() : RID());
```

(línea 435). **La aplicación del skinning (transformar cada vértice por sus hasta-4 huesos
ponderados) ocurre en el `RenderingServer`/GPU**, no en `Polygon2D::_notification`: `Polygon2D`
solo construye una vez (o cuando cambia) los índices/pesos de hueso por vértice y el `RID` de
esqueleto asociado (§5.2); el `RenderingServer` sube las matrices `final_xform` por hueso
(`skeleton_bone_set_transform_2d`, §4.3) a una textura de huesos y el shader de canvas la muestrea
por vértice en cada draw call. No hay evidencia en este checkout (sparse, sin `servers/`) de la
implementación del shader — se infiere de la combinación `canvas_item_attach_skeleton` +
`skeleton_bone_set_transform_2d` + `ARRAY_BONES`/`ARRAY_WEIGHTS`, el mismo patrón que Godot documenta
para GPU skinning 2D.

---

## 6. `SkeletonModification2D*` — existen, pero NO en `scene/resources/` directamente

**Corrección al brief**: el enunciado asumía `scene/resources/skeleton_modification_2d*.h/.cpp`.
Medido con `ls scene/resources/*.h | grep -i modification` → vacío. Los archivos sí existen, pero
en un subdirectorio: `scene/resources/2d/skeleton/` (confirmado con
`ls scene/resources/2d/skeleton/`, 17 archivos: `skeleton_modification_2d.{h,cpp}`,
`skeleton_modification_stack_2d.{h,cpp}` y un `.h`/`.cpp` por modificador). Como
`scene/resources/2d/` está dentro de la carpeta con sparse-checkout `scene/resources`, el contenido
sí está disponible en este checkout de referencia.

`SkeletonModification2D` es la clase base (`skeleton_modification_2d.h/.cpp`); cada modificador
corre desde una `SkeletonModificationStack2D` (`Skeleton2D::execute_modifications`,
`skeleton_2d.cpp:702` y `:712`, llamada en `NOTIFICATION_INTERNAL_PROCESS`/
`NOTIFICATION_INTERNAL_PHYSICS_PROCESS` respectivamente). Los siete modificadores presentes
(`scene/register_scene_types.cpp:874-884`) y qué hace cada uno según su `_execute`:

| Modificador | Archivo:línea de `_execute` | Qué hace |
|---|---|---|
| `SkeletonModification2DCCDIK` | `skeleton_modification_2d_ccdik.cpp:159` | CCDIK (Cyclic Coordinate Descent IK): itera una cadena de huesos rotando cada uno para acercar el "tip" al `target_node`, un paso por hueso por frame. |
| `SkeletonModification2DFABRIK` | `skeleton_modification_2d_fabrik.cpp:103` | FABRIK (Forward And Backward Reaching IK): resuelve la cadena en dos pasadas (hacia el target y hacia atrás al origen) usando una `fabrik_transform_chain` (`skeleton_modification_2d_fabrik.h:61`); requiere al menos dos joints (línea 116-117 del `.cpp`). |
| `SkeletonModification2DJiggle` | `skeleton_modification_2d_jiggle.cpp:140`, joint individual en `:162` (`_execute_jiggle_joint`) | Simulación resorte-masa-amortiguador por joint ("Adopted from Unity JiggleBone", comentario línea 163): `force = (target - dynamic_position) * stiffness * delta` más gravedad opcional, `acceleration = force / mass`, integra velocidad con `damping`, con detección de colisión opcional (`use_colliders`). |
| `SkeletonModification2DLookAt` | `skeleton_modification_2d_lookat.cpp:110` | Rota un `Bone2D` para que su eje "adelante" apunte hacia `target_node`, con `clamp_angle` opcional (límites de ángulo, `skeleton_modification_2d.cpp:84`). |
| `SkeletonModification2DTwoBoneIK` | `skeleton_modification_2d_twoboneik.cpp:108` | IK analítico de dos huesos (ley de cosenos, "Adapted from... simple-two-joint / IK-2D-2", comentario líneas 146-148): calcula el ángulo de cada joint a partir de la distancia al target y los largos de los dos huesos, con `target_minimum_distance`/`target_maximum_distance` para evitar sobre-extensión. |
| `SkeletonModification2DPhysicalBones` | `skeleton_modification_2d_physicalbones.cpp:106` | Sincroniza una cadena de `PhysicalBone2D` (huesos con cuerpo físico/`RigidBody2D`) con el `Skeleton2D`; expone `start_simulation`/`stop_simulation` (`skeleton_modification_2d_physicalbones.h:74-75`) para activar/desactivar la simulación física por hueso. |
| `SkeletonModification2DStackHolder` | `skeleton_modification_2d_stackholder.cpp:83` | No transforma huesos por sí mismo: delega ejecución a otro `SkeletonModificationStack2D` completo (`held_modification_stack->execute(...)`, línea 88) — permite anidar/componer pilas de modificadores. |

Todos comparten el mismo guard de entrada (`ERR_FAIL_COND_MSG(!stack || !is_setup ||
stack->skeleton == nullptr, ...)`, repetido idéntico al inicio de cada `_execute`) y resuelven sus
`NodePath` objetivo vía una caché de `ObjectID` (`target_node_cache`, patrón repetido en cada
clase) que se recalcula perezosamente con un `update_*_cache()` cuando está `is_null()`.

---

## Portable a Lotte

Con atribución MIT explícita en el commit que porte cada pieza (cita a este documento +
archivo:línea de Godot):

- **Modelo de canal `(NodePath, propiedad)` → `(ruta de nombres, atributo)`.** El patrón de Godot
  — `Track.path: NodePath` con subnombre tras `:`, resuelto a `(objeto, subpath)` una sola vez y
  cacheado (`animation_mixer.cpp:709-739`) — es directamente el modelo que la decisión abierta de
  Lotte ("identidad de canal como ruta de nombres + atributo") ya apunta a adoptar. Portable:
  la separación entre *dirección* (ruta) y *resolución* (a un `NodeId` de `RigDag`, cacheada) y el
  comportamiento de "nodo no encontrado → warning + skip del track, resto de la animación sigue"
  (`animation_mixer.cpp:734-739`). Adaptar: en Lotte la resolución es contra `NodeId(usize)`
  posicional del DAG, no un puntero a `Object`; la caché puede vivir en la capa de reproducción de
  `lotte-timeline`/`lotte-rig`, no en `FCurve` (que hoy es agnóstica de canal).

- **Evaluación de bezier track por bisección (§2.2).** El algoritmo de 10 iteraciones de bisección
  sobre `x(t)` seguido de un lerp final es simple, sin dependencias externas y encaja con el
  `Keyframe::Bezier{handle_in, handle_out}` que ya existe en `lotte-timeline::FCurve`. Portable
  casi literal: la función pura `bezier_track_interpolate` (`animation.cpp:3607-3670`) no toca
  estado de Godot. Adaptar: en Lotte las manijas de `Keyframe` no tienen el clamp de signo que
  Godot aplica en inserción (`in_handle.x ≤ 0`, `out_handle.x ≥ 0`,
  `animation.cpp:3352-3358`) — si Lotte quiere la misma garantía de curva monótona en tiempo, ese
  clamp debe vivir en el constructor/setter de `Keyframe::Bezier`, no asumirse.

- **`rest` / `apply_rest` de `Bone2D` como modelo de "reposo" para definición/instancia (§4.2).**
  El patrón `rest: Transform2D` local al padre + `get_skeleton_rest()` que compone la cadena +
  `rest_inverse = get_skeleton_rest().affine_inverse()` (bind pose) es exactamente la forma que
  necesita la decisión abierta de Lotte "valor efectivo = reposo + animación(t)" para
  `PegNode.transform`. Portable: la fórmula `final = accum_pose × inverse(rest_acumulado)`
  (`skeleton_2d.cpp:611`) como base del futuro pipeline de skinning en `lotte-rig` (M6). Adaptar:
  en Lotte `Transform2D` ya tiene `pivot` y `shear` que Godot no separa igual (Godot mete todo en
  una matriz 2x3); la composición de `rest` vía multiplicación de matrices debe pasar por el mismo
  camino que usa `RigDag::evaluate_all` (invalidación de subárbol) en vez de recalcularse ad-hoc
  como hace `_update_bone_setup`/`_update_transform` (dos flags dirty separados,
  `skeleton_2d.h:127,131`, redundante con la invalidación perezosa que `RigDag` ya implementa).

- **Pesos de `Polygon2D` (§5.1, §5.3), con adaptación de layout.** El límite de 4
  huesos-por-vértice con normalización e insertion-sort por peso descendente
  (`polygon_2d.cpp:279-308`) es un patrón razonable a portar para `DeformableMesh` en `lotte-rig` —
  pero el *layout* de almacenamiento de Godot (por-hueso × todos-los-vértices,
  `Vector<Bone{path, weights}>`) es la transposición del `SkinnedVertex.weights: Vec<BoneWeight>`
  por-vértice que ya existe en `lotte-rig`. Portar el algoritmo de selección/normalización de los
  N pesos más grandes por vértice, no la estructura de almacenamiento — construir el
  `Vec<BoneWeight>` por vértice directamente en vez de transponer desde un array por-hueso.

- **`RESET` como convención de nombre para el estado de reposo del blending (§3.3).** Útil como
  referencia para el M5 (blending por capas) de Lotte: una animación con nombre reservado que fija
  el valor base contra el que blendean las demás, sin ser un mecanismo nuevo en el motor — es una
  convención en la capa de aplicación (buscar `has_animation("RESET")`). Adaptar el nombre/mecanismo
  de búsqueda a como Lotte organice sus animaciones (hoy no hay ese concepto en
  `lotte-timeline`/`lotte-format`).

- **Separación pesos-por-track y normalización condicional a `deterministic` (§3.1).** El patrón
  `blend_track = peso_animación × peso_por_track`, con normalización opcional por `total_weight`
  cuando el modo no es determinista, es un buen punto de partida conceptual para el M5 de Lotte
  (blending por capas) — pero es best-effort/heurístico (Godot lo llama explícitamente
  "indeterministic blending" en el nombre de la función,
  `_blend_calc_total_weight`, comentario en `animation_mixer.h:371`). Adaptar, no copiar tal cual:
  Lotte puede optar por un modelo de capas con prioridad explícita en vez de normalización
  implícita por suma de pesos.

- **Qué NO portar:** el modelo dominante/recesivo de tracks discretas (§3.2,
  `AnimationCallbackModeDiscrete`) resuelve un problema específico de Godot (propiedades
  "oneshot", señales, tracks de método mezclados con tracks continuas en el mismo mixer) que no
  tiene equivalente hoy en el modelo de Lotte (`FCurve` es puramente numérica: `Hold | Linear |
  Bezier`, sin variante de tipo `Variant` polimórfico). El sistema de audio (`TYPE_AUDIO`,
  `AudioTrack`) y el de sub-animaciones anidadas (`TYPE_ANIMATION`) están fuera del alcance actual
  de Lotte (sin motor de audio, sin decisión de composición de animaciones tomada). El `capture`
  crossfade con curvas de `Tween` (§3.3) depende del sistema de `Tween` de Godot, no portable sin
  antes decidir si Lotte tendrá una capa de easing equivalente a nivel de reproducción (hoy el
  easing por-clave vive en `Keyframe::interpolation`, no hay una capa de transición entre
  animaciones separada).

---

## Lo que NO existe o no encontré

- **`scene/resources/skeleton_modification_2d*.h/.cpp` en la ruta que asumía el brief**: no existen
  ahí; existen en `scene/resources/2d/skeleton/` (ver §6, corrección explícita).
- **`TYPE_TRANSFORM_2D` o track dedicado de transform 2D**: no existe en `TrackType`
  (`animation.h:48-58`); las animaciones de posición/rotación/escala 2D pasan por `TYPE_VALUE`
  sobre las propiedades del nodo, no por un track de transform compuesto como los 3D.
  ("A verificar por el developer": no confirmé si `AnimationTrackEditor` del editor genera 3
  `TYPE_VALUE` separados o un patrón distinto para 2D — el editor no está en el sparse checkout,
  solo `scene/`).
- **Implementación del shader de skinning 2D en GPU**: no está en este checkout (`servers/` no
  forma parte del sparse checkout); la inferencia de §5.4 se apoya en las llamadas de
  `RenderingServer` (`canvas_item_attach_skeleton`, `skeleton_bone_set_transform_2d`,
  `ARRAY_BONES`/`ARRAY_WEIGHTS`) visibles desde `scene/`, no en el código del vertex shader mismo.
- **`Curve` (`scene/resources/curve.h`)**: presente en el sparse checkout pero es un recurso
  genérico (usado en gradientes, parámetros de partículas, etc.), no forma parte del sistema de
  pistas de `Animation`; no se detalla en este informe centrado en animación/esqueleto porque no
  estaba en la lista de puntos a extraer.
- **Cómo `AnimationTree`/`AnimationNodeBlendTree` arma los pesos por-track antes de llamar a
  `_blend_process`**: `animation_blend_tree.*`/`animation_node_state_machine.*` están en el sparse
  checkout (`scene/animation/`) pero no se auditaron — el punto 3 del brief pide `AnimationMixer`/
  `AnimationPlayer`, no `AnimationTree`; se dejan fuera de alcance de este informe.

---

## Cómo se midió

Todos los comandos corrieron desde `references/godot/` el 2026-09-08, sobre el checkout
sparse ya presente (no se clonó ni modificó nada).

| Verificación | Comando | Resultado |
|---|---|---|
| Commit y fecha | `git log -1 --format="%H %ci"` | `6a0f6f32cfb2ce4cc5bad6641d0afda413b62a9d 2026-09-07 16:16:21 -0500` |
| Sparse-checkout activo | `git sparse-checkout list` | `scene/2d`, `scene/animation`, `scene/gui`, `scene/resources` |
| Ausencia de `skeleton_modification_2d*` en `scene/resources/` (top-level) | `ls scene/resources/ \| grep -iE "modification"` | vacío |
| Ubicación real de `SkeletonModification2D*` | `ls scene/resources/2d/skeleton/` | 17 archivos (`.h`/`.cpp` de la base + 7 modificadores + stack) |
| Enums de `Animation` | `grep -n "enum TrackType\|enum InterpolationType\|enum UpdateMode\|enum LoopMode\|enum LoopedFlag\|enum HandleMode\|enum HandleSetMode" scene/resources/animation.h` | líneas 48, 60, 68, 74, 81, 94, 100 |
| Núcleo de interpolación de valores | `Read scene/resources/animation.cpp` offset 2519, limit 220 | función `_interpolate` líneas 2519-2721 |
| Bisección en bezier track | `Read scene/resources/animation.cpp` offset 3341, limit 330 | `bezier_track_interpolate` líneas 3607-3670, loop de bisección 3640-3662 |
| Blending por peso/track en el mixer | `Read scene/animation/animation_mixer.cpp` offset 1234, limit 130 | `_blend_process`, normalización en líneas 1281-1288 |
| RESET animation y resolución de NodePath | `Read scene/animation/animation_mixer.cpp` offset 662, limit 140 | líneas 698-701 (RESET), 733-739 (resolución + warning), 775-785 (override de init_value) |
| `rest`/`apply_rest`/`get_skeleton_rest` de Bone2D | `Read scene/2d/skeleton_2d.cpp` offset 387, limit 100 | líneas 387-453 |
| `final_xform = accum_transform * rest_inverse` y base transform | `Read scene/2d/skeleton_2d.cpp` offset 508-614 y 679-699 | líneas 552-578 (`_update_bone_setup`), 590-614 (`_update_transform`), 686 (`skeleton_set_base_transform_2d`) |
| Layout de `Polygon2D::Bone` y conexión a `Skeleton2D` | `Read scene/2d/polygon_2d.h` offset 36, limit 80 | líneas 47-68 |
| Empaquetado CPU de bones/weights y draw call | `Read scene/2d/polygon_2d.cpp` offset 230, limit 150; `grep -n "canvas_item_add_mesh\|ARRAY_BONES\|attach_skeleton"` | líneas 132, 246-309, 387-388, 435 |
| `_execute` de los 7 modificadores 2D | `grep -n "_execute\b" skeleton_modification_2d_*.cpp` (dentro de `scene/resources/2d/skeleton/`) | líneas citadas en la tabla de §6 |

Ningún comando ejecutó binarios ni escribió en el repositorio del motor ni en `O:/Lotte/references`; todas las
lecturas fueron de solo lectura sobre el checkout de referencia, y este informe es el único archivo
escrito, en `(fuera del repositorio)/godot-animacion-esqueleto.md`.
