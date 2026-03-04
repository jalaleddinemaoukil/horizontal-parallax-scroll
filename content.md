Creating a Smooth Horizontal Parallax Gallery: From DOM to WebGL
Build a smooth horizontal parallax gallery in DOM/CSS/JS, then upgrade it to GPU-powered WebGL (Three.js) with shaders.

By David Faure in Tutorials on February 19, 2026

Demo
Code
Free course recommendation: Master JavaScript animation with GSAP through 34 free video lessons, step-by-step projects, and hands-on demos. Enroll now →

Horizontal scroll galleries with parallax effects have become a staple of modern web design. You’ve probably seen countless tutorials on this effect, and for good reason. It’s visually striking and adds depth to what would otherwise be a simple image carousel.

But here’s the thing: most implementations are purely DOM-based, using CSS transforms and JavaScript to move elements around. While this works fine for simple cases, it can quickly become janky when you’re dealing with multiple images, heavy parallax calculations, and smooth animations all running on the main thread.

You’ve probably come across Camille Mormal’s portfolio a stunning example of how fluid and performant a horizontal gallery can be. What sets it apart? It’s all rendered in WebGL. The smoothness comes from offloading the heavy lifting to the GPU, where these operations thrive.

This got me thinking: what if we could build this effect step by step, starting with a traditional 2D DOM approach and then elevating it to WebGL? Not only would this show the performance benefits, but it would also demystify how these effects actually work under the hood.

In this tutorial, we’ll create a horizontal parallax gallery in two ways:

The 2D approach: using HTML, CSS, and JavaScript with custom smooth scrolling and parallax transforms
The WebGL approach: using Three.js to render everything on the GPU with shader-based parallax for buttery-smooth performance
We’ll keep dependencies minimal (only Three.js for the 3D part) and focus on understanding the core mechanics: how smooth scrolling works, how parallax is calculated, and how to synchronize DOM measurements with WebGL rendering.

Let’s dive in.

The Initial Setup
For this tutorial, we’ll keep things simple and focused. No complex build tools or heavy dependencies just Vite for fast development and Three.js for the WebGL part later on.

Project structure
Here’s what our project looks like:

├── css/
│   └── base.css
├── public/
│   ├── 1.webp
│   ├── 2.webp
│   └── ... (10 images total)
├── src/
│   ├── gallery/
│   │   └── gallery.css
│   ├── utils/
│   │   └── math.ts
│   └── main.ts
├── index.html
└── package.json
Dependencies
{
  "dependencies": {
    "three": "^0.170.0"
  },
  "devDependencies": {
    "typescript": "^5.6.3",
    "vite": "^6.0.3",
    "vite-plugin-glsl": "^1.3.0"
  }
}
That’s it. We’re using:

Vite for bundling and dev server
TypeScript for type safety
Three.js (we’ll add this later for the WebGL version)
vite-plugin-glsl to import shader files (also for later)
HTML Structure
Let’s start with the markup for our gallery. It’s deliberately simple, a wrapper and a container with images:

<body class="demo-1 loading">
  <main>
    <div class="content">
      <div class="gallery__wrapper">
        <div class="gallery__image__container">
          <picture class="gallery__media">
            <img
              src="1.webp"
              alt="Image 1"
              class="gallery__media__image"
              draggable="false"
            />
          </picture>
          <picture class="gallery__media">
            <img src="2.webp" alt="Image 2" class="gallery__media__image" draggable="false" />
          </picture>
          <!-- ... 8 more images -->
        </div>
      </div>
    </div>
  </main>
  <script type="module" src="/src/main.ts"></script>
</body>
A few things to note:

The loading class on the body, we’ll remove this once images are preloaded
draggable="false" prevents the default drag behavior
We’re wrapping images in <picture> tags for flexibility (could add different sources later)
Base CSS
The CSS does most of the heavy lifting for layout. Here’s the essential structure:

.gallery__wrapper {
  height: 100vh;
  width: 100vw;
  overflow: hidden;
  position: relative;
}

.gallery__image__container {
  display: flex;
  gap: 2rem;
  height: 100%;
  width: max-content;
  will-change: transform;
}

.gallery__media {
  position: relative;
  overflow: hidden;
  height: 60vh;
  aspect-ratio: 4 / 3;
  flex-shrink: 0;
}

.gallery__media__image {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  will-change: transform;
}
Key details:

.gallery__wrapper is our viewport, it hides overflow
.gallery__image__container uses display: flex and width: max-content so it expands based on content
will-change: transform hints to the browser that we’ll be transforming these elements (optimization)
Each image wrapper (.gallery__media) has overflow: hidden this is crucial for the parallax effect later. The images inside will move, but the container clips them.
Setting up TypeScript
We’ll create a simple utility file for math functions we’ll need:

// src/utils/math.ts

export function lerp(start: number, end: number, factor: number): number {
  return start * (1 - factor) + end * factor;
}

export function clamp(min: number, max: number, value: number): number {
  return Math.max(min, Math.min(max, value));
}
These two functions are essential:

lerp (linear interpolation): smoothly transitions from one value to another. This is what makes our scroll feel buttery smooth instead of instant.
clamp : restricts a value between a min and max. We’ll use this to prevent scrolling beyond the gallery bounds.
For now, we should have something like this :


Building the 2D Version
Now that we have our HTML and CSS in place, let’s bring this gallery to life with custom horizontal scrolling. We’re going to build everything from scratch to understand exactly what’s happening under the hood.

Custom Smooth Scrolling
Here’s the thing about default browser scroll: it’s either too snappy or not smooth enough, and you can’t easily control its behavior. For a polished gallery experience, we need full control.

Let’s create our App class in main.ts:

// src/main.ts
import "./gallery/gallery.css";
import { clamp, lerp } from "./utils/math";

interface Scroll {
  current: number;
  target: number;
  ease: number;
  limit: number;
}

class App {
  container: HTMLElement | null;
  wrapper: HTMLElement | null;
  scroll: Scroll;

  constructor() {
    this.container = document.querySelector(".gallery__image__container");
    this.wrapper = document.querySelector(".gallery__wrapper");
    this.scroll = {
      current: 0,
      target: 0,
      ease: 0.07,
      limit: 0,
    };

    this.setLimit();
    this.addEventListeners();
    this.render();
  }

