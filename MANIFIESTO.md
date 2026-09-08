# Manifiesto

## Qué es

Lotte es un motor y un entorno de animación 2D de **recorte** (*cutout*) y **vectorial**: personajes
armados con piezas dibujadas y una jerarquía de pivotes (*pegs*), animados con curvas en el tiempo,
compuestos en planos con fondo y cámara, y renderizados por GPU. Es lo que Lotte Reiniger hacía con
cartulina, bisagras y una cámara sobre una mesa de vidrio, llevado a la computación gráfica actual.

## Tres razones

1. **Rust.** Sin recolector de basura ni pausas; concurrencia sin carreras de datos — el guardado
   en segundo plano y el hilo de dibujo comparten estado sin bloqueos; un solo lenguaje del motor a
   la interfaz; un ecosistema gráfico maduro (`wgpu`, `vello`, `kurbo`, `egui`) con **una sola
   copia** de cada tipo geométrico en todo el árbol de dependencias.
2. **Aceleración por GPU para animación vectorial.** El dibujo es vector (Bézier), no píxeles: se
   rasteriza en la GPU en cada frame, a cualquier zoom, sin caché de bitmaps que invalidar cuando
   un peg se mueve.
3. **Velocidad al animar.** El caso de uso que manda: un animador moviendo cientos de pegs de varios
   personajes sobre un fondo complejo, con autoguardado, deshacer profundo, y sin que el motor se
   ponga pesado por guardar. Un rig de producción tiene ~300 pegs, ~1 000 nodos y ~3 500 canales
   animables; un plano dura 20–30 segundos.

## Cómo se decide

- **Nada entra sin medirse.** Toda afirmación con un número lleva el comando que lo produjo y su
  salida. Una versión de biblioteca se elige mirando el `Cargo.lock` resultante, no el README.
- **Se lee solo código con licencia permisiva**, y se cita con archivo y línea. Lo demás se toma de
  estándares abiertos, de matemática publicada, o de la experiencia de producción. Ningún código
  copyleft se lee, se copia ni se traduce; ningún formato de herramienta comercial se documenta.
- **No se escribe código de producto hasta cerrar las decisiones que lo cambiarían.** Rehacer un
  modelo de datos con archivos guardados encima es lo más caro que hay. Por eso esta especificación
  existe antes que el motor.
- **La interfaz no tiene privilegios.** Todo lo que un menú hace es una operación invocable sin
  ventana: desde la biblioteca Rust, desde una CLI con JSON, desde un servidor MCP. El scripting vive
  fuera del proceso; las extensiones son Rust contra la API pública.

## Qué no es

No es un editor de video ni de audio, no es una herramienta de dibujo raster, no reproduce un
producto existente ni lee sus formatos. Da soporte a fondos raster, audio básico y texto vectorial
porque un plano real los necesita — como nodos y pistas aditivos, nunca como foco.

## Para quién

Para animadores de recorte que necesitan una herramienta rápida, abierta y automatizable; para
estudios que quieren integrarla a su pipeline por archivos y línea de comandos; y para
desarrolladores —humanos o no— que quieran construir sobre una especificación medida en vez de
sobre una intuición.
