# Stack tecnológico y decisiones — dónde estamos (2026-09-08)

Un solo documento que responde tres preguntas: **qué** hemos elegido, **por qué** (velocidad,
arquitectura, compatibilidad, flujo del animador o licencia), y **en qué punto estamos** antes de
escribir código de producto. Las fuentes detalladas están enlazadas; acá está la síntesis.

Todo lo que dice "medido" tiene su comando y su salida en el documento que enlaza. Lo que no se ha
medido se marca así.

---

## 1. Punto de partida: tres razones y un método

**Las tres razones que ordenan todo lo demás:**

1. **Rust.** Sin recolector de basura ni pausas; concurrencia sin carreras de datos (el guardado en
   segundo plano y el hilo de dibujo comparten estado sin bloqueos); un solo lenguaje del motor a
   la interfaz; ecosistema gráfico maduro (`wgpu`, `vello`, `kurbo`, `egui`) con **una sola copia**
   de cada tipo geométrico en todo el árbol.
2. **Aceleración por GPU para animación vectorial.** El dibujo es vector (Bézier), no píxeles: se
   rasteriza en la GPU en cada frame, a cualquier zoom, sin caché de bitmaps que invalidar cuando
   un peg se mueve.
3. **Velocidad al animar.** El caso de uso que manda: un animador moviendo cientos de pegs de varios
   personajes sobre un fondo complejo, con autoguardado, deshacer profundo, y sin que el motor se
   ponga pesado por guardar. Medido en producción: un rig tiene ~300 pegs y ~1 000 nodos
   (rig de producción medido: 293 pegs, 1 061 nodos, 3 438 canales animables, 12 639 claves).

**El método**, que también es una decisión: nada entra sin medirse (`medir-antes-de-razonar`);
solo se lee código de licencia permisiva (MIT/BSD/Apache), con atribución — el código copyleft no se lee —; y **no se escribe código de producto hasta cerrar
las decisiones que lo cambiarían**. Rehacer un modelo de datos con archivos guardados encima es lo
más caro que hay; por eso esta etapa fue larga y fue de papel.

## 2. El stack, capa por capa

Leyenda de razones: **[V]** velocidad · **[A]** arquitectura · **[C]** compatibilidad de versiones ·
**[F]** flujo del animador · **[L]** licencia.

### 2.1 Base y matemática

| Crate | Versión | Rol | Por qué |
|---|---|---|---|
| `glam` | 0.32.1 | Vectores y matrices del motor (`Vec2`, `Affine2`) | **[V]** SIMD, `Copy`, sin asignaciones. **[C]** Es la misma versión que usa `bevy_animation 0.19.1`: sus tipos son los nuestros |
| `kurbo` | 0.13.1 | Geometría vectorial: `BezPath`, `Affine`, `Stroke`, `offset_cubic`, ajuste de curvas | **[A]** Es la geometría de `vello`: el dibujo del animador y lo que la GPU pinta son el **mismo tipo**, sin conversión. **[C]** `linesweeper 0.4.0` (booleanas) y `usvg 0.46` (importar SVG) usan **esta** `kurbo` |
| `peniko` | 0.6.1 | Color, pinceles, `Fill`, `Mix` | **[C]** la fija `vello` |
| `serde` / `serde_json` | 1.0.229 / 1.0.151 | Serialización de todo lo que se guarda | **[A]** un esquema, varias codificaciones (JSON, CBOR, safetensors): la decisión de formato no se casa con el struct. **[C]** dependencia ya presente en `egui`, `egui_tiles`, `safetensors` |
| `thiserror` | 1.0.69 | Errores tipados por crate | **[A]** un `Err` nunca es un pánico en pantalla |

### 2.2 Render

| Crate | Versión | Rol | Por qué |
|---|---|---|---|
| `vello` | 0.10.0 | Rasterizador vectorial **por cómputo en GPU** | **[V]** rasteriza rutas Bézier con *compute shaders*, sin teselar en CPU: el costo por frame no crece con el zoom ni con el número de trazos redibujados. **[A]** modelo de escena declarativo (`Scene`, `push_layer`, `fill`, `stroke`) que se reconstruye cada frame — el motor no mantiene estado gráfico. **[L]** Apache/MIT. Es el motor de Graphite, la única aplicación de dibujo vectorial en Rust en producción |
| `wgpu` | 29.0.4 | Abstracción de GPU (Vulkan/DX12/Metal) | **[C]** **no se declara**: la fija `vello`, y toda otra dependencia con GPU tiene que resolver a **esta** (por eso `egui 0.35`, no 0.33 ni 0.36). Una sola `wgpu` en el árbol: medido en el lock |

