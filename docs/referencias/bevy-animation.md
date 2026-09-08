# Auditoría: sistema de animación de Bevy (bevy_animation)

**Crate:** `bevy_animation` **Versión leída:** `0.19.1`
**Licencia (textual, `Cargo.toml` generado por Cargo):** `license = "MIT OR Apache-2.0"`
(`.../bevy_animation-0.19.1/Cargo.toml:26`). `bevy_math` (para `cubic_splines/`) es del mismo
workspace y declara la misma licencia.

**Fecha:** 2026-09-08. **Estrategia de ingesta:** patrones y código portables con atribución
MIT/Apache-2.0.

**Alcance leído** (solo lectura, registro de Cargo — no `O:\Lotte\references`):
- `bevy_animation-0.19.1/src/`: `lib.rs`(1785), `animatable.rs`(413), `animation_curves.rs`(845),
  `animation_event.rs`(62), `gltf_curves.rs`(409), `graph.rs`(954), `morph.rs`(227),
  `transition.rs`(159), `util.rs`(10). `wc -l *.rs` → **4864** líneas, coincide con el brief.
- `bevy_math-0.19.1/src/cubic_splines/` SOLAMENTE: `mod.rs`(1833), `curve_impls.rs`(159). El
  resto de `bevy_math` no se leyó.

---

## 1. `AnimationTargetId`: identidad de canales

`AnimationTargetId` es un `Uuid` en un tuple struct — y él mismo es el `#[derive(Component)]` que
se pone en la entidad (`lib.rs:183-187`):

```rust
#[derive(Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Reflect, Debug, Serialize, Deserialize, Component)]
pub struct AnimationTargetId(pub Uuid);
```

Construcción, `lib.rs:1311-1349` (`from_names`/`from_name` delegan en `from_iter`):

```rust
// lib.rs:1331-1342, impl<T: AsRef<str>> FromIterator<T> for AnimationTargetId
fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self {
    let mut blake3 = blake3::Hasher::new();
    blake3.update(ANIMATION_TARGET_NAMESPACE.as_bytes());
    for str in iter {
        // Include the string's length in the hash. Avoids ["ab"] == ["a","b"].
        blake3.update(&str.as_ref().len().to_le_bytes());
        blake3.update(str.as_ref().as_bytes());
    }
    let hash = blake3.finalize().as_bytes()[0..16].try_into().unwrap();
    Self(*uuid::Builder::from_sha1_bytes(hash).as_uuid())
}
```

`ANIMATION_TARGET_NAMESPACE` es una constante fija de 128 bits (`lib.rs:75`,
`Uuid::from_u128(0x3179f519d9274ff2b5966fd077023911)`). **No es UUID v5 canónico**: usan blake3
sobre namespace + longitud-y-bytes de cada segmento del path, truncan a 16 bytes, y los empaquetan
con `Builder::from_sha1_bytes` (que en la crate `uuid` es solo un constructor de bytes crudos, no
vuelve a correr SHA-1). El prefijo de longitud por segmento evita la colisión `["ab"]` vs
`["a","b"]` (comentario explícito en el código).

**Estabilidad**: determinístico entre ejecuciones y archivos — función pura de la lista de nombres.
`test_animation_target_id` (`lib.rs:1721-1762`) prueba 14 paths distintos (incluyendo variantes de
segmentación y reordenamientos) sin colisiones, y que `from_iter`≡`from_names`. Es solo función de
**nombres**, no de índices de jerarquía: cualquier armadura con un hueso raíz `Hips` obtiene el
mismo ID (retargeting intencional, `lib.rs:167-175`), pero es el **path completo**: un `Chest`
colgado de `Hips` difiere de un `Chest` colgado de `Stomach` (`lib.rs:177-180`).

