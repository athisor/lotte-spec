# Auditoría OpenToonz — Stage (Z, handles, xsheet, ejes, pintado de celdas)

- **Repo:** `references/opentoonz/` (solo lectura, no existe en worktrees)
- **Commit:** `d35882e299b6` · **Licencia:** BSD-3-Clause · **Fecha:** 2026-09-08
- **Estrategia de ingesta:** se porta el algoritmo con atribución BSD-3 (el código portado cita
  archivo:línea de origen + licencia).
- **Alcance:** todo grep acotado a `toonz/sources/` (432 MB de clon completo). Sin `find /`,
  `find C:\`, `find O:\`, sin `cargo`/`cmake`.

---

## 1. Profundidad Z y proyección de cámara — `toonzlib/tstageobject.cpp`

**`getZ`** — prevista 1423-1429, real **1423-1429** (exacta):
```cpp
double TStageObject::getZ(double t) {
  double tt = paramsTime(t);
  if (m_parent) return m_parent->getZ(t) + m_z->getValue(tt);
  else return m_z->getValue(tt);
}
```
Suma recursiva por la cadena de padres. `getGlobalNoScaleZ` (1443-1445) usa el mismo patrón
sobre `m_noScaleZ`, un canal distinto (§2).

**`perspective`** — prevista 1945-1968, real **1945-1968** (exacta):
```cpp
// objectZ: default 0. valori grandi = oggetti piu' vicini alla camera.
// la distanza iniziale fra tavolo e camera e' 1000.
// cameraZ: default 0, negativo se avvicina al tavolo (lo alcanza en -1000).
bool TStageObject::perspective(TAffine &aff, const TAffine &cameraAff, double cameraZ,
                               const TAffine &objectAff, double objectZ, double objectNoScaleZ) {
  const double focus = 1000, nearPlane = 1;
  double dz = focus + cameraZ - objectZ;
  if (dz < nearPlane) { aff = TAffine(); return false; }
  double noScaleSc = 1 - objectNoScaleZ / focus;
  aff = cameraAff * TScale((focus + cameraZ) / dz) * cameraAff.inv() * objectAff * TScale(noScaleSc);
  return true;
}
```

**Signo (comentario de origen, líneas 1934-1943):** `objectZ` grande = objeto **más cerca** de
la cámara. `cameraZ` negativo = cámara acercándose a la mesa.

**Constante focal:** `focus = 1000` (línea 1951), literal, sin campo de cámara configurable. Se
repite en `toonzlib/txsheet.cpp:621`: `TAffine aff = cameraAff * TScale((1000 + cameraZ) / 1000);`
— confirma que `1000` es la unidad de referencia del motor, no un número aislado.

**Caller** (`toonzlib/scenefx.cpp:206-220`, `AffineFx::getPlacement`): `objZ = m_stageObject->
getZ(frame)` y `objNoScaleZ = m_stageObject->getGlobalNoScaleZ()` — ambos YA acumulados por la
cadena de padres antes de entrar a `perspective`.

---

## 2. "Maintain size" / noScaleZ

Es un **canal aparte**, no una escala derivada de Z, con su propia acumulación jerárquica aditiva
(`getGlobalNoScaleZ`, tstageobject.cpp:1443-1445, mismo patrón que `getZ`).

Se aplica en `perspective` como `TScale(noScaleSc)` con `noScaleSc = 1 - objectNoScaleZ/focus`
(líneas 1963-1965), multiplicado **al final** de la cadena (después de la perspectiva de cámara y
del `objectAff`): es un factor compensatorio, no una re-derivación de Z.

El usuario lo edita a mano en un campo aparte: `NoScaleField`
(`tnztools/tooloptionscontrols.cpp:1128-1163`), instanciado en `tnztools/tooloptions.cpp:433`
(`m_noScaleZField = new NoScaleField(m_tool, "field");`) junto al campo `Z:`. `onChange`
(líneas 1140-1153) llama `obj->setNoScaleZ(v)` directo — ningún cálculo lo deriva de `getZ()`.

---

## 3. Handles / anclajes al padre

**Archivo real:** `toonzlib/xshhandlemanager.cpp` — el brief decía `toonz/xshhandlemanager.cpp`,
que **no existe**; vive en `toonzlib/`, no en `toonz/` (confirmado con `find` acotado a
`toonz/sources/`).

**`XshHandleManager::getHandlePos`** — prevista 147-150, real **147-150** (exacta; función
completa 80-153):
```cpp
static const double unit = 8.0;                       // línea 83
...
else if ('A' <= handle[0] && handle[0] <= 'Z')         // 147
  return TPointD(unit * (handle[0] - 'B'), 0);         // 148
