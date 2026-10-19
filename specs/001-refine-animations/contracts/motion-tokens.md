# Contract: Motion Tokens

The shared motion vocabulary (FR-019, SC-009). Rationale: [research.md](../research.md) D9.

## CSS (on `:root`, next to the colour tokens)

```css
--t-fast: .25s;
--t-base: .5s;
--t-slow: .9s;
--ease-out: cubic-bezier(.2,.7,.2,1);
--ease: ease;
--stagger: 70ms;
```

## Allowed uses

| Kind of change | Duration | Curve |
|---|---|---|
| Hover colour, fill, border, opacity (nav, tabs, gift buttons, cards, roles, gifts parts, cursor ring size) | `--t-fast` | `--ease` |
| Movement feedback on hover (gift button indent) | `--t-fast` | `--ease-out` |
| State change, return to rest (tilt, magnetic), single-item entrance (Lab card, Paths roles and note), card glow | `--t-base` | `--ease-out` |
| Section reveal (`.rv`) | `--t-slow` | `--ease-out` |
| Lab card entrance offset | `calc(var(--i) * var(--stagger))` delay | — |

## Mapping from today's stylesheet (commit 15429cc)

| Rule | Today | New |
|---|---|---|
| `#hud a` opacity | .2s | `--t-fast --ease`; plus transform `--t-base --ease-out` (magnetic return) |
| `#cur .ring` size, background | .3s | `--t-fast --ease` |
| `.flex .ch` weight | .45s | removed; weight damped in JS with `T.base` |
| `.js .rv` opacity, transform | .9s | `--t-slow --ease-out` |
| `.wipe` | 1s, `cubic-bezier(.75,0,.2,1)` | removed (unused, FR-024) |
| `#asm .b, #asm .f` fill | .35s | `--t-fast --ease` |
| `.glist .num` background, colour | .3s | `--t-fast --ease` |
| `.gbtn` colour / padding | .25s / .35s | `--t-fast --ease` / `--t-fast --ease-out` |
| `.role` border / transform | .3s / .5s | `--t-fast --ease` / `--t-base --ease-out` |
| `.tabs button` background, colour | .2s | `--t-fast --ease`; plus transform `--t-base --ease-out` |
| `.card` background / transform / opacity | .35s / .5s / .6s | `--t-fast --ease` / `--t-base --ease-out` / `--t-base --ease-out` |
| `.card::before` opacity | .3s | `--t-fast --ease` |

## Tracking override

```css
.trk { transition-property: <all other properties of that element>; }  /* transform follows pointer directly */
```

Implementation detail: the simplest form is a per-component rule that repeats the
element's non-transform transitions, for example
`.card.trk{transition:background var(--t-fast) var(--ease),opacity var(--t-base) var(--ease-out)}`.

## JS mirror

```text
T = { fast, base, slow }  // seconds, read once from getComputedStyle(:root), fallback .25/.5/.9
```

All damping (`damp`, `decay`) and WAAPI `duration` values MUST come from `T`. WAAPI
`easing` MUST be the `--ease-out` string.

## Reduced motion

The existing rule `*{transition-duration:.01ms!important;animation:none!important}` stays.
WAAPI calls MUST be skipped when `state.reduce` is `true` (CSS does not cut them).