**Asociación a una entidad**: no hay un componente separado `AnimationTarget { id, player }` en
esta versión — son **dos componentes**: `AnimationTargetId` mismo, y `AnimatedBy(pub Entity)`
apuntando a la entidad con el `AnimationPlayer` (`lib.rs:213-215`). El sistema de aplicación
consulta ambos juntos (`lib.rs:1089`: `Query<(Entity, &AnimationTargetId, &AnimatedBy,
AnimationEntityMut), Without<IsResource>>`). Bevy no fuerza parentesco target↔player; si se
agrega/borra un hueso en runtime hay que actualizar el componente a mano (`lib.rs:196-212`).

**Nombres duplicados / rutas que cambian**: comportamiento buscado, no manejado como error. Dos
entidades sibling con el mismo path de nombres reciben el mismo ID y por diseño (`lib.rs:101`:
"any entity with that ID") ambas quedan animadas por las mismas curvas de
`AnimationCurves: HashMap<AnimationTargetId, Vec<VariableCurve>, NoOpHash>` (`lib.rs:161`). No hay
detección de colisión; si el `Name` cambia en runtime el ID viejo queda huérfano.

---

## 2. `AnimationClip`: estructura y propiedades animables

```rust
// lib.rs:103-111
#[derive(Asset, Reflect, Clone, Debug, Default)]
pub struct AnimationClip {
    #[reflect(ignore, clone)]
    curves: AnimationCurves,   // HashMap<AnimationTargetId, Vec<VariableCurve>, NoOpHash>
    events: AnimationEvents,
    duration: f32,
}
```

- **Agrupación por target**: un `Vec<VariableCurve>` por target (`lib.rs:161`), no una curva por
  (target, propiedad); la desambiguación entre curvas del mismo target se hace comparando
  `evaluator_id()` en tiempo de evaluación (§4), no en la clave del mapa. `VariableCurve`
  (`lib.rs:81`) es `Box<dyn AnimationCurve>` — type-erasure sobre cualquier curva.
- **Propiedades arbitrarias, no solo transform**: mecanismo genérico `AnimatableCurve<P, C>`
  (`animation_curves.rs:288-297`) = selector de propiedad `P: AnimatableProperty` (típicamente
  `AnimatedField<C, A, F>`, construido con la macro `animated_field!(Componente::campo)` vía
  reflexión de `TypeInfo`, `animation_curves.rs:250-272`) + curva de valores `C: Curve<P::Property>`:
  ```rust
  // animation_curves.rs:190-203
  pub trait AnimatableProperty: Send + Sync + 'static {
      type Property: Animatable;
      fn get_mut<'a>(&self, entity: &'a mut AnimationEntityMut)
          -> Result<&'a mut Self::Property, AnimationEvaluationError>;
      fn evaluator_id(&self) -> EvaluatorId<'_>;
  }
  ```
  Cualquier componente `Mutable` con un campo `Reflect + Animatable` es animable
  (`animation_curves.rs:222-227`), no solo `Transform`. `morph.rs` es un segundo ejemplo real:
  pesos de morph target (`Vec<f32>` de tamaño variable) implementan `AnimationCurveEvaluator` a
  mano porque `Animatable` solo cubre tamaño fijo (`morph.rs:1-57`, `EvaluatorId::Type`).
- **Duración inferida, no declarada**: `add_curve_to_target` (`lib.rs:284-298`) extiende
  `self.duration` a `domain().end()` de la curva solo si es finito; `add_event_internal`
  (`lib.rs:446-452`) hace lo mismo con el tiempo del evento.
- **Eventos**: `AnimationEvents = HashMap<AnimationEventTarget, Vec<TimedAnimationEvent>>`
  (`lib.rs:157`), `AnimationEventTarget::{Root, Node(AnimationTargetId)}` (`lib.rs:150-155`),
  insertados ordenados por tiempo (`binary_search_by_key`, `lib.rs:454-459`). Dos APIs:
  `add_event`/`add_event_to_target` (dispara `EntityEvent` vía `Commands::trigger_with`,
  `lib.rs:326-354`) y `add_event_fn`/`add_event_fn_to_target` (closure cruda, `lib.rs:414-444`).
  Disparo real en `animate_targets` vía `TriggeredEvents::from_animation` (`lib.rs:1209-1227`).

