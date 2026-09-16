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

**Estado (2026-09-08): todas decididas** — el acta, con las mediciones y las palabras del autor, está en
[`decisiones-modelo-de-datos.md`](decisiones-modelo-de-datos.md). La columna "Recomendación medida"
se conserva como registro; donde difiere del acta (D1), gana el acta.

| # | Decisión | Recomendación medida | Bloquea |
|---|---|---|---|
| **D1** | Signo de `z` | **Decidido**: mayor `z` = más cerca, arte en `z ≈ 0`, cámara en `z = D`, validación `z < D`. Evidencia (`tstageobject.cpp:1945-1968`): OpenToonz hace `dz = focus + cameraZ − objectZ`, escala `(focus+cameraZ)/dz` — a mayor Z, más cerca; es también la convención de los animadores cutout. No copiar su "maintain size" como canal manual (`m_noScaleZ`): la compensación derivada de la spec es mejor | Lote B |
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
| **D-Lang** | Idioma de los identificadores | **Decidido (2026-09-14)**: el código va en **inglés** (`Edit`, `begin_gesture`, `Pose`), consistente con los cinco crates (`RigDag`, `ViewportCamera`) y con toda dependencia; documentación, commits e issues en español, como hasta ahora. Mezclar los dos en el mismo árbol obligaría a un rename masivo después | todas las issues nuevas |
| **D-i18n** | Idioma de la **interfaz** | **Decidido (2026-09-14)**: ningún texto visible se escribe literal en el código; todo sale de archivos **Fluent** (`.ftl`, Project Fluent, Apache-2.0 OR MIT — `fluent-bundle 0.16`, `unic-langid 0.9`), uno por idioma en `locales/<lang>/lotte.ftl`, con `es` como idioma de desarrollo y `en` desde el primer día; idioma del sistema por defecto (`sys-locale 0.3`) y cambio en caliente desde el menú. Fluent es estándar abierto, nativo de Rust, maneja plurales y género, y lo entienden Weblate, Crowdin y Pontoon — un traductor no toca código. Alternativa medida y descartada: gettext `.po` (`gettext-rs` enlaza C) | A8; criterio transversal de toda issue con UI |
| **D-Doc** | Dónde vive el modelo de documento | **Decidido (2026-09-14)**: crate **`lotte-doc`** (documento, `Edit`, historial, diario, guardado), sobre `lotte-format` + `lotte-rig` + `lotte-timeline` + `lotte-core`; `lotte-app` y la CLI `lotte` son clientes de `lotte-doc` (D-API). El diagrama de capas de `AGENTS.md` gana dos niveles y se edita por las fuentes de la Fábrica | F1 |
| **D-Bench** | Banco de pruebas | **Decidido**: `criterion 0.8.2` como `[dev-dependencies]` (líneas base con `--baseline`, Windows); líneas base en un directorio fuera del repositorio; un número de rendimiento en un PR es una corrida local con su salida pegada, nunca la del CI. Alternativas medidas y descartadas: `divan`, `iai-callgrind` (solo Linux), `tango-bench` | M0.2: issue A7 |
| **D-Id** | Forma concreta de los **ids estables** (complementa D3) | Dos identidades por nodo, como recomienda la auditoría de Bevy: el `NodeId` posicional para el DAG en memoria (barato) **y** un id derivado de la **ruta de nombres** con separador de longitud por segmento (`lib.rs:1334-1338`; la técnica, no blake3/uuid) para canales, formato y copiar/pegar. Nombres duplicados en la misma jerarquía = **error de validación al cargar**, no colisión silenciosa. Godot resuelve `NodePath:propiedad` una vez y cachea; nodo inexistente → warning y se salta la pista, nunca fatal (`animation_mixer.cpp:734-739`) | Lotes C, D y E |

---

## Lote A — De mockup a aplicación (`lotte-app`)

Convierte el prototipo de laboratorio (`mockup`) en el sexto crate del workspace, con gate, sin
perder nada de lo verificado. Es el lote que hace que todo lo demás tenga dónde verse.

