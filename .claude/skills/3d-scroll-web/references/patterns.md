# Patrones listos para copiar

Todos pensados para HTML estático, sin bundler. Adaptar selectores y colores a la página.

## 1. Reveal al entrar en pantalla (solo CSS, con fallback)

```css
.reveal{opacity:1;transform:none}
@supports (animation-timeline: view()){
  @media (prefers-reduced-motion: no-preference){
    .reveal{animation:reveal linear both;animation-timeline:view();animation-range:entry 0% cover 30%}
    @keyframes reveal{from{opacity:0;transform:translateY(40px) scale(.98)}}
  }
}
```

Fallback para navegadores sin `animation-timeline` (Safari viejo / Firefox):

```js
if (!CSS.supports('animation-timeline: view()') && !matchMedia('(prefers-reduced-motion: reduce)').matches) {
  const io = new IntersectionObserver(es => es.forEach(e => {
    if (e.isIntersecting) { e.target.classList.add('is-in'); io.unobserve(e.target); }
  }), { rootMargin: '0px 0px -10% 0px' });
  document.querySelectorAll('.reveal').forEach(el => { el.classList.add('js-reveal'); io.observe(el); });
}
```
```css
.js-reveal{opacity:0;transform:translateY(40px);transition:opacity .8s, transform .8s cubic-bezier(.2,.8,.2,1)}
.js-reveal.is-in{opacity:1;transform:none}
```

## 2. Barra de progreso de lectura (solo CSS)

```css
.progress{position:fixed;inset:0 0 auto;height:3px;background:var(--ember);transform-origin:0 50%;transform:scaleX(0);z-index:70}
@supports (animation-timeline: scroll()){
  .progress{animation:grow linear both;animation-timeline:scroll(root)}
  @keyframes grow{to{transform:scaleX(1)}}
}
```

## 3. GSAP + Lenis: base común

```js
const reduce = matchMedia('(prefers-reduced-motion: reduce)').matches;
gsap.registerPlugin(ScrollTrigger);

if (!reduce) {
  const lenis = new Lenis({ lerp: 0.1 });
  lenis.on('scroll', ScrollTrigger.update);
  gsap.ticker.add(t => lenis.raf(t * 1000));
  gsap.ticker.lagSmoothing(0);
}
```

## 4. Sección fija con pasos (pin + scrub)

```html
<section class="steps">
  <div class="steps__stage">
    <div class="step">Escuchamos</div>
    <div class="step">Diseñamos</div>
    <div class="step">Publicamos</div>
  </div>
</section>
```
```js
const mm = gsap.matchMedia();
mm.add('(prefers-reduced-motion: no-preference)', () => {
  const steps = gsap.utils.toArray('.step');
  const tl = gsap.timeline({
    scrollTrigger: { trigger: '.steps', start: 'top top', end: '+=' + steps.length * 80 + '%', pin: '.steps__stage', scrub: 0.8 }
  });
  steps.forEach((s, i) => {
    tl.from(s, { yPercent: 60, opacity: 0, rotateX: -40, ease: 'power3.out' });
    if (i < steps.length - 1) tl.to(s, { yPercent: -60, opacity: 0, ease: 'power2.in' }, '+=0.3');
  });
});
```

## 5. Tarjeta que rota en 3D con el scroll (pseudo-3D, sin WebGL)

Idea tomada del "Container Scroll Animation" de 21st, en vanilla:

```css
.tilt-wrap{perspective:1200px}
.tilt{transform-origin:50% 0;transform:rotateX(25deg) scale(1.05);border-radius:24px;box-shadow:0 40px 80px rgba(14,15,20,.35)}
```
```js
gsap.to('.tilt', { rotateX: 0, scale: 1, ease: 'none',
  scrollTrigger: { trigger: '.tilt-wrap', start: 'top 80%', end: 'top 20%', scrub: true } });
```

## 6. Texto que se revela palabra por palabra

