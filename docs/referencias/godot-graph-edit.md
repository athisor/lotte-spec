# Auditoria: GraphEdit / GraphNode de Godot (visor de grafos)

- **Repo**: Godot Engine (sparse checkout, solo `scene/gui/`)
- **Commit**: `6a0f6f32cfb2`
- **Licencia**: MIT
- **Fecha**: 2026-09-08
- **Estrategia de ingesta**: traduccion C++ -> Rust con atribucion MIT
- **Archivos leidos** (solo lectura, `references/godot/scene/gui/`): `graph_edit.cpp` (3384 lineas), `graph_edit.h`, `graph_node.cpp` (1378 lineas), `graph_node.h`, `graph_edit.compat.inc`

---

## 1. Geometria del cable

`GraphEdit::get_connection_line` (`graph_edit.cpp:1526`) recibe `p_from`/`p_to` ya escalados por
zoom (los llamadores hacen `pos * zoom` antes de invocarla, p.ej. `graph_edit.cpp:1655` y `:1786`).
Primero intenta un override por GDExtension (`GDVIRTUAL_CALL(_get_connection_line, ...)`,
`:1528`); si no hay override, construye la curva:

```cpp
float x_diff = (p_to.x - p_from.x);
float cp_offset = x_diff * lines_curvature;
if (x_diff < 0) { cp_offset *= -1; }
```

Los puntos de control son horizontales puros: el punto de salida del nodo `from` es
`p_from + (cp_offset, 0)` y el punto de entrada del nodo `to` es `p_to + (-cp_offset, 0)`, via
`Curve2D::set_point_out(0, Vector2(cp_offset, 0))` y `set_point_in(1, Vector2(-cp_offset, 0))`
(`:1538-1542`). Es una cubica de Bezier con tangentes horizontales en ambos extremos, ancho
proporcional a la distancia horizontal entre puertos (`lines_curvature` es el factor, propiedad
`connection_lines_curvature`, `:3095`) y con el signo del offset invertido cuando `to` esta a la
izquierda de `from` (evita que el cable se pliegue sobre si mismo cuando el nodo destino esta
detras). La curva se tesela con `MAX_CONNECTION_LINE_CURVE_TESSELATION_STAGES = 5` (`:57`, usado
en `:1545`) si `lines_curvature > 0`, o con una sola etapa (linea recta) si es 0 (`:1546-1547`).

**Escala con zoom**: la curvatura escala con el zoom *indirectamente*, porque `x_diff` se calcula
sobre posiciones ya multiplicadas por `zoom` en el llamador — no hay un termino de zoom explicito
dentro de `get_connection_line`. El grosor es otra historia: `_get_shader_line_width()`
(`graph_edit.cpp:2638`) devuelve `lines_thickness * theme_cache.base_scale + 4.0` — depende de la
escala de tema (`base_scale`, tipicamente ligada al DPI de la UI), **no** del zoom del grafo. El
shader de la linea (`init_shaders`, `:222-244`) dibuja el rim/antialiasing en espacio UV
normalizado, por lo que el "grosor visual" en pantalla si crece con el zoom porque la geometria
completa (incluida la malla generada por `Line2D`) esta en el espacio ya escalado por zoom, aunque
el parametro `line_width` que entra al shader no lleve el zoom multiplicado explicitamente.

`_update_connections` (`:1617`) hace tres cosas por frame de redraw:
1. Recorre `connections`, y para cada una con cache `dirty` recalcula `from_pos`/`to_pos` a partir
   de `get_output_port_position`/`get_input_port_position` de los `GraphNode` (`:1641-1642`),
   regenera la polilinea con `get_connection_line` (`:1655`) y recalcula el AABB de la conexion
   sumando `lines_thickness * 0.5` de margen (`:1675`).
2. Poda conexiones "muertas" (nodo origen o destino ya no existe y `keep_alive` es falso,
   `:1618-1635`, `:1715-1723`).
3. Descarta actualizar/dibujar conexiones fuera del viewport (`:1680-1685`) y aplica tinte de
   actividad (`theme_cache.activity_color`, `:1690-1693`) y de hover
   (`theme_cache.connection_hover_tint_color`, mas un 1.0 + `connection_hover_thickness/100`
   de grosor extra, `:1695-1706`).

## 2. Snapping y puertos magneticos

