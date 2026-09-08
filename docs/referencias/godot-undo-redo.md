# Godot — `UndoRedo` y `EditorUndoRedoManager` (MIT) — 2026-09-08

Lectura directa del código, licencia MIT: **portable con atribución** (va al `NOTICE`, issue A6).
Repositorio en `references/godot` (checkout principal, solo lectura), commit `6a0f6f32cfb2`,
*sparse* ampliado hoy a `core/object` (629 KB medidos con `du -sh`); `editor/` se agregó para
leer el *manager* y se volvió a quitar (38 MB) — las citas de abajo a `editor/` se midieron antes
de quitarlo. Todas las líneas se verificaron con `grep -n` / `sed -n` en esta sesión.

Motivación: decisión **D-Undo** ([`../decisiones-modelo-de-datos.md`](../decisiones-modelo-de-datos.md))
— cómo se agrupan, fusionan y acotan los pasos de deshacer en un editor de producción.

---

## 1. Modelo de datos (`core/object/undo_redo.h`)

```
Action { name, do_ops: List<Operation>, undo_ops: List<Operation>, last_tick: u64, backward_undo_ops }
Operation { type: METHOD | PROPERTY | REFERENCE, object, name, callable, value, force_keep_in_merge_ends }
UndoRedo { actions: Vector<Action>, current_action: i32 = -1, action_level, max_steps = 0,
           merge_mode, merging, version: u64 = 1, merge_total }
```

- Un paso (`Action`) es **un par de listas**: lo que hace y lo que deshace. No es una instantánea
  del documento: es una lista de operaciones sobre objetos, cada una un método a invocar
  (`TYPE_METHOD`), una propiedad a asignar (`TYPE_PROPERTY`) o una referencia a mantener viva
  (`TYPE_REFERENCE`, para que el objeto borrado siga existiendo mientras se pueda rehacer).
- `version` es un contador monótono que sube en cada *commit* y baja en cada *undo*: es la
  **marca de "documento sucio"** para el guardado, sin recorrer nada (`get_version`,
  `undo_redo.h`).
- `action_level` permite **anidar** `create_action` / `commit_action`: solo el nivel 0 crea o
  cierra un paso real (`commit_action`, `.cpp:311-316`).

## 2. Fusión de gestos — lo que más nos importa (`.cpp:88-138`)

`create_action(name, MergeMode, backward_undo_ops)`. Tres modos (`.h`):

| Modo | Efecto al abrir una acción "igual" a la anterior |
|---|---|
| `MERGE_DISABLE` | siempre un paso nuevo |
| `MERGE_ENDS` | **borra los `do_ops` del paso anterior** (salvo los marcados `force_keep_in_merge_ends`) y conserva sus `undo_ops`: el paso resultante deshace al **estado original** y rehace al **estado final**. Es el modo para arrastrar una manija: cien movimientos, un paso |
| `MERGE_ALL` | acumula todos los `do_ops` (`merge_total` recuerda cuántos había) |

La condición de "igual" está en `.cpp:95`, y es tres cosas **a la vez**:

```
p_mode != MERGE_DISABLE
&& actions.back().name == p_name                       // mismo nombre de acción
&& actions.back().backward_undo_ops == p_backward_undo_ops
&& actions.back().last_tick + 800 > ticks               // dentro de una ventana de 800 ms
```

Es decir: la fusión **no** la decide el gesto (no hay "empezó a arrastrar / soltó") sino una
**ventana temporal de 800 ms** entre acciones del mismo nombre. Cada acción fusionada renueva
`last_tick` (`.cpp:120`), así que un arrastre continuo de diez segundos sigue siendo un paso; una
pausa de más de 800 ms lo corta. Al fusionar, `version` se decrementa antes de volver a subir
(`.cpp:320-323`) para que el documento no cuente dos cambios.

`backward_undo_ops` (`.cpp:123-126`, `:325-327`): las `undo_ops` se pueden **ejecutar en orden
inverso** al que se agregaron; hace falta cuando el deshacer tiene que desandar dependencias
(borrar hijos antes que el padre).

## 3. Tope de historia (`.cpp:333-338`, `:498-503`, `:555`)

```
max_steps = 0    // por defecto: sin tope
commit_action(): if (max_steps > 0) while (actions.size() > max_steps) _pop_history_tail();
```

