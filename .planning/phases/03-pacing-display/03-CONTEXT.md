# Phase 3: Pacing Display - Context

**Gathered:** 2026-03-07
**Status:** Ready for planning

<domain>
## Phase Boundary

Wire pacing deltas (from Phase 2's @Published properties) into the popover UI and apply monospaced digit font to the status bar. No new calculations — this phase is pure presentation.

</domain>

<decisions>
## Implementation Decisions

### Popover pacing layout
- Show pacing delta BELOW each usage bar (not inline), on its own line
- Show pacing for BOTH 5-hour and 7-day windows in the popover
- Format: "+12% ahead of pace" / "-3% behind pace" / "0%"
- Color coding is subtle with threshold: only color text when delta exceeds +/-10 percentage points
- Under 10pp: neutral/default text color
- Over +10pp: red text (over pace, bad)
- Under -10pp: green text (under pace, good)

### Number formatting
- Integer only — no decimal places ("+12%" not "+12.3%")
- Show sign prefix: "+" for positive, "-" for negative, no sign for zero
- Zero delta displays as "0%" (not "on pace" text)
- Pacing label: "ahead of pace" for positive, "behind pace" for negative, no label for zero

### Status bar
- Do NOT add pacing to the status bar text — keep it as just the usage percentage
- Apply monospaced digit font to the status bar percentage text to prevent width jitter as numbers change

### Claude's Discretion
- Exact SwiftUI text styling and alignment for the pacing line
- Font size for pacing text relative to the usage bar text
- How to handle the monospaced digit font (NSFont.monospacedDigitSystemFont or equivalent)
- Color values for green/red

</decisions>

<specifics>
## Specific Ideas

- Pacing should be secondary/subordinate to the usage bar — smaller font, not competing for attention
- The 10pp threshold keeps the UI calm during normal usage, only highlighting when meaningfully off pace

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 03-pacing-display*
*Context gathered: 2026-03-07*
