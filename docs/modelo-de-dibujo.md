# Modelo de dibujo — versión 1

Especifica qué es un dibujo y cómo se guarda. Enmienda a D-Draw del 2026-11-15
([`decisiones-modelo-de-datos.md`](decisiones-modelo-de-datos.md)). Detalle de las herramientas que
lo producen: [`requisitos-animador.md`](requisitos-animador.md) §5.

**Estado.** Especificado. Las formas JSON y SVG tienen una primera implementación fuera de este
repositorio, hecha por un conversor externo, y su ida y vuelta se midió con 2 482 dibujos reales
(124 214 trazos) sin una sola diferencia. En `lotte-rig` el dibujo sigue siendo la lista de
trazados con apariencia del hito 1; llevarlo a este modelo es trabajo pendiente.

## La idea

Un dibujo es una lista de **trazos**. Cada trazo dice tres cosas:

1. por dónde pasa: su línea central;
2. si se ve como línea, con qué color y con qué grosor;
3. qué color hay **a su izquierda** y **a su derecha**.

Una zona de color no es un objeto: es lo que queda encerrado por los trazos que nombran ese color
hacia adentro. Así se cumple, por construcción, el requisito «si muevo la línea, el color la
sigue».

Con esa sola pieza se cubre todo lo que D-Draw pedía:

| lo que hace el animador | cómo queda |
|---|---|
| lápiz, línea, polilínea | trazo con línea y grosor constante |
| lápiz con presión | trazo con línea y perfil de grosor |
| pincel vectorial | trazo **cerrado, sin línea, con color hacia adentro** |
| bote de relleno | color a un lado en los trazos que rodean la zona |
| cerrar un hueco sin dibujar | trazo **invisible** |

**`Contour` deja de ser una segunda primitiva.** Es el caso «trazo cerrado, sin línea, con color
hacia adentro». La unión de pinceladas del mismo color sigue siendo la de `linesweeper`; lo que
cambia es cómo se guarda el resultado.

## Las piezas

### Dibujo

| campo | qué es |
|---|---|
| `version` | `1` |
| `state` | `derived` (igual a su origen), `adopted` (editado aquí), `new` (creado aquí) |
| `units` | `{name, y}`: unidad de las coordenadas y sentido del eje vertical, declarados |
| `source` | opcional: `{kind, sha256, path}` del archivo del que se derivó |
| `opaque` | lista de lo que el origen trae y el conversor no descifró |
| `palette` | `{id: {name, palette, kind, rgba}}`; `kind` = `solid`, `gradient` o `texture` |
| `gradients` | `{id: [matriz]}` para los rellenos que no son sólidos |
| `arts` | siempre cuatro, en este orden: `underlay`, `colour`, `line`, `overlay` |

Cada arte tiene `layers` (capas ordenadas) y cada capa tiene `strokes` (trazos ordenados).

### Trazo

| campo | qué es |
|---|---|
| `id` | identificador estable y único en el dibujo (D-Id, D3) |
| `points` | `[[x, y, es_ancla]]`: anclas y tiradores, en orden |
| `geometry_of` | en vez de `points`: el id de un trazo anterior del mismo arte cuyos puntos comparte |
| `closed` | el primer punto es igual al último |
| `line` | id del color de la línea; sin este campo el trazo no se ve como línea |
| `width` | medio ancho de la línea |
| `profile` | `{knots: [[t, izquierda, derecha]], interp, raw}`: grosor variable |
| `fill_left`, `fill_right` | id del color a cada lado |
| `invisible` | el trazo no se dibuja: solo cierra zonas o delimita color |

Los campos que no aplican se omiten. Un trazo lleva `points` o `geometry_of`, nunca los dos.

**Lados.** Izquierda y derecha son los lados geométricos al recorrer el trazo en el orden de sus
puntos, en el mundo Y-arriba (D-F). Un contorno antihorario lleva su relleno en `fill_left`.

**Perfil.** `t` es la posición por longitud de arco, de 0 a 1; izquierda y derecha son medios
anchos independientes, como ya pedía `requisitos-animador.md` §5. `interp` dice cómo se interpola
entre nudos: `linear`, `catmull-rom` (la de Lotte), o un valor de extensión que empieza con `x-`.
`raw` guarda, sin interpretar, los números de un origen que el conversor no descifró, para poder
devolverlos.

**Colores.** Siempre por id. El id de un color que viene de afuera se conserva: es lo que permite
recolorear por paleta sin perder el enlace con el origen.

**Extensiones.** Todo valor que empieza con `x-` (en `units.name`, `interp`, `source.kind`) es de
un conversor y no forma parte de esta especificación. Un lector que no lo conozca lo conserva.

