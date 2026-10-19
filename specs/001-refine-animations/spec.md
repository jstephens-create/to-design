# Feature Specification: Refine Interactive Animations

**Feature Branch**: `001-refine-animations`

**Created**: 2026-10-18

**Status**: Draft

**Input**: User description: "Improve the interactive animations in my existing portfolio website. Preserve the current Swiss-industrial design, HTML structure, navigation, and GitHub Pages compatibility. First, inspect the existing animation behavior and identify opportunities to improve performance, responsiveness, and visual consistency."

**Supporting research**: [audit.md](./audit.md) records the inspection of the current animation
behavior. Finding IDs (P1, R2, V3, A1 and so on) are referenced below.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Smooth on any device (Priority: P1)

A visitor opens the shared link on a mid-range laptop or phone and scrolls from the hero
to the footer. Every section's animation plays without stutter, scrolling stays fluid,
and the device does not heat up or drain battery while the page sits open.

**Why this priority**: Stutter is the first thing a visitor notices, and it undercuts a
page whose message is responsible, careful design. It also affects every visitor.

**Independent Test**: Record a performance trace while scrolling the full page on a
throttled device profile, then leave the page idle. This can be compared directly against
the current version.

**Acceptance Scenarios**:

1. **Given** the page is loaded on a mid-range device, **When** the visitor scrolls the
   full page at a normal pace, **Then** no visible stutter occurs and scrolling never
   freezes. (Addresses P1, P3, P4, P6)
2. **Given** the visitor has stopped moving the pointer and is not scrolling, **When** no
   section on screen is animating by design, **Then** the page does no repeated drawing or
   layout work. (Addresses P2, P3)
3. **Given** the visitor is on a phone, **When** the browser's address bar shows or hides
   during scrolling, **Then** no canvas flashes, clears or resets. (Addresses P5)
4. **Given** the tab is in the background, **When** the visitor returns, **Then** every
   animation resumes from a sensible state with no jump. (Constitution Principle V)

---

### User Story 2 - Touch and screen changes behave correctly (Priority: P2)

A visitor on a phone or tablet taps, drags and scrolls through the page, and may rotate
the device. Interactions respond to the finger and then let go. Nothing stays stuck where
the finger lifted, and layouts adapt when the screen changes size.

**Why this priority**: Many visitors open the link on a phone (Constitution Principle IV).
Today, touch leaves effects stuck in place and the hero globe can jump.

**Independent Test**: On a touch device, tap and drag in the hero, the line field and the
closing stage, rotate the device, and observe each section.

**Acceptance Scenarios**:

1. **Given** a touch visitor taps in the line field or closing stage, **When** they lift
   their finger, **Then** the cursor-driven effect relaxes back to its idle state within
   about one second. (Addresses R2)
2. **Given** a touch visitor starts a horizontal drag on the hero globe, **When** the
   browser takes over the gesture as a vertical scroll, **Then** the drag ends cleanly and
   the next touch does not make the globe jump. (Addresses R3)
3. **Given** the Purpose band is on screen but the line field above it has scrolled away,
   **When** the visitor moves the pointer, **Then** the orange step bars still respond.
   (Addresses R4)
4. **Given** the visitor rotates a phone or resizes a window, **When** the layout
   settles, **Then** the hero globe, line field, wireframe and closing blocks redraw at the
   correct size and density, and the headline weight effect lines up with the cursor.
   (Addresses R5, R6)
5. **Given** two visitors on a 60Hz display and a 120Hz display, **When** they perform the
   same interaction, **Then** easing and momentum take the same real time to settle.
   (Addresses R1)

---

### User Story 3 - One consistent motion language (Priority: P3)

A visitor moves between sections and the motion feels like it comes from one designer:
the same rhythm of timing and easing, no flashes, no laggy or snapping controls. It
matches the precise, engineered character of the Swiss-industrial design.

**Why this priority**: The page is a portfolio. Inconsistent motion reads as unfinished
craft, which is one of the five gifts it argues for.

**Independent Test**: Hover and click every interactive control in turn (nav, tabs, gift
buttons, role cards, lab cards) and compare timing and easing side by side.

**Acceptance Scenarios**:

1. **Given** the visitor is in the Lab, **When** they commit to or release an action card,
   **Then** only that card changes state and the other cards do not fade or slide in again.
   (Addresses V2)
2. **Given** a fine-pointer visitor hovers a Lab card, **When** they move across it,
   **Then** the tilt follows the pointer as closely as the Gifts assembly tilt does.
   (Addresses V3)
