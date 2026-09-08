# Hoja de ruta

**Estado: decisiones cerradas; implementación por hitos.** Tres premisas fijadas por la etapa de
verificación previa:

1. **El stack está verificado en hardware**, no solo resuelto en el lock —
   [`stack-verificado.md`](stack-verificado.md). Vello, wgpu, winit, egui como biblioteca, egui_tiles
   y la presión de la Wacom corren juntos en un proceso, con los cinco crates del motor adentro.
2. **eframe queda descartado y el bucle de eventos es nuestro.** Es la única forma de bombear la
   tableta fuera del WndProc. El prototipo de laboratorio ya es, de hecho, la primera aplicación.
3. **El diseño se apoya en proyectos de licencia permisiva leídos en código**, un informe por
   proyecto en [`referencias/`](referencias/), con cada cita verificada por muestreo.

Lo que sigue son los lotes ordenados por dependencia, cada uno con issues con criterio medible,
más las decisiones ya tomadas que los abren.

---

## Decisiones que abren o cierran lotes

Ninguna la toma un developer. Están numeradas para citarlas desde los issues.

**Estado (2026-09-08): todas decididas** — el acta, con las mediciones y las palabras del dueño, está en
[`decisiones-modelo-de-datos.md`](decisiones-modelo-de-datos.md). La columna "Recomendación medida"
se conserva como registro; donde difiere del acta (D1), gana el acta.

