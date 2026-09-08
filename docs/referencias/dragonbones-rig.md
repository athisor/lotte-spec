# Auditoría: DragonBonesCPP — rig, FFD, blending, formato, topo-sort

- **Repo**: `DragonBonesCPP` (DragonBones team, MIT)
- **Ruta auditada**: `references/DragonBonesCPP/` (solo lectura, checkout principal)
- **Commit**: `f696143045e07b7c4c4590bdd95534e9e8074b37` (medido: `git log -1 --format="%H %ci"` → `2025-07-04 13:19:54 +0800`)
- **Licencia**: MIT (`LICENSE`, "Copyright (c) 2012-2016 DragonBones team and other contributors")
- **Fecha de esta auditoría**: 2026-09-08
- **Estrategia de ingesta**: port directo con atribución MIT

Todas las líneas de abajo fueron re-medidas con `grep -n` sobre este checkout el 2026-09-08. Donde
el brief traía una línea prevista, se anota "prevista vs real".

---

## 1. Herencia selectiva por eje — `Bone.cpp`

**Función real**: `Bone::_updateGlobalTransformMatrix(bool isCache)`,
`DragonBones/src/dragonBones/armature/Bone.cpp:30-236`.
Línea prevista: 74-197 (rango del brief). Línea real de la función completa: 30-236; el bloque de
herencia (`if (inherit) { ... }`) es 74-198. El rango previsto cae *dentro* de la función real pero
no la cubre entera — el `flag` no capturaba las líneas 30-73 (composición del transform local con
`offsetMode`) ni 199-236 (rama sin padre, con flip).

### Mecánica exacta

`offsetMode` decide cómo se compone `global` (pose local) ANTES de considerar al padre:
- `Additive` (`Bone.cpp:37-55`): `global = origin + offset + animationPose` (suma componente a
  componente; `scaleX/scaleY` se multiplican).
- `None` (`Bone.cpp:56-67`): `global = origin + animationPose` (sin `offset`).
- Otro valor (`Bone.cpp:68-72`): `inherit = false` — el hueso ignora al padre por completo,
  `global = offset`.

Si `inherit` es true (`Bone.cpp:74-198`), la matriz del padre entra por dos caminos alternativos
según `_boneData->inheritScale`:

**Caso A — `inheritScale == true`** (`Bone.cpp:77-125`):
```cpp
// Bone.cpp:79-101 — si NO hereda rotación, la rotación local se
// recalcula restando (o sumando, según flip) la del padre ANTES de
// convertir a matriz, para que el hijo mantenga su rotación absoluta
// independientemente de cómo rote el padre.
if (!_boneData->inheritRotation) {
    _parent->updateGlobalTransform();
    rotation = global.rotation - _parent->global.rotation;  // caso sin flip
    global.rotation = rotation;
}

global.toMatrix(globalTransformMatrix);
globalTransformMatrix.concat(parentMatrix);   // <- concat completo (incluye escala del padre)

if (_boneData->inheritTranslation) {
    global.x = globalTransformMatrix.tx;      // posición ya en espacio del padre
    global.y = globalTransformMatrix.ty;
} else {
    globalTransformMatrix.tx = global.x;      // pisa la traslación heredada con la local
    globalTransformMatrix.ty = global.y;
}
```
Es decir: cuando se hereda escala, se compone la matriz LOCAL completa contra la del padre
(`concat`, multiplicación de matrices 2x3 estándar) y LUEGO se decide si la traslación resultante
se conserva (heredada) o se pisa con la traslación local sin heredar.

