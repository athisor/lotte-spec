# Bitácora — presión de tableta y toolkit de interfaz

Laboratorio de medición para Lotte, escrito mientras ocurría. Las conclusiones pasaron a
[`stack-verificado.md`](stack-verificado.md); esto es la evidencia cruda.

**Fecha:** 2026-09-07
**Pregunta original:** ¿podemos levantar un mockup de ventanas con vista de cámara,
widget lateral de color y línea de tiempo abajo, y funciona la presión del lápiz?

---

## Entorno medido

| | |
|---|---|
| SO | Windows 10 Pro 19045 |
| Tableta | Wacom Intuos S |
| `TabletInputService` | Running |
| `WTabletServicePro` | Stopped al inicio → **Running** tras reiniciar |
| Windows Ink (Wacom Center) | **ON** (verificado en la interfaz del driver) |
| `Wintab32.dll` | presente en System32 y SysWOW64 |

Versiones en juego: `vello 0.10.0` · `wgpu 29.0.4` · `winit 0.30.13` · `eframe/egui 0.35.0`
· `kurbo 0.13.1` · `peniko 0.6.1` · `octotablet 0.1.0` · `raw-window-handle 0.6.2`.

---

## Resolución de versiones

El toolkit ancla `wgpu` igual que vello, así que hay que alinearlos o quedan dos en el árbol.

| Combinación | Entradas de `wgpu` |
|---|---|
| `vello 0.10` + `eframe 0.33` | 2 — `27.0.1` y `29.0.4` |
| `vello 0.10` + `eframe 0.36` (la última publicada) | 2 — `29.0.4` y `30.0.1` |
| `vello 0.10` + `eframe 0.34` | **1** — `29.0.4` |
| `vello 0.10` + `eframe 0.35` | **1** — `29.0.4` |

Con `octotablet 0.1` sumado: 1 wgpu · 1 vello · 1 kurbo · 1 peniko · 1 winit ·
1 raw-window-handle. `cargo check` exit 0 en 17.75s.

**La última versión publicada de eframe es la que rompe la alineación.** No es un
descuido elegir la 0.35.

## API verificada en el código, no en la documentación

| Qué | Dónde |
|---|---|
| `vello::Renderer::render_to_texture` | `vello-0.10.0/src/lib.rs:474` |
| `eframe::Frame: HasWindowHandle` | `eframe-0.35.0/src/epi.rs:695` |
| `eframe::Frame: HasDisplayHandle` | `eframe-0.35.0/src/epi.rs:704` |
| `egui_wgpu::Renderer::register_native_texture` | `egui-wgpu-0.35.0/src/renderer.rs:771` |
| `octotablet::Builder::build_shared` / `build_raw` | `octotablet-0.1.0/src/builder.rs:68` / `:92` |
| `kurbo::fit_to_bezpath` / `fit_to_bezpath_opt` | `kurbo-0.13.1/src/fit.rs:165` / `:564` |

**`winit 0.30.13` no tiene eventos de tableta:** cero ocurrencias de `Tablet` en su
código; `WindowEvent` no trae ninguna variante de stylus. `TouchpadPressure` está
documentado como *"only supported on Apple forcetouch-capable macbooks"*. De ahí que
haga falta `octotablet` como capa aparte.

**Cambio de API en egui/eframe 0.35:** el trait `App` pide `fn ui(&mut self, ui: &mut Ui, ...)`,
no `update`; `SidePanel` ya no existe (ahora `egui::Panel::right(id)`); los paneles se
anidan en un `Ui` en vez de colgar del `Context`.

---

## Cronología de pruebas

### 1. Laboratorio con eframe + wgpu

el laboratorio `tablet-spike` — eframe + octotablet, dibujo con el painter de egui.
Deliberadamente **sin vello**: mezclar el puente vello→egui con esta prueba habría hecho
imposible saber cuál de los dos falló.

- Primera corrida: `exit code 0xcfffffff`, sin salida.
- Corridas siguientes: `[build] conexion a la tableta abierta` — el `Manager` **sí** se
  construye. Después la ventana pinta, el **mouse funciona sin colgar**, y al **apoyar el
  lápiz se cuelga**: `Responding: False`, CPU 0.64, 32 hilos (bloqueado, no girando).
