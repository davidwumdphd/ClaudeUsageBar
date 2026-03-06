# Coding Conventions

**Analysis Date:** 2026-03-05

## Codebase Overview

This is a single-file macOS menu bar app written in Swift, with a companion static HTML marketing website. The entire app logic lives in one file: `app/ClaudeUsageBar.swift` (1,138 lines). There is no package manager, no Xcode project, and no build system beyond a shell script.

## Naming Patterns

**Files:**
- Swift source: PascalCase matching the app name (`ClaudeUsageBar.swift`)
- Shell scripts: snake_case with `.sh` extension (`build.sh`, `create_dmg.sh`, `make_app_icon.sh`)
- Config files: standard names (`Info.plist`, `.gitignore`)

**Classes:**
- PascalCase: `AppDelegate`, `UsageManager`, `CustomTextField`, `PasteableNSTextView`
- Classes named for their role/purpose

**Structs (SwiftUI Views):**
- PascalCase: `UsageView`, `PasteableTextField`, `Main`
- Views suffixed with `View` when they are SwiftUI views

**Functions:**
- camelCase with verb prefixes: `fetchUsage()`, `parseUsageData()`, `updateStatusBar()`, `loadSettings()`, `saveSettings()`
- Boolean checks use `check` prefix: `checkAccessibilityPermissions()`, `checkNotificationThresholds()`
- Format helpers use `format` prefix: `formatNumber()`, `formatTime()`, `formatResetTime()`

**Variables / Properties:**
- camelCase: `sessionUsage`, `weeklyLimit`, `sessionCookie`, `lastNotifiedThreshold`
- Published properties: camelCase with no prefix: `@Published var isLoading`, `@Published var errorMessage`
- Private properties: explicit `private` keyword: `private var statusItem`, `private var sessionCookie`
- Boolean properties: use `is`/`has` prefix: `isLoading`, `hasFetchedData`, `hasWeeklySonnet`, `isAccessibilityEnabled`

**UserDefaults Keys:**
- snake_case strings: `"claude_session_cookie"`, `"notifications_enabled"`, `"open_at_login"`, `"shortcut_enabled"`, `"last_notified_threshold"`, `"has_set_notifications"`

## Code Style

**Formatting:**
- No explicit formatter configured (no SwiftFormat or SwiftLint config files)
- 4-space indentation throughout `app/ClaudeUsageBar.swift`
- Opening braces on same line as declaration
- Single blank line between methods
- No trailing commas

**Linting:**
- No linter configured. No `.swiftlint.yml` or similar config.

## Architecture Pattern

**Single-file monolith:** All code lives in `app/ClaudeUsageBar.swift`. Classes/structs are arranged in this order:

1. **`AppDelegate`** (line 7) - App lifecycle, status bar, popover, hotkey management
2. **`NSColor` extension** (line 273) - Hex string conversion utility
3. **`Main` struct** (line 287) - `@main` entry point
4. **`UsageManager`** (line 298) - Data fetching, parsing, state management (ObservableObject)
5. **`CustomTextField`** (line 677) - NSTextField subclass for paste support
6. **`PasteableNSTextView`** (line 719) - NSTextView subclass for paste support
7. **`PasteableTextField`** (line 744) - NSViewRepresentable bridge for SwiftUI
8. **`UsageView`** (line 809) - Main SwiftUI popover view

## Import Organization

**Order in `app/ClaudeUsageBar.swift`:**
1. Apple frameworks in a single block, no blank lines between them:
```swift
import SwiftUI
import AppKit
import WebKit
import Carbon
```

No third-party imports exist. Use only Apple system frameworks.

## State Management

**Pattern:** ObservableObject + @Published properties

- `UsageManager` is an `ObservableObject` with `@Published` properties for all UI-visible state
- `UsageView` observes it via `@ObservedObject var usageManager: UsageManager`
- Local view state uses `@State`: `@State private var sessionCookieInput`, `@State private var showingCookieInput`
- Bindings created inline with `Binding(get:set:)` pattern for settings toggles

Example pattern from `app/ClaudeUsageBar.swift` line 1004:
```swift
Toggle(isOn: Binding(
    get: { usageManager.openAtLogin },
    set: { newValue in
        usageManager.openAtLogin = newValue
        usageManager.saveSettings()
    }
)) {
    // label
}
```

## Persistence

**UserDefaults** is the sole persistence mechanism. Pattern:
```swift
// Read
UserDefaults.standard.string(forKey: "claude_session_cookie")
UserDefaults.standard.bool(forKey: "notifications_enabled")

// Write
UserDefaults.standard.set(value, forKey: "key")
UserDefaults.standard.synchronize()  // Called explicitly after writes
```

