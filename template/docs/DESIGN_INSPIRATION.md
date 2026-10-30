# {{PROJECT_NAME}} design inspiration and toolkit

A curated set of libraries and tools to use as design material and inspiration, in six parts:

- **Part A, expressive effects:** 3D scenes, Apple-style liquid glass, moving shader gradients and liquid-metal logos.
- **Part B, UI building blocks:** Motion for interaction and layout animation, Bklit UI for charts, Kokonut UI for ready-made animated components.
- **Part C, outside tools:** Manus for quick AI-built prototype sites (not part of this stack).
- **Part D, scroll and animation libraries:** Lenis smooth scroll, GSAP timelines and scroll animation, Vanta 3D backgrounds, React Bits copy-paste animations.
- **Part E, premium interaction patterns:** magnetic buttons, controlled parallax, staggered scroll reveal, Easter eggs, page transitions, cursor with context.
- **Part F, visual reference archives:** Letterform Archive, The Met Open Access, the Rijksmuseum, archives.design and NYPL Digital Collections.

Each entry says what it is, its licence, how it fits this stack (Vite + React 19 + TypeScript + Tailwind v4 on Cloudflare Workers), a minimal example and the controls worth playing with.

The `product-designer` owns this file. Add a repo here only after checking its licence and that it still works with the stack; remove one that stops being maintained.

## Part A: expressive effects

### When to use these (read first)

`DESIGN_SYSTEM.md` says the app is "a serious web app, not a marketing site" (§1.1) and that "motion explains a change; it never decorates" (§15). These effects are decoration, so they are **opt-in exceptions**:

- **Where they belong:** a public landing or marketing page, a sign-in or empty-state backdrop, a brand moment (logo, launch page, onboarding splash the owner asked for), a showcase or demo. **Not** in the app shell, lists, forms, dialogs or anything people use every day.
- **Who decides:** the owner approves each surface. Record the decision in `DESIGN_SYSTEM.md` §22 ("Liquid-metal logo on the sign-in page only") and in the roadmap.
- **One hero effect per screen.** Don't stack a 3D scene, glass panels and a moving gradient on the same view.
- **New dependencies need the owner's OK** (`CLAUDE.md`). Pin the exact version: several of these ship breaking changes in patch releases.

### The repos

