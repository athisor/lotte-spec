# Stack verificado

Qué combinación de bibliotecas **funcionó de punta a punta en hardware real**, cómo está
armada, qué quedó descartado y qué decisiones abre. Todo lo que dice "verificado" tiene
una corrida detrás; lo que no, lo dice.

| | |
|---|---|
| Fecha | 2026-09-07 |
| Máquina | Windows 10 Pro 19045 · NVIDIA GeForce RTX 5060 Ti · Wacom Intuos S (Windows Ink ON) |
| Evidencia cruda | [`bitacora-tableta.md`](bitacora-tableta.md) — todas las corridas, comandos, salidas y las hipótesis falsadas |
| Prueba final | `winit + wgpu + vello + octotablet`: 2 trazos, 971 poses, presión 0.0000–0.5737, cierre limpio |

---

## 1. El stack que funcionó

| Crate | Versión | Rol | Estado |
|---|---|---|---|
| `winit` | 0.30.13 | Ventana y bucle de eventos, **propio** | ✅ verificado |
| `wgpu` | 29.0.4 | GPU. **No se declara**: la fija vello | ✅ verificado |
| `vello` | 0.10.0 | Render vectorial (escena, trazos, rellenos) | ✅ verificado |
| `kurbo` | 0.13.1 | Geometría (`Line`, `Stroke`, `BezPath`, ajuste de curvas) | ✅ verificado |
| `peniko` | 0.6.1 | Color, pinceles, `Fill` | ✅ verificado |
| `octotablet` | 0.1.0 | Presión/tilt de la tableta vía Windows Ink | ✅ verificado, con una condición (§2) |
| `pollster` | 0.3 | `block_on` para crear la superficie | ✅ verificado |
| `glam` | 0.32.1 | Matrices del motor (ya en el workspace) | ✅ en uso, gate verde |
| `egui-phosphor` | 0.13.0 | Tipografía de iconos (Phosphor, MIT) para la barra de herramientas | ✅ verificado 2026-09-08: declara `egui = "0.35"`, una sola `egui` en el lock |

Es el stack de Graphite (`f71d88f`) más la capa de tableta. Una sola `wgpu` en el árbol.

**Compatibles pero todavía sin correr** (§4): `egui` / `egui-winit` / `egui-wgpu` **0.35.0**.
Es la única versión de egui que resuelve al mismo `wgpu 29.0.4`: la 0.33 trae la 27, la
0.36 —la última— trae la 30.

## 2. Cómo está armado — y la condición que no se negocia

```
winit (bucle PROPIO)
 ├─ resumed()          → crear Arc<Window>, superficie wgpu, Renderer de vello,
 │                        octotablet::Builder::build_shared(&window)
 ├─ about_to_wait()    → octotablet::Manager::pump()   ← ÚNICO lugar válido
 │                        acumular trazos · request_redraw()
 └─ RedrawRequested    → armar vello::Scene · render_to_texture · blit · present
```

**`pump()` va entre mensajes, nunca dentro del manejador de ventana.** Probado por
contradicción: el mismo programa, sin cambiar nada más, funciona con `pump()` en
`about_to_wait` (535 poses) y se cuelga con `pump()` en `RedrawRequested`
(`Responding: False`, CPU 0.06). La razón es COM: un apartamento de un solo hilo (STA)
solo atiende llamadas marshaladas mientras bombea mensajes, y el plugin asíncrono de
RealTimeStylus que usa octotablet se traba dentro del WndProc.

Consecuencia directa: **el bucle de eventos tiene que ser nuestro.** Cualquier framework
que se lo quede y solo nos llame desde el repintado deja la tableta sin lugar donde vivir.

Notas de API medidas en el código (no en la documentación):

- `wgpu 29`: `Surface::get_current_texture()` ya no devuelve `Result` sino el enum
  `CurrentSurfaceTexture` (`Success` / `Suboptimal` / `Outdated` / `Timeout` / `Occluded` / …).
- `vello 0.10`: `Renderer::render_to_texture` (`lib.rs:474`) + `util::RenderContext` /
  `RenderSurface` con `blitter` para presentar. `RendererOptions` no implementa `Default`.
- `octotablet 0.1`: posición en **píxeles lógicos** desde la esquina de la ventana; hay que
  multiplicar por `scale_factor()` para vello. `emulate_tool_from_mouse` viene en `true` y
  la herramienta emulada se distingue por `Type::Emulated`.
- `egui 0.35`: el trait `App` de eframe pide `fn ui(&mut self, ui: &mut Ui, ..)`, no `update`;
  `SidePanel` desapareció, es `egui::Panel::right(id)`. Solo importa si egui se usa como
  biblioteca: los paneles se anidan en un `Ui`.

## 3. Descartado, con la medición que lo descarta

