# Svelte Neuro Shader

A lightweight, configurable **WebGL shader component** for **Svelte**.  
Smooth organic animation, color transitions, and zero external dependencies.
Based on https://codepen.io/ksenia-k/pen/vYwgrWv

![npm](https://img.shields.io/npm/v/@prinsc/svelte-neuro-shader)
![license](https://img.shields.io/npm/l/@prinsc/svelte-neuro-shader)

---

## Highlights

- Customizable base color (normalized RGB 0–1)  
- Adjustable animation speed and color transition speed  
- Tunable initial scale for the visual pattern  
- Zero runtime dependencies  
- SvelteKit friendly: render only in the browser  

---

## Installation

```bash

npm install @prinsc/svelte-neuro-shader
```
### Requirements

- **Svelte** ^4.x  
- **SvelteKit** (optional, but examples use SvelteKit’s browser guard)

---

## Quick Start (SvelteKit)

WebGL is browser-only. In SvelteKit, guard rendering with the browser flag so the component only mounts client-side.

```js

<script>
  import NeuroShader from '@prinsc/svelte-neuro-shader';
  import { browser } from '$app/environment';

  // Your preferred parameters
  let color = { r: 0.05, g: 0.35, b: 0.15 };
  let speed = 0.0002;
  let scale = 8.5;
</script>

{#if browser}
  <NeuroShader
    baseColor={color}
    timeSpeed={speed}
    initialScale={scale}
    colorTransitionSpeed={0.05}
  />
{/if}
```

**Tip:** To use the shader as a fullscreen background, wrap it in a positioned container or style the component’s canvas from your app stylesheet.

---

## Props

| Prop | Type | Default | Description |
|------|------|----------|-------------|
| `baseColor` | `{ r: number; g: number; b: number }` | `{ r: 0.1, g: 0.1, b: 0.1 }` | Base color, each channel in `[0, 1]`. |
| `timeSpeed` | `number` | `0.0002` | Animation speed (higher = faster). |
| `initialScale` | `number` | `8.5` | Spatial scale of the pattern (higher = finer detail/denser pattern). |
| `colorTransitionSpeed` | `number` | `0.05` | Interpolation speed for color changes (0–1). |

---

## Example Presets

```JS
<!-- Deep green -->
<NeuroShader baseColor={{ r: 0.05, g: 0.35, b: 0.15 }} />

<!-- Soft purple -->
<NeuroShader baseColor={{ r: 0.35, g: 0.2, b: 0.6 }} />

<!-- Warm amber -->
<NeuroShader baseColor={{ r: 0.7, g: 0.45, b: 0.15 }} />
```

---

## Full Example

```JS
<script>
  import NeuroShader from '@prinsc/svelte-neuro-shader';
  import { browser } from '$app/environment';

  let color = { r: 0.05, g: 0.35, b: 0.15 };
  let speed = 0.0002;
  let scale = 8.5;

  function randomize() {
    color = {
      r: +(Math.random().toFixed(2)),
      g: +(Math.random().toFixed(2)),
      b: +(Math.random().toFixed(2)),
    };
  }
</script>

{#if browser}
  <section class="hero">
    <NeuroShader
      baseColor={color}
      timeSpeed={speed}
      initialScale={scale}
      colorTransitionSpeed={0.05}
    />
    <div class="content">
      <h1>Svelte Neuro Shader</h1>
      <p>Organic motion and smooth colors.</p>
      <button on:click={randomize}>Randomize Color</button>
    </div>
  </section>
{/if}

<style>
  .hero {
    position: relative;
    min-height: 60vh;
    overflow: hidden;
  }
  .hero :global(canvas) {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
  }
  .content {
    position: relative;
    z-index: 1;
    padding: 3rem 1.5rem;
    color: white;
    text-shadow: 0 2px 12px rgba(0,0,0,0.35);
  }
  button {
    margin-top: 1rem;
  }
</style>
```
---

## SSR and Browser Usage

With SvelteKit, import `browser` from `$app/environment` and guard rendering using `{#if browser}`.  
If you are not using SvelteKit, ensure the component only mounts in environments where `window` and WebGL are available.

---

## Performance Tips

- Start with `timeSpeed` in the range 0.00005–0.001.  
- Larger `initialScale` increases detail density; tune based on device performance.  
- Avoid placing multiple fullscreen shaders on the same page if targeting low-power devices.  
- The component cleans up WebGL resources on unmount.

---

## Troubleshooting

| Issue | Cause / Fix |
|-------|--------------|
| **WebGL not supported** | Test your browser/GPU at [get.webgl.org](https://get.webgl.org) |
| **Black screen** | Ensure `baseColor` channels are within `[0, 1]`. |
| **Nothing renders on SvelteKit** | Verify the `{#if browser}` guard is present. |


---

## License

**MIT © Klyuvitkin Eléazar**
