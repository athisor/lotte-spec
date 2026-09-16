# Decisiones del modelo de datos

Acta de las decisiones de diseño de Lotte, con la razón de cada una. Lo que acá se contradiga con
otro documento de esta especificación, gana este. Cada decisión cita su respaldo: un proyecto de
licencia permisiva leído en código, un estándar abierto, matemática publicada, o la experiencia de
producción del estudio del autor. No hay decisiones tomadas por analogía con productos cerrados.

Las cifras de producción que aparecen ("un rig tiene ~300 pegs") son mediciones sobre rigs reales
del estudio: 293–305 pegs, 1 061–1 115 nodos, 3 438–3 634 canales animables y 12 639–13 124 claves
por rig; planos de 20–30 segundos (480–720 frames a 24 fps) con varios personajes.

---

## Tiempo, espacio y estructura

### D-T — El tiempo se mide en frames del proyecto

`f64` en las curvas (permite sub-frame para *motion blur* y *time remap*), `u32` en la exposición.
`fps` vive en `Scene.timing` y convierte a segundos **solo en la frontera** (audio, exportación).
Razón: es la unidad del animador — cambiar el fps de un plano no debe mover claves fuera de frame.
OpenToonz trabaja igual (celdas por frame, `TXshCell`).

### D1 — Signo de `z`: mayor `z` = más cerca de la cámara

El arte vive en `z ≈ 0`; la cámara está en `z = D > 0`; `S_z = D / (D − z)`. `z` negativo es
simplemente más lejos que el plano del arte. El lado peligroso es el otro: un objeto con `z ≥ D`
está en la cámara o detrás, y `S_z` explota. **Regla de validación**: `z < D`, con aviso y
*clamp*, nunca un `NaN` en pantalla. Razón: es la convención de OpenToonz (`tstageobject.cpp`:
`dz = focus + cameraZ − objectZ`, escala `(focus+cameraZ)/dz`) y la que los animadores cutout
ya conocen; y la experiencia de producción del estudio recomienda **Z en pocos lugares, con el arte
en Z ≈ 0**.

### D-F — Mundo Y-arriba; la cámara voltea

El motor razona en Y-arriba (como OpenToonz); el volteo a coordenadas de pantalla es una operación
de vista en la frontera ventana→mundo. Fijado con un test de orientación.

### D-Inst — Definición / instancia / animación

`RigDefinition` (pegs con transform de **reposo**, arreglos, dibujos, paleta, ids), `RigInstance`
(nombre único en la escena, referencia a una definición, peg de instancia), `Animation` (canales
`(ruta, atributo)` de la instancia). `valor(t) = reposo ⊕ animación(t)`; reset = quitar el canal;
clon = otra instancia de la misma definición; duplicado = definición nueva con ids nuevos. Es la
única decisión que cambia el núcleo (`set_transform` se parte en reposo y canales) y va antes del
modelo de dibujo. Respaldo: Godot (`Bone2D.rest`, `RESET`), glTF (`nodes` vs `animations`),
instanciación de escenas de Godot y `<use>` de SVG; y los requisitos 1–3 de
[`requisitos-animador.md`](requisitos-animador.md). Ver también
[`referencias/godot-animacion-esqueleto.md`](referencias/godot-animacion-esqueleto.md).

### D-Id — Dos identidades por nodo

`NodeId` posicional para el grafo en memoria (barato), e id derivado de la **ruta de nombres**
(separador de longitud por segmento, la técnica de `AnimationTargetId::from_names` de Bevy) para
canales, formato y copiar/pegar. Nombres duplicados en una misma jerarquía = **error de validación
al cargar**, nunca colisión silenciosa. Nodo inexistente al aplicar una animación → aviso y se salta
la pista, nunca fatal (Godot, `animation_mixer.cpp`). Razón de producción: la identidad por texto
libre "escrito exactamente igual" es la fuente de errores más frecuente que el estudio registró en
sus pipelines; los ids generados la eliminan. Ver [`referencias/bevy-animation.md`](referencias/bevy-animation.md).

### D2 — El crate de curvas no depende del núcleo

`lotte-timeline` evalúa curvas sin cargar un rig; es lo que permite copiar una `Animation` entre
escenas sin abrir la jerarquía. `PoseSpace` (controlador multivariado) vive en `lotte-rig`.