3. **Given** a visitor hovers a tab or HUD nav link, **When** they move away, **Then** it
   returns to rest with the same smooth easing on both. (Addresses V4)
4. **Given** a visitor picks a different gift in Paths, **When** the roles and note
   change, **Then** they transition in step with the wireframe morph instead of swapping
   instantly. (Addresses V5)
5. **Given** any two transitions of the same kind (hover feedback, state change, entrance),
   **When** they are compared, **Then** they use the same duration and easing. (Addresses V1)

---

### User Story 4 - Calm page for reduced-motion visitors (Priority: P4)

A visitor who has turned on "reduce motion" in their operating system sees a calm page.
Everything is still readable and every control still works, but nothing drifts,
rotates, pushes or tilts on its own or in response to the cursor.

**Why this priority**: Required by Constitution Principle III. Today it is only partly
met, and the gap affects a small but important group of visitors.

**Independent Test**: Turn on reduced motion in the OS, load the page, and move the
pointer, scroll, and use every control.

**Acceptance Scenarios**:

1. **Given** reduced motion is on, **When** the visitor moves the pointer over the hero,
   line field, headlines or closing stage, **Then** nothing moves, warps or changes weight
   in response. (Addresses A1)
2. **Given** reduced motion is on, **When** the visitor scrolls, **Then** each animated
   figure appears in its final, readable state, and sections still reveal without sliding.
3. **Given** the page is open, **When** the visitor turns reduced motion on or off in the
   OS, **Then** the page follows the new setting without a reload. (Addresses R7)
4. **Given** reduced motion is on, **When** nothing changes on screen, **Then** the page
   does no repeated drawing.

---

### Edge Cases

- The pointer leaves the browser window mid-interaction: every cursor-driven effect
  relaxes to idle, and the custom cursor hides.
- A mouse and touch screen are used together on a hybrid laptop: the custom cursor shows
  for the mouse, and touch input does not leave stuck effects.
- The visitor clicks a HUD link or the skip link and the page jumps instead of scrolling
  smoothly: sections that were skipped over still show their final state, with no half-run
  entrance.
- Web fonts load late or fail: headline weight effects still line up with the text once
  fonts arrive, and the fallback fonts work without errors.
- Very narrow (320px) and very wide (2560px+) screens: canvases fill their space with no
  horizontal page scroll.
- Very fast scrolling past animated sections: no section is left frozen mid-entrance.
- The stored Lab commitment refers to an action that no longer exists: the page shows the
  default prompt (current behavior, keep it).

## Requirements *(mandatory)*

### Preservation constraints

- **FR-001**: The visual design MUST stay the same: colors, type, layout, section order,
  and the look of each animation at rest and in motion. Changes in this feature are
  refinements, not redesigns.
- **FR-002**: The HTML content and landmark structure MUST stay the same. Headings, text,
  section IDs, ARIA attributes and the skip link MUST NOT change. The one exception is
  adding attributes needed for the requirements below.
- **FR-003**: HUD navigation MUST keep the same links, labels, active-section highlighting
  and smooth in-page scrolling.
- **FR-004**: The site MUST remain a single self-contained page that GitHub Pages can
  serve from `main` / root. It MUST NOT add new external dependencies or build steps
  (Constitution Principle I).
- **FR-005**: Every existing interaction MUST still be available: globe drag and signal,
  line-field focus, assembly explode and tilt, gift branch selection, Lab tabs and commit,
  closing assembly, custom cursor, flexible headlines, scroll progress.

### Performance

- **FR-006**: Animation updates MUST avoid forcing the browser to recalculate layout more
  than once per frame. (P1)
- **FR-007**: When the pointer is still, the page is not scrolling, and no on-screen
  section is animating by design, the page MUST stop per-frame work. (P2, P3)
- **FR-008**: Sections MUST NOT create new drawing objects on every frame where they can
  be reused. (P4)
- **FR-009**: Resizing or address-bar changes MUST NOT clear or rebuild canvases until the
  size has actually changed and settled. (P5)
- **FR-010**: The headline weight effect MUST NOT make headline lines wrap differently or
  shift surrounding content. (P6)
- **FR-011**: The Paths console typing effect MUST run in step with the page's frame
  updates rather than on its own timer. (P7)

### Responsiveness

- **FR-012**: Easing, momentum and drift MUST be based on elapsed time, so they behave
  the same at any display refresh rate. (R1)
- **FR-013**: Cursor-driven effects MUST relax to idle when a touch ends or the pointer
  leaves the page. (R2)