| # | Decisión | Recomendación medida | Bloquea |
|---|---|---|---|
| **D1** | Signo de `z` | **Decidido**: mayor `z` = más cerca, arte en `z ≈ 0`, cámara en `z = D`, validación `z < D`. Evidencia (`tstageobject.cpp:1945-1968`): OpenToonz hace `dz = focus + cameraZ − objectZ`, escala `(focus+cameraZ)/dz` — a mayor Z, más cerca; es también la convención de los animadores de recorte. No copiar su "maintain size" como canal manual (`m_noScaleZ`): la compensación derivada de la spec es mejor | Lote B |
| **D2** | ¿`lotte-timeline` depende de `lotte-core`? (plan, decisión 2) | No; `PoseSpace` vive en `lotte-rig` | Lote D |
| **D3** | Autoridad del modelo de dibujo e ids (plan, decisión 3) | El dibujo vive en `lotte-rig`. **Ids: la auditoría de Graphite (verificada) muestra que eligió `u64` numéricos generados por macro (`PointId`, `SegmentId`), sin `StrokeId`, y relleno/trazo como `Appearance` aparte, no por subruta.** La recomendación previa de newtypes `String` queda en duda: `u64` es lo que ya funciona en el único referente Rust y es más barato de serializar. Leer [`referencias/graphite-vector.md`](referencias/graphite-vector.md) §2 antes de decidir. Además: Graphite **no tiene grosor variable** — `patterns.md` §6 se construye desde cero | Lote C |
| **D-F** | Convención de ejes ([`stack-verificado.md`](stack-verificado.md) §5.F) | Mundo Y-arriba; la cámara voltea; test de orientación; fijarlo en `architecture.md` | Lote A (issue A2) |
| **D-App** | Nombre y alcance del sexto crate | `lotte-app`, dueño del bucle winit, depende de los cinco. Cambia el hecho "no hay aplicación" de `AGENTS.md` | Lote A |
| **D-Lic** | Archivos `LICENSE-MIT`, `LICENSE-APACHE`, `NOTICE` (plan, decisión 5) | Escribirlos ya: el `NOTICE` va a recibir atribuciones desde el primer port (Godot, DragonBones, Graphite) | Lote A (A6) |
| **D-Inst** | Modelo **definición / instancia / animación** ([`requisitos-animador.md`](requisitos-animador.md)) | Adoptarlo en `lotte-core`/`lotte-rig` **antes** del lote C: `RigDefinition` (reposo + arreglos), `RigInstance` (referencia a una definición), `Animation` (canales `(ruta, atributo)` de la instancia), valor efectivo `reposo ⊕ animación(t)`. Habilita copiar animación entre escenas, reset sin perder arreglos, y clonar vs duplicar — los cuatro requisitos del animador. Evidencia: Godot modela el reposo como *bind pose* (`Bone2D.rest`, `final = acumulada × inversa(rest)`, `skeleton_2d.cpp:611`) y una animación `RESET` como línea base del blending; Bevy **no** lo tiene (`commit` sobrescribe, sin `post_process`) y su propio informe recomienda que el blend de Lotte tome el reposo como semilla | Lotes C, D y E |
| **D-T** | **Unidad de tiempo** | Hoy es ambigua: `FCurve.time: f64` dice "frame index or seconds" en su doc, `ExposureTrack` usa `frame: u32`, `LotteProject` tiene `fps` y `duration_frames`. Recomendación: **el tiempo se mide en frames del proyecto**, como `f64` en las curvas (permite sub-frame para *motion blur* y *time remap*) y `u32` en la exposición; el `fps` convierte a segundos solo en la frontera (audio, exportación). Es lo que hace OpenToonz (celdas por frame) y lo que el pipeline de producción del estudio ya usa (`timing.frames`, `frameStart`, `fps` aparte). Fijarlo por escrito en `architecture.md` y en el doc de `Keyframe.time` | Lotes D y E |
| **D-Fmt** | Forma del documento en disco | **Decidido**: un plano es un **directorio** — `scene.json` (instancias, timing, cámara; chico), `anim/<instancia>.anim.json` + `.anim.safetensors` (una animación **por instancia**: unidad de copiar/pegar, reset y trabajo en paralelo), biblioteca de definiciones aparte. Patrón de glTF (estructura en JSON, números en buffers) y Godot (`Animation` como recurso externo). | Lote E (E7, E8) |
| **D-Tab** | Formato de las **tablas de tiempo** | **Decidido**: diccionario JSON de canales `(ruta, atributo)` + tensores en **`safetensors`** (`safetensors 0.8.0`, Apache-2.0, deps `serde`+`serde_json`): `keys F64 [N,6]`, `key_interp U8 [N]`, `channel_offsets U32 [C+1]`, `exposure_runs U32 [R,3]`, `exposure_offsets U32 [X+1]` — *ragged por offsets*, pocos tensores grandes, `mmap` zero-copy. Medido: una clave Bézier en JSON compacto ~95 bytes vs 24/48 como `f32`/`f64`. `F32` vs `F64` se decide con el fixture de E8 | Lote E (E8) |
| **D-Draw** | Forma del archivo de **dibujo** | **Decidido**: un archivo por dibujo, **SVG estándar + namespace `lotte:`** (línea central y perfil de grosor del `Stroke`, capas de arte, ids, paleta); el `<path>` lleva el contorno resuelto como *fallback* para cualquier visor. Extensibilidad por namespaces prevista por la propia especificación SVG; el dibujo queda fuera de la escena | Lotes C (C4) y E (E5, E6) |
| **D-Undo** | Modelo de **deshacer** | **Decidido** ([`decisiones-modelo-de-datos.md`](decisiones-modelo-de-datos.md)): `Edición` con inverso producida por la herramienta; pila única por plano (más una para la biblioteca) que mezcla ediciones diferenciales e instantáneas compartidas (`Arc`) para lo estructural; fusión por transacción explícita **y** ventana de 800 ms (Godot `undo_redo.cpp:95`); tope **por memoria** (OpenToonz `tundo.cpp:209`), no por cantidad; rehacer truncado al editar | Lote F |
| **D-Save** | **Autoguardado y guardado** | **Decidido**: el autoguardado anexa las ediciones a un diario (`.journal/`, JSON Lines, `fsync`, fuera del hilo de UI); Ctrl+S = instantánea `Arc` + hilo aparte que escribe **solo archivos sucios** (por instancia, D-Fmt) a temporal + *rename*, rota `.1`/`.2` (rotación de copias) y trunca el diario; abrir reproduce el diario si existe. El hilo de dibujo nunca toca disco | Lote F |
| **D-Img / D-Audio / D-Export / D-Text** | Alcance complementario ([`decisiones-modelo-de-datos.md`](decisiones-modelo-de-datos.md)) | **Decididos**: fondos raster con `image` + `peniko::Image` (soporte, no foco); audio básico con Firewheel/`symphonium` y onda propia; exportación PNG/EXR con `image` y video por `ffmpeg` **externo** (nunca enlazado); texto con `parley 0.11` + `skrifa 0.44` (alineación con `vello 0.10` medida); no se leen formatos de herramientas comerciales | Lote G |
| **D-API** | Headless y automatización | **Decidido** ([`decisiones-modelo-de-datos.md`](decisiones-modelo-de-datos.md)): la interfaz no tiene privilegios — toda acción es una `Edicion` u operación invocable sin ventana; render headless con `render_to_texture` (`vello-0.10.0/src/lib.rs:474`); tres puertas sobre la misma API: crates (`cargo doc`), CLI `lotte` con JSON (`clap 4.6`), servidor MCP (`rmcp 3.2.0`, Apache-2.0); esquemas JSON generados con `schemars 1.2`. Sin *scripting* embebido | Lote H; criterio transversal de F y del hito 1 |
| **D-Bench** | Banco de pruebas | **Decidido**: `criterion 0.8.2` como `[dev-dependencies]` (líneas base con `--baseline`, Windows); líneas base en un directorio fuera del repositorio; un número de rendimiento en un PR es una corrida local con su salida pegada, nunca la del CI. Alternativas medidas y descartadas: `divan`, `iai-callgrind` (solo Linux), `tango-bench` | M0.2: issue A7 |
| **D-Id** | Forma concreta de los **ids estables** (complementa D3) | Dos identidades por nodo, como recomienda la auditoría de Bevy: el `NodeId` posicional para el DAG en memoria (barato) **y** un id derivado de la **ruta de nombres** con separador de longitud por segmento (`lib.rs:1334-1338`; la técnica, no blake3/uuid) para canales, formato y copiar/pegar. Nombres duplicados en la misma jerarquía = **error de validación al cargar**, no colisión silenciosa. Godot resuelve `NodePath:propiedad` una vez y cachea; nodo inexistente → warning y se salta la pista, nunca fatal (`animation_mixer.cpp:734-739`) | Lotes C, D y E |

---

## Lote A — De mockup a aplicación (`lotte-app`)

Convierte el prototipo de laboratorio (`mockup`) en el sexto crate del workspace, con gate, sin
perder nada de lo verificado. Es el lote que hace que todo lo demás tenga dónde verse.