---

## 3. Curvas y keyframes: representaciones e interpolación

**En `bevy_animation`** (wrappers sobre `Curve<T>` de `bevy_math`):
- **`AnimatableKeyframeCurve<T>`** (`animation_curves.rs:723-761`): genérica para cualquier
  `T: Animatable`, envuelve `UnevenCore<T>` (de `bevy_math::curve`, fuera de alcance) y delega en
  `T::interpolate` (`animation_curves.rs:737-740`) — la forma de interpolar la define el *tipo*
  animado (§4), no la curva.
- **`SteppedKeyframeCurve<T>`** (`gltf_curves.rs:13-47`): modo "step" de glTF, mantiene el valor
  anterior hasta `t=1.0` exacto del segmento (`gltf_curves.rs:26-30`).
- **`CubicKeyframeCurve<T>`** y `CubicRotationCurve` (para cuaterniones vía `Vec4`)
  (`gltf_curves.rs:51-172`): modo `CUBICSPLINE` de glTF — cada keyframe trae `(tangente_in, valor,
  tangente_out)` en un `ChunkedUnevenCore<T>` de ancho 3. Fórmula de Hermite cúbico escrita a mano,
  `gltf_curves.rs:374-390`:
  ```rust
  fn cubic_spline_interpolation<T: VectorSpace<Scalar = f32>>(
      value_start: T, tangent_out_start: T, tangent_in_end: T, value_end: T,
      lerp: f32, step_duration: f32,
  ) -> T {
      let coeffs = (vec4(2.0, 1.0, -2.0, 1.0) * lerp + vec4(-3.0, -2.0, 3.0, -1.0)) * lerp;
      value_start * (coeffs.x * lerp + 1.0)
          + tangent_out_start * step_duration * lerp * (coeffs.y + 1.0)
          + value_end * lerp * coeffs.z + tangent_in_end * step_duration * lerp * coeffs.w
  }
  ```
  Matriz característica `[[1,0,0,0],[0,1,0,0],[-3,-2,3,-1],[2,1,-2,1]]` — la misma que
  `CubicHermite` de `bevy_math`; escrita a mano porque acá `step_duration = t1-t0` cambia por tramo
  (keyframes no uniformes), a diferencia de `CubicSegment` que asume dominio `[0,1]` por segmento.

**En `bevy_math::cubic_splines`** (alcance auditado) — todas construyen un `CubicCurve<P>`
(`Vec<CubicSegment<P>>`, `mod.rs:1170-1173`) vía `CubicGenerator<P>::to_curve`
(`mod.rs:914-919`; `CyclicCubicGenerator` para cerradas, `mod.rs:926-932`):

| Constructor | Líneas | Notas |
|---|---|---|
| `CubicBezier<P>` | `mod.rs:55-93` | 4 puntos de control por segmento |
| `CubicHermite<P>` | `mod.rs:145-176` | pares `(posición, tangente)`; `char_matrix()` idéntica a la de `gltf_curves.rs:385` |
| `CubicCardinalSpline<P>` | `mod.rs:273-278` | tensión configurable; **"Catmull-Rom es un caso particular con tensión 0.5"** (`mod.rs:234`) — no hay tipo `CatmullRom` separado |
| `CubicBSpline<P>` | `mod.rs:435` | uniforme, **no interpolante** (no pasa por los puntos de control) |
| `CubicNurbs<P>` | `mod.rs:612` | vía `RationalGenerator` → `RationalCurve<P>`, no `CubicCurve` |
| `LinearSpline<P>` | `mod.rs:838` | lineal por tramos, como caso degenerado de curva cúbica |

