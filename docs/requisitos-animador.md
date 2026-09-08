# Requisitos del flujo del animador — y lo que exigen del modelo de datos

Seis requisitos que los animadores de producción dan por sentados, recogidos por el autor del
proyecto a partir del trabajo diario con su equipo. Ninguno es un botón: **son propiedades del
modelo de datos**, y si el modelo no las contempla desde el principio, después no se agregan.

| | |
|---|---|
| Origen | Práctica de producción del estudio del autor (animación 2D cutout, varios personajes por plano, rigs de ~300 pegs) |
| Respaldo técnico | Proyectos de licencia permisiva y estándares abiertos, citados en cada sección |
| Estado | Requisitos aceptados; su implementación está en la [hoja de ruta](plan-siguiente-etapa.md) (decisión D-Inst y lotes E y C) |

---

## El modelo que los requisitos necesitan: definición, instancia, animación

Los requisitos 1 a 3 se resuelven con **una** separación:

```
RigDefinition   ← el rig "como fue construido" + sus arreglos (editar rig)
      │  se instancia (clonar) o se copia (duplicar)
      ▼
RigInstance     ← un personaje en una escena: referencia a UNA definición
      │  + valores de reposo heredados de la definición
      ▼
Animation       ← canales (ruta de nodo, atributo) → FCurve / ExposureTrack
                  pertenecen a la instancia, NUNCA a la definición
```

Un rig ingenuo mezcla las tres cosas: el transform de cada peg es a la vez el reposo, la edición y
el valor animado, y un `set_transform` sobreescribe sin saber si está editando el rig o animándolo.
Eso es lo que hay que separar. Godot lo modela igual: `Bone2D.rest` es el reposo, la pose final es
`acumulada × inversa(rest)`, y una animación `RESET` sirve de línea base
([`referencias/godot-animacion-esqueleto.md`](referencias/godot-animacion-esqueleto.md)). glTF
separa `nodes[]` (reposo) de `animations[]` (canales). Bevy, que **no** separa reposo de animación,
muestra por contraste el problema: una pista que escribe el valor absoluto pisa cualquier arreglo
([`referencias/bevy-animation.md`](referencias/bevy-animation.md)).

## 1. Copiar la animación de una escena a otra, con el mismo rig

**Lo que hace el animador.** Selecciona en la línea de tiempo las claves desde el peg más alto de
la jerarquía y las pega en otra escena que tiene el mismo rig. Toda la actuación viaja.

**Lo que exige del modelo.**
- Un canal animable es **`(ruta de nodo, atributo)`**, no `(índice, campo)`. La ruta se construye
  con los **nombres** de la jerarquía (`Root/Torso/Brazo_I`), así que dos instancias del mismo rig
  producen las mismas rutas y la animación es transferible. Cuatro sistemas independientes hacen
  exactamente esto: Godot (`NodePath:propiedad`), Bevy (`AnimationTargetId::from_names`), glTF
  (`target.node` + `path`) y DragonBones (timelines por nombre de hueso y de slot).
- La `Animation` es un objeto **separable** de la instancia: se serializa sola, se copia sola, se
  pega sobre otra instancia con la misma definición. Pegar = emparejar por ruta; **lo que no
  empareja se reporta, no se inventa** (Godot: nodo inexistente → aviso y se salta la pista, nunca
  fatal).
- Pegar sobre un rig *distinto* es un problema aparte (mapa de rutas), fuera de alcance.

**Lo que el diseño ya tiene a favor:** el crate de curvas es independiente del rig, así que una
`Animation` puede existir sin cargar la jerarquía — justo lo que hace falta para copiarla entre
escenas.

## 2. Editar el rig ≠ animar; y "reset" vuelve al reposo sin perder los arreglos

**Lo que hace el animador.** Mientras anima, en cualquier momento puede *resetear* una pieza o todo
el personaje a como estaba al construirlo. Pero si arregló algo del rig (un pivote mal puesto, una
pieza corrida), ese arreglo **sobrevive** al reset: era edición del rig, no animación.

**Lo que exige del modelo.**
- Dos **modos de edición** con destinos distintos: *Editar rig* escribe en la `RigDefinition`
  (y por tanto en todas las instancias que la comparten); *Animar* escribe canales en la
  `Animation` de la instancia.