| # | Issue | Criterio medible | Tamaño |
|---|---|---|---|
| A1 | **Crate `lotte-app` con bucle winit propio, wgpu, vello y egui como biblioteca.** `[[bin]]`. Ventana, surface, `Renderer` de vello, `egui-winit` + `egui-wgpu`, `pump()` de octotablet en `about_to_wait`. Declarar en `[workspace.dependencies]`: `egui = "0.35"`, `egui-winit`, `egui-wgpu`, `egui_tiles = "0.16"`, `octotablet = "0.1"`, con la nota de por qué esas y no las últimas | `cargo run -p lotte-app` abre la ventana y `cargo tree -d` no lista ningún duplicado de `wgpu`, `egui`, `glam`, `kurbo`. Gate verde. `AGENTS.md` regenerado sin "no hay aplicación" | **L** |
| A2 | **Convención Y-arriba en `ViewportCamera`** (D-F). El `−1` en `world_to_screen_affine`, `screen_to_world` por la inversa, y **el test de orientación**: un punto con `y > pan.y` cae con `y` de pantalla menor que el centro. Mutación: quitar el `−1` → el test cae. `architecture.md` fija la convención por escrito | Test nuevo verde; mutación en rojo pegada en el PR; el mockup deja de necesitar su `volteo_y` provisorio | **S** |
| A3 | **Los cuatro paneles como hojas de `egui_tiles`** con `Behavior` propio; cámara con `push_layer` al rect del pane; `is_tab_closable` en `true`; "Restablecer distribución" | Arrastrar, acoplar, cerrar y restablecer funcionan (lista de verificación manual en el PR) y el rect de cámara se remide por frame (log) | **M** |
| A4 | **Trazos en coordenadas de mundo + deshacer/rehacer/borrar.** Pila de rehacer; Ctrl+Z, Ctrl+Shift+Z, Supr; el ancho del pincel en unidades de mundo | Test unitario de la pila (deshacer tras N trazos deja N−1 y rehacer los devuelve en orden); mutación: invertir el orden de la pila → cae | **S** |
| A5 | **Capa de tableta como módulo aislado** (`lotte-app/src/tableta.rs`): construcción diferida con la ventana viva, `pump()` solo en `about_to_wait` (regla nueva del overlay), herramienta emulada distinguida por `Type::Emulated`, prueba de humo documentada con hardware | Regla `pump-solo-entre-mensajes` en `factory/rules/` con el incidente de la bitácora como Origen; el módulo no exporta `Manager` | **S** |
| A6 | **Archivos de licencia** (D-Lic): `LICENSE-MIT`, `LICENSE-APACHE`, `NOTICE` con las atribuciones ya debidas (DragonBonesCPP MIT, OpenToonz BSD-3, Graphite Apache/MIT, egui_tiles) | Los tres archivos existen; `cargo package --list -p lotte-core` los incluye; README enlaza | **S** |
| A7 | **Banco de pruebas con `criterion 0.8.2`** (D-Bench): `[dev-dependencies]` y `[[bench]]` en `lotte-timeline` (`FCurve::evaluate`) y `lotte-core` (`RigDag::evaluate_all` con jerarquía profunda de 300 pegs **y** plana de 300 pegs); líneas base guardadas en un directorio fuera del repositorio | `cargo bench` corre en verde en los dos crates; el PR pega la salida cruda de las cuatro mediciones y **reemplaza** la frase "convergencia sub-microsegundo" del README por el número medido (o la quita si no se cumple); `cargo tree` del workspace sin `criterion` fuera de `dev-dependencies` | **M** |

**Orden:** A6 y A2 no dependen de nada y pueden ir en paralelo con A1. A3, A4 y A5 dependen de A1.
**Retrospectiva al cierre:** ¿el laboratorio quedó vacío de código vivo? Si no, algo no se migró.

## Lote B — Profundidad y multiplano (M1 del plan, con OpenToonz medido)

Depende de D1 y de la auditoría de OpenToonz de esta noche (fórmula y signo verificados).

| # | Issue | Criterio | Tamaño |
|---|---|---|---|
| B1 | `z: f32` en `Transform2D` + acumulación jerárquica (plan 4) | hijo `z=10` bajo padre `z=5` → global `15` | **M** |
| B2 | `focal_distance` y $S_z = D/(D-z)$ en `ViewportCamera` (plan 5, D1 decidida: mayor `z` = más cerca) | `Sz(0)=1`, `Sz(-D)=0.5`, `z >= D` → `Err` de validación; paneo reproduce la fórmula de `multiplane-parallax.md` | **M** |
| B3 | "Maintain Size" (plan 6) | proyección idéntica dentro de `1e-5` | **S** |
| B4 | **Deslizador de `z` por peg en el panel de nodos de `lotte-app` y capas de la mesa multiplano visibles en la cámara** — para *ver* B1–B3 | Mover `z` cambia el tamaño en pantalla según $S_z$ (captura + número) | **M** |

## Lote C — Visor de nodos que edita, y modelo de dibujo (M3 + `ui-widgets` §1)

Depende de D3 y de las auditorías de Godot y Graphite de esta noche.