- Con el lápiz por encima: no aparece ninguna tableta, no llega ninguna pose.

### 2. Control: eframe solo, sin octotablet

`scratchpad/solo-eframe` — eframe 0.35 + wgpu, sin tableta.

**Resultado: `Responding: True`, 1260+ frames pintados.** eframe y wgpu están sanos en
esta máquina. Quedan exonerados.

### 3. Corte decisivo: octotablet sobre winit pelado

`scratchpad/crudo` — winit 0.30 + octotablet, sin eframe, sin egui, sin wgpu, sin dibujo.
Construye el `Manager` con `build_shared(&Arc<Window>)` dentro de `resumed()`.

```
[build] conexion abierta
[evt] In
[evt] Down
[resumen] poses=282  presion 0.0000 .. 0.5095
[evt] Up
[evt] Down
[resumen] poses=417  presion 0.0000 .. 0.5989
[evt] Up
[evt] Out
[resumen] poses=535  presion 0.0000 .. 0.5989
```

`Responding: True`. **535 poses y presión real, continua, variable.** No un 1.0 binario.

**Este es el resultado que responde la pregunta original: la presión de la Wacom funciona.**

### 4. ¿Es wgpu el culpable? — eframe con backend `glow`

Mismo laboratorio, única diferencia `features = ["glow"]` en vez de `["wgpu"]`.

**Se cuelga igual.** `Responding: False`, CPU 0.56, 10 hilos.

**wgpu queda exonerado.** El backend gráfico no es la causa.

---

## Hipótesis falsadas

Las cuatro se escriben porque descartarlas costó tiempo y evita repetirlo.

| # | Hipótesis | Cómo se refutó |
|---|---|---|
| 1 | El servicio `WTabletServicePro` caído | Se levantó tras reiniciar; sigue colgando |
| 2 | Repintado a máxima tasa saturando el mutex de `pump()` | Se acotó a 60 Hz como el ejemplo del autor; sin cambio |
| 3 | "Usar Windows Ink" apagado en el driver Wacom | Está **ON**, verificado en Wacom Center |
| 4 | `wgpu` inicializando COM y cambiando el apartamento del hilo | Con `glow` se cuelga igual |

---

## Estado

**Funciona:** la tableta, el driver, Windows Ink, octotablet sobre winit pelado, eframe
por sí solo, y la alineación de versiones del stack completo.

**No funciona:** octotablet **dentro de eframe**. Se cuelga al llegar entrada real del
lápiz, con cualquiera de los dos backends gráficos.

**Dónde está el bloqueo, con lo medido hasta acá:** octotablet registra un plugin
*asíncrono* de RealTimeStylus (`AddStylusAsyncPlugin`, `ink/mod.rs:690`) implementado con
un `IMarshal` escrito a mano (`ink/mod.rs:557`). Los callbacks toman un
`std::sync::Mutex` compartido —una docena de sitios en `ink/com_impl.rs`— y `pump()` toma
el mismo mutex (`ink/mod.rs:772`). Un `std::sync::Mutex` no es reentrante. En todo el
crate **no hay una sola llamada a `CoInitializeEx`**: el apartamento COM del hilo lo
define quien lo creó.

Esto es *dónde*, por lectura del código. No hay traza de depurador, así que el mecanismo
exacto es inferencia.

**La integración no es el problema:** `build_raw(context)` + `pump()` en el bucle es
exactamente lo que hace el ejemplo `eframe-viewer` del propio autor del crate.

---

## Defectos de esta propia instrumentación

Se anotan porque invalidan parte de lo que creí leer.

1. **`println!` hacia un archivo redirigido va en bloques, no por línea.** Al colgarse el
   proceso (y más al matarlo) el buffer se pierde. En `crudo` no se notó porque generaba
   salida suficiente para vaciarlo varias veces; en el laboratorio con eframe, los eventos
   pueden haberse producido y haber quedado en el buffer. **Falta `flush()` tras cada
   `println!`.**
2. **`let _ = std::fs::write(...)` descarta el error.** No se puede distinguir "no se llegó
   a ejecutar" de "se ejecutó y falló".

