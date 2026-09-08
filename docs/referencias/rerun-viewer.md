# Auditoría: visor de Rerun (crates/viewer)

- **Repo fuente:** `references/rerun/` (sparse checkout: solo `crates/viewer/` y el `Cargo.toml` raíz)
- **Commit:** `bc395cc561eb`
- **Licencia:** Apache-2.0 (dual MIT/Apache-2.0 según `Cargo.toml` raíz, `license = "MIT OR Apache-2.0"`)
- **Fecha del informe:** 2026-09-08
- **Estrategia de ingesta:** patrones y código portables con atribución Apache-2.0

Todas las líneas citadas abajo están medidas con `grep -n` sobre el árbol de trabajo en la ruta
indicada, sin editar nada bajo `references/` (solo lectura). No se ejecutó `cargo` en ningún
momento.

---

## 1. `re_time_panel/` (5618 líneas) — virtualización del panel de tiempo

**Medido:** `find crates/viewer/re_time_panel -name "*.rs" | xargs wc -l` → `5618 total` (coincide
con el enunciado). Archivos: `src/data_density_graph.rs`, `src/lib.rs`,
`src/recursive_chunks_per_timeline_subscriber.rs`, `src/streams_tree_data.rs`,
`src/time_axis.rs`, `src/time_control_ui.rs`, `src/time_panel.rs`, `src/time_selection_ui.rs`.

**No hay virtualización por filas en el sentido de "solo instanciar N widgets visibles"** (no hay
ningún `show_rows`/`show_rows_fixed_height` de egui). En cambio:

1. El árbol completo de entidades se dibuja dentro de un `egui::ScrollArea::vertical()`
   (`time_panel.rs:752`), con `.scroll_source(ScrollSource::MOUSE_WHEEL | ScrollSource::SCROLL_BAR)`
   (`time_panel.rs:754-757`, sin `ScrollSource::DRAG` porque el `ScrollArea` no debe robarle el
   drag al pan-and-zoom del área de tiempo) y auto-scroll manual con el botón central
   (`time_panel.rs:761-763`).
2. Cada fila (`show_entity`, `time_panel.rs:812`) SÍ se materializa como `ListItem`
   (`show_hierarchical_with_children`, `time_panel.rs:851-885`) — egui hace layout de la fila
   igual, colapsada o no. Lo que se **corta** es el trabajo caro post-layout:
   `let is_visible = ui.is_rect_visible(full_width_rect);` en `time_panel.rs:925` (fila de
   entidad) y `time_panel.rs:1113` (fila de componente). Solo si `is_visible` se consulta si la
   fila tiene datos y se llama a `data_density_graph_ui(...)` (`time_panel.rs:926-957`,
   `:1113-1140`). Es **culling por fila usando el rect visible de `egui::Ui`**, no un cálculo
   manual de índices contra el scroll offset.