- El valor efectivo de un atributo en el frame `t` es **`reposo(definición) ⊕ animación(t)`**: la
  animación es un *offset* (o un reemplazo, según el canal) sobre el reposo, nunca el valor absoluto
  a secas. Así **reset** = quitar el canal animado, y el reposo —con sus arreglos— queda.
- El grafo de pegs deja de guardar "el" transform: guarda el de reposo; la evaluación toma la
  animación como entrada. Es un cambio del núcleo que conviene hacer **antes** de que exista más
  código que escriba transforms directamente.

## 3. Duplicar ≠ clonar

**Lo que hace el animador.** Quiere un gemelo malvado. Si es una **instancia** (clon), cualquier
cambio al personaje base le llega al gemelo. Si es un **duplicado**, le puede agregar una cicatriz
sin tocar al original. La trampa conocida en producción: la operación "obvia" (copiar) crea un
vínculo permanente que nadie ve, y meses después un arreglo en un personaje rompe a otro.

**Lo que exige del modelo.**
- **Clonar** = nueva `RigInstance` que **referencia la misma `RigDefinition`**. Cambios en la
  definición → en todos los clones. Barato: no copia geometría. Es la instanciación de escenas de
  Godot (`PackedScene` instanciada N veces), el `<use>` de SVG y las *precomps* de Lottie.
- **Duplicar** = nueva `RigDefinition` con **ids nuevos** para todo (nodos, trazos, canales) y una
  instancia propia. Cambios en una no tocan la otra.
- Las dos operaciones tienen que ser **visibles** en el visor de nodos (un clon se marca como tal),
  y el formato guarda la definición **una vez** y las instancias como referencias: es lo que hace
  que un plano con seis extras idénticos pese como uno. En disco es el esquema `defs` / `id` /
  `use` de SVG: una definición con `id`, cada uso escribe `use="id"`, y **cargar un `id` repetido es
  error**, no colisión silenciosa.
- Regenerar ids al duplicar es la razón número cuatro para que los ids sean **estables y
  generados**, no posicionales — se suma a los canales, a los trazos y a lo que Graphite ya hace
  con sus `PointId`/`SegmentId` numéricos ([`referencias/graphite-vector.md`](referencias/graphite-vector.md)).

## 4. Interoperabilidad: SVG y formatos abiertos

**La pregunta.** ¿Usar un estándar como SVG para el dibujo vectorial, y con qué implementación?

**Lo medido.**

| Opción | Resultado con `vello 0.10` + `kurbo 0.13` |
|---|---|
| `vello_svg` (Linebender) | **Va una versión atrás**: pide `vello 0.9`; todas las probadas (0.7–0.10) dejan **dos vello** en el árbol. No sirve hoy. |
| **`usvg 0.46`** (resvg, Apache-2.0 OR MIT) | **Una** vello, **una** kurbo, **una** wgpu. `usvg` ya depende de nuestro mismo kurbo. La conversión `usvg::Path` → `kurbo::BezPath` es nuestra, corta. |
| Graphite | Importa con `usvg` (está en su lock, con `roxmltree` y `svgtypes`) y exporta con un serializador propio (`renderer.rs:3382`). Es el mismo camino. |

**Lo que SVG no puede.** SVG no tiene **grosor variable a lo largo del trazo**: `stroke-width` es
un escalar (SVG 1.1 y 2). Tampoco tiene rig, tiempo ni capas de arte con semántica: `<g>` es un
grupo sin significado. Lo que sí tiene es **extensibilidad por namespaces**: la especificación
permite elementos y atributos de otros espacios de nombres, que un visor que no los conoce ignora.
Varios editores vectoriales guardan así sus datos propios dentro de SVG válido.

**Lo que exige del modelo.**
- **SVG es el formato de intercambio de dibujos, no el de documento.** Importar con `usvg`
  (rellenos, trazos de ancho fijo, degradados, transformaciones aplanadas); exportar generando el
  path desde `BezPath` — y para el grosor variable, **exportar el contorno resuelto** como relleno
  más la línea central y el perfil en un namespace propio (`lotte:centerline`, `lotte:thickness`),
  que cualquier visor ignora y Lotte relee. Es honesto: lo que otro programa ve es correcto, lo que
  Lotte ve es completo.
- Las **capas de arte** (§1 de [`patterns.md`](patterns.md)) se exportan como `<g id="line">`,
  `<g id="colour">` etc.: pierden semántica afuera, la recuperan al volver.