### D3 — El dibujo vive en `lotte-rig`, con ids numéricos

Una sola autoridad sobre trazos, rellenos y su apariencia, como en Graphite (`PointId`,
`SegmentId` numéricos generados; relleno y trazo como `Appearance` aparte). Ver
[`referencias/graphite-vector.md`](referencias/graphite-vector.md).

## El documento en disco

### D-Fmt — Un plano es un directorio, no un archivo

```
plano_010/
├── scene.json                      ← Scene: timing, cámara, instancias (nombre único, ref a definición,
│                                      peg de instancia, z), apilado, overlays. Chico: sin curvas, sin dibujos.
└── anim/
    ├── personaje_A.anim.json       ← UNA animación por instancia: el diccionario de canales…
    ├── personaje_A.anim.<bin>         …y sus tablas (D-Tab). Copiar animación entre planos = copiar el par.
    └── camara.anim.json / .<bin>

biblioteca/
├── rigs/gato/
│   ├── definition.json             ← pegs con reposo, slots, paleta, ids estables, versión
│   └── drawings/*.svg              ← un archivo por dibujo (D-Draw)
└── fondos/bosque/...
```

Razones. **Velocidad**: guardar solo lo sucio (animar a un personaje ensucia *su* archivo, no los
otros nueve ni el fondo). **Flujo**: copiar animación entre planos es copiar un archivo; dos
animadores, dos personajes, dos archivos. **Arquitectura**: para glTF y Godot **la animación es un
recurso que se comparte entre escenas**, no un residuo de la escena — glTF separa estructura
(JSON: `nodes[]`, `animations[].channels`) de números (buffers `.bin`); Godot guarda `Animation`
como recurso externo `.tres`, en texto para control de versiones y en binario `.scn` para
distribuir, **con el mismo esquema**; DragonBones tiene varias animaciones independientes dentro de
la definición. Una animación **por instancia** y no por canal: la instancia es la unidad de
copiar/pegar, de reset y de trabajo en paralelo; por canal fragmenta demasiado.

`scene.json` y `definition.json` son JSON con `serde`, bajo la regla de **contratos solo aditivos**
(campos nuevos opcionales con default; fixture de la versión anterior que tiene que seguir
abriendo).

### D-Tab — Las tablas de tiempo son tensores, no texto

Con ~3 500 canales y ~13 000 claves por rig, y varios rigs por plano, el texto no alcanza: una clave
Bézier en JSON compacto pesa ~95 bytes; los mismos seis números como `f32` pesan 24, como `f64` 48.
Decisión: **diccionario JSON + tablas binarias**, con el patrón *ragged por offsets* de glTF y
Apache Arrow — pocos tensores grandes, no miles chicos:

| Tensor | dtype / shape | Contenido |
|---|---|---|
| `keys` | `F64 [N, 6]` — **F64 firmado el 2026-09-16 con la medición de E8**; no por precisión sub-frame (el peor error de F32 medido es 3,3e-4 frames, despreciable) sino porque el round-trip exacto del contrato en disco lo exige: `keys` son seis columnas sin magnitud acotada | `time, value, hout_x, hout_y, hin_x, hin_y` de **todas** las claves de todos los canales, concatenadas |
| `key_interp` | `U8 [N]` | Hold / Linear / Bezier / TCB / Clamped |
| `channel_offsets` | `U32 [C+1]` | dónde empieza cada canal dentro de `keys` |
| `exposure_runs` | `U32 [R, 3]` | `(frame_inicio, largo, dibujo)`: un hold de 200 frames es **una** fila |
| `exposure_offsets` | `U32 [X+1]` | una corrida de filas por pista de exposición |

El diccionario (`*.anim.json`) lista los `C` canales en orden: `(ruta de nombres, atributo)`, id
estable (D-Id), extrapolación, y las `X` pistas de exposición con su nivel. Nombres afuera, números
adentro. La **codificación** de las tablas se elige midiendo tres candidatas con el mismo esquema
`serde` sobre un fixture de tamaño real: JSON (legible, `diff`), **CBOR con typed arrays**
(RFC 8746: un solo archivo, binario, isomorfo a JSON) y **safetensors** (Apache-2.0: cabecera JSON
`{nombre: {dtype, shape, offsets}}` + bytes crudos, lectura `mmap` sin parsear, sin ejecución de
código al cargar). safetensors queda al menos como exportación al ecosistema de tensores. Un
*baked* opcional `F32 [frames, canales]` se sube tal cual a un *storage buffer* de GPU; la
evaluación de curvas por frame es trabajo de CPU.

