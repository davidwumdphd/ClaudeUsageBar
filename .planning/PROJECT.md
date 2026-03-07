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
- Monochrome menu bar icon (SF Symbol "sparkle" template image) — v1
- Linear pacing calculation (% time elapsed vs % usage consumed) — v1
- Pacing display in popover with color coding (green/red at +/-10pp) — v1
- Monospaced digit font on status bar text — v1
- Remove "Buy a dev coffee" UI element — v1
- Fix popover width inconsistency (320 vs 360) — v1

### Active

(None — define in next milestone)

### Out of Scope

- Website changes — this fork is for personal use, not redistribution
- New API integrations — existing claude.ai API is sufficient
- Xcode project migration — command-line build works fine
- Custom pacing curves — linear is sufficient for personal use
- Usage history/charts — overengineering for a status bar app

## Context

- Single-file Swift app (~1151 lines in `app/ClaudeUsageBar.swift`)
- No external dependencies — Apple system frameworks only (SwiftUI, AppKit, Carbon)
- Built via `swiftc` shell scripts, no Xcode project
- Usage data comes from `https://claude.ai/api/organizations/{orgId}/usage`
- API returns `five_hour` and `seven_day` utilization percentages with reset timestamps
- Pacing calculation uses reset timestamps to derive expected linear usage, refreshed every 60 seconds
- Shipped v1 on 2026-03-07: monochrome icon, pacing calculation + display, cleanup

## Constraints

- **Platform**: macOS 12.0+ (Monterey), menu bar app only (LSUIElement)
- **Build**: Must remain buildable via `app/build.sh` without Xcode project
- **Dependencies**: No external dependencies — Apple frameworks only
- **Single file**: All app code lives in `app/ClaudeUsageBar.swift`

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Linear pacing model | Simple mental model: if 50% of time elapsed, 50% usage is "on pace" | Good |
| Monochrome icon | Matches macOS menu bar conventions; colored icons look out of place | Good |
| SF Symbol "sparkle" with isTemplate=true | Auto dark/light mode, Retina-crisp | Good |
| Positive delta = over pace (red) | Matches raw math: usage% - expected% | Good |
| RunLoop.common for pacing timer | Reliable ticks during popover interaction | Good |
| Color thresholds at +/-10pp | Avoids noisy color changes near zero | Good |
| Skip Sonnet pacing | Only two usage windows (session + weekly) need pacing | Good |

---
*Last updated: 2026-03-07 after v1 milestone*
