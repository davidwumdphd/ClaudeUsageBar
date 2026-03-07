# Phase 2: Pacing Calculation - Context

**Gathered:** 2026-03-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Add linear pacing calculation to UsageManager for both 5-hour and 7-day windows. Pure math/logic — no UI changes. Phase 3 consumes the pacing data for display.

</domain>

<decisions>
## Implementation Decisions

### Window definition
- Calculate pacing for BOTH 5-hour and 7-day usage windows
- Window start = resets_at - window_duration (5 hours or 7 days are fixed constants)
- Window durations are hardcoded constants, not parsed from API

### Delta semantics
- Raw difference: delta = usage% - expected%
- expected% = (time_elapsed / window_duration) * 100
- Positive delta = over pace (consuming faster than linear — bad)
- Negative delta = under pace (headroom remaining — good)
- Zero = exactly on pace
- API format for usage% needs to be checked from existing code (researcher should verify whether 0-1 or 0-100)

### Data flow
- Pacing recalculates continuously via a 1-minute timer, not just on API refresh
- This matters because expected% changes as time passes even without new usage data
- Add @Published properties on UsageManager for pacing deltas (both windows)
- Phase 3 binds directly to these properties in SwiftUI views

### Claude's Discretion
- Edge case handling (window just started, near end, no data)
- Exact property naming and types on UsageManager
- Timer implementation details

</decisions>

<specifics>
## Specific Ideas

- The 5-hour window is the one that triggers rate limits — it's the more actionable pacing number
- Pacing should feel like a simple mental model: "if 50% of time elapsed, 50% usage is on pace"

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 02-pacing-calculation*
*Context gathered: 2026-03-07*
