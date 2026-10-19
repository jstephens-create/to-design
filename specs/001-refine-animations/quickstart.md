# Quickstart: Validating Refined Animations

How to prove the feature works. Expected behaviour is defined in
[contracts/ui-behavior.md](./contracts/ui-behavior.md); success criteria are in
[spec.md](./spec.md#success-criteria-mandatory).

## Prerequisites

- Chrome or Edge (Performance and Rendering panels), plus Firefox or Safari for a
  cross-browser pass.
- Python 3 (for a static server) or VS Code Live Server.
- A real phone for touch and rotation checks (iOS Safari and Android Chrome if possible).
- Optionally a 120Hz display, or Chrome's frame-rate override, for SC-004.

## Setup: serve the new page and the baseline side by side

From the repository root (PowerShell or Git Bash):

```sh
# baseline (commit 15429cc) in a throwaway worktree
git worktree add ../to-design-baseline 15429cc

# terminal 1: work in progress
python -m http.server 5500

# terminal 2: baseline
python -m http.server 5501 --directory ../to-design-baseline
```

New: http://localhost:5500/ · Baseline: http://localhost:5501/

To test on a phone, serve with `--bind 0.0.0.0` and open `http://<your-PC-IP>:5500/` on the
same network. Remove the worktree afterwards with `git worktree remove ../to-design-baseline`.

## Scenario checks

### 1. Performance (US1 · SC-001, SC-002, SC-003)

1. DevTools → Performance → CPU: **4x slowdown**. Record while scrolling from hero to
   footer at a normal pace. **Expect**: no task or frame over 50ms after load.
2. Scroll so the Purpose band or the Lab fills the viewport (no hero, Gifts or Paths on
   screen). Keep the pointer still, record 10s. **Expect**: no `Animation Frame Fired`
   entries after the first second; main-thread busy time < 2% (< 200ms of 10s).
3. Repeat the same 10s pointer-move recording over Gifts and over Close on both ports.
   Compare "Scripting + Rendering + Painting" totals. **Expect**: new ≤ 60% of baseline.
4. Rendering → **Paint flashing**. Hold still over Purpose. **Expect**: no repeating green
   flashes once settled.

### 2. Address bar and resize (US1-3, US2-4 · FR-009, FR-017)

- Phone: scroll down and up so the URL bar hides and shows. **Expect**: no canvas flash or
  blank frame.
- Rotate the phone, or drag a desktop window across 700px wide. **Expect**: hero redraws
  with the other node count, keeping its current angle; field, wireframe and closing
  blocks redraw at the new size; headline thinning lines up with the cursor.

### 3. Touch (US2 · SC-005)

- 10 taps each in the hero, line field and closing stage. **Expect**: every effect relaxes
  within 1s of lifting the finger.
- Start a horizontal drag on the globe, then swing into a vertical swipe. **Expect**: page
  scrolls, drag ends; next touch does not jump the globe.

### 4. Refresh-rate independence (US2-5 · SC-004)

Record the Gifts assembly settling after a scroll stop at 60Hz and at 120Hz (or with the
Rendering panel's frame-rate limiter at 30fps vs. unthrottled). **Expect**: settle times
within 10% of each other.

### 5. Motion consistency (US3 · SC-006, SC-009)

- Click 10 Lab cards (commit, release, switch commitment). **Expect**: only the affected
  cards change; no other card fades or slides; focus stays on the clicked card.
- Hover a Lab card and a Gifts part. **Expect**: tilt follows directly on both; card eases
  back on leave.
- Hover then leave a Lab tab and a HUD link. **Expect**: both ease back the same way.
- Select each gift in Paths. **Expect**: roles and note enter as the wireframe morphs.
- Inventory: search the `<style>` block for `transition` and `animation`, and the script
  for `.animate(` and damping calls. **Expect**: every duration and curve is a token from
  [contracts/motion-tokens.md](./contracts/motion-tokens.md); no `.wipe` rules remain.

### 6. Reduced motion (US4 · SC-008, FR-018)

1. Rendering → **Emulate CSS media feature prefers-reduced-motion: reduce**, with the page
   already open. **Expect**: page switches immediately, no reload needed.
2. Move the pointer across every section. **Expect**: nothing moves except the cursor.
3. Scroll the page. **Expect**: Gifts appears exploded, Close appears assembled, sections
   reveal without sliding.
4. Use every control by mouse and keyboard. **Expect**: state changes are visible as
   instant colour or fill changes.
5. Switch emulation off. **Expect**: motion returns without a reload.

### 7. Preservation (FR-001 to FR-005 · SC-007, SC-010)

- Screenshot every section at rest at 375px and 1440px on both ports (DevTools → device
  toolbar → "Capture full size screenshot"). **Expect**: no visible differences.
- Tab through the page. **Expect**: skip link, focus outlines, HUD links and all controls
  unchanged.
- Network panel. **Expect**: only `index.html` and Google Fonts requests.
- `(Get-Item index.html).Length` (PowerShell) or `wc -c index.html`. **Expect**: < 102400.

### 8. Edge cases

- Move the pointer out of the window mid-hover. **Expect**: effects relax; custom cursor
  hides.
- Click the skip link and each HUD link from the top. **Expect**: destination sections are
  fully in their scroll state.
- DevTools → Network → block `fonts.gstatic.com`, reload. **Expect**: no console errors;
  thinning still aligns.
- 320px and 2560px widths. **Expect**: no horizontal scroll; canvases fill their space.
- Set `localStorage['td.commit']='{"g":9,"k":9}'` and reload. **Expect**: default prompt.

## Constitution checklist (before each commit)

Run the Development Workflow list in `.specify/memory/constitution.md`: 320px and desktop
render, keyboard and focus, reduced motion, no new dependencies or hard-coded colours.