`CubicSegment<P>` (`mod.rs:949-951`) guarda coeficientes `[a,b,c,d]` ya resueltos para
`a+bt+ct²+dt³`, evaluados con Horner (`mod.rs:957-977`) — la conversión puntos→polinomio
(`coefficients()`, `mod.rs:995-1006`) se paga una vez al construir, no por sample.

**`t` fuera de rango**: `CubicCurve::segment(t)` clampea el índice de segmento, no extrapola:
```rust
// mod.rs:1278-1284
fn segment(&self, t: f32) -> (&CubicSegment<P>, f32) {
    if self.segments.len() == 1 { (&self.segments[0], t) }
    else {
        let i = (ops::floor(t) as usize).clamp(0, self.segments.len() - 1);
        (&self.segments[i], t - i as f32)
    }
}
```
Junto con `domain()` (`curve_impls.rs:51-55`, `Interval::new(0.0, segments.len())`), el efecto es
**clamping a los extremos**, nunca extrapolación más allá del último segmento (salvo curva cíclica
explícita vía `to_curve_cyclic`).

**Fuera de la lista pedida pero relevante** — `CubicSegment<Vec2>::ease()` (`mod.rs:1053-1157`) es
un solver de easing Bézier por **Newton-Raphson**, muy cercano al `evaluate(t)` de `FCurve` en
Lotte:
```rust
// mod.rs:1140-1156 — MAX_ITERS=8 (mod.rs:1071), MAX_ERROR=1e-5 (mod.rs:1068)
fn find_y_given_x(&self, x: f32) -> f32 {
    let mut t_guess = x;
    let mut pos_guess = Vec2::ZERO;
    for _ in 0..Self::MAX_ITERS {
        pos_guess = self.position(t_guess);
        let error = pos_guess.x - x;
        if ops::abs(error) <= Self::MAX_ERROR { break; }
        let slope = self.velocity(t_guess).x;
        t_guess -= error / slope;
    }
    pos_guess.y
}
```
Sin manejo explícito de pendiente cero (si `slope`=0, `t_guess` se vuelve `NaN`/`inf` y el bucle
sigue hasta `MAX_ITERS` sin fallback a bisección).

---

## 4. Aplicación del valor: `Animatable` / `blend` / `interpolate`

```rust
// animatable.rs:19-30, con BlendInput<T> { weight: f32, value: T, additive: bool } (animatable.rs:9-17)
pub trait Animatable: Reflect + Sized + Send + Sync + 'static {
    fn interpolate(a: &Self, b: &Self, time: f32) -> Self;
    fn blend(inputs: impl Iterator<Item = BlendInput<Self>>) -> Self;
}
```

Implementaciones: flotantes/vectores vía macro `impl_float_animatable!` (`animatable.rs:32-55`,
lerp + fold aditivo/interpolado); `Vec3` especial-cased en `Vec3A`/SIMD (`animatable.rs:99-117`);
**`Quat`/`Rot2` usan slerp, no lerp**, comentario explícito "rather than using a quicker but less
correct linear interpolation" (`animatable.rs:172-198,200-226`), con `blend` aditivo componiendo
`slerp(IDENTITY, valor, peso) * acumulado`; `Transform` compone las tres por campo
(`animatable.rs:133-170`); `bool` interpola por step y blend por mayor peso (`animatable.rs:119-131`).

**Del stack al componente** (todo en `animation_curves.rs`):
1. **`apply`** (`:371-391`) samplea la curva en `t` y **empuja** a una pila
   (`BasicAnimationCurveEvaluatorStackElement<A>`, `:439-446`) con `weight` y nodo de grafo.
2. **`blend`/`add`** (`:399-406` → `combine`, `:464-518`): por nodo del grafo, hace *pop* del tope
   y combina con un **registro de blend** (`Option<(A, f32)>`) usando `A::interpolate` (reemplazo)
   o `A::blend` con `additive=true`.