| # | Issue | Criterio medible | Tamaño |
|---|---|---|---|
| A1 | **Crate `lotte-app`** sobre la API de **E1** (`RigDefinition` + `Pose`): bucle winit propio, wgpu, vello, egui como biblioteca, `egui_tiles`, **`egui-phosphor`**, `pump()` de octotablet en `about_to_wait`; **tema oscuro forzado** (`Context::set_theme(Dark)` + `Some(winit::window::Theme::Dark)` en `State::new` — es el `Theme` de winit, no el de egui); caja de herramientas de dos columnas (70 px: 4 + 30 + 2 + 30 + 4). Declarar en `[workspace.dependencies]`: `egui = "0.35"`, `egui-winit`, `egui-wgpu`, `egui_tiles = "0.16"`, `egui-phosphor = "0.13"`, `octotablet = "0.1"`, con la nota de por qué esas versiones y no las últimas. `AGENTS.md` se corrige por las **fuentes** de la Fábrica (`fabrica.yaml`, `factory/templates/AGENTS.md.hbs`): fuera "no hay aplicación" y "ningún crate usa vello/wgpu/winit". La lógica de el mockup de laboratorio se **porta, no se copia**: su `Motor` desaparece a favor de `RigDefinition` + `Pose` | `cargo run -p lotte-app` abre la ventana con el rig de ejemplo animando; `cargo tree -d` no lista ningún duplicado de `wgpu`, `egui`, `glam`, `kurbo`, `skrifa`; `fabrica check` OK; gate verde. **Depende de E1** | **L** |
| A2 | **Convención Y-arriba en `ViewportCamera`** (D-F). El `−1` en `world_to_screen_affine`, `screen_to_world` por la inversa, y **el test de orientación**: un punto con `y > pan.y` cae con `y` de pantalla menor que el centro. Mutación: quitar el `−1` → el test cae. `architecture.md` fija la convención por escrito | Test nuevo verde; mutación en rojo pegada en el PR; `lotte-app` no necesita ningún volteo provisorio | **S** |
| A3 | **Los cuatro paneles como hojas de `egui_tiles`** con `Behavior` propio; cámara con `push_layer` al rect del pane; `is_tab_closable` en `true`; "Restablecer distribución" | Arrastrar, acoplar, cerrar y restablecer funcionan (lista de verificación manual en el PR) y el rect de cámara se remide por frame (log) | **M** |
| A5 | **Capa de tableta como módulo aislado** (`lotte-app/src/tablet.rs`): construcción diferida con la ventana viva, `pump()` solo en `about_to_wait`, herramienta emulada distinguida por `Type::Emulated`, y el hallazgo del spike: el `Down` de octotablet **no trae posición** — el módulo conserva el último `Pose` y lo entrega con el `Down`. **Prueba de humo**: un trazo con presión en el lienzo (sin deshacer propio: el deshacer llega con F1) | Regla `pump-solo-entre-mensajes` en `factory/rules/` con el incidente de la bitácora como Origen; el módulo no exporta `Manager`; prueba de humo documentada con hardware en el PR (rango de presión medido) | **S** |
| A8 | **Internacionalización desde el primer día** (D-i18n): `fluent-bundle 0.16` + `unic-langid 0.9` + `sys-locale 0.3`; `locales/es/lotte.ftl` y `locales/en/lotte.ftl` con **todos** los textos de A1 (menús, herramientas, tooltips, indicadores); idioma del sistema por defecto, cambio en caliente desde el menú; un helper `t!("clave")` o equivalente, sin cadenas literales visibles en `lotte-app` | Test que carga las dos locales y afirma que tienen **el mismo conjunto de claves** (conteo > 0, `todo-barrido-afirma-que-leyo-algo`); test que recorre `lotte-app/src` y falla si aparece una cadena literal en una llamada de UI (`ui.label("…")`, `menu_button("…")`) — mutación: agregar una → cae; cambiar de idioma en la app no reinicia ni pierde el estado | **M** |
| A6 | **Archivos de licencia** (D-Lic): `LICENSE-MIT`, `LICENSE-APACHE`, `NOTICE` con las atribuciones ya debidas (DragonBonesCPP MIT, OpenToonz BSD-3, Graphite Apache/MIT, egui_tiles, egui-phosphor), y `license = "MIT OR Apache-2.0"` en el `Cargo.toml` de **cada** crate (medido 2026-09-14: ninguno lo declara) | Los tres archivos existen en la raíz; `grep -L 'license = ' crates/*/Cargo.toml` no devuelve nada; README enlaza. (No se afirma nada sobre `cargo package`: empaqueta el directorio del crate, no la raíz; se decide cuando haya publicación) | **S** |
| A7 | **Banco de pruebas con `criterion 0.8.2`** (D-Bench): `[dev-dependencies]` y `[[bench]]` con `harness = false` en `lotte-timeline` (`FCurve::evaluate`) y `lotte-core` (`RigDag::evaluate_all` con jerarquía profunda de 300 pegs **y** plana de 300 pegs); líneas base fuera del repo (`CRITERION_HOME` apuntando a un directorio fuera del repositorio, o copia tras `--save-baseline`) | `cargo bench` corre en verde en los dos crates; el PR pega la salida cruda de las cuatro mediciones y **reemplaza** la frase "convergencia sub-microsegundo" del README por el número medido (o la quita si no se cumple); `cargo tree` del workspace sin `criterion` fuera de `dev-dependencies` | **M** |
| A9 | **Perfil `dev` para worktrees persistentes**: en el `Cargo.toml` del workspace, `[profile.dev] incremental = false` y `debug = "line-tables-only"`, con el comentario de por qué. Cómo se midió (2026-09-15, worktree limpio, clippy + test): valores por defecto 10,5 s / 249 MB / 989 archivos, de los cuales 70 MB son `incremental/`; con los dos ajustes 11,0 s / 162 MB / 464 archivos. Compartir el directorio de build entre worktrees se descartó por medición: cargo no mete la ruta del worktree en el hash del crate y un worktree termina enlazando el `.rlib` que compiló otro (falso verde) | En un worktree limpio, `cargo clippy --workspace --all-targets -- -D warnings` + `cargo test --workspace` dejan `target/` con **≤ 500 archivos** (`find target -type f \| wc -l`, salida pegada) y sin `target/debug/incremental/`; el gate no tarda más de un 10 % que con el perfil por defecto (las dos corridas pegadas); un `panic` en un test sigue mostrando `archivo:línea` en el backtrace (pegar uno) | **S** |

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
| C0 | **Armar el puppet** (hito 1): panel de **jerarquía** (árbol con nombres — no el visor de nodos con cables, que es C1); crear peg, emparentar por arrastre en el árbol, renombrar con **validación de nombre duplicado** en la jerarquía (D-Id); mover el pivote con el gizmo; asignar un `Drawing` (E5) a un peg. Lo medido en el spike entra como especificación: *hit-test* del pivote más cercano en píxeles físicos con radio 12 (aciertos medidos de 0,6 a 9,7 px); el delta del arrastre va al espacio local del **padre** (`globals[parent].inverse().transform_vector2(delta)`); selección múltiple por marco, y arrastrar sobre un pivote seleccionado **mueve el conjunto** (de un conjunto se mueven solo los pegs sin ancestro seleccionado). Modos **Editar rig / Animar** con destino distinto (reposo vs `Pose` animada) e indicador visible — la forma mínima de E2. Todo como `Edit` (F1), nada llama a `set_*` del modelo | Test sin ventana: crear 3 pegs, emparentar, mover el hijo con padre rotado 90° → la posición local cambia en el eje correcto (mutación: no convertir al espacio del padre → cae); renombrar a un nombre existente → `Err`; mover en modo Editar rig cambia el reposo y no la `Pose`; en Animar, al revés (mutación: escribir en el reposo en modo Animar → cae) | **L** |
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
| D0 | **Claves y reproducción** (hito 1): poner/mover/borrar clave desde el dope sheet; **auto-key** al mover un peg en modo Animar (posición y rotación), interpolación Bézier automática por defecto; sustitución de dibujo por celda sobre el `ExposureTrack` actual (`HashMap<u32, ExposureCell>`, alcanza para el hito); play/stop/bucle a los fps del plano con reloj inyectable | Test sin ventana: mover un peg en el frame 10 y en el 30 crea dos claves y el frame 20 evalúa un valor **intermedio distinto de ambos** (mutación: auto-key escribe en el reposo → cae); borrar la clave del 30 deja el 20 igual al 10; una celda de exposición en el frame 5 se mantiene (hold) en el 6–9 y cambia en el 10 | **L** |
| D1 | **Dope sheet con culling**, no virtualización: *culling* por fila con `is_rect_visible` y por **rango de tiempo** visible — el patrón real de `re_time_panel` según la auditoría verificada (`time_panel.rs:925,1115`; `data_density_graph.rs:591-631`) | **Test**: con 1000 pistas × 2000 frames sintéticos, el conteo de filas pintadas es ≤ las visibles + 2 (mutación: pintar todas → cae). **Número del PR, no del CI**: el frame de UI por debajo de 4 ms medido en la máquina del developer con timestamps, pegado en la tabla "Números de este PR" | **M** |
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
| E1 | **Definición / instancia / animación en `lotte-core`** (D-Inst) **conservando D2**: `RigDag` guarda el **reposo**; una **`Pose`** en `lotte-core` (valores por nodo y atributo — posición, rotación, escala — sin saber de curvas) y `evaluate_all_with(&Pose)`; quien muestrea las `FCurve` en el frame `t` y arma la `Pose` es **`lotte-rig`** (`lotte-core` no puede recibir `Animation`: dependería de `lotte-timeline`, y `lotte-timeline` no conoce `NodeId`). `set_local_transform` se parte en `set_rest_transform` + `Pose`. **Formato**: el campo serializado de `PegNode` conserva su nombre o lleva `#[serde(alias = "transform")]` — renombrarlo rompe todo proyecto guardado (`LotteProject` serializa `RigDag` entero, `lib.rs:31`); entra el **primer fixture de versión anterior** `crates/lotte-format/tests/fixtures/proyecto_v1.json` (regla `contratos-solo-aditivos`; medido 2026-09-14: `lotte-format` tiene un solo test, misma versión). Doc de `Keyframe.time` (`fcurve.rs:36`) → frames del proyecto (D-T). El ejemplo de la regla `mutaciones-antes-del-pr` (`factory/rules/`, fuente) se actualiza a la API nueva y se regenera | Test: animar un peg vía `Pose`, resetear (Pose vacía) → vuelve al reposo; editar el reposo con `Pose` puesta → el valor efectivo cambia y la `Pose` se conserva (mutación: reset borra el reposo → cae). `proyecto_v1.json` abre y `evaluate_all` da las mismas matrices que antes de E1 (fixture generado **antes** de tocar el código). `cargo tree -p lotte-core` no lista `lotte-timeline` (D2) | **L** |
| E2 | **Modos Editar rig / Animar en `lotte-app`** con destino distinto, indicador visible del modo, y **Reset** (pieza / personaje) | Lista de verificación manual + el test de E1 detrás de cada botón | **M** |
| E3 | **`Animation` separable: copiar y pegar entre escenas** por `(ruta, atributo)`; lo que no empareja se **reporta**, no se inventa | Test: dos instancias de la misma definición, pegar la animación de una en la otra reproduce las mismas matrices globales frame a frame (`evaluate_all` idéntico); pegar sobre una definición distinta devuelve la lista de rutas sin par | **M** |
| E4 | **Clonar vs duplicar**: clon = instancia nueva de la misma definición; duplicado = definición nueva con **ids regenerados**; marca de clon en el visor de nodos. Formato como `defs`/`use` de SVG: definiciones en una sección `defs` con `id`, instancias con `use="id"`; **cargar un id repetido es error explícito**. Semántica: la de la instanciación de escenas de Godot y las *precomps* de Lottie | Test: editar la definición mueve al clon y no al duplicado; `lotte-format` guarda la definición una vez para N clones (fixture con dos clones pesa ~igual que uno); un fixture con `id` duplicado no carga | **M** |
| E5 | **Importar SVG con `usvg 0.46`** (Apache-2.0 OR MIT; verificado: una vello, una kurbo, una wgpu): rellenos, trazos de ancho fijo, degradados, transformaciones aplanadas → un **`Drawing` mínimo en `lotte-rig`** (D3: id estable + `Vec<(BezPath, Appearance)>`), que hoy no existe — el único tipo con geometría es `VectorItem` en `lotte-render` (`scene.rs:11`), que pasa a ser una vista de render sobre `Drawing`. C7 lo extiende después con `Stroke`/`Contour` de forma aditiva. **Fixture real**: un puppet de ≥ 10 piezas del estudio, exportado desde su editor vectorial, aportado por el autor en `crates/lotte-rig/tests/fixtures/`; más un SVG sintético para el test de `<g>` | Test con el fixture real: cuenta de piezas y *bounding box* esperados; test con el sintético: una transformación en `<g>` se aplana (mutación: ignorarla → cae); `cargo tree -d` sin duplicados | **M** |
| E6 | **Exportar SVG**: path desde `BezPath`; grosor variable como **contorno resuelto** (relleno) más línea central y perfil en namespace `lotte:` que Lotte relee; capas de arte como `<g id="line|colour|…">` | Round-trip Lotte → SVG → Lotte conserva la envolvente; un visor SVG externo abre el archivo y ve el contorno correcto (captura en el PR) | **M** |
| E7 | **`Scene`, `Library` y `RigInstance` en `lotte-format`** ([`decisiones-modelo-de-datos.md`](decisiones-modelo-de-datos.md), "La escena"): la escena como lista ordenada de instancias con tipo (`background` 0–1 · `props[]` · `characters[]` · `overlays`), `timing { frames, frame_start, frame_start_global, fps }`, cámara animable; cada instancia con **nombre único en la escena**, referencia a una definición por `id` + versión, y colocación (offset, escala, z) en su **peg de instancia**, nunca dentro de la definición; definiciones en `defs`, instancias con `use`; al abrir se re-resuelven contra la biblioteca y lo que no empareja se **avisa**, no se inventa. Serialización JSON de `scene.json` y `definition.json` como **tipos**; el directorio de plano es E9 | Fixture de escena con un fondo de tres capas y dos instancias del mismo rig: `evaluate_all` por instancia da matrices distintas por el peg de instancia e idénticas por debajo; mover una instancia no toca la definición (mutación: escribir la colocación en el peg interno → cae); una definición ausente carga con aviso y deja la instancia vacía; dos instancias con el mismo nombre → `Err` | **M** |
| E9 | **Directorio de plano** (D-Fmt): crear/abrir `plano/` con `scene.json`, `anim/<instancia>.anim.json` (+ tablas de E8 cuando existan) y la referencia a la biblioteca; resolución de rutas relativas; `Library` como directorio `rigs/<id>/definition.json` + `drawings/*.svg` | Test: crear un plano con dos instancias en un `tempfile::tempdir()`, cerrar, abrir → `Scene` igual (`PartialEq`) y los 48 frames evaluados dan las mismas matrices (hash); una animación de una instancia se copia a otro plano copiando **un** archivo y aplica por `(ruta, atributo)` (mutación: resolver por índice → cae) | **M** |
| E8 | **Tablas de tiempo: tres codificaciones medidas** (D-Tab): `lotte-format` define las tablas (`keys [N,6]`, `key_interp [N]`, `channel_offsets [C+1]`, `exposure_runs [R,3]`, `exposure_offsets [X+1]`) como structs `serde` y las escribe/lee en **JSON**, **CBOR con typed arrays** y **safetensors** (`safetensors 0.8.0`). Aviso al developer: `ciborium 0.2` **no** trae RFC 8746 listo — los *tags* 64–87 se escriben con serde a mano (a verificar; si existe un crate alineado que lo haga, se prefiere). El **tiempo de apertura se mide con `criterion`** (A7), no en un test | Fixture reducido (1 instancia, ≥ 3 400 canales, ≥ 12 000 claves) con round-trip sin pérdida en las tres codificaciones en `cargo test`; fixture completo (× 10 instancias) solo en el bench; tabla "Números de este PR" con bytes y tiempo de apertura de cada una — **esa medición elige el predeterminado, el issue no lo fija**; un `channel_offsets` corrupto (no monótono) da `Err`, no pánico (mutación: quitar la validación → cae). `cargo tree` con un solo `serde_json`. **Depende de A7** | **M** |

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
| F1 | **`Edit` con inverso** en el crate nuevo **`lotte-doc`** (D-Doc; identificadores en inglés, D-Lang): `enum` serializable de mutaciones sobre `Scene`/`Animation`/`RigDefinition` con `apply`, `undo` y `bytes()`; instantánea compartida (`Arc`) como variante para lo estructural. Incluye la regla **`ui-sin-privilegios`** en `factory/rules/` (un PR que agregue una acción a la interfaz sin su `Edit` u operación invocable desde la CLI se rechaza) y el diagrama de capas de `AGENTS.md` con `lotte-doc`, por las fuentes de la Fábrica | Cada variante tiene test de ida y vuelta: `apply` + `undo` deja el documento **igual** (`PartialEq`), y `apply` cambia un valor **propagado** (no el que el setter escribió); `bytes()` de "poner clave" < 100 (medido con `size_of` + payload); `fabrica check` OK | **L** |
| F2 | **Pila por documento con tope por memoria**: una por plano abierto y una para la biblioteca; `version` monótona como marca de sucio; rehacer truncado al editar; tope configurable en bytes que descarta lo más viejo | Test: 10 000 ediciones con tope de 1 MB → la pila no supera el tope y el documento sigue coherente; mutación: quitar el descarte → cae. Editar la biblioteca no aparece en la pila del plano (test) | **M** |
| F3 | **Fusión de gestos**: transacción explícita `comenzar_gesto`/`terminar_gesto` **y** ventana temporal (800 ms, mismo nombre) para repeticiones; al fusionar se conserva `antes` del primero y `despues` del último | Test: 200 ediciones "mover manija" dentro de un gesto = **1** paso que deshace al estado inicial; dos pulsaciones de flecha a 100 ms = 1 paso, a 1 s = 2 pasos (reloj inyectable, sin `sleep`) | **M** |
| F4 | **Diario** `plano/.journal/<sesion>.jsonl`: `append(&self, edit)` encola en un canal y un hilo propio anexa con `fsync`, una edición por línea; al abrir, si existe y no está vacío, se ofrece reproducirlo; una línea truncada se descarta con aviso | Test: escribir 1 000 ediciones, cortar el archivo a mitad de la última línea, reabrir → 999 aplicadas y un aviso. **`append` no bloquea**: test con un *sink* inyectado que se detiene en una barrera — `append` retorna igual y la edición llega cuando el sink se libera (mutación: escribir sincrónico → el test se cuelga con timeout → cae) | **M** |
| F5 | **Guardado en segundo plano por archivo sucio**: instantánea `Arc` del documento, hilo que escribe solo `scene.json`/`anim/<instancia>.*` con `version` distinta de la guardada, a temporal + `rename`; rota los anteriores a `.1`, `.2` (por defecto 2); al terminar trunca el diario | Test con dos instancias: editar solo una → el **hash de contenido** del archivo de la otra no cambia (no `mtime`: depende del sistema de archivos); un `panic` inyectado en el hilo a mitad de escritura → el archivo original sigue intacto y el diario sigue completo; mutación: escribir en el original directo → cae | **L** |

