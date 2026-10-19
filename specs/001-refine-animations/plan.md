# Implementation Plan: Refine Interactive Animations

**Branch**: `001-refine-animations` (feature ID; work lands on `main` per the constitution's workflow) | **Date**: 2026-10-18 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-refine-animations/spec.md`

## Summary

Refine every animation in `index.html` for performance, responsiveness and consistency
without changing how the page looks. The approach (see [research.md](./research.md)):

- Replace the always-on `requestAnimationFrame` loop with a demand-driven scheduler whose
  tickers read layout first and write second, and stop once settled (D1, D2, D13).
- Make every ease and momentum time-based and tie it to a shared set of motion tokens
  that CSS and JS both use (D3, D9).
- Fix touch, drag-cancel, resize and density issues with pointer-type tracking, pointer
  capture and a debounced `ResizeObserver` (D4 to D7).
- Stop headline reflow by locking word widths and measuring transform-proof positions on
  real events instead of timers (D8).
- Unify tilt and magnetic behaviour, update Lab cards in place, animate Paths content with
  the morph, and make reduced motion a live, complete setting (D10 to D12).

## Technical Context

**Language/Version**: HTML5, CSS3 (custom properties, `svh`), ES2017+ JavaScript inline in
one `<script>` (IIFE, `"use strict"`)

**Primary Dependencies**: None. Browser APIs only: `requestAnimationFrame`,
`IntersectionObserver`, `ResizeObserver`, Pointer Events (incl. pointer capture),
`matchMedia` change events, Web Animations API, CSS Font Loading API, Canvas 2D,
inline SVG. Google Fonts remains the only external resource.

**Storage**: `localStorage` key `td.commit` (unchanged)

**Testing**: Manual and DevTools-based, no test runner (Principle I). Scenarios in
[quickstart.md](./quickstart.md), baseline comparisons against a worktree of commit 15429cc

**Target Platform**: Current evergreen browsers (Chrome/Edge, Firefox, Safari incl. iOS),
desktop and mobile; served statically by GitHub Pages from `main` / root

**Project Type**: Single-file static website (portfolio piece)

**Performance Goals**: No frame > 50ms while scrolling at 4x CPU throttle (SC-001); < 2%
of one core when idle with no continuous section on screen (SC-002); ≥ 40% less main-thread
work during pointer movement over Gifts and Close (SC-003); same settle time within 10% at
60 and 120Hz (SC-004)

**Constraints**: No visual change at rest (SC-007); markup, IDs, ARIA and navigation
unchanged (FR-002, FR-003); no new dependencies or build steps (FR-004); `index.html` under
100 KB (currently 59 KB, SC-010)

**Scale/Scope**: One file, ~920 lines; six sections, five section animators plus
site-wide cursor, flex type, reveal, magnetic controls and progress bar; 27 functional
requirements (FR-001 to FR-027) tracing to 22 audit findings

All Technical Context items are resolved; no NEEDS CLARIFICATION remain. One judgement call
for the owner is flagged in research D13 (whether the Paths wireframe keeps turning at
rest).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Gate | Pre-research | Post-design |
|---|---|---|---|
| I. Single File, No Build | All work stays in inline CSS/JS of `index.html`; no libraries, bundlers or new external requests | PASS | PASS: only browser-native APIs (D1 to D12); verification uses DevTools, not a test toolchain (D16) |
| II. Every Section Serves the Argument | No new decoration; section order, gifts and HTML copy unchanged; animations still reinforce their content | PASS | PASS: no new effects; drift in the field and close becomes scroll-driven rather than removed (D13); dead `.wipe` CSS removed (FR-024) |
| III. Accessible to Everyone | Landmarks, headings, skip link, focus outline, keyboard operation, `aria-pressed`/`aria-selected` preserved; reduced motion disables non-essential motion; custom cursor only for fine pointers | PASS | PASS and improved: reduced motion becomes live and complete (D12); Lab commit keeps focus on the clicked card (D11); cursor limited to mouse input on fine-pointer devices (D4) |
| IV. Responsive at Every Width | 320px to large desktop, no horizontal scroll, `--gut` gutters, 44px targets | PASS | PASS and improved: `svh` heights stop URL-bar jumps (D6); density and canvases follow size changes (D6, D7); no layout widths changed |
| V. Light Footprint | < 100 KB; loops pause off screen and when hidden | PASS | PASS and improved: loop stops at rest (D1); per-frame allocations removed (D14); estimated net growth ≈ +4 to +6 KB, staying well under 100 KB |
| Design System Constraints | Colours from tokens, three font families, surface classes, `.inner` width | PASS | PASS: adds motion tokens beside the colour tokens; no new colours or fonts. Existing hard-coded colours in canvas/JS are out of scope and untouched |

**Result**: No violations. Complexity Tracking is empty.

## Project Structure

### Documentation (this feature)

```text
specs/001-refine-animations/
├── spec.md              # Feature specification (/speckit-specify)
├── audit.md             # Animation audit of commit 15429cc (input to spec)
├── plan.md              # This file
├── research.md          # Phase 0: decisions D1 to D16
├── data-model.md        # Phase 1: runtime state and transitions
├── quickstart.md        # Phase 1: validation scenarios
├── contracts/
│   ├── frame-loop.md    # Ticker registration, frame order, waking, guarantees
│   ├── motion-tokens.md # Shared durations/curves and their mapping
│   └── ui-behavior.md   # Visitor-facing behaviour and preservation invariants
├── checklists/
│   └── requirements.md  # Spec quality checklist
└── tasks.md             # Phase 2 (/speckit-tasks, not created here)
```

### Source Code (repository root)

```text
index.html            # The only file changed
├── <style>
│   ├── :root               + motion tokens (D9)
│   ├── #hud, #cur          token transitions; magnetic return (D10)
│   ├── .flex .w / .ch      word slots, remove per-char transition (D8)
│   ├── .rv, .wipe          tokenised reveal; remove .wipe (FR-024)
│   ├── #field, .stage      svh height overrides (D6)
│   ├── .gbtn, .role        tokenised transitions
│   ├── .tabs, .card, .trk  tracking pattern, stagger delay (D10, D11)
│   └── @media reduce       unchanged rule
└── <script>
    ├── prefs + tokens      live matchMedia, T from CSS (D9, D12)
    ├── pointer state       pointerType, leave/cancel handling (D4)
    ├── scheduler           register/wake, read→write order, damp/decay (D1 to D3)
    ├── resize service      shared debounced ResizeObserver (D6)
    ├── cursor, progress    write-on-change (D14)
    ├── flex type           em word slots, offset-chain measure, event re-measure (D8)
    ├── magnetic            .trk pattern, reduce check at event time (D10)
    ├── hero                buildGraph(N) cache, pointer capture, Path2D-free draw (D5, D7, D14)
    ├── field               single union ticker, scroll-phase drift (D13, R4)
    ├── gifts               write-on-change, snap on enter (D14, D15)
    ├── careers             frame-driven log, WAAPI role/note entrance (D11)
    ├── lab                 renderTab / updateCards split, tilt via .trk (D10, D11)
    └── close               scroll-phase drift, snap on enter (D13, D15)

inspo/                # References only; not touched
README.md             # Not touched
```

**Structure Decision**: Single static file per Principle I. All changes are confined to
`index.html`; the script keeps its current order of sections so diffs stay reviewable.
Suggested commit sequence (one concern per commit, per the Development Workflow):

1. Motion tokens and CSS transition mapping; remove `.wipe` (FR-019, FR-024).
2. Scheduler, `damp`/`decay`, read/write split for all tickers (FR-006, FR-007, FR-012).
3. Pointer state, hero drag capture, field ticker union (FR-013 to FR-015).
4. Resize service, `svh` overrides, hero density swap (FR-009, FR-017).
5. Allocation and write-on-change pass (FR-008, P2).
6. Flex type rework (FR-010, FR-016).
7. Lab and Paths consistency (FR-011, FR-020 to FR-023).
8. Live reduced motion and final reduced-motion behaviour (FR-018, FR-025 to FR-027).

Each commit is verified with the relevant [quickstart.md](./quickstart.md) section before
the next.

## Risks

| Risk | Mitigation |
|---|---|
| Word-slot locking (D8) leaves a visible gap at word ends while thinning | Slots are centred; check at 375px and 1440px in quickstart §5 and §7; fall back to left alignment if centring looks off |
| Scroll-phase drift (D13) reads as "frozen" to the owner | Drift still moves on every scroll; owner reviews Purpose and Close in motion before merge |
| `svh` unsupported in an old browser | `vh` declaration stays first as fallback, same as `#hero` today |
| Demand-driven loop misses a wake source and leaves a section stale | Wake table in [contracts/frame-loop.md](./contracts/frame-loop.md); quickstart §2, §6 and §8 exercise every source |

## Complexity Tracking

> Not applicable: the Constitution Check has no violations.