## Tres formas de guardarlo

Dicen lo mismo y dan ida y vuelta entre sí.

### A. SVG con namespace `lotte:` — el archivo de verdad

```xml
<svg xmlns="http://www.w3.org/2000/svg" xmlns:lotte="https://lotte.dev/ns/draw" viewBox="…"
     lotte:version="1" lotte:precision="f32">
  <metadata>
    <lotte:drawing state="new" units="…" y="up">
      <lotte:source kind="…" sha256="…" path="…"/>
      <lotte:opaque what="…"/>
      <lotte:color key="…" name="…" palette="…" kind="solid" rgba="220,40,40,255"/>
      <lotte:gradient color="…" m="1.0,0.0,0.0,1.0,0.0,0.0"/>
    </lotte:drawing>
  </metadata>
  <g transform="matrix(1 0 0 -1 0 0)"> … apariencia resuelta … </g>
  <g lotte:role="data" display="none" transform="matrix(1 0 0 -1 0 0)">
    <g lotte:art="line"><g lotte:layer="0">
      <path id="…" d="M 0 0 C 1 2 3 4 5 6" fill="none" stroke="none"
            lotte:line="…" lotte:width="1.5" lotte:knots="0,1.5,2.5;1,0.5,0.25" lotte:interp="catmull-rom"
            lotte:fill-left="…" lotte:fill-right="…" lotte:invisible="true"/>
      <path id="…" lotte:geometry-of="…" fill="none" stroke="none" lotte:fill-right="…"/>
    </g></g>
  </g>
</svg>
```

- **La apariencia resuelta es derivada.** Va primero y con SVG corriente, para que cualquier visor
  pinte el dibujo; se regenera desde los datos del trazo y no se edita. Corrige a D-Draw en un
  punto: los datos no pueden ir en el mismo `<path>` que la apariencia, porque una zona de color la
  forman varios trazos.
- **Los datos del trazo mandan.** Un `<path>` por trazo, con `fill="none" stroke="none"` para que
  no se pinte. `d` es la línea central: `M` y, por cada ancla, `L`, `Q` o `C` según lleve 0, 1 o 2
  tiradores antes; solo comandos absolutos. Si los puntos no caben en ese patrón, van en
  `lotte:points="x,y,1;x,y,0;…"`.
- **Números.** `lotte:precision="f32"` cuando todos caben exactos en un real de 32 bits: se
  escriben con las cifras justas y se devuelven exactos. Si no, `f64` y van completos.
- `lotte:pivot-x` y `lotte:pivot-y` del hito 1 siguen valiendo, en la raíz.

### B. JSON

Los mismos campos de «Las piezas», como objeto. Para pruebas, casos sintéticos y conversores.

### C. Tabla de tensores — sin diseñar

Una tabla con los puntos y trazos de muchos dibujos, con el patrón de D-Tab (pocos tensores
grandes con índices de inicio), para análisis por lotes. Es derivada: se regenera desde A. Se
diseña cuando haya una pregunta concreta que contestar con ella.

## Origen y estado

| `state` | qué significa | al exportar hacia el origen |
|---|---|---|
| `derived` | el dibujo es igual a su archivo de origen | vuelve el archivo original, sin reescribir; se comprueba con `source.sha256` |
| `adopted` | se editó aquí | lo escribe la herramienta dueña de ese formato |
| `new` | se creó aquí | igual |

Lotte no lee ni escribe formatos de herramientas comerciales: `source` y `opaque` son datos que
deja un conversor externo, y Lotte los conserva.

## Cómo se comprueba una implementación

1. JSON → SVG → JSON: igual.
2. La apariencia resuelta se regenera desde los datos del trazo y da el mismo SVG.
3. Un dibujo bien formado: cuatro artes en orden, ids únicos, `geometry_of` hacia un trazo anterior
   con puntos del mismo arte, `closed` coherente con los puntos.

## Puntos abiertos

| punto | propuesta |
|---|---|
| Edición de la línea central con un editor de SVG que reescribe `d` en comandos relativos o borra los `id` | un normalizador de entrada; hoy el lector lo rechaza |
| Qué manda si alguien cambia la apariencia resuelta a mano | los datos del trazo; guardar una huella de la apariencia para poder avisar |
| Matriz del degradado por trazo y no por color | campo aditivo cuando un conversor la entregue |
| Trazos con textura | fuera de alcance (Lotte no hace raster); un conversor los declara en `opaque` |
| Tabla de tensores de dibujos | diseñarla con un caso de uso |
