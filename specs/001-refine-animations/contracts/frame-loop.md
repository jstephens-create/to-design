# Contract: Frame Loop and Tickers

Internal contract inside `index.html`'s script. Every animated section MUST follow it.
Design rationale: [research.md](../research.md) D1, D2, D3, D13, D15.

## Registration

```text
register(el | el[], { read, write, continuous? }) -> Ticker
```

- `el`: the element(s) whose visibility controls the ticker. With several elements the
  ticker is active while **any** of them is within the root margin (fixes audit R4).
  Root margin stays `120px`, except the hero, which stays `0px`.
- `read()`: MAY read layout (`getBoundingClientRect`, `clientWidth`, `innerHeight`,
  `scrollY`). MUST NOT write styles, attributes or canvas.
- `write(now, dt) -> boolean`: MAY write styles, attributes and canvas. MUST NOT read
  layout. Returns `true` if it needs another frame.
- `continuous(state) -> boolean` (optional): returns `true` when the ticker animates by
  design while visible (hero, gifts once exploded, careers). MUST return `false` when
  `state.reduce` is `true`.

## Frame order

Each frame, in this order:

1. `dt = min(50, now - lastT)`.
2. `read()` for every awake, active ticker.
3. Site-wide writes: cursor, flex weights, scroll progress (each only if its inputs changed
   or it is still easing).
4. `write(now, dt)` for every awake, active ticker. On a ticker's first frame after
   becoming active, `ticker.entered` is `true` and scroll-derived state snaps to target.
5. A ticker stays awake if `write` returned `true` or `continuous()` is `true`.
6. Request the next frame only if any ticker is awake, the cursor or flex weights are still
   easing, or new input arrived during this frame.

## Waking

`wake(ticker?)` marks one ticker (or all active tickers) awake and requests a frame if
none is pending. Called by:

| Event | Wakes |
|---|---|
| `pointermove`, `pointerdown`, `pointerup`, `pointercancel`, pointer leaving window | all active |
| `scroll` (passive) | all active |
| Applied resize (after 150ms settle) | the resized ticker, plus a flex re-measure |
| Reduced-motion or fine-pointer `change` | all active |
| Visitor choice (gift button, Lab tab, Lab commit) | the owning ticker |
| Ticker becomes visible | that ticker |

## Easing helper

```text
damp(current, target, duration_s, dt_ms) -> number
decay(value, duration_s, dt_ms) -> number
```

- `duration_s` MUST be one of `T.fast`, `T.base`, `T.slow` (see
  [motion-tokens.md](./motion-tokens.md)).
- Under reduced motion, callers pass the target straight through (no easing) for
  pointer-driven values.

## Guarantees

- At most one forced style/layout flush per frame (FR-006).
- Zero `requestAnimationFrame` callbacks while idle with no continuous ticker on screen
  (FR-007, SC-002).
- No `setInterval` or `setTimeout` drives visual change (FR-011, FR-016). Timers MAY
  debounce resize only.
- Hidden tabs: no frames (browser-native `rAF` pause); on return `dt` is capped at 50ms, so
  nothing jumps.