**Caso B — `inheritScale == false`** (`Bone.cpp:126-197`): NO hay `concat` de matriz. Cada eje se
compone a mano:
```cpp
// Traslación (Bone.cpp:128-146): si inheritTranslation, rota+traslada
// el punto local por la matriz del padre SIN aplicar su escala via concat
// (matriz 2x2 a,b,c,d ya trae la rotación/escala del padre, pero acá se usa
// tal cual — la escala del padre SÍ afecta la traslación, no la orientación local).
global.x = parentMatrix.a * x + parentMatrix.c * y + parentMatrix.tx;
global.y = parentMatrix.b * x + parentMatrix.d * y + parentMatrix.ty;

// Rotación (Bone.cpp:148-172): si inheritRotation, se SUMA la rotación
// absoluta del padre a la local (composición angular pura, no matricial).
rotation = global.rotation + _parent->global.rotation;
global.rotation = rotation;

global.toMatrix(globalTransformMatrix);  // matriz final se reconstruye desde los campos ya combinados
```

**La parte de `flipX`/`flipY` que Lotte NO debería portar** (`Bone.cpp:32-33, 83-98, 137-145,
152-194, 201-231`): `flipX`/`flipY` son booleanos por-armadura (`Armature::getFlipX/getFlipY`) que
espejan TODO el árbol invirtiendo signos de traslación y sumando `PI` a la rotación, con una rama
distinta por cada combinación de `(flipX, flipY, inheritRotation, inheritScale, sign(determinante
de parentMatrix))`. El código resultante (`Bone.cpp:83-98` y `152-194`) tiene 6 combinaciones
explícitas de suma/resta de `PI` y un caso adicional que ajusta `global.skew += PI` cuando el
determinante de la matriz del padre es negativo (un padre ya espejado) XOR el flip local — es una
forma de "deshacer" el espejo doble sin usar escala negativa consistente. Es un enredo porque:
1. Mezcla dos mecanismos de espejado (`flip` a nivel armadura + escala negativa a nivel hueso) que
   interactúan de forma no conmutativa.
2. La corrección de signo depende de `parentMatrix.a*d - parentMatrix.b*c < 0` (el signo del
   determinante), es decir, detecta espejado ya acumulado por handedness de la matriz en vez de
   por un flag explícito — frágil ante cualquier reordenamiento de operaciones.
3. Con `Transform2D { scale: Vec2 }` de Lotte, una escala negativa en X o Y ya expresa un flip de
   forma composable y conmutativa dentro del propio pipeline de matrices (`glam::Affine2`), sin
   necesitar un flag de armadura aparte ni las ramas de `skew += PI`.

---

## 2. FFD por keyframe — `DeformVertices` + `SFMLSlot::_updateMesh` + `DeformTimelineState`

**Almacenamiento del delta**: `DeformVertices::vertices` (`std::vector<float>`),
`DragonBones/src/dragonBones/armature/DeformVertices.h:38` (declaración) y
`DeformVertices.cpp:16-31` (`init`, reserva `vertexCount * 2` floats en cero — un delta `(dx, dy)`
por vértice del bind pose, en el mismo espacio que las coordenadas del mesh en el atlas).

**Aplicación del delta (LBS)**: `SFMLSlot::_updateMesh()`,
`SFML/src/dragonBones/SFMLSlot.cpp:255-347`.
Línea prevista: 280-317. Línea real de la función completa: 255-347; el bloque LBS+FFD con pesos es
280-317 (coincide con lo previsto) y el bloque no-skinneado (mesh libre, sin huesos) es 319-346 —
el brief no cubría esta segunda rama.

**Orden exacto** (rama skinneada, `SFMLSlot.cpp:280-308`):
```cpp
auto xL = floatArray[iV++] * scale;   // posición LOCAL de bind (espacio del atlas, escalada)
auto yL = floatArray[iV++] * scale;

if (hasFFD) {
    xL += deformVertices[iF++];       // 1) delta FFD se SUMA en espacio de BIND, ANTES del LBS
    yL += deformVertices[iF++];
}

xG += (matrix.a * xL + matrix.c * yL + matrix.tx) * weight;   // 2) LBS: cada hueso influyente
yG += (matrix.b * xL + matrix.d * yL + matrix.ty) * weight;   //    transforma (xL,yL) YA deformado
                                                                //    y acumula ponderado por weight
```
Confirmado: **el delta de FFD se suma en espacio de bind ANTES del skinning**, y luego el vértice ya
deformado es el que entra al blend lineal de matrices de huesos (LBS clásico, `sum(weight_i *
M_i * v_local)`). Para mesh sin huesos (`SFMLSlot.cpp:319-346`) el delta se suma directo a la
posición del vértice en espacio mundo del atlas (no hay LBS: `xG = bindX*scale + deformVertices[i]`).