| # | Issue | Criterio | Tamaño |
|---|---|---|---|
| C1 | **Cables editables en el visor**: arrastrar desde un puerto crea/reasigna el padre (`set_parent`), con *snap* al puerto según lo medido en `graph_edit.cpp`; cable rojo y rechazo cuando `DagError::CycleDetected` | Test: intentar conectar un ancestro como hijo devuelve `Err` y el visor no cambia el DAG; mutación: quitar la detección → el test cae | **M** |
| C2 | **`NodeKind`**: Peg / Drawing / Deformer / Composite (§1.1) **+ `Group`** — contenedor con un puerto de entrada y uno de salida, color por tipo en el visor. **Ids estables** para nodos (no el `NodeId(usize)` posicional): un canal animable es `(ruta de nodo, atributo)` y tiene que sobrevivir entre archivos (Godot, Bevy, glTF) | Serialización aditiva en `lotte-format` con fixture de la versión anterior; un índice de nodos que cuente grupos y falle con cero (`todo-barrido-afirma-que-leyo-algo`) | **M** |
| C3 | **Fijar la autoridad del dibujo** (plan 10, D3) con la evidencia de Graphite | ADR en `docs/decisiones/modelo-de-dibujo.md`; `VectorItem` o su reemplazo tiene ids estables | **M** |
| C4 | Sub-capas Under/Fill/Stroke/Over (plan 11) y paleta indexada (plan 12) | Los criterios del plan | **M** + **M** |
| C5 | **Trazo con presión como envolvente de anchura** (plan 13, `patterns.md` §6): línea central editable + perfil de anchura, contorno con `kurbo` y `fill()` | Un trazo del lápiz se guarda como centro + anchuras, no como polígono; se re-rasteriza a otro zoom sin pérdida | **L** |
| C6 | Hit-testing de curvas portado de Graphite `click_target.rs` con atribución | Seleccionar un trazo con un click a ≤ tolerancia; test con un punto fuera y otro dentro | **S** |
| C7 | **Dos primitivas de arte: `Stroke` (línea central + perfil de grosor) y `Contour` (forma rellena)** ([`requisitos-animador.md`](requisitos-animador.md) §5). Lápiz / línea / polilínea capturan la línea central de distinta forma y guardan lo mismo; el pincel resuelve presión → envolvente → `Contour` al soltar | Test: un `Stroke` convertido a `Contour` con `kurbo::stroke` tiene el área esperada ±1 %; un `Contour` no tiene conversión inversa (no compila / `None`) | **M** |
| C8 | **Fusión de pinceladas del mismo color** con `linesweeper 0.4.0` (MIT OR Apache-2.0; mismo `kurbo 0.13.1`, misma versión que Graphite, verificado 2026-09-08) — unión booleana sobre Bézier al soltar, solo dentro de la misma capa de arte y el mismo `colorID` | Test: dos contornos que se solapan → un contorno; dos que no → dos; dos de color distinto → dos. Mutación: unir sin comprobar color → cae | **M** |
| C9 | **Editor de perfil de grosor** para `Stroke`: nudos sobre la **longitud de arco**, **izquierda y derecha independientes**, interpolador Catmull-Rom centrípeta por defecto (más lineal y ajuste Bézier), remates y uniones; interacción: arrastrar nudo, modificador para agregar, otro para borrar, diálogo numérico como alternativa; curva 1D de anchura **separada** de la rutina que la dobla sobre la spline (testeable aislada), límite anti-*overshoot* en tangentes vecinas de igual signo, y la posición por longitud de arco como default con la alternativa por índice expuesta al usuario | El perfil editado se serializa en `lotte-format` de forma aditiva y re-rasteriza igual a otro zoom (round-trip con fixture); **test del pliegue**: un ancho mayor que el radio de curvatura del lado cóncavo no produce lazo, y un caso en que el pliegue cruza más de un segmento vecino queda cubierto o marcado `#[ignore]` con justificación | **M** |

## Lote D — Modelo temporal completo (con Rerun como referencia)

Depende de D2. Referencias: Rerun (`re_time_panel`) para el dope sheet, Godot para las manijas, matemática publicada para TCB/Clamped.

| # | Issue | Criterio | Tamaño |
|---|---|---|---|
| D1 | **Dope sheet con culling**, no virtualización: *culling* por fila con `is_rect_visible` y por **rango de tiempo** visible — el patrón real de `re_time_panel` según la auditoría verificada (`time_panel.rs:925,1115`; `data_density_graph.rs:591-631`) | 1000 pistas × 2000 frames sintéticos mantienen el frame de UI por debajo de 4 ms (timestamps, número en el PR); el test cuenta cuántas filas se pintaron y afirma que son ≤ las visibles + 2 | **M** |
| D2 | Regla de tiempo con espaciado por zoom y scrub (patrón `re_time_ruler`) | Marcas mayores cada 12/24 frames según zoom; el cabezal sigue el mouse con snap a frame | **S** |
| D3 | `AnimatableProperty<T>`, monotonía de manijas, inserción con Casteljau, fuera de rango (plan 18–21), **Dos interpolaciones nuevas en `Interpolation`**, las únicas que no se expresan con manijas fijas: **TCB/Auto** (Kochanek-Bartels 1984, fórmula pública; el modo por defecto de la mayoría de los motores) y **Clamped** (sin *overshoot* en extremos locales — clave para opacidad, escala y para el perfil de grosor de C9); Halt/Ease queda como `Bezier` con manijas en cero. **Monotonía en tiempo garantizada por construcción**, como Godot: el constructor/setter de `Keyframe::Bezier` clampa `handle_in.x ≤ 0` y `handle_out.x ≥ 0` (`animation.cpp:3352-3358`), no se asume. Interpolación angular por camino corto (plan 22) — Bevy usa slerp incluso en 2D | Los criterios del plan; test: un handle con `x` del signo equivocado se clampa y la curva sigue monótona en tiempo; **ninguna línea de código ajeno** (revisión explícita) | **L** |
| D4 | Editor de curvas en `lotte-app`: manijas smooth/broken/flat (§2.3), lista de capacidades de la auditoría. Los modos de manija son **estado del editor**, no del formato (Godot los compila solo con `TOOLS_ENABLED`) | Cada modo con test sobre la `FCurve` resultante, no sobre el dibujo | **L** |
| D7 | **Blending por capas** (plan 24) con el diseño del grafo de Bevy y la convención de Godot: nodos `Clip` / `Blend` (peso normalizado) / `Add` (peso libre) evaluados sobre el DAG de blend, **máscaras por bitfield `u64`** sobre la jerarquía de pegs, el reposo de D-Inst como semilla del blend, y una capa base tipo `RESET`; transiciones como ajuste de pesos, no como tipo de nodo | Test: dos clips con pesos 0.3/0.7 sobre el mismo peg dan el valor ponderado; un `Add` no normaliza; una máscara excluye un subárbol; con todos los pesos en 0 el resultado es el reposo (mutación: semilla `Default` → cae) | **L** |
| D5 | Operaciones de hoja `reverse`/`step`/`each` (plan 23, OpenToonz) | Los criterios del plan | **S** |
| D6 | Persistencia del layout de `egui_tiles`. **No** serializar el `Tree` (Rerun tampoco lo hace): guardar los paneles y contenedores como datos propios de Lotte, versionados con `lotte-format` de forma aditiva, y **reconstruir el árbol** al abrir — patrón de `re_viewport_blueprint` (`build_tree_from_views_and_containers`, `:1193`) | Cerrar y abrir `lotte-app` restaura la distribución; un layout guardado con un panel que ya no existe abre igual (test con fixture) | **M** |