else if ('a' <= handle[0] && handle[0] <= 'z')         // 149
  return TPointD(0.5 * unit * (handle[0] - 'b'), 0);   // 150
```
- `A`-`Z`: `x = 8.0*(letra-'B')`, `y = 0`. `'B'` → offset 0 (handle neutro); `'A'`→ -8, `'C'`→ +8.
  **Todos los handles fijos están sobre el eje X únicamente.**
- `a`-`z`: `x = 4.0*(letra-'b')` (mitad de `unit`), `y = 0`.
- Antes (líneas 90-146): `"H"+dígitos` = hook port — consulta `PlasticSkeletonDeformation` o el
  `HookSet` del nivel (`Hook::getAPos`), acumulando un delta contra frames anteriores del mismo
  nivel. Mucho más complejo que A-Z/a-z: liga el handle a geometría de malla/skinning.

**`getHandle`/`getParentHandle`** (`tstageobject.cpp`): `m_handle`/`m_parentHandle`, default
`"B"` (offset 0) para ambos (líneas 378-379). `setHandle`/`setParentHandle`: 775-785. Wrapper
`getHandlePos`: 791-798, delega a `m_tree->getHandlePos` (el manager de arriba).

**Uso en `computeLocalPlacement`** (líneas 1380-1390):
```cpp
TPointD handlePos = getHandlePos(m_handle, (int)frame);        // 1382 — pivote propio
TPointD center    = (m_center + handlePos) * Stage::inch;      // 1383
TPointD pos = m_offset;
if (m_parent) pos += m_parent->getHandlePos(m_parentHandle, (int)frame);  // 1386 — ancla al padre
```
El **handle propio** desplaza el pivote de rotación/escala (se suma a `m_center`); el **handle
del padre** desplaza la posición — el hijo se ancla a un punto del padre distinto de su origen.

---

## 4. Operaciones de hoja de exposición — `toonzlib/txsheet.cpp`

Prevista como bloque único 627-716: **incorrecta** — `reverseCells` empieza en 627, pero
`stepCells`/`eachCells` están más abajo (median `swingCells` e `incrementCells`).

| Función | Prevista | Real |
|---|---|---|
| `reverseCells` | 627-716 | **627-640** |
| `stepCells` | 627-716 | **733-762** |
| `eachCells` | 627-716 | **823-849** |

**`reverseCells`** (627-640): por columna, `i1` avanza desde `r0`, `i2` retrocede desde `r1`,
intercambio de dos punteros clásico. Celdas vacías: **sin caso especial**, se intercambian igual
que cualquier otra celda.

**`stepCells(r0,c0,r1,c1,type)`** (733-762): "Step N" — `type`=2 o 3 (comentario línea 759:
`// depends on step type (2 or 3 for now)`). Cada celda original se repite `type` veces
consecutivas: inserta `nr*(type-1)` filas al final, luego reescribe repitiendo cada celda leída.
Celdas vacías: si la celda leída está vacía, llama `clearCells` en cada fila destino (no
`setCell` con celda vacía).

**`eachCells(r0,c0,r1,c1,type)`** (823-849): inverso de Step — "Each N" submuestrea 1 de cada
`type` filas (`j += type` al leer), reduce a `newRows = ceil(nr/type)`, borra el sobrante
(`removeCells`) y reescribe compactado. Celdas vacías: mismo patrón, `clearCells` en vez de
`setCell`.

Nota compartida: ambas evitan pasar una celda vacía a `setCell`, prefiriendo `clearCells` —
sugiere que no son intercambiables (posible diferencia de notificación/side-effect). No confirmé
la diferencia interna; a verificar por el developer si se porta la distinción.

---

## 5. Convención de ejes (Y-arriba en el mundo)

**`winToWorld`** (`toonz/sceneviewer.cpp:1039`, confirmada exacta):
```cpp
TPointD pp(pos.x() - (double)width() / 2.0, -pos.y() + (double)height() / 2.0);  // 1042-1043, niega Y
```
El overload `TPointD` (1079-1080) delega en este vía `height() - winPos.y` — misma familia, no
es una confirmación independiente.

