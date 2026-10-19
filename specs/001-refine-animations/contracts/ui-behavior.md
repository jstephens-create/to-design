# Contract: Visitor-Facing Behaviour and Invariants

What a visitor and the published page can rely on after this feature. Anything not listed
here keeps its behaviour from commit 15429cc.

## Invariants (MUST NOT change)

| Area | Invariant | Spec |
|---|---|---|
| Markup | Headings, copy, section IDs (`hero`, `purpose`, `gifts`, `careers`, `lab`, `close`), `aria-*`, skip link | FR-002 |
| Additions allowed | Inline `style` written by script, classes `.trk`, CSS custom property `--i` on Lab cards | FR-002 exception |
| Navigation | Same five HUD links and labels, `.on` highlight via the existing section observer, smooth in-page scroll | FR-003 |
| Hosting | One `index.html`, inline CSS/JS, only Google Fonts external, served from `main` / root | FR-004 |
| Storage | `localStorage['td.commit']` = `{"g":n,"k":n}` or `null` | data-model §9 |
| Look at rest | No visible difference at 375px and 1440px | SC-007 |
| Size | `index.html` < 100 KB | SC-010 |

## Per-section behaviour

| Section | Pointer (fine, motion on) | Touch | Reduced motion |
|---|---|---|---|
| Hero globe | Nodes repel near the cursor; drag spins with momentum; click sends a signal | Horizontal drag spins; vertical swipe scrolls and cancels drag cleanly; tap sends signal; repel relaxes ≤ 1s after lift | Static globe at current angle; drag still turns it without momentum; signal ripple is instant; no repel |
| Purpose field | Lines thicken near the cursor; step bars shift with the cursor | Thicken on touch, relax ≤ 1s after lift | Drawn once in rest state; bars still |
| Purpose band bars | Respond while the band is on screen, even when the field above is off screen | Same | Still |
| Gifts assembly | Explodes with scroll; tilts toward cursor; parts bob gently once exploded | Explodes with scroll; no tilt | Shown fully exploded; no tilt, no bob |
| Paths | Hover previews the morph; click selects; roles and note fade/slide in with the morph; log types | Tap selects; same transitions | Instant swap; log appears complete; wireframe drawn once per change |
| Lab | Tabs and nav links follow the pointer magnetically and ease back; cards tilt directly and ease back; glow follows | Tap commits/releases | No magnetism, no tilt; instant colour change |
| Lab commit | Only clicked card (and any previously committed card) changes; focus stays on it | Same | Same, instant |
| Close stage | Blocks assemble with scroll; cursor pushes blocks | Push relaxes ≤ 1s after lift | Blocks shown assembled; no push |
| Headlines (`.flex`) | Weight thins near the cursor; no line rewrap or content shift | Not active | Not active |
| Custom cursor | Shown for a mouse on fine-pointer devices; hides when the pointer leaves the window | Hidden; native behaviour | Tracks the pointer without trailing |

## Cross-cutting

- Easing settles in the same real time at 30, 60 and 120Hz (within 10%, SC-004).
- Rotating a phone or resizing a window redraws every figure at the right size and
  density without resetting the globe's rotation (FR-017).
- Turning reduced motion on or off in the OS applies immediately (FR-018).
- Arriving at a section by skip link, HUD link or fast scroll shows it in its correct
  scroll state, never half-entered.
