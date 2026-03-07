# Phase 1: Icon and Cleanup - Context

**Gathered:** 2026-03-06
**Status:** Ready for planning

<domain>
## Phase Boundary

Make the app look native in the macOS menu bar and clean up the popover. Monochrome template icon, remove donation/coffee references, fix popover width. No new features — just polish what exists.

</domain>

<decisions>
## Implementation Decisions

### Icon style
- Use SF Symbol "sparkle" instead of the custom-drawn star path
- Set as template image so it adapts to light/dark mode automatically
- Static icon — no state changes based on usage level (same sparkle always)
- Retina-crisp via SF Symbols (no manual bitmap drawing)

### Status bar text
- Keep showing 5-hour percentage next to the icon (e.g., sparkle + "75%")
- 5-hour window is the one that matters for rate limiting

### Coffee button and donation cleanup
- Remove the "Buy Dev a Coffee" button entirely
- Remove ALL donation/support references including the Stripe link in settings
- Let the layout collapse naturally — no replacement element needed
- Remove the website link from the popover
- Keep the GitHub repo link

### Popover width
- Claude's Discretion: Fix the horizontal clipping issue — reconcile width values so content isn't cut off

</decisions>

<specifics>
## Specific Ideas

- Should feel like a native macOS menu bar app — no colored icons, no branding clutter
- Personal fork, so strip out anything upstream-promotional (website, donations) but keep GitHub link for reference

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-icon-and-cleanup*
*Context gathered: 2026-03-06*