  setLimit() {
    if (!this.container || !this.wrapper) return;
    this.scroll.limit = this.container.scrollWidth - this.wrapper.clientWidth;
  }

  onWheel(e: WheelEvent) {
    this.scroll.target += e.deltaY;
  }

  onResize() {
    this.setLimit();
  }

  addEventListeners() {
    window.addEventListener("resize", this.onResize.bind(this), {
      passive: true,
    });
    window.addEventListener("wheel", this.onWheel.bind(this), {
      passive: true,
    });
  }

  render() {
    this.scroll.target = clamp(0, this.scroll.limit, this.scroll.target);

    this.scroll.current = lerp(
      this.scroll.current,
      this.scroll.target,
      this.scroll.ease,
    );

    if (this.container) {
      this.container.style.transform = `translateX(${-this.scroll.current}px)`;
    }

    requestAnimationFrame(this.render.bind(this));
  }
}

new App();
Breaking it down
The Scroll object:

scroll: {
  current: 0,    // Where we are right now
  target: 0,     // Where we want to be
  ease: 0.07,    // How fast we interpolate (lower = smoother but slower)
  limit: 0,      // Maximum scroll distance
}
This is the heart of smooth scrolling. Instead of jumping directly to the scroll position, we gradually move current toward target using lerp().

Setting the scroll limit:

setLimit() {
  this.scroll.limit = this.container.scrollWidth - this.wrapper.clientWidth;
}
This calculates how far we can scroll. scrollWidth is the total width of all images plus gaps, and we subtract the viewport width to get the maximum scroll distance.

Capturing wheel events:

onWheel(e: WheelEvent) {
  this.scroll.target += e.deltaY;
}
When you scroll your mouse wheel, we add the delta to our target position. Notice we’re using deltaY (vertical scroll) but applying it horizontally, this feels more natural than forcing users to scroll sideways.

The render loop:

render() {
  // Clamp target to valid range
  this.scroll.target = clamp(0, this.scroll.limit, this.scroll.target);

  // Smoothly interpolate current toward target
  this.scroll.current = lerp(
    this.scroll.current,
    this.scroll.target,
    this.scroll.ease,
  );

  // Apply transform
  if (this.container) {
    this.container.style.transform = `translateX(${-this.scroll.current}px)`;
  }

  requestAnimationFrame(this.render.bind(this));
}
This runs 60 times per second (or at your monitor’s refresh rate). Each frame:

We clamp the target to prevent scrolling out of bounds
We lerp current toward target by the ease factor (7%)
We apply the transform to move the gallery
Why lerp makes it smooth: Let’s say target is 1000 and current is 0. With ease = 0.07:

Frame 1: current = 0 + (1000 - 0) * 0.07 = 70
Frame 2: current = 70 + (1000 - 70) * 0.07 = 135.1
Frame 3: current = 135.1 + (1000 - 135.1) * 0.07 = 195.6
…and so on
It starts fast and slows down as it approaches the target that’s the smooth, eased motion you feel.

The result
At this point, you have a fully functional horizontal scroll gallery with buttery smooth interpolation. Try scrolling, notice how it glides instead of snapping? That’s the power of lerp.

But we’re not done yet. Let’s add the parallax effect to make it truly special.

Adding the Parallax Effect
Now comes the magic part. Parallax creates depth by making images move at different speeds relative to the scroll. But here’s what most tutorials don’t explain well: parallax requires both CSS and JavaScript working together, and understanding how they connect is essential.

The CSS Setup (Critical!)
Here’s the CSS that makes parallax actually possible:

.gallery__media {
  position: relative;
  overflow: hidden;  /* Clips the oversized image */
  height: 60vh;
  aspect-ratio: 4 / 3;
  flex-shrink: 0;
}

.gallery__media__image {
  position: absolute;
  top: 0;
  left: -12.5%;      /* Start offset to the left */
  width: 125%;       /* Make image BIGGER than container */
  height: 100%;
  object-fit: cover;
  will-change: transform;
}
This is the foundation of the entire effect. Let me explain why each value matters:

Making the image larger:

width: 125%;
The image is 25% wider than its container. This extra space is what allows the image to move. Think of it like this: if the image was exactly 100% width, there’d be no room for it to shift left or right without showing empty space.

By making it 125%, we’re creating a “buffer zone”, 12.5% on each side that the image can slide into.

Offsetting the starting position:

left: -12.5%;
We position the image 12.5% to the left of its container’s left edge. This centers the extra width we created. Without this, the image would start flush left, and we’d only be able to parallax in one direction.

With this offset:

The image has 12.5% hidden on the left
The image has 12.5% hidden on the right
The visible portion is centered
We can now move the image both directions as it scrolls through the viewport
Why these specific values?

The math is simple: if your image is 125% wide (25% extra), you split that extra space in half: 25% / 2 = 12.5%. This creates equal movement range in both directions.

Want a stronger parallax effect? Increase both values proportionally:

width: 150% and left: -25% → more dramatic movement
width: 115% and left: -7.5% → more subtle effect
Just remember: left should always be (width - 100%) / 2 to keep it centered.

The clipping container:

overflow: hidden;
Without this, you’d see the oversized image bleeding out of its container, breaking the entire layout. The parent acts as a “window” that clips the image to the intended size, while the image itself can shift around inside that window.

How CSS and JavaScript Work Together
Here’s the key connection:

CSS sets up the playground → The image is 125% wide with a -12.5% offset, giving it room to move within its clipped container
JavaScript moves the image within that space → As we scroll, we calculate each image’s position and apply transforms
The result → The image shifts horizontally within its fixed-size container, creating the parallax illusion
Without the CSS sizing and positioning, the JavaScript would have nowhere to move the image. Without the JavaScript, the CSS would just show a static, offset image.

The JavaScript Implementation
Now let’s add the parallax effect to our App class:

import './gallery.css';

export class Gallery {
  container: HTMLElement | null;
  wrapper: HTMLElement | null;
  images: NodeListOf<HTMLElement>;

  constructor() {
    this.container = document.querySelector('.gallery__image__container');
    this.wrapper = document.querySelector('.gallery__wrapper');
    this.images = document.querySelectorAll('.gallery__media__image');
  }

  private clamp(v: number, min: number, max: number) {
    return Math.max(min, Math.min(max, v));
  }