**Búsqueda de otro lugar que asuma Y-arriba:** grep de `TScale(1,-1)`/`TScale(-1,1)` en
`toonzlib/tstageobject.cpp`, `toonzlib/scenefx.cpp` y `toonzlib/tcamera.cpp` → **cero
resultados** en las tres. Los únicos `TScale(1,-1)`/`TScale(-1,1)` de `sceneviewer.cpp` (líneas
2603, 2606, 2798-2799, 2819-2820, 2960, 3044, 3103) están atados a `m_isFlippedX/Y` — el flip
espejo **opcional** de UI (`flipY()`, línea 2810), no la convención de base.
`getNormalZoomScale()` (3506-3508) es solo `TScale(getDpiFactor()).inv()`, sin flip de signo.

**Conclusión medida:** el flip Y-mundo↔Y-pantalla es responsabilidad exclusiva del límite del
viewer (`winToWorld`/`worldToPos`), implementada una sola vez. El núcleo de transformación
(`perspective`, `scenefx.cpp`, `tcamera.cpp`) nunca niega Y. No encontré una segunda ubicación que
*reafirme* Y-arriba — la convención está aislada a un solo punto de entrada/salida.

---

## 6. Pintado de xsheet — `toonz/xshcellviewer.cpp`

**`CellArea::paintEvent`** — prevista 3153, real **3153** (exacta):
```cpp
void CellArea::paintEvent(QPaintEvent *event) {
  QRect toBeUpdated = event->rect();
  QPainter p(this); p.setClipRect(toBeUpdated);
  p.fillRect(toBeUpdated, QBrush(m_viewer->getEmptyCellColor()));  // capa 1: fondo
  drawCells(p, toBeUpdated);                                       // capa 2
  if (Preferences::instance()->isShowKeyframesOnXsheetCellAreaEnabled())
    drawKeyframe(p, toBeUpdated);                                  // capa 3 (opcional)
  drawNotes(p, toBeUpdated);                                       // capa 4
  drawFocusCellBorder(p);                                          // capa 5
  if (getDragTool()) getDragTool()->drawCellsArea(p);              // capa 6
}
```
Capas en orden: (1) fondo plano "celda vacía"; (2) celdas — dentro de `drawCells`
(1265-1401): fondo no-vacío, selección, contenido, separadores, y **el cabezal (frame actual)
dibujado inline** vía `drawCurrentTimeIndicator` (definida en 1801, invocada >12 veces dentro de
`drawCells`) cuando `row == currentRow`, no como capa separada; (3) keyframes, condicional a
preferencia; (4) notas; (5) borde de celda con foco; (6) overlay de drag activo.

**Culling** (dentro de `drawCells`, 1265-1401):
```cpp
CellRange visible = m_viewer->xyRectToRange(toBeUpdated);   // 1276
int r0=visible.from().frame(), r1=visible.to().frame();
int c0=visible.from().layer(), c1=visible.to().layer();
for (col = c0; col <= c1; col++)      // 1304 — solo columnas visibles
  for (row = r0; row <= r1; row++)    // 1360 — solo filas visibles, por columna
```
Parte de `event->rect()` (el rect "sucio" de Qt, no la ventana completa). Columnas plegadas
(`!columnFan->isActive(col)`) usan `drawFoldedColumns` (1309), más barato que el pintado completo.

---

## Portable a Lotte

Tipos verificados en el repo (solo lectura): `crates/lotte-core/src/math.rs:11`
(`Transform2D{position,pivot,rotation,scale,shear}`), `crates/lotte-render/src/camera.rs:6`
(`ViewportCamera{pan,zoom,rotation}`), `crates/lotte-timeline/src/exposure.rs:29/42/52/58`
(`ExposureTrack`, `set_cell`/`get_cell`/`evaluate_held_cell`).

