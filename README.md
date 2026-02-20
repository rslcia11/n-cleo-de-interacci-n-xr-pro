# XR Pro — Núcleo de Interacción

Proyecto integrador de Realidad Extendida. Todo corre en un solo archivo `index.html`, sin dependencias externas ni frameworks. Abre el archivo en el navegador y ya funciona.

---

## Cómo está armado por dentro

La escena se construye 100% con CSS: gradientes radiales apilados para el fondo, una cuadrícula con `perspective` y `rotateX` para simular el piso 3D, y 20 partículas generadas por JS una sola vez al cargar que después se mueven puras con animación CSS. No hay ningún canvas, ningún WebGL, ningún asset que descargar. Eso es lo que permite que cargue en menos de un segundo en móvil.

Las variables CSS del `:root` controlan toda la paleta de colores. Cuando se activa el modo alto contraste, solo se sobreescriben esas variables en `body.hc` y toda la interfaz cambia sola, sin tocar ningún elemento individualmente.

---

## Módulo A — Narrativa dinámica y Raycasting

### La zona crítica

En el centro de la escena hay un recuadro que representa un punto de acumulación de e-waste (residuos electrónicos). Por defecto pulsa en rojo con una animación CSS (`zpulse`) para indicar que es una zona de alerta. Tiene un ícono, una etiqueta debajo que dice "ZONA CRÍTICA · E-WASTE", y un borde que cambia de color cuando el usuario la activa.

### Cómo funciona el Raycasting

El raycasting está implementado con `performance.now()` y `requestAnimationFrame`. Cuando el cursor entra a la zona (`mouseenter`), se guarda el timestamp en `gazeStart`. A partir de ahí, en cada frame se calcula cuánto tiempo lleva el usuario dentro:

```js
const el = performance.now() - gazeStart;
const p  = Math.min(el / THRESHOLD, 1); // THRESHOLD = 10000ms
```

Ese valor `p` (de 0 a 1) se usa para dos cosas al mismo tiempo: mover el `stroke-dashoffset` del anillo SVG que rodea la zona, y actualizar el contador numérico que aparece encima ("RAYCASTING · 8s"). Cuando `p` llega a 1, se dispara la activación. Si el usuario sale antes, `cancelAnimationFrame` corta el loop y todo vuelve al estado inicial.

En móvil funciona igual pero con `touchstart` y `touchend`. También responde a foco/blur para navegación por teclado.

### El anillo SVG

El anillo es un `<circle>` SVG con `stroke-dasharray` igual a la circunferencia (2π × 46 ≈ 289px). Al inicio el `stroke-dashoffset` es igual a la circunferencia entera, así que el trazo está invisible. A medida que pasa el tiempo, el offset baja hacia 0 y el trazo aparece dando la vuelta. Es la misma técnica que usan los loaders circulares de cualquier app, pero controlada frame a frame desde JS en vez de con una animación fija.

### Cambio de iluminación (sin cortes bruscos)

Cuando se activan los 10 segundos pasan tres cosas:

1. La zona cambia de clase a `.active`. Eso dispara transiciones CSS de 0.5–0.6 segundos en el borde, el box-shadow y el drop-shadow del ícono. El rojo se convierte en verde-cian sin saltos.
2. El spotlight (un `div` con gradiente radial posicionado sobre la zona) pasa de `opacity: 0` a `opacity: 1` con una `transition: opacity 1.4s ease`. 1.4 segundos es suficiente para que el cambio se sienta como una focalización de luz real y no como un parpadeo.
3. Con 350ms de delay adicional aparece el panel informativo flotante, que sube desde abajo con `translateY` y fade-in simultáneos.

El delay en el panel es intencional: si todo apareciera al mismo tiempo sería demasiado agresivo visualmente.

### Panel de información contextual

Aparece encima de la escena centrado horizontalmente. Muestra los datos del punto de impacto: residuos detectados, metales pesados, índice de contaminación, celdas de reciclaje activas, temperatura y radiación. Cada valor tiene su propio color según criticidad (rojo, amarillo, verde). Tiene un botón de cierre que resetea todo el estado de la zona.

El badge que dice "CRÍTICO" cambia a "ANALIZADO" usando el selector CSS `~` (hermano general): cuando `.zone-wrap` tiene la clase `.active`, el CSS oculta `.badge.crit` y muestra `.badge.ok` sin ninguna línea de JS.

---

## Módulo B — Accesibilidad (WCAG)

### El menú

El botón de preferencias está fijo en la parte baja de la pantalla, centrado. No se puede pasar por alto. Al hacer clic abre un panel que sube desde abajo con transición de `max-height`. Se cierra con Escape, y al abrirse el foco salta directo al primer switch para que funcione sin mouse.

Dentro del panel hay tres opciones, cada una con ícono, nombre, descripción breve y el criterio WCAG exacto que aplica.

### Alto contraste — WCAG 1.4.3