3. **`commit`** (`:416-426`) obtiene `&mut A` vía `AnimatableProperty::get_mut` y **sobrescribe**
   con el único valor que quedó en la pila:
   ```rust
   fn commit(&mut self, mut entity: AnimationEntityMut) -> Result<(), AnimationEvaluationError> {
       let property = self.property.get_mut(&mut entity)?;
       *property = self.evaluator.stack.pop()...?.value;
       self.evaluator.stack.clear();
       Ok(())
   }
   ```

**No existe `post_process`** en este trait ni en ningún otro del crate (`grep -n "post_process"`
sin resultados) — dato del brief no encontrado.

**Punto clave para Lotte**: `commit` **reemplaza**, no compone con un "valor de reposo" leído
antes. Si nada anima un target en un frame, `apply` nunca se llama y el componente **conserva el
valor del frame anterior** — no vuelve a su reposo por sí solo. El reposo no es un concepto de
primera clase del motor de animación de Bevy: lo aporta quien puebla la escena. Ver "Portable" (b).

Mezcla aditiva vs reemplazo no son dos algoritmos: es un **flag por input** (`additive` en
`BlendInput`) interpretado por cada `blend`, decidido a nivel de grafo por el *tipo de nodo*
(`Blend` vs `Add`, §5; `animation_curves.rs:401` vs `:405`).

---

## 5. `AnimationGraph` / `AnimationPlayer`

```rust
// graph.rs:112-131
pub struct AnimationGraph {
    pub graph: AnimationDiGraph,   // petgraph::DiGraph<AnimationGraphNode, (), u32>
    pub root: NodeIndex,
    pub mask_groups: HashMap<AnimationTargetId, AnimationMask>,   // AnimationMask = u64 (graph.rs:426)
}
```
Tres tipos de nodo, `AnimationNodeType` (`graph.rs:211-238`): **`Clip`** (siempre hoja);
**`Blend`** (default) combina hijos por reemplazo ponderado, pesos normalizados a 1.0; **`Add`**
combina aditivamente sin normalizar (superponer p.ej. un ataque de brazo sobre correr). Cada
`AnimationGraphNode` (`graph.rs:169-205`) trae `weight: f32` que **no se propaga hacia abajo**
(`graph.rs:186-199`) — solo pesa el resultado ya combinado del nodo en el blending del *padre*
(como si el subárbol fuera un clip virtual).

**Máscaras** (`graph.rs:81-101,130,426`): bitfield de hasta 64 mask groups.
`mask_groups: HashMap<AnimationTargetId, AnimationMask>` asigna targets a grupos
(`add_target_to_mask_group`, `graph.rs:672-674`: `*entry(target).or_default() |= 1 << group`); cada
nodo trae su propia máscara de exclusión (`AnimationGraphNode.mask`, `graph.rs:178-184`, bit 1 =
deshabilita). Un target se anima solo si ninguno de sus grupos está enmascarado
(`graph.rs:92-93`). Caso de uso documentado: enmascarar la mano de un personaje al sostener un
objeto (`graph.rs:95-101`).

**Combinar varias animaciones sobre el mismo target**: el mecanismo de pila + registro de blend de
§4, recorriendo el grafo en **postorden precomputado** — `ThreadedAnimationGraph`
(`graph.rs:298-374`), hermanos en orden descendente para que la pila procese en orden ascendente
(traza paso a paso en `graph.rs:341-354`).

**`AnimationPlayer`** (`lib.rs:730-734`): `active_animations: HashMap<AnimationNodeIndex,
ActiveAnimation>`. Cada `ActiveAnimation` (`lib.rs:509-530`) trae `weight, repeat, speed, elapsed,
seek_time, last_seek_time, completions, just_completed, paused`.
- **Velocidad**: `speed` multiplica el delta (`lib.rs:572`); negativo = reversa (`lib.rs:656-658`).
- **Repetición**: `RepeatAnimation::{Never, Count(u32), Forever}` (`lib.rs:471-479`),
  `is_finished()` (`lib.rs:552-559`) gatea el avance.
