# Technology Stack: Menu Bar Icon and Pacing Text

**Project:** ClaudeUsageBar -- macOS menu bar usage monitor
**Researched:** 2026-03-05
**Scope:** APIs for monochrome menu bar icon, dynamic status text, and pacing calculation

## Current State

The app currently uses:
- `NSStatusItem` with `variableLength` (correct)
- `button.image` set to a custom `NSBezierPath` spark icon with `isTemplate = false` (colored)
- `button.title` set to `" XX%"` string for percentage display
- Custom `createSparkIcon(color:)` that draws a colored spark via bezier paths
- Build target: `arm64-apple-macos12.0` / `x86_64-apple-macos12.0`

## Recommended Changes

### 1. Template Image for Menu Bar Icon

**Recommendation: Keep the custom NSBezierPath icon, set `isTemplate = true`**
**Confidence: HIGH** (verified via Apple docs JSON API)

The current `createSparkIcon(color:)` draws a spark shape as a colored image with `isTemplate = false`. To make it monochrome and respect dark/light mode:

```swift
func createSparkIcon() -> NSImage {
    let size = NSSize(width: 18, height: 18)
    let image = NSImage(size: size)
    image.lockFocus()

    NSColor.black.setFill()  // Template images use black + alpha only
    // ... existing bezier path code ...
    path.fill()

    image.unlockFocus()
    image.isTemplate = true  // macOS handles tinting automatically
    return image
}
```

**What `isTemplate = true` does** (verified, Apple docs):
- Tells macOS the image contains only black and clear colors
- macOS automatically tints it white in dark mode, dark in light mode
- Handles vibrancy, highlight states, and `appearsDisabled` automatically
- No need to manage colors yourself -- the system does it

**Why NOT SF Symbols:**

| Factor | Custom BezierPath | SF Symbols |
|--------|-------------------|------------|
| Visual match to Claude spark | Exact match (current icon) | No exact "Claude spark" symbol exists |
| Availability | macOS 10.5+ (`isTemplate`) | macOS 11.0+ (`systemSymbolName:`) |
| Build complexity | Zero -- already works | Would need to find closest symbol |
| Consistency | Same icon users already recognize | Different icon |

SF Symbols (`NSImage(systemSymbolName:accessibilityDescription:)`, available macOS 11.0+) provides 6,900+ symbols including `sparkle`, `sparkles`, and `wand.and.stars`. However:
- None match the specific Claude spark shape already drawn in the codebase
- The custom bezier path is already pixel-perfect at 16x16
- Using `isTemplate = true` is a one-line change to get full dark/light mode support
- SF Symbols would be the right choice for a new app, but here the custom icon is better

**SF Symbols API reference (for future use if needed):**
```swift
// Available macOS 11.0+ -- compatible with build target macos12.0
let image = NSImage(systemSymbolName: "sparkle", accessibilityDescription: "Usage")

// With sizing configuration (macOS 11.0+)
let config = NSImage.SymbolConfiguration(pointSize: 14, weight: .medium, scale: .small)
let sized = image?.withSymbolConfiguration(config)
```

### 2. Dynamic Status Text (Pacing Info)

**Recommendation: Use `button.attributedTitle` with `monospacedDigitSystemFont`**
**Confidence: HIGH** (verified via Apple docs JSON API)

The current code uses `button.title = " \(percentage)%"`. For pacing text like `42% | 38%` (usage vs expected pace):

```swift
// NSStatusBarButton inherits from NSButton which has:
//   .title (String) -- plain text
//   .attributedTitle (NSAttributedString) -- styled text
//   .image (NSImage) -- icon
//   .imagePosition (NSControl.ImagePosition) -- layout

button.imagePosition = .imageLeading  // icon left of text (RTL-aware)

let text = "\(usage)% | \(pace)%"
let attrs: [NSAttributedString.Key: Any] = [
    .font: NSFont.monospacedDigitSystemFont(ofSize: NSFont.systemFontSize, weight: .regular),
    .baselineOffset: 0.5  // fine-tune vertical alignment if needed
]
button.attributedTitle = NSAttributedString(string: text, attributes: attrs)
```

**Why `monospacedDigitSystemFont`:**
- Numbers have equal width -- prevents the menu bar text from jittering as digits change (e.g., "9%" vs "10%")
- Uses the system font design, just with tabular (fixed-width) digits
- Matches the visual weight of other menu bar text
- Available on all macOS versions (NSFont method)

**Why `attributedTitle` over `title`:**
- Lets you control font (monospaced digits) without affecting the rest of the button
- Can add subtle styling if needed (e.g., lighter weight for the pacing number)
- `title` is fine for plain text but doesn't support font customization

**`imagePosition` values** (verified, Apple docs):

| Value | Behavior |
|-------|----------|
| `.imageLeading` | Image on leading edge (left in LTR) -- **use this** |
| `.imageLeft` | Image on left (ignores RTL) |
| `.imageTrailing` | Image on trailing edge |
| `.imageOnly` | No title shown |
| `.noImage` | No image shown |