3. Dentro de cada fila visible, `data_density_graph.rs` agrega **culling por rango de tiempo**:
   `build_density_graph` (`:576`) calcula `visible_time_range = time_ranges_ui.time_range_from_x_range(...)`
   (`:591-594`, con margen `MARGIN_X`) y lo usa para listar entradas no cargadas
   (`:598-607`) y para un `RangeQuery::new(*timeline, visible_time_range)` (`:619`) que solo trae
   los chunks relevantes vía `store.range_relevant_chunks(...)` (`:624-631`,
   `ChunkTrackingMode::Ignore` — comentario: *"Don't cause chunks to be downloaded just to show
   the density graph"*). Los puntos se acumulan en un histograma de buckets por pixel
   (`bucket_index_from_x`/`x_from_bucket_index`, `:121-135`), no un punto por evento.
4. También hay corte por rect al pintar los ticks de la regla (ver §2).

**Tipos/funciones clave con línea:**
- `TimePanel::tree_ui` — `time_panel.rs:740`
- `TimePanel::show_entity` — `time_panel.rs:812`
- `TimePanel::show_entity_contents` (fila de componente, no localizada por nombre exacto pero es
  la llamante de `time_panel.rs:1080-1144`)
- `ui.is_rect_visible(full_width_rect)` — `time_panel.rs:925`, `time_panel.rs:1113`
- `build_density_graph` — `data_density_graph.rs:576`
- `DensityGraphBuilder::new` — `data_density_graph.rs:824`

---

## 2. `re_time_ruler/` — la regla de tiempo

Archivos: `src/lib.rs` (24 líneas), `src/paint_gaps.rs` (158), `src/paint_ticks.rs` (227),
`src/time_ranges_ui.rs` (506). Total medido: `wc -l src/*.rs` → 915 líneas.

**Conversión tiempo↔píxel — `TimeRangesUi` (`time_ranges_ui.rs:62`):** no es un mapeo lineal
simple, es una lista de `Segment` (`:40-54`, uno por rango contiguo de datos) con compresión de
huecos (`gap_width`, `:28`; `GAP_EXPANSION_FRACTION`, `:142-146`) para que un salto sin datos no
ocupe espacio proporcional a su duración real. `TimeRangesUi::new(time_x_range, time_view,
time_ranges)` (`:100`) calcula `points_per_time` (`:121-126`) y expande cada segmento levemente
para facilitar seleccionar sus extremos (`:160-165`). Conversión tiempo→pixel:
`x_from_time_f32`/`x_from_time` (`:259`, `:263`). Pixel→tiempo: `time_from_x_f32`/`time_from_x_f64`
(`:316`, `:320`), con snapping en `snapped_time_from_x(ui, pointer_x)` (`:293`). Rango de tiempo
visible para un rango de x: `time_range_from_x_range` (`:346`, usado por
`data_density_graph.rs:591`, ver §1). Pan/zoom: `pan(delta_x)` (`:357`), `zoom_at(x, zoom_factor)`
(`:365`) — el caller decide cuándo llamarlos; doc en `lib.rs:14-16`: *"The crate has no opinion on
where the ruler sits on screen or how the user pans and zooms — callers wire those up themselves."*

**Espaciado de marcas según el zoom — `paint_ticks.rs`:** `paint_ticks` (`:98`) recibe
`next_time_step: fn(i64) -> i64` (`:105`, siguiente "escalón" natural de tiempo). Constante
`minimum_small_line_spacing = 4.0` px (`:135`). Bucle `while width_time / small_spacing_time >
max_small_lines { small_spacing_time = next_time_step(...) }` (`:156-160`) hasta que quepan en el
ancho visible; `medium_spacing_time`/`big_spacing_time` son el siguiente y siguiente-siguiente
escalón (`:161-162`). Intensidad de línea y color de texto se calculan con `remap_clamp` del
espaciado en píxeles (`line_strength_from_spacing`, `:138-143`) — las marcas se desvanecen al
zoomear en vez de aparecer de golpe. Recorte al zoom: comentario *"Clamp segment to the visible
portion to save CPU when zoomed in"* en `paint_ticks.rs:31` (función en `:7`).

**Cabezal (playhead) y scrub — en `re_time_panel/src/time_panel.rs`, no en `re_time_ruler`:**
`re_time_ruler` no dibuja el cabezal, solo expone `TimeRangesUi` y el pintado de ticks/huecos
(`lib.rs:19-23`). El cabezal vive en `TimePanel::time_marker_ui` (`time_panel.rs:1941`):
detecta hover/drag sobre el rect de streams (`:1958-1966`), convierte pixel→tiempo con
`snapped_time_from_x(ui, pointer_pos.x)` (`:1970`), y mueve el tiempo mientras el botón primario
está presionado —no solo al soltar— con
`i.pointer.primary_pressed() || i.pointer.primary_down() || i.pointer.primary_released()`
(`:1981`), empujando `TimeControlCommand::SetTimeClamped(...)` (`:1987-1989`). Pinta con
`ui.paint_time_cursor(...)` (`:2044`, extensión definida en `re_ui/src/ui_ext.rs:795`, fuera de
alcance). Zoom/pan del área completa (no del cabezal): `pan_and_zoom_interaction` (`:1819`) —
scroll horizontal (`input.smooth_scroll_delta.x`, `:1836`), pellizco (`input.zoom_delta_2d().x`,
`:1837`), drag derecho como zoom vertical (`zoom_factor *= (response.drag_delta().y *
0.01).exp();`, `:1849-1851`), drag con botón central como pan (`:1854-1857`); aplica con
`time_ranges_ui.pan(-delta_x)` / `time_ranges_ui.zoom_at(pointer_pos.x, zoom_factor)`.

---

## 3. `re_viewport_blueprint/` — persistencia del layout de `egui_tiles`

**No hay `serde` sobre `egui_tiles::Tree` en ningún lado del crate** (medido:
`grep -rn "serde\|Serialize\|Deserialize" re_viewport_blueprint/src/*.rs` no encontró impls de
serde para el árbol). El "blueprint" se modela como datos: cada contenedor y cada vista son
**entidades con componentes**, iguales en forma a los datos que Rerun loguea normalmente,
guardados en un `EntityDb` (`re_entity_db`) separado (el "blueprint store").

- `ContainerBlueprint` (`container.rs:26-35`) es *"la versión nativa de
  [`re_sdk_types::blueprint::archetypes::ContainerBlueprint`]"* (doc en `container.rs:17-23`):
  campos `id`, `container_kind: egui_tiles::ContainerKind`, `display_name`,
  `contents: Vec<Contents>`, `col_shares`/`row_shares: Vec<f32>`, `active_tab`, `visible`,
  `grid_columns: Option<u32>`.
- **Reconstrucción al abrir** — `ViewportBlueprint::from_db` (`viewport_blueprint.rs:81`): lee
  `RootContainer` de la entidad `VIEWPORT_PATH` vía `blueprint_engine.cache().latest_at(...)`
  (`:86-95`, consulta "latest-at" sobre el store de chunks, no deserialización de un blob);
  recorre contenedores con una pila (`container_ids_to_visit`, `:118-134`) cargando cada uno con
  `ContainerBlueprint::try_from_db(...)`; carga cada vista con `ViewBlueprint::try_from_db(...)`
  (`:139`); arma el `egui_tiles::Tree<ViewId>` con
  `build_tree_from_views_and_containers(views.values(), containers.values(), root_container)`
  (`:159-163`, función en `:1193`, `egui_tiles::Tree::empty("viewport_tree")` en `:1199`).
- **Guardado** — `ViewportBlueprint::save_to_blueprint_store` (`:934`): solo actúa si hubo
  comandos diferidos en el frame — *"No changes this frame - no need to save to blueprint
  store."* (`:939-941`) — **no se guarda cada frame**, solo cuando algo cambió. Antes de guardar
  corre `self.tree.simplify(&tree_simplification_options())` (`:956`, porque `egui_tiles` también
  simplifica en `tree.ui()` pero "eso es tarde" para lo persistido). `save_tree_as_containers`
  (`:826`) recorre `self.tree.tiles.iter()` (`:841`), mapea cada `TileId` a
  `Contents::{View,Container}`, detecta "trivial tabs" de un solo hijo que `egui_tiles` agrega
  automáticamente y no hace falta persistir (`:853-871`), limpia contenedores huérfanos para que
  el GC libere RAM (`:881-888`), y por cada contenedor restante construye un
  `ContainerBlueprint::from_egui_tiles_container(...)` (`:899-904`, definido en `container.rs:230`)
  que se guarda con `blueprint.save_to_blueprint_store(ctx)`. El root se guarda aparte:
  `ctx.save_blueprint_component(VIEWPORT_PATH.into(), &...descriptor_root_container(), &root_container)`
  (`:919-923`).
- **Formato final en disco/red:** no medible desde `crates/viewer` (la serialización a
  `.rbl`/Arrow/gRPC vive en `crates/store/re_entity_db` y `crates/store/re_log_encoding`, fuera
  del sparse checkout) — a verificar por el developer si hace falta ese detalle.

**`egui_tiles::Behavior` — implementado en `re_viewport`, no en `re_viewport_blueprint`:**
`re_viewport/src/viewport_ui.rs:410`: `impl<'a> egui_tiles::Behavior<ViewId> for TilesDelegate<'a, '_>`,
con `pane_ui` (`:411`), `tab_title_for_pane` (`:514`), `top_bar_right_ui` (`:613`),
`tab_bar_height` (`:754`), `simplification_options` (`:761`), `on_edit(edit_action:
egui_tiles::EditAction)` (`:767`) — el trait que Lotte deberá re-verificar contra egui_tiles 0.17
al saltar de versión (ver §4).

---

## 4. `Cargo.toml` raíz — versiones exactas y notas de migración

Medido con `grep -n` sobre `Cargo.toml` (raíz del checkout, fuera de `crates/viewer` pero
explícitamente permitido por la consigna):

| Paquete | Versión | Línea |
|---|---|---|
| `eframe` | `0.36.1` | `Cargo.toml:188` |
| `egui` | `0.36.1` | `Cargo.toml:198` |
| `egui_extras` | `0.36.1` | `Cargo.toml:199` |
| `egui_inspection` | `0.36.1` | `Cargo.toml:200` |
| `egui_kittest` | `0.36.1` | `Cargo.toml:201` |
| `egui-wgpu` | `0.36.1` | `Cargo.toml:202` |
| `emath` | `0.36.1` | `Cargo.toml:203` |
| `egui_commonmark` | `0.25.0` | `Cargo.toml:207` |
| `egui_dnd` | `0.17.0` | `Cargo.toml:208` |
| `egui_mcp` | `0.2.0` | `Cargo.toml:209` |
| `egui_plot` | `0.37.0` | `Cargo.toml:210` |
| `egui_table` | `0.10.0` | `Cargo.toml:211` |
| `egui_tiles` | `0.17.1` | `Cargo.toml:212` |
| `winit` | `0.30.13` | `Cargo.toml:487` |
| `wgpu` | `30.0` | `Cargo.toml:490` |

Esto es **exactamente** el par de versiones al que Lotte tiene que saltar (egui 0.36 / wgpu 30 /
egui_tiles 0.17), así que este checkout de Rerun ya es "destino", no "diario de migración": no
tiene código puente entre 0.35→0.36. Confirmado con
`grep -rln "0\.35\|0\.36\|migrat" crates/viewer --include=*.rs`: los tres hits que aparecieron
(`re_time_panel/src/time_axis.rs:89`, `re_viewer/src/version_check.rs:136,160`) son falsos
positivos — un factor `0.35` de interpolación y un string de versión de release de Rerun mismo
(`"0.36.3"`/`"you are running 0.35.0"`), no de `egui`/`wgpu`.

**No hay ningún `#[cfg(...)]` ni comentario en `crates/viewer` que distinga comportamiento por
versión de `egui`/`wgpu`/`egui_tiles`** (mismo grep, cero coincidencias reales). Lo único
"sensible a versión" que se puede citar con línea es la superficie del trait
`egui_tiles::Behavior<ViewId>` ya listada en §3 (`re_viewport/src/viewport_ui.rs:410-767`) — es
la lista concreta de métodos que Lotte deberá re-chequear contra el changelog de `egui_tiles`
0.16→0.17 al saltar de versión, porque es la única superficie de API de `egui_tiles` que este
código ejercita más allá de construir/leer el árbol (`Tree`, `Tiles`, `Container`, `ContainerKind`,
`GridLayout`, ver `container.rs` y `auto_layout.rs`, listados en la búsqueda de §3).

---

## 5. Integración egui + wgpu — ¿usa `eframe`?

**Sí, `re_viewer` usa `eframe`** — a diferencia de Lotte, que tiene su propio bucle de eventos.
`impl eframe::App for App` — `re_viewer/src/app/mod.rs:1254` (`fn save` `:1269`, `fn logic`
`:1304`, `fn ui` `:1309`); `App` construido desde `eframe::CreationContext` (`:195`, `:218`).
Campo `egui_renderer: Option<Arc<egui::epaint::mutex::RwLock<egui_wgpu::Renderer>>>`
(`app/mod.rs:90`) — Rerun guarda una referencia directa al `egui_wgpu::Renderer` que `eframe` crea.

**Cómo comparte el dispositivo wgpu entre `egui` y su propio renderer (`re_renderer`, equivalente
de `vello` en Lotte):** `customize_eframe_and_setup_renderer(cc: &eframe::CreationContext)`
(`re_viewer/src/lib.rs:256`) — si `cc.wgpu_render_state` existe (`:261`), toma
`render_state.{adapter,device.clone(),queue.clone(),target_format}` (`:267-273`) y construye
`re_renderer::RenderContext::new(...)`: **mismo `Device`/`Queue` que usa `egui-wgpu`**, no uno
separado. Ese `RenderContext` se guarda en
`render_state.renderer.write().callback_resources.insert(render_ctx)` (`:266-274`), el mecanismo
estándar de `egui_wgpu::CallbackResources`.

El puente real es un **paint callback dentro de un panel** (no una textura registrada):
`re_viewer_context/src/gpu_bridge/re_renderer_callback.rs`. `new_renderer_callback(view_builder,
viewport, clear_color) -> egui::PaintCallback` (línea 5) envuelve
`egui_wgpu::Callback::new_paint_callback(viewport, ReRendererCallback { .. })` (línea 10).
`impl egui_wgpu::CallbackTrait for ReRendererCallback` (línea 24): `fn prepare(...)` (línea 28)
saca el `RenderContext` de `paint_callback_resources` (línea 36) y llama a
`self.view_builder.lock().draw(ctx, self.clear_color)` (línea 43), devolviendo un
`wgpu::CommandBuffer` que `egui-wgpu` somete junto con los suyos. `fn paint(...)` (línea 53) hace
`render_pass.set_viewport(...)` con el rect del callback en píxeles (líneas 73-87, con comentario
explícito sobre por qué NO usan el clamp de `egui_wgpu` — tarjetas de grilla que se scrollean
fuera del viewport del SO) y compone con `self.view_builder.lock().composite(ctx, render_pass)`
(línea 89).

Conclusión: Rerun **no registra una textura externa** (`register_native_texture`) para contenido
3D — pinta dentro del mismo `wgpu::RenderPass` que usa `egui-wgpu`, en un callback scopeado al
rect del panel, compartiendo un único `Device`/`Queue`.

---

## Portable a Lotte

Contexto de Lotte: mockup con una timeline pintada a mano en egui (filas por peg, rombos de
keyframe, cabezal arrastrable, 96 frames) y un árbol `egui_tiles` de 4 paneles armado a mano en
código, sin persistencia.

**(a) Dope sheet con cientos de pistas:** no hace falta un widget de virtualización de filas tipo
`show_rows_fixed_height` para arrancar — con `is_rect_visible(row_rect)` (`time_panel.rs:925`)
alcanza para saltar el trabajo caro (rombos de keyframe) en filas fuera de pantalla, dejando que
egui siga haciendo layout de todas las filas dentro del `ScrollArea`. Es más simple que
virtualización real y ya escala a "cientos de pistas" en Rerun; si 96 frames × cientos de pistas
duele en el layout mismo (no solo el pintado), ahí sí migrar a `show_rows`. Portar también el
culling por rango de tiempo (§1.3, `data_density_graph.rs:591-594`): recortar contra el rango de
tiempo visible antes de iterar los keyframes de una pista, no después. Y
`ScrollSource::MOUSE_WHEEL | ScrollSource::SCROLL_BAR` sin `ScrollSource::DRAG` (`time_panel.rs:757`)
si el dope sheet de Lotte también quiere pan-and-zoom con el mismo drag que usa el `ScrollArea`.

**(b) Guardar/restaurar la distribución de `egui_tiles`:** el patrón completo de Rerun (loguear
cada contenedor como entidad-componente en un store propio, §3) es demasiado para un árbol de 4
paneles fijo — es una solución para un editor de layouts dinámico con decenas de contenedores
anidados por el usuario. **No portarlo entero.** Lo que vale copiar es la *forma*, no la solución:
separar "reconstruir el árbol" (`build_tree_from_views_and_containers`, `viewport_blueprint.rs:1193`)
de "leer y mapear cada nodo" (`ContainerBlueprint::try_from_db`). Para Lotte, acorde a
`contratos-solo-aditivos`, lo más simple es una struct propia serializable con `serde` directo
(JSON, como el resto de `lotte-format`) que espeje `egui_tiles::Tile`/`Container` a mano — un enum
chico (`Pane(PanelKind)` / `Tabs(Vec<Id>)` / `Linear{dir, children, shares}` / `Grid{...}`) en vez
de `#[derive(Serialize)]` sobre `egui_tiles::Tree` (no lo soporta de fábrica: Rerun tampoco lo
hace). Copiar el guard de "solo guardar si cambió algo" (`viewport_blueprint.rs:939-941`) y el
`tree.simplify(...)` antes de persistir (`:956`).

**(c) La regla de tiempo:** el tipo más directamente portable de todo el audit es `TimeRangesUi`
(§2, `time_ranges_ui.rs:62`) — su compresión de huecos es irrelevante si Lotte es fijo a 96 frames
sin huecos, pero el resto de la API (`x_from_time`, `time_from_x`, `pan`, `zoom_at`) es
exactamente la interfaz que necesita un cabezal arrastrable; vale copiar la firma aunque la
implementación interna empiece como mapeo lineal puro. El algoritmo de espaciado de marcas
(`paint_ticks.rs:98-166`: escalones naturales + desvanecido bajo 4px) es aplicable al ruler de 96
frames si algún día hay zoom variable. El patrón de scrub de `time_marker_ui` — mover el tiempo
mientras el botón está *presionado*, no solo al soltar (`time_panel.rs:1981`) — es el
comportamiento correcto para un cabezal arrastrable y vale copiarlo literal.

---

## Lo que NO existe o no encontré

- No hay ningún `show_rows`/`show_rows_fixed_height` de egui en `re_time_panel` — la
  "virtualización" es culling por visibilidad + culling por rango de tiempo, no reciclado de
  widgets.
- No hay `serde`/`Serialize`/`Deserialize` sobre `egui_tiles::Tree` en `re_viewport_blueprint` —
  el layout se modela como componentes de un store de entidades, no como blob serializado. El
  formato final en disco de ese store (Arrow/`.rbl`/protobuf) no es verificable desde
  `crates/viewer` — vive en `crates/store/*`, fuera del sparse checkout. A verificar por el
  developer si hace falta ese detalle.
- No encontré comentario, `#[cfg(...)]` ni nota de migración en `crates/viewer` sobre la
  transición de versiones anteriores de `egui`/`wgpu`/`egui_tiles` — este checkout ya está en el
  par de versiones destino de Lotte (egui 0.36.1, wgpu 30.0, egui_tiles 0.17.1): es el punto de
  llegada, no el diario de la migración.
- No encontré uso de `register_native_texture` ni texturas registradas para contenido 3D — la
  integración es 100% vía `egui_wgpu::CallbackTrait` (paint callback dentro del panel).
- No verificable desde este sparse checkout: cómo se serializa el blueprint store a disco/red
  (`crates/store/re_entity_db`, `crates/store/re_log_encoding` no existen aquí).

---

## Cómo se midió

```bash
# tamaño y archivos de re_time_panel
find crates/viewer/re_time_panel -name "*.rs" | xargs wc -l

# virtualización / culling
grep -n "ScrollArea|show_rows|show_viewport|clip_rect|visible_rect|row_height|scroll_offset|is_visible|viewport\b|cull|virtual" \
  crates/viewer/re_time_panel/src/time_panel.rs

grep -n "fn |visible_time_range|time_range|clip|skip|filter" \
  crates/viewer/re_time_panel/src/data_density_graph.rs

# regla de tiempo: TimeRangesUi
grep -rn "TimeRangesUi" crates/viewer/re_time_panel/src/*.rs
find crates/viewer/re_time_ruler -name "*.rs"
grep -n "^pub struct|^pub fn|^    pub fn|^impl|fn x_from_time|fn time_from_x|fn time_range_from_x_range" \
  crates/viewer/re_time_ruler/src/time_ranges_ui.rs
grep -n "^pub fn|^fn |spacing|step|zoom" crates/viewer/re_time_ruler/src/paint_ticks.rs

# blueprint / egui_tiles
grep -rn "egui_tiles|Tree<|serde|Serialize|Deserialize" crates/viewer/re_viewport_blueprint/src/*.rs
grep -n "fn |Tree<ViewId>|egui_tiles::Tree" crates/viewer/re_viewport_blueprint/src/viewport_blueprint.rs
grep -rln "impl.*Behavior|egui_tiles::Behavior" crates/viewer/re_viewport/src/*.rs
grep -n "impl.*Behavior|fn pane_ui|fn tab_title|fn top_bar_right_ui|fn on_edit|fn simplification_options|fn tab_bar_height" \
  crates/viewer/re_viewport/src/viewport_ui.rs

# versiones y migración
grep -n "^egui\|^egui-wgpu\|^egui_extras\|^egui_tiles\|^egui_kittest\|^wgpu \|^winit\|\"wgpu\"|\"egui\"" Cargo.toml
grep -rln "0\.35|0\.36|wgpu 29|wgpu 30|migrat" --include=*.rs crates/viewer

# integración egui+wgpu sin eframe (resultó: SÍ usa eframe)
grep -rln "egui_wgpu::Renderer|egui_wgpu::RenderState|PaintCallback|CallbackTrait|register_native_texture|paint_callback_resources" \
  --include=*.rs crates/viewer
grep -n "eframe|winit|egui_wgpu::Renderer|RenderState" crates/viewer/re_viewer/src/lib.rs crates/viewer/re_viewer/src/app/mod.rs
```

Todos los comandos anteriores se corrieron con el directorio de trabajo en
`references/rerun` (solo lectura), nunca `find /` ni `find C:\`/`find O:\` sobre la raíz
del disco.
