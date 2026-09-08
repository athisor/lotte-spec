
# Product Requirements Document (PRD)

**Documento:** Especificación de Investigación Técnica y Arquitectura Base

**Nombre Oficial del Proyecto:** Lotte (Lotte Animation Engine / Lotte 2D)

**Homenaje e Identidad:** En honor a **Lotte Reiniger**, pionera de la animación de siluetas articuladas (1920, *Las aventuras del príncipe Achmed*), cuyas bisagras y articulaciones físicas sobre cartulina constituyen la base conceptual y mecánica directa de los pivotes (*pegs*) y jerarquías de piezas modernas.

**Diferenciación Técnica:** Para evitar colisiones y confusiones con el formato *Lottie* de Airbnb:
* La denominación formal en repositorios y documentación es **Lotte Animation Engine** o **Lotte 2D**.
* El espacio de nombres de crates en Cargo adoptará el prefijo explícito: `lotte-core`, `lotte-viewport`, `lotte-rig`, `lotte-timeline`, `lotte-render`.
* Comando de CLI / binarios: `lotte` (ej. `lotte init`, `cargo run -p lotte-viewport`).

**Objetivo:** Diseñar un motor y entorno de animación 2D vectorial con aceleración por GPU, enfocado en cut-out avanzado mediante jerarquías de transformación afín (pegs/pivotes), mallas deformables y curvas de interpolación, tomando como horizonte funcional las herramientas profesionales de animación cutout y como hito mínimo de ejecución un pipeline esquelético estilo Spine/DragonBones.

---

### 1. Resumen Ejecutivo y Metodología

El proyecto busca crear un motor de animación 2D nativo en Rust desacoplado de suites monolíticas heredadas. El agente asignado debe auditar cuatro repositorios de referencia de código abierto para documentar estructuras de datos, solvers matemáticos y modelos de representación temporal, determinando qué componentes pueden adoptarse directamente, cuáles deben portarse y cuáles deben descartarse.

```
┌─────────────────────────────────────────────────────────┐
│                 Lotte Animation Engine                  │
├──────────────────────────┬──────────────────────────────┤
│ Interfaz de Usuario      │ Slint / egui                 │
├──────────────────────────┼──────────────────────────────┤
│ Motor Gráfico Vectorial  │ Vello + wgpu + kurbo         │
├──────────────────────────┼──────────────────────────────┤
│ Cinemática y Deformación │ Grafo DAG (Pegs) + Skinning  │
├──────────────────────────┼──────────────────────────────┤
│ Motor Temporal           │ F-Curves desacopladas        │
└──────────────────────────┴──────────────────────────────┘
```

---

### 2. Matriz de Auditoría de Repositorios

El agente debe inspeccionar el código fuente de los siguientes tres proyectos, abstrayendo las soluciones de ingeniería según los objetivos específicos asignados:

| Proyecto           | Repositorio Base / Módulos Críticos | Foco de Inspección | Componentes a Extraer / Aprender | Componentes a Descartar |
| ------------------ | ------------------------------------- | ------------------- | -------------------------------- | ----------------------- |
| **Graphite** | `graphite.rs`                       |                     |                                  |                         |

• `graphene-core`

• Integración `vello` / `kurbo` | Renderizado vectorial acelerado por GPU y manipulación de curvas 2D. | Pipeline de dibujo con Vello, manejo analítico de curvas Bézier y conversión de coordenadas pantalla/documento. | Frontend en Svelte/TypeScript, wrapper WebAssembly y despachador de documentos estáticos sin tiempo. |
| **OpenToonz** | `opentoonz`

• `toonzlib/plastictool/`

• `toonzlib/tstageobject.cpp`

• `toonzlib/txsheet.cpp` | Deformación de mallas poligonales 2D y separación de exposición. | Algoritmos de triangulación 2D (*Constrained Delaunay*), asignación de pesos de vértices a huesos (*skinning*) y desacoplamiento celda-dibujo en Xsheet. | Base de código en C++ tradicional, dependencias Qt monolíticas y formatos de archivo binarios propios. |

| **DragonBones** | `DragonBonesCPP`

• `src/dragonBones/armature/`

• `src/dragonBones/geom/` | Serialización de jerarquías esqueléticas y evaluación de matrices. | Especificación del esquema de datos (huesos, slots, mallas, pesos) y cálculo plano de transformaciones locales a globales. | El plugin antiguo `DesignPanel` en ActionScript 3 y cualquier referencia al editor cerrado *DragonBones Pro*. |

---

### 3. Requisitos Funcionales del Motor

#### 3.1. Subsistema de Cinemática y Transformación (Rigging por Pegs)

* **Abstracción por Pivotes:** El motor no utilizará huesos rígidos con longitud fija como primitiva base. Empleará **PegNodes**: puntos en el espacio bidimensional con definición de pivote explícito $P(x,y)$, traslación $T(x,y)$, rotación $\theta$ y escala $S(x,y)$.
* **Propagación Matricial:** Cálculo en cascada de matrices afines $3\times3$ mediante una pasada lineal $O(N)$ sobre un Grafo Acíclico Dirigido (DAG) ordenado topológicamente:

$$
M_{local} = T(t_x, t_y) \times T(p_x, p_y) \times R(\theta) \times S(s_x, s_y) \times T(-p_x, -p_y)
$$

$$
M_{global} = M_{parent\_global} \times M_{local}
$$

* **Hitos de Escalamiento:**