| Qué | Por qué |
|---|---|
| **`eframe`** | Se queda con el bucle. Llama a `App::ui` dentro del repintado (`epi_integration.rs:295`) y su único otro punto, `App::logic`, seis líneas antes (`:289`), en la misma pasada. **No hay lugar para `pump()`.** Probado: se cuelga con `wgpu` y con `glow`, construyendo el Manager al inicio o diferido. |
| **`eframe 0.36` / `egui 0.36`** | Arrastra `wgpu 30.0.1` junto a la `29.0.4` de vello: dos `wgpu`, tipos incompatibles en silencio. |
| **Xilem 0.4 / masonry 0.4** (hoy) | Ancla `vello 0.6` / `kurbo 0.12` / `peniko 0.5` / `wgpu 26`. Al lado del nuestro: dos de cada. Y **no** resuelve la tableta: corre sobre el mismo `winit 0.30`, cero manejo de presión en masonry. Revisable cuando su vello alcance al nuestro. |
| **Presión vía `winit`** | `winit 0.30.13` no tiene eventos de stylus: cero ocurrencias de `Tablet`; `TouchpadPressure` es solo macOS. |
| **Qt** (como necesidad) | El único argumento fuerte era la tableta, y la tableta funciona sin él. Sigue siendo referencia de *qué* (OpenToonz), no de *cómo*. |
| **CEF + web** (Graphite) | Frontera de procesos y memoria; Graphite necesita un pipeline de tres texturas para componer Chromium con la escena. Lotte no tiene esa razón. |

Hipótesis que **no** eran la causa y no hay que volver a probar: servicio `WTabletServicePro`
caído, ritmo de repintado, Windows Ink apagado, wgpu alterando el apartamento COM,
`drag_and_drop` de winit (egui-winit solo lo aplica si es `Some`; el defecto es `true`).

## 4. Qué NO está verificado todavía

Escrito para que no se dé por hecho:

1. ~~**egui como biblioteca dentro de nuestro bucle**~~ ✅ **Verificado** (2026-09-07,
   `scratchpad/mockup`): barra de menús, panel de color y línea de tiempo redimensionables,
   cámara vello al centro; egui pinta encima del blit de vello en el mismo frame
   (`LoadOp::Load`). 4 trazos, 3820 poses, presión 0.0000–0.5125, `Responding: True`,
   cierre limpio. Una sola `wgpu`, un solo `winit`. Valoración del autor tras usarlo:
   "en general funciona bien". Esto **cierra la decisión A a favor de egui como biblioteca**.
1b. ✅ **Los cinco crates del motor contra el stack, en ejecución** (2026-09-07,
   `scratchpad/mockup` con `lotte-core/rig/timeline/render` por ruta): `FCurve` → `Transform2D`
   → `evaluate_all` → `VectorItem::glam_to_kurbo_affine` → `vello::Scene`, `ViewportCamera`
   para pan/zoom del lienzo, gizmos de pivote (§5) y visor de nodos (§1). Compiló al primer
   intento; `glam 0.32.1` y `kurbo 0.13.1` del workspace unifican con los del render
   (1 entrada cada uno). Confirmado por el autor tras usarlo. Un hallazgo: la convención de
   ejes (decisión F). Detalle en [`bitacora-tableta.md`](bitacora-tableta.md).
2. **El puente vello → textura → egui** (`register_native_texture`, `renderer.rs:771`).
   API verificada, sin correr. Solo hace falta si la vista de cámara vive *dentro* de un
   panel de egui; si egui solo pinta los paneles laterales y vello pinta directo al
   surface, no hace falta.
2b. ✅ **Paneles acoplables — verificado** (2026-09-08, `scratchpad/mockup`).
   **`egui_tiles 0.16.0`** (MIT OR Apache-2.0, de rerun-io, 4242 líneas; depende solo de
   `egui`, `ahash`, `itertools`, `log`, `serde`, todos ya en el árbol) es la única versión
   que resuelve con `egui 0.35` dejando **una** `egui` en el lock — la 0.15 y la 0.17 traen
   dos. Los cuatro paneles (nodos, cámara, color, timeline) son hojas de un árbol de 10
   tiles, cada una con pestaña arrastrable: se acoplan en cualquier borde, se apilan como
   pestañas hermanas, se redimensionan por el divisor y se cierran con la ×
   (`is_tab_closable` viene en `false`; hay que sobreescribirlo). La cámara vello vive
   *dentro* de un pane: su rect se remide cada frame y el dibujo se recorta con
   `push_layer`. Confirmado por el autor: "lo demás funciona". Costo asumido: **otra
   versión anclada en tándem**, `egui 0.35 ↔ egui_tiles 0.16`.
   Lo que sigue sin correr: **paneles flotantes en ventana propia** (viewports de egui
   sobre nuestro bucle) — `egui_tiles` acopla dentro de la ventana, no desprende.