- El **documento** de Lotte es propio (decisiones D-Fmt y D-Tab): no hay estándar abierto que
  cubra un rig cutout con tiempo. Lo más cerca es el **JSON de DragonBones (MIT)**, que la
  hoja de ruta contempla importar; Lottie es para *reproducir* animación acabada, no para
  editarla, y también carece de grosor variable.
- **Estándares más nuevos:** SVG 2 no cambia nada de lo anterior; *SVG Native* es un subconjunto.
  No hay un estándar de dibujo vectorial con envolvente de grosor. La §6 de `patterns.md` seguirá
  siendo propia, y su puente al mundo es el contorno resuelto.

## 5. Lápiz, línea, polilínea y pincel — dos representaciones, no cuatro herramientas

**Lo que hace el animador.** El *lápiz* traza una línea sin grosor propio: el grosor es un
parámetro que se administra después, a lo largo del trazo, con un editor propio. *Línea* y
*polilínea* son el mismo tipo de trazo con la línea central recta o poligonal. El *pincel*, en
cambio, deja un vector **con dos contornos** cuyo ancho sigue la presión, y si se pinta encima con
el mismo color, las pinceladas **se fusionan en una forma mayor** en vez de apilarse. En rigging se
usa poco; para ciertos dibujos es indispensable. La distinción es tan vieja como la tinta y el
pincel: una es una **curva con perfil de anchura**; la otra, una **envolvente unida**.

**Lo que hace Graphite** (medido, [`referencias/graphite-vector.md`](referencias/graphite-vector.md)):
`freehand_tool` produce una línea central con `stroke.weight` constante — un lápiz sin perfil;
`brush_tool` es **raster**. No tiene el pincel vectorial. Pero sí tiene lo que hace posible la
fusión: **operaciones booleanas sobre Bézier sin aplanar**, con el crate **`linesweeper`**
(`nodes/path-bool/src/lib.rs`).

**Lo medido para Lotte:**

| Pieza | Resultado |
|---|---|
| Línea central → contorno | **`kurbo::stroke()`** (`stroke.rs:260`), ya en el stack; para el perfil variable, la envolvente de `patterns.md` §6 sobre **`kurbo::offset::offset_cubic(c, d, tolerance, &mut BezPath)`** (`offset.rs:108`), con la tolerancia como decisión de calidad explícita |
| Unión de contornos (fusión de pinceladas) | **`linesweeper 0.4.0`** (MIT OR Apache-2.0; `binary_op`, `lib.rs:103`) — depende de **nuestro mismo `kurbo 0.13.1`**, resuelve con `vello 0.10` dejando una de cada, y es **la misma versión que usa Graphite**. Booleanos directamente sobre Bézier. |
| Alternativa | `i_overlay 8.1.1` (MIT OR Apache-2.0): también resuelve limpio, pero trabaja sobre polígonos: aplanar → unir → re-ajustar con `kurbo::fit_to_bezpath`. Segunda opción si `linesweeper` falla en casos raros. |
| Fit de la mano alzada | `kurbo::fit_to_bezpath` (`fit.rs:165`): de puntos muestreados a cúbicas |

**Lo que exige del modelo.** El modelo de dibujo necesita **dos tipos de primitiva** en la capa de
arte, no una:

- **`Stroke`** = línea central (`BezPath`) + perfil de grosor (dos funciones de la longitud de arco,
  una por lado, §6) + estilo (remates, color por id). Cubre lápiz, línea y polilínea: la diferencia
  entre las tres es *cómo se captura la línea central*, no cómo se guarda. Editable después con el
  **editor de perfil de grosor**: nudos sobre la longitud de arco, izquierda y derecha
  independientes, interpolación Catmull-Rom centrípeta por defecto. Poner los nudos por longitud de
  arco (y no por segmento + t) hace que sobrevivan a insertar o borrar nodos de la curva; dos lados
  independientes permiten caligrafía asimétrica.
- **`Contour`** = forma cerrada rellena (`BezPath`) + color por id. Lo que deja el pincel: la
  presión se resuelve **al soltar** en un contorno (línea central + presión → envolvente →
  contorno), y si toca otro contorno del mismo color en la misma capa, se **une** con
  `linesweeper`. Opcional: guardar la línea central y el perfil como metadato del contorno para
  poder "des-pintar" — una mejora barata si los ids son estables.