**Dónde se guarda el delta por keyframe**: `DeformTimelineState` (declarado
`DragonBones/src/dragonBones/animation/TimelineState.h:159`), implementado en
`DragonBones/src/dragonBones/animation/TimelineState.cpp:772-952`. Cada keyframe trae un array
plano de floats (`_current`, tamaño `_valueCount` = 2×vértices afectados) leído directo del blob
binario (`TimelineState.cpp:806-816`); `_onArriveAtFrame` calcula `_delta[i] = next - current` para
interpolar entre keyframes (`TimelineState.cpp:806-809`), y `_onUpdateFrame` (línea 828-842) hace
`_result[i] = _current[i] + _delta[i] * _tweenProgress` — interpolación lineal por-componente del
delta, NO del vértice absoluto. El resultado final se escribe en
`DeformVertices::vertices` (el mismo array que lee `_updateMesh`) dentro de
`DeformTimelineState::update` (`TimelineState.cpp:883-...`), incluyendo mezcla de fade-in/out
(`TimelineState.cpp:902-935`, ver sección 3).

---

## 3. Blending de animaciones por capas — `AnimationState.cpp`

**Función clave real**: `BlendState::update(float weight, int p_layer)`,
`DragonBones/src/dragonBones/animation/AnimationState.cpp:898-939`.
Línea prevista: 898-939 — coincide exacto con lo medido.

### Modelo de presupuesto de peso descendente (por hueso, no global)

Cada `Bone` tiene su propio `_blendState` (tipo `BlendState`, campos en `AnimationState.h:521-529`:
`layer`, `layerWeight`, `blendWeight`, `leftWeight`, `dirty`). Se resetea una vez por frame con
`BlendState::clear()` (`AnimationState.cpp:941-948`: `dirty=false; layer=0; leftWeight=0;
layerWeight=0`).

Cada `AnimationState` activo, en orden de capa (mayor `layer` primero — así lo dice el comentario
del header: *"High layer animation state will get the blend weight first"*,
`AnimationState.h:108-110`), llama `bone->_blendState.update(_weightResult, layer)` por cada hueso
que toca (`AnimationState.cpp:608`).

```cpp
int BlendState::update(float weight, int p_layer) {
    if (dirty) {                              // ya hubo un estado de mayor prioridad este frame
        if (leftWeight > 0.0f) {
            if (layer != p_layer) {           // cambia de capa (capa nueva, más baja)
                if (layerWeight >= leftWeight) {
                    leftWeight = 0.0f;
                    return 0;                  // presupuesto agotado por la capa anterior: NO se aplica
                } else {
                    layer = p_layer;
                    leftWeight -= layerWeight;  // resto de presupuesto que baja a la nueva capa
                    layerWeight = 0.0f;
                }
            }
        } else {
            return 0;                          // sin presupuesto restante: este estado no pinta nada
        }
        weight *= leftWeight;                  // el peso de este estado se recorta al presupuesto que queda
        layerWeight += weight;
        blendWeight = weight;
        return 2;                              // "blend" (mezclar con lo que ya hay)
    }
    dirty = true;                              // primer estado que toca este hueso este frame
    layer = p_layer;
    layerWeight = weight;
    leftWeight = 1.0f;                         // presupuesto total = 1.0
    blendWeight = weight;
    return 1;                                  // "pose" (sobreescribir)
}
```