**Consecuencia:** que `resultado.txt` no exista **no es evidencia** de en qué momento se
colgó. Esa conclusión queda retirada hasta reinstrumentar.

---

## Corrección de una afirmación anterior

Se había concluido que la presión era "una capa lateral, no una propiedad del toolkit", y
que eso sacaba a Qt de la conversación. Se apoyaba en que los tipos encajaban y compilaba
limpio — con la salvedad explícita de que no se había probado contra hardware real.

**Compila y no funciona dentro de eframe.** La conclusión era demasiado fuerte. Lo que
sobrevive, ya medido: la presión funciona sobre winit pelado, así que la capa *puede*
existir; lo que no está resuelto es su convivencia con el toolkit.

---

### 5. Construcción diferida al frame 30

Se movió la construcción del `Manager` fuera del arranque de eframe, al frame 30, con la
ventana ya activa. **Cambió algo importante:**

```
tabletas: 4
  - \\.\DISPLAY1
  - Intuos S
  - Intuos S
  - \??\Microsoft HID RID\000D_0002\1
```

Antes reportaba **0** tabletas. El `Manager` construido durante el arranque de eframe
nacía ciego. Pero seguía colgándose al apoyar el lápiz, con `herramientas: 0`.

Hipótesis 5 —el apartamento COM alterado por `drag_and_drop`— también se descartó:
`egui-winit` solo llama a `with_drag_and_drop` si el valor es `Some`
(`egui-winit-0.35.0/src/lib.rs:2140`), el defecto de egui es `None`, y el de winit es
`true` (`winit-0.30.13/src/platform_impl/windows/mod.rs:49`). **Los dos casos están en
STA.**

### 6. Experimento decisivo — mover `pump()` dentro del repintado

Se tomó `crudo`, el programa que **funciona**, y se cambió **una sola cosa**: `pump()`
pasó de `about_to_wait` a `WindowEvent::RedrawRequested`. Todo lo demás idéntico —
`build_shared(&Arc<Window>)`, construcción en `resumed()`, sin eframe, sin egui, sin wgpu.

| `pump()` desde | Resultado |
|---|---|
| `about_to_wait` (winit **entre** mensajes) | 535 poses, presión 0.0000–0.5989, `Responding: True` |
| `RedrawRequested` (**dentro** del WndProc) | `Responding: False`, CPU 0.06, ni un latido de 1 s |

---

## Conclusión

**La causa es *desde dónde* se llama a `pump()`, no qué biblioteca se use.**

Un apartamento COM de un solo hilo (STA) solo atiende llamadas marshaladas mientras está
bombeando mensajes. Dentro de un manejador de ventana no lo está, y el plugin asíncrono
de RealTimeStylus de octotablet se traba ahí.

eframe ejecuta `App::ui` dentro del repintado (`epi_integration.rs:295`), y su único otro
punto de entrada, `App::logic`, se invoca seis líneas antes (`:289`) en la misma pasada.
**eframe no expone ningún punto fuera del WndProc**, así que octotablet no puede convivir
con `eframe::run_native`.

Quedan exonerados, todos por medición: la tableta, el driver Wacom, Windows Ink,
octotablet en sí, egui, wgpu, el backend gráfico y la alineación de versiones.

## Consecuencia de arquitectura

La salida no es cambiar de toolkit sino **no delegar el bucle de eventos**: usar `egui` +
`egui-winit` + `egui-wgpu` como biblioteca dentro de un bucle winit propio, en vez de
`eframe`, que se queda con el bucle. Con eso:

- `pump()` de la tableta va en `about_to_wait` — la posición que está probada que funciona.
- egui se dibuja en `RedrawRequested`.
- Se controla directamente el `wgpu::Device`, que hace falta igual para que vello pinte la
  vista de cámara.

Es justo lo que ya describen los issues 7, 8 y 9 de M2 del plan: ventana winit propia,
wgpu propio, vello propio. La conclusión de este laboratorio es que **eframe se descarta y
egui se conserva**, y que el mockup se construye sobre bucle propio desde el principio.