  applyParallaxEffect() {
    const vw = window.innerWidth;
    const viewportCenter = vw * 0.5;

    this.images.forEach((image) => {
      const parent = image.parentElement as HTMLElement;
      if (!parent) return;

      const rect = parent.getBoundingClientRect();
      const elementCenter = rect.left + rect.width * 0.5;

      // -1 (far left) .. 0 (center) .. 1 (far right)
      const t = this.clamp((elementCenter - viewportCenter) / viewportCenter, -1, 1);

      // For 125% image width, safe max translate ~= 10% of image width
      const MAX_SHIFT = 10;
      const shift = -t * MAX_SHIFT; // negative for counter-motion depth

      image.style.transform = `translate3d(${shift}%, 0, 0)`;
    });
  }

  render(container: HTMLElement, scroll: number) {
    container.style.transform = `translateX(${scroll < 0.01 ? 0 : -scroll}px)`;
    this.applyParallaxEffect();
  }
}
Breaking Down the Parallax Calculation
Let’s dissect applyParallaxEffect() line by line and see how it connects to our CSS:

applyParallaxEffect() {
  const vw = window.innerWidth;
  const viewportCenter = vw * 0.5;

  // With CSS: image width = 125% => extra = 25% => 12.5% each side.
  // translateX(%) is relative to IMAGE width (125%), so safe max ≈ 12.5/125 = 10%.
  const MAX_SHIFT = 10; // %

  this.images.forEach((image) => {
    const parent = image.parentElement as HTMLElement;
    if (!parent) return;

    const rect = parent.getBoundingClientRect();
    const elementCenter = rect.left + rect.width * 0.5;

    // -1..1 as the element moves across the viewport
    const t = (elementCenter - viewportCenter) / viewportCenter;
    const clamped = Math.max(-1, Math.min(1, t));

    // Counter-motion: element right => image shifts left, etc.
    const shift = -clamped * MAX_SHIFT;

    image.style.transform = `translate3d(${shift}%, 0, 0)`;
  });
}
Step 1: Get the container’s position in viewport

const rect = parent.getBoundingClientRect();
getBoundingClientRect() tells us where each image container sits relative to the viewport, and we use it to compute the element’s center position.

Entering from right: rect.left is large (e.g. > viewport width)
Centered: the element’s center is near the viewport center
Exiting left: rect.left becomes negative
Step 2: Compute distance from the viewport center

const elementCenter = rect.left + rect.width * 0.5;
We use the viewport center as the reference point, so the parallax is strongest near the center and naturally eases toward the edges—without relying on an arbitrary 25% offset.

Step 3: Calculate and apply the parallax

image.style.transform = `translate3d(${shift}%, 0, 0)`;
Here’s where the magic happens:

We compute a normalized position t in the range [-1, 1] based on the element’s center relative to the viewport center
The negative is crucial: as the element moves right, the image shifts left (counter-motion), creating depth
We map t to a bounded percentage shift (e.g. ±10%) so the effect is consistent across viewports
The bound ensures the movement stays inside the CSS buffer, so you won’t reveal empty space
Connecting back to CSS:

Remember, our image is:

125% wide
Positioned at -12.5% left
Has 12.5% hidden on each side
The JavaScript transform shifts the image within that extra 25% of width. Since translateX(%) is relative to the image width (125%), translateX(-10%) moves the image by 10% of its own width, which is 12.5% of the container width—exactly the hidden buffer on each side.

Adjusting the effect intensity:

Want to modify the parallax strength? Change MAX_SHIFT (the maximum translate in %):

MAX_SHIFT = 4 → subtle
MAX_SHIFT = 8 → medium
MAX_SHIFT = 10 → max-safe for width: 125%
If MAX_SHIFT is too high for your CSS sizing, you’ll reveal the texture edges—so either lower MAX_SHIFT or increase the CSS buffer (e.g. wider image).

What’s Actually Happening
Let’s trace through an example:

Imagine a container at left = 800px (right of center):

Compute normalized position: t = (elementCenter - viewportCenter) / viewportCenter
Clamp to [-1, 1]
Apply bounded shift: shift = -t * MAX_SHIFT
Example (1920px wide viewport):
If the element’s center is 320px right of the viewport center:

t = 320 / 960 = 0.333
shift = -0.333 * 10% ≈ -3.33%
Transform: translate3d(-3.33%, 0, 0)
Because the image is 125% wide and positioned at -12.5%, this -3.33% transform shifts it slightly left within its container, creating the illusion that it’s on a different depth plane.

As the container moves left (scroll increases):

As the element moves across the viewport, t moves toward 0 (center) and changes sign past center, so shift smoothly moves in the opposite direction (counter-motion).
This counter-movement at a different rate creates the parallax depth effect
The Complete Picture: CSS + JS
The parallax effect is impossible without both layers:

CSS creates the physical space → 125% width + -12.5% offset = 25% room to move
CSS clips the result → overflow: hidden hides the extra width
JavaScript calculates position → Uses viewport position to determine how much to shift
JavaScript applies transform → Moves the image within the space CSS created
Remove the CSS sizing? No room to move, no effect. Remove the JavaScript? Static image, no parallax. Remove overflow: hidden? Layout breaks completely.

They’re inseparable.

Performance Considerations
You might wonder: “Isn’t calling getBoundingClientRect() and setting styles in a loop every frame expensive?”

Yes and no. Here’s why this approach works well:

What makes it performant:

will-change: transform tells the browser to optimize for transform changes, often promoting the element to its own compositor layer. This means the browser prepares for frequent transform updates.
transform is GPU accelerated and operates on the composite layer. Unlike properties like top, left, or margin, transforms don’t trigger layout recalculations or repaints, only compositing, which is extremely fast.
getBoundingClientRect() is relatively cheap when you’re not causing layout thrashing. Since we’re reading all positions first, then writing transforms, we avoid the read-write-read-write pattern that forces synchronous layout.
When you might need to optimize:

For 10-15 images (like in our gallery), this approach runs buttery smooth at 60fps. However, if you were dealing with 50+ images or targeting low-end mobile devices, you’d want to optimize by:

Only calculating parallax for images currently in or near the viewport
Using Intersection Observer to track which images are visible
Debouncing or throttling calculations on slower devices
But for most practical use cases, this straightforward approach is perfectly performant. The key is understanding what triggers layout/paint/composite and sticking to GPU-accelerated properties.

