# Lotte — especificación abierta de un motor de animación 2D de recorte

> Homenaje a **Lotte Reiniger** (1899–1981): sus siluetas de cartulina con bisagras son el
> ancestro directo de los pegs y las jerarquías de piezas de la animación de recorte moderna.

Este repositorio contiene **la especificación**, no el código: qué es Lotte, qué decisiones de
diseño se tomaron, por qué, con qué evidencia, y en qué orden se construye. Está escrito para que
otro equipo —o un modelo de lenguaje— pueda tomarlo y construir sobre él sin heredar nuestras
vueltas.

Tres razones ordenan todo lo demás: **Rust**, **aceleración por GPU para vector**, y **velocidad al
animar**. El [manifiesto](MANIFIESTO.md) las desarrolla en una página.

## Cómo leer

| Si querés… | Leé |
|---|---|
| Entender el proyecto en una página | [`MANIFIESTO.md`](MANIFIESTO.md) |
| Qué debe hacer el motor | [`prd.md`](prd.md) |
| Qué stack se eligió y por qué cada pieza | [`docs/stack-y-decisiones.md`](docs/stack-y-decisiones.md) |
| Las decisiones del modelo de datos, con su razón | [`docs/decisiones-modelo-de-datos.md`](docs/decisiones-modelo-de-datos.md) |
| Lo que los animadores necesitan y qué exige del modelo | [`docs/requisitos-animador.md`](docs/requisitos-animador.md) |
| Arquitectura de crates y pipeline por frame | [`docs/architecture.md`](docs/architecture.md) |
| Los problemas de la animación 2D, abstractos, sin código | [`docs/patterns.md`](docs/patterns.md) |
| Profundidad, multiplano y paralaje | [`docs/multiplane-parallax.md`](docs/multiplane-parallax.md) |
| Interfaz: visor de nodos, línea de tiempo, trazo, paletas | [`docs/ui-widgets.md`](docs/ui-widgets.md) |
| Qué combinación de bibliotecas funcionó en hardware real | [`docs/stack-verificado.md`](docs/stack-verificado.md) |
| Cómo se llegó a que la tableta funcione (evidencia cruda) | [`docs/bitacora-tableta.md`](docs/bitacora-tableta.md) |
| Hoja de ruta: hitos, lotes e issues con criterio medible | [`docs/plan-siguiente-etapa.md`](docs/plan-siguiente-etapa.md) |
| Qué se aprendió de cada proyecto permisivo leído | [`docs/referencias/`](docs/referencias/) |

## Fuentes

Todo lo que esta especificación toma de otros proyectos viene de código con **licencia permisiva**
(MIT, BSD-3, Apache-2.0), leído y citado con archivo y línea, o de **estándares abiertos** (SVG,
glTF, Lottie, CBOR, safetensors, JSON Schema, MCP), o de **matemática publicada**, o de la
**experiencia de producción** del estudio del autor. No hay código copyleft leído ni traducido, y
no se documentan formatos de herramientas comerciales.

| Proyecto | Licencia | Qué aportó |
|---|---|---|
| DragonBonesCPP | MIT | armature / bone / slot, *skinning*, timelines de slot separadas de hueso |
| OpenToonz | BSD-3-Clause | pegbar, celdas de exposición, signo de `z`, deshacer acotado por memoria |
| Graphite | Apache-2.0 / MIT | el stack `vello` + `kurbo` + `wgpu` alineado, ids numéricos, booleanas |
| Rerun | Apache-2.0 | `egui` como biblioteca sobre bucle propio, paneles acoplables, panel de tiempo |
| Godot | MIT | visor de nodos, animación por ruta, reposo + `RESET`, `UndoRedo` |
| Bevy (`bevy_animation`) | MIT / Apache-2.0 | ids por ruta de nombres, grafo de mezcla; y un contraejemplo útil |
| WebKit (`UnitBezier`) | BSD | el solver de Bézier 1D |

## Estado

La especificación está cerrada en sus decisiones; la implementación empieza por el
**hito 1 — animar un puppet**: importar desde SVG, armar con pegs, animar con claves y exposición,
guardar y reabrir idéntico, exportar la secuencia PNG. Ver la hoja de ruta.

## Licencia

Licenciado bajo cualquiera de las dos, a elección de quien la use:

- Apache License, Version 2.0 — [`LICENSE-APACHE`](LICENSE-APACHE)
- MIT license — [`LICENSE-MIT`](LICENSE-MIT)

Es la misma licencia dual del motor. Salvo que se indique lo contrario, toda contribución
enviada intencionalmente para su inclusión queda bajo esa misma licencia dual, sin términos
adicionales.

Las citas a proyectos de terceros (DragonBonesCPP — MIT; OpenToonz — BSD-3-Clause; Graphite —
Apache-2.0 / MIT; Rerun — Apache-2.0; Godot — MIT; Bevy — MIT / Apache-2.0; WebKit — BSD) son
referencias con archivo y línea a su código público, bajo sus propias licencias; esta
especificación no reproduce su código.