Solo **por cantidad**, y por defecto **ilimitado**; la propiedad se expone con rango `"0,50,1,or_greater"`.
No hay tope por memoria: contrasta con OpenToonz (`tundo.cpp:209-210`: descarta mientras
`count > 100` **o** la suma de `getSize()` supera `undoMemorySize`, 100 MB por defecto en
`preferences.cpp:384`).

## 4. Rehacer descartado y *callbacks* (`.cpp:88-92`, `.h`)

`create_action` en nivel 0 llama a `discard_redo()`: cualquier acción nueva **trunca** la rama de
rehacer — el comportamiento estándar de la industria. Tres *callbacks* opcionales:
`commit_notify` (nombre del paso, para la barra de estado / historial), `method_notify` y
`property_notify` (para grabar macros o registrar cambios). Ninguno es obligatorio.

## 5. El *manager* del editor: **una historia por escena** (`editor/editor_undo_redo_manager.h:45-68`)

```
GLOBAL_HISTORY = 0, INVALID_HISTORY = -99
struct History { id, ... }        HashMap<int, History> history_map
get_history_id_for_object(Object*)  → decide a qué historia va una acción según el objeto tocado
```

Godot no tiene una sola pila: tiene **una pila por escena abierta** más una global (proyecto,
preferencias), y la acción cae en la pila del **objeto que modifica** (`.cpp:63-101`). Deshacer en
la pestaña A no deshace lo que hiciste en la pestaña B. `clear_history(idx)` y
`discard_history(idx)` (`.h:134,146`) vacían o eliminan la pila de una escena al cerrarla.

---

## 6. Qué tomamos para Lotte y qué no

| Godot | Lotte (propuesta D-Undo) |
|---|---|
| Paso = par `do_ops` / `undo_ops` de operaciones sobre objetos | Paso = lista de `Edición` con inverso, sobre el modelo de documento (no sobre widgets). Misma forma; en Rust, un `enum` de ediciones serializable, que además es lo que va al **diario** (D-Save) |
| Fusión por **nombre + ventana de 800 ms** | Tomar los dos criterios y agregar el explícito: el gesto de arrastre abre/cierra una transacción (como `StartTransaction`/`CommitTransaction` de Graphite, `document_history.rs`); la ventana temporal queda como red para acciones repetidas por teclado (flechas, *nudge*) |
| `MERGE_ENDS` conserva el primer `undo` y el último `do` | Idéntico: al fusionar, la `Edición` guarda `antes` del primer paso y `después` del último; lo intermedio se descarta |
| `max_steps` solo por cantidad, ilimitado por defecto | **Por memoria** (OpenToonz), no por cantidad: cada `Edición` declara `bytes()`; tope configurable |
| `version` como marca de sucio | Igual: el guardado en segundo plano compara `version` guardada vs actual, y por archivo sucio (D-Fmt) |
| Una historia **por escena** | Una historia **por plano abierto**; la biblioteca (definiciones) con la suya, porque editar un rig afecta a todos los planos que lo instancian y no debe deshacerse desde uno de ellos |
| `TYPE_REFERENCE` para mantener vivos objetos borrados | En Rust no hace falta: la `Edición` de borrado **posee** lo borrado (`Box`/`Arc`) hasta que la pila la descarte |
| `backward_undo_ops` | Innecesario si la `Edición` compuesta se deshace siempre en orden inverso al de aplicación — hacerlo la regla, no una bandera |

Lo que Godot **no** resuelve y nosotros sí necesitamos: la relación entre la pila de deshacer y el
autoguardado. Godot guarda la escena completa (`.tscn`) cuando el usuario guarda; su pila es solo
de memoria. La idea de que **las mismas ediciones sean el diario en disco** viene del *write-ahead
log* de SQLite y de la separación entre pasos con estado y pasos diferenciales (ver D-Save en
[`../decisiones-modelo-de-datos.md`](../decisiones-modelo-de-datos.md)), no de Godot.

## 7. Método

`git sparse-checkout add core/object editor` sobre `references/godot` (`6a0f6f3`); lectura de
`undo_redo.h` completo (151 líneas) y de `undo_redo.cpp` en los rangos citados; `grep -n` en
`editor/editor_undo_redo_manager.{h,cpp}`; después `git sparse-checkout set` sin `editor/`.
No se midió el editor de Godot en ejecución: las afirmaciones son sobre el código, no sobre
comportamiento observado.
