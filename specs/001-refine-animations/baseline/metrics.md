# Baseline Metrics (commit 15429cc)

Captured 2026-10-18 with headless Microsoft Edge (Puppeteer, run from outside the repo),
1440×900 viewport, 4x CPU throttling. `TaskDuration` is the main-thread busy time reported
by the Chrome DevTools Protocol. The CPU % divides by the throttle factor (4) to estimate
unthrottled load. Screenshots are kept outside the repository (they are not committed).

| Measure | Baseline (15429cc) |
|---|---|
| Full-page wheel scroll: frames > 50ms | 100 (max 150ms) |
| Idle 10s, Purpose band in view: rAF callbacks/s | 59.9 |
| Idle 10s, Purpose band in view: main-thread busy | 6736ms (≈ 16.8% of a core) |
| Idle 10s, Lab in view: rAF callbacks/s | 60.1 |
| Idle 10s, Lab in view: main-thread busy | 2993ms (≈ 7.5% of a core) |
| Pointer moving 10s over Gifts: busy / script / layout / style | 9226 / 599 / 5253 / 913 ms |
| Pointer moving 10s over Gifts: layout count | 638 |
| Pointer moving 10s over Close: busy / script / layout / style | 8600 / 802 / 4658 / 996 ms |
| Pointer moving 10s over Close: layout count | 283 |
| `index.html` size | 58422 bytes (LF) / 59340 bytes (CRLF working copy) |

## After implementation (same method)

| Measure | Baseline | After | Criterion |
|---|---|---|---|
| Full-page scroll: frames > 50ms (several runs) | 100 to 126 (max 150ms) | 2 to 6 (max 60ms) | SC-001 target is 0; remaining frames are raster time in headless software rendering; confirm on a real device |
| Idle, Purpose band in view: rAF/s, CPU | 59.9/s, ≈ 16.8% | 0/s, ≈ 0.01% | SC-002 met (< 2%) |
| Idle, Lab in view: rAF/s, CPU | 60.1/s, ≈ 7.5% | 0/s, ≈ 0.2% | SC-002 met |
| Gifts pointer move: main-thread work per frame | 45.6ms (212 frames) | 13.1ms (≈ 590 frames) | SC-003 met per frame (−71%); total over 10s −14% because the page now renders 2.8× more frames |
| Close pointer move: main-thread work per frame | 29.2ms (285 frames) | 7.3ms (≈ 600 frames) | SC-003 met (−75% per frame, −50% total) |
| Close settle after scroll at 61fps vs 36fps | — | 983ms vs 1034ms (5%) | SC-004 met (< 10%) |
| `index.html` size | 59340 bytes | 68144 bytes (CRLF) | SC-010 met (< 100 KB) |

Screenshot comparison at rest (375px and 1440px, pixelmatch threshold 0.1): Close and Lab
0.00%; headlines and copy identical in every section. Differences are limited to canvases
that move by design (hero rotation, Gifts bob, Paths turntable) and the Purpose line field,
whose time-based shimmer was replaced by a scroll-driven pattern (research D13), so it shows
a different phase of the same pattern.