Et voilà ! You now have a fully functional 2D parallax gallery with smooth scrolling and optimized transforms. The combination of properly sized CSS containers and frame by frame position calculations creates a polished, professional effect.

But here’s the thing: we’re still doing all of this on the CPU, manipulating the DOM every frame, and calling getBoundingClientRect() for each image. What if we could offload this to the GPU entirely and make it even smoother? That’s where WebGL comes in.

Elevating to WebGL
Alright, our 2D version works great. But remember that silky smooth feeling from Camille Mormal’s site? That comes from rendering everything on the GPU with WebGL. Let’s bring Three.js into the picture.

First, let’s install Three.js:

npm install three
Now, let’s start adding WebGL to our main.ts. We’ll need a renderer, a scene, a camera…

import * as THREE from 'three';

class App {
  container: HTMLElement | null;
  wrapper: HTMLElement | null;
  images: NodeListOf<HTMLElement>;
  scroll: Scroll;
  
  // WebGL stuff
  renderer!: THREE.WebGLRenderer;
  scene!: THREE.Scene;
  camera!: THREE.PerspectiveCamera;
  // ... and we'll need meshes for each image
  // ... and materials
  // ... and textures
  // ... and update logic for each mesh

  constructor() {
    // ... all our existing scroll code
    // ... plus all the WebGL initialization
    // ... plus mesh creation for each image
    // ... plus synchronization logic
  }

  applyParallaxEffect() {
    // Keep this for 2D version
  }

  updateWebGLParallax() {
    // New parallax for 3D version
  }

  render() {
    // Handle both 2D and 3D rendering
    // This is getting messy...
  }
}
Wait… our main.ts is about to explode. We’re mixing scroll logic, 2D DOM manipulation, WebGL scene setup, per-mesh updates, and rendering all in one class. This is going to be unreadable.

Refactoring for Sanity
Let’s take a step back. We need to separate concerns before this becomes unmaintainable. Here’s what we actually need:

main.ts → The orchestrator

Handles scroll state (works for both 2D and 3D)
Detects which version to load (DOM vs WebGL)
Coordinates everything
gallery/index.ts → The 2D implementation

Takes a container and scroll value
Handles all DOM based parallax logic
Self contained, reusable
gallery/GL.ts → The WebGL scene manager

Sets up renderer, scene, camera
Creates all the meshes
Handles resize
gallery/GLMedia.ts → Individual WebGL image

One instance per image
Manages its own mesh, material, texture
Handles its own position and parallax updates
This separation means:

Each file has one clear responsibility
We can work on 2D or 3D independently
The code is testable and maintainable
main.ts stays clean and readable
Here’s the refactored main.ts:

// src/main.ts
import { Gallery } from "./gallery";
import { GL } from "./gallery/GL";
import { clamp, lerp } from "./utils/math";

class App {
  container: HTMLElement | null;
  wrapper: HTMLElement | null;
  scroll: Scroll;
  gallery!: Gallery;
  gl!: HTMLElement | null;
  canvas!: GL | null;

  constructor() {
    this.container =
      document.querySelector(".gallery__image__container") ||
      document.querySelector(".gallery__image__container__gl");
    this.wrapper =
      document.querySelector(".gallery__wrapper") ||
      document.querySelector(".gallery__wrapper__gl");
    this.gl = document.getElementById("gl");
    this.scroll = {
      current: 0,
      target: 0,
      ease: 0.07,
      limit: 0,
    };

      this.init();
      this.setLimit();
      this.onResize();
      this.addEventListeners();
      this.render();
    });
  }

  init() {
    this.gallery = new Gallery();
    if (this.gl) {
      this.canvas = new GL();
    }
  }

  setLimit() {
    if (!this.container || !this.wrapper) return;
    this.scroll.limit = this.container.scrollWidth - this.wrapper.clientWidth;
  }

  onWheel(e: WheelEvent) {
    this.scroll.target += e.deltaY;
  }

  onResize() {
    this.setLimit();
    this.canvas?.onResize({
      width: window.innerWidth,
      height: window.innerHeight,
    });
  }

  addEventListeners() {
    window.addEventListener("resize", this.onResize.bind(this));
    window.addEventListener("wheel", this.onWheel.bind(this), {
      passive: true,
    });
  }

  render() {
    this.scroll.target = clamp(0, this.scroll.limit, this.scroll.target);
    this.scroll.current = lerp(
      this.scroll.current,
      this.scroll.target,
      this.scroll.ease,
    );

    this.gallery?.render(this.container!, this.scroll.current);
    this.canvas?.render(this.scroll.current);

    requestAnimationFrame(this.render.bind(this));
  }
}

new App();
Much cleaner! Now main.ts just orchestrates. The 2D logic lives in gallery/index.ts (which we already have), and the WebGL magic will live in GL.ts and GLMedia.ts.

A quick note: If I’d been smarter from the start, I would’ve structured it this way from commit one. But honestly? It’s totally fine to build a prototype, realize it’s getting messy, and then refactor. That’s real development. The important thing is recognizing when to refactor before it becomes technical debt.

Now let’s dive into the WebGL implementation.

Building the WebGL Version
This is where things get interesting. We’re going to render our gallery entirely on the GPU, which means we need to think differently about positioning, sizing, and parallax.

The Camera Setup
One of the trickiest parts of mixing DOM and WebGL is synchronization. We want our WebGL meshes to match the size and position of DOM elements exactly. To do this, we need a camera setup where 1 pixel = 1 Three.js unit.

Here’s the trick in GL.ts:

// src/gallery/GL.ts
import * as THREE from "three";
import { GLMedia } from "./GLMedia";

export class GL {
  renderer: THREE.WebGLRenderer;
  scene: THREE.Scene;
  camera: THREE.PerspectiveCamera;
  group: THREE.Group;
  screen = {
    width: window.innerWidth,
    height: window.innerHeight,
  };
  medias: HTMLElement[];
  allMedias: GLMedia[];

