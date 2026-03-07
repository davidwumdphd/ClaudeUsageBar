# Roadmap: ClaudeUsageBar

## Overview

Three sequential phases that clean up the existing app, add pacing logic, then wire pacing into the UI. Each phase produces a working app. The ordering is driven by code dependencies: Phase 1 simplifies `updateStatusIcon` (removing color), Phase 2 adds pacing calculation to `UsageManager`, and Phase 3 extends the status bar and popover to display pacing with proper formatting.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Icon and Cleanup** - Monochrome template icon, remove coffee button, fix popover width
- [x] **Phase 2: Pacing Calculation** - Linear pacing math in UsageManager
- [x] **Phase 3: Pacing Display** - Pacing UI in popover and status bar with formatting

## Phase Details

### Phase 1: Icon and Cleanup
**Goal**: App looks native in the macOS menu bar and the popover is clean
**Depends on**: Nothing (first phase)
**Requirements**: ICON-01, CLNP-01, CLNP-02
**Success Criteria** (what must be TRUE):
  1. Menu bar icon renders as monochrome and adapts correctly in both light and dark mode
  2. "Buy a dev coffee" button is no longer visible in the popover
  3. Popover content is not horizontally clipped (width values reconciled)
  4. Icon renders crisply on Retina displays (no blurry 1x artifacts)
**Plans**: 1 plan

Plans:
- [x] 01-01-PLAN.md -- SF Symbol sparkle icon, remove donation button, fix popover width

### Phase 2: Pacing Calculation
**Goal**: UsageManager computes correct pacing deltas from existing API data
**Depends on**: Phase 1
**Requirements**: PACE-01
**Success Criteria** (what must be TRUE):
  1. Pacing delta is computed for the active usage window using resets_at timestamp and window duration
  2. Positive delta means over pace (consuming faster than linear), negative means under pace (headroom remaining)
  3. Pacing values update on every API refresh cycle (every 5 minutes)
**Plans**: 1 plan

Plans:
- [x] 02-01-PLAN.md -- Pacing calculation logic, 60s timer, wire into UsageManager data flow

### Phase 3: Pacing Display
**Goal**: User sees pacing status at a glance in both the status bar and popover
**Depends on**: Phase 2
**Requirements**: ICON-02, PACE-02
**Success Criteria** (what must be TRUE):
  1. Popover shows pacing delta near usage bars with color coding (green = under pace, red = over pace)
  2. Status bar text uses monospaced digit font so numbers do not cause width jitter
  3. Pacing text is compact enough to not get truncated on notch MacBooks
**Plans**: 1 plan

Plans:
- [x] 03-01-PLAN.md -- Pacing delta text in popover, monospaced digit font on status bar

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Icon and Cleanup | 1/1 | Complete | 2026-03-07 |
| 2. Pacing Calculation | 1/1 | Complete | 2026-03-07 |
| 3. Pacing Display | 1/1 | Complete | 2026-03-07 |
