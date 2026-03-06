# ClaudeUsageBar

## What This Is

A macOS menu bar app that shows Claude API usage (5-hour and 7-day utilization) by reading from the claude.ai API via session cookie. Forked from Artzainnn/ClaudeUsageBar to customize for personal workflow.

## Core Value

At-a-glance visibility into Claude usage and pacing so you know whether to throttle or push harder.

## Requirements

### Validated

- Menu bar icon showing usage status with color coding — existing
- Popover UI displaying 5-hour and 7-day usage bars — existing
- Session cookie authentication via browser DevTools paste — existing
- Auto-refresh every 5 minutes — existing
- Desktop notifications at 25/50/75/90% thresholds — existing
- Global hotkey (Cmd+U) to toggle popover — existing
- Open at login support — existing
- Universal binary (Apple Silicon + Intel) — existing

### Active

- [ ] Monochrome menu bar icon (replace colored icon with SF Symbol or template image)
- [ ] Pacing indicator showing "X% ahead" or "X% behind" based on linear pace through billing cycle (% days elapsed vs % usage consumed)
- [ ] Remove "Buy a dev coffee" UI element

### Out of Scope

- Website changes — this fork is for personal use, not redistribution
- New API integrations — existing claude.ai API is sufficient
- Xcode project migration — command-line build works fine

## Context

- Single-file Swift app (~1138 lines in `app/ClaudeUsageBar.swift`)
- No external dependencies — Apple system frameworks only (SwiftUI, AppKit, Carbon)
- Built via `swiftc` shell scripts, no Xcode project
- Usage data comes from `https://claude.ai/api/organizations/{orgId}/usage`
- API returns `five_hour` and `seven_day` utilization percentages with reset timestamps
- The reset timestamps are key for pacing calculation — they define the billing window boundaries

## Constraints

- **Platform**: macOS 12.0+ (Monterey), menu bar app only (LSUIElement)
- **Build**: Must remain buildable via `app/build.sh` without Xcode project
- **Dependencies**: No external dependencies — Apple frameworks only
- **Single file**: All app code lives in `app/ClaudeUsageBar.swift`

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Linear pacing model | Simple mental model: if 50% of time elapsed, 50% usage is "on pace" | -- Pending |
| Monochrome icon | Matches macOS menu bar conventions; colored icons look out of place | -- Pending |

---
*Last updated: 2026-03-05 after initialization*
