# Horizontal Parallax Gallery

> **Stack:** HTML · CSS · JavaScript · GSAP (CDN)  
> **No build tools. No frameworks. Open `index.html` in a browser and it works.**

You will build this completely from scratch. Each lesson explains the concept first, then tells you exactly what to write. Don't copy-paste — type it out.

---

## Table of Contents

1. [What You're Building](#1-what-youre-building)
2. [Project Setup](#2-project-setup)
3. [Lesson 1 — Lerp: The Secret Behind Smooth Motion](#3-lesson-1--lerp-the-secret-behind-smooth-motion)
4. [Lesson 2 — The CSS Parallax Buffer](#4-lesson-2--the-css-parallax-buffer)
5. [Lesson 3 — The HTML](#5-lesson-3--the-html)
6. [Lesson 4 — The CSS](#6-lesson-4--the-css)
7. [Lesson 5 — Smooth Horizontal Scroll](#7-lesson-5--smooth-horizontal-scroll)
8. [Lesson 6 — The Parallax Effect](#8-lesson-6--the-parallax-effect)
9. [Lesson 7 — Wire Everything Up](#9-lesson-7--wire-everything-up)
10. [Tuning Reference](#10-tuning-reference)

---

## 1. What You're Building

A full-screen horizontal gallery where:

- You scroll **vertically** with the mouse wheel → the gallery moves **horizontally**
- The scroll **glides and eases** instead of teleporting (lerp)
- Each image shifts slightly as it moves through the viewport — **parallax depth**
- Everything runs at 60fps using **GPU-accelerated CSS transforms**

No WebGL. No canvas. Just `translateX` done right.

---

## 2. Project Setup

Create the following three empty files in this folder:

```
horizontal-parallax-scroll/
├── index.html
├── style.css
└── main.js
```

That's it. You'll fill each one as you go through the lessons.

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

---

## 5. Lesson 3 — The HTML

**File: `index.html`**

Open `index.html` and write the following structure. Each piece is explained inline.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Horizontal Parallax Gallery</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body>
```

Now the gallery structure. Three layers:

```html
  <!-- Layer 1: the viewport window -->
  <div class="gallery__wrapper">

    <!-- Layer 2: the horizontal strip that moves -->
    <div class="gallery__track">

      <!-- Layer 3: one card per image -->
      <div class="gallery__item">
        <img
          src="https://images.unsplash.com/photo-1506905925346-21bda4d32df4?w=800&q=80"
          alt="Mountain landscape"
          draggable="false"
        />
      </div>

      <div class="gallery__item">
        <img
          src="https://images.unsplash.com/photo-1469474968028-56623f02e42e?w=800&q=80"
          alt="Forest path"
          draggable="false"
        />
      </div>

      <div class="gallery__item">
        <img
          src="https://images.unsplash.com/photo-1500534314209-a25ddb2bd429?w=800&q=80"
          alt="Ocean view"
          draggable="false"
        />
      </div>

      <div class="gallery__item">
        <img
          src="https://images.unsplash.com/photo-1447752875215-b2761acb3c5d?w=800&q=80"
          alt="Autumn trees"
          draggable="false"
        />
      </div>

      <div class="gallery__item">
        <img
          src="https://images.unsplash.com/photo-1464822759023-fed622ff2c3b?w=800&q=80"
          alt="Mountain peaks"
          draggable="false"
        />
      </div>

      <div class="gallery__item">
        <img
          src="https://images.unsplash.com/photo-1682687221038-404cb8830901?w=800&q=80"
          alt="Desert dunes"
          draggable="false"
        />
      </div>

    </div>
  </div>
```

Close with GSAP from CDN and your script:

```html
  <!-- GSAP from CDN — no npm, no bundler -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js"></script>
  <script src="main.js"></script>
</body>
</html>
```

**What each class does:**

| Class | Role |
|---|---|
| `.gallery__wrapper` | Full-screen viewport. `overflow: hidden` clips the moving track. |
| `.gallery__track` | Horizontal strip. This is the element you'll `translateX`. |
| `.gallery__item` | Individual card. Has `overflow: hidden` — the parallax "window". |
| `img` | The image. Will be 125% wide with room to shift. |

> **Why `draggable="false"`?**  
> Without it, clicking and dragging an image triggers the browser's native "drag image" ghost. This interrupts your scroll events.

**Checkpoint:** Open `index.html` in a browser. You should see unstyled images stacked vertically. It looks wrong — that's expected. CSS fixes that next.

---

## 6. Lesson 4 — The CSS

**File: `style.css`**

Write each block and read the comment above it before moving on.

### Reset and base

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
    Without this, the browser would scroll the page AND your gallery scrolls —
    causing a double-scroll bug.
  */
  overflow: hidden;
  background: #0f0f0f;
  font-family: system-ui, sans-serif;
}
```

### The wrapper — your viewport window

```css
.gallery__wrapper {
  width: 100vw;
  height: 100vh;
  overflow: hidden;       /* Clips anything outside the viewport */
  display: flex;
  align-items: center;    /* Cards are vertically centered */
}
```

### The track — what you'll translateX

```css
.gallery__track {
  display: flex;          /* Side-by-side layout */
  gap: 1.5rem;
  padding: 0 5vw;         /* Side breathing room */
  /*
    will-change: transform tells the browser this element transforms frequently.
    The browser promotes it to its own GPU compositor layer.
    Transforms on compositor layers skip layout and paint — only compositing runs.
    This is why it's smooth even at 60fps.
  */
  will-change: transform;
}
```

### The card — the parallax "window"

```css
.gallery__item {
  flex-shrink: 0;         /* Don't let flex crush the cards */
  width: 38vw;
  height: 58vh;
  /*
    THIS IS THE CRITICAL LINE.
    The image inside is 125% wide.
    Without overflow: hidden, you'd see image edges bleeding out of the card.
    With it, the card acts as a clipping mask — only the centered portion is visible.
  */
  overflow: hidden;
  border-radius: 6px;
}
```

### The image — the parallax buffer

```css
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
    Without this, the image would start flush left and only extend to the right.
  */
  margin-left: -12.5%;

  object-fit: cover;      /* Prevent distortion as dimensions vary */
  will-change: transform; /* GPU layer hint for per-image transforms */
  pointer-events: none;
  user-select: none;
}
```

**Checkpoint:** Refresh `index.html`. You should now see a horizontal row of images, all clipped inside their cards, centered on screen. Scrolling does nothing yet — that's Lesson 5.

---

## 7. Lesson 5 — Smooth Horizontal Scroll

**File: `main.js`**

Start writing `main.js`. Build it section by section.

### Grab your elements

```js
const wrapper = document.querySelector('.gallery__wrapper')
const track   = document.querySelector('.gallery__track')
const images  = document.querySelectorAll('.gallery__item img')
```

### Import the GSAP clamp utility

```js
// gsap.utils.clamp(min, max, value) — keeps a value within a range.
// Prevents scrolling before the start or past the last image.
const clamp = gsap.utils.clamp
```

### Define scroll state

```js
/*
  This object is the heart of the scroll system.

  current → where the track physically is right now (px)
  target  → where we WANT the track to be (px)
  ease    → the lerp factor (how fast current chases target)
  limit   → max scrollable distance, calculated from the DOM
*/
const scroll = {
  current: 0,
  target:  0,
  ease:    0.07,
  limit:   0,
}
```

### Calculate the scroll limit

```js
/*
  How far can we scroll?
  Total track width minus what's already visible in the viewport.

  Example:
    track.scrollWidth = 4000px (all images + gaps)
    wrapper.clientWidth = 1440px (viewport)
    limit = 2560px — we can scroll 2560px before running out of content
*/
function setLimit() {
  scroll.limit = track.scrollWidth - wrapper.clientWidth
}
```

### Capture wheel input

```js
/*
  When the user scrolls the wheel (vertical), add the delta to our target.
  We're deliberately converting vertical wheel movement into horizontal travel.

  { passive: true } — we promise not to call e.preventDefault() here.
  This lets the browser skip a blocking check on every scroll event.
*/
window.addEventListener('wheel', (e) => {
  scroll.target += e.deltaY
}, { passive: true })
```

### The render loop

```js
/*
  gsap.ticker.add() fires on every animation frame — like requestAnimationFrame,
  but synced with GSAP's internal clock for better accuracy across devices.

  Every frame:
    1. Clamp target so it never goes below 0 or above the limit
    2. Lerp current toward target
    3. Apply the result as translateX to the track
*/
gsap.ticker.add(() => {
  scroll.target = clamp(0, scroll.limit, scroll.target)

  // Apply lerp: current += (target - current) * ease
  scroll.current += (scroll.target - scroll.current) * scroll.ease

  // Negative because we move the track LEFT as scroll increases
  gsap.set(track, { x: -scroll.current })
})

/*
  Disable lag smoothing.

  GSAP normally compensates for large time gaps between frames
  (e.g. when a tab was hidden and becomes active again).
  With lag smoothing ON, a hidden tab re-appearing can cause the gallery
  to "catch up" — a violent single-frame jump.
  Setting it to 0 disables this behavior.
*/
gsap.ticker.lagSmoothing(0)
```

### Handle resize

```js
window.addEventListener('resize', () => {
  setLimit()
}, { passive: true })
```

### Initialize

```js
setLimit()
```

**Checkpoint:** Refresh. Scrolling the mouse wheel should now move the gallery horizontally with a smooth ease. The parallax is still missing — images don't shift yet. That's next.

---

## 8. Lesson 6 — The Parallax Effect

Still in `main.js`. Add these two pieces.

### Why quickSetter instead of gsap.set?

```js
/*
  For a few elements, gsap.set(el, { x: val }) per frame is fine.
  For multiple elements every frame, gsap.quickSetter is faster.

  quickSetter(element, property, unit) returns a plain function.
  Calling that function batches DOM writes and skips unnecessary overhead.

  setTrackX(-300)  →  same as  gsap.set(track, { x: -300 })
  setImageX[0](8)  →  same as  gsap.set(images[0], { x: '8%' })
*/
const setTrackX = gsap.quickSetter(track, 'x', 'px')
const setImageX = Array.from(images).map(img => gsap.quickSetter(img, 'x', '%'))
```

Replace the `gsap.set(track, { x: -scroll.current })` line in your ticker with `setTrackX(-scroll.current)`.

### The parallax calculation

```js
/*
  MAX_SHIFT is the maximum image offset in % of its own width.

  Safe limit from Lesson 2:
    Extra buffer per side = 12.5% of card width
    As % of image width (125% of card): 12.5 / 125 * 100 = 10%

  Raise this above 10 and you'll see image edges — experiment to understand.
*/
const MAX_SHIFT = 10

function applyParallax() {
  const vw     = window.innerWidth
  const center = vw * 0.5   // The midpoint of the viewport

  images.forEach((img, i) => {
    const item = img.closest('.gallery__item')
    const rect = item.getBoundingClientRect()

    /*
      getBoundingClientRect() gives the item's current position in the viewport.
      This is accurate because it reflects the translateX we applied to the track.
      No need to manually account for scroll — the DOM already reflects it.
    */

    // Skip items not currently visible — no point calculating parallax off-screen
    if (rect.right < 0 || rect.left > vw) return

    // Center of this item relative to the viewport (in px)
    const itemCenter = rect.left + rect.width * 0.5

    /*
      Normalize: express the item's position as a value from -1 to 1.
        -1 → item center is at the LEFT edge of the viewport
         0 → item center is at the CENTER of the viewport
        +1 → item center is at the RIGHT edge of the viewport
    */
    const t = clamp(-1, 1, (itemCenter - center) / center)

    /*
      Counter-motion (the depth illusion):
        Item moving RIGHT through viewport → image shifts LEFT
        Item at center → image at 0 offset
        Item moving LEFT through viewport → image shifts RIGHT

      The image "lags behind" the card. Because they move at different rates,
      your brain perceives them as being on different depth planes.
    */
    const shift = -t * MAX_SHIFT

    setImageX[i](shift)
  })
}
```

### Call it inside the ticker

Inside `gsap.ticker.add(...)`, after the `setTrackX` line, add:

```js
applyParallax()
```

Your full ticker should now look like this:

```js
gsap.ticker.add(() => {
  scroll.target = clamp(0, scroll.limit, scroll.target)
  scroll.current += (scroll.target - scroll.current) * scroll.ease
  setTrackX(-scroll.current)
  applyParallax()
})
```

**Checkpoint:** Refresh. As you scroll, each image should visibly shift horizontally within its card. Images near the edges shift more than images near the center. That's the parallax.

---

## 9. Lesson 7 — Wire Everything Up

Here is the complete `main.js` for reference. Use it to check your work — don't copy it before writing your own.

```js
const wrapper = document.querySelector('.gallery__wrapper')
const track   = document.querySelector('.gallery__track')
const images  = document.querySelectorAll('.gallery__item img')

const clamp = gsap.utils.clamp

const scroll = {
  current: 0,
  target:  0,
  ease:    0.07,
  limit:   0,
}

const MAX_SHIFT = 10

const setTrackX = gsap.quickSetter(track, 'x', 'px')
const setImageX = Array.from(images).map(img => gsap.quickSetter(img, 'x', '%'))

function setLimit() {
  scroll.limit = track.scrollWidth - wrapper.clientWidth
}

function applyParallax() {
  const vw     = window.innerWidth
  const center = vw * 0.5

  images.forEach((img, i) => {
    const item = img.closest('.gallery__item')
    const rect = item.getBoundingClientRect()

    if (rect.right < 0 || rect.left > vw) return

    const itemCenter = rect.left + rect.width * 0.5
    const t          = clamp(-1, 1, (itemCenter - center) / center)
    const shift      = -t * MAX_SHIFT

    setImageX[i](shift)
  })
}

gsap.ticker.add(() => {
  scroll.target  = clamp(0, scroll.limit, scroll.target)
  scroll.current += (scroll.target - scroll.current) * scroll.ease
  setTrackX(-scroll.current)
  applyParallax()
})

gsap.ticker.lagSmoothing(0)

window.addEventListener('wheel', (e) => {
  scroll.target += e.deltaY
}, { passive: true })

window.addEventListener('resize', () => {
  setLimit()
}, { passive: true })

setLimit()
```

---

## 10. Tuning Reference

Once it's working, experiment with these values to understand how they interact.

| Variable | File | What it does |
|---|---|---|
| `scroll.ease` | `main.js` | Lerp speed. `0.03` = dreamy glide, `0.15` = snappy |
| `MAX_SHIFT` | `main.js` | Parallax intensity. Max safe value = `10` with current CSS |
| `width: 125%` | `style.css` | Image buffer size. Increase for stronger parallax room |
| `margin-left: -12.5%` | `style.css` | Must always equal `-(width - 100%) / 2` |
| `gap` | `style.css` | Space between cards |
| `width: 38vw` | `style.css` | Card width |

**The buffer triangle — always keep these in sync:**

```
Image width:     W%
Margin-left:    -(W - 100) / 2 %
Max safe shift:  (W - 100) / 2 / W * 100 %

Example with W = 130:
  margin-left:    -15%
  max safe shift:  15/130*100 ≈ 11.5%
```

Break the rules intentionally — set `MAX_SHIFT = 20` and watch the edges appear. That's how you learn where the boundaries are.