- Conversión `Stroke → Contour` gratis (`kurbo::stroke` o la envolvente); la inversa no existe,
  porque un contorno no recuerda su línea central.
- Rasterizar es lo mismo para ambas: `fill()` de vello. El `stroke()` de vello con ancho fijo queda
  solo como atajo para `Stroke` de perfil constante.

Dificultad honesta: la envolvente de §6 (el offset de una cúbica no es una cúbica; se compone y se
aplana a Bézier al final con tolerancia explícita, y el pliegue cóncavo se detecta por intersección
real de los dos segmentos de contorno vecinos y se recorta) y la unión booleana son los dos puntos
con matemática de verdad. Los dos tienen crate alineado y verificado; ninguno hay que inventarlo.

## 6. ¿Existe un estándar abierto para las tablas de transformación en el tiempo?

**De los proyectos permisivos revisados, ninguno es un estándar**: OpenToonz guarda sus curvas en
su propio XML, Rerun en su formato de registro (Arrow). Lo que sí existe afuera:

| Estándar | Qué modela | Cuánto se parece a `(ruta, atributo) → FCurve` |
|---|---|---|
| **glTF 2.0 — `animations`** (Khronos, abierto) | Canales `{ target: { node, path }, sampler }`; samplers con keyframes `STEP`, `LINEAR`, **`CUBICSPLINE` con tangentes de entrada y salida explícitas**; los números en buffers binarios aparte | **El más cercano.** Es exactamente nodo + atributo + keyframes con manijas. Le falta: exposición de dibujos, canales arbitrarios (solo TRS) y pivotes explícitos |
| **DragonBones JSON** (MIT) | Rig cutout completo: huesos, slots, *displays*, timelines por hueso y por slot (sustitución de dibujo), FFD | El único que cubre **exposición de dibujos** y rig cutout; ya está en la hoja de ruta como importador |
| **OpenTimelineIO** (ASWF, Apache-2.0) | Editorial: pistas, clips, cortes, tiempo de secuencia | La capa de *storyboard* (secuencia → paneles), no la de curvas de parámetro |
| **USD** (Pixar/AOUSD, Apache-2.0 modificada) | Atributos con *time samples* por prim; composición de capas | Bake por muestras, no curvas editables; pesado. Referencia para *capas y overrides* (definición/instancia), no para keyframes |
| **Lottie** (JSON, abierto) | Reproducción de animación acabada con easing Bézier por propiedad; *precomps* | Para *exportar* a web, After Effects y móviles; no para editar rigs |

**Lo que exige del modelo.** No adoptar ninguno como formato de documento. Sí alinear el **modelo
interno** con glTF donde coincide — canal = `(nodo, atributo)`, sampler = keyframes con
interpolación y tangentes explícitas, estructura en texto y números en buffers — porque eso hace
que exportar a glTF/Lottie sea una traducción y no un rediseño, y porque valida la decisión de
guardar las manijas en el archivo. Los canales que glTF no tiene (exposición, pivote, Z, canales de
deformador) se modelan igual, con el mismo esquema, y se exportan solo a lo que los soporte
(DragonBones para la exposición).

---

## Lo que esto le agrega a la hoja de ruta

Una decisión y dos lotes, detallados en [`plan-siguiente-etapa.md`](plan-siguiente-etapa.md):

- **D-Inst** — adoptar el modelo *definición / instancia / animación* en el núcleo, con valor
  efectivo `reposo ⊕ animación(t)`. Es la decisión que habilita los requisitos 1–3, y conviene
  tomarla **antes** del modelo de dibujo, porque los ids estables se diseñan encima de ella.
- **Lote E — Flujo del animador**: modos editar/animar con reset al reposo; `Animation` separable
  con copiar/pegar por `(ruta, atributo)` y reporte de lo que no empareja; clonar vs duplicar con
  marca visible y definición guardada una vez; importar SVG con `usvg 0.46`; exportar SVG con
  contorno resuelto + namespace `lotte:`.
- **Lote C — Modelo de dibujo**: `Stroke` y `Contour`, pincel como envolvente, fusión con
  `linesweeper`, editor de perfil de grosor.

Y confirma la conclusión de las auditorías: **identidad estable y generada en todo** — nodos,
canales, trazos, definiciones. Sin eso no hay copiar, ni pegar, ni duplicar.
