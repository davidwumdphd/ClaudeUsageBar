# Project Research Summary

**Project:** ClaudeUsageBar -- Pacing Indicator + Icon Cleanup
**Domain:** macOS menu bar app modification (AppKit/SwiftUI, single-file Swift)
**Researched:** 2026-03-05
**Confidence:** HIGH

## Executive Summary

ClaudeUsageBar is an existing macOS menu bar app (single-file, 1138 lines of Swift) that monitors Claude API usage across 5-hour session and 7-day weekly windows. The planned changes are well-scoped modifications to an already-working app: convert the colored menu bar icon to a standard monochrome template image, add a pacing indicator that compares actual usage against linear time-elapsed pace, remove the "Buy Dev a Coffee" button, and fix a popover width mismatch. All required APIs (`NSImage.isTemplate`, `NSFont.monospacedDigitSystemFont`, `NSButton.attributedTitle`) are stable AppKit APIs available on macOS 12.0+, the current build target.

The recommended approach is a three-phase implementation ordered by dependency: first simplify the icon and remove the coffee button (independent, zero-risk changes that clean up the codebase), then add the pacing calculation to `UsageManager` (pure math on existing data), and finally wire pacing into the status bar text and popover UI. This ordering matters because the icon change and pacing change both modify `updateStatusIcon` -- doing the icon simplification first establishes the clean signature that pacing then extends.

The primary risks are cosmetic and semantic, not technical. The biggest trap is the "ahead/behind" language ambiguity: being "ahead of pace" in usage consumption is bad (burning through allocation), but "ahead" sounds positive. The research consensus is to use "+X% over pace" / "-X% under pace" with color coding where positive (under) = green = good, negative (over) = red = bad. The other notable risk is menu bar width -- pacing text must be kept compact (`42% +5` not `42% | 5% ahead of pace`) to avoid truncation on notch MacBooks.

## Key Findings

### Recommended Stack

No new dependencies. All changes use existing AppKit and Foundation APIs already available at the current build target (macOS 12.0+). The app builds via `swiftc` with no Xcode project.

**Core technologies:**
- `NSImage.isTemplate = true`: monochrome icon that adapts to dark/light mode automatically (macOS 10.5+)
- `NSFont.monospacedDigitSystemFont`: fixed-width digits prevent menu bar text jitter as numbers change (macOS 10.11+)
- `NSButton.attributedTitle`: styled text for pacing display with font control (macOS 10.0+)
- `Foundation Calendar/Date`: pacing calculation via time-elapsed percentage -- no external library needed
- `NSImage(size:flipped:drawingHandler:)`: replaces deprecated `lockFocus`/`unlockFocus` for Retina-correct icon drawing

**Key decision: Custom bezier path over SF Symbols.** The app already has a pixel-perfect Claude spark icon drawn with `NSBezierPath`. No SF Symbol matches the Claude brand. Setting `isTemplate = true` on the existing icon is a one-line change vs. redesigning the icon with SF Symbols.

### Expected Features

**Must have (table stakes):**
- Window-aware pacing calculation using `resets_at` timestamps from the API
- Signed pacing delta displayed in the popover near each usage bar
- Color coding for pacing status (green = under pace/headroom, red = over pace/burning fast)
- Per-window pacing for both 5-hour and 7-day windows

**Should have (high value, low cost):**
- Tooltip on menu bar icon with pacing summary (`statusItem.button.toolTip`) -- one line of code
- Menu bar pacing text showing compact delta next to percentage

**Defer:**
- Visual pace marker on progress bar (requires custom SwiftUI view to replace `ProgressView`)
- Pacing-aware notifications (adds state management, validate core pacing first)
- Time-to-limit estimate (edge cases when usage is 0 or rate is erratic)
- Historical usage graphs, configurable pacing models, per-model breakdown (anti-features -- YAGNI)

### Architecture Approach

The app is a single-file architecture with three logical layers: `AppDelegate` (menu bar chrome), `UsageManager` (API client + state), and `UsageView` (SwiftUI popover). Pacing belongs in `UsageManager` as computed state derived from existing `@Published` properties (`sessionUsage`, `sessionResetsAt`, etc.). The delegate pattern for `UsageManager -> AppDelegate` communication stays unchanged. The `updateStatusIcon` signature expands from `(percentage:)` to `(percentage:, pacingDelta:)`. No file splitting, no new patterns.

**Modified components:**
1. `AppDelegate.createSparkIcon()` -- remove color param, add `isTemplate = true`, use drawing handler instead of `lockFocus`
2. `AppDelegate.updateStatusIcon(percentage:, pacingDelta:)` -- remove color logic, add pacing text formatting
3. `UsageManager` -- add `sessionPacingDelta`/`weeklyPacingDelta` `@Published` properties, `calculatePacing()` method
4. `UsageView` -- remove coffee button, show pacing delta inline, fix popover width (320 vs 360 mismatch)

### Critical Pitfalls

1. **Template image alpha masking** -- `isTemplate = true` uses alpha channel only; all RGB color info is ignored. Must draw with opaque black on clear background, remove all color selection logic atomically. Test in both light and dark mode.

2. **Pacing text width in menu bar** -- Adding pacing text roughly doubles status item width. On notch MacBooks with many menu bar items, long text gets truncated or the item disappears entirely. Use compact format `42% +5` and monospaced digit font to prevent width jitter.

3. **"Ahead" semantics are inverted** -- "Ahead of pace" = burning faster = bad. Use "over pace" / "under pace" with matching colors (over = red, under = green). Decide semantics before implementing display.

4. **Wrong window start calculation** -- API provides `resets_at` but not window start. Must compute `window_start = resets_at - window_duration`. Getting this wrong makes pacing meaningless. Verify with manual spot-check against claude.ai usage page.

