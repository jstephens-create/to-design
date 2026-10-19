# Animation Audit: index.html (as of commit 15429cc)

**Purpose**: Record how the page's interactive animation behaves today, as input to
[spec.md](./spec.md). Line numbers refer to `index.html` at commit 15429cc.

## How motion is wired today

- One global `requestAnimationFrame` loop (`loop`, ~L459) runs on every frame for the life
  of the page. It updates the custom cursor, the flexible-type weights, the scroll progress
  bar, and any section "tickers" registered by `whenVisible` (~L455).
- `whenVisible` adds or removes a section's frame function using an IntersectionObserver.
- Five sections own an animation:
  | Section | Effect | Drawn with |
  |---|---|---|
  | 01 Hero | Rotating network globe; drag to spin, click to send a signal; nodes push away from the cursor | Canvas, ~1150 nodes |
  | 02 Purpose | Weingart line field that thickens near the cursor; orange step bars drift | Canvas plus CSS transforms |
  | 03 Gifts | Exploded isometric assembly spreads apart on scroll and tilts toward the cursor | Inline SVG |
  | 04 Paths | Wireframe that morphs per gift, typed console log | Canvas plus `setInterval` |
  | 05 Lab | Card stagger-in, cursor tilt and glow | CSS transitions plus inline transforms |
  | 06 Act | Five blocks drift in and lock together on scroll; cursor pushes them | DOM transforms |
- Site-wide: custom crosshair cursor, headline weights that follow the cursor ("flex"),
  scroll-reveal (`.rv`), magnetic nav and tab buttons, scroll progress bar.

## Findings

### Performance

- **P1 Layout thrashing every frame.** `cursorTick` writes styles first, then each
  ticker calls `getBoundingClientRect()`: hero ~L613, gifts ~L751 and ~L758, careers ~L786,
  close ~L893. Reading layout after writing styles forces the browser to recalculate
  layout several times per frame.
- **P2 Work continues when nothing changes.** The global loop runs every frame even when
  the pointer is still and the page is not scrolling. The cursor and coordinate readout are
  rewritten every frame. The gifts assembly rewrites SVG `transform` attributes every frame
  after it has settled.
- **P3 Line field redraws its full width every frame** (~L660 to L675), thousands of
  `fillRect` calls, even under reduced motion when the picture does not change.
- **P4 Allocation per frame.** The hero creates three `Path2D` objects every frame (~L631)
  and the wireframe creates two (~L800), which adds garbage-collection pauses.
- **P5 Resize handlers are not debounced.** Hero, field, wire and close each listen to
  `resize` directly (~L594, L652, L782, L891). Mobile browsers fire `resize` when the URL
  bar shows or hides, so the canvases are reallocated and cleared while the visitor scrolls.
- **P6 Flexible type changes layout.** Changing `font-variation-settings` per character
  (~L547), with a 0.45s transition (~L66), reflows large headlines on every pointer move.
  Glyph widths change with weight, so lines can shift or rewrap.
- **P7 Typed log** rewrites `innerHTML` every 16ms with `setInterval` (~L820), outside the
  frame loop.

### Responsiveness and correctness

- **R1 Animation speed depends on refresh rate.** All smoothing uses fixed per-frame
  factors (`*.18`, `*.08`, `*.1`, `*.12`, momentum `*= .94`). On a 120Hz display, motion
  settles about twice as fast as on 60Hz. On a throttled 30fps device it feels sluggish.
- **R2 Touch leaves "ghost" pointer effects.** On touch, `pointermove` sets `M.on = true`
  (~L451) and nothing resets it, because `pointerleave` on the root rarely fires for touch.
  After a tap, the hero repel, line-field focus and closing-block push stay stuck where the
  finger lifted.
- **R3 Hero drag can stick on touch.** No `pointercancel` handler and no pointer capture
  (~L603 to L609). When the browser takes over a vertical pan, `pointerup` never arrives,
  `drag` stays set, and the next touch makes the globe jump.
- **R4 Line-field ticker is registered twice** (`whenVisible(cv)` and `whenVisible(band)`,
  ~L679 and L680). Both observers add and remove the same function, so when the canvas
  leaves view the step bars in the band freeze even though the band is still on screen.
- **R5 Flex measurement is timing-based.** Character positions are measured during
  reveal transitions and corrected by a 1s `setTimeout` (~L506) and a 1.2s start timer
  (~L913). Until then, the weight hot spot can sit away from the cursor.
- **R6 Hero density is fixed at load** from `innerWidth` (~L568). Rotating a phone or
  resizing a window does not adjust it.
- **R7 Reduced-motion preference is read once** (~L401). Changing the OS setting does not
  apply until the page reloads.

### Visual consistency

- **V1 No shared motion vocabulary.** CSS transitions use 0.2s, 0.25s, 0.3s, 0.35s,
  0.45s, 0.5s, 0.6s, 0.9s and 1s with two different easing curves. JS uses its own ad hoc
  easing per section.
- **V2 Lab grid flashes on every commit click.** Clicking a card calls `render(i)`
  (~L855), which rebuilds all five cards with the `pre` class, so the other four fade and
  slide in again.
- **V3 Card tilt lags.** The tilt is written straight onto `transform` while `.card` has
  a 0.5s `transform` transition (~L179), so tilt trails the cursor with a mushy feel, unlike
  the crisp tilt on the gifts assembly.
- **V4 Magnetic tabs snap.** Tabs get a magnetic `transform` but have no transition on
  `transform`, so they jump back on `pointerout`, while HUD links fade smoothly.
- **V5 Role cards swap instantly.** In Paths, the wireframe morphs smoothly but role cards
  and the note are replaced with no transition.
- **V6 Dead CSS.** `.wipe` (~L71 to L73) is defined but not used in the markup.

### Accessibility of motion

- **A1 Reduced motion is only partly honoured.** CSS transitions are cut, but the canvases
  and transforms keep reacting to the cursor: hero repel, line field focus, closing-block
  push, flexible type, cursor ring easing. The constitution (Principle III) asks for
  non-essential motion to be disabled or softened.
- **A2 Hero signal is pointer-only.** The globe "send a signal" interaction has no
  keyboard equivalent. Its information is decorative, so this is noted, not required.

## What already works well (keep)

- Off-screen sections stop their tickers.
- Canvas pixel ratio is capped at 2.
- `prefers-reduced-motion` and `(pointer: fine)` checks already exist.
- Flex weight writes are skipped below a 3-unit change and outside the viewport.
- All core text is in the HTML. Animations only decorate it.
