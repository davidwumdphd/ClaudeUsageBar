# Codebase Concerns

**Analysis Date:** 2026-03-05

## Tech Debt

**Monolithic Single-File Architecture:**
- Issue: The entire macOS app lives in a single 1138-line file (`app/ClaudeUsageBar.swift`) containing AppDelegate, UsageManager, custom text field classes, NSViewRepresentable bridges, and SwiftUI views
- Files: `app/ClaudeUsageBar.swift`
- Impact: Difficult to navigate, modify one concern without risk to others, and reason about responsibilities
- Fix approach: Split into separate files - `AppDelegate.swift`, `UsageManager.swift`, `UsageView.swift`, `CustomTextField.swift`, `PasteableTextField.swift`. Use a Swift Package or Xcode project instead of raw `swiftc` compilation

**No Build System / Package Manager:**
- Issue: The app compiles via raw `swiftc` in a shell script (`app/build.sh`) with no Xcode project, no Swift Package Manager, and no dependency management
- Files: `app/build.sh`
- Impact: No incremental builds, no IDE integration for debugging, no way to add dependencies, no test target possible
- Fix approach: Create a Package.swift or Xcode project. This would also enable unit testing

**Deprecated NSUserNotification API:**
- Issue: Uses `NSUserNotification` and `NSUserNotificationCenter` which were deprecated in macOS 11 (Big Sur). The code acknowledges this in a comment on line 15: "deprecated but works without permissions for unsigned apps"
- Files: `app/ClaudeUsageBar.swift` (lines 644-661)
- Impact: May stop working in future macOS versions. Apple could remove the API at any time
- Fix approach: Migrate to `UNUserNotificationCenter` from the UserNotifications framework. This requires requesting notification permissions

**Deprecated lockFocus/unlockFocus API:**
- Issue: Uses `NSImage.lockFocus()` / `unlockFocus()` for drawing the status bar icon, deprecated since macOS 10.14
- Files: `app/ClaudeUsageBar.swift` (lines 241, 266)
- Impact: Thread-unsafe drawing approach that may produce incorrect results on Retina displays or in future macOS versions
- Fix approach: Use `NSImage(size:flipped:drawingHandler:)` block-based API instead

**UserDefaults.synchronize() Calls:**
- Issue: Calls `UserDefaults.standard.synchronize()` in 5 places. Apple has documented this as unnecessary since iOS 12 / macOS 10.14 - UserDefaults auto-synchronizes
- Files: `app/ClaudeUsageBar.swift` (lines 362, 369, 377, 629, 639)
- Impact: No functional issue, but it is unnecessary code that suggests outdated patterns
- Fix approach: Remove all `.synchronize()` calls

## Known Bugs

**"Open at Login" Toggle Does Nothing:**
- Symptoms: The Settings panel has an "Open at Login" toggle that saves a boolean to UserDefaults but never registers the app as a login item
- Files: `app/ClaudeUsageBar.swift` (lines 312, 348, 360, 1004-1018)
- Trigger: Enable the toggle - the app does not actually launch at login
- Workaround: Manually add the app to Login Items in System Settings
- Fix approach: Use `SMAppService.mainApp.register()` (macOS 13+) or `LSSharedFileListInsertItemURL` to actually register/unregister as a login item when the toggle changes

**Cookie Header Inconsistency Between Bootstrap and Usage Requests:**
- Symptoms: The bootstrap API call wraps the cookie as `sessionKey=\(sessionCookie)` (line 419), while the usage API call sends `sessionCookie` raw (line 478). If the user pastes the full cookie string (as instructed by the UI), the bootstrap call will be malformed (double-prefixed)
- Files: `app/ClaudeUsageBar.swift` (lines 419, 478)
- Trigger: Paste full cookie string, have no `lastActiveOrg` cookie present - the bootstrap fallback will fail
- Workaround: The `lastActiveOrg` cookie is typically present in the full cookie string, so `fetchOrganizationId` usually returns from the cookie-parsing branch (line 401-409) and never hits the bootstrap call

**Redundant Percentage Calculation:**
- Symptoms: `sessionUsage` is already a percentage (0-100, parsed from `utilization` which is a 0-100 double). But `updateStatusBar()` on line 603 recalculates it as `Int((Double(sessionUsage) / Double(sessionLimit)) * 100)` where `sessionLimit` is always 100. This happens to produce the correct result (N/100*100 = N) but is confusing and would break if `sessionLimit` were ever changed
- Files: `app/ClaudeUsageBar.swift` (lines 536-539, 602-603)
- Trigger: Not currently a bug, but a latent issue
- Workaround: None needed currently

## Security Considerations

**Session Cookie Stored in Plaintext UserDefaults:**
- Risk: The full Claude session cookie (which grants full account access) is stored in plaintext in UserDefaults (a plist file on disk). Any process with read access to `~/Library/Preferences/com.claude.usagebar.plist` can steal the session
- Files: `app/ClaudeUsageBar.swift` (lines 336-337, 365-369)
- Current mitigation: None - cookie is in plaintext
- Recommendations: Store the cookie in macOS Keychain using `SecItemAdd`/`SecItemCopyMatching`. This protects the credential with the system's encrypted credential store and requires user consent for other apps to access it