**Fórmula del peso efectivo**: `_weightResult = weight * _fadeProgress`
(`AnimationState.cpp:553`), donde `weight` es el peso configurado del `AnimationState`
(`AnimationConfig::weight`, `AnimationState.cpp:440`) y `_fadeProgress` viene de la rampa de
fade (ver abajo). Ese `_weightResult` es el `weight` que entra a `BlendState::update`. El peso que
realmente aplica un hueso en una capa dada es:

```
leftWeight_capa = 1.0 - Σ(layerWeight de todas las capas de MAYOR prioridad ya procesadas)
blendWeight_estado = weightResult_estado * leftWeight_capa   (si leftWeight_capa > 0, si no: 0)
```
Es decir: la capa de mayor `layer` consume presupuesto de 0 a 1 entre todos sus `AnimationState`
activos; lo que sobra (`1 - Σ layerWeight`) baja a la siguiente capa; si una capa ya consumió todo
(`layerWeight >= leftWeight`), las capas inferiores no pintan nada en ese hueso. Es un
presupuesto descendente por capa, evaluado hueso-por-hueso, no global por armadura.

### Rampa de fade in/out

`AnimationState::_advanceFadeTime` (`AnimationState.cpp:362-420`): `_fadeTime` avanza con el tiempo
transcurrido; `_fadeProgress` es una rampa LINEAL en `[0,1]` sobre `fadeTotalTime`:
```cpp
_fadeProgress = isFadeOut ? (1.0f - _fadeTime / fadeTotalTime) : (_fadeTime / fadeTotalTime);
```
Para el FFD específicamente (`TimelineState.cpp:904`) se usa `pow(_fadeProgress, 2)` (rampa
cuadrática) en vez de la lineal — asimetría documentada donde el FFD se atenúa más rápido al
principio del fade.

### Qué huesos toca cada estado — boneMask

`AnimationState::containsBoneMask(name)` (`AnimationState.cpp:744-747`):
```cpp
return _boneMask.empty() || std::find(_boneMask.cbegin(), _boneMask.cend(), boneName) != _boneMask.cend();
```
`_boneMask` vacío = afecta a TODOS los huesos (comportamiento por defecto). Si no está vacío, es una
lista blanca por nombre. Se usa como filtro al construir las timelines del estado
(`AnimationState.cpp:147` para timelines de slot/acción, `AnimationState.cpp:247` para bone
timelines) — un hueso fuera de la máscara simplemente no tiene `BoneTimelineState` en ese
`AnimationState`, así que nunca llama a `_blendState.update` para él. `addBoneMask`/`removeBoneMask`
(`AnimationState.cpp:749-826`) soportan modo recursivo (`currentBone->contains(bone)`) para incluir
toda una sub-jerarquía de huesos con una sola llamada.

---

## 4. Formato JSON — jerarquía, unidades y claves

**Parser**: `DragonBones/src/dragonBones/parser/JSONDataParser.cpp` (clase `JSONDataParser`,
hereda de `DataParser`, que declara ~150 constantes `static const char*` con las claves JSON en
`DataParser.h:60-182` y sus valores string literales en `DataParser.cpp:5-...`).

**Jerarquía real** (medida en `JSONDataParser.cpp`):
- `parseDragonBonesData` (línea 2000) → por cada `armature` (dentro de `_parseDragonBonesData`,
  línea 1829) llama `_parseArmature` (línea 148).
- `_parseArmature` (148-334): parsea `bone[]` (194-231, con resolución de `parent` por nombre y
  caché para huesos declarados fuera de orden), `ik[]` (233-244), `sortBones()` (246), `slot[]`
  (247-255, con `zOrder` incremental), `skin[]` (256-263, `_parseSkin` no listada arriba pero
  invocada en 261), `animation[]` (287-293).
- Dentro de cada `skin`: slots con `display[]` (`_parseDisplay`, línea 481) — imagen, mesh
  (`_parseMesh`, 605), o armadura anidada.
- `animation` trae timelines por hueso (`_parseBoneTimeline`, 1088) y por slot
  (`_parseSlotTimeline`, 1155, que incluye `displayFrame`, `colorFrame` y `ffd` — línea 879).

