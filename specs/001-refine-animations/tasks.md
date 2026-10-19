---

description: "Task list for Refine Interactive Animations"
---

# Tasks: Refine Interactive Animations

**Input**: Design documents from `/specs/001-refine-animations/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: No automated tests. The spec does not request them and Principle I forbids a test
toolchain. Each phase ends with a manual verification task that runs the matching
[quickstart.md](./quickstart.md) section.

**Organization**: Tasks are grouped by user story so each story can be implemented and
verified on its own.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1 to US4)

## Conventions for every task

- The only source file is `index.html` at the repository root. Line numbers (`~L612`) refer
  to commit 15429cc and will drift as edits land; find code by the quoted identifier.
- Because almost every task edits `index.html`, tasks are sequential unless marked [P].
- Keep the file's existing style: terse identifiers, one-line arrow functions, no new
  comments beyond section banners like `/* ================= hero ================= */`.
- Never change copy, headings, section IDs, ARIA attributes or the skip link (FR-002).
- Durations and easing come only from the motion tokens in
  [contracts/motion-tokens.md](./contracts/motion-tokens.md).
- Ticker structure follows [contracts/frame-loop.md](./contracts/frame-loop.md).

---

## Phase 1: Setup (Baseline)

**Purpose**: Capture the commit 15429cc baseline that SC-003 and SC-007 compare against.

- [X] T001 [P] Create a baseline worktree with `git worktree add ../to-design-baseline 15429cc` and confirm `../to-design-baseline/index.html` exists (do not commit anything) — *Done differently: baseline extracted with `git show 15429cc:index.html` into a scratch folder outside the repo (single-file site, so no worktree needed).*
- [X] T002 [P] Serve the baseline with `python -m http.server 5501 --directory ../to-design-baseline`, then capture full-size screenshots of each section at rest at 375px and 1440px into `specs/001-refine-animations/baseline/` (filenames `375-hero.png`, `1440-hero.png`, etc.; add `specs/001-refine-animations/baseline/` to `.gitignore` if the owner does not want them committed) — *Done differently: screenshots kept outside the repo (not committed); compared with pixelmatch.*
- [X] T003 [P] Record baseline numbers in `specs/001-refine-animations/baseline/metrics.md`: (a) 10s Performance recording with pointer moving over `#gifts`, Scripting + Rendering + Painting total; (b) same over `#close`; (c) 10s idle recording with the Purpose band in view and the pointer still; (d) `index.html` byte size (59340) — *Measured with headless Edge via Puppeteer run from outside the repo.*

