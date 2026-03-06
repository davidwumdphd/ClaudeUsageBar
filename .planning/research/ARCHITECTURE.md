# Architecture Patterns

**Domain:** macOS menu bar app modifications (monochrome icon, pacing calculation, buy-coffee removal)
**Researched:** 2026-03-05

## Current Architecture

Single-file Swift app (`app/ClaudeUsageBar.swift`, 1138 lines) with three logical layers:

| Component | Lines | Responsibility |
|-----------|-------|----------------|
| `AppDelegate` | 7-271 | Owns `NSStatusItem`, popover lifecycle, icon rendering, global hotkey |
| `UsageManager` | 298-674 | API client, state holder (`ObservableObject`), notifications, settings |
| `UsageView` | 809-1138 | SwiftUI popover UI with usage bars, cookie input, settings |
| Input Components | 677-807 | AppKit text input bridges for paste support in popovers |

Data flow: `UsageManager.fetchUsage()` -> parse API response -> update `@Published` properties -> SwiftUI auto-refreshes `UsageView` -> `updateStatusBar()` calls `AppDelegate.updateStatusIcon()` to update menu bar.

## Where Each Change Lives

### 1. Monochrome Icon (AppDelegate)

**Current state:** `AppDelegate.createSparkIcon(color:)` draws a 4-pointed star path and fills it with a color determined by usage percentage (green < 70%, yellow < 90%, red >= 90%). `image.isTemplate = false` forces the color to render literally.

**Target state:** Set `image.isTemplate = true` and remove the color parameter. Template images in macOS automatically render in the system's menu bar color (dark on light menu bar, light on dark), matching native app conventions.

**Affected code:**
- `createSparkIcon(color:)` at line 237 -- remove `color` parameter, use `NSColor.black` fill (the actual rendered color is controlled by the template flag, but the path must be filled with an opaque color for the mask to work)
- `updateStatusIcon(percentage:)` at line 216 -- remove the color-determination logic (the `if percentage < 70` chain), stop passing color to `createSparkIcon`
- `updateStatusBar()` in UsageManager at line 602 -- still calls `delegate?.updateStatusIcon(percentage:)` but the percentage is now only used for the title text, not color

**Key detail:** `NSImage.isTemplate = true` tells AppKit to treat the image as a mask. AppKit composites it with the appropriate menu bar appearance. The image should use black fill with appropriate alpha. This is the standard macOS menu bar convention -- all native Apple menu bar icons are template images.

### 2. Pacing Calculation (UsageManager)

**Current state:** `UsageManager` has `@Published` properties for `sessionUsage` (0-100 utilization), `sessionResetsAt` (Date?), and equivalent weekly properties. These come directly from the API response's `five_hour.utilization` and `five_hour.resets_at` fields.

**Target state:** Add pacing calculation that compares time-elapsed percentage against usage percentage. The reset timestamp defines the end of the billing window; the window start is implied (5 hours before `resets_at` for session, 7 days before for weekly).

**Pacing model:**
```
windowStart = resetsAt - windowDuration  (5h or 7d)
timeElapsed = now - windowStart
timeElapsedPct = timeElapsed / windowDuration
pacingDelta = usagePct - timeElapsedPct
```

If `pacingDelta > 0`, usage is ahead of pace (consuming faster than linear). If `pacingDelta < 0`, usage is behind pace (has room to spend).

**Where it belongs:** New computed properties or `@Published` properties on `UsageManager`:

```swift
// Add to UsageManager
@Published var sessionPacingDelta: Double? = nil  // e.g., +15.0 means 15% ahead
@Published var weeklyPacingDelta: Double? = nil

func calculatePacing() {
    sessionPacingDelta = calculatePacingDelta(
        usage: Double(sessionUsage),
        resetsAt: sessionResetsAt,
        windowDuration: 5 * 3600  // 5 hours in seconds
    )
    weeklyPacingDelta = calculatePacingDelta(
        usage: Double(weeklyUsage),
        resetsAt: weeklyResetsAt,
        windowDuration: 7 * 24 * 3600  // 7 days in seconds
    )
}

private func calculatePacingDelta(usage: Double, resetsAt: Date?, windowDuration: TimeInterval) -> Double? {
    guard let resetsAt = resetsAt else { return nil }
    let windowStart = resetsAt.addingTimeInterval(-windowDuration)
    let now = Date()
    let elapsed = now.timeIntervalSince(windowStart)
    let timeElapsedPct = (elapsed / windowDuration) * 100.0
    return usage - timeElapsedPct  // positive = ahead, negative = behind
}
```

