---
name: 3d-scroll-web
description: Diseñar y programar páginas web con escenas 3D (Three.js / WebGL) y animaciones atadas al scroll (GSAP ScrollTrigger, Lenis, CSS scroll-driven animations) para 973 web studio. Usar cuando se pida un hero 3D, efectos de scroll, parallax, secciones fijas (pinned), texto que se revela, transiciones entre secciones, o "que la página se sienta como una experiencia". También al buscar referencias visuales en los MCP 21st y One_page.
---

# 3D y animaciones de scroll — 973 web studio

Objetivo: páginas que se sienten como una experiencia ("No es una página. Es una experiencia."),
pero que siguen siendo rápidas, legibles y accesibles en un celular de gama media.

## 1. Antes de programar: referencias

1. **One_page MCP** (`mcp__One_page__search_inspiration`): una sola llamada con `limit:N`.
   - 3D: `style:"3d"`, `tech:"three-js"` (o `tech:"spline"`).
   - Scroll: `style:"scroll-effects"`, `tech:"gsap"`.
   - Secciones: `mcp__One_page__search_sections` con `section_type` (`how-it-works`, `metrics`, `cta`, `scroll-hint`…).
2. **21st MCP** (`mcp__21st__search`, `type:"component"`): buscar patrones ("3d scroll hero",
   "sticky scroll reveal", "hero scrub", "parallax"). Los componentes son React/Tailwind/Motion:
   **usarlos como referencia de coreografía**, no copiarlos tal cual en un sitio estático.
   `get_component` consume cuota diaria: pedir código solo del componente elegido.
3. Elegir **un solo momento memorable** por página (el "wow") y que el resto acompañe.

## 2. Elegir la herramienta más liviana que alcance

| Necesidad | Herramienta |
|---|---|
| Fade/slide al entrar en pantalla, barra de progreso, parallax simple | CSS `animation-timeline: view()` / `scroll()` con fallback `IntersectionObserver` |
| Sección fija (pin) con pasos, scrub, timelines encadenados, texto palabra por palabra | GSAP + ScrollTrigger |
| Scroll suave e inercial (solo si hay scrub/pin pesado) | Lenis conectado al ticker de GSAP |
| Objeto/escena 3D real, partículas, shaders, luz | Three.js (módulos ES vía import map) |
| Pseudo-3D (tarjetas que rotan, perspectiva, tilt) | CSS `transform: perspective() rotateX()` — no hace falta WebGL |
| Modelo 3D diseñado por alguien más | `.glb` comprimido (Draco/Meshopt) + `GLTFLoader` |

Regla: si CSS lo resuelve, no cargar JS. Si CSS 3D lo resuelve, no cargar Three.js.

## 3. Librerías (sitio estático, sin bundler)

El sitio es un `index.html` único con CSS y JS inline. Cargar por CDN con versión exacta y `defer`:

```html
<script defer src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js"></script>
<script defer src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/ScrollTrigger.min.js"></script>
<script defer src="https://cdn.jsdelivr.net/npm/lenis@1.1.13/dist/lenis.min.js"></script>
<script type="importmap">{"imports":{
  "three":"https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js",
  "three/addons/":"https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/"}}</script>
```

Cargar solo lo que la página usa. Three.js: importarlo de forma diferida (`import()` cuando la
sección 3D se acerca al viewport) para no frenar el primer render.

Patrones de código listos: ver [references/patterns.md](references/patterns.md).

## 4. Reglas de rendimiento (no negociables)

- Animar solo `transform`, `opacity`, `filter` (con cuidado) y uniforms de shaders. Nunca `top/left/width/height/margin`.
- Nada de trabajo pesado en el evento `scroll`: usar ScrollTrigger, `animation-timeline` o un único `requestAnimationFrame`.
- WebGL: `renderer.setPixelRatio(Math.min(devicePixelRatio, 2))` (1.5 en móvil), `antialias` solo en desktop,
  pausar el loop cuando el canvas no está visible (`IntersectionObserver`) o la pestaña está oculta.
- Un solo `<canvas>` WebGL por página. Geometrías y texturas chicas; texturas en `.webp`/`.ktx2`; modelos < 1 MB.
- `will-change` solo durante la animación, no en todo el sitio.
- Imágenes con `width`/`height` para no tener saltos de layout (CLS).
- CSS 3D (`transform-style: preserve-3d`): `opacity < 1`, `filter`, `overflow` distinto de visible, `clip-path`
  o una animación de `opacity` activa (aunque esté en su estado final con `fill: both`) **aplanan** el 3D.
  Quitar la animación de entrada al terminar (`animationend` → `animation: none`) y animar la entrada con
  `translate` (propiedad individual) para no pisar el `transform` que pone el JS. Ejemplo real: el 973 del hero en `index.html`.
- Probar en Chrome con CPU throttling 4x y en un viewport de 390 px: debe ir a ~60 fps sin cortes.

## 5. Accesibilidad y fallback

- Respetar `prefers-reduced-motion: reduce`: sin scrub, sin pin largo, sin Lenis, escena 3D estática
  (render de un solo frame) o una imagen. El contenido debe verse completo sin animaciones.
- El texto nunca vive solo dentro del canvas: siempre en HTML real (SEO + lectores de pantalla). El canvas lleva `aria-hidden="true"`.
- Si WebGL no está disponible o falla, mostrar un fondo/imagen de respaldo (`try/catch` al crear el renderer).
- No secuestrar el scroll: el usuario debe poder bajar a velocidad normal; pins de como máximo ~2–3 alturas de pantalla.
- Contraste suficiente sobre fondos animados; foco visible en todos los controles.

## 6. Coreografía y estilo de la marca

- Paleta y tipografías de `:root` en `index.html` (`--ink`, `--bone`, `--blue`, `--ember`, `--wine`…;
  Bricolage Grotesque para display, Instrument Sans para texto). Las escenas 3D usan esos mismos colores.
- Easing por defecto: `cubic-bezier(.2,.8,.2,1)` en CSS, `"power3.out"` / `"expo.out"` en GSAP. Scrub: `scrub: 0.6–1`.
- Cada sección = un color y un movimiento distinto, pero con el mismo ritmo (duraciones 0.6–1.2 s, stagger 0.04–0.08 s).
- Menos es más: un efecto fuerte por sección. Si dos cosas se mueven, una lidera y la otra acompaña.
- Mobile-first: en pantallas chicas reducir distancias de parallax y cantidad de partículas, y simplificar pins (`ScrollTrigger.matchMedia` / `gsap.matchMedia`).

## 7. Checklist antes de entregar

- [ ] Funciona y se lee bien con JS desactivado y con reduced-motion.
- [ ] Sin scroll horizontal a 390 px; sin saltos de layout.
- [ ] Lighthouse móvil: Performance ≥ 85, CLS < 0.1.
- [ ] El loop de WebGL se pausa fuera de pantalla.
- [ ] Versiones de CDN fijas; nada importado que no se use.
- [ ] Captura de pantalla con Playwright (desktop y móvil) en varios puntos del scroll para verificar.