## Lote E — Flujo del animador ([`requisitos-animador.md`](requisitos-animador.md))

Depende de D-Inst y de los ids estables de C2/C3. Es el lote que hace que Lotte se *sienta* como
una herramienta de producción y no como un motor.

| # | Issue | Criterio | Tamaño |
|---|---|---|---|
| E1 | **Definición / instancia / animación en `lotte-core`** (D-Inst): `RigDag` guarda el reposo; la evaluación recibe la `Animation` como entrada; `set_local_transform` se parte en `set_rest_transform` (edita el rig) y canales animados | Test: animar un peg, resetear → vuelve al reposo; editar el reposo con animación puesta → el valor efectivo cambia y la animación se conserva. Mutación: hacer que reset borre el reposo → cae | **L** |
| E2 | **Modos Editar rig / Animar en `lotte-app`** con destino distinto, indicador visible del modo, y **Reset** (pieza / personaje) | Lista de verificación manual + el test de E1 detrás de cada botón | **M** |
| E3 | **`Animation` separable: copiar y pegar entre escenas** por `(ruta, atributo)`; lo que no empareja se **reporta**, no se inventa | Test: dos instancias de la misma definición, pegar la animación de una en la otra reproduce las mismas matrices globales frame a frame (`evaluate_all` idéntico); pegar sobre una definición distinta devuelve la lista de rutas sin par | **M** |
| E4 | **Clonar vs duplicar**: clon = instancia nueva de la misma definición; duplicado = definición nueva con **ids regenerados**; marca de clon en el visor de nodos. Formato como `defs`/`use` de SVG: definiciones en una sección `defs` con `id`, instancias con `use="id"`; **cargar un id repetido es error explícito**. Semántica: la de la instanciación de escenas de Godot y las *precomps* de Lottie | Test: editar la definición mueve al clon y no al duplicado; `lotte-format` guarda la definición una vez para N clones (fixture con dos clones pesa ~igual que uno); un fixture con `id` duplicado no carga | **M** |
| E5 | **Importar SVG con `usvg 0.46`** (Apache-2.0 OR MIT; verificado: una vello, una kurbo, una wgpu): rellenos, trazos, degradados, transformaciones aplanadas → `BezPath` + apariencia | Test con un SVG de fixture exportado por un editor vectorial: cuenta de subrutas y bounding box esperados; mutación: ignorar la transformación del `<g>` → cae | **M** |
| E6 | **Exportar SVG**: path desde `BezPath`; grosor variable como **contorno resuelto** (relleno) más línea central y perfil en namespace `lotte:` que Lotte relee; capas de arte como `<g id="line|colour|…">` | Round-trip Lotte → SVG → Lotte conserva la envolvente; un visor SVG externo abre el archivo y ve el contorno correcto (captura en el PR) | **M** |
| E7 | **`Scene` y `Library` en `lotte-format`** ([`decisiones-modelo-de-datos.md`](decisiones-modelo-de-datos.md), "La escena"): la escena como lista ordenada de instancias con tipo (`background` 0–1 · `props[]` · `characters[]` · `overlays`), `timing { frames, frame_start, frame_start_global, fps }`, cámara animable; cada instancia con **nombre único en la escena**, referencia a una definición por `id` + versión, y colocación (offset, escala, z) en su **peg de instancia**, nunca dentro de la definición; definiciones en `defs`, instancias con `use`; al abrir se re-resuelven contra la biblioteca y lo que no empareja se **avisa**, no se inventa | Fixture de escena con un fondo de tres capas y dos instancias del mismo rig: `evaluate_all` por instancia da matrices distintas por el peg de instancia e idénticas por debajo; mover una instancia no toca la definición (mutación: escribir la colocación en el peg interno → cae); una definición ausente carga con aviso y deja la instancia vacía. **Con D-Fmt**: `Scene` se guarda como `scene.json` dentro del directorio de plano y las animaciones de sus instancias en `anim/` | **L** |
| E8 | **Tablas de tiempo: tres codificaciones medidas** (D-Tab, [`decisiones-modelo-de-datos.md`](decisiones-modelo-de-datos.md)): `lotte-format` define las tablas (`keys [N,6]`, `key_interp [N]`, `channel_offsets [C+1]`, `exposure_runs [R,3]`, `exposure_offsets [X+1]`) como structs `serde` y las escribe/lee en **JSON**, **CBOR con typed arrays** (`ciborium 0.2`) y **safetensors** (`safetensors 0.8.0`); `cargo tree` debe mostrar un solo `serde_json` | Fixture generado a las magnitudes de un rig de producción (≥ 3 400 canales, ≥ 12 000 claves) × 10 instancias; round-trip sin pérdida en las tres; tabla "Números de este PR" con bytes y tiempo de apertura de cada una (es la medición que elige el predeterminado); un `channel_offsets` corrupto (no monótono) da `Err`, no pánico (mutación: quitar la validación → cae) | **M** |