- **Pausa**: `paused: bool`, `pause()`/`resume()` (`lib.rs:615-624`); el chequeo de disparo de
  eventos vive en el llamador (`lib.rs:1207`).
- **Wrap/finalización**: `update()` (`lib.rs:563-592`) hace módulo manual sobre `clip_duration`
  con rama separada para reversa; asume `delta` nunca menor que `-clip_duration` (`lib.rs:588`).
- **Seek**: `set_seek_time` (sin disparar eventos saltados) vs `seek_to`/`rewind` (sí los dispara
  en el próximo `update`) — `lib.rs:693-723`.

**Transiciones** (`transition.rs`): `AnimationTransitions` (`:33-36`), componente aparte que hace
fade-out lineal del `weight` de `ActiveAnimation`s salientes (`AnimationTransition`, `:56-63`). El
doc-comment del módulo lo marca **"unstable temporary API. It may be replaced by a state machine
in the future"** (`:1-4`).

---

## 6. Rendimiento y estructura

- **Sin hash por frame para localizar targets**: `animate_targets` (`lib.rs:1082-1274`) es una
  query de ECS directa y paralelizada (`par_iter_mut`, `lib.rs:1094`) sobre `(Entity,
  &AnimationTargetId, &AnimatedBy, AnimationEntityMut)` — el hash del path de nombres ocurre una
  sola vez, al construir el `AnimationTargetId` (asset loader), no en el loop de animación.
- **`NoOpHash`** para `AnimationCurves` (`lib.rs:44,161`): la clave ya es un hash bien distribuido
  (blake3 truncado), así que se evita re-hashear con SipHash; `Hash` de `AnimationTargetId`
  (`lib.rs:189-194`) solo xorea los dos `u64` del UUID.
- **Grafo pre-recorrido**: `ThreadedAnimationGraph` (`graph.rs:283-374`) cachea postorden, rangos
  de hijos y máscaras por `AssetId<AnimationGraph>` en `ThreadedAnimationGraphs`
  (`graph.rs:289-291`), recalculado solo cuando cambia el asset (`thread_animation_graphs`, corre
  `before(AssetEventSystems)`, `lib.rs:1291`) — no en cada frame.
- **Evaluadores de curva cacheados** frame a frame y target a target: `AnimationEvaluationState`
  (`lib.rs:751-772`) con `AnimationCurveEvaluators` (`lib.rs:775-813`, comentario explícito de
  cacheo en `lib.rs:758-761`), separado en `PreHashMap<(TypeId, usize), _>` para propiedades de
  componente y `TypeIdMap<_>` para curvas custom.
- **Coeficientes polinómicos precomputados** (§3): `CubicSegment` guarda `[P;4]` ya resuelto —
  cada sample es solo Horner de grado 3, no reconstrucción de matriz.

---

## Portable a Lotte

Mapeo a (a) identidad de canal, (b) reposo + animación(t), (c) blending por capas/pesos (M5).

### (a) Identidad de canal: ¿hash de ruta como Bevy, o ruta en texto?

**No copiar el hash blake3+UUID tal cual; sí copiar la idea de un ID derivado determinísticamente
de un path de nombres, separado del `NodeId` posicional del DAG.** El motivo de Bevy para un ID
opaco es retargeting entre armaduras + `HashMap` sin costo de comparar strings en el hot path
(`lib.rs:44,161,189-194`). El `NodeId(usize)` posicional de Lotte ya es barato de hashear/comparar
pero no es estable entre reordenamientos ni entre reimportaciones — el problema que Bevy resuelve.
El patrón de "hashear segmentos con separador de longitud explícito para evitar `["ab"]` ==
`["a","b"]`" (`lib.rs:1334-1338`) es chico y copiable con atribución MIT/Apache-2.0; no hace falta
blake3 específicamente, un FNV/xxhash de 64 bits sobre la misma construcción alcanza — el valor
está en la técnica, no en el hasher. **No conviene copiar** que nombres duplicados colisionen
silenciosamente (§1): con `contratos-solo-aditivos`, Lotte probablemente quiere que eso sea un
error de validación al cargar el proyecto, no un comportamiento silencioso. Concretamente: guardar
tanto el `NodeId` posicional (DAG en memoria) como un hash-de-ruta estilo Bevy calculado al
serializar, para que `lotte-timeline` (independiente de `lotte-core`) referencie canales sin
conocer el DAG.

