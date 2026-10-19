<!--
Sync Impact Report
- Version change: template (unversioned) → 1.0.0
- Modified principles: all five template slots filled for the first time
  - [PRINCIPLE_1_NAME] → I. Single File, No Build
  - [PRINCIPLE_2_NAME] → II. Every Section Serves the Argument
  - [PRINCIPLE_3_NAME] → III. Accessible to Everyone (NON-NEGOTIABLE)
  - [PRINCIPLE_4_NAME] → IV. Responsive at Every Width
  - [PRINCIPLE_5_NAME] → V. Light Footprint
- Added sections: Design System Constraints; Development Workflow; Governance
- Removed sections: none
- Templates: not modified (dependent templates read this file at runtime)
- Deferred TODOs: none. Ratification date set to first adoption, 2026-10-18.
  Confirm with the owner if an earlier adoption date should apply.
-->

# TO DESIGN: Planetary Responsibility Constitution

## Core Principles

### I. Single File, No Build

- The published site MUST live in one file, `index.html`, at the repository root.
- CSS and JavaScript MUST be inline. No frameworks, bundlers, package managers or build
  steps are permitted.
- The only permitted external runtime dependency is Google Fonts. Any other third-party
  script, stylesheet or tracker requires a constitution amendment.
- The page MUST work when served by any static file server and by GitHub Pages from
  `main` / root with no configuration.

Rationale: one file keeps the project easy to understand, preview and deploy, and
removes tooling that would outlast the deliverable.

### II. Every Section Serves the Argument

- The site argues one thesis: designing things, systems and experiences is a planetary
  responsibility.
- The narrative order is fixed: Index, Purpose, Gifts, Paths, Lab, Act. New sections
  MUST fit this arc or the arc MUST be amended here first.
- The five gifts are Empathy, Visual Imagination, Systems Thinking, Craft and Clear
  Communication. Renaming, adding or removing a gift is a MINOR or MAJOR amendment.
- Every interaction or animation MUST reinforce the content it sits on. Decoration
  that does not carry meaning MUST be removed.
- All core text MUST be present in the HTML markup, not injected only by JavaScript.

Rationale: the page is a persuasive piece by Jordan Stephens (i.c.stars, C60). Clarity
of the message outranks visual novelty.

### III. Accessible to Everyone (NON-NEGOTIABLE)

- Each section MUST be a landmark with an accessible name via `aria-labelledby` or
  `aria-label`. Heading levels MUST NOT skip.
- A skip link and a visible `:focus-visible` outline MUST be kept.
- Every interactive control MUST be reachable and operable by keyboard alone. Toggle
  state MUST be exposed with `aria-pressed` or `aria-selected`.
- Purely decorative elements MUST carry `aria-hidden="true"`.
- `prefers-reduced-motion: reduce` MUST disable or soften non-essential motion,
  including scroll effects and the hero animation.
- The custom cursor MUST only replace the native cursor on fine-pointer devices.
- Text and essential UI MUST meet WCAG 2.2 AA contrast against its background.

Rationale: a site about designing responsibly has to be usable by everyone who visits.

### IV. Responsive at Every Width

- The layout MUST work from 320px wide up to large desktop screens.
- There MUST be no horizontal page scroll at any width.
- Side gutters MUST use the `--gut` token and never drop below 16px.
- Navigation that does not fit, such as the HUD links below 820px, MUST collapse
  gracefully rather than overflow.
- Touch targets MUST be at least 44 by 44 CSS pixels.

Rationale: many visitors will open the shared link on a phone.

### V. Light Footprint

- `index.html` SHOULD stay under 100 KB. Growth beyond that MUST be justified in the
  commit message.
- Visuals SHOULD be built with CSS or inline SVG. Raster images, if added, MUST be
  compressed, sized for display, lazy-loaded below the fold and given `alt` text.
- Files in `inspo/` are references only. They MUST NOT be linked from the published page.
- Fonts MUST load with `display=swap` and fall back to system font stacks.
- Animation loops MUST pause when off-screen or when the tab is hidden.

Rationale: a page about planetary responsibility should use as little energy and data
as it can.

## Design System Constraints

- The visual language is Swiss typography combined with a heads-up-display (HUD) layer:
  strict grids, heavy tight sans headlines, monospace labels and hairline rules.
- All colours MUST come from the tokens defined on `:root`: `--ink`, `--night`,
  `--deep`, `--paper`, `--white`, `--grey`, `--stone`, `--orange`, `--cyan`, `--mute`
  and `--hair`. A new colour MUST be added as a token, never hard-coded in a rule.
- Orange (`--orange`) is the single accent for action, focus and progress. Cyan
  (`--cyan`) is secondary and MUST NOT compete with it.
- Type is limited to three families: Inter Tight for display and body, JetBrains Mono
  for labels and data, Newsreader for editorial quotes. A fourth family requires an
  amendment.
- Section backgrounds MUST use the existing surface classes (`paper`, `white`, `dark`,
  `night`) so the rhythm of light and dark sections stays deliberate.
- Content width MUST be bounded by the `.inner` container, max 1320px.

## Development Workflow

1. Edit `index.html` only. Keep changes small and focused on one concern per commit.
2. Preview locally with VS Code Live Server, or `python -m http.server 5500`.
3. Before committing, check every change against this list:
   - The page renders at 320px and at desktop width with no horizontal scroll.
   - Every new control works with keyboard only and shows a focus outline.
   - Reduced-motion mode removes or softens any new animation.
   - No new external dependencies, hard-coded colours or untokenised fonts were added.
4. Commit and push to `main`. GitHub Pages deploys automatically.
5. Confirm the "pages build and deployment" run succeeds in the Actions tab, then check
   the live site at https://jstephens-create.github.io/to-design/.

Larger features SHOULD go through the Spec Kit flow (`/speckit-specify`,
`/speckit-plan`, `/speckit-tasks`) so each plan records a Constitution Check.

## Governance

- This constitution supersedes other project practices. Where a spec, plan or task
  conflicts with it, the constitution wins until it is amended.
- The project owner, Jordan Stephens, approves all amendments. An amendment MUST edit
  this file, update the version line and include a Sync Impact Report in the change.
- Versioning follows semantic versioning:
  - MAJOR: a principle is removed or redefined in a way that breaks earlier work.
  - MINOR: a principle or section is added, or guidance is materially expanded.
  - PATCH: wording, clarifications and typo fixes with no change in meaning.
- Every `/speckit-plan` output MUST include a Constitution Check against Principles
  I to V. Any justified violation MUST be recorded in that plan's complexity tracking.
- Compliance is reviewed at each commit using the Development Workflow checklist above.

**Version**: 1.0.0 | **Ratified**: 2026-10-18 | **Last Amended**: 2026-10-18