No hay "puerto magnetico" con radio de captura configurable como concepto separado del snapping de
grilla — son dos sistemas distintos y los nombres reales son otros:

- **Snapping de grilla** (`snapping_enabled`, `snapping_distance`, `snapping_distance_scale`,
  `graph_edit.cpp:2709-2741`) afecta el *movimiento de nodos* (`:983-984`) y el *resize*
  (`:598-599`), no el enganche de cables. Se invierte con CTRL/CMD_OR_CTRL sostenido (`:598`,
  `:983`, `:2017`).
- **Deteccion de puerto bajo el cursor durante un arrastre de cable**: los nombres reales son
  `is_in_input_hotzone`/`is_in_output_hotzone` (`:1463`, `:1473`) que delegan en
  `is_in_port_hotzone` (`:1491`). Esta funcion define un rectangulo de captura, no un radio
  circular:
  ```cpp
  Rect2 hotzone = Rect2(
      p_pos.x - (p_left ? theme_cache.port_hotzone_outer_extent : theme_cache.port_hotzone_inner_extent),
      p_pos.y - p_port_size.height / 2.0,
      theme_cache.port_hotzone_inner_extent + theme_cache.port_hotzone_outer_extent,
      p_port_size.height);
  ```
  (`:1492-1496`). El ancho es asimetrico segun el lado: para un puerto de entrada (`p_left =
  true`) el margen "hacia afuera del nodo" es `port_hotzone_outer_extent` y "hacia adentro"
  `port_hotzone_inner_extent` (viceversa para output); la altura es la del icono de puerto. Son
  valores de theme (`BIND_THEME_ITEM`), no encontre el default numerico en este archivo.
  Ademas descarta el hit si hay un `Control` clickeable de otro nodo encima
  (`_check_clickable_control`, `:1502-1519`).
