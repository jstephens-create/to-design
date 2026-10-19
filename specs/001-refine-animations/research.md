# Phase 0 Research: Refine Interactive Animations

**Feature**: [spec.md](./spec.md) · **Audit**: [audit.md](./audit.md) · **Plan**: [plan.md](./plan.md)

Each decision below resolves an open question from the plan's Technical Context. Line
references are to `index.html` at commit 15429cc.

---

## D1. Frame scheduling: demand-driven loop instead of an always-on loop

**Decision**: Replace the unconditional `requestAnimationFrame(loop)` (~L459 to L468) with a
scheduler that only requests the next frame when something needs it. Any of these
"wakes" the loop: pointer movement, scroll, resize, a visibility or preference change, a
user choice (gift, tab, commit), or a ticker that reports it is still moving. Each ticker
returns `true` while it needs another frame and `false` once it has settled. When every
active ticker returns `false` and no input arrived, the loop stops requesting frames.

**Rationale**: Meets FR-007 and SC-002 directly. Off-screen sections already drop out
through `whenVisible`; this extends the same idea to "on screen but at rest". `rAF` is
already paused by browsers in hidden tabs, so tab-visibility pausing (Principle V) comes
for free.

**Alternatives considered**:
- *Keep the always-on loop and skip work inside each ticker*: still wakes the main thread
  60 to 120 times a second, so it cannot meet SC-002.
- *A separate `rAF` loop per section*: duplicates input handling and makes the read/write
  ordering in D2 impossible to guarantee.

---

## D2. Layout reads: one read phase before any writes

**Decision**: Split every ticker into `read()` and `write(now, dt)`. Each frame the loop
runs all `read()` calls first (every `getBoundingClientRect`, `clientWidth`, `innerHeight`
read), then the cursor, flex and progress writes, then every `write()`. No ticker reads
layout inside `write()`.

**Rationale**: Today `cursorTick` writes transforms and then four tickers read rects
(audit P1), so the browser flushes style and layout several times per frame. Reading
first gives at most one flush per frame (FR-006). It is a smaller change than caching
rects and stays correct for the sticky Gifts assembly, whose rect changes every scroll
frame.

**Alternatives considered**:
- *Cache rects in page coordinates and update them on resize*: breaks for the sticky
  `.assembly` and for anything above it whose height changes (for example font load).
- *`IntersectionObserver` entries as the only geometry source*: entries arrive
  asynchronously and only on threshold crossings, which is too coarse for cursor math.

---

## D3. Frame-rate independent easing and momentum

**Decision**: Replace every fixed per-frame factor with a time-based damp:

```text
damp(current, target, duration, dt) = current + (target - current) * (1 - exp(-3 * dt / duration))
```

`duration` is a motion token in seconds (D9) and means "time to reach about 95% of the
target". Momentum uses the same form: `v *= exp(-3 * dt / duration)`. `dt` stays capped
at 50ms, as today, so a returning tab does not jump.

Mapping from today's 60Hz factors (equivalent settle time at 60fps in brackets):

| Today | Where | Settle (95%) | Token |
|---|---|---|---|
| `*.18` | cursor ring and tag (L461) | ~0.26s | fast (0.25s) |
| `*.12` | closing blocks (L904) | ~0.39s | base (0.5s) |
| `*.1` | gifts explode (L753) | ~0.47s | base |
| `*.08` | field focus, gifts tilt, wire tilt (L659, L759, L789) | ~0.60s | base |
| `*.07` | wire morph (L785) | ~0.69s | base (in step with role transition, D11) |
| `*.05` | field amplitude (L659) | ~0.97s | slow (0.9s) |
| `*= .94` | hero momentum (L612) | ~0.81s | slow |

**Rationale**: FR-012 and SC-004. Mapping each factor to the nearest token keeps the
current feel within the "about" tolerance the spec allows, and ties scripted motion to
the same vocabulary as CSS (FR-019).

**Alternatives considered**:
- *Fixed-timestep simulation at 60Hz*: exact match to today's feel, but adds catch-up
  loops and still needs interpolation on 120Hz displays.
- *`pow(k, dt/16.67)`*: equivalent maths, but keeps the magic per-frame constants instead
  of shared durations.

**Settle test**: a damped value is "settled" when `|target - current|` is below a
per-quantity epsilon (0.1px for positions, 0.01deg for rotations, 0.001 for normalized
values). On settle the value snaps to its target so the ticker can return `false`.