## Lote G — Medios: fondos, audio, exportación, texto (alcance complementario)

Soporte, no foco. Cada issue empieza con su *spike* medido (alineación de versiones contra el lock,
licencia, un caso real del estudio) y entra como **tipo de nodo o de pista aditivo**: nada de esto
toca `lotte-core`. Depende de A (ventana) y E7 (`Scene`).

| # | Issue | Criterio medible | Tamaño |
|---|---|---|---|
| G1 | **Fondo raster** como nodo de dibujo: `image 0.25` decodifica PNG/JPEG/WebP a `peniko::Image`; mismo peg y `z` que un dibujo vectorial; `Scene.background` con capas ATRÁS/MEDIO/FRENTE | Un fondo de tres capas PNG se compone con dos rigs delante y el paneo de cámara da paralaje distinto por capa (`S_z` medido por capa); `cargo tree` con **una** `png` (la de vello) | **M** |
| G2 | **PSD por capas** con `psd 0.3.5`: cada capa visible → un nodo raster con su offset y su orden | Un PSD real del estudio (a elegir por el autor) importa N capas con el mismo orden y posición que en Photoshop (comparación de un render contra una exportación plana, diferencia de píxel medida); si el crate no cubre el archivo, el issue lo reporta con el error crudo y se cierra sin forzar | **M** |
| G3 | **Pista de audio**: cargar con `symphonium`, onda como pirámide de picos/RMS dibujada con `vello` bajo el dope sheet, reproducción con Firewheel (o `cpal` directo si el *spike* lo descarta) sincronizada a frames, *scrub* al arrastrar el cursor | WAV de 30 s a 24 fps: la posición reportada por el motor de audio y el frame del cursor difieren < 1 frame durante toda la reproducción (medido con un *log* de 720 muestras); la onda de un archivo de 10 min se dibuja sin recalcular picos por frame (tiempo por frame medido); `cargo tree` con **una** `glam` | **L** |
| G4 | **Exportar**: render fuera de pantalla con `vello` a textura + *readback* → secuencia PNG (y EXR si `image` lo da) con `image`; video invocando **`ffmpeg` externo** como proceso, detectado en `PATH`, con mensaje claro si falta; nunca enlazado ni distribuido | 48 frames exportados son idénticos píxel a píxel al render en pantalla del mismo frame (diferencia medida = 0); sin `ffmpeg` instalado, la exportación de video falla con un mensaje y la de PNG funciona; `cargo tree` sin ningún crate `ffmpeg*` | **L** |
| G5 | **Texto vectorial**: `parley 0.11` + `skrifa 0.44` convierten un texto a `Contour`s (glifos como `BezPath`) en un nodo de dibujo; a partir de ahí es geometría común | `cargo tree` muestra **una** `skrifa 0.44` compartida con `vello` y **una** `peniko`; el contorno de una letra se rellena, se anima con un peg y se exporta a SVG como `<path>` | **M** |