Lo que **no** se adopta: *runtimes* de tensores (PyTorch, Triton) — son Python con CUDA, no
dependencias de un motor Rust, y el volumen de un plano no los necesita. Si algún día hay cómputo
tensorial real, los candidatos Rust son `burn` (backend `wgpu`) y `candle`, a verificar.

### D-Draw — Un archivo por dibujo: SVG + namespace `lotte:`

Los dibujos **no viven en la escena**: un archivo por dibujo en la biblioteca, **SVG estándar más
un namespace `lotte:`** para lo que SVG no expresa — el `Stroke` como línea central + perfil de
grosor por longitud de arco, las capas de arte (Underlay / Colour / Line / Overlay), los ids
estables y las referencias a paleta. El `<path>` estándar lleva el **contorno ya resuelto**, así
cualquier visor lo pinta bien; un lector Lotte usa los atributos `lotte:` y reconstruye el trazo
editable. Es la extensibilidad que la propia especificación SVG prevé: el archivo es abierto sin ser
pobre. Cumple a la vez el requisito de interoperabilidad y el de que la escena sea chica.

## Modelo de dibujo y modelo temporal (especificados)

**Dibujo**: dos primitivas — `Stroke` (línea central + perfil de grosor a dos lados por longitud de
arco, editable) y `Contour` (forma rellena); pincel de presión = envolvente → `Contour` → unión con
`linesweeper`; cuatro capas de arte (la separación tradicional entre línea limpia y color, más una
capa debajo y una encima); relleno por referencia a los trazos ("si muevo la línea, el color la
sigue"); paleta por indirección (estilos por índice, como `TPalette` de OpenToonz). Detalle en
[`requisitos-animador.md`](requisitos-animador.md) §5 y [`patterns.md`](patterns.md) §6.

**Tiempo**: canal `(ruta, atributo)` → `FCurve` con manijas explícitas en el archivo (como
`CUBICSPLINE` de glTF); monotonía en el tiempo por construcción (clamp del signo en x de las
manijas, como el editor de animación de Godot); interpolaciones Hold / Linear / Bézier / **TCB**
(Kochanek–Bartels, 1984) / **Clamped**; solver de Bézier 1D por Newton-Raphson con caída a
bisección (el método de `UnitBezier` de WebKit, BSD); exposición separada de la geometría
(`TXshCell` de OpenToonz; timelines de slot separadas de las de hueso en DragonBones); los canales
discretos **se seleccionan, no se promedian**; mezcla Clip / Blend / Add con máscaras por subárbol y
el reposo como semilla (Bevy, Godot). Fuera de rango: hold del extremo, con ciclo opcional.

## La escena

`Scene { timing, camera, background 0–1, props[], characters[], overlays }`, apilado
background → props → characters → overlays; cada instancia con **nombre único en la escena**,
referencia a una definición por `id` + versión, y colocación (offset, escala, `z`) en su **peg de
instancia**, nunca dentro de la definición. `Library` por encima del documento: definiciones con id
+ versión; la escena las referencia y se re-resuelve al abrir, **avisando** lo que no empareja.

Siete reglas aprendidas en la producción del estudio (138 planos compuestos automáticamente): la
escena es un **documento declarativo que se regenera** desde datos; la instancia tiene **nombre
único** y es la raíz de sus rutas; **la definición no se toca al componer** — la colocación va en el
peg de instancia, por fuera del rig; **los ajustes manuales son datos con nombre** (`overrides`),
no residuos que la regeneración pisa; **los fondos se importan por estructura, nunca por nombre de
capa**; **Z en pocos lugares**, con el arte en Z ≈ 0; y **una sola puerta** de normalización de
assets, que avisa y no bloquea.

## Deshacer y guardar

### D-Undo — Ediciones con inverso, pila por memoria, fusión de gestos

Toda mutación del documento es una `Edicion` con inverso, producida por la herramienta (el gesto),
no por el `set_` del modelo — como los operadores de Godot (`create_action`/`commit_action`) y de
Graphite (`StartTransaction`/`CommitTransaction`). Pila única por **plano abierto** (más una para
la biblioteca: editar un rig afecta a todos los planos que lo instancian y no debe deshacerse desde
uno de ellos), que admite ediciones **diferenciales** (poner clave, mover peg — decenas de bytes) e
**instantáneas** con estado compartido (`Arc`) solo para lo estructural (importar rig, borrar
instancia). Fusión de gestos por **transacción explícita** (arrastre) **y** ventana temporal
(800 ms, mismo nombre — Godot `undo_redo.cpp`) para repeticiones por teclado; al fusionar se
conserva el `antes` del primero y el `después` del último (`MERGE_ENDS`). Tope **por memoria**
(OpenToonz `TUndoManager`: descarta lo más viejo cuando la suma de tamaños supera el límite), no
por cantidad; cada `Edicion` declara `bytes()`. Rehacer se trunca al editar. El historial no
sobrevive al cierre — el diario sí. Ver [`referencias/godot-undo-redo.md`](referencias/godot-undo-redo.md).

Por qué no instantánea completa por paso (lo que hace Graphite, con tope 100): con ~3 MB por rig y
diez rigs por plano serían cientos de MB por paso. Para animar, pasos diferenciales.

### D-Save — El autoguardado es un diario; el guardado corre en otro hilo

El autoguardado **anexa** las ediciones nuevas a un diario (`plano/.journal/`, JSON Lines, una
edición por línea, `fsync`, fuera del hilo de interfaz): cuesta lo que costó la edición, no lo que
pesa el plano — es el *write-ahead log* de SQLite aplicado a un directorio de archivos. Guardar
(Ctrl+S) toma una instantánea con `Arc` (comparte estructura, casi gratis) y escribe **en otro hilo
solo los archivos sucios** (por instancia, D-Fmt), a temporal + *rename* atómico, rota los
anteriores a `.1`/`.2` (por defecto 2) y trunca el diario. Abrir = leer archivos (tablas por
`mmap`, dibujos perezosos) y, si hay diario no vacío, ofrecer recuperarlo. **El hilo de dibujo
nunca toca disco.** Una línea truncada del diario (corte de luz) se descarta con aviso, sola.

Vocabulario que usa esta especificación: un paso es *con estado* (se carga en cualquier dirección)
o *diferencial* (se aplica o desaplica; para deshacer se desaplica el paso n+1, no se carga el n);
la pila es *relativa* (saltar en el historial reproduce los intermedios) con las instantáneas como
puntos de control absolutos.

## Arquitectura de acceso

### D-API — La interfaz no tiene privilegios; todo se puede hacer sin ventana

1. **Toda operación** que un menú, un gizmo o un atajo pueda hacer es una `Edicion` o una operación
   del documento (`abrir`, `guardar`, `exportar`, `importar`, `combinar`) invocable **sin ventana**.
   La aplicación es un cliente de esa API, no su dueña. Si algo solo se puede hacer con el ratón,
   es un bug de arquitectura. Es el patrón del editor de Godot, que usa la misma API que los scripts.
2. **El núcleo es headless de verdad, GPU incluida**: `vello::Renderer::render_to_texture(device,
   queue, scene, view, params)` solo necesita un `Device` y una `Queue`, que `wgpu` da sin
   superficie. Exportar frames y validar planos ocurre en un proceso sin ventana; la ventana es un
   consumidor más del mismo render.
3. **Tres puertas, un solo vocabulario**, adaptadores finos sin lógica propia: la **API Rust** de
   los crates (`cargo doc`); la **CLI `lotte`** (`export`, `info`, `validate`, `import-anim`,
   `combine`, `render-thumb`) con entrada y salida **JSON**, encadenable desde cualquier lenguaje —
   y como el plano es un directorio de archivos abiertos, **el formato es la API**; y un **servidor
   MCP** (`rmcp`, SDK oficial de Model Context Protocol, Apache-2.0) que expone las mismas
   operaciones como herramientas para un modelo de lenguaje.
4. **La documentación es un artefacto generado**: cada `Edicion`, operación y archivo deriva su
   **JSON Schema** (`schemars`); ese esquema alimenta la ayuda de la CLI, las definiciones de
   herramientas del MCP y la validación al cargar. Un modelo de lenguaje lee el esquema, no un
   tutorial; los ejemplos vivos son los fixtures de test.

### Modelo de extensión — sin scripting embebido

No hay intérprete dentro del proceso (ni Lua, ni Python, ni WASM), por tres razones del autor:

1. **El scripting vive fuera del proceso.** Python, Bash, un modelo de lenguaje o cualquier otra
   cosa hablan con Lotte por la CLI, el MCP o los archivos.
2. **Las extensiones son Rust contra la API pública, y el core crece por propuestas.** Quien
   necesite algo que la API no da lo implementa como crate sobre los crates públicos o propone la
   extensión al core — con test y medición como todo lo demás. Un solo lenguaje, un solo conjunto de
   reglas.
3. **Un intérprete embebido es costo cierto por valor hipotético**: obliga a validar situaciones
   que hoy no existen, a dar soporte a comportamientos que podrían llegar a existir, y corre más
   lento que Rust.

Regla derivada: **la interfaz no tiene privilegios** — un cambio que agregue una acción a la
interfaz sin su `Edicion` u operación invocable desde la CLI no se acepta.

## Alcance complementario

Soporte, no foco; todo como nodos o pistas **aditivos** que no tocan el núcleo.

- **D-Img — Fondos e imágenes planas**: un fondo es un nodo de dibujo cuyo contenido es un bitmap
  (`peniko::Image`, que `vello` ya pinta; decodificación con `image`; capas de PSD con `psd`),
  con el mismo peg y la misma `z` que un dibujo vectorial. Sin edición de píxeles.
- **D-Audio — Audio básico, sin editor**: cargar, ver la onda bajo la línea de tiempo, reproducir en
  sincronía con los frames, *scrub*, marcar frames para bocas. Un tipo de pista más. Candidato:
  **Firewheel** (MIT/Apache; motor de grafo de audio con `cpal` debajo y *features* para alinear la
  versión de `glam`) + `symphonium` para cargar; la onda (pirámide de picos/RMS por bloques) es
  propia. No existe un editor de audio maduro en Rust.
- **D-Export — Imagen y video**: render fuera de pantalla con `vello` y lectura de píxeles
  (propio); secuencias PNG/EXR con `image`; para video, **`ffmpeg` como proceso externo** si el
  usuario lo tiene instalado — nunca enlazado ni distribuido (invocar un ejecutable no crea obra
  derivada; enlazar sus bibliotecas sí abriría la discusión).
- **D-Text — Texto vectorial**: `parley` + `skrifa` (Linebender; medido: `parley` declara la misma
  `skrifa 0.44` y `peniko 0.6` que `vello 0.10`) convierten texto a contornos de glifos; a partir
  de ahí es un `Contour` como cualquier otro.
- **Formatos propietarios**: Lotte no lee formatos de herramientas comerciales. El intercambio es
  por estándares abiertos (SVG para dibujos; DragonBones JSON para rigs; Lottie y glTF como
  exportadores candidatos).
- **D-Bench**: `criterion` como `[dev-dependencies]`, con líneas base guardadas fuera del repo y la
  regla de que un número de rendimiento en un PR es una corrida local con su salida pegada. Primer
  uso: `FCurve::evaluate` y `RigDag::evaluate_all` con jerarquía profunda **y** plana.

### D-Tab, medido — 2026-09-16

E8 midió las tres codificaciones sobre un fixture sintético de 3 500 canales × 17 500 claves, y QA lo
reprodujo de forma independiente en tres vueltas. Bytes: JSON 1,76 MB · CBOR 873,6 KB · safetensors
873,9 KB. Apertura de diez instancias con `criterion` en release: safetensors **≈2,3× más rápido que
CBOR** y **≈9–10× que JSON** (los milisegundos absolutos varían hasta 1,5× entre máquinas; las razones
no). **Predeterminado firmado: safetensors**, con el diccionario JSON al lado; CBOR disponible; JSON
solo para inspección. `serde_json` lleva el feature `float_roundtrip` para que el round-trip de `f64`
sea exacto (costo medido: +3 a +6 % al parsear los fixtures versionados, +37 % sobre una tabla JSON
de 17,6 MB).

### Convenciones fijadas durante la primera implementación — 2026-09-16

Decisiones chicas que aparecieron al construir el hito 1 y que un developer no podía tomar solo. Cada
una salió de un desacuerdo medido en un PR y tiene su candado ejecutable en el crate que la implementa.

| Convención | Qué dice | Dónde salió |
|---|---|---|
| Ángulos | Positivo = antihorario en el mundo Y-arriba y así se ve en pantalla; la cámara compone el volteo *después* de la rotación | A2 / A2b |
| Rendimiento de interfaz | Todo número de un criterio de UI se mide en **release**, que es el perfil del animador; el de `dev` se pega como contexto, marcado | D1 |
| Radio del hit-test | 12 **píxeles físicos**; el único candado que distingue las dos lecturas es el test con `scale_factor = 2.0` | C0 |
| Directorio del plano | No existe un tercer archivo de índice: la referencia a la biblioteca es un campo aditivo de `scene.json`; la lista de archivos que el guardado rota y marca sucios sigue cerrada (`scene.json`, `anim/*`) | E9 |
| Dibujos | Viven en la **biblioteca** (`rigs/<id>/drawings/*.svg`), nunca en el plano; se escriben con el mismo namespace `lotte:` que se importa (D-Draw), con el `viewBox` real para que abran enteros en cualquier editor, y se resuelven por identificador, nunca por ruta serializada | H3b-1 |
| Redondeo del *scrub* | La mitad de camino se aleja de cero (`f64::round`); como el frame se acota a `[0, max]` *después* del snap, en el rango válido equivale a "mitad hacia arriba" | D2 |
| Fusión de gestos | La clave de fusión es (nodo, atributo); asignar un dibujo dos veces al mismo peg dentro de la ventana es **un** paso | F3 |
| Capas | `lotte-app` **depende de `lotte-doc`** y el estado del editor es un `Document`: core → rig / timeline → format → doc → app. Las dos reglas duras no cambian | A0 |
| Evidencia visual | Una captura nunca es el candado; si se toma, solo de la ventana de Lotte y nunca se publica; los agentes no simulan el ratón ni la tableta en la máquina del autor | A3 / A5 / A8 |

### D-Lang, D-i18n y D-Doc — firmadas el 2026-09-14

> "Incluir que el sistema permita cambiar el idioma de manera fácil: un archivo de lenguaje en algún
> formato estándar de traducción, para que los diálogos se carguen desde esos archivos y sea fácil
> hacer traducciones."

- **D-Lang — identificadores en inglés.** El código de los cinco crates ya está en inglés
  (`RigDag`, `ViewportCamera`, `set_local_transform`), igual que toda dependencia; los tipos nuevos
  (`Edit`, `Pose`, `begin_gesture`) siguen esa convención. Documentación, commits e issues en
  español. Mezclar los dos en el árbol obligaría a un rename masivo más adelante.
- **D-i18n — la interfaz no tiene cadenas literales.** Todo texto visible sale de archivos
  **Fluent** (`.ftl`, Project Fluent — `fluent-bundle 0.16`, `unic-langid 0.9`, Apache-2.0 OR MIT),
  uno por idioma en `locales/<lang>/lotte.ftl`; `es` como idioma de desarrollo y `en` desde el
  primer día; idioma del sistema por defecto (`sys-locale 0.3`) y cambio en caliente desde el menú.
  Fluent es un estándar abierto nativo de Rust, maneja plurales y género, y lo entienden las
  plataformas de traducción (Weblate, Crowdin, Pontoon): un traductor no toca código. Alternativa
  medida y descartada: gettext `.po` (`gettext-rs` enlaza una biblioteca C). Entra en el hito 1
  (issue A8) porque externalizar textos el primer día cuesta nada y el último cuesta un barrido
  entero; el candado es un test que falla ante cualquier cadena literal en una llamada de UI.
- **D-Doc — el modelo de documento vive en el crate `lotte-doc`**, sobre `lotte-format`,
  `lotte-rig`, `lotte-timeline` y `lotte-core`; `lotte-app` y la CLI `lotte` son sus clientes
  (D-API). Decidido en vez de dejarlo entre paréntesis en F1.