1. *Fase 1 (Cut-out Rígido):* Los trazos vectoriales se asocian de forma unívoca a un `PegNode`, heredando su matriz de transformación completa.
2. *Fase 2 (Mallas con Influencia):* Soporte de mallas trianguladas 2D donde los vértices calculan su posición deformada sumando la influencia de múltiples Pegs vecinos (*Linear Blend Skinning*).
3. *Fase 3 (Deformadores de Curva):* Puntos de control Bézier interpolados paramétricamente para generar curvatura elástica sobre los trazos de contorno y relleno.

#### 3.2. Subsistema de Renderizado Vectorial (GPU Viewport)

* **Backend de Dibujo:** Exclusivamente **Vello** sobre **`wgpu`**.
* **Preservación Analítica:** Las figuras no se rasterizan a texturas estáticas intermedias en reposo. Se conservan como definiciones geométricas puras vía `kurbo::BezPath`.
* **Transformaciones en Memoria de Video:** Las matrices globales del rig se envían como buffers de uniformes o almacenamiento a la GPU para evitar transferencias redundantes de geometría por el bus PCIe.

#### 3.3. Subsistema Temporal y Canales de Animación

* **Desacoplamiento de Propiedades:** Cada parámetro numérico modificable (rotación, posición X, posición Y, escala) constituye un canal de animación independiente.
* **Evaluación de Fotogramas Clave:** Soporte para tres tipos de interpolación:
* *Constant / Hold:* Retención de valor sin interpolación.
* *Linear:* Interpolación directa constante.
* *Bézier (F-Curve):* Control de tangentes temporales de entrada y salida (*ease-in / ease-out*).

---

#### 3.4. Alcance Complementario: Medios (agregado el 2026-09-08)

Soporte, no foco — decidido por el autor y detallado en
[`docs/decisiones-modelo-de-datos.md`](docs/decisiones-modelo-de-datos.md) (D-Img, D-Audio,
D-Export, D-Text):

* **Fondos e imágenes planas** como nodos de dibujo raster (`peniko::Image` vía `vello`; decodificación con `image`, capas PSD con `psd`), con el mismo peg y la misma profundidad `z` que un dibujo vectorial. Sin edición de píxeles.
* **Audio básico**: cargar, ver la onda bajo la línea de tiempo, reproducir en sincronía con los frames, *scrub*, marcar frames para *lip-sync*. Sin edición de audio.
* **Exportación**: secuencias PNG/EXR desde el render de `vello`; video invocando `ffmpeg` como proceso **externo**, nunca enlazado ni distribuido.
* **Texto vectorial** convertido a contornos (`parley` + `skrifa`, la misma familia que `vello`), animable como cualquier dibujo.
* **No se leen formatos de herramientas comerciales.** El intercambio es por estándares abiertos (SVG y DragonBones JSON; Lottie y glTF como exportadores candidatos).

### 4. Marco Legal y Compatibilidad de Licencias

El agente debe asegurar la viabilidad de distribución del código según las licencias de origen:

```
┌─────────────────────────────────────────────────────────────┐
│                   Código Abierto de Origen                  │
├──────────────────────────┬──────────────────────────────────┤
│ Graphite                 │ Apache 2.0 / MIT (Permisiva)     │
│ DragonBonesCPP           │ MIT (Permisiva)                  │
│ OpenToonz                │ BSD 3-Clause (Permisiva)         │
└──────────────────────────┴──────────────────────────────────┘

```

#### Reglas de Ingesta y Limpieza de Código

* **De Graphite, DragonBonesCPP y OpenToonz:** Al utilizar licencias permisivas (Apache 2.0, MIT y BSD-3-Clause), es jurídicamente viable estudiar, portar algoritmos a Rust o adaptar estructuras de datos directamente, siempre que se mantengan los créditos y avisos de copyright originales en los encabezados correspondientes.
* **De proyectos con licencia copyleft (GPL):** no se lee su código. Lo que haga falta se toma de estándares abiertos, de matemática publicada o de proyectos permisivos; ninguna línea se copia ni se traduce.

#### Licencia Final Sugerida para el Proyecto

* **Opción A (Recomendada - Máxima Adopción): Dual License Apache 2.0 / MIT**
  Alineada con el estándar de la comunidad de Rust y el ecosistema Linebender (Vello, Kurbo). Exige que ningún código copyleft entre al árbol de dependencias ni al repositorio.
* **Opción B (Copyleft Fuerte): GNU GPLv3** — descartada: el proyecto es permisivo y no incorpora código copyleft.

---

### 5. Tareas Concretas Asignadas al Agente

1. **Auditoría de Plastic Tool (OpenToonz):** Inspeccionar `toonz/sources/toonzlib/plastictool/` y extraer en un informe la representación en memoria de las mallas triangulares y el algoritmo que asocia los pesos de los vértices a los controladores.
2. **Especificación del Formato de Intercambio:** Analizar la serialización JSON de `DragonBonesCPP` para redactar una especificación técnica de esquema (`schema.json`) orientada a rigs basados en Pegs/Pivotes.
3. **Evaluador de curvas Bézier 1D:** redactar la lógica matemática de interpolación de keyframes (Newton-Raphson con caída a bisección, el método de `UnitBezier` de WebKit, BSD) lista para implementar en Rust sin dependencias externas.
4. **Prototipo de Integración Vello + Peg:** Diseñar el flujo de ejecución para tomar una figura vectorial de `kurbo`, multiplicarla por la `Affine2` resultante del rig y despacharla al command encoder de Vello.