## Hito 1 — Animar un puppet (definido por el autor el 2026-09-08)

> "Tener una base mínima que permita animar puppets. Quizás incluso sin sonido: animar puppets y
> exportarlos como secuencia de imágenes. Y poder guardar una escena y poder abrirla."

Es una **rebanada vertical**, no un lote: toma de cada lote solo lo que hace falta para que un
animador importe un puppet dibujado afuera, lo arme con pegs, lo anime con claves, guarde el plano,
lo vuelva a abrir idéntico y exporte los frames. Todo lo demás espera a que esto funcione en manos
del autor.

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
| 1 | **E1** | reposo + `Pose` en `lotte-core` conservando D2; alias serde y primer fixture de versión anterior — la única issue que cambia código existente, y va **sola** antes de todo lo que la use | 1 |
| 2 | **A2, A6, A7, A9** | Y-arriba con test; licencias en raíz y en cada crate; `criterion` y el número real del README; perfil `dev` sin incremental para los worktrees persistentes | 1 |
| 3 | **A1, A3, A5, A8** | `lotte-app` sobre la API de E1: ventana, vello, egui, paneles, tableta, tema oscuro, iconos, textos en Fluent | 2 |
| 4 | **F1, F2, F3** | `lotte-doc`: `Edit` con inverso, pila por memoria, fusión de gestos; regla `ui-sin-privilegios` | 2 |
| 5 | **E5** | importar SVG a `Drawing` en `lotte-rig`, con el puppet real del estudio como fixture | 3 |
| 6 | **C0** | armar el puppet: jerarquía, pivotes, asignar dibujos, selección simple y por marco, modos Editar rig / Animar | 3 |
| 7 | **D1, D2, D0** | dope sheet con *culling*, regla con *scrub*, claves y reproducción con auto-key | 3 |
| 8 | **E7, E9, E8** | `Scene` + `Library` como tipos; directorio de plano; tablas en tres codificaciones medidas | 4 |
| 9 | **F4, F5** | diario con recuperación; guardado en segundo plano por archivo sucio | 4 |
| 10 | **H3** | `lotte export` headless, y el test de "terminado" 4 y 5 corrido desde la CLI | 4 |


