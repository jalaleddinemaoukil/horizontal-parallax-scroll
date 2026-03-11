# Horizontal Parallax Scroll

A smooth horizontal parallax scrolling gallery built with **Vue 3** and **GSAP**. Scroll vertically with your mouse wheel to move through a full-screen image gallery horizontally, while each image subtly shifts with a depth parallax effect.

> Created by [jalaleddinemaoukil](https://github.com/jalaleddinemaoukil)

---

## Preview

| Scroll direction       | Effect                                          |
| ---------------------- | ----------------------------------------------- |
| Mouse wheel (vertical) | Drives horizontal gallery movement              |
| Per-image              | Parallax depth shift based on viewport position |

---

## Features

- **Horizontal scroll** — converts vertical wheel input into smooth horizontal track movement
- **Per-image parallax** — each image shifts proportionally to its distance from the viewport center
- **GSAP `quickSetter`** — performant animation without re-rendering Vue state on every frame
- **Lerp easing** — fluid, inertia-like motion (`ease: 0.07`)
- **Fully responsive** — recalculates scroll limits on window resize
- **Clean teardown** — removes GSAP ticker and event listeners on component unmount

---

## Tech Stack

| Tool                        | Role                           |
| --------------------------- | ------------------------------ |
| [Vue 3](https://vuejs.org/) | UI framework (Composition API) |
| [GSAP](https://gsap.com/)   | Animation engine               |
| [Vite](https://vitejs.dev/) | Dev server & bundler           |
| [pnpm](https://pnpm.io/)    | Package manager                |

---

## Getting Started

### Prerequisites

- [Node.js](https://nodejs.org/) v18+
- [pnpm](https://pnpm.io/) (`npm i -g pnpm`)

### Install

```sh
pnpm install
```

### Development

```sh
pnpm dev
```

### Production Build

```sh
pnpm build
```

### Preview Build

```sh
pnpm preview
```

### Lint

```sh
pnpm lint
```

---

## Project Structure

```
src/
├── App.vue                    # Root component
├── main.js                    # App entry point
├── components/
│   └── GalleryParallax.vue    # Core gallery — scroll logic & parallax
└── styles/
    └── global.css             # Base reset & body styles
public/
└── images/                    # Gallery images
```

---

## How It Works

1. **Wheel events** accumulate into `scroll.target`
2. On every GSAP ticker tick, `scroll.current` lerps toward `scroll.target`
3. `gsap.quickSetter` applies the interpolated value directly to the track's `x` transform
4. For each visible image, its center offset from the viewport center is computed and mapped to a horizontal `x%` shift, creating the parallax effect

---

## License

MIT © [jalaleddinemaoukil](https://github.com/jalaleddinemaoukil)