| # | Repo | Use it for | Licence | Package for this stack |
| --- | --- | --- | --- | --- |
| 1 | [pmndrs/react-three-fiber](https://github.com/pmndrs/react-three-fiber) | Full 3D scenes in React (three.js as components) | MIT | `@react-three/fiber` + `three` |
| 2 | [dashersw/liquid-glass-js](https://github.com/dashersw/liquid-glass-js) ([demo](https://dashersw.github.io/liquid-glass-js/)) | Apple Liquid Glass refraction with live controls | MIT | Not on npm (the npm `liquid-glass-js` is a different project): vendor the files |
| 3 | [ruucm/shadergradient](https://github.com/ruucm/shadergradient) ([customiser](https://shadergradient.co/customize)) | Moving 3D gradients: shape, colour, motion | MIT | `@shadergradient/react` (built on R3F) |
| 4 | [paper-design/liquid-logo](https://github.com/paper-design/liquid-logo) ([try it](https://liquid.paper.design)) | Chrome liquid-metal look on a logo or shape | App: **PolyForm Shield** (not open source) | Use `@paper-design/shaders-react` (`LiquidMetal`, Apache-2.0) instead |

---

#### 1. React Three Fiber: 3D in React

[github.com/pmndrs/react-three-fiber](https://github.com/pmndrs/react-three-fiber) · docs: [r3f.docs.pmnd.rs](https://r3f.docs.pmnd.rs)

A React renderer for three.js: the whole 3D library as JSX (`<mesh>`, `<ambientLight>`), with hooks such as `useFrame` for animation. The companion repos are where most of the inspiration lives: [pmndrs/drei](https://github.com/pmndrs/drei) (ready helpers: text, environment maps, controls, glass-like `MeshTransmissionMaterial`) and [pmndrs/react-postprocessing](https://github.com/pmndrs/react-postprocessing) (bloom, depth of field).

- **Fit:** v9 targets React 19 (check the `react` peer range in its `package.json` against ours before upgrading either). Heavy: three.js adds several hundred KB, so always lazy-load the scene.
- **Install (owner OK first):** `pnpm add @react-three/fiber three` and `pnpm add -D @types/three`.

```tsx
// src/components/effects/hero-scene.tsx (load with React.lazy, see "Shared rules")
import { Canvas, useFrame } from "@react-three/fiber"
import { useRef } from "react"
import type { Mesh } from "three"
import { tokenColor, useReducedMotion } from "@/components/effects/effect-utils"

function Knot({ color, still }: { color: string; still: boolean }) {
  const ref = useRef<Mesh>(null)
  useFrame((_, delta) => {
    if (ref.current && !still) ref.current.rotation.y += delta * 0.3
  })
  return (
    <mesh ref={ref}>
      <torusKnotGeometry args={[1, 0.3, 160, 24]} />
      <meshStandardMaterial color={color} roughness={0.25} metalness={0.6} />
    </mesh>
  )
}

export default function HeroScene() {
  const still = useReducedMotion()
  return (
    <Canvas aria-hidden dpr={[1, 1.75]} frameloop={still ? "demand" : "always"} camera={{ position: [0, 0, 4] }}>
      <ambientLight intensity={0.6} />
      <directionalLight position={[3, 3, 3]} />
      <Knot color={tokenColor("--primary")} still={still} />
    </Canvas>
  )
}
```

**Play with:** geometry and materials, `useFrame` motion, `drei` environments and text, post-processing. `frameloop="demand"` renders only when something changes (good for still scenes and battery).

---

#### 2. Liquid Glass JS: Apple's liquid glass effect

[github.com/dashersw/liquid-glass-js](https://github.com/dashersw/liquid-glass-js) · live demo with the full control panel: [dashersw.github.io/liquid-glass-js](https://dashersw.github.io/liquid-glass-js/)

WebGL glass with real-time refraction, blur and masking. `Container` and `Button` classes, three shapes (`rounded`, `circle`, `pill`), nested glass (children refract their parent). The demo's panel exposes the whole look: edge, rim and base intensity and distance, corner boost, ripple, blur radius, tint opacity and centre warp. Use it to design the look, then copy the values.

- **Fit (important):**
  - Plain JavaScript classes, not a React component, and not published to npm. Vendor `container.js`, `button.js` and `glass.css` into `src/components/effects/liquid-glass/` with the MIT licence file, and wrap them in a small React component that creates the instance in `useEffect` and removes it on unmount.
  - It snapshots the page behind the glass with `html2canvas`. Install that from npm (`pnpm add html2canvas`, owner OK first) rather than the CDN script in its README: our CSP only allows scripts from `'self'` (`SECURITY.md`).
  - Snapshotting is expensive and can lag behind moving content. Use glass over a mostly static background (an image, a gradient), not over live lists.
  - Its buttons render text inside the effect: check contrast at the worst point of the background, and that the element is a real `<button>` with an accessible name and keyboard focus. If it isn't, wrap the glass as decoration behind a real shadcn `Button`.
  - Pills are fine for a decorative glass shape, but our buttons keep 10 px corners (`DESIGN_SYSTEM.md` §9): use `type: "rounded"` with `borderRadius: 10` for anything that acts as a button.
- **Lighter alternative** when you only need the look in React: [rdev/liquid-glass-react](https://github.com/rdev/liquid-glass-react) (MIT, SVG filters; the full refraction works best in Chromium, so check Safari and Firefox fall back to a plain frosted panel).

```ts
// Inside a React wrapper's useEffect
const panel = new Container({ type: "rounded", borderRadius: 16, tintOpacity: 0.25 })
hostRef.current?.appendChild(panel.element)
return () => panel.element.remove()
```

**Play with:** tint opacity, edge/rim/base intensity and distance, blur radius, warp, nesting a button inside a container.

---

#### 3. Shader Gradient: moving gradients in React

[github.com/ruucm/shadergradient](https://github.com/ruucm/shadergradient) · visual customiser: [shadergradient.co/customize](https://shadergradient.co/customize)

Animated 3D gradients, customisable by **shape** (`type`: `plane`, `sphere`, `waterPlane`), **colour** (`color1`, `color2`, `color3`), **motion** (`animate`, `uSpeed`, `uStrength`, `uFrequency`, `uAmplitude`), plus camera, lighting, grain and reflection. Design it in the customiser, copy the URL, and paste it as `urlString` with `control="query"`, or copy the values into props.

- **Fit:** built on React Three Fiber (same cost; lazy-load it). Works with Vite and React 19 when the installed `@react-three/fiber` major matches.
- **Install (owner OK first):** `pnpm add @shadergradient/react @react-three/fiber three three-stdlib camera-controls` and `pnpm add -D @types/three`.
- `lightType="env"` loads environment maps: check the network tab, self-host anything it fetches, and keep `connect-src`/`img-src` in the CSP honest. `lightType="3d"` needs no extra files.

```tsx
// src/components/effects/gradient-backdrop.tsx (lazy-loaded)
import { ShaderGradient, ShaderGradientCanvas } from "@shadergradient/react"
import { tokenColor, useReducedMotion } from "@/components/effects/effect-utils"

export default function GradientBackdrop() {
  const still = useReducedMotion()
  return (
    <ShaderGradientCanvas aria-hidden pixelDensity={1} style={{ position: "absolute", inset: 0 }}>
      <ShaderGradient
        type="waterPlane"
        color1={tokenColor("--primary")}
        color2="#7dd3fc"
        color3="#f5f5f4"
        animate={still ? "off" : "on"}
        uSpeed={0.2}
        uStrength={2}
        grain="on"
        lightType="3d"
      />
    </ShaderGradientCanvas>
  )
}
```

**Play with:** `type`, the three colours, `uSpeed` (keep it slow: 0.1 to 0.3 reads as calm), `uStrength`/`uFrequency`, camera angle, `grain`.

---

#### 4. Liquid Logo: chrome liquid metal on anything

[github.com/paper-design/liquid-logo](https://github.com/paper-design/liquid-logo) · try it in the browser: [liquid.paper.design](https://liquid.paper.design)

Upload a logo (SVG or PNG with a transparent background) and see it as moving liquid chrome. Great for exploring the look and exporting assets.

- **Licence (important):** the repo is a Next.js app under the **PolyForm Shield** licence, not an open-source library. Don't copy its code into a product. The effect itself comes from **Paper Shaders**, which is Apache-2.0 and allowed in commercial products without visible attribution: [paper-design/shaders](https://github.com/paper-design/shaders) · [`LiquidMetal` docs and playground](https://shaders.paper.design/liquid-metal).
- **Fit:** zero-dependency canvas shaders, light compared with three.js. Versions are `0.0.x` with breaking changes: pin the exact version.
- **Install (owner OK first):** `pnpm add @paper-design/shaders-react --save-exact`.

```tsx
// src/components/effects/metal-logo.tsx (lazy-loaded)
import { LiquidMetal } from "@paper-design/shaders-react"
import { useReducedMotion } from "@/components/effects/effect-utils"

export default function MetalLogo() {
  const still = useReducedMotion()
  return (
    <LiquidMetal
      aria-hidden
      image="/brand/logo-mark.svg"
      colorBack="#00000000"
      colorTint="#ffffff"
      repetition={2}
      softness={0.1}
      distortion={0.07}
      contour={0.4}
      speed={still ? 0 : 1}
      fit="contain"
      style={{ width: 160, height: 160 }}
    />
  )
}
```

Keep the real logo (with alt text or an accessible name) in the DOM; the metal canvas is decoration on top or beside it. `shape` also offers `circle`, `daisy`, `diamond` and `metaballs` with no image. The same package has more inspiration: `MeshGradient`, `DotOrbit`, grain and dithering effects.

**Play with:** `repetition`, `softness`, `shiftRed`/`shiftBlue` (colour fringing), `distortion`, `contour`, `angle`, `speed`.

---

### Shared rules for every effect

1. **Lazy-load.** Each effect is its own `React.lazy` chunk inside `<Suspense>` with a static fallback (the brand gradient or image), under the existing `ChunkBoundary`. The page must be usable before it loads.
2. **Decorative only.** Canvases get `aria-hidden`; every meaningful thing (text, logo, button) stays in normal HTML.
3. **Reduced motion.** The global CSS safety net in `src/index.css` doesn't stop WebGL loops, so each effect reads `useReducedMotion()` and shows a still frame (`frameloop="demand"`, `animate="off"`, `speed={0}`).
4. **Contrast over moving backgrounds.** Text over an effect needs 4.5:1 at its *worst* frame and in both themes: add a scrim (`bg-background/70` plus a light blur) behind text, and measure.
5. **Theme tokens.** WebGL can't read CSS variables or `oklch()`, so take colours from the tokens with `tokenColor()` (below) and read them again when the theme changes. Hard-coded colours are allowed only for the shader's own palette and noted in a comment.
6. **Performance.** Cap pixel density (`dpr={[1, 1.75]}`, `pixelDensity={1}`), pause when off screen or when the tab is hidden, and check a mid-range phone: no dropped scroll, no hot device. Fall back to the static image when WebGL isn't available.
7. **Security.** Install from npm, never from a CDN `<script>`; no `unsafe-eval`; self-host textures and environment maps. Test the CSP in the console.
8. **Licences.** Record each library and any vendored file in `ASSETS_LICENSES.md` (item 20 of `PRE_LAUNCH_COMPLIANCE_CHECKLIST.md`), and keep the licence files that came with vendored code.

```ts
// src/components/effects/effect-utils.ts
import { useEffect, useState } from "react"

const REDUCED = "(prefers-reduced-motion: reduce)"

export function useReducedMotion(): boolean {
  const [reduced, setReduced] = useState(() => window.matchMedia(REDUCED).matches)
  useEffect(() => {
    const query = window.matchMedia(REDUCED)
    const onChange = () => setReduced(query.matches)
    query.addEventListener("change", onChange)
    return () => query.removeEventListener("change", onChange)
  }, [])
  return reduced
}

/** Resolves a CSS colour token (any format, including oklch) to #rrggbb for WebGL. */
export function tokenColor(name: string, fallback = "#000000"): string {
  const value = getComputedStyle(document.documentElement).getPropertyValue(name).trim()
  const ctx = document.createElement("canvas").getContext("2d", { willReadFrequently: true })
  if (!ctx || !value) return fallback
  ctx.fillStyle = value
  ctx.fillRect(0, 0, 1, 1)
  const [r = 0, g = 0, b = 0] = ctx.getImageData(0, 0, 1, 1).data
  return `#${[r, g, b].map((n) => n.toString(16).padStart(2, "0")).join("")}`
}
```

### Review checklist for an effect

- [ ] The owner approved this surface; the decision is in `DESIGN_SYSTEM.md` §22.
- [ ] One effect on the screen; it is not in the app shell, a list, a form or a dialog.
- [ ] Lazy-loaded with a static fallback; the page works with WebGL off.
- [ ] Canvas `aria-hidden`; text, logo and buttons are real HTML with names and keyboard focus.
- [ ] Reduced motion shows a still frame.
- [ ] Text over the effect is 4.5:1 at the worst frame in light and dark.
- [ ] Smooth on a mid-range phone; pauses off screen; pixel density capped.
- [ ] No CDN scripts; CSP passes; assets self-hosted.
- [ ] Exact version pinned; licence recorded in `ASSETS_LICENSES.md`.

---

## Part B: UI building blocks

These are for the everyday app as well, so they follow `DESIGN_SYSTEM.md` fully: theme tokens, 10 px button corners (never pills), `font-medium`, the §15 motion durations (nothing over 300 ms) and WCAG 2.2 AA. When a component you copy disagrees with the design system, change the component.

| # | Library | Use it for | Licence | How it installs |
| --- | --- | --- | --- | --- |
| 5 | [Motion](https://motion.dev) ([github.com/motiondivision/motion](https://github.com/motiondivision/motion)) | Drag, gestures, layout and shared-element transitions, exit animations | MIT | `pnpm add motion` |
| 6 | [Bklit UI](https://bklit.com) ([github.com/bklit/bklit-ui](https://github.com/bklit/bklit-ui)) | Charts and data visuals | MIT | shadcn registry `@bklit` |
| 7 | [Kokonut UI](https://kokonutui.com) ([github.com/kokonut-labs/kokonutui](https://github.com/kokonut-labs/kokonutui)) | 100+ ready-made components with animations built in | MIT (a paid Pro set also exists) | shadcn registry `@kokonutui` |

### 5. Motion: drag, gestures and layout transitions

[motion.dev](https://motion.dev) · [github.com/motiondivision/motion](https://github.com/motiondivision/motion)

The animation library formerly called Framer Motion (import from `motion/react`). Springs, gestures (`whileHover`, `whileTap`, `drag`), `layout` and `layoutId` for smooth layout and shared-element transitions, and `AnimatePresence` for exit animations. Supports React 18 and 19.

- **When to use it, not CSS:** drag and swipe (reorderable lists, a swipe-to-dismiss sheet), layout transitions (items reflowing when a filter changes, a tab indicator sliding between tabs), shared-element moves (a card growing into a detail view) and exit animations. Plain hover, press, fades and the `ENTER` tokens stay in CSS: they're already in `src/index.css`.
- **Hover:** the design system says hover changes colour, not position. Use Motion for hover only on expressive surfaces from Part A, not in app chrome.
- **Rules:**
  - Wrap the app once in `<MotionConfig reducedMotion="user">` so every Motion animation respects reduced motion (the CSS safety net doesn't reach JavaScript animations).
  - Keep §15 timing: `transition={{ duration: 0.3, ease: [0.32, 0.72, 0, 1] }}` for things moving into place, or a spring tuned to settle in about 300 ms (`{ type: "spring", bounce: 0, duration: 0.3 }`).
  - Animate only `transform` and `opacity` (`layout` animations use transforms, so they're fine).
  - Keep the bundle small with `LazyMotion` + `m` components and `domAnimation` features where possible.
  - Drag must have a keyboard alternative (move up/down buttons or arrow keys) to meet WCAG 2.5.7.

```tsx
import { motion } from "motion/react"

// A tab indicator that slides between tabs (layoutId shares it between renders)
export function TabIndicator({ active }: { active: boolean }) {
  return active ? (
    <motion.span
      layoutId="tab-indicator"
      className="absolute inset-x-2 -bottom-px h-0.5 rounded-full bg-primary"
      transition={{ type: "spring", bounce: 0, duration: 0.3 }}
    />
  ) : null
}
// Wrap the tab list in <LayoutGroup> and render <TabIndicator active={tab === current} /> inside each tab.
```

### 6. Bklit UI: charts and data visuals

[bklit.com](https://bklit.com) · [github.com/bklit/bklit-ui](https://github.com/bklit/bklit-ui)

Open-source, composable chart components distributed as a shadcn registry: area, bar, line, live line, candlestick, pie, ring, radial, radar, gauge, funnel, heatmap, scatter, sankey, sunburst, choropleth, sparkline and more, plus utilities (tooltip, legend, brush, reference area, trend line, markers). Its **Studio** lets you tweak style and animation in the browser and copy the React code.

- **Install (owner OK first, since it adds dependencies):** add `"@bklit": "https://ui.bklit.com/r/{name}.json"` to `registries` in `components.json`, then `pnpm dlx shadcn@latest add @bklit/area-chart`. Read the diff: the CLI may add packages and files.
- **Fit:** the starter already ships the shadcn `chart` component (Recharts) and the `--chart-1` … `--chart-5` tokens. **Pick one chart library per project** and write the choice in `DESIGN_SYSTEM.md` §22. Choose Bklit when you need the richer types (sankey, funnel, heatmap, live line) or its look; keep Recharts for simple bars and lines.
- **Rules:** colours come from the `--chart-*` tokens in both themes; animation stays short and respects reduced motion; every chart has a text alternative (a title, a one-line summary of the takeaway, and a data table or `aria-describedby` for the numbers); never colour alone to separate series (WCAG 1.4.1); tooltips work with keyboard focus, not only hover.

### 7. Kokonut UI: 100+ animated components

[kokonutui.com](https://kokonutui.com) · [github.com/kokonut-labs/kokonutui](https://github.com/kokonut-labs/kokonutui)

If "coconut UI" means this one: it's spelled **Kokonut UI**. More than 100 open-source React components built with Tailwind CSS, Motion and shadcn/ui, grouped as AI (tool approval, diff review, model selector), buttons (slide to confirm, hold button, undo), cards, inputs (wheel picker, signature pad, amount pad), navigation (command palette, smooth drawer, swipe tabs), media, text effects and backgrounds. The shadcn CLI copies the source into your repo, so you own and can edit it.

- **Install one component at a time (owner OK first):** add `"@kokonutui": "https://kokonutui.com/r/{name}.json"` to `registries` in `components.json`, then `pnpm dlx shadcn@latest add @kokonutui/slide-to-confirm`. It pulls in `motion` (item 5).
- **Fit (important):**
  - Keep copied components in `src/components/kokonutui/`, **never** in `src/components/ui/` (that folder stays the stock shadcn set).
  - Our shadcn style is Base UI (`base-nova`), not Radix. If a component brings Radix primitives, swap them for the Base UI component we already have, or skip that component.
  - Restyle every copied component to our tokens: no hard-coded colours, 10 px button corners instead of pills, `font-medium`, our motion durations. Run `ux-reviewer` and `qa-tester` on it like any new UI.
  - Many components are showpieces (backgrounds, text effects): they fall under Part A's rules (owner opt-in, marketing and brand surfaces). The practical ones (undo button, command palette, wheel picker, amount pad) can go in the app once restyled.
  - The free set is MIT. **Kokonut UI Pro** is a separate paid licence: check before copying anything from it, and record it for checklist item 20.

---

## Part C: outside tools (not part of this stack)

### 8. Manus: AI-built sites with a backend

[manus.im](https://manus.im) · website builder: [manus.im/features/webapp](https://manus.im/features/webapp)

A hosted AI agent (closed source; its site says Manus is now part of Meta) that builds a full-stack web app from a plain-language description: built-in database, sign-in, Stripe payments, analytics, hosting and custom domains, with an option to download the code.

- **What it is good for:** a fast clickable prototype to test an idea with real people, or a small standalone site (an event page, a waitlist) that doesn't need this stack.
- **Why it isn't in the starter's build:** the starter's backend is a Hono API on Cloudflare Workers that our own agents build, test and review. A Manus app runs on Manus's hosting with its own code and database. Its exported code won't follow our structure, tests or security rules, so treat it as a reference to rebuild from, not code to merge.
- **Before using it for anything real:** it becomes a data processor for any personal data the site collects. List it in `PRE_LAUNCH_COMPLIANCE_CHECKLIST.md` items 8 and 9, check its terms, data location and pricing, and don't rely on marketing claims such as "flawless" (item 16): test the result like any other build.
---

## Part D: scroll and animation libraries

Libraries for scroll-driven storytelling, text and image animation, animated backgrounds and copy-paste effects. Smooth scroll, scroll-scrubbed timelines and animated backgrounds are **expressive effects**: they follow Part A's rules (owner opt-in per surface, public and brand pages, lazy-loaded, reduced motion, contrast, performance) and its review checklist. Small, practical pieces (a text reveal on a heading, a counter) can go elsewhere once restyled to the design system.

| # | Library | Use it for | Licence | How it installs |
| --- | --- | --- | --- | --- |
| 9 | [Lenis](https://lenis.darkroom.engineering) ([github.com/darkroomengineering/lenis](https://github.com/darkroomengineering/lenis)) | Smooth, inertia-based page scroll | MIT | `pnpm add lenis` |
| 10 | [GSAP](https://gsap.com) ([github.com/greensock/GSAP](https://github.com/greensock/GSAP)) | Timelines, scroll-driven animation (ScrollTrigger), text splitting, SVG morphing | GSAP Standard "no charge" licence (free, all plugins included; not open source) | `pnpm add gsap @gsap/react` |
| 11 | [Vanta](https://www.vantajs.com) ([github.com/tengbao/vanta](https://github.com/tengbao/vanta)) | Animated 3D hero backgrounds: waves, clouds, birds, fog, net, globe | MIT | `pnpm add vanta` + a pinned three.js |
| 12 | [React Bits](https://reactbits.dev) ([github.com/DavidHDev/react-bits](https://github.com/DavidHDev/react-bits)) | 200+ copy-paste animated React components, text effects and backgrounds | MIT + Commons Clause (free in apps and sites; don't resell the components) | shadcn registry `@react-bits` |

**One animation engine per surface.** Motion (item 5) stays the default for app UI: drag, layout and exit animations. Use GSAP for scroll-driven storytelling and complex timelines on public pages. Don't drive the same element with both.

### 9. Lenis: smooth scroll

[lenis.darkroom.engineering](https://lenis.darkroom.engineering) · [github.com/darkroomengineering/lenis](https://github.com/darkroomengineering/lenis)

A small library that replaces the browser's stepped wheel scroll with a smooth, eased one, which a lot of award-winning sites use. It keeps native scrolling underneath (scroll position, keyboard, find in page still work), syncs with GSAP ScrollTrigger and WebGL scenes, and has a React wrapper (`lenis/react`).

- **Where it fits:** long public or marketing pages with scroll storytelling. **Not in the app shell:** our app has a sticky top bar, sheets, dialogs, nested scroll areas and route focus handling, and people expect native scroll in a tool they use every day.
- **Rules:**
  - Turn it off under reduced motion (render without `ReactLenis`).
  - Import `lenis/dist/lenis.css`.
  - Add `data-lenis-prevent` to elements that scroll on their own (dialogs, menus, code blocks, chat panes).
  - Turn on `anchors` so in-page links still work, and check `scroll-padding-top` still keeps focused elements clear of the sticky bar (WCAG 2.4.11).
  - Keep it subtle: the default `lerp` (about 0.1) is enough. Heavy smoothing feels laggy and can make some people motion-sick.

```tsx
// src/components/effects/smooth-scroll.tsx (wrap a public page, not the app)
import "lenis/dist/lenis.css"
import { ReactLenis } from "lenis/react"
import type { ReactNode } from "react"
import { useReducedMotion } from "@/components/effects/effect-utils"

export function SmoothScroll({ children }: { children: ReactNode }) {
  const reduced = useReducedMotion()
  if (reduced) return <>{children}</>
  return (
    <>
      <ReactLenis root options={{ anchors: true }} />
      {children}
    </>
  )
}
```

**With GSAP ScrollTrigger:** run Lenis from GSAP's ticker so both use one loop: set `autoRaf: false` in the options, then `lenis.on("scroll", ScrollTrigger.update)`, `gsap.ticker.add((time) => lenis.raf(time * 1000))` and `gsap.ticker.lagSmoothing(0)` (the Lenis README has the full snippet).

**Play with:** `lerp` or `duration` and `easing`, `orientation`, `wheelMultiplier`, `infinite`.

### 10. GSAP: the professional animation toolkit

[gsap.com](https://gsap.com) · [github.com/greensock/GSAP](https://github.com/greensock/GSAP) · React guide: [gsap.com/resources/React](https://gsap.com/resources/React)

The long-standing pro animation library: tweens and timelines for any property, plus plugins such as **ScrollTrigger** (pin, scrub and trigger animations on scroll), **SplitText** (animate words, lines and characters), **ScrollSmoother**, **Flip** (layout transitions), **MorphSVG**, **DrawSVG** and **MotionPath**. React projects use the `useGSAP` hook from `@gsap/react`, which cleans up automatically.

- **Licence (read it):** since Webflow took over GSAP, the library and every plugin are free, including commercial use, under the **GSAP Standard "no charge" licence**. It isn't an open-source licence: you may not use GSAP to build a no-code visual animation builder that competes with Webflow's, and Webflow can change the terms. Using it on websites and apps is fine. Record it for item 20 of the compliance checklist.
- **Rules:**
  - Register plugins once.
  - Write every animation inside `useGSAP` with a `scope` so it's cleaned up when the component unmounts.
  - Use `gsap.matchMedia()` with `(prefers-reduced-motion: no-preference)` so nothing animates for people who asked for less motion. The CSS safety net doesn't reach JavaScript.
  - Animate `transform` and `opacity` (`x`, `y`, `scale`, `autoAlpha`), not layout properties.
  - SplitText: keep the heading readable to screen readers (the `aria` option, on by default, does this) and call `revert()` on cleanup.
  - Don't pin or scroll-jack in app screens; scrubbed sections belong on public pages.

```tsx
// src/components/effects/reveal-heading.tsx
import { useGSAP } from "@gsap/react"
import gsap from "gsap"
import { ScrollTrigger } from "gsap/ScrollTrigger"
import { SplitText } from "gsap/SplitText"
import { useRef } from "react"

gsap.registerPlugin(useGSAP, ScrollTrigger, SplitText)

export function RevealHeading({ children }: { children: string }) {
  const ref = useRef<HTMLHeadingElement>(null)

  useGSAP(
    () => {
      const mm = gsap.matchMedia()
      mm.add("(prefers-reduced-motion: no-preference)", () => {
        const split = SplitText.create(ref.current, { type: "words" })
        gsap.from(split.words, {
          yPercent: 100,
          autoAlpha: 0,
          stagger: 0.04,
          duration: 0.6,
          ease: "power3.out",
          scrollTrigger: { trigger: ref.current, start: "top 80%", once: true },
        })
        return () => split.revert()
      })
      return () => mm.revert()
    },
    { scope: ref },
  )

  return (
    <h2 ref={ref} className="text-3xl font-medium">
      {children}
    </h2>
  )
}
```

**Play with:** timelines (`gsap.timeline()`), ScrollTrigger `scrub` and `pin` (with `markers: true` while building), SplitText lines and characters, `Flip` for layout changes, eases from the [ease visualiser](https://gsap.com/docs/v3/Eases).

### 11. Vanta: animated 3D backgrounds

[vantajs.com](https://www.vantajs.com) (gallery with live controls) · [github.com/tengbao/vanta](https://github.com/tengbao/vanta)

Ready-made WebGL backgrounds in a few lines: **Waves**, **Clouds**, **Birds**, **Fog**, **Net**, **Globe**, **Cells**, **Halo**, **Rings**, **Dots** and more. You tune colours and motion in the gallery, then copy the options. It draws a canvas behind the element's other children, so your hero text sits on top.

- **Fit (important):**
  - Vanta is old (last release 0.5.24) and built for **three.js r134**. It expects `THREE` as a global or passed in, and ships no TypeScript types.
  - Don't load three from a CDN `<script>` (our CSP blocks it). Install a pinned copy under an alias, so it doesn't clash with a newer `three` used by React Three Fiber: `pnpm add vanta three-r134@npm:three@0.134.0`.
  - Pass it in as the `THREE` option, and add a small type declaration (below).
  - A few effects (Topology, Trunk) need p5.js instead.
  - About 120 KB gzipped: lazy-load it like any Part A effect.
- **Rules:** decorative only (`aria-hidden`); turn off mouse and gyro controls unless they add something; show a still image under reduced motion and when WebGL fails; colours from the theme tokens; check text contrast at the busiest frame.

```ts
// src/types/vanta.d.ts
declare module "three-r134"
declare module "vanta/dist/vanta.*.min" {
  type VantaEffect = { destroy(): void; resize(): void; setOptions(options: Record<string, unknown>): void }
  const create: (options: { el: HTMLElement; THREE?: unknown } & Record<string, unknown>) => VantaEffect
  export default create
}
```

```tsx
// src/components/effects/waves-backdrop.tsx (lazy-loaded)
import { useEffect, useRef } from "react"
import * as THREE from "three-r134"
import WAVES from "vanta/dist/vanta.waves.min"
import { tokenColor, useReducedMotion } from "@/components/effects/effect-utils"

const toHex = (color: string) => Number.parseInt(color.slice(1), 16)

export default function WavesBackdrop() {
  const ref = useRef<HTMLDivElement>(null)
  const still = useReducedMotion()

  useEffect(() => {
    if (!ref.current || still) return
    const effect = WAVES({
      el: ref.current,
      THREE,
      color: toHex(tokenColor("--primary")),
      waveSpeed: 0.6,
      mouseControls: false,
      touchControls: false,
      gyroControls: false,
    })
    return () => effect.destroy()
  }, [still])

  // Under reduced motion this stays a plain block: give it a still brand image or gradient with Tailwind.
  return <div ref={ref} aria-hidden className="absolute inset-0" />
}
```

**Play with:** the gallery's per-effect controls (colours, speed, zoom, density, `waveHeight`, `shininess`, `birdSize`, `quantity`), then `effect.setOptions()` to change them live (for example on theme change).

### 12. React Bits: copy-paste animations for React

[reactbits.dev](https://reactbits.dev) · [github.com/DavidHDev/react-bits](https://github.com/DavidHDev/react-bits)

More than 200 free animated components, each with a live demo and controls:

- text animations (split text, blur text, shiny text, count up, scrambled text)
- interactive components (galleries, docks, magnetic buttons, cards)
- backgrounds (aurora, particles, silk, liquid chrome, threads)

Every component comes in four flavours (JS or TS, CSS or Tailwind) and installs through the shadcn CLI, so the code lives in your repo.

- **Install one at a time (owner OK first):** add `"@react-bits": "https://reactbits.dev/r/{name}.json"` to `registries` in `components.json`, then pick the **TypeScript + Tailwind** flavour: `pnpm dlx shadcn@latest add @react-bits/SplitText-TS-TW`. Read the diff: some components pull in GSAP, Motion, three.js or OGL.
- **Licence (important):** MIT + **Commons Clause**. You can use the components in any app or site, including paid ones. You may **not** sell, sublicense or redistribute the components themselves, alone or bundled.
  - So **don't copy React Bits code into this starter's own files** or any template you share or sell. Install it per project.
  - **React Bits Pro** is a separate paid product: check its terms before using anything from it.
  - Record what you use for item 20 of the compliance checklist.
- **Fit:**
  - Keep copied components in `src/components/reactbits/`, never in `src/components/ui/`.
  - Restyle to our tokens: no hard-coded colours, 10 px button corners instead of pills, `font-medium`.
  - Add reduced-motion handling where it's missing.
  - Backgrounds and big text effects follow Part A's rules. Small ones (count up, a subtle text reveal) can go in the app after `ux-reviewer` and `qa-tester`.
- **With Claude Code:** React Bits works with the shadcn MCP server (`pnpm dlx shadcn@latest mcp init --client claude`), so agents can search and add components. The same owner-approval and restyling rules apply.

**Play with:** each component's demo controls on the site; the site's background and texture tools for tuned exports.
---

## Part E: premium interaction patterns

Six small interactions that make a site feel crafted and responsive. They're **patterns, not libraries**: each one names the tool from this guide to build it with. Used with restraint they feel premium. Overdone, they're exactly what people complain about (lag, motion sickness, things that won't hold still).

**Where they belong:** public pages, marketing and brand moments, under Part A's rules: the owner opts in per surface, and reduced motion, keyboard, contrast and performance are checked. The everyday app keeps the calm rules in `DESIGN_SYSTEM.md` (hover changes colour, not position; motion explains a change). Page transitions and staggered reveals are the exceptions that also fit the app, at its own timings.

| Pattern | What it does for people | Build it with | Hard limits |
| --- | --- | --- | --- |
| Magnetic buttons | The interface seems to respond to *you* | Motion springs (item 5), GSAP `quickTo` (10) or React Bits "Magnet" (12) | Fine pointers only; a few px of pull; the hit area never moves away |
| Controlled parallax | Depth without a 3D scene | CSS scroll-driven animations, Motion `useScroll`, GSAP ScrollTrigger `scrub` | 2–3 decorative layers, small speed differences, never the text |
| Staggered scroll reveal | Paces how fast people take information in | Motion `whileInView` or GSAP ScrollTrigger; in the app, the existing `.list-enter` | Once only, short stagger, content readable without JavaScript |
| Easter eggs | Rewards curiosity; makes the site worth revisiting | Plain React state, a small Motion or canvas moment | Never hides or blocks a real feature; easy to dismiss |
| Page transitions | Navigation feels continuous instead of a hard cut | The View Transitions API in our router; Motion `AnimatePresence` | ≤ 300 ms, focus handling unchanged, instant under reduced motion |
| Cursor with context | The cursor tells you what a click will do ("View", "Drag", "Play") | Motion `useMotionValue` + `useSpring`; React Bits cursor effects as reference | Fine pointers only; never hides the real cursor on text or inputs; never replaces focus rings |

### 1. Magnetic buttons

When the pointer comes near a button, the button drifts a few pixels toward it and springs back when the pointer leaves.

- Only for **one or two primary calls to action** on a public page, never for every button or in dense app UI.
- Only on devices with a fine pointer and hover (`(hover: hover) and (pointer: fine)`); touch screens get the normal button.
- Pull is small: about 0.2–0.3 of the pointer's distance, capped at 6–8 px. Move with `transform` only, so layout never shifts.
- The attraction zone is a little larger than the button, but the click target stays the real button. Keyboard focus and the focus ring are unaffected.
- Off under reduced motion.

```tsx
// src/components/effects/magnetic.tsx: wrap a real <Button>, don't replace it
import { motion, useSpring } from "motion/react"
import type { PointerEvent, ReactNode } from "react"
import { useReducedMotion } from "@/components/effects/effect-utils"

const SPRING = { stiffness: 300, damping: 20, mass: 0.5 }
const finePointer = () => window.matchMedia("(hover: hover) and (pointer: fine)").matches

export function Magnetic({ children, strength = 0.25, max = 8 }: { children: ReactNode; strength?: number; max?: number }) {
  const reduced = useReducedMotion()
  const x = useSpring(0, SPRING)
  const y = useSpring(0, SPRING)
  const clamp = (v: number) => Math.max(-max, Math.min(max, v))

  function onMove(event: PointerEvent<HTMLSpanElement>) {
    if (reduced || !finePointer()) return
    const box = event.currentTarget.getBoundingClientRect()
    x.set(clamp((event.clientX - (box.left + box.width / 2)) * strength))
    y.set(clamp((event.clientY - (box.top + box.height / 2)) * strength))
  }

  function reset() {
    x.set(0)
    y.set(0)
  }

  // The padding makes the attraction zone larger than the button; the negative margin keeps layout unchanged.
  return (
    <span className="-m-4 inline-block p-4" onPointerMove={onMove} onPointerLeave={reset}>
      <motion.span className="inline-block" style={{ x, y }}>
        {children}
      </motion.span>
    </span>
  )
}
```

### 2. Controlled parallax

Background layers move a little slower than the content as you scroll, creating depth. This is the pattern people complain about most when it's overdone, so the limits are strict:

- **At most 2–3 layers**, and only decorative ones (shapes, a background image, a glow). Text, buttons and forms scroll normally.
- **Small differences:** a layer moves at 80–95 % of scroll speed, or shifts at most about 10–15 % of its height over the section. No horizontal drift, no rotation, no zoom tied to scroll.
- **Turn it off** under reduced motion (large-area motion can trigger vestibular symptoms; WCAG 2.3.3) and consider turning it off on phones.
- Prefer **CSS scroll-driven animations** (`animation-timeline: view()`) where supported, falling back to a static layer; or Motion `useScroll` + `useTransform`, or GSAP ScrollTrigger with `scrub`. Never listen to `scroll` and set `top` by hand.
- Check that scrolling stays smooth on a mid-range phone, and that nothing lags behind or jitters.

### 3. Staggered scroll reveal

Content comes in as it enters the screen, item by item, so people read in order instead of facing a wall.

- **Reveal once.** Don't hide things again on scroll up, and never re-animate content people have already seen.
- **Short and quick:** a 30–60 ms stagger, each item 300–600 ms, capped so a whole group finishes within about half a second. Small rise (8–24 px) plus fade.
- **Content must be readable without JavaScript and to crawlers:** set the hidden start state from JavaScript, not in CSS that ships hidden, so search engines and failed scripts still show the text (`SEO_AND_DISCOVERABILITY.md`).
- Under reduced motion, show everything immediately.
- In the app, use the existing `.list-enter` (30 ms stagger, capped at 150 ms) and the `ENTER` tokens from `DESIGN_SYSTEM.md` §15 instead of new timings.
- Tools: Motion `whileInView` with `viewport={{ once: true }}`; GSAP ScrollTrigger with `once: true` (see the `RevealHeading` example in item 10).

### 4. Easter eggs

A small hidden delight: a playful response to the Konami code, a logo that reacts after a few clicks, a fun 404 page, a secret theme. A site with nothing to discover feels like it was built to be used once and forgotten.

- **Never hide or gate anything people need.** Easter eggs add; they never replace a feature, a setting or information.
- **Not accidental:** triggered by something deliberate (a key sequence, several clicks), so screen-reader and keyboard users don't trip into it by normal use. Easy to dismiss with Esc or a click.
- **Kind and short:** no sound without consent, no flashing (WCAG 2.3.1: nothing flashes more than three times a second), respects reduced motion, no data collection.
- On brand and in good taste; the owner approves it. Note it in `docs/GUIDE.md` so QA knows it exists and tests it.
- Keep it light: load any heavy part (confetti, a canvas, a game) only when triggered.

### 5. Page transitions

A smooth change between pages (a quick cross-fade, or a card that grows into its detail page) instead of a hard cut.

- **Use the View Transitions API** with our router (`src/lib/route.ts`): wrap the state update that changes the page in `document.startViewTransition`. Where it isn't supported, or under reduced motion, the change happens instantly. Shared elements get the same `view-transition-name` on both pages.
- **Timing from `DESIGN_SYSTEM.md` §15:** 200 ms fades, 300 ms at most for things that move. The app shell already fades pages in (`animate-fade`), so don't stack a second fade on top.
- **Route focus (§16) still runs:** after the transition, focus moves to the new page's `h1` as before. The tab title still updates.
- Don't block navigation on an animation: if data is still loading, show the page's loading state inside the transition rather than holding the old page.
- The Vercel `react-view-transitions` skill covers React's `<ViewTransition>` component; read it through `skills-scout` (`WORKFLOW.md` §8).

```ts
// src/lib/view-transition.ts: call from the router's navigate before updating the route state
import { flushSync } from "react-dom"

export function withViewTransition(update: () => void) {
  const reduced = window.matchMedia("(prefers-reduced-motion: reduce)").matches
  if (reduced || !("startViewTransition" in document)) {
    update()
    return
  }
  document.startViewTransition(() => flushSync(update))
}
```

### 6. Cursor with context

A small follower near the pointer that changes to say what a click will do: "View" over a project card, "Drag" over a carousel, "Play" over a video.

- **Fine pointers only.** Touch devices and keyboard users get the normal UI, so the label is never the only place that information appears: cards still look clickable and have accessible names.
- **Keep the real cursor** on text, inputs and links (the follower sits beside it, or the system cursor returns there). Hiding the system cursor everywhere breaks text selection and confuses people.
- **No lag:** a spring that settles in about 100 ms. A cursor that trails behind feels broken, not premium.
- `aria-hidden` on the follower; it never covers content you're about to click, and it never replaces focus rings.
- Under reduced motion, show the label without movement or scaling animations (or turn the effect off).
- Build it with Motion `useMotionValue` (pointer position) + `useSpring`, with `data-cursor="View"` attributes on target elements. React Bits has cursor effects to borrow ideas from.

### Review checklist for interactions

- [ ] The owner approved the surface; each pattern is used sparingly (one magnetic CTA, not ten).
- [ ] Everything works with keyboard only and on touch, with the effect absent.
- [ ] Reduced motion turns parallax, magnetism and reveals off, and makes transitions instant.
- [ ] No content is hidden without JavaScript; nothing needed is behind an Easter egg.
- [ ] Smooth on a mid-range phone: no scroll jank, no cursor lag, no layout shift (CLS).
- [ ] Focus rings, route focus and tab titles behave exactly as without the effect.
---

## Part F: visual reference archives

The best design reference often isn't on the usual inspiration sites. It's in museum and library archives: centuries of people who solved type, colour, layout and print long before software. Use these for mood boards, colour palettes, type studies and texture, and in some cases as real assets.

| Archive | What's there | Can you use the images in a product? |
| --- | --- | --- |
| [Letterform Archive online archive](https://letterformarchive.org/online-archive/) | Lettering, type specimens, posters and print, photographed close enough to see the letterpress bite in the paper | **Reference only, by default.** Each item shows a copyright status: public domain items have no known restrictions; copyrighted or unknown items need permission through their [rights and reproductions](https://letterformarchive.org/rights-repro) process, with fees for commercial use. |
| [The Met Open Access](https://www.metmuseum.org/art/collection) | More than 400,000 high-resolution images of public-domain works across 5,000 years of art and design | **Yes, for images marked Open Access** (CC0, no attribution required, commercial use allowed). Works still under copyright aren't in the set. ([policy](https://www.metmuseum.org/en/policies/image-resources)) |
| [Rijksmuseum collection](https://www.rijksmuseum.nl/en/collection) | Hundreds of thousands of works in high resolution: four centuries of colour, pattern and composition | **Often, for public-domain works the museum offers as free downloads**; check each item's rights line before using it |
| [archives.design](https://archives.design) | A curated index of graphic design material on the Internet Archive: Olivetti brochures, Apple ephemera (1980s brochures and logo standards), type specimens, *Emigre*, the *Vignelli Canon* | **Reference only.** It links to scans of material still owned by its publishers. Logos and brand material (Olivetti, Apple) are trademarks: study them, never reuse them. |
| [NYPL Digital Collections](https://digitalcollections.nypl.org) | Prints, maps, fashion plates, photographs, menus, ephemera; very searchable | **Yes, for items marked public domain** ("No known copyright restrictions"), with high-resolution downloads; other items are reference only |

How to use them well:

- **Study, then make your own.** Pull palettes, proportions, type pairings and grids from what you find, then build them from our tokens. A palette taken from a 17th-century painting still needs measured contrast in both themes (`DESIGN_SYSTEM.md` §2).
- **Check the rights line on every image you ship.** Only images clearly marked public domain or CC0 go into a product. Record each one (source URL, licence, date) in `ASSETS_LICENSES.md` (compliance checklist item 20).
- **Public domain covers the artwork, not everything around it.** Trademarks, recognisable people and modern photos of objects can still have rights. When in doubt, use it as reference only.
- **Mind the weight:** archive files are huge. Resize and convert to modern formats (AVIF or WebP) with explicit width and height before they go on a page.
- **Keep a mood board** per project (a folder or a `docs/design/references.md`) with source links, so `product-designer` and the owner can see where a direction came from.

## More places to look for inspiration

Add repos here with the same table row (repo, use, licence, package) after checking the licence: the rest of the [pmndrs](https://github.com/pmndrs) ecosystem (drei, react-postprocessing, react-spring), the [Paper Shaders gallery](https://shaders.paper.design), [Bklit Studio](https://bklit.com), the [Motion examples](https://motion.dev/examples), the [GSAP showcase](https://gsap.com/showcase), the [Vanta gallery](https://www.vantajs.com), [React Bits](https://reactbits.dev), and the three.js [examples](https://threejs.org/examples/).