**No entra:** pegar animación entre rigs distintos (mapa de rutas), Lottie, Rive, `vello_svg`
(va una versión atrás de vello, medido 2026-09-08), pincel raster (Graphite lo tiene; Lotte no lo
necesita), adoptar glTF/OTIO/USD como formato de documento — solo alinear el modelo interno de
canales con el de glTF para que exportar sea traducir (`requisitos-animador.md` §6, **sin medir
todavía contra la especificación**).

---


## Lote F — Modelo de documento: deshacer, diario y guardado (D-Undo, D-Save)

Va **antes** de que `lotte-app` tenga herramientas de edición reales (lote C): toda herramienta
produce `Edición`, así que el modelo tiene que existir primero. El mockup de hoy tiene una pila de
deshacer de juguete que se reemplaza. Fuentes: [`referencias/godot-undo-redo.md`](referencias/godot-undo-redo.md),
OpenToonz `tundo.cpp`,
Graphite `document_history.rs`.

| # | Issue | Criterio medible | Tamaño |
|---|---|---|---|
| F1 | **`Edicion` con inverso** en un crate `lotte-doc` (o módulo de `lotte-format`): `enum` serializable de mutaciones sobre `Scene`/`Animation`/`RigDefinition` con `aplicar`, `deshacer` y `bytes()`; instantánea compartida (`Arc`) como variante para lo estructural | Cada variante tiene test de ida y vuelta: aplicar + deshacer deja el documento **igual** (`PartialEq`), y aplicar cambia un valor **propagado** (no el que el setter escribió); `bytes()` de "poner clave" < 100 (medido con `size_of` + payload) | **L** |
| F2 | **Pila por documento con tope por memoria**: una por plano abierto y una para la biblioteca; `version` monótona como marca de sucio; rehacer truncado al editar; tope configurable en bytes que descarta lo más viejo | Test: 10 000 ediciones con tope de 1 MB → la pila no supera el tope y el documento sigue coherente; mutación: quitar el descarte → cae. Editar la biblioteca no aparece en la pila del plano (test) | **M** |
| F3 | **Fusión de gestos**: transacción explícita `comenzar_gesto`/`terminar_gesto` **y** ventana temporal (800 ms, mismo nombre) para repeticiones; al fusionar se conserva `antes` del primero y `despues` del último | Test: 200 ediciones "mover manija" dentro de un gesto = **1** paso que deshace al estado inicial; dos pulsaciones de flecha a 100 ms = 1 paso, a 1 s = 2 pasos (reloj inyectable, sin `sleep`) | **M** |
| F4 | **Diario** `plano/.journal/<sesion>.jsonl`: anexado con `fsync` desde un hilo propio, una edición por línea; al abrir, si existe y no está vacío, se ofrece reproducirlo; una línea truncada se descarta con aviso | Test: escribir 1 000 ediciones, cortar el archivo a mitad de la última línea, reabrir → 999 aplicadas y un aviso; el hilo de UI nunca llama a `write` (test de que la API pública no expone I/O sincrónica) | **M** |
| F5 | **Guardado en segundo plano por archivo sucio**: instantánea `Arc` del documento, hilo que escribe solo `scene.json`/`anim/<instancia>.*` con `version` distinta de la guardada, a temporal + `rename`; rota los anteriores a `.1`, `.2` (por defecto 2); al terminar trunca el diario | Test con dos instancias: editar solo una → el `mtime` de la otra no cambia; matar el proceso (simulado: `panic` en el hilo) a mitad → el archivo original sigue intacto y el diario sigue completo; mutación: escribir en el original directo → cae | **L** |

## Lote G — Medios: fondos, audio, exportación, texto (alcance complementario)

Soporte, no foco. Cada issue empieza con su *spike* medido (alineación de versiones contra el lock,
licencia, un caso real del estudio) y entra como **tipo de nodo o de pista aditivo**: nada de esto
toca `lotte-core`. Depende de A (ventana) y E7 (`Scene`).

