# Feature Landscape: Pacing Indicator for ClaudeUsageBar

**Domain:** macOS menu bar usage tracking -- pacing/budget indicator
**Researched:** 2026-03-05
**Mode:** Focused features research for subsequent milestone

## Context

The app already tracks Claude API usage (5-hour session and 7-day weekly windows) with progress bars, color-coded menu bar icons, and threshold notifications. The API returns `utilization` (0-100%) and `resets_at` (ISO 8601 timestamp) for each window. The goal is to add pacing -- showing whether the user is ahead or behind a linear usage schedule through each window.

## Table Stakes

Features that are baseline-expected for a pacing indicator in this context. Without these, the feature feels incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Signed pacing percentage | Users need to see "+12% ahead" or "-8% behind" at a glance | Low | Core calculation: `usage% - (timeElapsed% through window)`. Positive = ahead of pace (using faster than linear), negative = behind (have headroom). |
| Per-window pacing | Both 5-hour and 7-day windows need pacing since both have independent limits | Low | Same formula applied to each window. The 7-day window is the one users care most about for pacing -- 5-hour is more tactical. |
| Color coding for pacing status | Green/yellow/red visual signal matching existing color scheme | Low | Reuse existing `colorForPercentage` pattern. Green = on pace or behind (headroom), yellow = moderately ahead, red = significantly ahead. Note: color semantics are inverted from the usage bar -- being "ahead" is bad for pacing. |
| Pacing text in popover | "12% ahead of pace" or "On pace" displayed near each usage bar | Low | Natural placement: below the existing "X% used" caption on each progress bar section. |
| Window-aware calculation | Pacing must account for the actual reset timestamp, not assumed window lengths | Low | Use `resets_at` minus window duration (5h or 7d) as window start. Calculate `timeElapsed = (now - windowStart) / windowDuration`. Already have `resets_at` parsed as `Date`. |

## Differentiators