Veintitrés issues, cuatro sesiones de orquestación (revisadas el 2026-09-14: A4 eliminado; A8, C0, D0 y E9 nuevos; E1 primero y sola; A9 agregada el 2026-09-15 tras medir el costo en disco de los worktrees).
Orden de fusión por `Cargo.lock`: A7 y A1 primero, E5 y E8 rebasadas después, H3 última — cinco issues
agregan dependencias y ningún par de ellas se fusiona el mismo día.
**Criterio transversal del hito (D-API)**: cada acción de la interfaz que entre en H1/H2 existe
primero como `Edicion` u operación con test sin ventana; el test de "terminado" 4 y 5 (reabrir
idéntico, exportar idéntico) se corre **desde la CLI**, sin abrir la aplicación.

### Estado al cierre del primer lote (2026-09-16)

El primer lote del hito 1 corrió con agentes (un orquestador, developers y QA con medición): 30 PR
fusionados, 8 crates, 249 tests. Los pasos 1 a 3 de la definición de terminado quedaron cerrados sin
ventana; el 4 y el 5 abiertos. Issues nacidas del lote y agregadas al hito: A2b (rotación antihoraria
de la cámara), A0 (`lotte-app` monta un `Document`), A0b (el candado de inventario sigue los módulos
de archivo), H3b-1 (dibujos en la biblioteca) y H3b-2 (el export dibuja la figura real), D0b (dope
sheet sobre curvas reales), y de la prueba manual del autor C1 (mano y barra espaciadora), C2 (la
herramienta gobierna la entrada) y F6 (menú Archivo). Las convenciones chicas que aparecieron están
en el acta, "Convenciones fijadas durante la primera implementación".