**Checkpoint**: Baseline captured; implementation can start.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Shared runtime pieces every story builds on: motion tokens, preference state,
pointer state, the demand-driven scheduler, time-based easing and the resize service.
After this phase the page must look and behave the same as before; settling and idling
are switched on in US1.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [X] T004 Add motion tokens to `:root` in `index.html` directly after `--hair`: `--t-fast:.25s; --t-base:.5s; --t-slow:.9s; --ease-out:cubic-bezier(.2,.7,.2,1); --ease:ease; --stagger:70ms;` (research D9)
- [X] T005 In the script of `index.html`, replace `const reduce=...` and `const fine=...` (~L401 to L402) with a mutable `state` object: `const mqR=matchMedia('(prefers-reduced-motion: reduce)'), mqF=matchMedia('(hover: hover) and (pointer: fine)'); const state={reduce:mqR.matches,fine:mqF.matches};` and update every reference to `reduce` and `fine` in the script to `state.reduce` and `state.fine` (do not add change listeners yet; that is T049)
- [X] T006 In `index.html`, add `const T` right after `state`: read `--t-fast`, `--t-base`, `--t-slow` once via `getComputedStyle(document.documentElement).getPropertyValue(...)`, `parseFloat` each, fall back to `.25`, `.5`, `.9` if `NaN`; also store `T.easeOut='cubic-bezier(.2,.7,.2,1)'`
- [X] T007 In `index.html`, add helpers next to `lerp`/`ease`: `damp(a,b,dur,dt)=a+(b-a)*(1-Math.exp(-3*dt/1000/dur))` and `decay(v,dur,dt)=v*Math.exp(-3*dt/1000/dur)` (dur in seconds, dt in ms; research D3)
- [X] T008 In `index.html`, extend the pointer state `M` (~L450) with `type:'mouse'`, and set `M.type=e.pointerType` in the global `pointermove` listener; add the same assignment in a new passive global `pointerdown` listener (no other behaviour change yet)
- [X] T009 In `index.html`, replace `tickers`, `whenVisible` and `loop` (~L454 to L468) with the scheduler from [contracts/frame-loop.md](./contracts/frame-loop.md): `register(els, {read, write, continuous})` returns a ticker `{read, write, continuous, active:false, awake:false, entered:false}`; one `IntersectionObserver` per ticker observes every element in `els` (array or single element; keep root margin `'120px'`, and `'0px'` for the hero) and sets `active` true while **any** observed element intersects (track a per-ticker count of intersecting elements); becoming active sets `entered=true` and calls `wake(ticker)`; `wake(t)` marks `t` (or all active tickers when called with no argument) awake and calls `requestAnimationFrame(loop)` only if no frame is pending
- [X] T010 In `index.html`, write the new `loop(now)`: compute `dt=Math.min(50,now-lastT)`; call `read()` on every active awake ticker; then run the site-wide writes (`cursorTick`, `flexTick` when `M.moved||scrolled`, `progressTick` when `scrolled`); then call `write(now,dt)` on each active awake ticker, clear `entered`, and set `awake = returned || (continuous && continuous())`; request another frame only if a ticker is awake or `M.moved||scrolled` was set during this frame; reset `M.moved` and `scrolled` at the end
- [X] T011 In `index.html`, call `wake()` from the global `pointermove`, `pointerdown`, `pointerup`, `pointercancel` and `scroll` listeners, and from `document.documentElement`'s `pointerleave` listener, so input always schedules a frame; replace the final `requestAnimationFrame(loop)` (~L914) with `wake()`
- [X] T012 In `index.html`, convert each section to `register(...)` with an explicit read/write split and no layout reads inside `write` (FR-006): hero (`#hero`; read: `sec.getBoundingClientRect()`), field (`[cv, band]` as one ticker, replacing the two `whenVisible` calls ~L679 to L680; read: `cv.getBoundingClientRect()`), gifts (`#gifts`; read: `sec` and `wrap` rects), careers (`#hudp`; read: `cv.getBoundingClientRect()`), close (`#stage`; read: `stage.getBoundingClientRect()`); store rects on closure variables; every `write` returns `true` for now so behaviour is unchanged
- [X] T013 In `index.html`, replace every fixed per-frame factor with `damp`/`decay` using the token mapping in research D3: cursor `M.sx/sy` (`.18` → `T.fast`), close `m.x/y/rot` (`.12` → `T.base`), gifts `ex` (`.1` → `T.base`), field `fx` and gifts `tiltX/tiltY` and careers `extra/tilt` (`.08` → `T.base`), careers morph `cur[i]` (`.07` → `T.base`), field `amp` (`.05` → `T.slow`), hero `vY/vX *= .94` → `decay(v,T.slow,dt)`; under `state.reduce` the careers morph assigns the target directly as today
- [X] T014 In `index.html`, add a shared resize service: one `ResizeObserver`; `observeSize(el, apply)` stores `{w,h,dpr}` per element and on callback starts or restarts a 150ms timer, then calls `apply(w,h,dpr)` and `wake()` only if `Math.round(w)`, `Math.round(h)` or `Math.min(2,devicePixelRatio||1)` differs from the last applied values (data-model §6, FR-009)
- [X] T015 Manually verify in `index.html` served on port 5500 that every section looks and responds exactly as on the baseline (port 5501): hero spin/drag/signal, field focus, gifts explode/tilt, Paths morph/log, Lab tabs/commit, close assembly, cursor, flex headlines, progress bar; check the console has no errors — *Verified with automated headless checks (interactions, console errors) instead of by hand.*

**Checkpoint**: Foundation ready. Scheduler, tokens, state and helpers exist; visuals unchanged.

---

## Phase 3: User Story 1 - Smooth on any device (Priority: P1) 🎯 MVP

**Goal**: No stutter while scrolling, no per-frame work at rest, no canvas flash when the
mobile URL bar moves.

