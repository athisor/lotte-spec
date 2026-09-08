# Arquitectura General del Sistema: Lotte Animation Engine

Este documento describe la arquitectura modular del workspace en Rust, las responsabilidades de cada crate, los flujos de datos y el modelo de memoria de **Lotte Animation Engine**.

---

## 1. Topología del Workspace de Cargo

```
Lotte/
├── Cargo.toml               # Configuración del workspace raíz
├── README.md                # Documento de presentación y guía rápida
├── prd.md                   # Requisitos de producto e investigación
├── docs/                    # Especificaciones técnicas completas
│   ├── architecture.md      # Este documento (arquitectura de crates y pipeline)
│   ├── repository-audit.md  # Auditoría técnica detallada de repositorios
│   ├── ui-widgets.md        # Especificación de widgets, visor de nodos y timeline
│   └── multiplane-parallax.md # Especificación de profundidad, multiplano y paralaje
└── crates/
    ├── lotte-core/          # Primitivas matemáticas y Grafo DAG de Pegs
    ├── lotte-rig/           # Mallas deformables (LBS), Slots y Attachments
    ├── lotte-timeline/      # F-Curves (Newton-Raphson), Canales, Celdas y Exposición
    ├── lotte-render/        # Cámara de Viewport, VectorItem y puente Kurbo/Vello
    └── lotte-format/        # Especificación JSON de proyectos (LotteProject)
```

---

## 2. Responsabilidades de los Crates

### 2.1. `lotte-core`
* **`Transform2D`:** Estructura que encapsula traslación, escala, rotación, shear y el **pivote excéntrico explícito** $P(x,y)$.
* **`RigDag`:** Grafo Acíclico Dirigido que almacena los `PegNode`.
  * Validación estricta anti-ciclos (`DagError::CycleDetected`).
  * Evaluación perezosa (*lazy evaluation*): cada nodo almacena `cached_global` y una bandera `dirty`. Al modificar un nodo, se invalida recursivamente su subárbol sin reevaluar todo el rig.

### 2.2. `lotte-rig`
* **`DeformableMesh`:** Malla 2D compuesta por vértices con pesos a múltiples Pegs (`BoneWeight`).
  * Implementa **Linear Blend Skinning (LBS)**:
    $$V_{\text{deformado}} = \sum_{j=1}^{K} w_j \cdot (M_{\text{global}, j} \cdot V_{\text{bind}, j})$$
* **`Slot`:** Desacopla la transformación cinemática de la apariencia visual. Controla visibilidad, orden de apilamiento ($Z$-order discreto) y el adjunto activo (`Attachment`).

### 2.3. `lotte-timeline`
* **`FCurve`:** Función temporal continua parametrizada por `Keyframe` con interpolación `Hold`, `Linear` o `Bezier`.
  * Utiliza el solver analítico híbrido **Newton-Raphson con fallback a bisección** para evaluar $Y(t)$ dado un fotograma $X$ en tiempo constante y sub-microsegundo.
* **`ExposureTrack`:** Hoja de exposición para sustitución de dibujos. Cada celda referencia un `(LevelId, DrawingIndex)`, desacoplando la exposición del rig.

### 2.4. `lotte-render`
* **`ViewportCamera`:** Cámara 2.5D con control de encuadre, zoom, rotación y distancia focal.
  * Transforma coordenadas de forma bidireccional entre píxeles de pantalla y espacio de escenario (*World $\leftrightarrow$ Screen*).
  * Aplica el factor de escala por profundidad multiplano $S_z = D / (D - z)$ para paralaje óptico (mayor $z$ = más cerca; $z < D$).
* **`VectorItem`:** Elementos vectoriales analíticos representados mediante `kurbo::BezPath` vinculados a un `NodeId` de Peg.
  * Proporciona la conversión de matrices afines `glam::Affine2` a `kurbo::Affine` para despacho directo a GPU con Vello.

### 2.5. `lotte-format`
* **`LotteProject`:** Esquema canónico serializable en JSON con Serde.
  * Contiene el rig completo, los slots, las curvas de propiedades F-Curves y las pistas de exposición.
  * **Decidido el 2026-09-08** ([`decisiones-modelo-de-datos.md`](decisiones-modelo-de-datos.md), D-Fmt / D-Tab / D-Draw): el documento pasa a ser un **directorio de plano** — `scene.json` (instancias, timing, cámara), `anim/<instancia>.anim.json` + `.anim.safetensors` (diccionario de canales + tablas de claves y exposición como tensores) — más una biblioteca de definiciones (`definition.json` + un `.svg` por dibujo con namespace `lotte:`). `LotteProject` tal como existe hoy es el documento de *una* definición con sus curvas; los issues E1, E7 y E8 lo reparten en esas piezas de forma aditiva.

---

## 3. Pipeline de Ejecución por Fotograma

```
1. Actualización Temporal
   └── Para cada canal activo: FCurve::evaluate(frame)
       └── Aplica valores a Transform2D de los PegNodes en RigDag

2. Evaluación Cinemática
   └── RigDag::evaluate_all()
       └── Recorrido topológico perezoso: M_global = M_parent * M_local

3. Deformación de Mallas (Skinning)
   └── DeformableMesh::evaluate_skinning(&global_matrices)
       └── Suma ponderada de vértices por influencia de Pegs (LBS)

4. Proyección de Cámara y Multiplano
   └── ViewportCamera::world_to_screen_affine(viewport_size)
       └── Aplica factor Sz = D / (D − z) para paralaje óptico (mayor z = más cerca; z < D)

5. Despacho Gráfico a GPU (Vello)
   └── kurbo::BezPath con kurbo::Affine despachado a vello::Scene
       └── Rasterización analítica en GPU vía compute shaders de wgpu
```