**Unidades**:
- Posición/tamaño: **píxeles**, escalados por `armature->scale` (p.ej.
  `_parseTransform(rawData[TRANSFORM], bone->transform, _armature->scale)`, línea 353).
- Rotación y skew: **grados en el JSON**, convertidos a radianes al parsear con la constante
  `Transform::DEG_RAD`: `transform.rotation = Transform::normalizeRadian(_getNumber(rawData,
  ROTATE, 0.0f) * Transform::DEG_RAD)` (`JSONDataParser.cpp:1794`, y de nuevo en 1434 para
  keyframes de rotación). Confirmado con `grep -n "DEG_RAD\|ROTATE"` — no existe una constante
  separada "ANGLE_TO_RADIAN"; la conversión vive inline en cada sitio que lee `ROTATE`/`SKEW_X`/
  `SKEW_Y`.

### 15 claves JSON más importantes (nombre string real, medido en `DataParser.cpp`)

| Clave JSON | Constante | Significado |
|---|---|---|
| `"armature"` | `ARMATURE` | Array raíz de armaduras (un archivo puede traer varias) |
| `"bone"` | `BONE` | Array de huesos del armature, con `name`/`parent`/`transform`/flags de herencia |
| `"slot"` | `SLOT` | Array de slots (contenedores de display), define `zOrder` por índice |
| `"skin"` | `SKIN` | Array de skins; cada skin mapea slot→lista de displays alternativos |
| `"display"` | `DISPLAY` | Lista de displays de un slot dentro de un skin (imagen/mesh/armadura anidada) |
| `"animation"` | `ANIMATION` | Array de clips de animación del armature |
| `"parent"` | `PARENT` | Nombre del hueso padre (string; resuelto por búsqueda, con caché para orden libre) |
| `"transform"` | `TRANSFORM` | Bloque `{x,y,skX,skY,scX,scY}` de pose de bind (grados en skX/skY) |
| `"inheritTranslation"` | `INHERIT_TRANSLATION` | Flag de herencia selectiva de traslación (ver sección 1) |
| `"inheritRotation"` | `INHERIT_ROTATION` | Flag de herencia selectiva de rotación |
| `"inheritScale"` | `INHERIT_SCALE` | Flag de herencia selectiva de escala (cambia la rama completa del algoritmo) |
| `"inheritReflection"` | `INHERIT_REFLECTION` | Flag que afecta el ajuste de `skew` cuando hay flip acumulado |
| `"ffd"` | `FFD` | Timelines de free-form deformation por slot, un delta `(dx,dy)` por vértice y keyframe |
| `"frame"` | `FRAME` | Array genérico de keyframes (reutilizado por varias timelines) |
| `"ints"` / `"floats"` | `INTS` / `FLOATS` | Blobs binarios compartidos (mesh, pesos, deform) codificados como arrays planos indexados por offset |

Otras claves medidas y relevantes para un importador: `"ik"` (`IK`, constraints de cinemática
inversa — Lotte no las tiene todavía), `"zOrder"` (`Z_ORDER`, timeline de reordenamiento de slots),
`"userData"` (`USER_DATA`, metadata libre por hueso/slot/armature), `"actions"` /
`"defaultActions"` (eventos de reproducción encadenada entre clips).

---

## 5. Topo-sort de `ArmatureData` — orden de evaluación de huesos

**Función real**: `ArmatureData::sortBones()`,
`DragonBones/src/dragonBones/model/ArmatureData.cpp:82-131`. Invocada una sola vez por armadura
al terminar de parsear huesos e IK constraints (`JSONDataParser.cpp:246`, justo después del bloque
`if (rawData.HasMember(IK))`).

