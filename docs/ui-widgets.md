# Especificación de Interfaz de Usuario y Widgets (Lotte UI)

Este documento define la arquitectura y especificación técnica de los cinco subsistemas de interfaz y widgets interactivos de **Lotte Animation Engine**.

---

## 1. Visor de Nodos (*Node Graph / Stage Schematic*)

Inspirado en el *Stage Schematic* de OpenToonz y el `GraphEdit` de Godot, proporciona una vista espacial y no lineal de la jerarquía de la escena.

### 1.1. Tipos de Nodos
* **Peg Nodes (Verdes/Azules):** Representan sistemas de coordenadas y pivotes (`lotte_core::PegNode`). Poseen un puerto superior de entrada (*Parent In*) y puertos inferiores de salida (*Children Out*).
* **Drawing / Element Nodes (Morados):** Contenedores de trazos vectoriales y slots de visualización (`lotte_rig::Slot`).
* **Deformer Nodes (Naranjas):** Modificadores de malla 2D (Linear Blend Skinning / ARAP) y deformadores de curvas Bézier.
* **Composite Nodes:** Nodos de agrupación de capas con orden de apilamiento explícito.

### 1.2. Interacción y Comportamiento
* **Navegación:** Lienzo infinito con desplazamiento (*Pan* con botón central o barra espaciadora) y escala (*Zoom* con rueda del ratón).
* **Conexión por Cables Bézier:** Trazado de cables elásticos con detección de proximidad e imantado (*snap-to-port*).
* **Prevención Visual de Ciclos en Tiempo Real:** Si el usuario intenta conectar un cable que generaría un bucle de dependencia padre-hijo, el cable se tiñe de rojo y la conexión se rechaza, mapeando directamente a `lotte_core::DagError::CycleDetected`.

---

## 2. Timeline Modular (Dope Sheet, Xsheet y Editor de Curvas)

La línea de tiempo se divide en tres vistas especializadas y sincronizadas:

### 2.1. Dope Sheet (Gestión de Fotogramas Clave)
* Filas jerárquicas expandibles por cada Peg (`Position X/Y`, `Rotation`, `Scale X/Y`, `Shear X/Y`, `Z-Depth`).
* **Glifos de Keyframe por Código de Color:**
  * **Rojo / Cuadrado:** Fotograma de retención (*Hold*).
  * **Verde / Círculo:** Interpolación lineal.
  * **Azul / Rombo:** Curva Bézier continua (*Ease-in / Ease-out*).
* Herramientas de selección en caja (*Box Select*) para retemporizado y desplazamiento masivo de claves.

### 2.2. Exposure Sheet / Xsheet (Sustitución de Dibujos)
* Representación en columnas de celdas basada en `lotte_timeline::ExposureTrack`.
* Cada celda contiene una referencia indirecta `(LevelId, DrawingIndex)`.
* **Mecánica de Sustitución (*Drawing Substitution*):** Permite cambiar de dibujo activo (bocas fonéticas para *lip-sync*, variantes de manos o pestañeo) mediante un carrusel o teclado numérico sin alterar las curvas de transformación ni duplicar geometría.

### 2.3. Graph Editor (Editor de F-Curves)
* Visualización gráfica de las funciones temporales calculadas por el solver Newton-Raphson (`lotte_timeline::FCurve`).
* **Manijas de Tangente ($C_0, C_1$):**
  * *Smooth:* Manijas colineales con ángulo bloqueado.
  * *Broken:* Manijas independientes para esquinas anguladas y rebotes.
  * *Flat:* Tangente horizontal perfecta ($\text{pendiente} = 0$) para anticipaciones y amortiguaciones.

---

## 3. Selector y Motor de Estilo de Trazo (*Stroke Engine*)

Define la apariencia y parámetros geométricos de los trazos vectoriales analíticos (`kurbo::BezPath`):

* **Perfiles de Grosor (*Variable Width Profiles*):**
  * *Uniforme:* Grosor constante a lo largo de la curva.
  * *Sensible a Presión:* Modulación por presión de tableta gráfica (estilógrafo / pincel de cerda).
  * *Paramétrico:* Perfil Bézier editable de grosor vs. longitud de arco.
* **Geometría de Trazado:**
  * **Line Caps:** *Round* (redondeado), *Butt* (plano en vértice), *Square* (extensión cuadrada).
  * **Line Joins:** *Miter* (esquina viva con límite de inglete), *Round* (unión redondeada), *Bevel* (chaflán plano).
* **Líneas Discontinuas (*Dash Patterns*):**
  * Control de secuencias `[guión, espacio, guión, espacio]` con desfase dinámico (*dash offset*).

---

## 4. Sistema de Color y Paletas Dinámicas (*Palette-Linked Styles*)

A diferencia de editores genéricos donde los colores se graban estáticamente en los vértices, Lotte utiliza **Paletas Indexadas**:

* **Estilos Vinculados (*Linked Palette Styles*):**
  * Los trazos y rellenos almacenan un `ColorStyleId`.
  * La paleta del personaje almacena la definición del color o gradiente.
  * Modificar una muestra en la paleta actualiza instantáneamente todos los fotogramas y niveles de dibujo del proyecto.
* **Selector de Color (Color Picker):**
  * Rueda de color y triángulo/cuadrado HSV con valores numéricos RGB, Hexadecimal y canal Alfa.
  * Editor de degradados lineales y radiales con puntos de parada (*stops*) interactivos.

---

## 5. Gizmo de Manipulación en el Viewport

Superpuesto sobre la vista de Vello:

* **Pivote Excéntrico:** Punto de mira circular que representa `Transform2D::pivot`. Permite mover el centro de giro independientemente de la geometría.
* **Anillos de Rotación y Flechas de Traslación:** Manipulador local ortogonal para traslación $X/Y$ y anillo periférico para rotación angular con lectura en grados.
* **Visualizador de Esqueleto / Alambre:** Líneas guía conectando pivotes padres e hijos, en tributo a las articulaciones de alambre de Lotte Reiniger.