**Z y proyección (§1) — signo OPUESTO confirmado.** OpenToonz: Z grande = más cerca. Spec Lotte
(`docs/multiplane-parallax.md:41`, ya escrita): `z>0` = Fondo Lejano (`S_z<1`), Z grande = más
lejos. Álgebra de equivalencia (derivada, a verificar con test): con cámara en reposo, escala OT
= `focus/(focus-objectZ_ot)`; spec Lotte hasta el 2026-09-08 = `D/(D+z_lotte)` → `z_lotte = -objectZ_ot`. **Superado por D1 decidida** ([`../decisiones-modelo-de-datos.md`](../decisiones-modelo-de-datos.md)): Lotte adopta la misma convención, `S_z = D/(D-z)`, así que `z_lotte = objectZ_ot` sin inversión.
```rust
// Puerto de perspective() (tstageobject.cpp:1945-1968) con Z invertida para calzar con
// multiplane-parallax.md (z>0=lejos). focus_d ~ D de la spec (no existe hoy en ViewportCamera).
fn project_with_depth(camera_aff: Affine2, camera_z: f32, object_aff: Affine2, object_z: f32,
                      focus_d: f32) -> Option<Affine2> {
    const NEAR_PLANE: f32 = 1.0; // unidad a definir; OT usa "inch" de escena
    let dz = focus_d - camera_z + object_z;      // signos invertidos vs dz_ot=focus+cameraZ-objectZ
    if dz < NEAR_PLANE { return None; }
    let scale = (focus_d - camera_z) / dz;
    Some(camera_aff * Affine2::from_scale(Vec2::splat(scale)) * camera_aff.inverse() * object_aff)
}
// getZ (tstageobject.cpp:1423-1429): suma simple por la cadena de padres.
fn accumulated_z(node: NodeId, dag: &Dag, frame: u32) -> f32 {
    let local = dag.z_channel(node).evaluate(frame);
    dag.parent(node).map_or(local, |p| accumulated_z(p, dag, frame) + local)
}
```

**"Maintain size" (§2).** Lotte ya tiene la fórmula derivada (`multiplane-parallax.md:55`):
`Escala_reposo = (D-z)/D = 1/S_z` (signo según D1 decidida). Es mejor que el canal `noScaleZ` de OpenToonz (sincronizado a
mano, riesgo de desincronización): recomendación — no portar un campo aparte en `Transform2D`;
aplicar `scale_compensada = pose_scale * (focus_d+z)/focus_d` en la evaluación de perspectiva,
detrás de un flag `maintain_size: bool` por nodo.

**Handles (§3).**
```rust
const HANDLE_UNIT: f32 = 8.0; // xshhandlemanager.cpp:83
fn handle_offset(handle: char) -> Vec2 {
    match handle {
        'A'..='Z' => Vec2::new(HANDLE_UNIT * (handle as i32 - 'B' as i32) as f32, 0.0),
        'a'..='z' => Vec2::new(0.5*HANDLE_UNIT * (handle as i32 - 'b' as i32) as f32, 0.0),
        _ => Vec2::ZERO, // "H<n>" hook port: fuera de v1, requiere skinning que Lotte no tiene
    }
}
// Puerto de computeLocalPlacement (tstageobject.cpp:1380-1390), calza con Transform2D real:
fn local_placement(t: &Transform2D, own: char, parent: Option<char>) -> Affine2 {
    let pivot = t.pivot + handle_offset(own);                    // handle propio -> pivote
    let position = t.position + parent.map_or(Vec2::ZERO, handle_offset); // handle padre -> ancla
    Affine2::from_translation(position) * Affine2::from_angle(t.rotation)
        * shear_matrix(t.shear) * Affine2::from_scale(t.scale) * Affine2::from_translation(-pivot)
}
```
Default `'B'` (offset 0), igual que `m_handle`/`m_parentHandle` en OpenToonz (378-379).

**Xsheet (§4).** `ExposureTrack` opera sobre una sola columna; las tres operaciones se portan 1:1
por track, el caller itera columnas:
```rust
fn reverse_cells(t: &mut ExposureTrack, r0: u32, r1: u32) {
    let (mut i, mut j) = (r0, r1);
    while i < j {
        let (a, b) = (t.get_cell(i).cloned(), t.get_cell(j).cloned());
        set_or_clear(t, i, b); set_or_clear(t, j, a);
        i += 1; j -= 1;
    }
}
fn step_cells(t: &mut ExposureTrack, r0: u32, r1: u32, step: u32) {
    let mut i = r0;
    for cell in (r0..=r1).map(|r| t.get_cell(r).cloned()).collect::<Vec<_>>() {
        for k in 0..step { set_or_clear(t, i + k, cell.clone()); }
        i += step;
    }
}
fn each_cells(t: &mut ExposureTrack, r0: u32, r1: u32, each: u32) {
    for (off, cell) in (r0..=r1).step_by(each as usize)
        .map(|r| t.get_cell(r).cloned()).enumerate() {
        set_or_clear(t, r0 + off as u32, cell);   // + truncar el resto del rango
    }
}
// set_or_clear: réplica de la distinción setCell/clearCells de txsheet.cpp (§4, nota
// compartida) — verificar contra ExposureTrack::set_cell con valor vacío antes de portar.
```