5. **`lockFocus` deprecated since macOS 10.15** -- Current icon drawing may produce blurry 1x icons on Retina. Fix during template image conversion by switching to `NSImage(size:flipped:drawingHandler:)`.

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Icon Cleanup + Coffee Button Removal

**Rationale:** These are independent, zero-risk changes that simplify the codebase before adding new functionality. The icon change modifies `updateStatusIcon` which pacing also needs -- doing cleanup first avoids merge conflicts with yourself.

**Delivers:** Monochrome template icon (dark/light mode adaptive), removed donation button, fixed popover width mismatch (320 -> 360), Retina-correct icon drawing.

**Addresses:** Monochrome icon (table stakes for native macOS appearance), popover cleanup.

**Avoids:** Pitfall 1 (template alpha masking), Pitfall 3 (color param removal across all callers), Pitfall 5 (lockFocus deprecation), Pitfall 6 (popover size after removal).

### Phase 2: Pacing Calculation

**Rationale:** Core logic must be correct before wiring to any UI. Pacing is pure math on data already available in `UsageManager` -- implement and verify the calculation in isolation.

**Delivers:** `sessionPacingDelta` and `weeklyPacingDelta` as `@Published` properties on `UsageManager`, called after each API response parse.

**Addresses:** Window-aware calculation (table stakes), per-window pacing (table stakes).

**Avoids:** Pitfall 4 (wrong time boundaries -- verify with spot-check), Pitfall 7 (decide over/under semantics before Phase 3).

### Phase 3: Pacing UI

**Rationale:** With correct pacing values computed, wire them into the status bar text, popover display, and tooltip. This is where semantics and formatting decisions materialize.

**Delivers:** Pacing delta in menu bar text (`42% +5`), pacing text in popover near each usage bar, color coding for pacing status, tooltip on menu bar icon.

**Addresses:** Signed pacing percentage in popover (table stakes), color coding (table stakes), menu bar pacing text (differentiator), tooltip (differentiator).

**Avoids:** Pitfall 2 (menu bar width -- compact format), Pitfall 7 (semantics -- use over/under not ahead/behind), Pitfall 9 (font alignment), Pitfall 10 (unnecessary text flicker).

### Phase Ordering Rationale

- Phase 1 before Phase 2 because both touch `updateStatusIcon`. Clean the signature first (`remove color`), then extend it (`add pacingDelta`).
- Phase 2 before Phase 3 because the calculation must be verified correct before it appears in the UI. A wrong number displayed confidently is worse than no number.
- All three phases modify a single file, so they are sequential by nature. Each phase produces a working app -- no phase leaves the build broken.
- Total estimated net code change: +25 lines (pacing calc adds ~25, coffee removal subtracts ~13, icon simplification subtracts ~10, pacing UI adds ~20).

### Research Flags

Phases with standard patterns (skip deeper research):
- **Phase 1:** Well-documented AppKit APIs (`isTemplate`, `NSImage` drawing). All verified against Apple docs. No unknowns.
- **Phase 2:** Pure math on known data types. Formula is straightforward. Only risk is edge cases (covered in PITFALLS.md).
- **Phase 3:** Standard `NSAttributedString` and SwiftUI text display. Formatting is a design choice, not a technical question.

No phases need `/gsd:research-phase`. All APIs are verified, the codebase is fully analyzed, and the changes are modifications to existing patterns rather than new integrations.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All APIs verified via Apple Developer Documentation JSON endpoints. APIs are stable since macOS 10.11-12.0. |
| Features | HIGH | Well-constrained feature set on known data inputs. Pacing is pure math on existing API data. |
| Architecture | HIGH | Single-file app fully analyzed (1138 lines). Data flow traced end-to-end. No architectural decisions needed. |
| Pitfalls | HIGH | Based on direct codebase analysis and well-established AppKit behavior. No speculative pitfalls. |

**Overall confidence: HIGH**

This is a modification to an existing, working app using stable macOS APIs. The scope is narrow, the data inputs are known, and the architecture doesn't change. The research identified one existing bug (popover width mismatch: 320 vs 360) that should be fixed alongside the planned changes.

### Gaps to Address

- **Popover width mismatch (320 vs 360):** Existing bug found during research. The popover `contentSize` is 320pt wide but the SwiftUI view requests 360pt. Investigate whether this causes horizontal clipping, and reconcile during Phase 1.
- **Pacing display format final decision:** Research recommends `+X over pace` / `-X under pace` with color, but the exact string format for the menu bar (`42% +5` vs `42% (+5)` vs other) should be decided during Phase 3 implementation. Try a few options and pick the most readable at menu bar size.
- **Pacing sign convention:** Research files have conflicting conventions on what "positive" means. FEATURES.md recommends `+X = over pace (bad)` while PITFALLS.md suggests positive should mean headroom (good). Settle this definitively before Phase 3. Recommendation: positive delta = over pace (using more than expected), displayed in red/orange. This matches the raw math (`usage% - expected%`) and avoids a sign flip that could introduce bugs.

## Sources

### Primary (HIGH confidence)
- Apple Developer Documentation JSON API -- NSImage.isTemplate, NSStatusItem, NSStatusBarButton, NSFont, NSButton, NSControl.ImagePosition
- Direct codebase analysis -- `app/ClaudeUsageBar.swift` (1138 lines, read in full)
- API response format from `.planning/codebase/INTEGRATIONS.md`

### Secondary (MEDIUM confidence)
- macOS menu bar UX conventions (training data knowledge of established patterns)
- Budget/pacing patterns from data cap tracking and financial budgeting apps

---
*Research completed: 2026-03-05*
*Ready for roadmap: yes*
