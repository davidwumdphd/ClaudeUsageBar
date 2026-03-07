# Phase 2: Pacing Calculation - Research

**Researched:** 2026-03-07
**Domain:** Swift date arithmetic, Timer, ObservableObject @Published
**Confidence:** HIGH

## Summary

Phase 2 adds linear pacing calculation to `UsageManager`. The core math is straightforward: compare percentage of time elapsed in a usage window against percentage of usage consumed. The implementation involves adding @Published properties for pacing deltas, a 1-minute recalculation timer, and edge case handling.

The existing codebase already uses Timer, Date, and @Published extensively, so no new patterns are introduced. The main research task was verifying the API data format and identifying edge cases that could produce misleading pacing values.

**Primary recommendation:** Add `@Published` Double properties for pacing deltas directly on `UsageManager`, recalculated by a 60-second Timer. Use `Date.timeIntervalSince()` for all time arithmetic. Keep the math in a single `recalculatePacing()` method that both the timer and `parseUsageData` call.

## Standard Stack

No new dependencies. This phase uses only Foundation APIs already present in the codebase.

### Core
| API | Source | Purpose | Why Standard |
|-----|--------|---------|--------------|
| `Foundation.Date` | Apple SDK | Time elapsed calculation | Already used for `sessionResetsAt`, `weeklyResetsAt` |
| `Foundation.Timer` | Apple SDK | 1-minute recalculation | Already used for 5-minute API refresh (line 46) |
| `Combine @Published` | Apple SDK | Reactive pacing properties | Already used for all UsageManager properties |
| `ISO8601DateFormatter` | Apple SDK | Parse `resets_at` timestamps | Already used in `parseUsageData` (line 484) |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Timer | Combine Timer.publish | More idiomatic Combine but adds complexity for no benefit; existing code uses Timer |
| @Published | @Observable macro | Requires iOS 17/macOS 14; app targets macOS 12+ |

## Architecture Patterns

### Existing Code Structure (Single File)
```
app/ClaudeUsageBar.swift
  AppDelegate          -- App lifecycle, status bar, timers
  UsageManager         -- API fetch, data parsing, @Published state
  UsageView            -- SwiftUI popover UI
  (helpers)            -- NSColor extension, custom text fields
```

All pacing code goes into `UsageManager`. No new files, types, or protocols.

### Pattern: Derived @Published Properties with Timer Refresh

**What:** @Published properties that are recalculated periodically from other state, not directly set from API responses.

**When to use:** When a derived value changes over time even without new input data (expected% increases as time passes).

**Implementation approach:**
```swift
// Source: Existing codebase patterns + Foundation.Date docs

// Constants
private let fiveHourDuration: TimeInterval = 5 * 60 * 60   // 18000 seconds
private let sevenDayDuration: TimeInterval = 7 * 24 * 60 * 60 // 604800 seconds

// Published pacing deltas (usage% - expected%, range roughly -100 to +100)
@Published var sessionPacingDelta: Double = 0.0
@Published var weeklyPacingDelta: Double = 0.0

// Timer reference for invalidation
private var pacingTimer: Timer?

// Called from init, after parseUsageData, and every 60s by timer
func recalculatePacing() {
    sessionPacingDelta = calculateDelta(
        usage: Double(sessionUsage),
        resetsAt: sessionResetsAt,
        windowDuration: fiveHourDuration
    )
    weeklyPacingDelta = calculateDelta(
        usage: Double(weeklyUsage),
        resetsAt: weeklyResetsAt,
        windowDuration: sevenDayDuration
    )
}

private func calculateDelta(usage: Double, resetsAt: Date?, windowDuration: TimeInterval) -> Double {
    guard let resetsAt = resetsAt else { return 0.0 }

    let now = Date()
    let windowStart = resetsAt.addingTimeInterval(-windowDuration)
    let elapsed = now.timeIntervalSince(windowStart)

    // Clamp elapsed to [0, windowDuration]
    let clampedElapsed = min(max(elapsed, 0), windowDuration)
    let expectedPercent = (clampedElapsed / windowDuration) * 100.0

    // usage is already 0-100 from API
    return usage - expectedPercent
}
```