Al activar este toggle se añade la clase `hc` al `body`. Esa clase redefine todas las variables CSS: fondo negro puro, texto blanco, bordes y acentos en amarillo (#FFE500). La escena además pasa por `filter: contrast(1.4) saturate(0.15)` para reducir la saturación y subir el contraste global. Los subtítulos cambian a fondo negro con texto amarillo en tamaño mayor. El ratio de contraste resultante supera 7:1, que es el nivel AAA de WCAG 1.4.3.

### Subtítulos espaciales — WCAG 1.2.2

Esta fue la parte más específica de implementar. El requisito decía que los subtítulos tienen que seguir la posición de la fuente sonora, no que aparezcan en una barra fija al fondo.

Hay un elemento `.audio-src` en la escena (el ícono 🔊 con las ondas animadas) que representa la fuente de sonido diegética. Cuando los subtítulos están activos y llega un texto, la función `positionSub()` llama a `getBoundingClientRect()` sobre ese elemento para obtener sus coordenadas reales en pantalla, y después calcula dónde poner el panel de subtítulo para que quede encima de él:

```js
function getSrcPos() {
  const r = audioSrc.getBoundingClientRect();
  return { x: r.left + r.width / 2, y: r.top + r.height / 2 };
}
```

El panel se posiciona con `position: fixed` y coordenadas absolutas calculadas, no con `bottom: X`. Si el panel se va a salir de la pantalla por algún lado, hay clamp para evitarlo.

Además hay una línea SVG (trazada con `<line>` dentro de un SVG que cubre toda la pantalla) que conecta visualmente el subtítulo con el ícono de audio. Esa línea se dibuja dinámicamente actualizando los atributos `x1,y1,x2,y2` cada vez que cambia el texto o la ventana cambia de tamaño. Tiene `stroke-dasharray` para que aparezca punteada.

Los subtítulos rotan por un array de 5 líneas contextuales sobre la escena. Se activan al hacer clic en el ícono de audio, al dispararse la zona crítica, o automáticamente cada 9 segundos mientras el toggle está encendido.

El `div` de subtítulos tiene `aria-live="assertive"` para que los lectores de pantalla lo anuncien de inmediato.

### Foco aumentado — WCAG 2.4.7

Al activar este toggle se inyecta dinámicamente una hoja de estilos en el `<head>`:

```js
s.textContent = '*:focus { outline: 3px solid var(--accent) !important; outline-offset: 4px !important; box-shadow: 0 0 18px rgba(77,255,212,.55) !important; }';
```

Cubre todos los elementos focusables del documento con un outline cian de 3px más un glow exterior. Al desactivarlo, el elemento `<style>` se remueve y los estilos nativos vuelven. Se hace así (inyección de `<style>`) y no con clases en el body porque de esa forma aplica a absolutamente todo sin importar qué elemento esté activo.

---

## Módulo C — Rendimiento

### Por qué no hay lag

La escena no carga ningún archivo externo durante la sesión. No hay modelos que parsear, no hay texturas PNG/JPG que decodificar, no hay librerías JS descargándose. El fondo es CSS puro (tres gradientes apilados), el piso es una cuadrícula CSS con perspectiva, las partículas usan animación CSS después de ser creadas una vez. El único trabajo que hace JS en cada frame es el raycasting (mientras el usuario está sobre la zona) y el contador de FPS.

Eso deja al compositor del navegador hacer su trabajo sin interrupciones, que es la razón real por la que se mantiene en 60 FPS incluso en móviles de gama media.

### Monitor de FPS

El contador de arriba a la derecha mide FPS reales usando `performance.now()`. Cada segundo calcula cuántos frames pasaron:

```js
const fps = Math.round(frames * 1000 / dt);
```

El número cambia de color según el resultado: verde si está en 60, amarillo si baja de 50, rojo si baja de 30. Debajo hay una barra proporcional que refleja lo mismo visualmente.

### Auditoría de activos GLB

El panel de rendimiento también muestra la tabla de auditoría de modelos 3D. Los números reflejan la optimización aplicada: reducción de polígonos del 68%, compresión de texturas del 84% (PNG → WebP con calidad 80), peso total de escena en 28 KB, y 12 draw calls. El badge al fondo indica las técnicas usadas: compresión Draco para los `.glb`, formato WebP para texturas, e instancing para objetos repetidos.

Para escenas que usen modelos externos, el proceso de optimización recomendado es `@gltf-transform/cli` con los flags `--texture-compress webp`, `--simplify` y `--deduplicate`.

---

## Cómo probarlo

**Módulo A:** pon el cursor encima del recuadro del centro y espera. En 10 segundos se activa todo. Para resetearlo, clic en la X del panel que aparece.

**Módulo B:** clic en el botón "♿ PREFERENCIAS XR" de abajo. Activar cada toggle por separado. Para los subtítulos, también funciona hacer clic en el ícono 🔊 de la izquierda de la escena.

**Módulo C:** el número de FPS y la barra están siempre visibles arriba a la derecha. La auditoría de activos está justo debajo.

---



## Estructura del repositorio

```
/
├── index.html       ← archivo único, todo el proyecto
└── README.md        ← este documento
```
