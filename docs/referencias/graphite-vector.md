# Auditoría de Graphite — modelo vectorial, hit-testing, herramientas de path, render a vello

- **Repo fuente:** `references/Graphite/` (solo lectura, clon superficial)
- **Commit medido:** `e725f5504338703052cab25fa51aa4eda742fd42` (`git log -1`, 2026-09-06 23:57:37 -0700)
- **Licencia:** Apache-2.0 OR MIT (`LICENSE-APACHE`, `LICENSE-MIT` en la raíz del repo, "Copyright (c) Graphite contributors")
- **Fecha de la auditoría:** 2026-09-08
- **Estrategia de ingesta:** se porta con atribución Apache-2.0/MIT; mismo stack, casi directo
  (Lotte usa vello 0.10 + kurbo 0.13 + peniko 0.6, el mismo stack que trae Graphite en este commit).

Nota de corrección sobre el brief: el archivo no es `click_target.rs` a secas sino
`node-graph/libraries/vector-types/src/vector/click_target.rs` (495 líneas), y el rango de
hit-testing real no es 197-314 sino 197-328 (ver sección 1). El resto de rutas del brief
(`node-graph/gcore/src/vector/`, `editor/src/messages/tool/tool_messages/`) tampoco son del
todo exactas: `gcore` no existe como tal — el crate de tipos vectoriales es
`node-graph/libraries/vector-types/src/vector/`; la ruta de las tool messages sí es correcta.

---

## 1. Hit-testing de curvas

Archivo real: `node-graph/libraries/vector-types/src/vector/click_target.rs`.

### Estructura y bounding box

`ClickTarget` (líneas 150-156) guarda un `ClickTargetType` (enum `Path(BezPath)` o
`FreePoint(FreePoint)`, líneas 61-66), un `stroke_width: f64` que hace de tolerancia, un
`bounding_box: Option<[DVec2; 2]>` cacheado, y un `bounding_box_cache: Arc<RwLock<BoundingBoxCache>>`.

El bounding box laxo inicial es el **control-box** de la curva (el hull de los puntos de
control, no el bounding box ajustado a la curva real): `ClickTarget::new_with_path`
(líneas 165-175) usa `path.control_box()` de kurbo directamente — es O(puntos), sin resolver
ninguna ecuación de Bézier.

El bounding box *ajustado* (tight) se computa en `bezpath_bounding_box_with_transform`
(líneas 15-22): itera `bezpath.segments()`, aplica el transform a cada segmento y llama a
`.bounding_box()` de kurbo (que sí resuelve el extremo real de la curva), reduciendo con
`.union()`. Este es el que cachea `BoundingBoxCache`.

### Cómo lo hace rápido: `BoundingBoxCache`

Líneas 68-145. Es un ring buffer fijo de 8 entradas (`CACHE_SIZE = 8`, línea 85) que cachea
pares `(rotation_angle, bounding_box)` para evitar recalcular el bounding box de la curva bajo
rotación (`bezpath_bounding_box_with_transform` es el costo caro: recorre todos los segmentos).
Escala y traslación se aplican después sobre el resultado cacheado (líneas 110-111,
142-143) sin recomputar la curva, porque son transformaciones baratas sobre dos puntos.
El lookup usa un fingerprint de 7 bits del ángulo de rotación (`rotation_fingerprint`,
líneas 90-92, tomado de `rotation.to_bits() % 128`) empaquetado en un `u64` (8 bytes = 8
fingerprints), con el bit más significativo de cada byte como flag de presencia — permite
descartar candidatos con una comparación de bitmask (líneas 96-103) antes de comparar
`f64` exactos. Transforms con *skew* bypasean el cache entero (`has_skew()`, línea 209) porque
la descomposición rotación/escala/traslación no es válida ahí.

### Cómo decide si un punto cae sobre un trazo