---

## D4. Pointer state on touch, pen and leaving the window

**Decision**: Track `pointerType` on every pointer event. Set `M.on = false` on
`pointerup` and `pointercancel` when the type is `touch` or `pen`, and on any pointer when
it leaves the window (`pointerout` on `document` with no `relatedTarget`, plus `blur`).
Effects then relax to idle through their normal damp (base, about 0.5s), well inside the
one-second budget in US2 and SC-005. The custom cursor shows only while the most recent
pointer is a mouse and the device matches `(hover: hover) and (pointer: fine)`.

**Rationale**: Fixes audit R2 and the hybrid-laptop edge case without separate code paths
per section. Sections keep reading `M.on`, so their logic does not change.

**Alternatives considered**: a timeout that clears `M.on` after inactivity. Rejected
because a still mouse over the hero should keep its repel effect.

---

## D5. Hero drag: pointer capture and cancel

**Decision**: On `pointerdown` call `sec.setPointerCapture(e.pointerId)` and record the
pointer ID. End the drag on `pointerup`, `pointercancel` and `lostpointercapture` for that
ID. A new drag always starts from the current `rotY`/`rotX` with a fresh `drag` object, so
no delta is computed from a stale position. Move and up listeners attach to `sec`, not
`window`. `touch-action: pan-y` on `#hero` stays as it is, so vertical swipes still scroll
the page.

**Rationale**: Fixes audit R3 and FR-014. When the browser claims a vertical pan it fires
`pointercancel`, which now clears the drag.

---

## D6. Resizing without clearing canvases

**Decision**:
1. Each canvas (`#net`, `#field`, `#wire`) and the closing `#stage` gets one shared
   `ResizeObserver`. Callbacks are debounced to 150ms and only apply when the rounded CSS
   size or the device pixel ratio actually changed. Until then the existing bitmap
   stretches with CSS, which is invisible for that short time.
2. The two heights that use plain `vh` gain an `svh` override with the `vh` value kept as
   the fallback, matching the pattern already used by `#hero` and `.assembly`:
   `#field` (`46vh`) and `.stage` (`62vh`). On desktop `svh` equals `vh`, so SC-007 is not
   affected; on phones the URL bar no longer changes their size.
3. Remove the four direct `resize` listeners (~L594, L652, L782, L891).

**Implementation note (supersedes step 2)**: the `svh` overrides were not applied. In
current mobile browsers `vh` already equals `lvh` and does not follow the URL bar, while
`svh` would shorten both areas on phones, a visible change at rest (analysis finding I1).
The size check in step 1 is what prevents clearing.

**Rationale**: FR-009 and US1 scenario 3. `ResizeObserver` reports element size, so it
also catches layout changes that are not window resizes (for example the wireframe
column reflowing when fonts load).

**Alternatives considered**: debouncing `window.resize` only. Rejected because it still
fires on URL-bar changes and misses element-only resizes.

---

## D7. Hero density follows the viewport without resetting rotation

**Decision**: Factor the graph build (~L565 to L589) into `buildGraph(N)` that reseeds the
PRNG each time, so the 560-node and 1150-node graphs are identical to today's. Build the
graph for the current width at load. When the debounced resize (D6) crosses the 700px
threshold, build (and cache) the other graph and swap it in. `rotY`, `rotX` and momentum
live outside the graph and are untouched. A signal ripple in flight is dropped on swap
because node indices differ.

**Rationale**: FR-017 and audit R6. The build is O(N²) for nearest neighbours, about
1.3M distance checks at 1150 nodes, which costs tens of milliseconds once, only on a
threshold crossing.

**Alternatives considered**: always building 1150 and drawing a subset on phones. Rejected
because the subset's edges would differ from today's phone globe (SC-007).

---

## D8. Flexible headline weights without reflow or timers

**Decision**:
1. **No rewrap (FR-010)**: when measuring, give every `.w` (word) span an inline
   `width` in `em` equal to its width at base weight, with `display: inline-block` and
   `text-align: center`. Weights in this effect only go down from base (lighter is
   narrower), so letters can shift inside their word but the word's box, and therefore
   line breaks and surrounding content, never changes.
2. **No per-character CSS transition**: remove `.flex .ch { transition: ... }` and damp each
   character's weight in the flex write step with the base token (D3). Writes are still
   skipped below a 3-unit change, as today. This turns hundreds of concurrent CSS
   transitions into one JS pass that stops when settled.