  constructor() {
    this.renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
    document.body.appendChild(this.renderer.domElement);

    this.scene = new THREE.Scene();
    
    // The magic formula for 1px = 1 unit
    const fov = 2 * Math.atan(this.screen.height / 2 / 100) * (180 / Math.PI);

    this.camera = new THREE.PerspectiveCamera(
      fov,
      this.screen.width / this.screen.height,
      0.01,
      1000,
    );
    this.camera.position.set(0, 0, 100);
    
    this.group = new THREE.Group();
    this.medias = Array.from(
      document.querySelectorAll(".gallery__media__image__gl"),
    );
    
    this.createGeometry();
    this.createGallery();
  }

  createGeometry() {
    this.geometry = new THREE.PlaneGeometry(1, 1, 32, 32);
  }

  createGallery() {
    // We'll instantiate our GLMedia instances here soon
    this.scene.add(this.group);
  }

  onResize(viewport = { width: window.innerWidth, height: window.innerHeight }) {
    this.screen = viewport;

    this.camera.aspect = this.screen.width / this.screen.height;
    this.camera.fov = 2 * Math.atan(this.screen.height / 2 / 100) * (180 / Math.PI);
    this.camera.updateProjectionMatrix();

    this.renderer.setSize(this.screen.width, this.screen.height);
    this.renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));

  }

  render(scroll: number) {
    this.renderer.render(this.scene, this.camera);
  }
}
The FOV calculation explained:

const fov = 2 * Math.atan(this.screen.height / 2 / 100) * (180 / Math.PI);
This formula calculates the field of view so that at z = 100 (where our camera sits), the visible height exactly matches window.innerHeight in pixels.

Here’s why it works:

Math.atan(this.screen.height / 2 / 100) gives us the angle from the camera to the top edge of the screen
We multiply by 2 to get the full vertical angle
We convert from radians to degrees with * (180 / Math.PI)
The result: when we position a mesh at z = 0 and set its scale to mesh.scale.set(width, height, 1), those width/height values correspond directly to pixels.

This is essential for syncing DOM and WebGL.

Now let’s build the GLMedia class where the real magic happens.

Syncing DOM with WebGL: The GLMedia Class
Each image in our gallery needs a WebGL representation, a mesh with a texture. But here’s the challenge: we need to make each WebGL plane match its corresponding DOM element’s size and position exactly.

HTML Setup
First, a quick note about the HTML. For the WebGL version (index2.html), we use slightly different class names:

<body class="demo-2 loading" id="gl">
  <!-- ... -->
  <div class="gallery__wrapper__gl">
    <div class="gallery__image__container__gl">
      <picture class="gallery__media__gl">
        <img src="1.webp" class="gallery__media__image__gl" draggable="false" />
      </picture>
      <!-- ... more images -->
    </div>
  </div>
</body>
Notice the id="gl" on the body, this is how main.ts detects which version to load.

In our CSS, we hide these DOM images once WebGL is ready:

body.demo-2 .gallery__media__image__gl {
  opacity: 0; /* Hide DOM images, show only WebGL */
}
We keep the DOM elements in the HTML because:

They let us use getBoundingClientRect() to get positions
They maintain the layout structure (flexbox, gaps, etc.)
They provide fallback if WebGL fails
They’re our “source of truth” for sizing and positioning
Building GLMedia: Step by Step
Let’s start with the basic structure:

// src/gallery/GLMedia.ts
import * as THREE from "three";

interface Props {
  scene: THREE.Group;
  element: HTMLElement;
  viewport: { width: number; height: number };
  camera: THREE.PerspectiveCamera;
  geometry: THREE.PlaneGeometry;
  renderer: THREE.WebGLRenderer;
}

export class GLMedia {
  camera: THREE.PerspectiveCamera;
  element: HTMLElement;
  scene: THREE.Group;
  geometry: THREE.PlaneGeometry;
  renderer: THREE.WebGLRenderer;
  material: THREE.ShaderMaterial;
  texture: THREE.Texture;
  viewport: { width: number; height: number };
  bounds: DOMRect;
  mesh: THREE.Mesh;

  constructor({ scene, element, viewport, camera, geometry, renderer }: Props) {
    this.scene = scene;
    this.element = element;
    this.viewport = viewport;
    this.camera = camera;
    this.geometry = geometry;
    this.renderer = renderer;

    this.bounds = this.element.getBoundingClientRect();
    this.createMesh();
    this.createTexture();
  }
}
Nothing fancy yet. We’re just storing references and getting the DOM element’s bounding box.

Step 1: Creating the Mesh
Let’s create a basic mesh with a shader material:

createMesh() {
  this.material = new THREE.ShaderMaterial({
    uniforms: {
      uTexture: { value: null },
      uResolution: {
        value: new THREE.Vector2(
          this.bounds?.width || 1,
          this.bounds?.height || 1,
        ),
      },
      uImageResolution: { value: new THREE.Vector2(1, 1) },
    },
    vertexShader: `
      varying vec2 vUv;
      void main() {
        vUv = uv;
        gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
      }
    `,
    fragmentShader: `
      precision highp float;
      varying vec2 vUv;
      uniform sampler2D uTexture;

      void main() {
        vec3 col = texture2D(uTexture, vUv).rgb;
        gl_FragColor = vec4(col, 1.0);
      }
    `,
  });

  this.mesh = new THREE.Mesh(this.geometry, this.material);
  this.scene.add(this.mesh);
}
What are these uniforms?

uTexture: The image texture (we’ll load this next)
uResolution: The mesh’s width and height in pixels, this will match the DOM element
uImageResolution: The actual image’s width and height, we need this for object-fit: cover behavior
For now, the shaders are basic: just pass through UVs and render the texture.

Step 2: Loading the Texture
createTexture() {
  this.texture = new THREE.TextureLoader().load(
    this.element.getAttribute("src") as string,
    (text) => {
      const material = this.mesh?.material as THREE.ShaderMaterial;
      if (material?.uniforms?.uImageResolution) {
        material.uniforms.uImageResolution.value.set(
          text.image.width,
          text.image.height,
        );
      }
    },
  );

  this.material.uniforms.uTexture.value = this.texture;
}
Why do we need uImageResolution?

Here’s the problem: our mesh might be 800×1000 pixels, but the image might be 1920×1080. If we just map the texture directly to the mesh, it’ll stretch and distort.

In CSS, we use object-fit: cover to handle this, it crops and centers the image to fill the container without distortion. In WebGL, we need to do this manually in the shader.