**Full API Response Logged to System Console:**
- Risk: NSLog on line 508 logs the entire API response body (`📦 Response: \(responseString)`), and line 486 logs the full URL with org ID. Line 366 logs cookie length. These logs are readable by any process via Console.app or `log stream`
- Files: `app/ClaudeUsageBar.swift` (lines 366, 405, 432, 486, 505, 508)
- Current mitigation: None
- Recommendations: Remove verbose response logging or gate it behind a debug flag. Never log org IDs or cookie metadata in production builds

**39 NSLog Statements in Production Code:**
- Risk: Excessive logging of operational details (cookie operations, API responses, org IDs, notification states) creates an information leak surface and performance overhead
- Files: `app/ClaudeUsageBar.swift` (39 occurrences)
- Current mitigation: None
- Recommendations: Use `os_log` with appropriate privacy levels, or use `#if DEBUG` guards. Remove all emoji from log statements for professionalism and searchability

**Hardcoded Stripe Donation URL Contains Identifiers:**
- Risk: The Stripe donation link `https://donate.stripe.com/3cIcN5b5H7Q8ay8bIDfIs02` is hardcoded in two places. If the link needs updating, both must change
- Files: `app/ClaudeUsageBar.swift` (line 984), `website/index.html` (lines 1052, 1353)
- Current mitigation: None
- Recommendations: Extract to a constant in the Swift code. For the website, use a single variable

## Performance Bottlenecks

**Timer-Based Polling on Main Thread:**
- Problem: Usage data is fetched every 5 minutes via a `Timer.scheduledTimer` on the main run loop (line 46). The URL session callback dispatches back to main thread for all processing (line 489)
- Files: `app/ClaudeUsageBar.swift` (lines 46-48, 488-519)
- Cause: JSON parsing and all state updates happen on the main thread inside the URL session completion handler
- Improvement path: Parse JSON off the main thread, only dispatch UI updates to main. For a menu bar app with 5-minute intervals this is not a real problem, but it is poor practice

## Fragile Areas

**API Response Format Parsing:**
- Files: `app/ClaudeUsageBar.swift` (lines 522-600)
- Why fragile: The entire data parsing relies on Anthropic's undocumented internal API format (`/api/organizations/{orgId}/usage`). The JSON keys (`five_hour`, `seven_day`, `seven_day_sonnet`, `utilization`, `resets_at`) are hardcoded strings with no versioning or schema validation. If Anthropic changes the response shape, the app silently shows 0% for everything
- Safe modification: When adding support for new usage tiers, follow the existing pattern of optional parsing (check for key existence before accessing). Always test with real API responses
- Test coverage: None - no test infrastructure exists

**Cookie Format Assumptions:**
- Files: `app/ClaudeUsageBar.swift` (lines 400-409)
- Why fragile: The org ID extraction splits cookies by ";" and looks for `lastActiveOrg=` prefix. If Anthropic renames this cookie or changes its format, the fallback bootstrap endpoint is the only recovery, and that has its own cookie header bug (see Known Bugs)
- Safe modification: Add fallback for multiple cookie name formats. Validate org ID format before using it
- Test coverage: None

**Popover Sizing:**
- Files: `app/ClaudeUsageBar.swift` (lines 38, 1091)
- Why fragile: The popover content size is hardcoded to 320x260 (line 38) but the view frame is 360 wide (line 1091). The actual content height varies based on whether the cookie input, settings panel, weekly Sonnet usage, and error messages are visible. This can cause content clipping or excessive whitespace
- Safe modification: The popover should use auto-sizing based on content, or the height should be calculated dynamically
- Test coverage: None

## Scaling Limits

Not applicable - this is a single-user local menu bar app with no server component.

## Dependencies at Risk

**Dependency on Undocumented Anthropic API:**
- Risk: The app relies on `https://claude.ai/api/organizations/{orgId}/usage` which is an internal, undocumented API endpoint. Anthropic could change, move, or authentication-gate this endpoint at any time without notice
- Impact: The entire app stops functioning - it cannot show usage data
- Migration plan: None available. If Anthropic provides an official usage API, migrate to that. Otherwise, maintain awareness of API changes through user reports

**Carbon Framework for Hot Keys:**
- Risk: Uses the Carbon framework's `RegisterEventHotKey` API (imported via `import Carbon`) for global keyboard shortcuts. Carbon is a legacy framework from Classic Mac OS
- Impact: Apple may deprecate or remove Carbon hotkey APIs in future macOS versions
- Migration plan: Consider using `NSEvent.addGlobalMonitorForEvents(matching: .keyDown)` (requires Accessibility permission already requested) or the newer `KeyboardShortcuts` Swift package

## Missing Critical Features

**No Automatic Cookie Refresh:**
- Problem: Session cookies expire, requiring manual re-entry of the cookie string through DevTools
- Blocks: Unattended long-term usage without periodic manual intervention

**No Error Recovery for Expired Cookies:**
- Problem: When a cookie expires, the app shows "HTTP 401" or "HTTP 403" with no guidance to the user about re-authenticating. The status bar icon shows 0% which looks like normal low usage
- Blocks: Users may not realize their data is stale until they open the popover

## Test Coverage Gaps

**No Tests Exist:**
- What's not tested: Everything - there is zero test infrastructure. No unit tests, no UI tests, no integration tests
- Files: `app/ClaudeUsageBar.swift` (entire file)
- Risk: Any change to API parsing, notification logic, cookie handling, or percentage calculations could introduce silent regressions
- Priority: High for `parseUsageData()`, `checkNotificationThresholds()`, `fetchOrganizationId()`, and `updateStatusBar()` which contain the core business logic

---

*Concerns audit: 2026-03-05*
