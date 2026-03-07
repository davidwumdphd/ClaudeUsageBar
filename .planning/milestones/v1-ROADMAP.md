# Milestone v1: Usage and Pacing

**Status:** SHIPPED 2026-03-07
**Phases:** 1-3
**Total Plans:** 3

## Overview

Three sequential phases that clean up the existing app, add pacing logic, then wire pacing into the UI. Each phase produces a working app. The ordering is driven by code dependencies: Phase 1 simplifies `updateStatusIcon` (removing color), Phase 2 adds pacing calculation to `UsageManager`, and Phase 3 extends the status bar and popover to display pacing with proper formatting.

## Phases

### Phase 1: Icon and Cleanup

**Goal**: App looks native in the macOS menu bar and the popover is clean
**Depends on**: Nothing (first phase)
**Requirements**: ICON-01, CLNP-01, CLNP-02
**Plans**: 1 plan

Plans:

- [x] 01-01: SF Symbol sparkle icon, remove donation button, fix popover width

**Success Criteria:**
1. Menu bar icon renders as monochrome and adapts correctly in both light and dark mode
2. "Buy a dev coffee" button is no longer visible in the popover
3. Popover content is not horizontally clipped (width values reconciled)
4. Icon renders crisply on Retina displays (no blurry 1x artifacts)

**Completed:** 2026-03-07

### Phase 2: Pacing Calculation

**Goal**: UsageManager computes correct pacing deltas from existing API data
**Depends on**: Phase 1
**Requirements**: PACE-01
**Plans**: 1 plan

Plans:

- [x] 02-01: Pacing calculation logic, 60s timer, wire into UsageManager data flow

**Success Criteria:**
1. Pacing delta is computed for the active usage window using resets_at timestamp and window duration
2. Positive delta means over pace (consuming faster than linear), negative means under pace (headroom remaining)
3. Pacing values update on every API refresh cycle (every 5 minutes)

**Completed:** 2026-03-07

### Phase 3: Pacing Display

**Goal**: User sees pacing status at a glance in both the status bar and popover
**Depends on**: Phase 2
**Requirements**: ICON-02, PACE-02
**Plans**: 1 plan

Plans:

- [x] 03-01: Pacing delta text in popover, monospaced digit font on status bar

**Success Criteria:**
1. Popover shows pacing delta near usage bars with color coding (green = under pace, red = over pace)
2. Status bar text uses monospaced digit font so numbers do not cause width jitter
3. Pacing text is compact enough to not get truncated on notch MacBooks

**Completed:** 2026-03-07

---

## Milestone Summary

**Key Decisions:**

- SF Symbol "sparkle" for menu bar icon with isTemplate=true (auto dark/light mode)
- Positive pacing delta = over pace (red), matching raw math (usage% - expected%)
- Use Double(sessionUsage) (0-100 Int) for delta math, not sessionPercentage (0-1 Double)
- Timer uses RunLoop.common mode for reliability during popover interaction
- Skip Sonnet pacing -- exactly two pacing windows (5-hour session, 7-day weekly)
- Color thresholds at +/-10pp boundaries (red/green)
- Pacing text uses .caption2 for visual hierarchy below .caption usage percentage

**Issues Resolved:**

None -- all phases executed without issues.

**Issues Deferred:**

- app/README.md references stale pre-Phase-1 behavior (colored emoji icon)

**Technical Debt Incurred:**

- No VERIFICATION.md files created during phase execution (verified retroactively by integration checker)
- Stale app/README.md documentation

---

*For current project status, see .planning/PROJECT.md*