### 2.3 Ventana, eventos y tableta

| Crate | Versión | Rol | Por qué |
|---|---|---|---|
| `winit` | 0.30.13 | Ventana y bucle de eventos, **propio** | **[A]** el bucle es nuestro, no de un framework: es la única forma de meter la tableta donde tiene que ir (abajo). **[C]** `egui-winit 0.35` y `octotablet 0.1` lo comparten: un solo `winit` |
| `octotablet` | 0.1.0 | Presión, tilt y eventos de lápiz vía **Windows Ink** (`RealTimeStylus`) | **[F]** la presión es lo que separa un lápiz de un ratón; `winit` no la expone (cero ocurrencias de `Tablet` en 0.30.13). Verificado con Wacom Intuos S: presión 0.0000–0.5125 en trazos reales. Condición **no negociable** (medida por experimento de contradicción): `pump()` corre en `about_to_wait`, entre mensajes, nunca dentro del repintado — si no, *deadlock* del apartamento COM |
| `pollster` | 0.3 | `block_on` para crear la superficie | mínimo; una sola llamada |

### 2.4 Interfaz

| Crate | Versión | Rol | Por qué |
|---|---|---|---|
| `egui` + `egui-winit` + `egui-wgpu` | **0.35.0** | Widgets, paneles, menús, línea de tiempo, visor de nodos — **como biblioteca**, sobre nuestro bucle | **[V]** modo inmediato: la UI se describe cada frame, sin árbol de widgets que sincronizar con el modelo — para una línea de tiempo con miles de celdas y un visor de nodos que se mueve a 60 fps es la forma barata. **[A]** pinta encima del blit de `vello` en el **mismo frame** (`LoadOp::Load`); la vista de cámara vive dentro de un panel de egui y se recorta con `push_layer`. **[C]** 0.35 es la **única** versión que resuelve a `wgpu 29`. Verificado: menús, panel de color, timeline, cámara, 4 trazos con presión, cierre limpio |
| `egui_tiles` | 0.16.0 | Paneles acoplables (pestañas arrastrables, divisores, cerrar) | **[F]** el animador ordena su espacio de trabajo. **[C]** la única versión que deja **una** `egui` en el lock (0.15 y 0.17 traen dos). **[L]** MIT/Apache, de rerun-io. Costo asumido: se ancla en tándem con `egui` |

### 2.5 Geometría avanzada, importación y formato (compatibles, medidos, todavía sin usar)

| Crate | Versión | Para qué | Por qué |
|---|---|---|---|
| `linesweeper` | 0.4.0 | Unión de contornos (el pincel de presión produce un contorno por segmento; la unión los funde) | **[C]** misma `kurbo`. Es lo que usa Graphite para booleanas. Alternativa medida: `i_overlay 8.1.1` |
| `usvg` | 0.46 | Importar SVG (de cualquier editor vectorial) a `Stroke`/`Contour` | **[C]** depende de nuestra `kurbo`. `vello_svg` descartado: ninguna versión alinea |
| `safetensors` | 0.8.0 | Tablas de tiempo como tensores; exportación al ecosistema de tensores | **[V]** `mmap` sin parsear números. **[C]** solo `serde` + `serde_json`. **[L]** Apache |
| `ciborium` | 0.2.2 | CBOR con *typed arrays*: el mismo esquema `serde` en binario, un solo archivo | **[A]** un modelo, dos codificaciones (texto para depurar y `diff`, binario para abrir rápido) — el patrón de USD, Godot y glTF |
| `image` / `psd` | 0.25.10 / 0.3.5 | Fondos raster y capas de PSD como nodos de dibujo (`peniko::Image`) | **[C]** `vello` ya trae `png`; **[F]** soporte a fondos de producción, no foco |
| `firewheel` + `symphonium` | 0.14.0 / 0.13.0 | Audio básico: carga, reproducción sincronizada a frames, *scrub*; la onda es nuestra | **[L]** MIT/Apache, activo; **[C]** *features* `glam-32` para alinear con el nuestro. No existe un "Audacity en Rust" (medido) |
| `parley` + `skrifa` | 0.11.1 / 0.44.0 | Texto → contornos de glifos → `Contour` | **[C]** medido: `parley` declara `skrifa 0.44.0` y `peniko 0.6`, las mismas de `vello 0.10` |
| `ffmpeg` (**externo**, no crate) | el del sistema | Video de salida | **[L]** invocado como proceso: no enlaza ni distribuye código GPL/LGPL. Las secuencias PNG/EXR salen con `image`, sin nada externo |
| `clap` | 4.6.6 | CLI `lotte` (`export`, `info`, `validate`, `import-anim`, `combine`) con JSON | **[A]** la interfaz no tiene privilegios: todo existe primero sin ventana (D-API) |
| `rmcp` | 3.2.0 | Servidor MCP: las mismas operaciones como herramientas para un modelo de lenguaje | **[A]** adaptador fino sobre la misma API; **[L]** SDK oficial, Apache-2.0 |
| `schemars` | 1.2.2 | JSON Schema de `Edicion`, operaciones y archivos | **[A]** la documentación se genera del código y alimenta CLI, MCP y validación |
| `criterion` | 0.8.2 | Benchmarks con líneas base (`[dev-dependencies]`) | **[V]** único modo de que un número de rendimiento exista y de detectar cuándo empeora (`--baseline`); Windows. Decidido |