### Pattern: Timer Lifecycle in init/deinit

**What:** Create the pacing timer when UsageManager initializes; invalidate on deinit.

```swift
// In init:
pacingTimer = Timer.scheduledTimer(withTimeInterval: 60, repeats: true) { [weak self] _ in
    self?.recalculatePacing()
}

// In deinit:
deinit {
    pacingTimer?.invalidate()
}
```

### Anti-Patterns to Avoid
- **Don't use a computed property:** `var sessionPacingDelta: Double { ... }` would recalculate on every SwiftUI view access, not just every 60 seconds. Use stored @Published properties updated by timer instead.
- **Don't create the timer in AppDelegate:** The pacing timer is internal to UsageManager's concerns. Keep it there alongside the existing data management.
- **Don't use DispatchQueue.asyncAfter for repeating work:** Timer is the correct tool for periodic recalculation. The existing code already uses Timer (line 46).

## Critical Data Format Finding

### API `utilization` is 0-100 (percentage), NOT 0-1 (fractional)

**Confidence: HIGH** (verified from source code)

Evidence from `ClaudeUsageBar.swift`:

1. **Line 489-491:** `utilization` is read as `Double`, cast directly to `Int`:
   ```swift
   if let sessionUtil = fiveHour["utilization"] as? Double {
       sessionUsage = Int(sessionUtil)  // e.g., Int(55.3) = 55
       sessionLimit = 100
   }
   ```

2. **Line 541:** Logged with `%` suffix: `"Parsed: Session \(sessionUsage)%"`

3. **Line 556:** Status bar percentage calculated as `(sessionUsage / sessionLimit) * 100`:
   ```swift
   let sessionPercent = Int((Double(sessionUsage) / Double(sessionLimit)) * 100)
   ```
   If utilization were 0.55, `Int(0.55) = 0`, and the app would always show 0%.

4. **Line 623:** Progress bar value = `sessionUsage / sessionLimit` = e.g., `55/100 = 0.55`, which is correct for SwiftUI `ProgressView(value:)` expecting 0.0-1.0.

**Implication for pacing:** Use `sessionUsage` directly as `usage%` (already 0-100). The `expected%` formula `(timeElapsed / windowDuration) * 100` also produces 0-100. Delta = `usage% - expected%`.

### API `resets_at` Format

**Confidence: HIGH** (verified from source code)

- Parsed with `ISO8601DateFormatter` configured with `.withInternetDateTime` and `.withFractionalSeconds` (line 484-485)
- Format is ISO 8601 with fractional seconds, e.g., `"2026-03-07T15:30:00.000Z"`
- Already parsed into `Date?` properties: `sessionResetsAt`, `weeklyResetsAt`
- These are the window END times. Window start = `resetsAt - windowDuration`

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Date arithmetic | Manual second calculations | `Date.timeIntervalSince()` and `Date.addingTimeInterval()` | Already used in codebase; handles DST, leap seconds at Foundation level |
| Periodic updates | DispatchQueue loops | `Timer.scheduledTimer` | Already the established pattern in this codebase (line 46) |
| Reactive state | Manual notification posting | `@Published` property wrapper | Already used for all 15+ properties on UsageManager |

**Key insight:** This phase introduces zero new patterns. Every API needed (Timer, Date, @Published) is already used in the codebase. The work is pure math and wiring.

## Common Pitfalls

### Pitfall 1: Using the Wrong Base for Percentage Comparison
**What goes wrong:** Comparing fractional usage (0.0-1.0) against percentage expected (0-100), producing wildly wrong deltas.
**Why it happens:** `sessionPercentage` (line 618) is 0.0-1.0, but `sessionUsage` (line 252) is 0-100. Easy to grab the wrong one.
**How to avoid:** Use `sessionUsage` (the Int, 0-100) for pacing, NOT `sessionPercentage` (the Double, 0-1). Or use `sessionPercentage * 100`. Document this clearly.
**Warning signs:** Delta values consistently near -100 or +100 regardless of actual usage.