The uImageResolution uniform stores the original image dimensions. Later, we’ll write a shader function that calculates the correct UV coordinates to achieve object-fit: cover behavior.

The callback function fires when the texture loads, letting us grab the actual image dimensions from text.image.width and text.image.height.

Step 3: Sizing the Mesh
Now let’s make the mesh match the DOM element’s size:

updateScale() {
  this.bounds = this.element.getBoundingClientRect();
  this.mesh?.scale.set(this.bounds.width, this.bounds.height, 1);
  this.material?.uniforms.uResolution.value.set(
    this.bounds.width,
    this.bounds.height,
  );
}
Remember our camera setup where 1 pixel = 1 unit? This is where it pays off.

We read the DOM element’s dimensions with getBoundingClientRect(), then directly apply them to the mesh’s scale. Because of our camera FOV calculation, a scale of (800, 1000, 1) makes the mesh exactly 800×1000 pixels on screen.

We also update the uResolution uniform so the shader knows the mesh’s dimensions, crucial for the object-fit: cover calculation.

Step 4: Positioning the Mesh
This is where DOM/WebGL sync gets tricky. Three.js uses a center-origin coordinate system, but getBoundingClientRect() gives us top-left positions. We need to convert between them.

But there’s another challenge: horizontal scrolling. Remember our smooth scroll system in main.ts? The scroll.current value that’s being lerped every frame? We need to account for that in our WebGL positioning.

updatePosition(scroll: number) {
  const x =
    this.bounds.left -
    scroll -
    this.viewport.width / 2 +
    this.bounds.width / 2;
  const y =
    -this.bounds.top + this.viewport.height / 2 - this.bounds.height / 2;

  this.mesh.position.set(x, y, 0);
}
Breaking down the math:

X position (horizontal):

const x = this.bounds.left - scroll - this.viewport.width / 2 + this.bounds.width / 2;
Let’s go step by step:

this.bounds.left → DOM element’s left edge in pixels from viewport left (from getBoundingClientRect())
- scroll → This is crucial. The scroll parameter comes from main.ts‘s scroll.current value, our smoothly lerped horizontal scroll position. As the user scrolls, this value increases, and we need to shift all meshes left by that amount to simulate horizontal movement. Without this, the meshes would stay fixed while the DOM scrolls.
- this.viewport.width / 2 → Shift origin from left edge to center. Three.js positions meshes from their center point, but the viewport’s origin (0,0) is at the center of the screen, not the top-left like the DOM.
+ this.bounds.width / 2 → Offset by half the mesh width. Since getBoundingClientRect() gives us the left edge of the element, but Three.js positions from the center, we need to shift right by half the mesh’s width.
Y position (vertical):

const y = -this.bounds.top + this.viewport.height / 2 - this.bounds.height / 2;
-this.bounds.top → Invert Y axis (DOM counts down, WebGL counts up)
+ this.viewport.height / 2 → Shift origin from top to center
- this.bounds.height / 2 → Offset by half the mesh height
This formula ensures the WebGL mesh sits exactly where the DOM element would be.

Putting It Together
Let’s update our GL.ts to instantiate these meshes:

createGallery() {
  this.allMedias = this.medias.map((media) => {
    return new GLMedia({
      scene: this.group,
      element: media,
      viewport: this.screen,
      camera: this.camera,
      geometry: this.geometry,
      renderer: this.renderer,
    });
  });

  this.scene.add(this.group);
}
And add the render method to GLMedia:

render(scroll: number) {
  this.updatePosition(scroll);
}

onResize(viewport: { width: number; height: number }) {
  this.viewport = viewport;
  this.updateScale();
}
What You Should See
At this point, if you open index2.html, you should see… stretched, distorted images positioned correctly but looking terrible.


Why? Because we’re mapping textures 1:1 to the mesh UVs without accounting for aspect ratio differences. A 1920×1080 image stretched onto an 800×1000 mesh looks awful.

This is exactly why we need the object-fit: cover equivalent in our shader.

Achieving object-fit: cover in Shaders
Remember those stretched, distorted images? That’s because we’re mapping the texture directly to the mesh without accounting for aspect ratio differences. In CSS, object-fit: cover handles this automatically. In WebGL, we need to do it ourselves.

The coverUv() Function
Let’s add this function to our fragment shader:

// src/shaders/mediaFragment.glsl
precision highp float;

varying vec2 vUv;

uniform sampler2D uTexture;
uniform vec2 uResolution;
uniform vec2 uImageResolution;

vec2 coverUv(vec2 uv, vec2 resolution, vec2 imageResolution) {
  vec2 ratio = vec2(
    min((resolution.x / resolution.y) / (imageResolution.x / imageResolution.y), 1.0),
    min((resolution.y / resolution.x) / (imageResolution.y / imageResolution.x), 1.0)
  );

  return vec2(
    uv.x * ratio.x + (1.0 - ratio.x) * 0.5,
    uv.y * ratio.y + (1.0 - ratio.y) * 0.5
  );
}

void main() {
  vec2 uv = coverUv(vUv, uResolution, uImageResolution);
  vec3 col = texture2D(uTexture, uv).rgb;
  gl_FragColor = vec4(col, 1.);
}
How coverUv() Works
This function calculates the correct UV coordinates to crop and center the image, just like object-fit: cover.

Step 1: Calculate aspect ratios

vec2 ratio = vec2(
  min((resolution.x / resolution.y) / (imageResolution.x / imageResolution.y), 1.0),
  min((resolution.y / resolution.x) / (imageResolution.y / imageResolution.x), 1.0)
);
We’re comparing two aspect ratios:

resolution → the mesh dimensions (e.g., 800×1000)
imageResolution → the actual image dimensions (e.g., 1920×1080)
For each axis, we calculate how much we need to scale to fit. The min(..., 1.0) ensures we only scale down, never up.

Example:

Mesh: 800×1000 (aspect ratio 0.8)
Image: 1920×1080 (aspect ratio 1.78)
The image is wider than the mesh. To cover the mesh height wise, we need to crop the sides.

ratio.x = min((0.8) / (1.78), 1.0) = min(0.45, 1.0) = 0.45
ratio.y = min((1.25) / (0.56), 1.0) = min(2.23, 1.0) = 1.0
Step 2: Apply the ratio and center