**Nada de esto bloquea el mockup:** los tres paneles, la vista de cámara y la línea de
tiempo no necesitan presión.

---

## Prueba funcional — trazo con presión sobre el stack real

`scratchpad/trazo`: **winit 0.30 (bucle propio) + wgpu 29.0.4 + vello 0.10 + octotablet 0.1**,
armado según la conclusión: `pump()` en `about_to_wait`, dibujo en `RedrawRequested`,
`build_shared(&Arc<Window>)` con el bucle vivo. Sin eframe. Una sola `wgpu` en el lock.

```
[gpu] adaptador: NVIDIA GeForce RTX 5060 Ti
[gpu] vello listo
[tableta] conexion abierta
[evt] In
[evt] Down
[resumen] poses=197 trazos=0 presion 0.0000 .. 0.5286
[evt] Up  puntos_en_el_trazo=187
[evt] Down
[resumen] poses=600 trazos=1 presion 0.0000 .. 0.5737
[evt] Up  puntos_en_el_trazo=471
[resumen] poses=867 trazos=2 presion 0.0000 .. 0.5737
```

`Responding: True` durante todo el uso. **Dos trazos dibujados con vello, 187 y 471 puntos,
ancho modulado por presión real (0.0000–0.5737).** Cada segmento se traza con
`Scene::stroke` y `Stroke::new(1 + p·24)` con remates redondos; la presión en vivo se ve en
una barra al pie.

Esto cierra la pregunta original con un sí medido sobre el mismo stack del proyecto, y
demuestra que la arquitectura de M2 —bucle propio, wgpu propio, vello propio— absorbe la
tableta sin fricción.

## Mockup de distribución — egui como biblioteca sobre el mismo bucle

`scratchpad/mockup`: `trazo` + **`egui` / `egui-winit` / `egui-wgpu` 0.35** como biblioteca
(sin eframe). Barra de menús arriba, panel de color a la derecha, línea de tiempo abajo
(ambos redimensionables arrastrando el borde, con mínimo), cámara vello al centro. Por
frame: `Context::run_ui` → vello `render_to_texture` → blit al surface → `egui_wgpu::Renderer`
pinta encima con `LoadOp::Load`. Una sola `wgpu`, un solo `winit` en el lock.

```
[gpu] vello + egui listos
[tableta] conexion abierta
lienzo=[[0.0 22.0] - [1120.0 670.0]]   → maximizada: [[0.0 22.0] - [1640.0 827.0]]
[salida] limpia. trazos=4 poses=3820 presion 0.0000 .. 0.5125
```

`Responding: True` toda la sesión, cierre limpio. El lápiz pinta solo dentro del rect de
cámara que egui mide cada frame; sobre los paneles opera los widgets. Valoración del autor
tras usarlo: **"en general funciona bien"** (2026-09-07). Con esto la §4.1 de
`docs/stack-verificado.md` pasa de "sin correr" a verificada.

## El motor adentro del mockup — los cinco crates contra el stack, en ejecución

`scratchpad/mockup` + `lotte-core`, `lotte-rig`, `lotte-timeline`, `lotte-render` por ruta.
Rig de 7 pegs y 6 `VectorItem`; dos `FCurve` (brazo y antebrazo) y una lineal (torso)
evaluadas en el frame del cabezal; `set_local_transform` → `evaluate_all()` →
`glam_to_kurbo_affine` → `vello::Scene`; `ViewportCamera` del crate para pan y zoom del
lienzo; gizmos de pivote y alambres (§5); visor de nodos pintado a mano (§1) con pan, zoom,
cables Bézier, arrastre y selección; Ctrl+Z / Ctrl+Shift+Z / Supr con pila de rehacer.

```
[motor] rig con 7 pegs, 6 items
frame_tl=0   rot_brazo=0.500  brazo_global_t=(-33.5,118.4)
frame_tl=38  rot_brazo=1.637  brazo_global_t=(-27.9,120.8)   ← la curva del TORSO propagada al hijo
Cargo.lock: glam 1 · kurbo 1 · wgpu 1 · winit 1 · egui 1
[salida] limpia. frames=15219  (4 × [edicion] deshacer)
```