**Ejes (§5).** No hay fórmula que portar, es regla de arquitectura — OpenToonz ya la sigue bien:
```rust
// Vive en el crate de viewport/UI (aún sin crear), NUNCA en lotte-core ni en lotte-render::camera.
fn screen_to_world(screen: Vec2, viewport_size: Vec2, cam: &ViewportCamera) -> Vec2 {
    let centered = Vec2::new(screen.x - viewport_size.x/2.0, -(screen.y - viewport_size.y/2.0));
    cam.inverse_matrix() * centered   // único lugar del sistema que niega Y
}
```

**Pintado de xsheet (§6).**
```rust
fn paint_cell_area(p: &mut Painter, dirty_rect: Rect, viewer: &XsheetViewerState) {
    p.fill_rect(dirty_rect, viewer.empty_cell_color());
    let visible = viewer.rect_to_cell_range(dirty_rect);  // culling: solo el rect sucio
    draw_cells(p, visible);   // fondo, selección, contenido, cabezal inline por celda
    if viewer.prefs.show_keyframes_in_cell_area { draw_keyframes(p, visible); }
    draw_notes(p, visible);
    draw_focus_cell_border(p);
    if let Some(drag) = viewer.active_drag_tool() { drag.draw_overlay(p); }
}
```
Punto a copiar: el cabezal se dibuja inline dentro del bucle de celdas, no como capa aparte.

---

## Lo que NO existe o no encontré

- **`toonz/xshhandlemanager.cpp` no existe** — el archivo real es `toonzlib/xshhandlemanager.cpp`.
- **No hay checkbox/acción "Maintain Size".** Grep de `maintain` (case-insensitive) en
  `toonz/*.cpp`, `tnztools/*.cpp`, `toonzlib/*.cpp`: el único combo "Maintain"
  (`tnztools/tooloptions.cpp:721-723`, `m_maintainCombo`) es del scale tool (aspect ratio), no de
  Z. No hay label dedicado (`noScaleZLabel`/"N/S" no aparecen) — el usuario solo teclea un valor.
- **No hay segunda ubicación fuera de `sceneviewer.cpp` que reafirme Y-arriba** — hallazgo
  documentado en §5, no vacío de cobertura.
- **No confirmé la diferencia real `setCell(vacía)` vs `clearCells()`** — solo que `stepCells`/
  `eachCells` las tratan distinto. Leer ambas funciones completas queda fuera de alcance.
- **Sin benchmarks ni cargo/cmake** — auditoría de algoritmo/formato, no de performance; ninguno
  de los dos se ejecutó (prohibido para esta tarea).

---

## Cómo se midió

Corrido desde `references/opentoonz/toonz/sources` (o subcarpetas):

```bash
wc -l toonzlib/tstageobject.cpp toonzlib/txsheet.cpp toonz/sceneviewer.cpp toonz/xshcellviewer.cpp
grep -n "getZ\|noScaleZ\|cameraZ\|::getZ\|1000\." toonzlib/tstageobject.cpp
find . -iname "*handlemanager*"     # acotado a toonz/sources/
grep -n "unit\|'A'\|'B'\|'Z'\|'a'\|'z'\|getHandle\|getParentHandle\|Hook\|hook" toonzlib/xshhandlemanager.cpp
grep -n "::reverseCells\|::stepCells\|::eachCells" toonzlib/txsheet.cpp
grep -n "winToWorld" toonz/sceneviewer.cpp
grep -n "TScale(1, *-1)\|TScale(-1, *1)\|TScale(1,-1)" toonzlib/*.cpp
grep -n "CellArea::paintEvent\|void.*paintEvent" toonz/xshcellviewer.cpp
grep -n "void CellArea::drawCells\|CellArea::drawKeyframe\|CellArea::drawNotes\|CellArea::drawFocusCellBorder" toonz/xshcellviewer.cpp
grep -rni "maintain" toonz/*.cpp tnztools/*.cpp toonzlib/*.cpp
```

Cada línea citada se leyó con `Read` sobre el archivo real inmediatamente después del grep que la
localizó, nunca copiada de memoria ni de la auditoría anterior.

Verificación adicional de tipos Lotte (solo lectura, desde el repositorio del motor, sin tocar código):
```bash
grep -rn "pub struct Transform2D\|pub struct ViewportCamera\|pub struct ExposureTrack\|fn set_cell\|fn get_cell\|fn evaluate_held_cell" crates/*/src/*.rs
```
