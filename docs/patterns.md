# Patrones de animación 2D

## Procedencia

| | |
|---|---|
| Redacción original | Dictado por Miguel Oliva |
| Incorporado al repositorio | 2026-09-07 |
| Reducción a esta versión | 2026-09-07 |
| Fuentes consultadas en esta reducción | Ninguna. |

Los proyectos de referencia auditados en [`referencias/`](referencias/) **no** son fuente de este
documento: se escribió antes y desde la práctica de la animación 2D.

---

## 1. Separación de trazo y relleno en piezas articuladas

**El problema.** En los formatos vectoriales corrientes, una figura une su contorno y su relleno en
un solo objeto. En animación de recorte esa unión es un obstáculo: cuando un antebrazo rota sobre un
torso, el *relleno* del antebrazo debe tapar las líneas del torso que quedan detrás, pero su *línea
de contorno* no debe cortar visualmente el cuerpo adyacente. Relleno y contorno necesitan orden de
apilamiento y reglas de visibilidad independientes.

**El modelo.** Cada dibujo se descompone en cuatro sub-capas ordenadas de abajo hacia arriba:

```
  (encima)      luces, texturas, brillos
  trazos        contornos con grosor modulado
  rellenos      regiones cerradas de color
  (debajo)      guías de boceto, sombras base
```

Las regiones de relleno no duplican las coordenadas de los contornos: se definen por referencia a la
geometría de los trazos existentes, o a trazos de delimitación que no se dibujan.

**Lo que este documento no resuelve.** La referencia por indirección exige que cada trazo tenga
identidad propia y estable dentro del dibujo. Definir esa identidad —y qué pasa con las regiones
cuando un trazo se edita o se borra— es un problema abierto.

---

## 2. Costuras visibles en las articulaciones

**El problema.** Dos piezas superpuestas que giran sobre una unión circular —muslo y pantorrilla,
bíceps y antebrazo— dejan ver el contorno del disco de unión sobre la pieza inferior. Esa línea
delata que el personaje está partido en trozos rígidos.

**El modelo.** No hace falta que el dibujante resuelva la unión a mano en cada fotograma: se resuelve
en la pasada de composición, y es una **regla de orden de dibujo**, no una operación booleana entre
regiones. Dadas una pieza padre y una pieza hija que comparten una articulación, el orden es:

```
1. rellenos del padre
2. trazos del padre
3. copia de los rellenos de la hija — sin sus trazos — recortada a la región de articulación
4. rellenos de la hija
5. trazos de la hija
```

El paso 3 es todo el mecanismo: el relleno de la hija inunda la intersección y cubre la línea
divisoria del padre, y como no arrastra sus propios trazos no agrega ninguna línea nueva. La unión
se lee continua en cualquier ángulo de rotación.

**Lo que este documento no resuelve.** Dos cosas, y son las dos importantes. Primero, **qué define la
región de articulación**: sin una regla que la determine, el paso 3 no se puede construir. Segundo,
cómo interactúa ese parche con el orden de apilamiento del resto de las piezas.

---

## 3. Deformación a lo largo de un eje curvo

**El problema.** Los pivotes rígidos obligan a que las extremidades se mantengan rectas. La
curvatura orgánica de un brazo de goma, un mechón de pelo o una ceja necesita deformar el trazo a lo
largo de un eje curvo continuo, sin el costo de una malla triangulada densa.

**El modelo.** Una curva paramétrica cúbica actúa como eje de deformación.

*En reposo*, cada punto del dibujo se asocia a la curva por dos coordenadas: la **longitud de arco
normalizada** $u \in [0,1]$ del punto de la curva más cercano, y la **distancia perpendicular con
signo** $v$ hasta ese punto.

*Deformado*, para cada punto se recupera el parámetro de curva $t'$ cuya longitud de arco
normalizada vale $u$, y se evalúa:

$$V' = C(t') + v \cdot \hat{N}(t')$$

Es esencial que $u$ sea longitud de arco y no el parámetro de la curva: una cúbica no está
parametrizada por arco, y usar el parámetro directamente apiña los puntos donde la parametrización
corre rápido, produciendo deformación desigual. Convertir entre ambos exige integrar la longitud de
arco y después invertirla.

La ventaja es la densidad de control: cuatro puntos de control gobiernan cientos de vértices, y el
trazo sigue siendo vectorial.

**Lo que este documento no resuelve.** La proyección al punto más cercano de una cúbica no tiene
forma cerrada y **no siempre es única**. Y el modelo tiene un modo de falla propio: cuando $|v|$
supera el radio de curvatura del lado cóncavo, la correspondencia deja de ser inyectiva y la
geometría se pliega sobre sí misma. Hay que decidir qué hace el motor en ese caso.

---

## 4. Coordinar muchos canales en un giro de personaje

**El problema.** Un giro de cabeza —de frente a tres cuartos a perfil— coordina decenas de
parámetros a la vez: posición de los ojos, escala de la nariz, curvatura de la mandíbula, y además
el cambio de dibujo de bocas y orejas. Ajustarlos uno por uno en la línea de tiempo no escala.

**El modelo.** Un plano de control $(u,v)$ normalizado, con un conjunto de poses clave situadas como
muestras en ese plano. Al arrastrar el punto de control, cada pose contribuye según un peso
$W_k(u,v)$.

Los pesos deben cumplir dos condiciones, y la segunda es la que se olvida: que sumen 1, y que en la
posición de cada muestra esa muestra reciba peso 1 y las demás 0 —o la pose clave no se reproduce
exactamente cuando el animador se para justo encima de ella—. Una ponderación por distancia inversa
simple no cumple la segunda condición sin un caso especial, y además es global: toda muestra influye
en todo punto. Una interpolación baricéntrica sobre una triangulación de las muestras sí cumple
ambas y tiene soporte local. **No son intercambiables.**

Los canales continuos se combinan así. Los **canales discretos** —qué dibujo de boca se expone— no
se pueden promediar: necesitan una regla de selección propia, por ejemplo la muestra de mayor peso,
con histéresis para que el dibujo no parpadee al cruzar una frontera.

**Lo que este documento no resuelve.** Qué es exactamente una pose: qué conjunto de canales abarca,
y si una misma pose puede mezclar canales continuos y discretos.

---

## 5. Recoloreo global

**El problema.** Si cada región almacena su color como valor literal, cambiar el tono de piel de un
personaje que aparece en cientos de planos obliga a reescribir todos los archivos, con el riesgo de
inconsistencia que eso trae.

**El modelo.** Indirección. Los dibujos no almacenan colores: almacenan un identificador estable.
El proyecto mantiene una tabla que asocia cada identificador con su definición —color sólido,
degradado, lo que sea—. Cambiar la entrada de la tabla cambia todo lo que la referencia, sin tocar
los datos de los trazos.

**Lo que este documento no resuelve.** Qué pasa cuando dos proyectos con tablas distintas se
combinan, y si las tablas admiten variantes o herencia para versiones de color por plano.

---

## 6. Grosor de línea variable

**El problema.** Un trazo de grosor uniforme se lee mecánico. Pero convertir un trazo de grosor
variable en geometría fija destruye la posibilidad de seguir editando la curva.

**El modelo.** El trazo se almacena como una **línea central** paramétrica más una **envolvente de
anchura**: dos funciones continuas de la longitud de arco que dan la distancia perpendicular hacia
cada lado. El contorno cerrado que se rasteriza se genera desplazando la línea central según esas
dos funciones y cerrando los extremos con los remates elegidos.

Esto permite grosor uniforme, modulado por presión de tableta, o definido por un perfil editable, sin
cambiar la representación.

**Lo que este documento no resuelve.** El desplazamiento exacto de una curva cúbica **no es** una
curva cúbica: solo se puede aproximar, y esa aproximación tiene una tolerancia que es una decisión
de calidad. Tampoco se define cómo se representan las dos funciones de anchura.

---

## Resumen

| Problema | Modelo abstracto |
|---|---|
| Trazo y relleno acoplados en piezas articuladas | Cuatro sub-capas ordenadas, rellenos por referencia |
| Costura visible en la articulación | Regla de orden de dibujo: relleno de la hija, sin sus trazos, sobre los trazos del padre |
| Extremidades rígidas | Eje curvo con coordenadas de arco y perpendicular |
| Coordinar muchos canales en un giro | Plano de poses con pesos de partición de la unidad |
| Recoloreo global | Indirección por identificador contra una tabla del proyecto |
| Línea de grosor uniforme | Línea central más envolvente de anchura |

Los huecos abiertos de cada modelo están al pie de su sección.