**Descartados con medición** ([`stack-verificado.md`](stack-verificado.md) §3): `eframe` (se
queda con el bucle, no hay dónde poner `pump()`: se cuelga con el lápiz); `egui 0.36` (trae
`wgpu 30` al lado de la 29); Xilem/masonry 0.4 (ancla `vello 0.6`, `kurbo 0.12`, y no maneja
presión); presión vía `winit` (no existe); **Qt** (su único argumento era la tableta, y la tableta
funciona sin él); **CEF + web** como Graphite (frontera de procesos y tres texturas para componer
Chromium; Lotte no tiene esa razón); PyTorch/Triton (*runtimes* Python con CUDA, no dependencias
de un motor Rust; el volumen de un plano no los necesita).

### 2.6 De dónde viene el diseño

No inventamos el modelo: lo medimos en los proyectos de licencia permisiva que lo resuelven y nos quedamos con lo mejor de cada uno,
según su licencia (un informe por proyecto en [`referencias/`](referencias/)).

| Fuente | Licencia | Cómo la usamos | Qué nos dio |
|---|---|---|---|
| DragonBonesCPP | MIT | código, con atribución | armature/bone/slot, *skinning* LBS, timelines de slot separadas de hueso |
| OpenToonz | BSD-3 | código, con atribución | pegbar `TStageObject`, celdas de exposición, signo de `z`, deshacer acotado por memoria |
| Graphite | Apache/MIT | código, con atribución; **es nuestro stack** | vello+kurbo+wgpu alineados, ids `u64`, booleanas con `linesweeper`, deshacer por instantánea con tope 100 (lo que **no** hacemos para animar) |
| Rerun | Apache | código, con atribución | `egui` como biblioteca sobre bucle propio, `egui_tiles`, panel de tiempo, no serializar el árbol de paneles |
| Godot | MIT | código, con atribución | `GraphEdit` (visor de nodos), `Animation` por `NodePath:propiedad`, `Bone2D.rest` + `RESET`, `UndoRedo` (fusión de gestos, historia por escena) |
| Bevy (`bevy_animation`) | MIT/Apache | código, con atribución; misma `glam` | id de destino por ruta de nombres, grafo de mezcla; y el contraejemplo: no separa reposo de animación |
| Estándares abiertos: glTF, SVG, Lottie, CBOR (RFC 8746), safetensors, JSON Schema, MCP | abiertos | referencia de modelo y de intercambio | canal nodo + atributo con tangentes explícitas (glTF); `defs`/`use` y namespaces (SVG); *precomps* (Lottie); estructura en texto + números en buffers |
| Producción del estudio del autor | experiencia propia | requisitos de los animadores y reglas de composición aprendidas en 138 planos | copiar animación, reset, clon vs duplicado, lápiz vs pincel, capas de arte, relleno por referencia; definición en biblioteca, instancia con nombre único, peg de instancia por fuera, fondos por estructura, Z en pocos lugares |

## 3. Las decisiones de modelo, con su razón

Acta completa en [`decisiones-modelo-de-datos.md`](decisiones-modelo-de-datos.md); la tabla con lo
que cada una bloquea en [`plan-siguiente-etapa.md`](plan-siguiente-etapa.md). Todas **firmadas**.