3. **Transform-proof measurement (FR-016)**: compute character centres with the
   `offsetLeft`/`offsetTop` chain, which ignores CSS transforms. Reveal transitions
   (`translateY(32px)`) no longer skew positions, so measuring once is enough.
4. **Event-driven re-measure, no timers**: re-measure (debounced to the next frame) on
   `document.fonts.ready`, `document.fonts` `loadingdone`, and a `ResizeObserver` on each
   `.flex` element and on `<main>` (catches layout shifts above a headline). Remove the
   1s `setTimeout` in the reveal observer (~L506) and the 1.2s start timer (~L913).

5. **Implementation note, added after measurement**: word slots removed rewrapping but
   not the cost of reshaping text, which still caused most slow frames during scroll.
   Two changes fixed that: weights are written in steps of 40 units (of a 600-unit range)
   while easing, and weight updates pause while the page is scrolling, resuming 120ms after
   scrolling stops. Slow scroll frames dropped from 73 to 2–6. The word-slot class (`.fx`)
   is applied only when the effect is active, so touch and reduced-motion visitors keep plain
   inline text. CSS `contain` was tested and gave no gain.

**Rationale**: Fixes audit P6 and R5 with the smallest visual change: at rest nothing
moves; in motion letters still thin out around the cursor as before.

**Alternatives considered**:
- *Fixed width per character*: also stops reflow, but leaves visible gaps between thin
  letters, which changes the look in motion more than per-word slots do.
- *`contain: layout size` on characters*: breaks baseline alignment of inline-blocks.
- *Animating `transform: scaleX` instead of weight*: a different effect, violates FR-001.

---

## D9. Motion vocabulary

**Decision**: Define on `:root`:

| Token | Value | Replaces |
|---|---|---|
| `--t-fast` | `.25s` | 0.2s, 0.25s, 0.3s, 0.35s (hover feedback, colour and fill changes) |
| `--t-base` | `.5s` | 0.45s, 0.5s, 0.6s (state changes, tilt return, entrances of single items) |
| `--t-slow` | `.9s` | 0.9s, 1s (section reveals) |
| `--ease-out` | `cubic-bezier(.2,.7,.2,1)` | the existing main curve, for anything that moves |
| `--ease` | `ease` | the CSS default already used by colour and opacity fades |
| `--stagger` | `70ms` | Lab card entrance step (today a `setTimeout` offset) |

JS reads the three durations once from `getComputedStyle(document.documentElement)` (with
the numbers above as fallback) so CSS stays the single source.

**Rationale**: FR-019 and SC-009. "About three durations and two curves" from the spec
assumptions. Fast is 0.25s rather than 0.2s because it is the median of the four short
values in use and keeps the cursor ring's current feel (D3).

**Note**: `--stagger` is a delay step, not a duration; SC-009's inventory counts it as part
of the set.

---

## D10. One "tracking" pattern for tilt and magnetic controls

**Decision**: Lab cards, Lab tabs and HUD nav links share one rule: while the pointer is
over the element, it carries a `.trk` class that sets its `transform` transition to `none`
so it follows the pointer directly. On `pointerleave` the class is removed and the
element returns to rest over `--t-base` with `--ease-out`. All three get
`transition: transform var(--t-base) var(--ease-out)` alongside their existing
transitions.

**Rationale**: FR-021 and FR-022, audit V3 and V4. One mechanism gives the "same feel" the
spec asks for and keeps Lab tilt as crisp as the Gifts tilt.

---

## D11. Lab commit and Paths content changes

**Decision**:
- **Lab (FR-020)**: split `render(i)` into `renderTab(i)` (rebuilds cards, plays the
  staggered entrance via `transition-delay: calc(var(--i) * var(--stagger))`) and
  `updateCards()` (only flips `aria-pressed`, `data-cursor` and the foot label on existing
  cards). Card click calls `updateCards()`, so focus stays on the clicked button and no card
  re-enters. Keyboard tab switching keeps calling `renderTab`.
- **Paths (FR-023)**: on gift change, swap the role cards and note text, then play a
  Web Animations API entrance on `.roles` and `#branch-note`
  (`opacity 0 → 1`, `translateY(12px) → 0`, `--t-base`, `--ease-out`). The wireframe morph
  starts in the same frame and also settles over base, so the two land together. Under
  reduced motion the animation is skipped and the swap is instant.