### Pitfall 2: Timer Not Firing During UI Interaction
**What goes wrong:** Timer stops firing when user is interacting with the popover (scrolling, clicking).
**Why it happens:** `Timer.scheduledTimer` adds to `RunLoop.Mode.default`. During UI events, the run loop switches to `.tracking` mode.
**How to avoid:** Add the timer to `.common` mode:
```swift
let timer = Timer(timeInterval: 60, repeats: true) { [weak self] _ in
    self?.recalculatePacing()
}
RunLoop.main.add(timer, forMode: .common)
pacingTimer = timer
```
**Warning signs:** Pacing values freeze while popover is open but update after closing.
**Assessment:** For a 60-second timer in a menu bar app, this is a minor issue. The existing 5-minute timer on line 46 also uses `scheduledTimer` (default mode) without reported problems. Either approach works, but `.common` mode is strictly better.

### Pitfall 3: Nil resets_at Before First API Fetch
**What goes wrong:** App crashes or shows NaN/Inf deltas on first launch before any API data arrives.
**Why it happens:** `sessionResetsAt` and `weeklyResetsAt` are `Date?`, initially `nil`. Division or arithmetic with nil produces unexpected results.
**How to avoid:** Guard on nil -- return 0.0 delta when `resetsAt` is nil. The code example above handles this with `guard let resetsAt = resetsAt else { return 0.0 }`.
**Warning signs:** Crash on first launch, or pacing shows before "Set Session Cookie" is completed.