return vec2(
  uv.x * ratio.x + (1.0 - ratio.x) * 0.5,
  uv.y * ratio.y + (1.0 - ratio.y) * 0.5
);
We scale the UVs by the ratio, then offset by half the remaining space to center the crop.

Using our example:

X: uv.x * 0.45 + (1.0 - 0.45) * 0.5 = uv.x * 0.45 + 0.275
Y: uv.y * 1.0 + 0.0 = uv.y
This means:

Horizontally, we’re only using 45% of the image width, centered
Vertically, we’re using the full image height
The result: the image covers the entire mesh without distortion, cropping the sides as needed.

Before and After
Without coverUv(), your images look stretched and wrong. With it, they look perfect, properly cropped and centered just like object-fit: cover in CSS.

Now your WebGL gallery should look identical to the DOM version (minus the parallax, which we’re about to add).


Parallax in WebGL
This is where WebGL really shines. Instead of calculating positions for each image in JavaScript and applying CSS transforms, we can do the parallax effect entirely in the shader on the GPU.

The Concept
Remember how we did parallax in 2D? We:

Made the image 125% wide with a -12.5% offset (CSS)
Calculated each image’s position in the viewport (JS)
Applied a translateX transform (JS → CSS)
In WebGL, we do something similar but in UV space:

We scale the UVs to make the texture smaller (like zooming out), creating space around it
We calculate the image’s position in the viewport (JS)
We shift the UVs horizontally based on that position (shader)
Adding the Parallax Uniform
First, add the uParallax uniform to our material in GLMedia.ts

createMesh() {
  this.material = new THREE.ShaderMaterial({
    uniforms: {
      uTexture: { value: null },
      uResolution: {
        value: new THREE.Vector2(
          this.bounds?.width || 1,
          this.bounds?.height || 1,
        ),
      },
      uImageResolution: { value: new THREE.Vector2(1, 1) },
      uParallax: { value: 0 },  // Add this
    },
    vertexShader: vertex,
    fragmentShader: fragment,
  });

  this.mesh = new THREE.Mesh(this.geometry, this.material);
  this.scene.add(this.mesh);
}
Calculating the Parallax Value
Add this method to GLMedia.ts:

updateParallax(scroll: number) {
  if (!this.bounds) return;

  const { innerWidth } = window;

  const elementLeft = this.bounds.left - scroll;
  const elementRight = elementLeft + this.bounds.width;

  // Only calculate parallax for visible elements
  if (elementRight >= 0 && elementLeft <= innerWidth) {
    // Calculate position relative to viewport center
    // Range from -1 to 1 as element moves through viewport
    const elementCenter = elementLeft + this.bounds.width / 2;
    const viewportCenter = innerWidth / 2;
    const distance = (elementCenter - viewportCenter) / innerWidth;

    // Apply parallax effect
    const parallaxValue = distance * 0.4;
    this.material.uniforms.uParallax.value = parallaxValue;
  }
}
Breaking it down:

Step 1: Calculate element position

const elementLeft = this.bounds.left - scroll;
const elementRight = elementLeft + this.bounds.width;
We get the element’s left and right edges relative to the viewport, accounting for scroll.

Step 2: Check visibility

if (elementRight >= 0 && elementLeft <= innerWidth)
Only calculate parallax for images currently visible on screen. Small optimization.

Step 3: Calculate distance from center

const elementCenter = elementLeft + this.bounds.width / 2;
const viewportCenter = innerWidth / 2;
const distance = (elementCenter - viewportCenter) / innerWidth;
We find the element’s center point, compare it to the viewport center, and normalize the result by dividing by viewport width.

This gives us a value that:

Is negative when the element is left of center
Is 0 when the element is centered
Is positive when the element is right of center
Ranges roughly from -1 to 1
Step 4: Apply intensity

const parallaxValue = distance * 0.4;
We multiply by 0.4 to control the intensity. This is your “parallax strength” knob:

0.2 → subtle effect
0.4 → what we’re using
0.6 → dramatic effect
Update the Render Method
Don’t forget to call updateParallax every frame:

render(scroll: number) {
  this.updateParallax(scroll);
  this.updatePosition(scroll);
}
Applying Parallax in the Shader
Now update the fragment shader to use the parallax value:

// src/shaders/mediaFragment.glsl
precision highp float;

varying vec2 vUv;

uniform sampler2D uTexture;
uniform vec2 uResolution;
uniform vec2 uImageResolution;
uniform float uParallax;

vec2 coverUv(vec2 uv, vec2 resolution, vec2 imageResolution) {
  vec2 ratio = vec2(
    min((resolution.x / resolution.y) / (imageResolution.x / imageResolution.y), 1.0),
    min((resolution.y / resolution.x) / (imageResolution.y / imageResolution.x), 1.0)
  );

  return vec2(
    uv.x * ratio.x + (1.0 - ratio.x) * 0.5,
    uv.y * ratio.y + (1.0 - ratio.y) * 0.5
  );
}

void main() {
  vec2 uv = coverUv(vUv, uResolution, uImageResolution);

  // Apply parallax effect by shifting UVs horizontally
  uv.x += uParallax * 1.0;

  // Scale UVs to create space for parallax movement
  // This is like making the image smaller within its frame
  uv -= 0.5;
  uv *= 0.85;  // Scale down to 85% (leaving 15% space for movement)
  uv += 0.5;

  vec3 col = texture2D(uTexture, uv).rgb;

  gl_FragColor = vec4(col, 1.);
}
What’s happening here:

Step 1: Shift UVs

uv.x += uParallax * 1.0;
We offset the UVs horizontally by the parallax value. As the image moves through the viewport:

Left of center → uParallax is negative → texture shifts left
Right of center → uParallax is positive → texture shifts right
The * 1.0 multiplier controls how much the texture moves. You can adjust this for stronger/weaker effects.

Step 2: Create movement space

uv -= 0.5;
uv *= 0.85;
uv += 0.5;
This is the shader equivalent of our 2D CSS trick (125% width, -12.5% offset).

We:

Center the UVs around (0, 0) by subtracting 0.5
Scale them down to 85% (making the texture effectively smaller)
Shift back to (0.5, 0.5) to re-center
The result: the texture is now 85% of its normal size, with 15% of empty space around it. When we shift the UVs with uv.x += uParallax, we’re moving within that 15% buffer.