- **Typed log (FR-011)**: progress is `chars = elapsed_ms * 0.25` (today's 4 characters per
  16ms), computed in the careers `write()` step. If the panel is off screen the log simply
  shows its full text when next drawn.

**Rationale**: WAAPI is built into every target browser, needs no dependency
(Principle I), and avoids forcing a reflow to restart a CSS animation.

---

## D12. Reduced motion as a live setting

**Decision**: Keep one `MediaQueryList` for `(prefers-reduced-motion: reduce)` and one for
`(hover: hover) and (pointer: fine)`, read `.matches` into `state.reduce` and
`state.fine`, and listen for `change`. On change:
- Clear inline `transform` on magnetic controls, Lab cards and band steps.
- Remove or add `has-cursor`.
- Wake every ticker once so each section renders its correct state (D13).
- Handlers that were registered only when `!reduce` (magnetic ~L553, card tilt ~L856) are
  registered always and check `state.reduce` at event time.

Under reduced motion (FR-025 to FR-027): pointer displacement, tilt, drift, weight change,
cursor ring easing (ring snaps to the pointer) and Paths/Lab entrances are off. Colour and
fill changes stay, cut to instant by the existing CSS media rule.

**Rationale**: FR-018, audit R7 and A1.

---

## D13. What may keep drawing while on screen

**Decision**: With reduced motion off, these tickers report "still moving" for as long as
their section is on screen, because the motion is intended:

- Hero globe slow rotation (spec assumption).
- Gifts assembly bob once exploded (spec assumption).
- Paths wireframe slow turntable rotation (see note).

Everything else settles and stops: the Purpose line field and closing blocks become
pointer- and scroll-driven, and their time-based drift terms (`t*.35`, `t*.12`, `t*.3` in
the field; `sin(t*.7+i)` in the close) are replaced by the equivalent scroll phase, so
they still move as the visitor scrolls but hold still at rest.

**Rationale**: SC-002 requires near-zero CPU at rest with the hero off screen, and the
spec's assumptions name only the hero and Gifts as continuous. The line field redraws
hundreds of rectangles per frame, so it is the main cost to remove. Its print reference
(Weingart) reads well standing still.

**Note for the owner**: the wireframe turntable is not listed in the spec's assumptions.
It is cheap (about 400 line segments) and central to the Paths HUD look, so this plan keeps
it. Measure SC-002 with the Purpose band or Lab in view. If you prefer the wireframe to
stop at rest too, it can settle like the field; amend the spec assumption either way.

**Alternatives considered**: keeping the field's time drift at a lower frame rate. Rejected
because a stepped shimmer changes the look and still wakes the main thread.

---

## D14. Allocation and redundant writes

**Decision**:
- Hero and wireframe: replace per-frame `new Path2D()` with `ctx.beginPath()` and batched
  `moveTo`/`lineTo` per stroke style, one pass per style (FR-008). The edge lists are
  bucketed into reusable `Uint16Array` index lists that are reset, not reallocated.
- Gifts: write each `transform` attribute and callout `opacity` only when its formatted
  value changes (string compare against the last written value).
- Cursor and coords: write only when the pointer or scroll moved, or the ring is still
  easing. Coords text keeps its 60ms throttle.
- Progress bar: unchanged (already scroll-gated).

**Rationale**: Audit P2 and P4, SC-003.

---

## D15. Entering a section after a jump

**Decision**: When a ticker becomes active (its section enters view), its first frame
snaps scroll-derived state (`ex` in Gifts, block positions in Close, `fx` in the field)
to the target instead of easing from a stale value. Pointer-derived state still eases.

**Rationale**: Covers the skip-link/HUD jump and fast-scroll edge cases: nothing is left
"half-run". Closing blocks already do this through `m.init`; this generalises it.

---

## D16. Verification method (no test framework)

**Decision**: Verify with browser DevTools and manual checks, as documented in
[quickstart.md](./quickstart.md): Performance panel recordings (4x CPU throttle) against a
git worktree of commit 15429cc served on a second port, the Rendering panel's
"Emulate CSS prefers-reduced-motion" and frame-rate meter, device-mode touch emulation, a
real phone for touch and rotation, and screenshot comparisons at 375px and 1440px.

**Rationale**: Principle I forbids package managers and build steps, so no test runner is
added. All success criteria are observable in the browser.