| # | Issue | Criterio medible | Tamaño |
|---|---|---|---|
| G1 | **Fondo raster** como nodo de dibujo: `image 0.25` decodifica PNG/JPEG/WebP a `peniko::Image`; mismo peg y `z` que un dibujo vectorial; `Scene.background` con capas ATRÁS/MEDIO/FRENTE | Un fondo de tres capas PNG se compone con dos rigs delante y el paneo de cámara da paralaje distinto por capa (`S_z` medido por capa); `cargo tree` con **una** `png` (la de vello) | **M** |
| G2 | **PSD por capas** con `psd 0.3.5`: cada capa visible → un nodo raster con su offset y su orden | Un PSD real del estudio (a elegir por el dueño) importa N capas con el mismo orden y posición que en Photoshop (comparación de un render contra una exportación plana, diferencia de píxel medida); si el crate no cubre el archivo, el issue lo reporta con el error crudo y se cierra sin forzar | **M** |
| G3 | **Pista de audio**: cargar con `symphonium`, onda como pirámide de picos/RMS dibujada con `vello` bajo el dope sheet, reproducción con Firewheel (o `cpal` directo si el *spike* lo descarta) sincronizada a frames, *scrub* al arrastrar el cursor | WAV de 30 s a 24 fps: la posición reportada por el motor de audio y el frame del cursor difieren < 1 frame durante toda la reproducción (medido con un *log* de 720 muestras); la onda de un archivo de 10 min se dibuja sin recalcular picos por frame (tiempo por frame medido); `cargo tree` con **una** `glam` | **L** |
| G4 | **Exportar**: render fuera de pantalla con `vello` a textura + *readback* → secuencia PNG (y EXR si `image` lo da) con `image`; video invocando **`ffmpeg` externo** como proceso, detectado en `PATH`, con mensaje claro si falta; nunca enlazado ni distribuido | 48 frames exportados son idénticos píxel a píxel al render en pantalla del mismo frame (diferencia medida = 0); sin `ffmpeg` instalado, la exportación de video falla con un mensaje y la de PNG funciona; `cargo tree` sin ningún crate `ffmpeg*` | **L** |
| G5 | **Texto vectorial**: `parley 0.11` + `skrifa 0.44` convierten un texto a `Contour`s (glifos como `BezPath`) en un nodo de dibujo; a partir de ahí es geometría común | `cargo tree` muestra **una** `skrifa 0.44` compartida con `vello` y **una** `peniko`; el contorno de una letra se rellena, se anima con un peg y se exporta a SVG como `<path>` | **M** |

## Hito 1 — Animar un puppet (definido por el dueño el 2026-09-08)

> "Tener una base mínima que permita animar puppets. Quizás incluso sin sonido: animar puppets y
> exportarlos como secuencia de imágenes. Y poder guardar una escena y poder abrirla."

Es una **rebanada vertical**, no un lote: toma de cada lote solo lo que hace falta para que un
animador importe un puppet dibujado afuera, lo arme con pegs, lo anime con claves, guarde el plano,
lo vuelva a abrir idéntico y exporte los frames. Todo lo demás espera a que esto funcione en manos
del dueño.

### Definición de terminado (medible)

1. **Importar** un puppet de ≥ 10 piezas desde SVG y verlas en la cámara con Y-arriba.
2. **Armar**: crear pegs, emparentarlos, poner pivotes y asignar cada dibujo a su peg; el reposo
   queda en la definición (D-Inst).
3. **Animar**: poner claves en posición/rotación/escala de varios pegs a lo largo de 48 frames,
   sustituir dibujos por exposición (bocas/manos), reproducir en bucle a 24 fps sin caídas
   visibles; deshacer/rehacer con fusión de gestos.
4. **Guardar** el plano como directorio (`scene.json` + `anim/*` + biblioteca), cerrar, **abrir** y
   obtener el mismo resultado: los 48 frames renderizados antes y después de reabrir son
   **idénticos píxel a píxel** (hash por frame). Autoguardado por diario: matar el proceso a mitad
   de sesión y recuperar al abrir.
5. **Exportar** los 48 frames a PNG, idénticos al render en pantalla.

### Issues, en orden de dependencias

| Paso | Issues | Qué entrega | Sesión |
|---|---|---|---|
| 1 | **A1, A2, A3, A5, A6** | `lotte-app` con ventana, vello, egui, paneles, tableta; Y-arriba; licencias | 1 |
| 2 | **E1** | reposo ⊕ animación en `lotte-core` (la única que cambia código existente: antes de todo lo que la use) | 1 |
| 3 | **F1, F2, F3** | `Edicion` con inverso, pila por memoria, fusión de gestos — toda herramienta del hito produce `Edicion` | 2 |
| 4 | **E5** | importar SVG con `usvg` a dibujos del rig | 2 |
| 5 | **H1** (nuevo) | **Armar el puppet**: panel de jerarquía para crear peg, emparentar (arrastrar; C1 lo hará también por cables), mover el pivote con el gizmo, asignar un dibujo importado a un peg, renombrar; todo como `Edicion` | 2 |
| 6 | **D1, D2, H2** (nuevo) | dope sheet con *culling*, regla con *scrub*; **H2: claves y reproducción**: poner/mover/borrar clave desde el dope sheet, *auto-key* al mover un peg en modo Animar, sustitución de dibujo por celda, play/stop/bucle a fps del plano, interpolación Bézier auto por defecto | 3 |
| 7 | **E7, E8** | `Scene` + `Library` como directorio de plano; tablas de tiempo en tres codificaciones medidas (el predeterminado sale de la medición) | 3 |
| 8 | **F4, F5** | diario con recuperación; guardado en segundo plano por archivo sucio con rotación `.1` | 4 |
| 9 | **H3** (nuevo, absorbe la mitad PNG de G4) | **`lotte export`**: CLI headless (`clap`) que abre un directorio de plano, renderiza con `render_to_texture` sin ventana y escribe la secuencia PNG; la interfaz llama al mismo comando. Sin `ffmpeg`, sin EXR | 4 |
| 10 | **A7** | `criterion` sobre `FCurve::evaluate` y `RigDag::evaluate_all`, y el número real en el README | 4 |