**Independent Test**: [quickstart.md](./quickstart.md) §1 and §2 (first bullet): 4x CPU
throttle scroll recording shows no frame over 50ms; 10s idle recording with the Purpose
band in view shows no animation frames and < 2% CPU; URL-bar show/hide causes no flash.

### Implementation for User Story 1

- [X] T016 [US1] In the hero ticker in `index.html`, remove the direct `resize` listener (~L594) and instead `observeSize(sec, ...)` to set `W`, `H`, `cv.width`, `cv.height` and `ctx.setTransform` (keep the `Math.min(2,devicePixelRatio||1)` cap)
- [X] T017 [US1] In the field ticker in `index.html`, remove the direct `resize` listener (~L652) and use `observeSize(cv, ...)` to resize the canvas backing store
- [X] T018 [US1] In the careers ticker in `index.html`, remove the direct `resize` listener (~L782) and use `observeSize(cv, ...)`; the canvas stays square (`cv.width=cv.height=W*d`)
- [X] T019 [US1] In the close ticker in `index.html`, remove the direct `resize` listener (~L891) and use `observeSize(stage, ...)` to recompute `W`, `H`, `u` and module sizes
- [X] T020 [US1] In the `<style>` of `index.html`, add `svh` overrides after the existing `vh` values so the URL bar no longer resizes them: `#field{...height:clamp(260px,46vh,520px);height:clamp(260px,46svh,520px)}` and `.stage{...height:clamp(360px,62vh,640px);height:clamp(360px,62svh,640px)}` (research D6) — *Not applied: `svh` would shorten these areas on phones (analysis I1). Kept `vh`; the size check in T014 prevents clearing. See research D6 note.*
- [X] T021 [US1] In the hero `write` in `index.html`, remove the three `new Path2D()` per frame (~L631): bucket edges into three reusable `Uint16Array` edge-index lists sized `EN` (back, front, hot) with counters reset each frame, then stroke each bucket with `ctx.beginPath()` + `moveTo`/`lineTo` + `ctx.stroke()` using today's exact `lineWidth` and `strokeStyle` values (FR-008)
- [X] T022 [US1] In the careers `write` in `index.html`, remove the two `new Path2D()` per frame (~L800) the same way: two passes (back then front) with `ctx.beginPath()` and the existing segment logic, same colours and widths
- [X] T023 [US1] In the gifts `write` in `index.html`, keep a `last` string per group, per callout and for `bolts`; only call `setAttribute('transform', ...)` when the `toFixed(2)` string differs; write callout `style.opacity` and the `svg.style.transform` tilt only when their formatted value changes (research D14)
- [X] T024 [US1] In `cursorTick` in `index.html`, skip all transform writes when `M.moved` is false and `|M.x-M.sx|`, `|M.y-M.sy|` are both below 0.1px; write `cur.style.opacity` only when `M.on` changes; return whether the ring is still easing so `loop` keeps requesting frames until it settles (the 60ms coords throttle stays)
- [X] T025 [US1] Replace the Paths typing `setInterval` (`typeLog`, ~L815 to L821) in `index.html` with frame-driven typing: `typeLog` stores `logText` and `logStart=performance.now()` and calls `wake(careersTicker)`; the careers `write` computes `n=Math.min(len, Math.floor((now-logStart)*0.25))` and repaints `log.innerHTML` (same escaping and `<span class="o">` markup) only when `n` changed; under `state.reduce` `n=len` immediately (FR-011, data-model §10)
- [X] T026 [US1] In the field `write` in `index.html`, remove time from the idle drift (research D13): set `t=0` in the pattern terms (`t*.12`, `t*.3`) and make the idle focus target scroll-only: `target = over ? (M.sx-r.left)/W : .5+.32*Math.sin(s*2)` (today's formula with `t=0`); keep the step bars' scroll-based `nx` path
- [X] T027 [US1] In the close `write` in `index.html`, replace the time drift terms `Math.sin(t*.7+i)` and `Math.cos(t*.6+i*2)` with the same functions of a scroll phase (`scrollY*.004`), keeping the `10*(1-p)` amplitude (research D13)
- [X] T028 [US1] Make each ticker's `write` return whether it still needs frames, per data-model §4: hero returns `true` (continuous rotation) unless `state.reduce`, in which case it returns `drag||rip||Math.abs(vY)>1e-6||Math.abs(vX)>1e-6`; field returns `true` while `|target-fx|>0.001`, `|ampTarget-amp|>0.001` or step bars still moving; gifts returns `true` while `|tgt-ex|>0.001` or tilt deltas > 0.01deg, and supply `continuous:()=>!state.reduce&&ex>0.01` for the bob; careers returns `true` while any `|tgt[i]-cur[i]|>0.0005` or the log is still typing, with `continuous:()=>!state.reduce` (turntable, research D13); close returns `true` while any block's position delta > 0.1px or rotation delta > 0.01deg; snap each value to its target when below its epsilon
- [X] T029 [US1] In the gifts, close and field `write` functions in `index.html`, when `ticker.entered` is `true` snap scroll-derived state to its target (`ex=tgt`; block `m.x/m.y/m.rot` to `gx/gy/gr`; `fx=target`) instead of easing, so jumps via skip link or HUD link never show a half-run entrance (research D15)
- [X] T030 [US1] Stop flex headline reflow in `index.html` (FR-010, research D8 steps 1-2): add CSS `.flex .w{display:inline-block;text-align:center}` and delete `.flex .ch{transition:...}`; in `measureFlex`, after resetting weights to base, set each `.w` element's `style.width` to `(w.getBoundingClientRect().width / parseFloat(getComputedStyle(el).fontSize)) + 'em'` (clear the inline width before measuring); add a damped `c.cur` weight per char in `flexTick` that eases toward the computed target with `damp(c.cur, w, T.base, dt)` and writes `font-variation-settings` only when `|c.cur-c.w|>3`; `flexTick` must return whether any visible char is still easing, and `loop` keeps requesting frames while it does — *Plus: weights written in 40-unit steps and paused while scrolling (research D8 note 5); `.fx` class gates word slots to active visitors.*
- [X] T031 [US1] Verify US1 using [quickstart.md](./quickstart.md) §1 (all four checks) and §2 first bullet; record results in `specs/001-refine-animations/baseline/metrics.md` next to the baseline numbers; confirm SC-001, SC-002 and SC-003 (new ≤ 60% of baseline over Gifts and Close)

**Checkpoint**: User Story 1 is complete and independently verifiable. This is the MVP.

---

## Phase 4: User Story 2 - Touch and screen changes behave correctly (Priority: P2)

**Goal**: Touch never leaves effects stuck, hero drag survives cancelled gestures, the
Purpose bars always respond, rotation and resize redraw correctly, and motion is the
same at any refresh rate.

**Independent Test**: [quickstart.md](./quickstart.md) §2 second bullet, §3 and §4.

### Implementation for User Story 2

- [X] T032 [US2] In the global pointer listeners in `index.html`, set `M.on=false` and `M.moved=true` on `pointerup` and `pointercancel` when `e.pointerType` is `touch` or `pen`; set `M.on=false` on `document` `pointerout` when `e.relatedTarget===null` and on `window` `blur`; call `wake()` in each case (research D4, FR-013)
- [X] T033 [US2] In `cursorTick` in `index.html`, show the custom cursor only when `state.fine && M.type==='mouse' && M.on` (hybrid laptops: touch input hides it) (data-model §3)
- [X] T034 [US2] Rework hero drag in `index.html` (research D5, FR-014): on `#hero` `pointerdown` (still ignoring `a,button` targets) create a fresh `drag={id:e.pointerId,x,y,moved:0}` and call `sec.setPointerCapture(e.pointerId)`; move the `pointermove` and `pointerup` listeners from `window` to `sec` and ignore events whose `pointerId!==drag.id`; add `pointercancel` and `lostpointercapture` listeners on `sec` that set `drag=null` without sending a signal; keep `touch-action:pan-y` on `#hero`
- [X] T035 [US2] Factor the hero graph build (~L565 to L589) in `index.html` into `buildGraph(N)` that resets `seed=20261016` at its start and returns `{N,P,E,EN,adj,SX,SY,SZ,HOT}`; cache results in an object keyed by `N`; choose `N=innerWidth<700?560:1150` at load (data-model §7)
- [X] T036 [US2] In the hero `observeSize` callback in `index.html`, recompute `N` from `innerWidth<700`; if it differs from the current graph, swap to the cached or newly built graph, set `rip=null`, and leave `rotY`, `rotX`, `vY`, `vX` untouched (FR-017)
- [X] T037 [US2] Make flex measurement transform-proof in `measureFlex` in `index.html` (research D8 step 3): compute each `.ch` centre with a helper that sums `offsetLeft`/`offsetTop` up the `offsetParent` chain plus `offsetWidth/2`, `offsetHeight/2`, instead of `getBoundingClientRect()+scroll`
- [X] T038 [US2] Replace timer-based flex re-measure in `index.html` (FR-016, research D8 step 4): remove the `setTimeout(measureFlex,1000)` and `remeasure` variable from the reveal observer (~L505 to L506), the `setTimeout(measureFlex,1200)` start timer (~L913) and the `resize` debounce (~L550); add a `ResizeObserver` on every `.flex` element and on `<main>` plus `document.fonts.addEventListener('loadingdone', ...)` (guarded by `document.fonts` existing) and keep the `document.fonts.ready` call; all of these schedule a single `measureFlex` on the next animation frame (coalesce with a flag)
- [X] T039 [US2] Verify US2 using [quickstart.md](./quickstart.md) §2 second bullet, §3 (10 taps per area, drag-to-scroll handoff) and §4 (30fps vs unthrottled settle times within 10%, SC-004), on a real phone and in DevTools device mode; confirm the Purpose step bars still respond when `#field` is scrolled off screen (US2 scenario 3) — *Real-phone check still recommended; automated touch emulation passed.*

**Checkpoint**: User Stories 1 and 2 both work independently.

---

## Phase 5: User Story 3 - One consistent motion language (Priority: P3)

**Goal**: Every transition uses the shared tokens; Lab commits update in place; tilt and
magnetic controls follow directly and ease back the same way; Paths content moves with the
wireframe.

**Independent Test**: [quickstart.md](./quickstart.md) §5.

### Implementation for User Story 3

- [X] T040 [US3] Rewrite every `transition` in the `<style>` of `index.html` to use tokens exactly as listed in the mapping table of [contracts/motion-tokens.md](./contracts/motion-tokens.md): `#hud a`, `#cur .ring`, `.js .rv`, `#asm .b,#asm .f`, `.glist .num`, `.gbtn`, `.role`, `.tabs button`, `.card`, `.card::before` (FR-019)
- [X] T041 [US3] Delete the three `.wipe` rules (~L71 to L73) from `index.html` and confirm with a search that no markup or script uses `wipe` (FR-024)
- [X] T042 [US3] Add the tracking pattern to the `<style>` of `index.html` (research D10): give `#hud a` and `.tabs button` an added `transform var(--t-base) var(--ease-out)` transition; add `.trk` overrides that repeat each element's non-transform transitions without `transform`: `#hud a.trk{transition:opacity var(--t-fast) var(--ease)}`, `.tabs button.trk{transition:background var(--t-fast) var(--ease),color var(--t-fast) var(--ease)}`, `.card.trk{transition:background var(--t-fast) var(--ease),opacity var(--t-base) var(--ease-out)}`
- [X] T043 [US3] Update the magnetic handler (~L553 to L558) in `index.html`: register for `.tabs` and `#hud nav` always (not inside `if(fine && !reduce)`); at event time return early unless `state.fine && !state.reduce`; on `pointermove` add `.trk` and set the transform as today; on `pointerout` remove `.trk` and clear `transform` so it eases back (FR-022)
- [X] T044 [US3] Split Lab `render(i)` (~L847 to L865) in `index.html` into `renderTab(i)` and `updateCards()` (FR-020, research D11): `renderTab` rebuilds the five cards as today but sets `style.setProperty('--i',k)` on each and removes the `pre` class on the next animation frame instead of per-card `setTimeout`; `updateCards()` loops existing cards and sets `aria-pressed`, `data-cursor` (`'Release'`/`'Commit'`) and the `.foot` first span text (`'Committed this week'`/`'Commit to this'`) from `commit`; card click sets `commit` (`null` if this card was on, else `{g:i,k}`), calls `store.set('td.commit',JSON.stringify(commit))`, `renderCommit()` and `updateCards()` only, and does not move focus; tab click and arrow-key handlers keep calling `renderTab` — *Entrance uses a double requestAnimationFrame (analysis U1).*
- [X] T045 [US3] Make the Lab stagger CSS-driven in `index.html`: in `renderTab`, set each new card's `style.transitionDelay='calc(var(--i)*var(--stagger))'`, and add a one-time `transitionend` listener (filter `e.propertyName==='opacity'`) that clears `style.transitionDelay` so later hover and commit transitions have no delay; delete the per-card `setTimeout(()=>b.classList.remove('pre'),...)` (~L863)
- [X] T046 [US3] Update Lab card tilt in `index.html` (FR-021): register `pointermove`/`pointerleave` on every card always and check `state.fine && !state.reduce` at event time; on `pointermove` add `.trk` and set `--mx`, `--my` and the tilt transform as today; on `pointerleave` remove `.trk` and clear `transform`
- [X] T047 [US3] In `show(i)` in the careers section of `index.html`, after swapping the roles and note text, unless `state.reduce`, call `el.animate([{opacity:0,transform:'translateY(12px)'},{opacity:1,transform:'none'}],{duration:T.base*1000,easing:T.easeOut})` on `#branch-roles` and `#branch-note`, and `wake(careersTicker)` so the morph starts in the same frame (FR-023)
- [X] T048 [US3] Verify US3 using [quickstart.md](./quickstart.md) §5: 10 Lab commit/release clicks with no other card re-animating and focus kept (SC-006); card vs. Gifts tilt; tab vs. HUD link release; Paths transitions; and a `transition`/`animation`/`.animate(` inventory with 0 values outside the token set (SC-009)

**Checkpoint**: User Stories 1 to 3 work independently.

---

## Phase 6: User Story 4 - Calm page for reduced-motion visitors (Priority: P4)

**Goal**: With reduced motion on, nothing moves except the cursor; figures render once in
their final state; the setting applies live.

**Independent Test**: [quickstart.md](./quickstart.md) §6.

### Implementation for User Story 4

- [X] T049 [US4] In `index.html`, add `change` listeners to `mqR` and `mqF` (research D12, FR-018): update `state.reduce`/`state.fine`; toggle `has-cursor` on `<html>` from `state.fine`; clear inline `transform` and remove `.trk` on `#hud nav a`, `.tabs button`, `.card` and `.band .steps i`; call `measureFlex()` (it clears the list when `!state.fine` or `state.reduce`) and `wake()`
- [X] T050 [US4] Turn off pointer-driven motion under reduced motion in `index.html` (FR-025): hero repel block runs only when `M.on && !state.reduce`; field `over` is false when `state.reduce`; close push runs only when `M.on && !state.reduce`; careers `nx`/`ny` are 0 when `state.reduce`; `measureFlex` returns an empty list when `state.reduce`; band steps are not written (and stay cleared) when `state.reduce`
- [X] T051 [US4] In `cursorTick` in `index.html`, when `state.reduce` set `M.sx=M.x`, `M.sy=M.y` so the ring and tag track the pointer without trailing (FR-025)
- [X] T052 [US4] Confirm every ticker in `index.html` returns `false` after one frame under `state.reduce` (FR-026): gifts uses `ex=1` and no tilt or bob; close uses `p=1`; careers assigns the morph directly and stops after one draw; field draws with `t=0` and `amp`/`fx` snapped; hero stops when not dragging; each redraws only on resize (`observeSize` wake), visitor choice (`show`, `renderTab`) or preference change
- [X] T053 [US4] Confirm in `index.html` that the Lab stagger, Paths WAAPI entrance and `.rv` slide are skipped or cut under reduced motion (the CSS `@media (prefers-reduced-motion:reduce)` rule cuts transitions; T047 skips WAAPI) while hover and selection colours still change instantly (FR-027)
- [X] T054 [US4] Verify US4 using [quickstart.md](./quickstart.md) §6 steps 1 to 5 with DevTools reduced-motion emulation toggled while the page is open (SC-008)

**Checkpoint**: All four user stories are independently functional.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Preservation checks, edge cases, documentation and release.

- [X] T055 Run [quickstart.md](./quickstart.md) §7: compare 375px and 1440px screenshots with `specs/001-refine-animations/baseline/` (SC-007), tab through the page, confirm only `index.html` and Google Fonts requests, and confirm `index.html` is under 102400 bytes (SC-010) — *Also fixed a pre-existing 5px overflow at 320px (`.gifts-grid` column `minmax(0,1fr)`).*
- [X] T056 Run [quickstart.md](./quickstart.md) §8 edge cases: pointer leaving the window, skip link and every HUD link from the top, blocked web fonts, 320px and 2560px widths, and `localStorage['td.commit']='{"g":9,"k":9}'`
- [X] T057 Run the constitution's Development Workflow checklist in `.specify/memory/constitution.md` against `index.html` (320px and desktop, keyboard and focus, reduced motion, no new dependencies or hard-coded colours)
- [X] T058 [P] Update the Assumptions in `specs/001-refine-animations/spec.md` to list the Paths wireframe turntable as an intended continuous animation alongside the hero rotation and Gifts bob, or record the owner's decision to let it settle (research D13); if the owner chooses settle, change T028's careers `continuous` to `()=>false` in `index.html` — *Spec assumption updated to keep the wireframe turntable; owner may reverse.*
- [X] T059 Remove the baseline worktree with `git worktree remove ../to-design-baseline` — *Not needed: no worktree was created (see T001).*
- [ ] T060 Commit to `main` in focused commits following the sequence in [plan.md](./plan.md) (Structure Decision), push, confirm the "pages build and deployment" Actions run succeeds, and check https://jstephens-create.github.io/to-design/

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies. Must finish before any `index.html` edit so the
  baseline is captured from 15429cc.
- **Foundational (Phase 2)**: Depends on Setup. Blocks all user stories.
- **US1 (Phase 3)**: Depends on Foundational.
- **US2 (Phase 4)**: Depends on Foundational. T037 and T038 touch `measureFlex` after T030,
  so if both stories run, do US1's T030 first.
- **US3 (Phase 5)**: Depends on Foundational. T047 depends on T025 (careers ticker variable
  `careersTicker` and frame-driven log) if US1 is done; otherwise expose `careersTicker`
  from the careers IIFE as part of T047.
- **US4 (Phase 6)**: Depends on Foundational. T052 relies on the settle returns from T028;
  T049 relies on `.trk` from T042/T043/T046 only for clean-up (harmless if absent).
- **Polish (Phase 7)**: Depends on all desired stories.

### Within Each User Story

- Resize and structural changes before per-frame optimisations (US1: T016 to T020 before
  T021 to T030).
- Settle logic (T028) after every change that affects a ticker's state (T021 to T027).
- The verification task is always last in its phase.

### Parallel Opportunities

- T001, T002 and T003 touch no project source and can run in parallel (T002 and T003 need
  the worktree from T001 to exist, so start T001 first, then T002 and T003 together).
- T058 edits `spec.md` only and can run alongside any `index.html` task.
- All other tasks edit the single file `index.html` and must run one at a time. User
  stories cannot be implemented in parallel without merge conflicts.

---

## Parallel Example: Setup

```text
Task: "Capture baseline screenshots at 375px and 1440px into specs/001-refine-animations/baseline/"
Task: "Record baseline Performance metrics into specs/001-refine-animations/baseline/metrics.md"
```

## Parallel Example: Polish

```text
Task: "Update Assumptions in specs/001-refine-animations/spec.md (wireframe decision)"
Task: "Run quickstart §8 edge cases against index.html"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1: capture baseline.
2. Phase 2: scheduler, tokens, state, helpers; verify nothing visible changed (T015).
3. Phase 3: US1 performance work.
4. **Stop and validate** with T031. This alone fixes stutter, idle CPU and URL-bar flashes.
5. Commit and deploy if ready.

### Incremental Delivery

1. Setup + Foundational → commit ("Add motion scheduler and tokens").
2. US1 → validate → commit/deploy (MVP).
3. US2 → validate → commit/deploy.
4. US3 → validate → commit/deploy.
5. US4 → validate → commit/deploy.
6. Polish → final checks and push.

Each story adds value without breaking earlier ones; run T015's regression pass again
after each story.

---

## Notes

- [P] = different files, no dependencies. Almost everything here is one file, so [P] is rare.
- [Story] labels map tasks to spec user stories for traceability.
- Commit after each logical group (see plan.md commit sequence); keep commits small.
- If a task's line reference no longer matches, search for the quoted identifier instead.