**Call site:** Invoke `calculatePacing()` at the end of `parseUsageData()` (after utilization and reset times are set), and also in `updatePercentages()` so it refreshes when the popover opens.

**Why UsageManager and not a separate class:** The pacing calculation depends directly on `sessionUsage`, `sessionResetsAt`, `weeklyUsage`, `weeklyResetsAt` -- all owned by `UsageManager`. Adding a separate class would mean either duplicating state or adding a dependency injection layer, both unnecessary complexity for a single-file app. Pacing is a derived value from existing state; it belongs next to that state.

### 3. Status Bar Text Update (AppDelegate)

**Current state:** `updateStatusIcon(percentage:)` sets `button.title = " \(percentage)%"`, showing raw usage like `72%`.

**Target state:** Show pacing information in the status bar text. Format: `72% +5` (where `+5` means 5% ahead of pace) or `72% -12` (12% behind). This keeps the display compact for the menu bar.

**Data flow for pacing text:**
1. `UsageManager.updateStatusBar()` already calls `delegate?.updateStatusIcon(percentage:)`
2. Expand this to also pass pacing delta: `delegate?.updateStatusIcon(percentage: sessionPercent, pacingDelta: sessionPacingDelta)`
3. `AppDelegate.updateStatusIcon` formats the title string

**Alternative considered -- UsageManager sets title directly:** `UsageManager` already holds a reference to `statusItem` (line 318), so it could set `statusItem?.button?.title` directly. However, `updateStatusIcon` in `AppDelegate` already owns button configuration (image + title). Keeping both in one place avoids split ownership of the status bar button. Recommend keeping `AppDelegate` as the single owner of button state, with `UsageManager` passing data through the delegate call.

**Title formatting logic:**
```swift
func updateStatusIcon(percentage: Int, pacingDelta: Double? = nil) {
    guard let button = statusItem.button else { return }

    let sparkIcon = createSparkIcon()  // no color param
    button.image = sparkIcon

    if let delta = pacingDelta {
        let sign = delta >= 0 ? "+" : ""
        button.title = " \(percentage)% \(sign)\(Int(delta))"
    } else {
        button.title = " \(percentage)%"
    }
}
```

### 4. Remove Buy-Coffee Element (UsageView)

**Current state:** Lines 982-994 in `UsageView.body`:
```swift
// Support Section
Button(action: {
    NSWorkspace.shared.open(URL(string: "https://donate.stripe.com/...")!)
}) {
    HStack(spacing: 4) {
        Text("coffee emoji")
        Text("Buy Dev a Coffee")
    }
}
.buttonStyle(.borderless)
.font(.caption)
.foregroundColor(.orange)
```

**Target state:** Delete these 13 lines. No other code references this button or depends on it. Clean removal with no side effects.

## Component Boundaries After Changes

```
AppDelegate
  |-- statusItem: NSStatusItem (owns menu bar button)
  |-- updateStatusIcon(percentage:, pacingDelta:)  [MODIFIED signature]
  |-- createSparkIcon() -> NSImage  [MODIFIED: no color param, isTemplate=true]
  |-- popover, hotkey, click handling  [UNCHANGED]

UsageManager (ObservableObject)
  |-- sessionUsage, weeklyUsage, etc.  [UNCHANGED]
  |-- sessionResetsAt, weeklyResetsAt  [UNCHANGED]
  |-- sessionPacingDelta, weeklyPacingDelta  [NEW @Published]
  |-- calculatePacing()  [NEW]
  |-- calculatePacingDelta(usage:resetsAt:windowDuration:)  [NEW private]
  |-- updateStatusBar()  [MODIFIED: passes pacing delta to delegate]
  |-- parseUsageData()  [MODIFIED: calls calculatePacing()]
  |-- updatePercentages()  [MODIFIED: calls calculatePacing()]

UsageView
  |-- Usage bars with pacing info  [MODIFIED: show pacing delta inline]
  |-- Buy-coffee button  [REMOVED]
  |-- Everything else  [UNCHANGED]
```