### (b) ¿El modelo `Animatable`/`blend` sirve para reposo + animación(t)?

**Parcialmente: la interpolación/blend sí, el modelo de "reposo" no.** El trait
`Animatable::{interpolate, blend}` y sus implementaciones vectoriales/rotacionales
(`animatable.rs:19-226`) son casi textuales para `Transform2D` (`position, pivot, rotation, scale,
shear`): lerp para posición/escala/shear, y — punto importante, explícito en el comentario de
`animatable.rs:176-177` — **slerp, no lerp**, para rotación, incluso en 2D vía `Rot2::interpolate`.
Lotte usa `rotation: f32`; el análogo correcto es interpolación angular por camino más corto, no
lerp ingenuo del ángulo — vale revisar si el `evaluate()` de `FCurve` ya lo hace. No aprovechable
tal cual: `commit()` **sobrescribe** en vez de componer con un reposo leído antes
(`animation_curves.rs:416-426`); si nada anima un target, éste conserva su último valor aplicado,
no vuelve al reposo. El modelo de Lotte (`reposo + animación(t)`) es más explícito y más seguro en
este punto que el de Bevy, que delega el "volver al reposo" a la aplicación. `BlendInput<T> {
weight, value, additive }` es un buen molde: Lotte podría definir
`Animatable2D::blend(reposo: T, capas: impl Iterator<Item = BlendInput<T>>) -> T` con el reposo
como semilla, en vez de `Default::default()`.

### (c) ¿El grafo de blending de Bevy es lo que M5 necesita?

**Sí, en su forma general** — nodos Blend/Add con pesos, evaluados bottom-up sobre un DAG, es
prácticamente el problema de M5. El árbol de tres tipos de nodo con pesos que no se propagan hacia
abajo (`graph.rs:186-238`) es un buen punto de partida de *diseño* (no de código: depende de
`petgraph`/ECS que Lotte no necesita) — cada nodo blend es "como un clip virtual" para su padre. El
sistema de **máscaras por bitfield `u64`** (`graph.rs:81-101,426`) es simple, barato (un AND por
target por nodo) y portable casi literal a la jerarquía de `PegNode`. El patrón **pila + postorden
precomputado** (`animation_curves.rs:429-458`, `graph.rs:298-374`) es lo menos portable literal —
existe para evitar allocar por nodo por frame dentro de un `Query` paralelo de ECS; Lotte puede
lograr el mismo resultado con una recursión simple sobre el DAG de blend, y la ganancia de cacheo
de Bevy solo importa una vez que haya un loop de reproducción en tiempo real. Sí conviene copiar:
la separación entre peso normalizado (Blend) y no normalizado (Add) como propiedad del *tipo de
nodo*, evitando la ambigüedad de si un peso es fracción de 1.0 o multiplicador libre.
`AnimationTransitions` (fade-out lineal, `transition.rs`) es referencia de API, no contrato — el
propio crate la marca inestable (`:1-4`) — pero confirma que las transiciones se implementan
*encima* del grafo (ajustando `weight`), no como un cuarto tipo de nodo.

### Resumen