Compiló al primer intento con los cuatro crates del motor: **las versiones del workspace
(`glam 0.32.1`, `kurbo 0.13.1`) y las del stack de render unifican sin duplicados.**
Confirmado por el autor tras usarlo (2026-09-07): visor de nodos con zoom y pan, play,
deshacer, pan/zoom del lienzo — "funciona". El saludo se ve raro, pero es autoría de la
curva, no motor.

**Hallazgo — convención de ejes.** El personaje salió **cabeza abajo**. Causa medida:
`ViewportCamera::world_to_screen_affine` es `T(centro)·R·S(zoom)·T(−pan)` sin volteo de Y;
el rig es Y-arriba por intención explícita (`character_rig`: cabeza en `(0, 80)` "sobre"
el torso); y **ninguna spec fija la convención** (cero menciones en `architecture.md`,
`multiplane-parallax.md`, `ui-widgets.md`, `prd.md`). El único test de la cámara es un
ida-y-vuelta, ciego a la orientación. OpenToonz voltea en el visor
(`SceneViewer::winToWorld`, `sceneviewer.cpp:1039`: `-pos.y() + height()/2`): mundo Y-arriba,
la frontera con la pantalla invierte. El laboratorio lleva el volteo explícito y marcado
provisorio; **el crate no se tocó** — es decisión del autor (ver `stack-verificado.md` §5.F).

**Hallazgo — crates de grafos de nodos.** `egui-snarl`, `egui_graphs 0.32` y
`egui_node_graph2 0.7` resuelven con `egui 0.35` pero los tres traen **un segundo egui**
al lock (2 entradas): ninguno está en 0.35 y un `Ui` de uno no es el `Ui` del otro. El
visor se pinta a mano sobre el painter de egui — como el schematic de OpenToonz.

## Paneles acoplables — egui_tiles 0.16 sobre el mismo bucle

`scratchpad/mockup` + `egui_tiles = "0.16"`. Los cuatro paneles pasan a ser hojas de un
`egui_tiles::Tree<Pane>`: cada una dentro de su propio contenedor de pestañas para tener
cabecera arrastrable (`SimplificationOptions { all_panes_must_have_tabs: true }`), con
proporciones iniciales por `Linear::shares.set_share`. `Behavior::pane_ui` despacha a las
mismas funciones de pintado de antes; el árbol se saca de `Panel` con `mem::replace` un
instante para poder prestar el resto de `Panel` al `Behavior`.

```
[gpu] vello + egui + egui_tiles listos
[motor] rig con 7 pegs, 6 items · arbol con 10 tiles
lienzo=[[300.6 46.0] - [1222.4 679.8]]   ← la camara, acoplada entre nodos y color
egui en el lock: 1 · egui_tiles 0.16.0
```

`Responding: True`. Confirmado por el autor (2026-09-08): arrastrar pestañas, acoplar,
redimensionar por el divisor — "lo demás funciona". Lo único que faltó fue la × de cierre:
`is_tab_closable` viene en `false` por defecto; se sobreescribió a `true` y `on_tab_close`
registra el cierre. **Ventanas → Restablecer distribución** reconstruye el árbol.

Detalles que costaron una compilación: `set_share` vive en `Linear::shares`, no en
`Linear`; y `Scene::push_layer` en vello 0.10 toma cinco argumentos
(`estilo, mezcla, alfa, transform, forma`). El recorte de la cámara a su pane se hace con
esa capa; el fondo blanco solo se pinta en el rect de la cámara y el `base_color` de vello
pasa a gris oscuro para lo que ningún panel cubre.

Notas de API: `wgpu 29` exige `multiview_mask: None` en `RenderPassDescriptor`;
`RenderPass::forget_lifetime()` para el `RenderPass<'static>` que pide egui-wgpu;
`egui::MenuBar::new().ui(..)` + `ui.menu_button(..)` para la barra; `egui::Panel::{top,right,bottom}`
con `.resizable(true).default_size(..).min_size(..)`. Nota de API: en `wgpu 29`, `Surface::get_current_texture` ya no
devuelve `Result` sino el enum `CurrentSurfaceTexture` (`Success`/`Suboptimal`/`Outdated`/
`Timeout`/`Occluded`/…).