| Decisión | Qué | Razón principal |
|---|---|---|
| **D-Inst** | Definición / instancia / animación: `valor(t) = reposo ⊕ animación(t)`; clon = otra instancia, duplicado = definición nueva | **[F]** copiar animación entre planos del mismo rig, resetear sin perder arreglos, editar el rig vs animarlo, clonar vs duplicar — los cuatro requisitos del animador. Godot y glTF lo hacen; Bevy no, y su propio informe dice por qué es un problema |
| **D-Id** | Dos ids por nodo: posicional en memoria + derivado de la ruta de nombres para canales y archivo; duplicados = error al cargar | **[V]** el DAG usa índices; **[F]** pegar animación entre rigs "compatibles" por ruta; **[A]** evitar la identidad por texto *case-sensitive* que rompió planos en producción |
| **D-T** | Tiempo en **frames** del proyecto (`f64` en curvas, `u32` en exposición); `fps` solo en la frontera | **[F]** "es la unidad básica"; cambiar el fps no mueve claves fuera de frame. OpenToonz y la práctica del animador |
| **D1** | Mayor `z` = más cerca, arte en `z ≈ 0`, cámara en `z = D`, validación `z < D` | **[C]** convención de OpenToonz y de los animadores cutout; **[F]** los animadores no reaprenden |
| **D-F** | Mundo Y-arriba, la cámara voltea | **[C]** OpenToonz; corrige el personaje cabeza abajo del mockup |
| **D2** | `lotte-timeline` no depende de `lotte-core` | **[A]** evaluar curvas sin cargar un rig; `PoseSpace` en `lotte-rig` |
| **D3** | El dibujo vive en `lotte-rig`; ids numéricos | **[A]** una sola autoridad sobre trazos y rellenos (Graphite) |
| **D-App** | Sexto crate `lotte-app`, dueño del bucle `winit` | **[A]** el bucle es el que manda (tableta) |
| **D-Lic** | `LICENSE-MIT`, `LICENSE-APACHE`, `NOTICE` ya | **[L]** el primer port necesita dónde atribuir |
| **D-Fmt** | Un plano es un **directorio**: `scene.json` + `anim/<instancia>.*` + biblioteca aparte | **[V]** guardar solo lo sucio; **[F]** copiar animación = copiar un archivo; dos animadores, dos archivos; **[A]** patrón glTF/Godot. |
| **D-Tab** | Tablas de tiempo como **tensores** con diccionario JSON (`keys [N,6]`, offsets por canal, exposición por corridas); codificación predeterminada la elige el fixture de E8 | **[V]** una clave en JSON pesa ~95 bytes; los mismos seis números como `f32` pesan 24: el texto no escala a 13 000 claves por rig. **[A]** `serde` permite JSON/CBOR/safetensors sin tocar el esquema |
| **D-Draw** | Un archivo por dibujo: **SVG + namespace `lotte:`**, contorno resuelto como *fallback* | **[C]** interoperable con todo; **[F]** fuera de la escena, para que el plano sea chico; **[A]** extensibilidad por namespaces prevista por la propia especificación SVG |
| **D-Undo** | `Edición` con inverso producida por la herramienta; pila única por plano (más biblioteca) que mezcla ediciones e instantáneas `Arc`; fusión por transacción **y** ventana de 800 ms; tope por **memoria** | **[F]** deshacer profundo estándar; **[V]** una edición de animación pesa decenas de bytes — la instantánea por paso (Graphite) serían cientos de MB con nuestros rigs |
| **D-Img / D-Audio / D-Export / D-Text** | Fondos raster, audio básico, exportación PNG + `ffmpeg` externo, texto vectorial — todo como nodos o pistas **aditivos**, en la familia de `vello` | **[F]** soporte a lo que un plano real necesita sin desviar el foco; **[C]** alineación medida; **[L]** nada GPL enlazado |
| **D-API** | La interfaz no tiene privilegios: toda acción es una `Edicion` u operación invocable sin ventana; render headless; tres puertas (crates, CLI JSON, MCP) sobre la misma API; esquemas generados | **[A]** automatizar exportar/combinar/importar animaciones desde fuera (pipeline del estudio, modelos de lenguaje) sin *scripting* embebido — el scripting vive fuera (Python, LLM) y las extensiones son Rust contra la API pública o propuestas al core; **[F]** la biblioteca de animaciones se aplica por lotes |
| **Formatos propietarios** | No se leen formatos de herramientas comerciales | **[L]** sin riesgo legal; intercambio por estándares abiertos (SVG, DragonBones JSON; Lottie/glTF como exportadores) |
| **D-Save** | Autoguardado = **anexar** ediciones a un diario (JSON Lines, `fsync`, hilo propio); Ctrl+S = instantánea `Arc` + hilo que escribe solo archivos sucios a temporal + *rename*, rota `.1`/`.2`, trunca el diario; abrir reproduce el diario | **[V]** el autoguardado cuesta lo que costó la edición, no lo que pesa el plano; el hilo de dibujo nunca toca disco; **[F]** recuperar la última sesión. |

