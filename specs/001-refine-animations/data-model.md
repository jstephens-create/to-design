# Data Model: Refine Interactive Animations

**Plan**: [plan.md](./plan.md) · **Research**: [research.md](./research.md)

This feature has no server data. The "entities" are the runtime state objects inside the
single `<script>` in `index.html`, plus one value kept in `localStorage`. Names below are
the intended identifiers; implementation may shorten them to match the file's terse style.

---

## 1. MotionTokens

Shared durations and curves (research D9). Defined once in CSS, mirrored once into JS.

| Field | Type | Source | Rule |
|---|---|---|---|
| `fast` | seconds | `--t-fast` (.25s) | Hover feedback, colour and fill changes, cursor ring |
| `base` | seconds | `--t-base` (.5s) | State changes, tilt and magnetic return, single entrances, most damping |
| `slow` | seconds | `--t-slow` (.9s) | Section reveals, hero momentum, field amplitude |
| `easeOut` | CSS easing | `--ease-out` | Anything that moves |
| `ease` | CSS easing | `--ease` | Colour and opacity fades |
| `stagger` | ms | `--stagger` (70ms) | Lab card entrance step |

**Validation**: every CSS `transition`, `transition-delay` and WAAPI `duration`/`easing`
uses one of these (SC-009). JS falls back to the literal values if the custom property
cannot be read.

---

## 2. PointerState (`M`)

| Field | Type | Meaning |
|---|---|---|
| `x`, `y` | px (viewport) | Latest pointer position |
| `sx`, `sy` | px (viewport) | Smoothed position (damped with `fast`; equals `x`,`y` under reduced motion) |
| `on` | boolean | A pointer is hovering or touching inside the window |
| `type` | `'mouse' \| 'pen' \| 'touch'` | `pointerType` of the most recent event |
| `moved` | boolean | Input arrived since the last frame (wakes the loop) |

**State transitions**:

```text
          pointermove (mouse)                  pointerdown/move (touch, pen)
 off ───────────────────────────▶ hovering      off ──────────────────────────▶ touching
  ▲                                  │            ▲                                │
  │ pointerout(no relatedTarget),    │            │ pointerup / pointercancel      │
  │ window blur                      │            └────────────────────────────────┘
  └──────────────────────────────────┘
```

`on` is `true` in `hovering` and `touching`. Leaving either sets `on = false`; effects
then relax through their own damp (≤ 1s, SC-005).

---

## 3. MotionPrefs (`state`)

| Field | Type | Source |
|---|---|---|
| `reduce` | boolean | `matchMedia('(prefers-reduced-motion: reduce)')`, live via `change` |
| `fine` | boolean | `matchMedia('(hover: hover) and (pointer: fine)')`, live via `change` |
| `cursorVisible` | derived | `fine && M.type === 'mouse' && M.on` |

Changing either preference wakes every ticker once (research D12).

---

## 4. Ticker (one per animated section)

| Field | Type | Meaning |
|---|---|---|
| `el` | Element | Observed element; ticker is active while any part is within the root margin |
| `read()` | function | All layout reads for this frame; stores rects on the ticker |
| `write(now, dt)` | function → boolean | All drawing and style writes; returns `true` if another frame is needed |
| `entered` | boolean | `true` on the first frame after becoming active (snap scroll-derived state, D15) |
| `size` | `{w, h, dpr}` | Last applied size from the shared `ResizeObserver` |

**Instances and their "needs another frame" rule (reduced motion off)**:

| Ticker | Observed element | Continuous while visible? | Otherwise active while |
|---|---|---|---|
| hero | `#hero` | Yes (rotation) | — |
| field | `#purpose .band` **and** `#field`, registered once with a union rule (fixes R4) | No | focus/amplitude/step bars not settled |
| gifts | `#gifts` | Yes once exploded (bob) | explode or tilt not settled |
| careers | `#hudp` | Yes (turntable, see research D13) | morph not settled, log still typing |
| close | `#stage` | No | any block not settled |

**State transitions**:

```text
 inactive ──(enters view)──▶ entered ──(first frame, snap)──▶ animating ──(write() → false)──▶ settled
     ▲                                                          ▲                                │
     └───────────────(leaves view)──────────────────────────────┴──(input, resize, pref change)──┘
```

Under reduced motion every ticker returns `false` after one frame, so it draws once in
its final state and again only on resize or a visitor choice (FR-026).

---

## 5. FrameScheduler

| Field | Type | Meaning |
|---|---|---|
| `active` | `Set<Ticker>` | Visible tickers |
| `awake` | `Set<Ticker>` | Tickers that asked for another frame or were woken |
| `pending` | boolean | A `requestAnimationFrame` is already queued |
| `lastT` | ms | Previous frame time; `dt = min(50, now - lastT)` |

Contract: see [contracts/frame-loop.md](./contracts/frame-loop.md).

---

## 6. CanvasSurface

Applies to `#net`, `#field`, `#wire` (and the size of `#stage`).

| Field | Type | Rule |
|---|---|---|
| `w`, `h` | CSS px, rounded | From `ResizeObserver` content box |
| `dpr` | number | `min(2, devicePixelRatio)` (kept from today) |
| `pendingSize` | `{w,h,dpr} \| null` | Set by the observer, applied after 150ms with no further change |

**Validation**: the backing store is reallocated only when `w`, `h` or `dpr` differs from
the applied size (FR-009).

---

## 7. HeroGraph

| Field | Type | Meaning |
|---|---|---|
| `N` | 560 or 1150 | Node count; chosen by viewport width `< 700px` |
| `P` | `Float32Array(N*3)` | Node positions on the unit sphere |
| `E` | `Uint16Array` | Edge index pairs |
| `adj` | `number[][]` | Adjacency for signal ripples |
| scratch | `SX, SY, SZ, HOT` | Per-frame projected values, reused |

Built by `buildGraph(N)` with a reset seed, cached per `N`. Separate **HeroView** state
(`rotY`, `rotX`, `vY`, `vX`, `drag`, `rip`) is not part of the graph and survives a swap,
except `rip`, which is dropped because node indices change (research D7).

**HeroDrag transitions**:

```text
 idle ──pointerdown (capture)──▶ pressed ──move > 6px──▶ dragging
   ▲                               │                        │
   │  pointerup (moved < 6px) → signal()                    │
   └───────────── pointerup / pointercancel / lostpointercapture ─┘
```

---

## 8. FlexWord and FlexChar

| Entity | Field | Meaning |
|---|---|---|
| FlexWord | `el` (`.w` span) | `display:inline-block`, inline `width` in `em` at base weight |
| FlexChar | `el` (`.ch` span) | One glyph |
| | `x`, `y` | Page-coordinate centre from the `offsetLeft`/`offsetTop` chain (transform-proof) |
| | `r`, `base`, `min` | Radius and weight range from the parent `.flex` element |
| | `w` | Current damped weight |
| | `written` | Last weight written to `font-variation-settings` (writes skipped below a 3-unit change) |

**Measured when**: fonts ready, fonts `loadingdone`, or a `ResizeObserver` callback on a
`.flex` element or `<main>`. Never on a timer (FR-016). Under reduced motion or a coarse
pointer the list is empty and no weights are written.

---

## 9. LabCommit (persisted)

| Field | Type | Rule |
|---|---|---|
| key | `td.commit` in `localStorage` | Unchanged from today |
| value | `{"g": 0-4, "k": 0-4}` or `null` | If `GIFTS[g].actions[k]` does not exist, show the default prompt (spec edge case) |

**Transitions**: `null ⇄ {g,k}` on card click; choosing a different card replaces the
value. Only the affected cards update (FR-020).

---

## 10. PathsSelection

| Field | Type | Meaning |
|---|---|---|
| `selected` | 0-4 | Gift index with `aria-pressed="true"` |
| `preview` | 0-4 | Gift the wireframe is morphing toward (hover preview on fine pointers) |
| `logStart` | ms | Time the current log started typing |
| `logText` | string | Full text; visible length = `min(len, (now - logStart) * 0.25)` |
