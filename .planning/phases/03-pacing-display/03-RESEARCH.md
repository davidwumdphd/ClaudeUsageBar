# Phase 3: Pacing Display - Research

**Researched:** 2026-03-07
**Domain:** SwiftUI text formatting, AppKit NSFont, macOS status bar presentation
**Confidence:** HIGH

## Summary

Phase 3 is a pure presentation phase: wire two existing `@Published` properties (`sessionPacingDelta`, `weeklyPacingDelta`) into the popover UI, and apply monospaced digit font to the status bar text. No new calculations, no new data sources.

The key APIs are well-established and all available on macOS 12.0+ (the app's deployment target): SwiftUI's `.monospacedDigit()` modifier (macOS 12.0+), `NSFont.monospacedDigitSystemFont(ofSize:weight:)` (macOS 10.11+), and standard SwiftUI `Color.red`/`Color.green` which adapt to dark mode automatically. The formatting logic (sign prefix, integer rounding, threshold-based coloring) is straightforward string manipulation with no library needs.

The popover currently has a fixed `contentSize` of 360x260. Adding two pacing lines (one per usage bar) will increase the required height. The popover should either increase its fixed height or (better) remove the height constraint and let SwiftUI size it intrinsically.

**Primary recommendation:** Add a pacing text line below each usage bar's "X% used" caption using `.font(.caption2)` for visual subordination, use `NSFont.monospacedDigitSystemFont` for the status bar button, and increase popover height to accommodate the new content.

## Standard Stack

### Core

No new libraries needed. All APIs are Apple system frameworks already imported.

| API | Available Since | Purpose | Why Standard |
|-----|-----------------|---------|--------------|
| `Font.monospacedDigit()` | macOS 12.0+ | Fixed-width digits in SwiftUI Text | SwiftUI-native, prevents jitter |
| `NSFont.monospacedDigitSystemFont(ofSize:weight:)` | macOS 10.11+ | Fixed-width digits for NSStatusItem button | AppKit-native, the only way to set font on status bar buttons |
| `Color.red` / `Color.green` | macOS 10.15+ | Adaptive system colors | Auto dark/light mode, semantic meaning |
| `.foregroundColor(_:)` | macOS 10.15+ | Text coloring | App already uses this throughout; `foregroundStyle` requires macOS 14+ |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `Color.red`/`.green` | Custom RGB colors | System colors adapt to dark mode automatically; custom colors need manual dark mode handling |
| `.foregroundColor()` | `.foregroundStyle()` | `.foregroundStyle()` is the modern API but requires macOS 14+; app targets macOS 12+ |
| `NSFont.monospacedDigitSystemFont` | `NSFont.monospacedSystemFont` | monospacedDigit only affects digits (keeps other chars proportional); monospacedSystem makes everything monospaced -- uglier |

## Architecture Patterns

### Current Popover Structure (per usage bar)

```
VStack(alignment: .leading, spacing: 4) {
    HStack { title + reset time }
    ProgressView(...)
    Text("X% used")              <-- .font(.caption), .foregroundColor(.secondary)
}
```

### Pattern: Add Pacing Line Below Each Bar

**What:** Add a new `Text` view after the "X% used" line displaying the pacing delta.
**When to use:** Both session and weekly usage bar sections.

```swift
// After the "X% used" Text in each VStack:
Text(formatPacingDelta(usageManager.sessionPacingDelta))
    .font(.caption2)
    .foregroundColor(pacingColor(usageManager.sessionPacingDelta))
```

This keeps the pacing subordinate (`.caption2` is ~11pt vs `.caption` at ~12pt) and on its own line as specified.

### Pattern: Format Function for Pacing Delta

**What:** A pure function that converts a Double delta to the display string.
**Logic:**
- Positive: `"+12% ahead of pace"`
- Negative: `"-3% behind pace"`
- Zero: `"0%"`

```swift
func formatPacingDelta(_ delta: Double) -> String {
    let rounded = Int(delta.rounded())
    if rounded > 0 {
        return "+\(rounded)% ahead of pace"
    } else if rounded < 0 {
        return "\(rounded)% behind pace"  // negative sign is automatic
    } else {
        return "0%"
    }
}
```

### Pattern: Threshold Color Function

**What:** Returns the appropriate color based on the 10pp threshold.
**Logic:**
- `abs(delta) <= 10`: `.secondary` (neutral default)
- `delta > 10`: `.red` (over pace, bad)
- `delta < -10`: `.green` (under pace, good)

```swift
func pacingColor(_ delta: Double) -> Color {
    let rounded = Int(delta.rounded())
    if rounded > 10 {
        return .red
    } else if rounded < -10 {
        return .green
    } else {
        return .secondary
    }
}
```

### Pattern: Monospaced Digit Font on NSStatusItem

**What:** Apply fixed-width digit font to the status bar button text.
**Where:** `updateStatusIcon(percentage:)` method in `AppDelegate`.

```swift
func updateStatusIcon(percentage: Int) {
    guard let button = statusItem.button else { return }
    if let icon = NSImage(systemSymbolName: "sparkle", accessibilityDescription: "Claude usage") {
        icon.isTemplate = true
        button.image = icon
    }
    button.title = " \(percentage)%"
    button.font = NSFont.monospacedDigitSystemFont(
        ofSize: NSFont.systemFontSize,
        weight: .regular
    )
}
```

Note: `NSFont.systemFontSize` returns the standard system font size (13pt on macOS), which matches the default status bar text size.

### Anti-Patterns to Avoid

- **Setting button.font every call**: Setting the font on every `updateStatusIcon` call is harmless (it's the same font object) but could be set once in `applicationDidFinishLaunching` instead. Either approach works; setting it in `updateStatusIcon` is simpler since that's where `button.title` is already set.
- **Using `.monospaced()` instead of `.monospacedDigit()`**: `.monospaced()` makes ALL characters fixed-width (letters, spaces, punctuation). `.monospacedDigit()` only affects digits -- much better for mixed text like " 42%".
- **Hardcoding font size for status bar**: Use `NSFont.systemFontSize` to match the system default. Hardcoding (e.g., 13.0) breaks if Apple changes the default.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Monospaced digits in status bar | Custom attributed string | `NSFont.monospacedDigitSystemFont(ofSize:weight:)` | One-liner, system-native |
| Monospaced digits in SwiftUI | Manual font descriptor tweaking | `.monospacedDigit()` modifier | SwiftUI-native modifier |
| Dark mode adaptive colors | Manual `@Environment(\.colorScheme)` checks | `Color.red`, `Color.green`, `Color.secondary` | System colors adapt automatically |
| Number sign formatting | NumberFormatter with custom positive/negative patterns | Simple if/else with string interpolation | Overkill for this use case |

## Common Pitfalls

### Pitfall 1: Popover Height Too Small

**What goes wrong:** Adding pacing text lines increases content height. The popover has a hardcoded `contentSize` of `NSSize(width: 360, height: 260)`. New content gets clipped.
**Why it happens:** The popover height was set for the original content without pacing lines.
**How to avoid:** Increase the height to accommodate the two new pacing lines (~30px total). Alternatively, remove the height from `contentSize` and use only width, letting SwiftUI's intrinsic sizing handle height. The view already uses `.frame(width: 360)` internally.
**Warning signs:** Content cut off at bottom of popover, scroll bars appearing.

### Pitfall 2: Using Int() Instead of rounded() for Delta

**What goes wrong:** `Int(delta)` truncates toward zero. A delta of 9.7 becomes 9, a delta of -9.7 becomes -9. This can misrepresent the pacing by a full percentage point.
**Why it happens:** Swift's `Int()` initializer truncates, not rounds.
**How to avoid:** Use `Int(delta.rounded())` to get proper rounding.
**Warning signs:** Pacing text shows values that seem too low/different from expected.

### Pitfall 3: Negative Sign Duplication

**What goes wrong:** If you write `"-\(rounded)%"` for negative values, you get `"--3%"` because `rounded` is already negative and the minus sign is implicit in string interpolation.
**Why it happens:** Easy to forget that negative Int already includes the minus sign.
**How to avoid:** For negative values, just use `"\(rounded)%"` -- the minus sign comes from the integer itself. Only the positive case needs a manual `"+"` prefix.
**Warning signs:** Double minus signs in the UI.

### Pitfall 4: Threshold Comparison on Unrounded Value

**What goes wrong:** If you compare the raw Double delta (e.g., 10.4) against the threshold of 10, but display the rounded integer (10), the text shows "10% ahead of pace" in colored text even though visually "10" seems like it should be at the threshold.
**Why it happens:** Mismatch between the value compared and the value displayed.
**How to avoid:** Round first, then compare the rounded integer against the threshold. This ensures the displayed number and the coloring are consistent.
**Warning signs:** Text colored when the displayed number is exactly at the threshold boundary.

## Code Examples

### Complete Pacing Display in Popover

```swift
// Inside UsageView, after each usage bar's "X% used" Text:

// Session bar pacing (add after line 851)
Text(formatPacingDelta(usageManager.sessionPacingDelta))
    .font(.caption2)
    .foregroundColor(pacingColor(usageManager.sessionPacingDelta))

// Weekly bar pacing (add after line 872)
Text(formatPacingDelta(usageManager.weeklyPacingDelta))
    .font(.caption2)
    .foregroundColor(pacingColor(usageManager.weeklyPacingDelta))
```

### Complete Helper Functions

```swift
// Add to UsageView
func formatPacingDelta(_ delta: Double) -> String {
    let rounded = Int(delta.rounded())
    if rounded > 0 {
        return "+\(rounded)% ahead of pace"
    } else if rounded < 0 {
        return "\(rounded)% behind pace"
    } else {
        return "0%"
    }
}

func pacingColor(_ delta: Double) -> Color {
    let rounded = Int(delta.rounded())
    if rounded > 10 {
        return .red
    } else if rounded < -10 {
        return .green
    } else {
        return .secondary
    }
}
```

### Monospaced Digit Font on Status Bar

```swift
// In AppDelegate.updateStatusIcon(percentage:)
button.font = NSFont.monospacedDigitSystemFont(
    ofSize: NSFont.systemFontSize,
    weight: .regular
)
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `.foregroundColor()` | `.foregroundStyle()` | macOS 14 / Xcode 15 | Deprecated warning only; `.foregroundColor()` still works and is required for macOS 12 target |
| Manual font descriptors for monospaced digits | `.monospacedDigit()` SwiftUI modifier | macOS 12 / SwiftUI 3 | Much simpler API |

**Note on foregroundColor deprecation:** The existing codebase uses `.foregroundColor()` throughout (~20 call sites). Since the app targets macOS 12+, this is the correct API to use. Switching to `.foregroundStyle()` would require raising the deployment target to macOS 14. Not worth changing.

## Open Questions

1. **Popover height strategy**
   - What we know: Current height is 260px. Adding two caption2 lines adds ~30px. The Sonnet bar (when visible) also takes space.
   - What's unclear: Whether to increase to a fixed 290px or remove the height constraint entirely.
   - Recommendation: Increase to ~290-300px. Removing height entirely could cause the popover to resize dynamically when the cookie input or settings expand, which may feel jarring. A modest fixed increase is safer.

2. **Pacing text when no data fetched**
   - What we know: The pacing deltas default to 0.0 when no data is fetched. The usage bars are already hidden behind `if usageManager.hasFetchedData`.
   - What's unclear: Nothing -- the pacing text is inside the same conditional block, so it won't show when there's no data.
   - Recommendation: No special handling needed; the pacing text is added inside the existing `hasFetchedData` guard.

## Sources

### Primary (HIGH confidence)
- Codebase: `app/ClaudeUsageBar.swift` -- read in full, all relevant sections analyzed
- Apple Developer Documentation: `NSFont.monospacedDigitSystemFont(ofSize:weight:)` -- available since macOS 10.11
- Apple Developer Documentation: `Font.monospacedDigit()` -- available since macOS 12.0 (iOS 15.0)
- Apple Developer Documentation: `Color.red`, `Color.green` -- system adaptive colors, available since macOS 10.15

### Secondary (MEDIUM confidence)
- [Use Your Loaf - Monospace Digits](https://useyourloaf.com/blog/monospace-digits/) -- confirmed AppKit and SwiftUI usage patterns
- [Michael Tsai - Monospaced Digits in SwiftUI](https://mjtsai.com/blog/2022/09/28/monospaced-digits-in-swiftui/) -- confirmed no known gotchas with the modifier
- [Nil Coalescing - foregroundColor vs foregroundStyle](https://nilcoalescing.com/blog/ForegroundColorStyleAndTintInSwiftUI/) -- confirmed foregroundColor is correct for macOS 12 target

### Tertiary (LOW confidence)
- None -- all findings verified with primary or secondary sources

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all APIs are Apple system frameworks, well-documented, already compatible with app's macOS 12 target
- Architecture: HIGH -- pattern is trivially extending existing VStack structure with one more Text view per bar
- Pitfalls: HIGH -- identified from direct code analysis (popover height, rounding, sign formatting)

**Research date:** 2026-03-07
**Valid until:** Indefinite -- these are stable Apple platform APIs unlikely to change