**Modelo de dibujo** (especificado, sin decisión pendiente): dos primitivas — `Stroke` (línea
central + perfil de grosor a dos lados por longitud de arco, editable con un editor de perfil propio) y `Contour` (forma rellena); pincel de presión = envolvente → `Contour` → unión con
`linesweeper`; cuatro capas de arte; relleno por referencia a los trazos; paleta por indirección.

**Modelo temporal** (especificado): canal `(ruta, atributo)` → `FCurve` con manijas explícitas en el
archivo, monotonía por construcción, Hold/Linear/Bézier + TCB y Clamped; exposición separada de la
geometría; canales discretos se seleccionan, no se promedian; mezcla Clip/Blend/Add con `RESET`
como línea base.

## 4. Dónde estamos

**Lo que existe en código** (medido hoy): cinco crates, 14 archivos `.rs`, 1 406 líneas,
**13 tests**, gate en verde (fmt + clippy `--all-targets` + test `--workspace`), CI en verde.
`Transform2D` con pivote y `RigDag` perezoso están **implementados y verificados** contra tres
referencias; `FCurve` y `ExposureTrack` existen; `LotteProject` guarda un rig con sus curvas
(documento de personaje, no de plano). **No hay aplicación ni GPU en los crates**: todo eso vive en
un laboratorio fuera del repositorio del motor (`mockup`) que ya corre el stack completo — ventana,
vello, egui, paneles acoplables, tableta con presión, y los cuatro crates del motor en proceso.

**Lo que está decidido**: todas las decisiones de arriba, D-Bench incluida (`criterion`). **Ninguna pendiente.**

**Lo que está especificado y planificado**: ocho lotes, ~51 issues, con criterio medible cada uno
([`plan-siguiente-etapa.md`](plan-siguiente-etapa.md)):

| Lote | Qué | Depende de |
|---|---|---|
| A | `lotte-app`: del mockup al sexto crate, test de orientación, licencias | nada — puede arrancar hoy |
| F | modelo de documento: `Edición`, pila, diario, guardado en segundo plano | A |
| E1 | reposo vs animación en `lotte-core` (**la única que cambia código existente**) | nada; va antes de C |
| B | profundidad y multiplano con el signo de `z` decidido | A |
| C | visor de nodos que edita, `Stroke`/`Contour`, pincel, editor de grosor | E1, F |
| D | dope sheet, editor de curvas, TCB/Clamped, mezcla | E1 |
| E2–E8 | modos editar/animar, copiar animación, clonar/duplicar, SVG, `Scene`+`Library`, codificación de tablas medida | E1, F |
| G | medios: fondos raster y PSD, pista de audio con onda y *scrub*, exportación PNG/EXR + `ffmpeg` externo, texto vectorial | A, E7 |
| H | automatización: CLI `lotte` con JSON (H3 en el hito 1), servidor MCP, esquemas generados | F, E7 |

**Lo que no está verificado y hay que medir cuando toque** ([`stack-verificado.md`](stack-verificado.md) §4):
rendimiento (ningún benchmark todavía; la frase "sub-microsegundo" del README no tiene medición);
`octotablet` en sesiones largas y con desconexión; tilt; paneles flotantes en ventana propia;
`F32` vs `F64` en las tablas; JSON vs CBOR vs safetensors como predeterminado.

**El primer hito ya está definido** ([`plan-siguiente-etapa.md`](plan-siguiente-etapa.md), "Hito 1 — Animar un puppet"): importar un puppet desde SVG, armarlo con pegs, animarlo con claves y exposición, guardar el plano como directorio, reabrirlo idéntico (hash por frame) y exportar la secuencia PNG. Sin audio, sin dibujo interno, sin multiplano. Dieciocho issues en cuatro sesiones.

**La tesis de esta etapa**, dicha por el autor: rehacer código antes de tomar las decisiones
importantes es trabajar en el aire. Las decisiones están tomadas y medidas. Lo que sigue es código.