**Algoritmo** (transcrito, 12 líneas relevantes):
```cpp
while (count < total) {
    const auto bone = sortHelper[index++];
    if (index >= total) index = 0;                       // recorre en anillo

    if (std::find(sortedBones.cbegin(), sortedBones.cend(), bone)
        != sortedBones.cend()) continue;                  // ya insertado

    // ... flag=true si un constraint con root==bone target aún no está en sortedBones
    if (bone->parent != nullptr &&
        std::find(sortedBones.cbegin(), sortedBones.cend(), bone->parent)
        == sortedBones.cend()) continue;                  // padre aún no insertado

    sortedBones.push_back(bone);
    count++;
}
```

**Confirmado: es O(n²) en el peor caso**, no O(n) ni O(n log n). Motivo, medido directamente en el
código (`ArmatureData.cpp:94-130`):
1. El `while` externo corre hasta `count == total` — hasta `n` inserciones exitosas.
2. Para cada intento (exitoso o no) se hace `std::find` sobre `sortedBones` (crece de 0 a n) para
   chequear "ya insertado" (línea 102) y otro `std::find` por chequeo de padre (línea 123) — cada
   uno O(tamaño actual de `sortedBones`).
3. En el peor caso (huesos declarados en orden inverso a la jerarquía, p.ej. hoja antes que raíz)
   el índice recorre el anillo completo varias veces antes de que cada padre esté disponible: el
   número total de iteraciones del `while` puede ser O(n) por nivel de profundidad, cada una con
   `std::find` O(n) → **O(n²)** total. No hay memoización de "ya visto sin condición cumplida"
   entre vueltas del anillo — es un topo-sort por fuerza bruta con reintentos, sin cola de listos
   ni conteo de grados de entrada (a diferencia de un Kahn's algorithm clásico que sería O(n+e)).

No hay comentario en el código que declare la complejidad — la conclusión es del análisis de las
tres operaciones anidadas (`while` + `std::find` × 2), no de un dato que el propio DragonBones
afirme.

---

## Portable a Lotte

Contexto: `lotte-core::Transform2D { position, pivot, rotation, scale, shear }`,
`lotte-core::RigDag::evaluate_all()`; `lotte-rig::DeformableMesh { vertices:
SkinnedVertex { bind_position, weights: Vec<BoneWeight> } }`,
`evaluate_skinning(&[Affine2]) -> Vec<Vec2>`; `Slot`/`Attachment`. Sin herencia selectiva, sin FFD,
sin blending, sin importador.

### 1. Herencia selectiva (sin flip)

```rust
// En BoneData (o en el nodo del RigDag): dos flags booleanos.
pub struct InheritFlags {
    pub translation: bool,
    pub rotation: bool,
    pub scale: bool,
    // NO portar `reflection`/flip — usar scale.x/y negativos en Transform2D en su lugar.
}

fn compose_global(local: &Transform2D, parent_global: Affine2, inherit: InheritFlags) -> Affine2 {
    if inherit.translation && inherit.rotation && inherit.scale {
        // Caso general: concat completo (equivalente a Bone.cpp:103-104).
        return parent_global * local.to_affine2();
    }
    // Caso parcial: componer eje por eje (equivalente a Bone.cpp:126-197,
    // sin las ramas de flip).
    let mut pos = local.position;
    if inherit.translation {
        pos = parent_global.transform_point2(pos);
    }
    let mut rot = local.rotation;
    if inherit.rotation {
        rot += parent_rotation(parent_global);
    }
    let scale = if inherit.scale {
        parent_scale(parent_global) * local.scale
    } else {
        local.scale
    };
    Affine2::from_scale_angle_translation(scale, rot, pos)
}
```

### 2. FFD antes del LBS

```rust
// DeformableMesh gana un campo opcional de deltas por vértice, mutado por
// keyframe (análogo a DeformVertices::vertices).
pub struct DeformDelta {
    pub deltas: Vec<Vec2>,   // uno por vértice del bind pose; vacío = sin FFD activo
}

fn evaluate_skinning_with_ffd(
    vertices: &[SkinnedVertex],
    ffd: Option<&DeformDelta>,
    bone_matrices: &[Affine2],
) -> Vec<Vec2> {
    vertices.iter().enumerate().map(|(i, v)| {
        // 1) delta FFD se suma en espacio de BIND, antes del LBS (igual que SFMLSlot.cpp:299-303)
        let bind = match ffd {
            Some(d) if !d.deltas.is_empty() => v.bind_position + d.deltas[i],
            _ => v.bind_position,
        };
        // 2) LBS: blend lineal de matrices ponderado, sobre el vértice YA deformado
        v.weights.iter().fold(Vec2::ZERO, |acc, w| {
            acc + bone_matrices[w.bone_index].transform_point2(bind) * w.weight
        })
    }).collect()
}
```
El delta por keyframe se interpola linealmente entre frames ANTES de sumarse (no se interpolan
vértices absolutos): `delta(t) = delta_frame_actual + (delta_frame_siguiente - delta_frame_actual)
* tween_progress` (igual que `TimelineState.cpp:840`).

### 3. Blending por capas — presupuesto descendente por hueso

```rust
// Estado por-hueso, reseteado cada frame (equivalente a BlendState::clear()).
#[derive(Default)]
struct BoneBlendState {
    dirty: bool,
    layer: i32,
    layer_weight: f32,
    left_weight: f32,
}

// Se llama en orden de capa DESCENDENTE (mayor layer primero) por cada
// AnimationState activo que incluya este hueso en su bone_mask.
fn apply_layer_weight(state: &mut BoneBlendState, weight: f32, layer: i32) -> BlendMode {
    if state.dirty {
        if state.left_weight <= 0.0 { return BlendMode::Skip; }
        if state.layer != layer {
            if state.layer_weight >= state.left_weight {
                state.left_weight = 0.0;
                return BlendMode::Skip;
            }
            state.layer = layer;
            state.left_weight -= state.layer_weight;
            state.layer_weight = 0.0;
        }
        let w = weight * state.left_weight;
        state.layer_weight += w;
        return BlendMode::Blend(w);
    }
    state.dirty = true;
    state.layer = layer;
    state.layer_weight = weight;
    state.left_weight = 1.0;
    BlendMode::Pose(weight)
}

// weight_result = animation_state.weight * fade_progress (lineal; ^2 solo para FFD)
```
`bone_mask: Option<HashSet<BoneId>>` (`None`/vacío = afecta a todos, igual que `_boneMask.empty()`
en `AnimationState.cpp:746`) filtra qué huesos reciben la llamada — un hueso fuera de la máscara
nunca entra al presupuesto de esa capa.

### 4. Importador JSON

Mapear 1:1 la jerarquía `armature → {bone[], slot[], skin[], animation[]}` a los tipos de
`lotte-format`/`lotte-rig`. Puntos de atención directos del análisis:
- Convertir `skX`/`skY`/`rotate` de **grados a radianes** en el borde del parser (Lotte usa
  radianes en `Transform2D::rotation`), replicando `* Transform::DEG_RAD` de
  `JSONDataParser.cpp:1794/1434`.
- `scale` del armature multiplica posiciones/tamaños pero NO ángulos — aplicar solo a los campos de
  longitud (igual que `_parseTransform(..., scale)` en `JSONDataParser.cpp:1787-1806`).
- Resolver `parent` por nombre con una pasada de caché (huesos pueden venir en cualquier orden en
  el array `bone[]`, igual que `_cacheBones` en `JSONDataParser.cpp:210-226`) — no asumir que el
  padre siempre precede al hijo en el JSON.
- No portar `ik[]` en la primera versión del importador (Lotte no tiene constraints todavía);
  documentar como campo ignorado, no como error de parseo.

### 5. Topo-sort — reemplazar, no portar

`lotte-core::RigDag` ya resuelve esto con invalidación perezosa. **No portar**
`ArmatureData::sortBones()` (O(n²), sección 5) — usarlo solo como referencia de qué invariante
debe mantener el importador (un hueso no puede evaluarse antes que su padre) y dejar que el DAG de
Lotte, con su propio orden topológico, sea la única fuente de verdad del orden de evaluación.

---

## Lo que NO existe o no encontré

- **No hay una constante `ANGLE_TO_RADIAN` separada**: la conversión grados→radianes está inline
  en cada lectura de `ROTATE`/`SKEW_X`/`SKEW_Y` vía `Transform::DEG_RAD` (buscado con `grep -rn
  "ANGLE_TO_RADIAN|RADIAN_TO_ANGLE|degree|Degree"` sobre todo `DragonBones/src/dragonBones/`: cero
  resultados).
- **No hay `SFMLSlot.cpp:280-317` como la ÚNICA implementación de `_updateMesh`**: es una
  implementación específica del backend SFML; `Slot.cpp` (el genérico, backend-agnóstico) solo
  define el punto de extensión (`Slot::update`, línea 547-675) y deja `_updateMesh` como método
  virtual sin cuerpo en esa clase — cada backend (SFML, Cocos2D-X, etc.) lo implementa aparte. Para
  Lotte esto es una señal a favor: `evaluate_skinning` ya vive en `lotte-rig`, agnóstico de
  renderer, que es la posición correcta.
- **No encontré tests unitarios de C++ en este checkout** (no se buscó exhaustivamente: fuera de
  alcance del brief, que pedía extracción de algoritmo, no cobertura de tests).
- **No revisé `Cocos2DX_3.x/` ni `AndroidCompose/`** — son bindings de otro motor/lenguaje sobre el
  mismo core C++; el core relevante para el port es `DragonBones/src/dragonBones/`, que es lo que
  se auditó.
- **`_parseSkin` no aparece en el grep de firmas `^SkinData\* JSONDataParser::_parse`** porque su
  firma real usa otro tipo de retorno o está definida distinto (se ve invocada en
  `JSONDataParser.cpp:261` como `_parseSkin(rawSkins[i])` pero no se aisló su línea de definición
  exacta — no se afirma un número de línea para ella en este informe).

---

## Cómo se midió

```bash
# Commit y licencia
git -C "references/DragonBonesCPP" log -1 --format="%H %ci"
head -3 "references/DragonBonesCPP/LICENSE"

# Ubicar archivos reales (sin find sobre raíz — find acotado al repo de referencia)
find "references/DragonBonesCPP/DragonBones/src/dragonBones" -maxdepth 2

# Herencia selectiva
grep -n "inherit" DragonBones/src/dragonBones/armature/Bone.cpp
grep -n "void Bone::" DragonBones/src/dragonBones/armature/Bone.cpp

# FFD: almacenamiento y aplicación
cat DragonBones/src/dragonBones/armature/DeformVertices.cpp
grep -n "_updateMesh\|void SFMLSlot::" SFML/src/dragonBones/SFMLSlot.cpp
grep -n "Deform" DragonBones/src/dragonBones/animation/TimelineState.cpp

# Blending por capas
grep -n "layer\|_weightResult\|fadeProgress\|weight" DragonBones/src/dragonBones/animation/AnimationState.cpp
grep -n "boneMask" DragonBones/src/dragonBones/animation/AnimationState.cpp DragonBones/src/dragonBones/animation/AnimationState.h

# Formato JSON
grep -n "^void JSONDataParser::_parse\|^ArmatureData\* JSONDataParser" DragonBones/src/dragonBones/parser/JSONDataParser.cpp
grep -n "DEG_RAD\|ROTATE" DragonBones/src/dragonBones/parser/JSONDataParser.cpp
grep -n "const char\* DataParser::" DragonBones/src/dragonBones/parser/DataParser.cpp

# Topo-sort
grep -n "sort\|Sort" DragonBones/src/dragonBones/model/ArmatureData.cpp
```

Ningún comando usó `find /`, `find C:\` ni `find O:\` sobre la raíz del disco (regla
`buscar-con-glob-nunca-find-raiz`); el único `find` se acotó a
`references/DragonBonesCPP/DragonBones/src/dragonBones`. No se ejecutó `cargo` ni `cmake`
en ningún momento de esta auditoría.
