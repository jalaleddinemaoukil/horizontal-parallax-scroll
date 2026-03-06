# Horizontal Parallax Gallery

> **Stack:** Vue 3 · Vite · GSAP · pnpm  
> **Composition API · Single File Components · Scoped CSS**

You will build this completely from scratch using Vue 3. Each lesson explains the concept first, then tells you exactly what to write. Don't copy-paste — type it out.

---

## Table of Contents

1. [What You're Building](#1-what-youre-building)
2. [Project Setup](#2-project-setup)
3. [Lesson 1 — Lerp: The Secret Behind Smooth Motion](#3-lesson-1--lerp-the-secret-behind-smooth-motion)
4. [Lesson 2 — The CSS Parallax Buffer](#4-lesson-2--the-css-parallax-buffer)
5. [Lesson 3 — Vue Concepts You'll Use](#5-lesson-3--vue-concepts-youll-use)
6. [Lesson 4 — The Template](#6-lesson-4--the-template)
7. [Lesson 5 — Global CSS and Scoped Styles](#7-lesson-5--global-css-and-scoped-styles)
8. [Lesson 6 — Smooth Horizontal Scroll in Vue](#8-lesson-6--smooth-horizontal-scroll-in-vue)
9. [Lesson 7 — The Parallax Effect](#9-lesson-7--the-parallax-effect)
10. [Lesson 8 — Wire Everything Up](#10-lesson-8--wire-everything-up)
11. [Tuning Reference](#11-tuning-reference)

---

## 1. What You're Building

A full-screen horizontal gallery where:

- You scroll **vertically** with the mouse wheel → the gallery moves **horizontally**
- The scroll **glides and eases** instead of teleporting (lerp)
- Each image shifts slightly as it moves through the viewport — **parallax depth**
- Everything runs at 60fps using **GPU-accelerated CSS transforms**

No WebGL. No canvas. Just `translateX` done right — now inside a Vue 3 Single File Component.

---

## 2. Project Setup

### Scaffold with pnpm

Run this in your terminal. Answer the prompts as shown:

```bash
pnpm create vue@latest
```

```
✔ Project name: horizontal-parallax-scroll
✔ Add TypeScript? › No
✔ Add JSX Support? › No
✔ Add Vue Router? › No
✔ Add Pinia? › No
✔ Add Vitest? › No
✔ Add an End-to-End Testing Solution? › No
✔ Add ESLint? › Yes
✔ Add Prettier? › Yes
✔ Add Vue DevTools? › No
```

Then install dependencies and install GSAP:

```bash
cd horizontal-parallax-scroll
pnpm install
pnpm add gsap
```

Start the dev server:

```bash
pnpm dev
```

### Clean up the scaffold

Delete everything Vue generated that you don't need:

```bash
rm -rf src/assets src/components/HelloWorld.vue src/components/TheWelcome.vue src/components/WelcomeItem.vue src/components/icons
```

### Project structure you'll end up with

```
horizontal-parallax-scroll/
├── public/
│   └── images/              ← move your images here
├── src/
│   ├── components/
│   │   └── GalleryParallax.vue
│   ├── styles/
│   │   └── global.css
│   ├── App.vue
│   └── main.js
├── index.html
├── package.json
└── vite.config.js
```

Move your images into `public/images/`. Vite serves `public/` at the root, so `src="/images/Serene Sand Dunes.png"` will work in the template.

---

## 3. Lesson 1 — Lerp: The Secret Behind Smooth Motion

**Read this before writing any code.**

Most scroll implementations look jerky because they teleport the element to the new position on every wheel event. The fix is **lerp** — linear interpolation.

The formula:

```
current = current + (target - current) * ease
```

Plain English: *each frame, close N% of the remaining gap between where you are and where you want to be.*

Trace through an example with `ease = 0.07` and `target = 1000`:

```
Start:   current = 0
Frame 1: current = 0   + (1000 - 0)   * 0.07 = 70
Frame 2: current = 70  + (1000 - 70)  * 0.07 = 135.1
Frame 3: current = 135 + (1000 - 135) * 0.07 = 195.6
...
Frame N: current ≈ 1000  (never exactly, but close enough)
```

Notice: it starts fast and **decelerates** as it closes in — that's the natural ease-out feel you see in polished UIs.

**Key insight:** You always have two values in memory:
- `target` — jumps immediately when the user scrolls
- `current` — chases `target` slowly, one frame at a time

The DOM reads `current`, not `target`. That's the entire trick.

> **Check your understanding:** If `ease = 0.5`, would the scroll feel faster or slower than `ease = 0.07`? Why?  
> *(Answer: faster — you close 50% of the gap each frame instead of 7%.)*

---

## 4. Lesson 2 — The CSS Parallax Buffer

**Read this before writing any CSS.**

Parallax means elements move at **different speeds**. In a gallery, the card (frame) moves with the scroll. The image *inside* the card shifts slightly in the opposite direction — this offset creates the illusion of depth.

For this to work, the image must have **physical room to move** inside its container without exposing blank space. Here's how:

```
Card:   400px wide  ← overflow: hidden (acts as a window)
Image:  500px wide  ← 25% larger than card
Offset: -50px left  ← centered so it can shift in both directions
```

Visualize it:

```
Card boundary:     |←────── 400px ──────→|

Image (500px):  ←─|───── visible ────────|─→
                   ↑                     ↑
              12.5% hidden          12.5% hidden
              on the left           on the right
```

The image sticks out equally on both sides. JavaScript can shift it left or right by up to 12.5% of the card width — and you'll never see the edge.

**The math you need:**

```
Image width:     125%         (25% extra)
Margin-left:    -12.5%        (= extra / 2, centers it)
Max safe shift:  10%          (of image width = 12.5% of card width)
```

> **Why 10% and not 12.5%?** `translateX(%)` is relative to the *image's own width* (125% of card), not the card. So the safe limit = `12.5 / 125 * 100 = 10%`.

Keep these numbers in your head as you write the CSS and JS.


## 5. Lesson 3 — Vue Concepts You'll Use

**Read this before writing any Vue code.**

You're using Vue 3 with the **Composition API** and **Single File Components (SFCs)**. Here's what that means for this project.

### Single File Component structure

Every `.vue` file has three sections:

```vue
<script setup>
  // JavaScript logic — runs once when the component mounts
  // "setup" means you don't need to export anything — Vue handles it
</script>

<template>
  <!-- HTML markup — Vue compiles this to DOM operations -->
</template>

<style scoped>
  /* CSS — "scoped" means it only applies to this component */
  /* Vue adds a unique attribute (e.g. data-v-7ba5bd90) to scope it */
</style>
```

### Template refs — how you get DOM elements

In plain JS you'd write `document.querySelector('.gallery__track')`. In Vue, you use `ref` and bind it with the `ref` attribute in the template:

```vue
<script setup>
import { ref } from 'vue'

const trackRef = ref(null)   // starts as null
</script>

<template>
  <!-- Vue fills trackRef.value with the DOM element after mount -->
  <div ref="trackRef"></div>
</template>
```

After `onMounted` fires, `trackRef.value` is the actual `<div>` DOM element. Before that, it's `null`.

### v-for — rendering a list

Instead of copy-pasting five `<div class="gallery__item">` blocks, you drive them from a data array:

```vue
<script setup>
const images = [
  { src: '/images/Serene Sand Dunes.png', alt: 'Serene Sand Dunes' },
  // ...
]
</script>

<template>
  <div v-for="(image, i) in images" :key="i" class="gallery__item">
    <img :src="image.src" :alt="image.alt" />
  </div>
</template>
```

`:src` and `:alt` are shorthand for `v-bind:src` and `v-bind:alt` — they evaluate the expression as JavaScript, not a plain string.

### Collecting multiple refs with v-for

You need a ref to each `<img>` element, not just one. The pattern is a ref array + a callback:

```vue
<script setup>
import { ref } from 'vue'

const imageRefs = ref([])
</script>

<template>
  <img
    v-for="(image, i) in images"
    :key="i"
    :ref="el => { if (el) imageRefs.value[i] = el }"
  />
</template>
```

Vue calls the function for each item. After mount, `imageRefs.value` is an array of DOM elements.

### onMounted and onUnmounted — lifecycle hooks

GSAP needs real DOM elements. You can only access them after Vue has rendered. `onMounted` fires right after:

```js
import { onMounted, onUnmounted } from 'vue'

onMounted(() => {
  // DOM is ready — attach GSAP, add event listeners
})

onUnmounted(() => {
  // Component is being torn down — remove listeners, kill ticker
  // Without this, event listeners and ticker callbacks leak memory
})
```

`onUnmounted` is the cleanup hook. In a real app the gallery component might get unmounted (e.g. navigating to another route). Always clean up.

> **Why not use `reactive()` for `scroll`?**  
> The `scroll` object is updated 60 times per second by GSAP. Vue's reactivity system would trigger re-renders on every change. You don't want that — GSAP owns the DOM updates here. A plain `{}` object is intentional: it's invisible to Vue's reactivity.

---

## 6. Lesson 4 — The Template

**File: `src/components/GalleryParallax.vue`** — create this file.

Start with the `<template>` block. Three layers, same as before — but now the image list is data-driven.

```vue
<template>
  <!-- Layer 1: the viewport window -->
  <div class="gallery__wrapper" ref="wrapperRef">

    <!-- Layer 2: the horizontal strip that moves -->
    <div class="gallery__track" ref="trackRef">

      <!-- Layer 3: one card per image — v-for replaces repeated markup -->
      <div
        v-for="(image, i) in images"
        :key="i"
        class="gallery__item"
      >
        <img
          :ref="el => { if (el) imageRefs[i] = el }"
          :src="image.src"
          :alt="image.alt"
          draggable="false"
        />
      </div>

    </div>
  </div>
</template>
```

**What changed from plain HTML:**

| Old (HTML) | New (Vue) |
|---|---|
| Five `<div class="gallery__item">` blocks | One block with `v-for` |
| `id="wrapper"` + `querySelector` | `ref="wrapperRef"` |
| `<script src="gsap.min.js">` CDN tag | `import gsap from 'gsap'` in script |
| `document.querySelectorAll('img')` | `imageRefs` array filled by `:ref` callback |

Add the image data in `<script setup>` (you'll build the full script section in Lesson 6):

```js
const images = [
  { src: '/images/Serene Sand Dunes.png',              alt: 'Serene Sand Dunes' },
  { src: '/images/Serene Coastal Reflection.png',      alt: 'Serene Coastal Reflection' },
  { src: '/images/Dramatic Mountain Landscape.png',    alt: 'Dramatic Mountain Landscape' },
  { src: '/images/Misty Tropical Rainforest.png',      alt: 'Misty Tropical Rainforest' },
  { src: '/images/Abstract Cityscape Night.png',       alt: 'Abstract Cityscape Night' },
]
```

> **Why `/images/...` and not `./images/...`?**  
> Vite serves the `public/` folder at the root `/`. A leading `/` means "from the server root" — it always works regardless of which route you're on. `./images/...` is a relative import path and would break inside `src/`.

**Checkpoint:** Add `<GalleryParallax />` to `src/App.vue` and visit `localhost:5173`. You should see unstyled images stacked. That's expected — styles come next.

---

## 7. Lesson 5 — Global CSS and Scoped Styles

Two CSS files. Different jobs.

### `src/styles/global.css` — the reset

Create this file. It touches `html` and `body` — things that exist outside any component, so they can't be scoped.

```css
*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

html, body {
  width: 100%;
  height: 100%;
  /*
    Disable native scroll entirely.
    You're replacing it with your own system in JS.
    Without this, the browser would scroll the page AND the gallery —
    causing a double-scroll bug.
  */
  overflow: hidden;
  background: #0f0f0f;
  font-family: system-ui, sans-serif;
}
```

Import it in `src/main.js`:

```js
import { createApp } from 'vue'
import './styles/global.css'
import App from './App.vue'

createApp(App).mount('#app')
```

### `<style scoped>` in `GalleryParallax.vue`

Add this at the bottom of the component. Same rules as before — the only difference is `scoped` prevents these styles from leaking into other components.

```vue
<style scoped>
/* The viewport window */
.gallery__wrapper {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  display: flex;
  align-items: center;
}

/* The horizontal strip — what you'll translateX */
.gallery__track {
  display: flex;
  gap: 1.5rem;
  padding: 0 5vw;
  /*
    will-change: transform tells the browser this element transforms frequently.
    The browser promotes it to its own GPU compositor layer.
    Transforms on compositor layers skip layout and paint — only compositing runs.
    This is why it's smooth even at 60fps.
  */
  will-change: transform;
}

/* The card — the parallax "window" */
.gallery__item {
  flex-shrink: 0;
  width: 38vw;
  height: 58vh;
  /*
    The image inside is 125% wide.
    Without overflow: hidden, you'd see image edges bleeding out of the card.
    With it, the card acts as a clipping mask.
  */
  overflow: hidden;
  border-radius: 6px;
}

/* The image — the parallax buffer */
.gallery__item img {
  display: block;
  /*
    Make the image wider than its container.
    Extra width = 25% (12.5% hidden on each side).
    This is the physical space JS will shift the image into.
  */
  width: 125%;
  height: 100%;
  /*
    Center the oversized image.
    Formula: -(width - 100%) / 2 = -(125 - 100) / 2 = -12.5%
  */
  margin-left: -12.5%;
  object-fit: cover;
  will-change: transform;
  pointer-events: none;
  user-select: none;
}
</style>
```

**Checkpoint:** Refresh. You should now see a horizontal row of images, all clipped inside their cards, centered on screen. Scrolling does nothing yet — that's Lesson 6.

---

## 8. Lesson 6 — Smooth Horizontal Scroll in Vue

Now build the `<script setup>` block in `GalleryParallax.vue`. The logic is identical to what you'd write in plain JS — the difference is *where* it runs (inside `onMounted`) and *how* you access the DOM (via `ref`).

### Imports and refs

```js
import { ref, onMounted, onUnmounted } from 'vue'
import gsap from 'gsap'

// DOM refs — populated after mount
const wrapperRef = ref(null)
const trackRef   = ref(null)
const imageRefs  = []          // plain array — no reactivity needed, GSAP owns this

// Image data
const images = [
  { src: '/images/Serene Sand Dunes.png',           alt: 'Serene Sand Dunes' },
  { src: '/images/Serene Coastal Reflection.png',   alt: 'Serene Coastal Reflection' },
  { src: '/images/Dramatic Mountain Landscape.png', alt: 'Dramatic Mountain Landscape' },
  { src: '/images/Misty Tropical Rainforest.png',   alt: 'Misty Tropical Rainforest' },
  { src: '/images/Abstract Cityscape Night.png',    alt: 'Abstract Cityscape Night' },
]
```

### Scroll state and constants

Same as in plain JS — a plain object, invisible to Vue reactivity:

```js
const clamp = gsap.utils.clamp

const scroll = {
  current: 0,
  target:  0,
  ease:    0.07,
  limit:   0,
}

const MAX_SHIFT = 10
```

### setLimit

```js
/*
  How far can we scroll?
  Total track width minus what's already visible in the viewport.
  .value unwraps the ref to get the underlying DOM element.
*/
function setLimit() {
  scroll.limit = trackRef.value.scrollWidth - wrapperRef.value.clientWidth
}
```

### onMounted — where everything starts

Inside `onMounted`, the DOM is real. Create the quickSetters, register the ticker, and attach event listeners:

```js
// Store references so onUnmounted can remove them
let tickerCb
let wheelCb
let resizeCb

onMounted(() => {
  // quickSetter returns a plain function — faster than gsap.set per frame
  const setTrackX = gsap.quickSetter(trackRef.value, 'x', 'px')
  const setImageX = imageRefs.map(img => gsap.quickSetter(img, 'x', '%'))

  setLimit()

  // The render loop — same lerp + parallax logic as before
  tickerCb = () => {
    scroll.target  = clamp(0, scroll.limit, scroll.target)
    scroll.current += (scroll.target - scroll.current) * scroll.ease
    setTrackX(-scroll.current)
    applyParallax(setImageX)
  }

  wheelCb  = (e) => { scroll.target += e.deltaY }
  resizeCb = ()  => { setLimit() }

  gsap.ticker.add(tickerCb)
  gsap.ticker.lagSmoothing(0)
  window.addEventListener('wheel',  wheelCb,  { passive: true })
  window.addEventListener('resize', resizeCb, { passive: true })
})
```

> **Why store `tickerCb`, `wheelCb`, `resizeCb` in variables?**  
> You need the same function reference to remove a listener. If you wrote  
> `window.removeEventListener('wheel', (e) => {...})` it would do nothing  
> because that's a brand new anonymous function, not the one you added.

### onUnmounted — always clean up

```js
onUnmounted(() => {
  gsap.ticker.remove(tickerCb)
  window.removeEventListener('wheel',  wheelCb)
  window.removeEventListener('resize', resizeCb)
})
```

---

## 9. Lesson 7 — The Parallax Effect

### applyParallax

Define this function *before* `onMounted` — it receives `setImageX` as a parameter since that's created inside `onMounted`:

```js
function applyParallax(setImageX) {
  const vw     = window.innerWidth
  const center = vw * 0.5

  imageRefs.forEach((img, i) => {
    const item = img.closest('.gallery__item')
    const rect = item.getBoundingClientRect()

    /*
      getBoundingClientRect() reflects the translateX already applied to the track.
      It gives the item's actual position in the viewport right now.
    */

    // Skip items that are completely off-screen
    if (rect.right < 0 || rect.left > vw) return

    // Where is this card's center relative to the viewport center?
    const itemCenter = rect.left + rect.width * 0.5

    /*
      Normalize to [-1, 1]:
        -1 → card center at left edge of viewport
         0 → card center at viewport center
        +1 → card center at right edge of viewport
    */
    const t     = clamp(-1, 1, (itemCenter - center) / center)

    /*
      Counter-motion creates depth:
        card moving right → image shifts left (negative shift)
        card at center    → image at 0
        card moving left  → image shifts right (positive shift)
    */
    const shift = -t * MAX_SHIFT

    setImageX[i](shift)
  })
}
```

**Checkpoint:** Save the file. Vite hot-reloads instantly. Scrolling should now move the gallery horizontally with smooth ease, and each image should visibly shift within its card as it passes through the viewport.

---

## 10. Lesson 8 — Wire Everything Up

Here is the complete `GalleryParallax.vue`. Use it to verify your work — don't copy it before writing your own version.

```vue
<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import gsap from 'gsap'

// ── DOM refs ──────────────────────────────────────────────
const wrapperRef = ref(null)
const trackRef   = ref(null)
const imageRefs  = []   // plain array — GSAP owns updates, not Vue

// ── Image data ────────────────────────────────────────────
const images = [
  { src: '/images/Serene Sand Dunes.png',           alt: 'Serene Sand Dunes' },
  { src: '/images/Serene Coastal Reflection.png',   alt: 'Serene Coastal Reflection' },
  { src: '/images/Dramatic Mountain Landscape.png', alt: 'Dramatic Mountain Landscape' },
  { src: '/images/Misty Tropical Rainforest.png',   alt: 'Misty Tropical Rainforest' },
  { src: '/images/Abstract Cityscape Night.png',    alt: 'Abstract Cityscape Night' },
]

// ── Scroll state ──────────────────────────────────────────
const clamp = gsap.utils.clamp

const scroll = {
  current: 0,
  target:  0,
  ease:    0.07,
  limit:   0,
}

const MAX_SHIFT = 10

// ── Helpers ───────────────────────────────────────────────
function setLimit() {
  scroll.limit = trackRef.value.scrollWidth - wrapperRef.value.clientWidth
}

function applyParallax(setImageX) {
  const vw     = window.innerWidth
  const center = vw * 0.5

  imageRefs.forEach((img, i) => {
    const item = img.closest('.gallery__item')
    const rect = item.getBoundingClientRect()

    if (rect.right < 0 || rect.left > vw) return

    const itemCenter = rect.left + rect.width * 0.5
    const t          = clamp(-1, 1, (itemCenter - center) / center)
    const shift      = -t * MAX_SHIFT

    setImageX[i](shift)
  })
}

// ── Lifecycle ─────────────────────────────────────────────
let tickerCb
let wheelCb
let resizeCb

onMounted(() => {
  const setTrackX = gsap.quickSetter(trackRef.value, 'x', 'px')
  const setImageX = imageRefs.map(img => gsap.quickSetter(img, 'x', '%'))

  setLimit()

  tickerCb = () => {
    scroll.target  = clamp(0, scroll.limit, scroll.target)
    scroll.current += (scroll.target - scroll.current) * scroll.ease
    setTrackX(-scroll.current)
    applyParallax(setImageX)
  }

  wheelCb  = (e) => { scroll.target += e.deltaY }
  resizeCb = ()  => { setLimit() }

  gsap.ticker.add(tickerCb)
  gsap.ticker.lagSmoothing(0)
  window.addEventListener('wheel',  wheelCb,  { passive: true })
  window.addEventListener('resize', resizeCb, { passive: true })
})

onUnmounted(() => {
  gsap.ticker.remove(tickerCb)
  window.removeEventListener('wheel',  wheelCb)
  window.removeEventListener('resize', resizeCb)
})
</script>

<template>
  <div class="gallery__wrapper" ref="wrapperRef">
    <div class="gallery__track" ref="trackRef">
      <div
        v-for="(image, i) in images"
        :key="i"
        class="gallery__item"
      >
        <img
          :ref="el => { if (el) imageRefs[i] = el }"
          :src="image.src"
          :alt="image.alt"
          draggable="false"
        />
      </div>
    </div>
  </div>
</template>

<style scoped>
.gallery__wrapper {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  display: flex;
  align-items: center;
}

.gallery__track {
  display: flex;
  gap: 1.5rem;
  padding: 0 5vw;
  will-change: transform;
}

.gallery__item {
  flex-shrink: 0;
  width: 38vw;
  height: 58vh;
  overflow: hidden;
  border-radius: 6px;
}

.gallery__item img {
  display: block;
  width: 125%;
  height: 100%;
  margin-left: -12.5%;
  object-fit: cover;
  will-change: transform;
  pointer-events: none;
  user-select: none;
}
</style>
```

### `src/App.vue`

```vue
<script setup>
import GalleryParallax from './components/GalleryParallax.vue'
</script>

<template>
  <GalleryParallax />
</template>
```

### `src/main.js`

```js
import { createApp } from 'vue'
import './styles/global.css'
import App from './App.vue'

createApp(App).mount('#app')
```

---

## 11. Tuning Reference

Once it's working, experiment with these values to understand how they interact.

| Variable | File | What it does |
|---|---|---|
| `scroll.ease` | `GalleryParallax.vue` | Lerp speed. `0.03` = dreamy glide, `0.15` = snappy |
| `MAX_SHIFT` | `GalleryParallax.vue` | Parallax intensity. Max safe value = `10` with current CSS |
| `width: 125%` | `<style scoped>` | Image buffer size. Increase for stronger parallax room |
| `margin-left: -12.5%` | `<style scoped>` | Must always equal `-(width - 100%) / 2` |
| `gap` | `<style scoped>` | Space between cards |
| `width: 38vw` | `<style scoped>` | Card width |

**The buffer triangle — always keep these in sync:**

```
Image width:     W%
Margin-left:    -(W - 100) / 2 %
Max safe shift:  (W - 100) / 2 / W * 100 %

Example with W = 130:
  margin-left:    -15%
  max safe shift:  15/130*100 ≈ 11.5%
```

Break the rules intentionally — set `MAX_SHIFT = 20` and watch the edges appear. That's how you learn where the boundaries sit.

**Vue-specific things to try next:**

- Extract the scroll + parallax logic into a composable `src/composables/useParallaxScroll.js` and call it with `useParallaxScroll(wrapperRef, trackRef, imageRefs)` — keeps the component lean
- Add a `props` declaration for `ease` and `maxShift` so the gallery is configurable from the parent
- Pass the `images` array in as a prop — makes the component reusable across multiple pages
