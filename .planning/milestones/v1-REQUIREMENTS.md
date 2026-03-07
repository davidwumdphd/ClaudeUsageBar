# Requirements Archive: v1 Usage and Pacing

**Archived:** 2026-03-07
**Status:** SHIPPED

This is the archived requirements specification for v1.
For current requirements, see `.planning/REQUIREMENTS.md` (created for next milestone).

---

**Defined:** 2026-03-05
**Core Value:** At-a-glance visibility into Claude usage and pacing

## v1 Requirements

### Icon

- [x] **ICON-01**: Menu bar icon uses monochrome template image (auto dark/light mode handling)
- [x] **ICON-02**: Status bar text uses monospaced digit font to prevent width jitter

### Pacing

- [x] **PACE-01**: Linear pacing calculation comparing % time elapsed vs % usage consumed
- [x] **PACE-02**: Pacing displayed in popover with color coding (green = under pace, red = over pace)

### Cleanup

- [x] **CLNP-01**: Remove "Buy a dev coffee" UI element from popover
- [x] **CLNP-02**: Fix popover width inconsistency (320 vs 360)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Custom pacing curves | Linear is sufficient for personal use |
| Usage history/charts | Overengineering for a status bar app |
| Per-model usage breakdown | Not in scope for this fork |
| Website changes | Personal use, not redistribution |
| SF Symbols icon | Existing spark shape has no matching SF Symbol |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| ICON-01 | Phase 1 | Complete |
| ICON-02 | Phase 3 | Complete |
| PACE-01 | Phase 2 | Complete |
| PACE-02 | Phase 3 | Complete |
| CLNP-01 | Phase 1 | Complete |
| CLNP-02 | Phase 1 | Complete |

---

## Milestone Summary

**Shipped:** 6 of 6 v1 requirements
**Adjusted:** None
**Dropped:** None

---
*Archived: 2026-03-07 as part of v1 milestone completion*