Why 0.85?

This value controls how much “room” the texture has to move:

0.9 → less space, more subtle parallax (but risk of showing edges)
0.85 → good balance (what we’re using)
0.8 → more space, stronger parallax possible (but texture appears more zoomed out)
You can adjust this based on your parallax intensity. Just make sure the parallax shift doesn’t exceed the available space, or you’ll see the texture edges.

The Complete Picture
Now you have:

JavaScript calculating each image’s position in the viewport
JavaScript passing a normalized parallax value to the shader
Shader shifting the texture’s UVs based on that value
Shader operating within a scaled-down UV space (the “buffer zone”)
All of this happens on the GPU, every frame, for every pixel. No DOM manipulation, no layout recalculations, just pure, smooth, GPU-accelerated rendering.

Comparing 2D vs WebGL Parallax
2D (DOM):

CSS creates physical space (125% width)
JS calculates position via getBoundingClientRect()
JS applies transform: translateX() to each image
Browser composites on GPU (hopefully)
WebGL:

Shader creates UV space (0.85 scale)
JS calculates position via getBoundingClientRect()
JS passes value to uniform
Shader shifts UVs and samples texture
Everything rendered on GPU
The end result looks similar, but the WebGL version:

Offloads more work to the GPU
Avoids touching the DOM every frame
Handles more images with better performance
Opens the door to more complex shader effects
Tweaking Values with lil-gui (Optional)
Want to experiment with the parallax effect and really understand how each value affects the result? Let’s add lil-gui, a lightweight GUI controller.

npm install lil-gui
Add this to your GL.ts:

import * as THREE from "three";
import { GLMedia } from "./GLMedia";
import GUI from "lil-gui";

export class GL {
  // ... existing properties
  gui: GUI;
  params = {
    parallaxIntensity: 0.4,
    uvScale: 0.85,
    shaderMultiplier: 1.0,
  };

  constructor() {
    // ... existing setup

    this.setupGUI();
  }

  setupGUI() {
    this.gui = new GUI();
    
    this.gui
      .add(this.params, "parallaxIntensity", 0, 1, 0.01)
      .name("Parallax Intensity")
      .onChange((value: number) => {
        this.allMedias.forEach((media) => {
          media.parallaxIntensity = value;
        });
      });

    this.gui
      .add(this.params, "uvScale", 0.7, 1.0, 0.01)
      .name("UV Scale (Buffer)")
      .onChange((value: number) => {
        this.allMedias.forEach((media) => {
          media.material.uniforms.uUvScale.value = value;
        });
      });

    this.gui
      .add(this.params, "shaderMultiplier", 0, 2, 0.1)
      .name("Shader Multiplier")
      .onChange((value: number) => {
        this.allMedias.forEach((media) => {
          media.material.uniforms.uShaderMultiplier.value = value;
        });
      });
  }
}
Update GLMedia.ts to use these values:

export class GLMedia {
  parallaxIntensity = 0.4;

  createMesh() {
    this.material = new THREE.ShaderMaterial({
      uniforms: {
        uTexture: { value: null },
        uResolution: { value: new THREE.Vector2(/* ... */) },
        uImageResolution: { value: new THREE.Vector2(1, 1) },
        uParallax: { value: 0 },
        uUvScale: { value: 0.85 },
        uShaderMultiplier: { value: 1.0 },
      },
      // ... shaders
    });
  }

  updateParallax(scroll: number) {
    // ... existing code
    const parallaxValue = distance * this.parallaxIntensity;
    this.material.uniforms.uParallax.value = parallaxValue;
  }
}
And update the fragment shader to use the uniforms:

uniform float uParallax;
uniform float uUvScale;
uniform float uShaderMultiplier;

void main() {
  vec2 uv = coverUv(vUv, uResolution, uImageResolution);

  uv.x += uParallax * uShaderMultiplier;

  uv -= 0.5;
  uv *= uUvScale;
  uv += 0.5;

  vec3 col = texture2D(uTexture, uv).rgb;
  gl_FragColor = vec4(col, 1.);
}
Now you can tweak values in real-time and see exactly how they affect the parallax:

Parallax Intensity: Controls how much JS calculates (the distance multiplier)
UV Scale: Controls the buffer space (lower = more space for movement)
Shader Multiplier: Controls how much the shader shifts UVs
Play around! You’ll quickly understand the relationship between these values and the visual effect.

(Optional) Exercise: Adding Touch Support
Want to take this further? Here’s a challenge: add touch/drag support so users can scroll the gallery by swiping.

Hints:

Listen for touchstart, touchmove, and touchend events
Track the touch delta and add it to scroll.target
Don’t forget to handle pointerdown, pointermove, and pointerup for both mouse and touch
Add momentum/inertia for a more natural feel
This will make the gallery work beautifully on mobile devices and add an extra layer of interactivity.

Conclusion
When I started building this gallery, I wanted to go beyond the typical “here’s how to do horizontal scroll with parallax” tutorial. There are already plenty of those out there, and they mostly focus on the DOM-based approach.

What inspired me, and what I hope this tutorial conveys, is how performance and technique can fundamentally change the feel of a website. The difference between a janky 30fps parallax and a buttery smooth 60fps GPU accelerated one isn’t just technical, it’s visceral. Users feel it immediately, even if they can’t articulate why.

Camille Mormal’s portfolio is a perfect example of this. The fluidity isn’t accidental, it’s the result of rendering everything in WebGL and letting the GPU do what it does best. That’s the approach I wanted to break down here: not just “how to make it work,” but “how to make it feel right.”

We’ve covered:

Custom smooth scrolling with lerp and easing
2D parallax with CSS sizing and JavaScript transforms
The critical connection between CSS buffer space and JavaScript movement
WebGL fundamentals with DOM/3D synchronization
Shader-based parallax that runs entirely on the GPU
Performance considerations and when to optimize
The 2D version is great for simple galleries. The WebGL version is for when you want to push further when you want that extra fluidity, when you want to handle dozens of images without breaking a sweat, or when you want to add more complex shader effects down the line.

I hope this deep dive into both approaches gives you not just copy paste code, but a real understanding of what’s happening under the hood. That’s what matters. That’s what lets you adapt these techniques to your own projects and push them even further.

Now go build something beautiful.