- `intersect_point` (líneas 296-311): infla el punto en un cuadrado de lado `stroke_width`
  (línea 297), primero descarta con el bounding box laxo transformado (líneas 300-305, "no muy
  preciso pero rápido" según el propio comentario), y si no descarta, llama a `intersect_path`
  pasándole las 4 líneas del cuadrado inflado como iterador de segmentos (línea 310). Es decir:
  **no hay distancia punto-a-Bézier real** — el comentario en la línea 308 lo dice explícito:
  `// TODO: actual intersection of stroke`. La "tolerancia" es literalmente el ancho de trazo
  (`stroke_width`), no una constante de picking fija.
- `intersect_path` (líneas 261-293): invierte el `layer_transform` (con guarda para matrices no
  invertibles, líneas 264-266) y prueba tres cosas en orden barato→caro:
  1. intersección de outline: `filtered_segment_intersections` segmento a segmento
     (línea 274, no vacío → true);
  2. si el trazo no intersectó, si el primer punto del segmento cae dentro del área de relleno
     usando solo los contornos **cerrados** (`closed_contours`, líneas 24-40, regla non-zero vía
     `BezPath::contains` de kurbo, línea 282);
  3. si nada de eso, arma un path cerrado con toda la selección y comprueba si la forma entera
     queda contenida dentro (`bezpath_is_inside_bezpath`, línea 289) — para selección por caja/lazo.
- `intersect_point_no_stroke` (líneas 314-328): variante sin tolerancia de trazo, usada para UI
  (botones, nombres de capa) — chequea bounding box AABB primero, después
  `closed_contours(path).contains(point)`.

### Cómo se usa para una capa vectorial real (tolerancia = ancho de trazo)

`node-graph/libraries/rendering/src/renderer.rs:2223-2258`, función `extend_targets_from_vector`.
El `stroke_width` que se le pasa a `ClickTarget::new_with_path` (línea 2255) no es una constante:
sale de `appearance.first_coverage_of(Cover::Stroke)` (línea 2232) y es `stroke.weight * 2.` si
el alineado no es centrado y todos los contornos están cerrados, o `stroke.weight` si no
(líneas 2233-2238) — así una capa sin trazo (peso 0) solo es clickeable por relleno.

---

## 2. Modelo de documento vectorial

Archivo: `node-graph/libraries/vector-types/src/vector/vector_types.rs` (struct `Vector`,
líneas 14-24). **No se llama `VectorData`** en este commit — el brief lo buscaba con ese nombre;
el tipo actual es `Vector` (el comentario de la línea 19 conserva el nombre viejo
`colinear_manipulators` como legado, y referencia a `graph_operation_message_handler.rs`
buscando el string `"Shape does not have both..."`, que ya no es literal en ese archivo con
ese commit — puede ser un comentario desactualizado del propio Graphite).

```rust
pub struct Vector {
    pub colinear_manipulators: Vec<[HandleId; 2]>,
    pub point_domain: PointDomain,
    pub segment_domain: SegmentDomain,
}
```

No tiene campos de `fill` ni `stroke` — ver más abajo, "relación con relleno/trazo".

### Identidad estable: ids numéricos, no strings

`node-graph/libraries/vector-types/src/vector/vector_attributes.rs:11-51`, macro `create_ids!`:

```rust
macro_rules! create_ids {
    ($($id:ident),*) => {
        $(
            #[derive(Clone, Copy, ... Hash, graphene_hash::CacheHash, DynAny)]
            pub struct $id(u64);
            impl $id {
                pub const ZERO: $id = $id(0);
                pub fn generate() -> Self { Self(core_types::uuid::generate_uuid()) }
                pub fn generate_from_hash(self, node_id: u64) -> Self { /* hash-derivado */ }
                pub fn next_id(&mut self) -> Self { self.0 += 1; *self }
            }
        )*
    };
}
create_ids! { PointId, SegmentId }
```

`PointId` y `SegmentId` son newtypes de `u64` — **numérico, no `String`**. Dos vías de generación:
`generate()` (UUID aleatorio, creación interactiva) o `generate_from_hash(node_id)` (hash
determinístico, para reproducibilidad en el grafo de nodos). `HandleId` (`misc.rs:545-551`) no
tiene generador propio: es un compuesto `{ ty: HandleType::Primary|End, segment: SegmentId }` —
la identidad de una manija es "la manija primaria/final de tal segmento", no un id independiente.
**No existe `StrokeId`** (`grep -rn "StrokeId"` sobre `vector-types/src`, sin resultados) — el
trazo no tiene identidad propia por subpath.

### Estructura de datos: Structure-of-Arrays, no árbol de objetos

`PointDomain` (`vector_attributes.rs:84-88`) y `SegmentDomain` (líneas 206-212) son SoA: vectores
paralelos.

```rust
pub struct PointDomain {
    id: Vec<PointId>,
    pub(crate) position: Vec<DVec2>,
}
pub struct SegmentDomain {
    id: Vec<SegmentId>,
    start_point: Vec<usize>,  // índice en PointDomain, no PointId
    end_point: Vec<usize>,
    handles: Vec<BezierHandles>,
}
```

Los segmentos referencian sus puntos extremos por **índice de array** (`usize`), no por
`PointId` directamente. La traducción id→índice es `PointDomain::resolve_id`
(`vector_attributes.rs:170-172`): `self.id.iter().position(|&check_id| check_id == id)` — scan
lineal O(n), no `HashMap`. Elección deliberada de layout-para-cache (arrays contiguos, baratos de
transformar en bloque) a costa del lookup por id; no hay índice auxiliar id→posición.

### Relleno y trazo: no van por subpath, van por capa completa

`Vector` no tiene campos de estilo. El fill/stroke vive en una capa de "appearance" separada del
crate `graphic-types`: `node-graph/libraries/graphic-types/src/appearance.rs:260-267`:

```rust
pub struct FillAndStroke<'a> {
    pub stroke: Option<Stroke>,
    pub fill_paint: Option<&'a Graphic>,
    pub stroke_paint: Option<&'a Graphic>,
    pub stroke_below: bool,
}
```

Y en `renderer.rs:2223-2239` (`extend_targets_from_vector`), el fill/stroke de una capa vectorial
sale de `appearance.first_coverage_of(Cover::Fill/Stroke)` — **un solo fill y un solo stroke por
capa (`Vector` completo), no uno por subpath**. Todos los subpaths de un `Vector` comparten el
mismo par fill/stroke; lo que varía por subpath es solo la geometría (abierto/cerrado, ver
`stroke_bezpath_iter`). La excepción es el caso de mesh (ver abajo): ahí el relleno se calcula
por cara (`construct_faces`), pero sigue siendo un solo `Fill` aplicado a todas las caras.

### Autoridad del dibujo: quién decide qué es un subpath

`build_stroke_path_iter` (`vector_attributes.rs:871-884`) recorre `segment_domain` construyendo,
para cada punto, qué segmentos entran y salen (`StrokePathIterPointMetadata`), y arma cadenas
conectadas. `stroke_bezpath_iter` (líneas 892-896) mapea cada cadena a un `kurbo::BezPath` vía
`bezpath_from_manipulator_groups` (`misc.rs:262`). Es decir: **la autoridad de "qué es una
subruta" no es una lista explícita de subpaths guardada en el documento — se deriva on-demand de
la topología de `segment_domain`** (qué segmentos comparten puntos). Esto habilita un caso que
Lotte hoy no tiene: vectores "ramificados" (`is_branching`, líneas 917-927 — cualquier punto con
más de 2 segmentos conectados) tratados como mallas (`use_face_fill`, líneas 929-933,
`construct_faces`, línea 939+), donde el relleno se calcula por cara delimitada en vez de por
subpath cerrado simple.

---

## 3. Herramientas de edición de path

Rutas confirmadas con `Glob "**/tool_messages/*.rs"`: `editor/src/messages/tool/tool_messages/`
existe tal cual el brief. `pen_tool.rs` (2619 líneas) y `path_tool.rs` (3663 líneas).

### Pen tool: máquina de estados

`pen_tool.rs:107-113`:

```rust
enum PenToolFsmState {
    Ready,
    DraggingHandle(HandleMode),
    PlacingAnchor,
    GRSHandle,
}
```

`HandleMode` (líneas 341-350) — cómo se comportan las manijas al arrastrar:

```rust
enum HandleMode {
    Free,               // 'C' rompe la colinealidad
    ColinearLocked,     // default; 'Alt' bloquea el largo de la manija
    ColinearEquidistant,// 'Alt' hace las manijas equidistantes
}
```

La colinealidad efectiva no es un booleano aparte por manija: se decide comparando el ángulo
entre la manija entrante y saliente contra π (líneas 811, 837: `(angle - pi).abs() < 1e-6`), y se
aplica al modelo vía un mensaje de modificación explícito
`VectorModificationType::SetG1Continuous { handles, enabled: colinear }` (líneas 814, 840) — el
par de manijas afectado es `colinear_manipulators: Vec<[HandleId; 2]>` en `Vector` (sección 2).
`apply_colinear_constraint` (línea 1167) es la función que, dado un ancla, fuerza que la manija
opuesta quede alineada 180° cuando el modo lo pide.

`TargetHandle` (líneas 356-389) modela cuál manija está bajo el cursor con 5 variantes
(`None`, `FuturePreviewOutHandle`, `PreviewInHandle`, `PriorOutHandle(SegmentId)`,
`PriorInHandle(SegmentId)`) — la manija de vista previa del próximo segmento vs. las manijas de
segmentos ya colocados, distinguidas por a qué `SegmentId` pertenecen.

### Path tool: máquina de estados

`path_tool.rs:544-553`:

```rust
enum PathToolFsmState {
    Ready,
    Dragging(DraggingState),
    Drawing { selection_shape: SelectionShapeType },
    SlidingPoint,
}
```

### Snapping

Ambas tools usan el mismo mecanismo, importado de
`crate::messages::tool::common_functionality::snapping` (`pen_tool.rs:16`, `path_tool.rs:24`):
`SnapManager`, `SnapCache`, `SnapCandidatePoint`, `SnapConstraint`, `SnapData`. Cada tool guarda
su propio `snap_manager: SnapManager` en su struct de datos (`pen_tool.rs:393`,
`path_tool.rs:557`) — no es un singleton global, vive con el estado de la tool y se limpia
explícitamente (`self.snap_manager.cleanup(responses)`, `pen_tool.rs:465`). La resolución real
por evento está en `compute_snapped_angle` (`pen_tool.rs:1220-...`), que llama a
`self.snap_manager.free_snap(...)` (línea 1301, 1458) contra `SnapData` construido del documento
+ input + viewport. La implementación de `free_snap` en sí vive en
`editor/src/messages/tool/common_functionality/snapping/layer_snapper.rs:282` (hay snappers
separados: `layer_snapper.rs`, `alignment_snapper.rs`, `distribution_snapper.rs`,
`grid_snapper.rs`, cada uno con su propio `free_snap`, combinados por `SnapManager`).

---

## 4. Estilo de trazo

Archivo: `node-graph/libraries/vector-types/src/vector/style.rs`.

```rust
pub struct Stroke {                     // línea 225
    pub weight: f64,                    // línea 227 — un solo ancho para todo el trazo
    pub dash_lengths: Vec<f64>,         // línea 228
    pub dash_offset: f64,               // línea 229
    pub cap: StrokeCap,                 // línea 231 (enum en línea 90: Butt/Square/Round)
    pub join: StrokeJoin,               // línea 232 (enum en línea 115: Miter/Bevel/Round, con join_miter_limit)
    pub join_miter_limit: f64,
    pub align: StrokeAlign,             // Center/Inside/Outside
    pub transform: DAffine2,
}
```

`weight: f64` es un **único escalar por trazo completo** — no hay grosor variable a lo largo del
trazo (ni un campo por punto, ni una curva de ancho, ni referencia a presión de ningún tipo).
Confirmado negativo: sin más resultados de "width" variable en `style.rs` ni en `vector_types.rs`
más allá de este campo único y `effective_width()` (línea 298), que solo aplica el multiplicador
de `align` (Center=1×, Inside=0×, Outside=2×) — sigue siendo un escalar, no una función del
parámetro de la curva. **Grosor variable por presión: no encontrado en este repo.**

`FillChoice<C>` (`style.rs:24-28`): `None | Solid(C) | Gradient(GradientRamp<C>)`, genérico sobre
el formato de color (`Color` en memoria del editor, `SRGBA8` en el borde hacia JS/UI).

---

## 5. Despacho a vello

Archivo: `node-graph/libraries/rendering/src/renderer.rs`.

No existe un `to_bezpath()` con ese nombre exacto (`grep -n "fn to_bezpath"` sobre el repo: sin
resultados). La conversión real es la cadena descrita en la sección 2:
`Vector::stroke_bezpath_iter()` (`vector_attributes.rs:893`) → por cada subpath detectado,
`bezpath_from_manipulator_groups()` (`misc.rs:262`) construye el `kurbo::BezPath` con
`MoveTo`/`CurveTo`/`ClosePath`.

El despacho a la escena de vello ocurre en dos closures dentro de la función de render de
capas vectoriales:

- `do_fill_path` (líneas 1811-1859): para relleno sólido, `scene.fill(fill_rule,
  Affine, &peniko::Brush::Solid(color), None, path)` (línea 1818); para degradé,
  pasa un `brush_transform` (línea 1831); para cualquier otro tipo de pintura
  (`Graphic::Vector`, `Graphic::Text`, listas, etc.) hace `push_clip_layer` + render recursivo +
  `pop_layer` (líneas 1854-1856) en vez de un brush directo.
- `do_stroke` (líneas 1877-1930ish): arma un `kurbo::Stroke` desde el `Stroke` de Graphite
  (líneas 1892-1900):

```rust
let stroke = kurbo::Stroke {
    width: stroke.weight * width_scale,
    miter_limit: stroke.join_miter_limit,
    join,                    // mapeado 1:1 desde StrokeJoin
    start_cap: cap, end_cap: cap,  // mismo cap en ambos extremos — Graphite no distingue start/end cap
    dash_pattern,             // Vec<f64> con cada largo clampeado a >= 0.
    dash_offset: stroke.dash_offset,
};
```

y llama `scene.stroke(&stroke, Affine, &brush, None, &path)` (línea 1911) o con
`brush_transform` para degradé (línea 1924). No hay una llamada literal `Scene::append` en este
archivo (`grep -n "Scene::append"` sin resultados) — el patrón de Graphite es `scene.fill(...)` /
`scene.stroke(...)` / `scene.push_layer(...)` / `scene.push_clip_layer(...)` directamente sobre
la instancia mutable, no un método estático `append`. `peniko::Fill::NonZero` es la regla de
relleno usada en todos los `scene.fill(...)` de este archivo (ningún `EvenOdd` encontrado en las
llamadas de fill de capas vectoriales).

---

## Portable a Lotte

Contexto: Lotte hoy tiene `VectorItem { peg_id, path: BezPath, fill_color, stroke_color,
stroke_width }` — sin ids de trazo/segmento/punto, sin herramientas interactivas.

**Portar casi tal cual (con atribución Apache-2.0/MIT):**

- Conversión a `kurbo::Stroke` + despacho `scene.fill`/`scene.stroke` (renderer.rs:1892-1924):
  mismo stack (vello 0.10 + kurbo 0.13 + peniko 0.6), mapeo directo `StrokeCap`/`StrokeJoin` →
  `kurbo::Cap`/`kurbo::Join`.
- Estrategia de ids: **numérico (`u64` newtype vía macro `create_ids!`, 11 líneas), no `String`**
  — más liviano para serializar/hashear que un `String`. Recomendado si Lotte necesita identidad
  estable de puntos/segmentos en `lotte-format`/`lotte-rig` (p. ej. animar vértices de una
  máscara). `generate_from_hash` (id determinístico desde un nodo) no aplica hoy: Lotte no tiene
  grafo de nodos.
- `BoundingBoxCache` (click_target.rs:68-145): útil para hit-testing rápido bajo rotación
  repetida, pero es complejidad no trivial (ring buffer + fingerprint) — portar solo si el
  profiling (`medir-antes-de-razonar`) muestra que el bounding box sin cachear es el cuello de
  botella.
- Tolerancia de hit-test = ancho de trazo real (renderer.rs:2232-2239), no una constante de
  picking fija.

**Adaptar (la idea sirve, el código no aplica directo):**

- El SoA (`PointDomain`/`SegmentDomain` con índices `usize`) es correcto si Lotte necesita
  mallas/skinning con muchos puntos (ya es el patrón de `lotte-rig`), pero el lookup id→índice es
  O(n) (`resolve_id`, scan lineal) — una tool de path interactiva que resuelve ids todo el tiempo
  (sección 3) debería agregar un `HashMap<PointId, usize>` auxiliar que Graphite no tiene.
- Fill/stroke por capa completa, no por subpath: si Lotte quiere lo mismo (como ya tiene con
  `fill_color`/`stroke_color` planos en `VectorItem`), no hay código que portar, solo la
  confirmación de que es un patrón razonable y probado.
- El FSM de pen/path tool (`PenToolFsmState`, `PathToolFsmState`, `HandleMode`) es vocabulario de
  diseño para cuando Lotte tenga una UI de edición de path — hoy no existe ningún crate de UI
  (`docs/ui-widgets.md` está "todavía sin crate"), así que es referencia futura, no código a
  copiar ahora.
- El `SnapManager` con snappers separados (layer/alignment/distribution/grid) es más de lo que
  Lotte necesita sin UI; el concepto "snap por tipo, combinado por un
  manager" es la parte reusable si/cuando Lotte construya su propio editor interactivo.

**No portar:**

- El soporte de mallas/topología ramificada (`is_branching`, `construct_faces`,
  `use_face_fill`) — Lotte es cutout 2D (piezas planas con pegs, no edición de mallas
  arbitrarias); es la parte más compleja del modelo de Graphite y no tiene contraparte en el PRD
  de Lotte.
- El grafo de nodos (`Graphic`, `Appearance`, `ATTR_*` cells) — Graphite es un editor de nodo-grafo
  general; Lotte no tiene ni planea un grafo de nodos genérico (ver capas de dependencia en
  `AGENTS.md`).

---

## Lo que NO existe o no encontré

- **`StrokeId`**: no encontrado (`grep -rn "StrokeId" node-graph/libraries/vector-types/src` → sin
  resultados). El trazo no tiene id propio; es un atributo de capa, no de geometría.
- **Grosor de trazo variable a lo largo del segmento (por presión o de otro tipo)**: no
  encontrado. `Stroke::weight` es `f64` único (style.rs:227); no hay campo de ancho por punto ni
  curva de ancho en `style.rs` ni en `vector_types.rs`.
- **`Scene::append` como llamada literal**: no encontrado en `renderer.rs`
  (`grep -n "Scene::append"` sin resultados) — el código usa `scene.fill`/`scene.stroke`/
  `scene.push_layer` sobre la instancia.
- **Función `to_bezpath()` con ese nombre**: no encontrada. La función real es
  `Vector::stroke_bezpath_iter()` + `bezpath_from_manipulator_groups()`.
- **`node-graph/gcore/src/vector/` como ruta**: no existe con ese nombre en este commit; los tipos
  vectoriales viven en `node-graph/libraries/vector-types/src/vector/`.
- **`VectorData` como nombre de tipo**: no existe en este commit; el tipo se llama `Vector`. Un
  comentario interno de Graphite (vector_types.rs:19) referencia un string de error en
  `graph_operation_message_handler.rs` que no verifiqué carácter por carácter — puede estar
  desactualizado dentro del propio Graphite; no lo cito como hecho, solo lo señalo.

---

## Cómo se midió

Todos los comandos corridos sobre `references/Graphite/` (solo lectura, sin `cargo`):

```bash
git log -1 --format="%H %ai"
# e725f5504338703052cab25fa51aa4eda742fd42 2026-09-06 23:57:37 -0700

# Ubicación real del archivo de hit-testing (el brief asumía una ruta distinta)
Glob "**/click_target.rs"
# -> node-graph/libraries/vector-types/src/vector/click_target.rs

wc -l node-graph/libraries/vector-types/src/vector/click_target.rs
# -> 495

grep -n "^pub fn|^    pub fn|^impl|^pub struct|^struct" .../click_target.rs
# -> localiza FreePoint:44, BoundingBoxCache:74/83, ClickTarget:150/158/164

Glob "**/tool_messages/*.rs"
# -> confirma pen_tool.rs, path_tool.rs y el resto en editor/src/messages/tool/tool_messages/

grep -n "struct VectorData|struct PointId|struct SegmentId|struct StrokeId|struct HandleId" \
  node-graph/libraries/vector-types/src
# -> solo HandleId con match directo; PointId/SegmentId salen de la macro create_ids! (grep aparte)

grep -n "^pub struct|^pub enum|^pub type" .../vector_types.rs
# -> pub struct Vector (línea 17), no VectorData

grep -n "struct PointId|struct SegmentId|impl PointId|impl SegmentId|fn next_id|fn generate|
  struct PointDomain|struct SegmentDomain|pub fn push" .../vector_attributes.rs

grep -rn "StrokeId" node-graph/libraries/vector-types/src
# -> sin resultados

grep -n "pub struct Fill|pub fill:|pub stroke:" node-graph  # (recursivo)
# -> graphic-types/src/appearance.rs:262 pub stroke: Option<Stroke> (dentro de FillAndStroke)

grep -n "enum PenToolFsmState|enum PathToolFsmState|SnapManager|struct PathToolData" \
  editor/src/messages/tool/tool_messages/{pen_tool,path_tool}.rs

grep -n "pub struct Stroke|pub enum StrokeCap|pub enum StrokeJoin|pub enum FillChoice|
  dash_lengths: Vec|pub weight: f64|fn effective_width" .../style.rs

grep -n "Scene::append|scene.append|.fill(|.stroke(|fn to_bezpath|to_bezpath(" \
  -r (recursivo sobre el repo, filtrado a *.rs vía Grep tool, excluyendo frontend/ y desktop/)
# -> sin match de "Scene::append" ni "fn to_bezpath" en ningún archivo *.rs relevante

grep -n "ClickTarget::new_with_path" -r node-graph
# -> localiza renderer.rs:2255 (stroke_width real de la capa, no una constante)
```

Alcance excluido según el brief: `frontend/` (Svelte/TypeScript) y `desktop/` (CEF) no se leyeron.