- **Prioridad cuando hay varios puertos superpuestos**: por orden de iteracion, no por distancia.
  El barrido de nodos va en orden inverso de hijos (`:1210`, `:2045` — "el ultimo hijo anadido
  gana primero") y dentro de cada nodo se prueban primero los puertos de salida y luego los de
  entrada (`:1218-1420`). No hay comparacion de distancia entre candidatos: el primer hotzone que
  matchea gana.
- Para clics (no arrastre) sobre una conexion existente, el radio de captura si es explicito:
  `get_closest_connection_at_point(p_point, p_max_distance)` (`:1551`, default `4.0` via
  `DEFVAL(4.0)` en el bind, `:2982`) compara la distancia punto-segmento
  (`Geometry2D::get_distance_to_segment`) contra `lines_thickness * 0.5 + p_max_distance`
  (`:1564`) y se queda con la conexion mas cercana (si hay varias, gana la de menor distancia,
  `:1565-1566` — aca si hay prioridad por distancia).
- Distancia minima de arrastre para considerar el drag una conexion valida:
  `MIN_DRAG_DISTANCE_FOR_VALID_CONNECTION = 20` (`:56`, usado en `:1338`).

## 3. Validacion y rechazo

`is_valid_connection_type` (`graph_edit.cpp:2652`) es una consulta pura sobre un `HashSet` de pares
`(from_type, to_type)` registrados con `add_valid_connection_type`/`remove_valid_connection_type`
(`:2642-2650`): no decide nada por si sola, solo responde si el par fue registrado explicitamente.

La decision real de "este puerto es un target valido mientras arrastro" ocurre en
`end_keyboard_connecting` (`:1143`, camino de conexion por teclado) y en el manejo de mouse
(`:1153-1170`, `:1360-1400` aprox.): un puerto es valido si `type == connecting_type` **o**
`p_node->is_ignoring_valid_connection_type()` **o** el par esta en `valid_connection_types`
(el set que respalda a `is_valid_connection_type`). El resultado se guarda en
`connecting_target_valid` (bool) y `connecting_target_node`/`connecting_target_port_index`.

**Senales emitidas** (grep sobre `emit_signal`, `graph_edit.cpp`):
- `connection_request(from_node, from_port, to_node, to_port)` — soltaste el cable sobre un
  puerto valido (`:1174`, `:1176`, `:1412-1416`).
- `connection_to_empty(from_node, from_port, release_position)` / `connection_from_empty(...)` —
  soltaste el cable en espacio vacio o sobre un puerto invalido, sin conectar (`:1180-1183`).
  No encontre una senal `connection_rejected` ni `invalid_connection` explicita: el "rechazo" se
  modela como "no hubo `connection_request`", nunca como un evento de error propio.
- `connection_drag_started(from_node, from_port, is_output)` (`:1071`, `:1090`, etc.) y
  `connection_drag_ended()` (`:2415`, via `force_connection_drag_end`, `:2404`).

**Visual de conexion invalida**: grep de `invalid` en todo `graph_edit.cpp` no encuentra ningun
color, tinte ni estilo dedicado a "conexion rechazada" — no existe un `connection_invalid_color`
en el theme cache (`grep -i invalid` solo matchea comentarios sobre cache invalidation). Lo que
existe es lo opuesto: un tinte que se aplica SOLO cuando el target es valido,
`theme_cache.connection_valid_target_tint_color` (bind en `:3157`), aplicado en
`_update_top_connection_layer` (`:1735`) sobre `from_color`/`to_color` cuando
`connecting_target_valid` es true (`:1764-1778`). Cuando el target no es valido, el cable en
arrastre simplemente conserva los colores normales del puerto de origen/destino — no hay
retroalimentacion de color de "esto esta mal", solo ausencia del tinte de "esto esta bien".

## 4. Minimapa

`GraphEditMinimap` (declarada junto a `GraphEdit` en el mismo archivo; metodos desde
`graph_edit.cpp:71`) mapea el grafo completo al rectangulo del minimapa con dos transformaciones
lineales por eje, sin rotacion:

- `_convert_from_graph_position` (`:138`): `map.x = pos.x * render_size.width / graph_proportions.x`
  (idem `y`) — graph -> minimapa.
- `_convert_to_graph_position` (`:148`): la inversa, minimapa -> graph.

`graph_proportions` es el tamano del grafo (`_get_graph_size`, `:125`, diferencia entre
`max_scroll_offset` y `min_scroll_offset` de `GraphEdit`) ajustado para mantener el aspect ratio
del area de render del minimapa (`update_minimap`, `:79-104`): si el grafo es mas ancho que el
minimapa, se lo "engorda" en altura (padding vertical) y viceversa, para que el contenido no se
deforme (`:90-100`).

- **Rectangulo de camara**: `get_camera_rect` (`:106`) proyecta `camera_position` (scroll actual
  menos offset del grafo) y `camera_size` (tamano del viewport de `GraphEdit`) al espacio del
  minimapa con la misma transformacion.
- **Cables en el minimapa**: `_draw_minimap_connection_line` (`:1595`) reusa
  `get_connection_line` con las posiciones **sin escalar por zoom** (a diferencia del dibujo
  principal) y luego convierte cada punto de la polilinea con
  `minimap->_convert_from_graph_position(point) + minimap->minimap_offset` (`:1600`). El color se
  interpola punto a punto por posicion normalizada a lo largo de la curva
  (`from.distance_to(point) * length_inv`, `:1608-1611`), no solo en los extremos.
- **Navegacion desde el minimapa**: `GraphEditMinimap::gui_input` (`:158`). Un click (no sobre el
  hitbox del resizer) llama `_adjust_graph_scroll` (`:176`, `:200`), que convierte la posicion de
  click a espacio de grafo y hace `ge->set_scroll_offset(p_offset + graph_offset - camera_size / 2)`
  (`:202`) — centra la vista principal en el punto clickeado. Arrastrar con el boton izquierdo
  sostenido repite el mismo ajuste en cada `InputEventMouseMotion` (`:184-197`), como un
  paneo continuo. El minimapa tambien es redimensionable arrastrando desde su esquina
  (`is_resizing`, `:172-174`, `:185-191`), acotado a no superar el tamano de `GraphEdit` menos el
  padding.

## 5. Seleccion por area (box selection)

Se inicia con click izquierdo en espacio vacio (`graph_edit.cpp:2246-2247`,
`box_selecting = true; box_selecting_from = mb->get_position();`). El modo aditivo/sustractivo se
decide una sola vez al iniciar, segun el modificador presionado:
- CTRL/CMD (`is_command_or_control_pressed`, `:2249`): `box_selection_mode_additive = true`,
  conserva la seleccion previa en `prev_selected` para poder restaurarla si el rectangulo se
  achica.
- SHIFT (`:2260`): igual pero `box_selection_mode_additive = false` (resta lo que entra en el
  rectangulo de la seleccion previa).
- Sin modificador (`:2271-2281`): limpia toda seleccion antes de empezar.

Durante el arrastre (`:2040-2069`), el rectangulo se recalcula en cada `InputEventMouseMotion`:
```cpp
box_selecting_to = mm->get_position();
box_selecting_rect = Rect2(box_selecting_from.min(box_selecting_to),
                            (box_selecting_from - box_selecting_to).abs());
```
(`:2041-2043`) — esquina superior-izquierda como el minimo componente a componente de los dos
puntos, tamano como el valor absoluto de la diferencia. Que nodos entran (`:2055-2064`):
- Si el elemento es un `GraphFrame`, tiene que estar **totalmente contenido**
  (`box_selecting_rect.encloses(r)`).
- Si es un `GraphNode` (u otro `GraphElement` no-frame), alcanza con **intersectar**
  (`box_selecting_rect.intersects(r)`).
- Los que no matchean vuelven al estado que tenian en `prev_selected` (para que soltar el mouse
  sobre un rectangulo mas chico no pierda la seleccion previa fuera del rectangulo en modo
  aditivo/sustractivo).

Se dibuja con `top_layer->draw_rect(box_selecting_rect, theme_cache.selection_fill)` +
`draw_rect(..., theme_cache.selection_stroke, false)` en `_top_layer_draw` (`:1726-1733`). Click
derecho durante el box-select lo cancela y restaura `prev_selected` (`:2073-2085`).

## 6. GraphNode: anatomia de la caja

- **Cabecera/titulo**: el constructor (`graph_node.cpp:1364-1374`) arma un `HBoxContainer`
  interno (`titlebar_hbox`, hijo `INTERNAL_MODE_FRONT` — no cuenta como hijo "de usuario") con un
  `Label` (`title_label`, variante de tema `"GraphNodeTitleLabel"`) dentro. Otros controles
  (botones de cerrar, etc.) se agregan al mismo hbox por fuera de este archivo. En
  `NOTIFICATION_DRAW` (`:645-727`) se calcula `titlebar_rect` a partir del tamano de
  `titlebar_hbox` mas el minimo del stylebox de titlebar, y se pinta con
  `theme_cache.titlebar_selected` o `theme_cache.titlebar` segun `selected` (`:650-651`, `:668`).
- **Slots de entrada/salida**: cada fila del cuerpo es un `Slot` (`graph_node.h:42-55`, struct con
  `enable_left/right`, `type_left/right`, `color_left/right`, iconos custom opcionales,
  `draw_stylebox`), guardado en `slot_table` (`Map<int, Slot>` indexado por indice de hijo).
  `set_slot`/`clear_slot`/`clear_all_slots` (`graph_node.cpp:731-770`) lo mutan. Cada slot
  corresponde 1:1 a un hijo `Control` del nodo (fila del layout).
- **Pintura de puertos**: `draw_port` (`:316-331`) dibuja `slot.custom_port_icon_left/right` si
  existe, o el icono generico `theme_cache.port` si no, centrado en `p_pos` (offset
  `-icon_size * 0.5`, `:329`). Se invoca desde `NOTIFICATION_DRAW` para cada slot con
  `enable_left`/`enable_right` en `Point2i(port_h_offset, slot_y)` /
  `Point2i(get_size().x - port_h_offset, slot_y)` (`:685-692`) — puertos de entrada pegados al
  borde izquierdo, salida al derecho, a una distancia fija `theme_cache.port_h_offset` del borde.
  El slot seleccionado por teclado (`selected_slot`) dibuja ademas un halo
  (`theme_cache.slot_selected`, `:694-707`).
  Las posiciones publicas (`get_input_port_position`/`get_output_port_position`,
  `:1118`, `:1155`) leen de cachés (`left_port_cache`/`right_port_cache`) recalculadas de forma
  lazy por `_port_pos_update` cuando `port_pos_dirty` es true (`:1119-1121`).
- **Redimensionado**: el flag `resizable` y el icono `theme_cache.resizer` se usan para *dibujar*
  el asa de resize en la esquina inferior-derecha (`:724-726`) y para decidir el cursor
  (`get_cursor_shape`, `:1231-1232`, cambia a `CURSOR_FDIAGSIZE` cuando el mouse esta sobre el
  hitbox del resizer o `resizing` es true). Los campos `resizable`/`resizing` en si y el manejo del
  arrastre de resize **no estan definidos en `graph_node.cpp`/`graph_node.h`** — se heredan de la
  clase base `GraphElement` (`graph_element.cpp`, fuera del alcance de esta auditoria).

## Portable a Lotte

Contexto del visor actual de Lotte: pan/zoom sobre `egui::Painter`, cables `CubicBezierShape`
entre puerto inferior del padre y superior del hijo, arrastre de nodos, seleccion — sin box-select,
sin minimapa, sin validacion de tipos, DAG en Rust con `DagError::CycleDetected`.

**Traducir casi tal cual (logica geometrica pura, portable sin dependencias de Godot), con
atribucion MIT a Godot Engine, `scene/gui/graph_edit.cpp`:**
1. **Formula de puntos de control del cable** (`:1532-1542`): `cp_offset = dx * curvatura`,
   invertido si `dx < 0`, aplicado como tangente horizontal en ambos extremos. Es la pieza de
   mayor valor/costo: hoy Lotte conecta puerto-inferior a puerto-superior (verticalmente), Godot
   conecta izquierda-a-derecha (horizontalmente) — el mismo principio (offset proporcional a la
   distancia en el eje principal del layout) aplica rotado 90 grados si Lotte migra a un layout
   horizontal, o tal cual si el layout de Lotte ya es horizontal.
2. **Rectangulo de hotzone asimetrico** (`is_in_port_hotzone`, `:1491-1500`) en vez de un radio
   circular: mas barato de calcular, y separa el margen "hacia afuera" del "hacia adentro" del
   nodo, que es justo el matiz que evita capturar el puerto equivocado en nodos apretados.
3. **Formula de box-select** (`:2041-2043`, `min`/`abs` para esquina y tamano) y la regla
   "frame necesita `encloses`, nodo necesita `intersects`" (`:2055-2058`) — trivial y de alto
   valor si Lotte agrega agrupacion tipo `GraphFrame` a futuro.
4. **Transformaciones de minimapa** (`_convert_from_graph_position`/`_convert_to_graph_position`,
   `:138-156`) y el ajuste de aspect ratio con padding (`update_minimap`, `:79-104`) — dos
   funciones lineales por eje, sin estado de Godot involucrado.

**Solo idea (la implementacion concreta depende de Godot: `Line2D`, `ShaderMaterial`, `Control`,
señales de `Node`), reimplementar con las herramientas de Lotte/egui:**
- El shader de rim/antialiasing (`:224-243`) — en egui esto se resuelve con el propio
  antialiasing de `CubicBezierShape` o un `Stroke` mas ancho, no con un shader dedicado.
- El modelo de señales (`connection_request`, `connection_drag_started/ended`,
  `connection_to_empty/from_empty`) es la idea correcta (separar "arranco el drag" / "solte sobre
  destino valido" / "solte en el vacio") pero la forma concreta en Lotte sera callbacks o eventos
  de `egui`, no `emit_signal`.
- Tinte de "puerto valido mientras arrastro" (`connection_valid_target_tint_color`) es una idea
  de UX que vale la pena copiar (portar el *comportamiento*, no el codigo): resaltar el color del
  cable en arrastre solo cuando hay un target valido bajo el cursor, sin necesidad de un color de
  "invalido" — el propio `is_valid_connection_type`-equivalente de Lotte decidiria el booleano.

**Que le falta al visor de Lotte respecto de Godot, en orden de valor:**
1. **Validacion de tipos de puerto al conectar** (equivalente a `valid_connection_types` +
   `is_valid_connection_type`, `:2652`): hoy Lotte no rechaza conexiones por tipo. Es la pieza que
   mas se conecta con lo que Lotte ya tiene (el DAG con `DagError::CycleDetected`) — la validacion
   de ciclos ya existe a nivel de datos, falta la de tipos a nivel de UI antes de intentar la
   conexion.
2. **Hotzone de puerto explicita para el arrastre de cables** (`is_in_input/output_hotzone`,
   `:1463-1489`): si Lotte hoy detecta el puerto solo por proximidad al punto exacto, un
   rectangulo de captura con margen asimetrico mejora la usabilidad del enganche sin agregar
   complejidad significativa.
3. **Retroalimentacion visual de "conexion vigente" durante el arrastre** (`:1764-1778`): tinte
   cuando el mouse esta sobre un target valido — barato de implementar sobre el `CubicBezierShape`
   existente, solo necesita el booleano de validacion del punto 1.
4. **Box selection** (seccion 5): Lotte tiene arrastre de nodos y seleccion, pero (segun el
   contexto dado) no seleccion por rectangulo. Es una feature de UX estandar y la formula
   (`:2041-2043`) es trivial de portar.
5. **Minimapa**: mayor esfuerzo (requiere una segunda superficie de render/`Painter` con su propia
   transformacion), pero la matematica (seccion 4) ya esta resuelta arriba; util recien cuando los
   grafos de Lotte crezcan lo suficiente para no caber en pantalla.
6. **Distincion "click sobre conexion existente" vs "arrastre de puerto"** con umbral de distancia
   propio (`get_closest_connection_at_point`, radio default `4.0`, `:1551`, `:2982`) — hoy no
   esta claro si Lotte permite seleccionar/eliminar un cable individual haciendo click sobre el;
   Godot lo resuelve con distancia punto-segmento sobre la polilinea tesellada.

## Lo que NO existe o no encontre

- **`connection_rejected` / `invalid_connection` como señal**: no existe. El rechazo se modela por
  ausencia de `connection_request`, nunca como evento propio (ver seccion 3).
- **Color/estilo dedicado a "conexion invalida"**: no existe (`grep -i invalid` en
  `graph_edit.cpp` no matchea ningun theme item de color). Solo hay tinte de "valido"
  (`connection_valid_target_tint_color`).
- **Radio de captura circular para el enganche de puertos**: no existe; es un rectangulo
  (`is_in_port_hotzone`, seccion 2). Un "radio" en el sentido pedido por el brief solo aparece
  para clic sobre conexiones existentes (`get_closest_connection_at_point`, default `4.0`), no
  para el enganche de puertos durante el arrastre.
- **`_get_connection_target` o `snapping` de puertos**: no existen esos nombres. Lo mas cercano es
  `is_in_input_hotzone`/`is_in_output_hotzone` (arrastre) y `snapping_enabled`/`snapping_distance`
  (grilla de posicion de nodos, sin relacion con puertos).
- **Manejo de resize de `GraphNode`**: los campos `resizable`/`resizing` se leen en
  `graph_node.cpp` pero no se definen ni mutan ahi (heredados de `GraphElement`, fuera del
  alcance de sparse-checkout declarado para esta auditoria — solo `scene/gui/graph_edit*` y
  `graph_node*` estaban disponibles).

## Como se midio

Todos los comandos corridos con el `Grep`/`Read` del agente, acotados a
`references/godot/scene/gui/` (nunca `find` sobre la raiz del disco):

```
grep -n "get_connection_line|_update_connections" graph_edit.cpp
grep -n "snapping|connection_drag|_get_closest_connection|_get_connection_target|is_in_input_hotzone|is_in_output_hotzone|connecting_target" graph_edit.cpp
grep -n "is_valid_connection_type|invalid_connection_type|connection_rejected|VALID_CONNECTION" graph_edit.cpp
grep -in "invalid" graph_edit.cpp
grep -n "class GraphEditMinimap|_convert_from_graph_position|_convert_to_graph_position|minimap_offset|minimap->|is_pressing|is_resizing" graph_edit.cpp
grep -n "box_selecting|box_selection_mode|_box_select|is_selected|set_selected" graph_edit.cpp
grep -n "_get_shader_line_width|lines_thickness =|lines_curvature =|MAX_CONNECTION_LINE_CURVE_TESSELATION_STAGES" graph_edit.cpp
grep -n "void GraphNode::_draw_port|_draw_title|title_label|get_input_port_position|get_output_port_position|is_resizable|_notification|slot_table|set_slot\(" graph_node.cpp
grep -n "resizable|resiz" graph_node.cpp
grep -n "is_resizable|set_resizable" graph_node.h
grep -n "struct Slot|bool enable_left|int type_left|Color color_left|bool draw_stylebox" graph_node.h
```

Mas lecturas puntuales de rangos de lineas (`Read` con `offset`/`limit`) sobre `graph_edit.cpp` en
1440-1800, 60-280, 1140-1220, 2030-2100, 2240-2290, 2632-2955, y sobre `graph_node.cpp` en
316-345, 628-730, 1118-1172, 1360-1378. Todos los numeros de linea citados en este informe salen
de estas corridas, no de memoria.