- **FR-014**: Hero globe dragging MUST end cleanly when the browser cancels the gesture,
  and the next gesture MUST start from the current position. (R3)
- **FR-015**: Each animated area MUST run while any part of it is on screen and stop when
  all of it is off screen. (R4)
- **FR-016**: Headline weight positions MUST stay aligned with the text after reveals,
  font loading, resizing and layout changes, without relying on fixed delays. (R5)
- **FR-017**: The hero globe density MUST adjust when the viewport crosses the
  phone/desktop threshold. The globe's rotation MUST NOT reset when this happens. (R6)
- **FR-018**: The page MUST react to a change in the reduced-motion setting while open.
  (R7)

### Visual consistency

- **FR-019**: The page MUST define a small shared set of motion durations and easing
  curves, about three durations and two curves. Every transition MUST use that set. (V1)
- **FR-020**: Committing or releasing a Lab action MUST update only the affected cards,
  with no re-entrance of the grid. Switching tabs MAY still play the staggered entrance. (V2)
- **FR-021**: Lab card tilt MUST follow the pointer directly and ease back to rest on
  leave. (V3)
- **FR-022**: Magnetic controls (HUD nav, Lab tabs) MUST ease out when released, with the
  same feel as each other. (V4)
- **FR-023**: Paths role cards and note MUST transition when the selected gift changes,
  timed with the wireframe morph. (V5)
- **FR-024**: Unused animation styles MUST be removed. (V6)

### Reduced motion

- **FR-025**: With reduced motion on, cursor-driven displacement, tilt, drift and weight
  changes MUST be off. The custom cursor MAY still track the pointer without trailing
  easing. (A1)
- **FR-026**: With reduced motion on, every animated figure MUST render once in its final,
  readable state and redraw only when its size changes or the visitor makes a choice
  (for example, picking a gift).
- **FR-027**: With reduced motion on, hover and selection state changes MUST still be
  visible as instant changes in color or fill.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Scrolling the full page on a mid-range phone, or on a desktop browser at 4x
  CPU throttling, shows no frame longer than 50ms in a performance recording, outside the
  initial page load.
- **SC-002**: With the pointer still, the page idle and the hero scrolled off screen, the
  page uses less than 2% of one CPU core over 10 seconds.
- **SC-003**: Main-thread work while the visitor moves the pointer over the Gifts or
  Closing sections drops by at least 40% compared with commit 15429cc, measured with the
  same recording method.
- **SC-004**: On 60Hz and 120Hz displays, the same easing (for example, the Gifts
  assembly settling after a scroll stop) completes within 10% of the same duration.
- **SC-005**: On a touch device, 0 of 10 taps in the hero, line field or closing stage
  leave an effect stuck after one second.
- **SC-006**: 0 of 10 Lab commit or release clicks make other cards re-animate.
- **SC-007**: A side-by-side screenshot comparison of every section at rest, at 375px and
  1440px, shows no visible change from commit 15429cc.
- **SC-008**: With reduced motion on, moving the pointer across every section produces
  no movement other than the native or custom cursor itself.
- **SC-009**: Every CSS transition and scripted ease on the page uses one of the shared
  motion values. An inventory of the stylesheet finds 0 values outside the set.
- **SC-010**: The page stays under 100 KB and adds no new external requests
  (Constitution Principle V).

## Assumptions

- "Mid-range device" means a phone about 3 to 4 years old, or a desktop browser with 4x
  CPU throttling in its developer tools.
- The hero globe's continuous slow rotation, the gentle bob in the Gifts assembly and the
  slow turntable rotation of the Paths wireframe are intended, so they may keep the page
  drawing while those sections are on screen. The idle requirement (FR-007) applies when
  they are off screen or when reduced motion is on. (Wireframe added during planning,
  research D13; owner may reverse.)
- The Purpose line field and the closing blocks drift with scrolling rather than with time,
  so they hold still at rest. This is the one intended change to "the look in motion" under
  FR-001 (research D13).
- Keyboard access to the hero "send a signal" interaction (audit A2) is out of scope,
  because the signal is decorative and duplicates no content. It can be a later feature.
- Measured comparisons use commit 15429cc as the baseline.
- Exact motion values for FR-019 are chosen during planning from the values already in
  use, keeping the current feel. The likely set is fast about 0.2s, base about 0.45s,
  slow about 0.9s, with the existing `cubic-bezier(.2,.7,.2,1)` as the main curve.
- No content, copy, colors, fonts or section order change in this feature.