3. **octotablet en producción**: 0.1.0, un solo autor, `IMarshal` a mano, un `// Fixme: unwrap`
   en la inicialización. Funciona; no está probado en duración ni con desconexión de la
   tableta. Sin backend para macOS ni X11 (Windows Ink y Wayland).
4. **Tilt**: el eje existe en la API; no se midió con el Intuos S.
5. **Rendimiento**: nada. Ningún benchmark en el workspace todavía (M0.2).

## 5. Decisiones que abre

Lo que este documento permite decidir sin volver a medir:

**A. Capa de paneles.** Tres opciones sobre el mismo bucle propio:

| | Costo ahora | Riesgo | Dónde te deja |
|---|---|---|---|
| **egui 0.35 como biblioteca** | bajo: widgets, color picker y layout hechos | §4.1 sin correr; egui congela vello y egui en pareja | Mockup en días; paneles inmediatos |
| **Todo pintado con vello/kurbo, sin toolkit** | alto: texto (parley), layout, widgets desde cero | ninguno de dependencia | Control total, ritmo lento |
| **Xilem, más adelante** | hoy imposible sin retroceder el stack | madurez 0.4 | Retenido, mismo linaje que vello |

Recomendación: **egui como biblioteca ahora**, pintando los tres paneles como superficies
propias (grilla de tiempo, visor, paleta) y dejando fino el pegamento del toolkit — que es
lo que OpenToonz tuvo que escribir igual bajo Qt (`CellArea::paintEvent`, 4356 líneas). Así
la migración a Xilem, si llega, es del marco y no de la aplicación.

**B. Adoptar octotablet 0.1 como dependencia.** Sí, con dos cláusulas: `pump()` solo en
`about_to_wait` (regla para el overlay), y una prueba de humo con hardware al subir de
versión. Alternativa si falla: Windows Ink directo vía `windows-rs` (`WM_POINTER*`), sin COM.

**C. Sexto crate.** Un `lotte-app` (o el nombre que decidas) con el `[[bin]]`, dueño del
bucle. Depende de los cinco; no rompe las dos reglas duras de capas. Cambia el hecho
"no hay aplicación" de `AGENTS.md`.

**D. Modelo del trazo con presión.** La prueba dibuja segmentos con `Stroke::new(1 + p·24)`.
Sirve para el mockup, **no** como formato: `patterns.md` §6 pide línea central + envolvente
de anchura editable, y eso obliga a construir el contorno y hacer `fill()`. Decisión de M3,
no de ahora — pero el mockup no debe fijar el formato por inercia.

**E. `eframe` fuera del vocabulario del proyecto.** Que no vuelva a aparecer en un brief.

**F. Convención de ejes del mundo — hoy no está fijada en ninguna spec, y se notó.** El
personaje salió cabeza abajo: el rig es Y-arriba por intención (`character_rig`, cabeza en
`(0, 80)` "sobre" el torso) y `ViewportCamera::world_to_screen_affine` no voltea Y. Su único
test es un ida-y-vuelta, ciego a la orientación.

| Opción | Qué cambia | Costo |
|---|---|---|
| **Mundo Y-arriba; la cámara voltea** (recomendada) | Un `−1` en Y dentro de `world_to_screen_affine`; la inversa sale sola | Una línea en `lotte-render` + **un test de orientación** (un punto por encima de `pan` cae más *arriba* en píxeles) + fijarlo en `architecture.md` |
| Mundo Y-abajo, como la pantalla | Renegociar el rig de ejemplo, los comentarios de la spec y la intuición del animador | Barato en código, caro para siempre |

OpenToonz hace lo primero: `SceneViewer::winToWorld` (`sceneviewer.cpp:1039`) niega Y en la
frontera ventana→mundo, y `flipY()` es una operación de vista. Graphite no dio
dato con la búsqueda hecha. La causa raíz es el vacío en la spec, no el bug: la decisión
tiene que quedar escrita además de codificada.

**G. Visor de nodos: pintado a mano, no con un crate.** Los tres crates de grafos para egui
(`egui-snarl`, `egui_graphs 0.32`, `egui_node_graph2 0.7`) traen un segundo `egui` al lock —
incompatibles con la 0.35. Pan, zoom, cables Bézier (`CubicBezierShape`), arrastre y
selección ya están hechos sobre el painter y verificados. Lo que falta es lo de diseño:
tipos de nodo (hoy solo existe `PegNode`; §1.1 pide Peg/Drawing/Deformer/Composite),
conexión por cable con rechazo visual de ciclos (`DagError::CycleDetected` ya existe).

## Procedencia

Todas las corridas, comandos, salidas crudas y las cinco hipótesis falsadas están en
la bitácora del laboratorio. Los programas: `tablet-spike` (eframe, el que
se colgaba), `crudo` (winit pelado, el experimento por contradicción) y `trazo` (la prueba
funcional). Ninguno es código del producto.