Features that would make the pacing indicator notably useful beyond a basic percentage. Not expected, but add real value for a power user tracking usage.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Menu bar pacing text | Show pacing delta directly in the menu bar next to the icon, e.g., "5h:45% +3%" | Low | Uses `NSStatusItem.variableLength` (already configured). Extremely high glanceability -- no need to open popover. Risk: menu bar width. Keep concise. |
| "Budget remaining" framing | Show "X% budget remaining for Y hours" instead of or alongside raw pacing | Low | Alternative framing: "18% left, 2h remain" is more actionable than "+5% ahead". Could show both. Derived from same data. |
| Pacing-aware notifications | Notify when pacing crosses a threshold, e.g., "You're 20% ahead of weekly pace" | Medium | Extends existing notification system. Needs its own threshold tracking (separate from usage thresholds) to avoid duplicate alerts. Store `lastNotifiedPacingThreshold` in UserDefaults. |
| Visual pace marker on progress bar | Small marker/line on the existing progress bar showing "where you should be" | Medium | A vertical tick or diamond on the ProgressView at the `timeElapsed%` position. Shows gap between actual usage and expected pace visually. Requires custom progress bar (SwiftUI ProgressView doesn't support markers natively). |
| Tooltip on menu bar icon | Hover over menu bar icon shows pacing summary without opening popover | Low | Set `statusItem.button.toolTip` string on each data refresh. Zero UI cost, useful for quick checks. |
| Time-to-limit estimate | "At current pace, you'll hit the limit in X hours" | Medium | Requires either: (a) linear extrapolation from current rate, or (b) simple `remaining% / ratePerHour`. The linear model is fine for a first version. Only meaningful when usage > 0. |

## Anti-Features

Things to deliberately NOT build. Each tempting but wrong for this app's scope and philosophy.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Historical usage graphs/charts | Adds significant complexity (data persistence, charting library) for marginal value in a menu bar app. Menu bar apps should be glanceable, not dashboards. | Show current pacing status only. If users want history, they can check claude.ai/settings/usage. |
| Configurable pacing model (exponential, weighted, custom curves) | YAGNI. Linear pacing is the right mental model for "should I slow down or speed up." Non-linear models add cognitive overhead and configuration surface. | Linear pacing only. Document the formula clearly in code. |
| Per-model usage breakdown in pacing | The API gives aggregate utilization per window, not per-model consumption. Trying to decompose would require tracking individual API calls. | Show pacing on aggregate utilization only (five_hour, seven_day, seven_day_sonnet). |
| Usage prediction with ML/trend analysis | Wildly overengineered for a single-user menu bar utility. The 5-minute refresh interval means data points are sparse. | Simple linear extrapolation at most. |
| Pacing alerts via system notification center (UNUserNotificationCenter) | The app deliberately uses deprecated NSUserNotification because it works without permissions for unsigned apps. Switching notification systems is a separate concern. | Use existing NSUserNotification infrastructure for any pacing notifications. |
| Separate pacing settings panel | Pacing should just work with zero configuration. Adding settings for thresholds, display format, or which windows to show pacing for creates decision fatigue. | Sensible defaults, no settings. Show pacing for all windows that have data. |
| Pacing as a separate view/tab in the popover | The popover is already compact and effective. Adding tabs or navigation breaks the "glance and go" model. | Integrate pacing inline with existing usage bars. |
| Cumulative daily/hourly usage logging | Would require a local database (SQLite/CoreData) and persistent storage beyond UserDefaults. Scope creep toward a different product. | In-memory only, calculated fresh on each API response. |

## Feature Dependencies

```
Window-aware calculation (table stakes)
  |
  +---> Signed pacing percentage (table stakes)
  |       |
  |       +---> Pacing text in popover (table stakes)
  |       |
  |       +---> Color coding for pacing status (table stakes)
  |       |
  |       +---> Menu bar pacing text (differentiator)
  |       |
  |       +---> Tooltip on menu bar icon (differentiator)
  |       |
  |       +---> Pacing-aware notifications (differentiator)
  |
  +---> "Budget remaining" framing (differentiator)
  |
  +---> Time-to-limit estimate (differentiator)
  |
  +---> Visual pace marker on progress bar (differentiator)
```

All features depend on the core pacing calculation. No circular dependencies. Features are independently implementable once the calculation exists.

## MVP Recommendation

For the pacing milestone, prioritize:

1. **Window-aware pacing calculation** -- the foundation everything depends on
2. **Signed pacing percentage in popover** -- the core deliverable
3. **Color coding** -- reuses existing patterns, near-zero cost
4. **Tooltip on menu bar icon** -- one line of code, high utility

Defer to later (or skip entirely):
- **Menu bar pacing text** -- worth considering but risks cluttering the menu bar. Try tooltip first. Could be added if tooltip proves insufficient.
- **Visual pace marker on progress bar** -- requires custom SwiftUI view to replace ProgressView. Nice but not worth the complexity for initial pacing.
- **Pacing-aware notifications** -- useful but adds state management. Better to validate the core pacing display is useful before building alerts around it.
- **Time-to-limit estimate** -- interesting but needs careful edge-case handling (what when usage is 0? what when rate is erratic?). Defer until pacing proves its value.

## Pacing Calculation Notes

The core formula for reference during implementation:

```
windowDuration = resets_at - window_start
  where window_start = resets_at - 5 hours (for five_hour) or resets_at - 7 days (for seven_day)

timeElapsed = (now - window_start) / windowDuration
  clamped to [0, 1]

expectedUsage = timeElapsed * 100  (as percentage)
actualUsage = utilization  (from API, 0-100)

pacingDelta = actualUsage - expectedUsage
  positive = ahead of pace (using faster, will run out early)
  negative = behind pace (have headroom)
```

Edge cases to handle:
- **Before window start**: shouldn't happen with correct reset timestamps, but clamp timeElapsed to 0
- **After window end**: resets_at is in the past, meaning the window has expired. Don't show pacing (or show "expired, waiting for reset")
- **No data yet**: don't show pacing until first successful API fetch
- **Usage at 100%**: show "limit reached" instead of pacing delta

## Semantic Note on "Ahead" vs "Behind"

The language matters. In budget tracking:
- "Ahead of pace" = spending faster than budgeted = BAD (will run out early)
- "Behind pace" = spending slower than budgeted = GOOD (have headroom)

This is counterintuitive because "ahead" usually has positive connotation. Options:
1. **Use neutral language**: "+12%" (red) and "-8%" (green) with colors providing the valence
2. **Flip the framing**: "12% over pace" (red) and "8% under pace" (green) -- clearer semantics
3. **Use budget framing**: "12% ahead of budget" with red color -- let color override word connotation

Recommendation: Use "+X% over pace" / "-X% under pace" with color coding. "Over" and "under" have clear directional meaning that aligns with the color (over = bad = red, under = good = green). Avoid "ahead/behind" to prevent confusion.

## Sources

- Direct analysis of existing codebase (`app/ClaudeUsageBar.swift`)
- API response format from `.planning/codebase/INTEGRATIONS.md`
- macOS menu bar UX conventions (NSStatusItem, tooltips, variable-length items)
- Budget/pacing patterns from data cap tracking (ISP usage meters, mobile data trackers) and financial budgeting apps

**Confidence: HIGH** -- This is a well-constrained feature addition to an existing app with known data inputs. The pacing calculation is pure math on data already available. No external research dependencies.

---

*Feature research: 2026-03-05*