Note: `synchronize()` is called after every write, which is technically unnecessary on modern macOS but is the established pattern here. Follow this convention for consistency.

## Networking

**Pattern:** URLSession with completion handlers (not async/await)

- Build `URLRequest` manually with explicit headers
- Use `URLSession.shared.dataTask(with:)` with closure callbacks
- Parse JSON with `JSONSerialization` (no Codable)
- All UI updates dispatched to main queue via `DispatchQueue.main.async`

Example from `app/ClaudeUsageBar.swift` line 488:
```swift
URLSession.shared.dataTask(with: request) { [weak self] data, response, error in
    DispatchQueue.main.async {
        self?.isLoading = false
        if let error = error {
            self?.errorMessage = "Network error"
            return
        }
        // ...parse response...
    }
}.resume()
```

## Error Handling

**Strategy:** Error messages stored in published property, displayed in UI

- Errors set via `errorMessage: String?` on `UsageManager`
- No `throw`/`catch` propagation; errors are caught at the call site and converted to user-facing strings
- Guard statements for early returns: `guard !sessionCookie.isEmpty else { ... return }`
- Optional chaining with `self?` in closures using `[weak self]`
- JSON parsing uses `try?` (silent failure) rather than `try`/`catch` for the bootstrap API call

Error messages are short, technical strings: `"Session cookie not set"`, `"Could not get org ID from cookie"`, `"Invalid URL"`, `"Network error"`, `"HTTP \(statusCode)"`, `"Invalid JSON"`, `"Parse error"`

## Logging

**Framework:** `NSLog` (Apple's system logging)

**Pattern:**
- Emoji prefixes indicate log type:
  - `"... App launched..."` - Success/info
  - `"... Accessibility permissions not granted"` - Warnings
  - `"... Could not parse org ID"` - Errors
  - `"... Fetching from: ..."` - Network activity
  - `"... Parsing usage data..."` - Data processing
  - `"... Checking notifications..."` - Debug flow
- Logs include relevant data values: `NSLog("ClaudeUsage: Saving cookie, length: \(cookie.count)")`
- Some logs use `"ClaudeUsage:"` prefix, others use emoji only (inconsistent)

## Comments

**Style:** Single-line `//` comments explaining "what" the next block does

- Comments placed above code blocks, not inline
- Used to label sections: `// Main entry point`, `// Session Usage`, `// Settings Section`
- Implementation notes in comments: `// NSUserNotification (deprecated but works without permissions for unsigned apps)`
- No documentation comments (no `///` or `/** */`)

## SwiftUI View Patterns

**Layout approach:**
- `VStack(alignment: .leading, spacing: N)` as primary layout container
- `HStack` for horizontal arrangements with `Spacer()` for alignment
- Explicit `.frame(width: 360)` on root view
- `.padding()` on root view, `.padding(8)` on subsections
- `.background(Color.secondary.opacity(0.1))` for section backgrounds
- `.cornerRadius(6)` for rounded sections

**Typography:**
- `.font(.headline)` for titles
- `.font(.subheadline)` for section headers
- `.font(.caption)` for secondary text and buttons
- `.font(.caption2)` for tertiary/help text
- `.foregroundColor(.secondary)` for muted text
- `.foregroundColor(.orange)` for warnings/accent

**Buttons:**
- `.buttonStyle(.borderless)` for text-link buttons
- `.buttonStyle(.borderedProminent)` for primary actions
- `.buttonStyle(.bordered)` for secondary actions
- `.controlSize(.small)` for compact buttons

## Shell Script Conventions

**In `app/build.sh`, `app/create_dmg.sh`, `app/make_app_icon.sh`:**
- `#!/bin/bash` shebang
- SCREAMING_SNAKE_CASE for variables: `APP_NAME`, `DMG_NAME`, `VERSION`, `TMP_DIR`
- Echo with emoji for status: `echo "... Build successful!"`
- Quoted variable references: `"$APP_PATH"`, `"${APP_NAME}"`
- Comments above significant operations

## Website Conventions

**In `website/index.html`:**
- Single-file static site (HTML + inline CSS + inline JS)
- CSS custom properties for theming (`--claude-orange`, `--bg-dark`, etc.)
- BEM-like class naming: `.feature-grid`, `.nav-content`, `.logo-icon`
- Google Fonts: Inter (body) and JetBrains Mono (code)
- Dark theme only with Claude brand orange (`#da7756`)

---

*Convention analysis: 2026-03-05*