```js
document.querySelectorAll('[data-split]').forEach(el => {
  el.setAttribute('aria-label', el.textContent);
  el.innerHTML = el.textContent.trim().split(/\s+/)
    .map(w => `<span aria-hidden="true" style="display:inline-block">${w}</span>`).join(' ');
  gsap.from(el.children, { opacity: 0.15, stagger: 0.05, ease: 'none',
    scrollTrigger: { trigger: el, start: 'top 75%', end: 'bottom 40%', scrub: true } });
});
```

## 7. Escena Three.js controlada por el scroll (carga diferida)

```html
<div class="scene" id="scene"><canvas aria-hidden="true"></canvas></div>
```
```js
<script type="module">
const host = document.getElementById('scene');
const reduce = matchMedia('(prefers-reduced-motion: reduce)').matches;

new IntersectionObserver(async ([e], io) => {
  if (!e.isIntersecting) return; io.disconnect();
  const THREE = await import('three');
  let renderer;
  try { renderer = new THREE.WebGLRenderer({ canvas: host.querySelector('canvas'), alpha: true, antialias: innerWidth > 768 }); }
  catch { host.classList.add('no-webgl'); return; }   // CSS muestra una imagen de respaldo
  renderer.setPixelRatio(Math.min(devicePixelRatio, innerWidth > 768 ? 2 : 1.5));

  const scene = new THREE.Scene();
  const camera = new THREE.PerspectiveCamera(35, 1, 0.1, 100); camera.position.z = 6;
  const mesh = new THREE.Mesh(new THREE.TorusKnotGeometry(1, .32, 160, 24),
    new THREE.MeshStandardMaterial({ color: 0x2D36FF, roughness: .3, metalness: .4 }));
  scene.add(mesh, new THREE.AmbientLight(0xffffff, .6));
  const key = new THREE.DirectionalLight(0xFF7A45, 2.2); key.position.set(3, 4, 5); scene.add(key);

  const resize = () => { const { width: w, height: h } = host.getBoundingClientRect();
    renderer.setSize(w, h, false); camera.aspect = w / h; camera.updateProjectionMatrix(); };
  new ResizeObserver(resize).observe(host); resize();

  // El scroll maneja un "progress" 0..1; el loop solo lo lee.
  const state = { p: 0 };
  if (!reduce) gsap.to(state, { p: 1, ease: 'none',
    scrollTrigger: { trigger: host, start: 'top bottom', end: 'bottom top', scrub: 1 } });

  let visible = true;
  new IntersectionObserver(([en]) => visible = en.isIntersecting).observe(host);
  renderer.setAnimationLoop(() => {
    if (!visible || document.hidden) return;
    mesh.rotation.y = state.p * Math.PI * 2;
    mesh.rotation.x = state.p * Math.PI * .5;
    renderer.render(scene, camera);
  });
}, { rootMargin: '400px' }).observe(host);
</script>
```

## 8. Parallax de capas

```js
gsap.utils.toArray('[data-speed]').forEach(el => {
  gsap.to(el, { yPercent: () => -parseFloat(el.dataset.speed) * 30, ease: 'none',
    scrollTrigger: { trigger: el.parentElement, start: 'top bottom', end: 'bottom top', scrub: true } });
});
```
`data-speed="0.5"` (lento) a `data-speed="2"` (rápido). En móvil, dividir por 2 con `gsap.matchMedia`.

## 9. Verificar con Playwright (capturas en distintos puntos del scroll)

```js
const { chromium } = require('playwright');
(async () => {
  const b = await chromium.launch(); const p = await b.newPage({ viewport: { width: 390, height: 844 } });
  await p.goto('file://' + process.cwd() + '/index.html');
  const h = await p.evaluate(() => document.body.scrollHeight);
  for (let i = 0; i <= 5; i++) {
    await p.evaluate(y => scrollTo(0, y), (h / 5) * i); await p.waitForTimeout(700);
    await p.screenshot({ path: `shot-${i}.png` });
  }
  await b.close();
})();
```
