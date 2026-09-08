# Especificación de Profundidad, Multiplano y Paralaje

Este documento formaliza el modelo de profundidad bidimensional y de perspectiva multiplano para **Lotte Animation Engine**, conectando su fundamentación histórica con las ecuaciones de proyección matemática.

---

## 1. Fundamento Histórico: La Mesa Multiplano de Lotte Reiniger

Entre 1923 y 1926, durante la producción de *Die Abenteuer des Prinzen Achmed*, **Lotte Reiniger inventó la primera mesa multiplano de la historia del cine**.

Para evitar que los fondos de cartulina y las siluetas de los personajes se mezclaran en un plano plano sin atmósfera, construyó un armazón de madera con **placas de cristal dispuestas a diferentes distancias verticales** sobre la caja de luz. La cámara cenital filmaba a través de las capas de cristal:
* Las siluetas articuladas se situaban en los niveles intermedios.
* Los fondos y elementos arquitectónicos se colocaban en los niveles inferiores.
* Los elementos de primer plano (hojas, ramas, columnas) se suspendían en los niveles superiores más cercanos al objetivo.

Al mover la cámara o las placas de cristal lateralmente, se producía un **desplazamiento de paralaje óptico natural**, donde los planos cercanos se movían más rápido que los fondos lejanos. Este principio es la base del sistema de profundidad de Lotte.

---

## 2. Los Dos Niveles de Profundidad en el Motor

Lotte distingue rigurosamente dos dimensiones de profundidad:

| Dimensión | Parámetro | Ámbito | Comportamiento |
| :--- | :--- | :--- | :--- |
| **Micro-Profundidad** | `z_order: i32` | Local al Slot / Pieza (`lotte_rig::Slot`) | **Orden de apilamiento discreto:** Define si el antebrazo se dibuja delante o detrás del torso en un giro de 360°, sin alterar escala ni perspectiva. |
| **Macro-Profundidad** | `z: f32` | Espacio Escenario / Peg (`lotte_core::Transform2D`) | **Profundidad espacial continua:** Posición en el eje óptico $Z$ perpendicular a la cámara para generar perspectiva y paralaje 2.5D. |

---

## 3. Modelo Matemático de Proyección y Paralaje

Sea una cámara con distancia de proyección focal $D$ (distancia al plano focal principal $Z = 0$) y posición en el escenario $(X_{\text{cam}}, Y_{\text{cam}})$. La cámara está en $z = D > 0$: **mayor $z$ = más cerca de la cámara**, la convención de OpenToonz y de los animadores de recorte (decisión D1 en [`decisiones-modelo-de-datos.md`](decisiones-modelo-de-datos.md)).

### 3.1. Factor de Proyección de Profundidad ($S_z$)
Para cualquier objeto situado a una profundidad continua $z$:

$$S_z = \frac{D}{D - z}, \qquad z < D$$

* **$z = 0$ (Plano Focal Principal):** $S_z = 1.0$. El objeto se visualiza a su tamaño de dibujo nativo (1:1).
* **$z > 0$ (Primer Término / *Foreground*):** $S_z > 1.0$. El objeto se acerca a la cámara y se magnifica.
* **$z < 0$ (Fondo Lejano):** $S_z < 1.0$. El objeto se aleja y se reduce proporcionalmente a la distancia.
* **$z \geq D$:** el objeto está en la cámara o detrás; $S_z$ no está definido. Es un **error de validación** (aviso y *clamp* a $z < D$), nunca un `NaN` en pantalla.

### 3.2. Ecuación de Paralaje en Panorámica de Cámara
Cuando la cámara efectúa un movimiento de paneo $(\Delta X_{\text{cam}}, \Delta Y_{\text{cam}})$:

$$\Delta X_{\text{pantalla}} = \Delta X_{\text{cam}} \cdot S_z = \Delta X_{\text{cam}} \cdot \frac{D}{D - z}$$
$$\Delta Y_{\text{pantalla}} = \Delta Y_{\text{cam}} \cdot S_z = \Delta Y_{\text{cam}} \cdot \frac{D}{D - z}$$

* **Consecuencia:** Un fondo con $z = -3D$ se moverá a $1/4$ de la velocidad de la cámara ($\Delta X \cdot 0.25$). Un primer plano con $z = 0.5D$ se moverá al doble de la velocidad ($\Delta X \cdot 2.0$), produciendo una sensación tridimensional de inmersión.

### 3.3. Herramienta de Compensación "Maintain Size"
Para permitir que un director de arte componga una escena fijando el tamaño visual exacto de un elemento y luego le asigne profundidad sin romper el encuadre:

$$\text{Escala}_{\text{reposo}} = \frac{D - z}{D} = \frac{1}{S_z}$$

Al aplicar esta escala inversa en reposo, el elemento luce exactamente igual en el encuadre inicial, pero reacciona con el paralaje correcto cuando la cámara comience su movimiento.

---

## 4. Vistas de la Interfaz para Multiplano

Para controlar la profundidad en la interfaz de usuario:
* **Camera View (Vista Principal):** Muestra la proyección final con paralaje acelerada por GPU.
* **Top View (Vista Superior):** Proyección ortográfica en el plano $X-Z$ que visualiza el campo visual de la cámara (cono de visión / *frustum*) y las láminas virtuales de cristal de cada Peg. Permite al animador arrastrar capas en profundidad con el ratón.
* **Side View (Vista Lateral):** Proyección ortográfica en el plano $Y-Z$ para ajustar la elevación y distancia focal.
