---
phase: 02-pacing-calculation
plan: 01
subsystem: logic
tags: [swift, timer, date-arithmetic, pacing, observable]

requires:
  - phase: 01-status-bar-icon
    provides: "UsageManager with @Published usage properties and parseUsageData"
provides:
  - "@Published sessionPacingDelta and weeklyPacingDelta on UsageManager"
  - "recalculatePacing method callable from timer, data parse, and cookie clear"
  - "calculateDelta helper with nil guard and elapsed time clamping"
affects: [03-pacing-ui]

tech-stack:
  added: []
  patterns:
    - "Derived @Published properties updated by periodic Timer"
    - "RunLoop.common mode for timer reliability during UI interaction"

key-files:
  created: []
  modified:
    - "app/ClaudeUsageBar.swift"

key-decisions:
  - "Use Double(sessionUsage) (0-100 Int) not sessionPercentage (0-1 Double) for delta math"
  - "Timer uses RunLoop.common mode for reliability during popover interaction"
  - "Skip Sonnet pacing -- CONTEXT.md specifies exactly two windows"

patterns-established:
  - "calculateDelta(usage:resetsAt:windowDuration:) generic helper reusable for any window"
  - "Timer lifecycle: create in init with [weak self], invalidate in deinit"

duration: 2min
completed: 2026-03-07
---

# Phase 2 Plan 1: Pacing Calculation Summary

**Linear pacing delta calculation for 5-hour and 7-day windows with 60-second timer refresh via RunLoop.common**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-07T16:42:13Z
- **Completed:** 2026-03-07T16:43:41Z
- **Tasks:** 2
- **Files modified:** 1

## Accomplishments
- Added `sessionPacingDelta` and `weeklyPacingDelta` @Published properties to UsageManager
- Implemented `calculateDelta` with nil guard on resetsAt and elapsed time clamping to [0, windowDuration]
- Wired 60-second pacing timer using RunLoop.common mode with [weak self] and deinit invalidation
- Integrated recalculatePacing into parseUsageData, clearSessionCookie, and timer callback

## Task Commits

Each task was committed atomically:

1. **Task 1: Add pacing calculation logic to UsageManager** - `9688d87` (feat)
2. **Task 2: Add pacing timer and wire into data flow** - `8e946dd` (feat)

## Files Created/Modified
- `app/ClaudeUsageBar.swift` - Added pacing constants, @Published deltas, calculateDelta helper, recalculatePacing method, 60-second timer setup/teardown, and integration calls

## Decisions Made
- Used `Double(sessionUsage)` (the 0-100 Int) for delta calculation, not `sessionPercentage` (0.0-1.0 Double), to stay consistent with API scale
- Used `RunLoop.main.add(timer, forMode: .common)` instead of `Timer.scheduledTimer` for timer reliability during UI interaction
- Skipped Sonnet pacing per CONTEXT.md specifying "both 5-hour and 7-day" (exactly two windows)

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- `sessionPacingDelta` and `weeklyPacingDelta` are @Published and ready for Phase 3 UI binding
- Phase 3 can display delta values with sign formatting (e.g., "+12.3%" / "-8.1%")
- Delta > 0 = over pace (red), delta < 0 = under pace (green), per roadmap decision

---
*Phase: 02-pacing-calculation*
*Completed: 2026-03-07*