Dieciocho issues (tres nuevos: **H1**, **H2**, **H3**), cuatro sesiones de orquestación. Se escriben
con criterio medible al abrirlos, como el resto; sus mutaciones en rojo pasan por la pila de F1.
**Criterio transversal del hito (D-API)**: cada acción de la interfaz que entre en H1/H2 existe
primero como `Edicion` u operación con test sin ventana; el test de "terminado" 4 y 5 (reabrir
idéntico, exportar idéntico) se corre **desde la CLI**, sin abrir la aplicación.

### Lo que queda **explícitamente fuera** del hito 1

Audio (G3), fondos raster y PSD (G1, G2), texto (G5), video por `ffmpeg` (G4b), multiplano (B1–B4),
pincel con presión, `Stroke`/`Contour`, fusión de pinceladas y editor de grosor (C4–C9), editor de
curvas y TCB/Clamped (D3, D4), *blending* (D7), operaciones de hoja (D5), layout persistente (D6),
modos Editar/Animar con Reset (E2), copiar animación entre planos (E3), clonar vs duplicar (E4),
exportar SVG (E6), Lottie. Nada de eso cambia el modelo de datos: todo entra después de forma
aditiva. **Dibujar dentro de Lotte** también queda fuera del hito 1: el puppet se dibuja afuera y
se importa; el mockup conserva su lápiz básico (A4) solo como prueba de la tableta.

### Nuevas decisiones: solo cuando algo suceda

Con el hito 1 definido, **no queda ninguna decisión técnica abierta**. Las que van a aparecer son de
medición, no de arquitectura, y cada una tiene su issue: `F32` vs `F64` y la codificación
predeterminada de las tablas (E8), el número de rendimiento del README (A7), `octotablet` en sesiones
largas (A5), y el comportamiento real del autoguardado con planos grandes (F4/F5). Lo que se aprenda
usando el hito 1 abre el hito 2, no reabre estas decisiones.

## Lote H — Automatización: CLI, MCP y esquemas (D-API)

La interfaz no tiene privilegios: cada operación existe primero sin ventana. H3 va en el hito 1;
el resto crece con los hitos que le dan operaciones que exponer.

| # | Issue | Criterio medible | Tamaño |
|---|---|---|---|
| H3 | **`lotte export`** (hito 1): binario `lotte` con `clap 4.6`; abre un directorio de plano, crea `wgpu` **sin superficie**, renderiza cada frame con `render_to_texture` y escribe PNG con `image`; la interfaz llama a la misma función | 48 frames exportados desde la CLI, sin ventana, idénticos píxel a píxel a los del visor (hash por frame); corre en el CI (adaptador de software `wgpu` si no hay GPU: si no existe, el test se marca y se documenta, no se inventa); `--help` lista los comandos | **M** |
| H4 | **CLI completa con JSON**: `info` (resumen del plano en JSON), `validate` (esquema + reglas: nombres duplicados, `z < D`, definiciones ausentes), `import-anim` (aplica un `anim/*` de la biblioteca a una instancia por `(ruta, atributo)`, reporta lo que no empareja), `combine` (instancias de varios planos en uno), `render-thumb`; salida JSON en `stdout`, errores en `stderr` con código de salida | Cada subcomando tiene un test que lo corre como proceso sobre un fixture y valida el JSON contra su esquema (H5); `import-anim` sobre un rig incompatible reporta N canales sin destino y no toca el plano (mutación: aplicar igual → cae) | **L** |
| H5 | **Esquemas y documentación generada**: `schemars 1.2` deriva JSON Schema de `Edicion`, de las operaciones y de `scene.json`/`anim/*.json`; `cargo doc` sin warnings; un comando `lotte schema` los emite | Los esquemas se regeneran en el gate y un test falla si difieren de los versionados (candado contra cambios no aditivos, junto a `contratos-solo-aditivos`); todo fixture del repo valida contra su esquema | **M** |
| H6 | **Servidor MCP** con `rmcp 3.2.0`: expone las operaciones de H4 y las `Edicion` sobre un plano abierto como herramientas, con las definiciones generadas desde H5; transporte stdio | Un cliente MCP de prueba lista las herramientas (conteo > 0, igual al de operaciones), abre un plano, pone una clave y exporta un frame; el resultado es idéntico al de la CLI (mismo hash); ninguna herramienta hace algo que la CLI no pueda | **L** |

## Después del hito 1 — orden propuesto

| Etapa | Qué | Por qué en este orden |
|---|---|---|
| Hito 2 — flujo del animador | E2, E3, E4, E6; C1–C3 | Es lo que un animador profesional busca el primer día: modos, reset, copiar animación, clonar; y el visor de nodos que edita |
| Hito 3 — dibujar en Lotte | C4–C9 | Pincel con presión, `Stroke`/`Contour`, fusión, editor de grosor; exige el modelo de dibujo asentado |
| Hito 4 — tiempo completo | D3, D4, D5, D6, D7 | Editor de curvas, TCB/Clamped, mezcla, layout |
| Hito 5 — profundidad | B1–B4 | Multiplano con el signo de `z` decidido; autocontenido |
| Hito 6 — medios | G1, G2, G3, G4b, G5; Lottie | Fondos, audio, video, texto; soporte, no foco |
| Transversal desde el hito 2 — automatización | H4, H5, H6 | CLI completa (`info`, `validate`, `import-anim`, `combine`, `render-thumb`), servidor MCP con `rmcp`, esquemas `schemars` que generan la documentación; regla `ui-sin-privilegios` |

Ocho lotes, ~51 issues. Los tamaños son estimaciones y se re-miden al abrir cada
issue; los criterios se escriben con "Cómo se midió" como manda la regla.