### Lo que queda **explícitamente fuera** del hito 1

Audio (G3), fondos raster y PSD (G1, G2), texto (G5), video por `ffmpeg` (G4b), multiplano (B1–B4),
pincel con presión, `Stroke`/`Contour`, fusión de pinceladas y editor de grosor (C4–C9), editor de
curvas y TCB/Clamped (D3, D4), *blending* (D7), operaciones de hoja (D5), layout persistente (D6),
modos Editar/Animar con Reset (E2), copiar animación entre planos (E3), clonar vs duplicar (E4),
exportar SVG (E6), Lottie. Nada de eso cambia el modelo de datos: todo entra después de forma
aditiva. **Dibujar dentro de Lotte** también queda fuera del hito 1: el puppet se dibuja afuera y
se importa; el lápiz básico queda en A5 solo como prueba de humo de la tableta.

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
| H3 | **`lotte export`** (hito 1): binario `lotte` con `clap 4.6`; abre un directorio de plano (E9), crea `wgpu` **sin superficie**, renderiza cada frame con `render_to_texture` y escribe PNG con `image`; la interfaz llama a la misma función | Tres afirmaciones separadas. (a) **Determinismo, en CI**: dos corridas de la CLI en el mismo adaptador producen el mismo hash por frame (`gate.yml` instala `mesa-vulkan-drivers` y la CLI acepta `--fallback-adapter`); (b) **visor = CLI, local**: 48 frames idénticos píxel a píxel **en la misma máquina y adaptador**, número en el PR; (c) **golden con tolerancia, en CI**: contra un golden generado en lavapipe, ≤ 0,5 % de píxeles distintos (el antialiasing difiere entre adaptadores; "idéntico" entre GPU y software fallaría con el render correcto). `--help` lista los comandos. **Depende de E9** | **M** |
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

Ocho lotes, ~54 issues. Los tamaños son estimaciones y se re-miden al abrir cada
issue; los criterios se escriben con "Cómo se midió" como manda la regla.