### Pitfall 4: Time Outside Window Bounds
**What goes wrong:** If `now` is before window start (shouldn't happen normally) or after `resetsAt` (window expired, awaiting new API data), the elapsed/expected calculation produces values outside 0-100%.
**Why it happens:** Clock skew, delayed API refresh, or stale `resetsAt` value.
**How to avoid:** Clamp elapsed time to `[0, windowDuration]`:
```swift
let clampedElapsed = min(max(elapsed, 0), windowDuration)
```
**Warning signs:** Delta values exceeding expected range (e.g., delta of -150 or +200).

### Pitfall 5: Retain Cycle in Timer Closure
**What goes wrong:** `UsageManager` is never deallocated because the Timer holds a strong reference to it.
**Why it happens:** Timer closures capture `self` strongly by default.
**How to avoid:** Use `[weak self]` in timer closure and `self?.recalculatePacing()`.
**Warning signs:** Memory leak (minor for a menu bar app, but still a defect).

### Pitfall 6: @Published Updates Off Main Thread
**What goes wrong:** Runtime warning "Publishing changes from background threads is not allowed" and potentially inconsistent UI state.
**Why it happens:** Timer callbacks run on the thread they were scheduled on. If `scheduledTimer` is called from a background context, updates to @Published properties would be off-main-thread.
**How to avoid:** Ensure the timer is created on the main thread (in `init` which is called from `applicationDidFinishLaunching`, already on main thread). The existing 5-minute timer in AppDelegate is also created on main thread.
**Warning signs:** Purple runtime warning in Xcode console.

## Code Examples

### Complete Pacing Calculation Method
```swift
// Source: Derived from existing codebase patterns

private let fiveHourDuration: TimeInterval = 5 * 60 * 60
private let sevenDayDuration: TimeInterval = 7 * 24 * 60 * 60

func recalculatePacing() {
    sessionPacingDelta = calculateDelta(
        usage: Double(sessionUsage),
        resetsAt: sessionResetsAt,
        windowDuration: fiveHourDuration
    )
    weeklyPacingDelta = calculateDelta(
        usage: Double(weeklyUsage),
        resetsAt: weeklyResetsAt,
        windowDuration: sevenDayDuration
    )
}

private func calculateDelta(usage: Double, resetsAt: Date?, windowDuration: TimeInterval) -> Double {
    guard let resetsAt = resetsAt else { return 0.0 }

    let now = Date()
    let windowStart = resetsAt.addingTimeInterval(-windowDuration)
    let elapsed = now.timeIntervalSince(windowStart)

    let clampedElapsed = min(max(elapsed, 0), windowDuration)
    let expectedPercent = (clampedElapsed / windowDuration) * 100.0

    return usage - expectedPercent
}
```

### Timer Setup in init
```swift
// Source: Existing Timer pattern from AppDelegate line 46

// In UsageManager.init, after existing setup:
pacingTimer = Timer.scheduledTimer(withTimeInterval: 60, repeats: true) { [weak self] _ in
    self?.recalculatePacing()
}
```

### Integration with parseUsageData
```swift
// At the end of parseUsageData, after updatePercentages():
recalculatePacing()
```

### Delta Interpretation (for Phase 3 UI)
```swift
// Phase 3 will use these values like:
// delta > 0  -> over pace (red)    e.g., "+12.3%"
// delta == 0 -> on pace  (neutral) e.g., "on pace"
// delta < 0  -> under pace (green) e.g., "-8.1%"
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| ObservableObject + @Published | @Observable macro | macOS 14 / iOS 17 (2023) | Not applicable -- app targets macOS 12+ |
| Timer.scheduledTimer | async/await Task.sleep | Swift 5.5 (2021) | Timer is still correct for periodic UI updates tied to run loop |

**Not migrating to @Observable:** The entire codebase uses `ObservableObject` + `@Published` + `@ObservedObject`. Migrating would touch every view and is out of scope for this phase. The macOS 12+ target also prevents using @Observable (requires macOS 14).

## Open Questions

1. **Weekly Sonnet pacing**
   - What we know: The API returns a `seven_day_sonnet` window for Pro plans. The code has `weeklySonnetUsage`, `weeklySonnetResetsAt`, and `hasWeeklySonnet`.
   - What's unclear: Should pacing be calculated for this third window too? The CONTEXT.md says "both 5-hour and 7-day windows" but doesn't mention Sonnet.
   - Recommendation: Skip Sonnet pacing for now. The CONTEXT.md explicitly says "BOTH 5-hour and 7-day" which implies exactly two. Can be added trivially later since the `calculateDelta` function is generic.

2. **Utilization precision loss**
   - What we know: API returns `utilization` as `Double` (e.g., 55.3), but code truncates to `Int` via `Int(sessionUtil)`.
   - What's unclear: Whether fractional precision matters for pacing accuracy. A delta of 55.3 - 50.0 = 5.3 vs 55 - 50.0 = 5.0 is a small difference.
   - Recommendation: Use `Double(sessionUsage)` (the truncated Int) for consistency with existing display. The truncation is at most 1 percentage point, which is acceptable for a "glance" tool. Fixing the truncation would require changing existing properties, which is out of scope.

## Sources

### Primary (HIGH confidence)
- `app/ClaudeUsageBar.swift` lines 475-553 -- API response parsing, data format verification
- `app/ClaudeUsageBar.swift` lines 251-270 -- UsageManager @Published properties
- `app/ClaudeUsageBar.swift` lines 617-626 -- Percentage calculation pattern
- `app/ClaudeUsageBar.swift` line 46 -- Existing Timer usage pattern
- Apple Foundation.Date documentation -- `timeIntervalSince()`, `addingTimeInterval()`
- Apple Foundation.Timer documentation -- `scheduledTimer(withTimeInterval:repeats:block:)`

### Secondary (MEDIUM confidence)
- [Hacking with Swift - Timer guide](https://www.hackingwithswift.com/articles/117/the-ultimate-guide-to-timer) -- RunLoop.Mode.common for timer reliability
- [Swift Forums - @Published thread safety](https://forums.swift.org/t/thread-safety-question-for-property-on-observableobject/62135) -- Main thread requirement for @Published

## Metadata

**Confidence breakdown:**
- API data format: HIGH -- verified from source code with multiple corroborating evidence
- Architecture pattern: HIGH -- follows existing codebase patterns exactly
- Timer implementation: HIGH -- uses same API as existing code (line 46)
- Edge cases: MEDIUM -- based on general Swift knowledge, not codebase-specific testing
- Pitfalls: MEDIUM -- combination of source code analysis and community patterns

**Research date:** 2026-03-07
**Valid until:** 2026-04-07 (stable domain -- Foundation APIs, no external dependencies)