## Spike de herramientas — tema oscuro, caja de herramientas y cuerpo completo (2026-09-08, tarde)

Sobre el mismo mockup (`el mockup de laboratorio, 1 621 líneas`), cinco iteraciones con el autor probando cada una. Lo que
quedó verificado:

- **Tema oscuro forzado.** egui seguía el tema de Windows (claro) porque `egui_winit::State::new`
  recibía `win.theme()`. Se pasa `Some(winit::window::Theme::Dark)`, se fija `Context::set_theme(Dark)`
  y una paleta propia de grises (paneles 46, ventanas 40, fondos 30, widgets 56–88). El papel de la
  cámara pasa a gris medio (66,66,70) y la tinta por defecto a clara. Nota de API: el `Theme` que
  pide `State::new` es el de **winit**, no el de egui.
- **Tipografía de iconos.** Los glifos Unicode sueltos (⬚, 🖌, ⌫) salían como cuadrados: egui trae
  Ubuntu-Light y un subconjunto de Noto Emoji. `egui-phosphor 0.13.0` (MIT) declara `egui = "0.35"` y
  deja **una** `egui` en el lock; se registra con `add_to_fonts(&mut FontDefinitions::default(),
  Variant::Regular)` y los iconos son constantes con nombre (`CURSOR`, `SELECTION`, `PAINT_BRUSH`,
  `PENCIL_SIMPLE`, `ERASER`). Nota de API: `SelectableLabel` ya no existe en egui 0.35; es
  `Button::selectable`. `Panel::left(..).exact_size(..)` (no `exact_width`).
- **Caja de herramientas a dos columnas**, 70 px exactos (4 + 30 + 2 + 30 + 4): `egui::Grid`
  repartía las columnas con aire; filas `horizontal` con `item_spacing = 2` y `Frame::inner_margin(4)`
  dan el resultado. Cinco herramientas con atajo: flecha (V), marco (M), pincel (B), lápiz de grosor
  fijo (P), goma (E).
- **Selección y movimiento de pegs.** Hit-test del pivote más cercano en píxeles físicos (radio 12);
  el delta del arrastre se lleva al espacio local del **padre** con `globales[padre].inverse()
  .transform_vector2(delta)`. Medido en sesión: distancias de acierto entre 0.6 y 9.7 px. Marco
  rectangular para selección múltiple; de un conjunto se mueven solo los pegs sin ancestro
  seleccionado. Hallazgo de UX del autor (log: dos marcos de 9 pegs seguidos de cuatro marcos vacíos
  y ningún movimiento): arrastrar con el marco sobre un pivote seleccionado tiene que **mover** el
  conjunto, no abrir otro marco. Corregido.
- **`reposo ⊕ animación(t)` en el motor del mockup** (D-Inst en miniatura): `Motor` guarda
  `reposo: Vec<Transform2D>`; cada frame parte del reposo, aplica las curvas y suma los
  desplazamientos puestos en modo Animar. Modo Editar rig escribe en el reposo; **Reset animación**
  vacía los desplazamientos y conserva los arreglos. Menú *Modo* con las dos opciones.
- **Cuerpo completo animado**: 14 pegs (pelvis, muslos, pantorrillas, pies) y 8 `FCurve` reales,
  incluidas dos de **posición** (balanceo de pelvis en x, rebote del root en y); la pierna derecha
  lee las mismas curvas medio ciclo después, en espejo. 104 578 frames en la última sesión sin caídas.
- **Historial de ediciones con inverso** (D-Undo en miniatura) para los trazos: `Agregar(trazo)` y
  `Borrar([(índice, trazo)…])`; la goma borra varios trazos como **una** edición y Ctrl+Z los
  restaura en su índice original; "Borrar todo" es una edición más.

Lo que **no** se probó en esta tanda: la presión del lápiz con las herramientas nuevas (el autor usó
mouse: `presion inf .. -inf` en los cuatro cierres), y el lápiz de grosor fijo con tableta. Nada de
esto tiene tests: es laboratorio, y es exactamente lo que A1, F1, H1 y H2 del hito 1 convierten en
código con gate.
