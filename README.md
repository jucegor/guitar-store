# Resonance — Handcrafted Guitars

Landing page de marca de guitarras con animación 3D impulsada por scroll, construida en un único archivo HTML sin dependencias externas.

**Live demo:** [github.com/jucegor/guitar-store](https://github.com/jucegor/guitar-store)

---

## Cómo funciona la animación scroll-driven

La ilusión de movimiento 3D se logra con una técnica llamada **frame-scrubbing**: en lugar de renderizar 3D en tiempo real, se exportan los frames de una animación 3D como una secuencia de imágenes JPG y se muestra el frame correcto según la posición del scroll.

### 1. Preparar los frames

Exporta tu animación desde Blender, Cinema 4D o cualquier software 3D como una secuencia numerada de imágenes:

```
Guitar/frame_0001.jpg
Guitar/frame_0002.jpg
...
Guitar/frame_0124.jpg
```

Cuantos más frames, más suave se ve la animación. 60–150 frames es un rango práctico.

### 2. Precargar todas las imágenes

Antes de que el usuario haga scroll, todas las imágenes deben estar en memoria para que el cambio sea instantáneo. Se usan objetos `Image` nativos y se rastrea el progreso de carga:

```js
const FRAME_COUNT = 124;
const frames = new Array(FRAME_COUNT);
let loadedCount = 0;

for (let i = 0; i < FRAME_COUNT; i++) {
  const img = new Image();
  img.src = `Guitar/frame_${String(i + 1).padStart(4, '0')}.jpg`;
  img.onload = () => {
    loadedCount++;
    if (loadedCount === FRAME_COUNT) onAllLoaded(); // ocultar loader
  };
  frames[i] = img;
}
```

### 3. Hacer sticky el canvas

El canvas debe quedarse fijo en la pantalla mientras el usuario hace scroll. Esto se logra con `position: sticky`:

```css
#scroll-container {
  height: 500vh; /* espacio de scroll — más alto = animación más lenta */
  position: relative;
}

#hero-canvas {
  position: sticky;
  top: 0;
  width: 100vw;
  height: 100vh;
}
```

La clave está en que el **contenedor** tiene `500vh` de altura (lo que genera el espacio de scroll), y el **canvas** es `sticky` dentro de él (se queda visible mientras se scrollea el contenedor).

### 4. Mapear scroll → frame

En el evento `scroll`, se calcula qué fracción del contenedor ya fue recorrida (de `0` a `1`) y se traduce a un índice de frame:

```js
function onScroll() {
  const maxScroll = scrollContainer.scrollHeight - window.innerHeight;
  const progress  = Math.min(Math.max(window.scrollY / maxScroll, 0), 1);
  const frameIndex = Math.round(progress * (FRAME_COUNT - 1));

  if (frameIndex !== currentFrame) {
    currentFrame = frameIndex;
    requestAnimationFrame(() => drawFrame(currentFrame));
  }
}

window.addEventListener('scroll', onScroll, { passive: true });
```

`passive: true` es importante para no bloquear el hilo principal durante el scroll.

### 5. Dibujar el frame en canvas

Se usa `drawImage` con escala `cover` para que la imagen siempre llene el viewport sin deformarse:

```js
function drawFrame(index) {
  const img = frames[index];
  if (!img?.complete) return;

  const cw = canvas.width,  ch = canvas.height;
  const iw = img.naturalWidth, ih = img.naturalHeight;

  // Escala tipo "cover": la imagen llena el canvas sin deformarse
  const scale = Math.max(cw / iw, ch / ih);
  const dw = iw * scale, dh = ih * scale;

  ctx.clearRect(0, 0, cw, ch);
  ctx.drawImage(img, (cw - dw) / 2, (ch - dh) / 2, dw, dh);
}
```

### 6. Sincronizar texto con el scroll

Se pueden mostrar y ocultar elementos de texto según el progreso del scroll. Se define una zona de entrada y salida para cada panel y se calcula la opacidad con una función de fade:

```js
const phases = [
  { el: document.getElementById('phase-2'), start: 0.22, end: 0.50 },
  { el: document.getElementById('phase-3'), start: 0.53, end: 0.80 },
];

function updatePhases(progress) {
  const fadeWidth = 0.06; // zona de transición (6% del recorrido)

  phases.forEach(({ el, start, end }) => {
    let opacity = 0;
    if (progress >= start && progress <= end) {
      const fadeIn  = Math.min((progress - start) / fadeWidth, 1);
      const fadeOut = progress > end - fadeWidth
        ? Math.max((end - progress) / fadeWidth, 0)
        : 1;
      opacity = fadeIn * fadeOut;
    }
    el.style.opacity = opacity;
  });
}
```

---

## Consideraciones de rendimiento

| Técnica | Por qué |
|---|---|
| `passive: true` en el listener de scroll | No bloquea el rendering thread |
| `requestAnimationFrame` para `drawFrame` | Limita el repintado a 60fps máximo |
| Preload completo antes de iniciar | Elimina el parpadeo al cambiar de frame |
| `Math.round` en el índice | Evita recalcular si el frame no cambió |
| Reducir `height` del contenedor en mobile (`320vh`) | La animación es más corta y se siente más natural al hacer scroll con el dedo |

## Stack

- HTML / CSS / Vanilla JS — sin frameworks ni librerías
- Canvas API nativa
- Intersection Observer API para animaciones de entrada en secciones
- Fuentes: Cormorant Garamond + Inter (Google Fonts)