## Data Flow After Changes

```
API Response
  |
  v
parseUsageData()
  |-- sets sessionUsage, weeklyUsage (utilization %)
  |-- sets sessionResetsAt, weeklyResetsAt (Date objects)
  |-- calls calculatePacing()
  |     |-- computes sessionPacingDelta from usage + reset time
  |     |-- computes weeklyPacingDelta from usage + reset time
  |-- calls updatePercentages() (for SwiftUI progress bars)
  |-- calls updateStatusBar()
        |
        v
  updateStatusBar()
    |-- computes sessionPercent
    |-- calls delegate.updateStatusIcon(percentage:, pacingDelta:)
    |     |-- creates template image (monochrome)
    |     |-- sets button.title = "72% +5"
    |-- calls checkNotificationThresholds()
```

## Build Order (Dependencies)

The three changes have minimal interdependencies but one clear ordering constraint:

**Phase 1: Monochrome icon + Remove buy-coffee** (independent, no dependencies)
- Monochrome icon: Modify `createSparkIcon` and `updateStatusIcon` in AppDelegate. Set `isTemplate = true`, remove color logic. Self-contained change.
- Remove buy-coffee: Delete 13 lines from UsageView. Zero dependencies.
- These two can be done in any order or simultaneously. Neither affects the other.

**Phase 2: Pacing calculation + status bar text** (depends on understanding the icon change)
- Add pacing properties and `calculatePacing()` to UsageManager.
- Modify `updateStatusIcon` signature to accept `pacingDelta` parameter.
- Modify `updateStatusBar()` to pass pacing delta through.
- Optionally show pacing in the popover UsageView too.
- This should come after the icon change because `updateStatusIcon` is modified in both changes. Doing the icon change first establishes the simplified signature, then pacing extends it.

**Recommended single-pass order:**
1. Remove buy-coffee button (trivial deletion, gets it out of the way)
2. Convert icon to monochrome (simplifies `updateStatusIcon`)
3. Add pacing calculation to UsageManager
4. Wire pacing into status bar text via modified `updateStatusIcon`
5. Optionally display pacing in popover UsageView

## Anti-Patterns to Avoid

### Don't Split Into Multiple Files
The app is 1138 lines in a single file. The three changes add roughly 30 lines net (pacing calc adds ~25, coffee removal subtracts ~13, icon simplification subtracts ~10). A ~1145-line single file is fine. Splitting would require changing the build scripts and adds complexity for no benefit.

### Don't Over-Engineer the Pacing Model
Linear pacing (time-elapsed % vs usage %) is the right model. Don't add exponential decay, weighted averages, or historical tracking. The API gives a point-in-time utilization percentage and a reset timestamp -- that's all the data available, and linear pacing is the most useful mental model for "should I use more or less right now?"

### Don't Add a Separate Timer for Pacing Updates
Pacing values are derived from `Date()` relative to reset timestamps. They'll be stale between API refreshes (5-minute interval), but that's fine -- the reset timestamps don't change between refreshes, only `Date()` does. If real-time pacing updates are desired later, a 60-second timer could recalculate, but don't add this now. The 5-minute refresh is sufficient.

### Don't Change the Delegate Pattern
`UsageManager` communicates to `AppDelegate` via a weak delegate reference for status icon updates. This works. Don't switch to Combine publishers, NotificationCenter, or callbacks for this interaction. The delegate pattern is appropriate for this one-directional "data layer tells app layer to update chrome" communication.

## Edge Cases to Handle

**No reset timestamp available:** When `sessionResetsAt` is nil (e.g., first launch, API error), `calculatePacingDelta` returns nil, and the status bar falls back to showing just the percentage without pacing info. The optional chaining handles this naturally.

**Window hasn't started yet:** If `now < windowStart` (clock skew or timezone issues), `timeElapsedPct` would be negative. Clamp to 0: `max(0, elapsed)`.

**Window has passed:** If `now > resetsAt`, the window is over and pacing is meaningless. Return nil or clamp. The API should return updated data on next refresh, but handle gracefully.

**Zero usage with pacing:** At 0% usage and 30% time elapsed, pacing shows `-30` (30% behind pace). This is correct and useful -- it means "you have plenty of room."

---

*Architecture research: 2026-03-05*
