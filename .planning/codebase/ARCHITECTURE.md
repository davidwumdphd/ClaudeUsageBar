# Architecture

**Analysis Date:** 2026-03-05

## Pattern Overview

**Overall:** Single-file monolithic macOS menu bar app with a separate static landing page

**Key Characteristics:**
- Entire native macOS app lives in one 1138-line Swift file (`app/ClaudeUsageBar.swift`)
- No Xcode project -- compiled directly via `swiftc` shell scripts
- Uses AppKit's `NSStatusItem` for menu bar presence (no Dock icon, `LSUIElement = true`)
- SwiftUI `NSPopover` for the popup UI, bridged through `NSHostingController`
- Static HTML landing page at `website/index.html` with inline CSS (no build step)

## Layers

**App Lifecycle (AppDelegate):**
- Purpose: Application entry point, status bar setup, popover management, global hotkey registration
- Location: `app/ClaudeUsageBar.swift` lines 7-271
- Contains: `AppDelegate` class, `Main` struct (`@main` entry point), `NSColor` hex extension
- Depends on: `UsageManager` for data, `UsageView` for popover content
- Used by: macOS app runtime

**Data Layer (UsageManager):**
- Purpose: Fetches usage data from Claude.ai API, manages state, persists settings, sends notifications
- Location: `app/ClaudeUsageBar.swift` lines 298-674
- Contains: `UsageManager` class (ObservableObject) with all published state
- Depends on: `URLSession` for HTTP, `UserDefaults` for persistence, `AppDelegate` for status icon updates
- Used by: `UsageView` (via `@ObservedObject`), `AppDelegate` (for data fetching triggers)

**View Layer (SwiftUI):**
- Purpose: Popover UI showing usage bars, cookie input, and settings
- Location: `app/ClaudeUsageBar.swift` lines 809-1138
- Contains: `UsageView` struct, helper formatting functions
- Depends on: `UsageManager` (data binding)
- Used by: `AppDelegate` (embedded in `NSPopover` via `NSHostingController`)

**Custom Input Components:**
- Purpose: AppKit text input components that properly handle paste (Cmd+V) in popovers
- Location: `app/ClaudeUsageBar.swift` lines 677-807
- Contains: `CustomTextField` (NSTextField subclass), `PasteableNSTextView` (NSTextView subclass), `PasteableTextField` (NSViewRepresentable bridge)
- Depends on: AppKit
- Used by: `UsageView` for cookie input

**Landing Page:**
- Purpose: Marketing/download page for the app
- Location: `website/index.html`
- Contains: Single self-contained HTML file with inline CSS and no JavaScript framework
- Depends on: Google Fonts (Inter, JetBrains Mono), hosted on Vercel at claudeusagebar.com
- Used by: End users discovering the app

## Data Flow

**Usage Fetch Flow:**

1. `AppDelegate.applicationDidFinishLaunching` calls `usageManager.fetchUsage()` (also on 5-minute timer)
2. `UsageManager.fetchUsage()` checks for session cookie in `UserDefaults`, then calls `fetchOrganizationId()`
3. `fetchOrganizationId()` extracts `lastActiveOrg` from cookie string, or falls back to `https://claude.ai/api/bootstrap`
4. `fetchUsageWithOrgId()` makes GET request to `https://claude.ai/api/organizations/{orgId}/usage` with full cookie header
5. `parseUsageData()` extracts `five_hour`, `seven_day`, and optionally `seven_day_sonnet` utilization percentages and reset times
6. Published properties update, triggering SwiftUI view refresh
7. `updateStatusBar()` calls `AppDelegate.updateStatusIcon()` to change menu bar icon color
8. `checkNotificationThresholds()` fires `NSUserNotification` at 25/50/75/90% thresholds

**Cookie Setup Flow:**

1. User clicks "Set Session Cookie" in popover, revealing `PasteableTextField`
2. User pastes full browser cookie string (from claude.ai Network tab)
3. "Save Cookie & Fetch" button calls `usageManager.saveSessionCookie()` (persists to `UserDefaults`) then `fetchUsage()`
4. Org ID extracted from `lastActiveOrg=` segment of cookie string

**State Management:**
- `UsageManager` is an `ObservableObject` with `@Published` properties for all UI state
- `UsageView` observes via `@ObservedObject`, getting automatic SwiftUI re-renders
- Persistent state stored in `UserDefaults`: session cookie, notification preferences, shortcut toggle, open-at-login, last notified threshold

## Key Abstractions

**UsageManager:**
- Purpose: Central state holder and API client -- combines data fetching, persistence, and notification logic
- Examples: `app/ClaudeUsageBar.swift` (single class, lines 298-674)
- Pattern: ObservableObject with @Published properties for SwiftUI binding

**AppDelegate:**
- Purpose: Owns the status bar item, popover, and global hotkey lifecycle
- Examples: `app/ClaudeUsageBar.swift` (lines 7-271)
- Pattern: Traditional NSApplicationDelegate with manual NSStatusItem setup

**PasteableTextField:**
- Purpose: Bridges AppKit's NSTextView into SwiftUI to work around paste issues in popovers
- Examples: `app/ClaudeUsageBar.swift` (lines 744-807)
- Pattern: NSViewRepresentable with Coordinator for delegate callbacks

## Entry Points

**App Entry Point:**
- Location: `app/ClaudeUsageBar.swift` lines 287-296, `@main struct Main`
- Triggers: macOS launches the compiled binary
- Responsibilities: Creates `NSApplication`, sets `AppDelegate`, sets activation policy to `.accessory` (no Dock icon), starts run loop

**Status Bar Click:**
- Location: `app/ClaudeUsageBar.swift` line 168, `handleClick()`
- Triggers: User left-clicks or right-clicks menu bar icon
- Responsibilities: Left click toggles popover; right click shows context menu (toggle, quit)

**Global Hotkey (Cmd+U):**
- Location: `app/ClaudeUsageBar.swift` lines 98-142, `registerGlobalHotKey()`
- Triggers: User presses Cmd+U anywhere in macOS
- Responsibilities: Toggles popover via Carbon `EventHotKey` API

**Timer (Auto-refresh):**
- Location: `app/ClaudeUsageBar.swift` line 46
- Triggers: Every 300 seconds (5 minutes)
- Responsibilities: Calls `usageManager.fetchUsage()` to refresh usage data

## Error Handling

**Strategy:** Minimal -- log via `NSLog`, set `errorMessage` published property for UI display

**Patterns:**
- Network errors surface as `"Network error"` in the popover via `usageManager.errorMessage`
- HTTP non-200 responses show `"HTTP {statusCode}"` in the popover
- JSON parse failures show `"Parse error"` or `"Invalid JSON"`
- Missing cookie shows `"Session cookie not set"`
- All errors are logged with `NSLog` using emoji prefixes for grep-ability

## Cross-Cutting Concerns

**Logging:** `NSLog` throughout with emoji prefixes (checkmark for success, X for error, etc.)
**Validation:** Cookie presence checked before API calls; no further input validation
**Authentication:** Session cookie from claude.ai browser session, stored in `UserDefaults` -- user manually copies from browser DevTools
**Persistence:** All settings and cookie stored via `UserDefaults.standard` with explicit `synchronize()` calls
**Notifications:** `NSUserNotification` (deprecated API) used intentionally because it works without requesting notification permissions on unsigned apps

---

*Architecture analysis: 2026-03-05*