| Origen | Qué es | Portable |
|---|---|---|
| `animatable.rs:19-30` | Contrato `interpolate`/`blend` | Copiar el patrón, adaptado a `Transform2D` (atribución MIT/Apache-2.0) |
| `animatable.rs:172-198` | Slerp en `Quat`/`Rot2` | Copiar el principio: interpolación angular, no lerp de ángulo |
| `mod.rs:1140-1156` | Newton-Raphson easing Bézier | Muy cercano a `evaluate(t)` de `FCurve`; comparar tolerancia/iteraciones |
| `lib.rs:1326-1343` | Hash de path para ID de canal | Adaptar la técnica, no la dependencia blake3/uuid |
| `animation_curves.rs:416-426` | `commit` sobrescribe | No copiar tal cual — Lotte necesita reposo+animación |
| `graph.rs:112-238` | Grafo Blend/Add | Portable como diseño para M5, no como código |
| `graph.rs:81-101,426` | Máscaras `u64` | Portable casi literal como bitfield |
| `graph.rs:298-374`, `lib.rs:751-813` | Cachés de grafo/evaluadores | Relevante con loop de reproducción en tiempo real |
| `transition.rs` | Fade-out entre animaciones | Referencia de API, no contrato — el crate la marca inestable |

---

## Lo que NO existe o no encontré

- **`post_process`**: no existe en `AnimationCurveEvaluator` ni en ningún otro trait del crate
  (`grep -n "post_process"` sin resultados). Pipeline real: `apply → blend/add → commit` (§4).
- **Componente `AnimationTarget { id, player }`**: no existe en 0.19.1
  (`grep -n "struct AnimationTarget\b" lib.rs` sin resultados). La asociación target↔player son
  dos componentes: `AnimationTargetId` y `AnimatedBy(Entity)` (§1) — divergencia con el brief,
  documentada, no omisión de búsqueda.
- **Un tipo `CatmullRom` separado**: no existe; es `CubicCardinalSpline` con tensión 0.5
  (`mod.rs:234`).
- **Benchmarks o números de rendimiento medidos**: no se corrió `cargo bench` ni ninguna medición
  dinámica — auditoría estática (`grep`/lectura), consistente con el mandato "solo lectura". Las
  afirmaciones de §6 se basan en lo que el código estáticamente evita hacer, no en un benchmark.
- **El resto de `bevy_math`** (fuera de `cubic_splines/`): no leído, por alcance del brief. En
  particular `UnevenCore<T>`/`ChunkedUnevenCore<T>` y el trait `Curve<T>` (con su `sample_clamped`
  por defecto) viven en `bevy_math::curve`, citados porque aparecen en el código leído pero sin
  auditar su implementación interna.
- **`advance_animations`** (llama a `ActiveAnimation::update` antes de `animate_targets`):
  mencionado por nombre en `lib.rs:1293`, no leído línea por línea; `update()` en sí
  (`lib.rs:563-592`) sí se citó completo.

---

## Cómo se midió

```bash
# Licencia
cat ".../bevy_animation-0.19.1/Cargo.toml" | grep license
# -> license = "MIT OR Apache-2.0"

# Alcance leído
cd ".../bevy_animation-0.19.1/src/" && wc -l *.rs
# animatable.rs 413, animation_curves.rs 845, animation_event.rs 62, gltf_curves.rs 409,
# graph.rs 954, lib.rs 1785, morph.rs 227, transition.rs 159, util.rs 10 -> total 4864

cd ".../bevy_math-0.19.1/src/cubic_splines/" && wc -l *.rs
# curve_impls.rs 159, mod.rs 1833 -> total 1992

# Toda cita archivo:línea de este informe salió de grep -n sobre estos archivos, confirmada
# leyendo el rango exacto (nunca citada de memoria). Ejemplos representativos:
grep -n "struct AnimationTargetId\|from_names\|from_name\b" lib.rs
grep -n "post_process" animation_curves.rs   # 0 resultados -> confirma ausencia
grep -n "struct AnimationTarget\b" lib.rs    # 0 resultados -> confirma ausencia del componente separado
```

No se ejecutó `cargo` ni ningún binario en ningún momento de esta auditoría (prohibido por el
brief); todo lo anterior es lectura estática de fuente vía `grep`/lectura de archivo.