Use `.imageLeading` for correct RTL support (though macOS menu bars are generally LTR, it's the modern API).

### 3. Pacing Calculation

**Recommendation: Simple linear interpolation -- no library needed**
**Confidence: HIGH** (pure math, no API dependency)

Pacing = what percentage of the billing cycle has elapsed, expressed as a usage target.

```swift
func calculatePace(resetDate: Date, periodDays: Int = 30) -> Double {
    let now = Date()
    let calendar = Calendar.current

    // Days elapsed since last reset
    let elapsed = calendar.dateComponents([.second], from: resetDate, to: now).second ?? 0
    let totalSeconds = periodDays * 24 * 3600

    // Linear pace: what % of the period has elapsed = expected usage %
    let pace = (Double(elapsed) / Double(totalSeconds)) * 100.0
    return min(pace, 100.0)
}
```

No external libraries needed. `Calendar` and `Date` from Foundation handle timezones and DST correctly.

**Display format options:**

| Format | Example | When to use |
|--------|---------|-------------|
| `42% \| 38%` | Usage vs pace side by side | Clear but wide |
| `42% (+4)` | Usage with delta from pace | Compact, shows over/under |
| `42%` (icon only indicates pace) | Just percentage | Most compact |

### 4. Removing the Color Parameter

**Recommendation: Remove the `color` parameter from `updateStatusIcon` and `createSparkIcon`**
**Confidence: HIGH**

Current signature: `func updateStatusIcon(percentage: Int)` determines color internally based on thresholds. With template images, color is irrelevant -- macOS controls it. The method simplifies to:

```swift
func updateStatusIcon(percentage: Int, pace: Double) {
    guard let button = statusItem.button else { return }

    // Template icon -- always the same, macOS handles tinting
    if button.image == nil {
        button.image = createSparkIcon()  // no color param
        button.image?.isTemplate = true
        button.imagePosition = .imageLeading
    }

    // Dynamic text
    let paceInt = Int(pace)
    let text = " \(percentage)% | \(paceInt)%"
    let attrs: [NSAttributedString.Key: Any] = [
        .font: NSFont.monospacedDigitSystemFont(ofSize: 0, weight: .regular)
        // size 0 = default system size for menu bar
    ]
    button.attributedTitle = NSAttributedString(string: text, attributes: attrs)
}
```

## Alternatives Considered

| Decision | Recommended | Alternative | Why Not Alternative |
|----------|-------------|-------------|---------------------|
| Icon approach | Custom bezier + `isTemplate = true` | SF Symbols `sparkle` | Custom icon matches Claude brand; SF Symbol is generic |
| Text rendering | `attributedTitle` + monospaced digits | Plain `title` string | Plain title causes jittery width changes |
| Image+text layout | `.imageLeading` on button | Separate `NSTextField` in custom view | Custom views in NSStatusItem are deprecated since macOS 10.10 |
| Pacing math | Foundation `Calendar`/`Date` | Third-party date library | Foundation handles DST/timezone; no dependency needed |
| Font for digits | `monospacedDigitSystemFont` | `monospacedSystemFont` | Monospaced digits keeps letters proportional; fully monospaced looks robotic |

## API Summary

All APIs used are available on macOS 12.0+ (the current build target). Most are available much earlier.

| API | Available Since | Purpose |
|-----|-----------------|---------|
| `NSImage.isTemplate` | macOS 10.5 | Monochrome adaptive icon |
| `NSButton.attributedTitle` | macOS 10.0 | Styled text on button |
| `NSButton.imagePosition` | macOS 10.0 | Icon + text layout |
| `NSControl.ImagePosition.imageLeading` | macOS 10.12 | RTL-aware image position |
| `NSFont.monospacedDigitSystemFont(ofSize:weight:)` | macOS 10.11 | Fixed-width digit rendering |
| `NSImage(systemSymbolName:accessibilityDescription:)` | macOS 11.0 | SF Symbols (not recommended here) |
| `NSImage.SymbolConfiguration` | macOS 11.0 | SF Symbol sizing (not needed) |

## What NOT to Use

| Avoid | Why |
|-------|-----|
| `statusItem.title` (direct) | Deprecated -- use `statusItem.button.title` instead |
| `statusItem.image` (direct) | Deprecated -- use `statusItem.button.image` instead |
| `statusItem.view` (custom view) | Deprecated since macOS 10.10 -- use `button` property |
| `NSImage.isTemplate = false` with manual colors | Doesn't adapt to dark mode, vibrancy, or highlight states |
| `NSFont.monospacedSystemFont` | Makes ALL characters fixed-width; menu bar text looks wrong |
| SF Symbols for this specific icon | No Claude spark equivalent; custom bezier is already correct |

## Sources

All API details verified via Apple Developer Documentation JSON endpoints:
- NSStatusItem: developer.apple.com/documentation/appkit/nsstatusitem (properties, deprecated members)
- NSStatusBarButton: developer.apple.com/documentation/appkit/nsstatusbarbutton (inherits NSButton, `appearsDisabled`)
- NSImage.isTemplate: developer.apple.com/documentation/appkit/nsimage/1520017-istemplate (black + clear only, macOS 10.5+)
- NSImage(systemSymbolName:): developer.apple.com/documentation/appkit/nsimage/3622472-init (macOS 11.0+)
- NSImage.SymbolConfiguration: developer.apple.com/documentation/appkit/nsimage/symbolconfiguration (pointSize, weight, scale)
- NSControl.ImagePosition: developer.apple.com/documentation/appkit/nscontrol/imageposition (all enum cases)
- NSButton: developer.apple.com/documentation/appkit/nsbutton (image, title, attributedTitle, imagePosition)
- NSFont: developer.apple.com/documentation/appkit/nsfont (monospacedDigitSystemFont, monospacedSystemFont)
- SF Symbols: developer.apple.com/sf-symbols (v7, 6900+ symbols, macOS 11.0+ for NSImage init)
