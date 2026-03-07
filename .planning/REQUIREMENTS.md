# Requirements: ClaudeUsageBar

**Defined:** 2026-03-05
**Core Value:** At-a-glance visibility into Claude usage and pacing

## v1 Requirements

### Icon

- [x] **ICON-01**: Menu bar icon uses monochrome template image (auto dark/light mode handling)
- [ ] **ICON-02**: Status bar text uses monospaced digit font to prevent width jitter

### Pacing

- [x] **PACE-01**: Linear pacing calculation comparing % time elapsed vs % usage consumed
- [ ] **PACE-02**: Pacing displayed in popover with color coding (green = under pace, red = over pace)

### Cleanup

- [x] **CLNP-01**: Remove "Buy a dev coffee" UI element from popover
- [x] **CLNP-02**: Fix popover width inconsistency (320 vs 360)

## v2 Requirements

### Status Bar

- **STAT-01**: Compact pacing text shown in status bar next to icon
- **STAT-02**: Tooltip with pacing details on hover

### Pacing

- **PACE-03**: Separate pacing display for both 5-hour and 7-day windows

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
| ICON-02 | Phase 3 | Pending |
| PACE-01 | Phase 2 | Complete |
| PACE-02 | Phase 3 | Pending |
| CLNP-01 | Phase 1 | Complete |
| CLNP-02 | Phase 1 | Complete |

**Coverage:**
- v1 requirements: 6 total
- Mapped to phases: 6
- Unmapped: 0

---
*Requirements defined: 2026-03-05*
*Last updated: 2026-03-05 after roadmap creation*
