# Technology Stack

**Analysis Date:** 2026-03-05

## Languages

**Primary:**
- Swift - Native macOS app (`app/ClaudeUsageBar.swift`, single-file application, ~1139 lines)

**Secondary:**
- HTML/CSS/JavaScript - Marketing website (`website/index.html`, single-page static site, ~1446 lines)
- Bash - Build tooling (`app/build.sh`, `app/create_dmg.sh`, `app/make_app_icon.sh`)

## Runtime

**Environment:**
- macOS 12.0 (Monterey) minimum, per `app/Info.plist` `LSMinimumSystemVersion`
- No external runtime dependencies -- compiles to native binary

**Package Manager:**
- None. No dependency manager (no SPM Package.swift, no CocoaPods, no Carthage)
- All dependencies are Apple system frameworks

## Frameworks

**Core (Apple System Frameworks):**
- SwiftUI - UI layer for popover content (`UsageView`, `PasteableTextField`)
- AppKit - Menu bar integration, status item, popover management, notifications (`NSStatusBar`, `NSPopover`, `NSUserNotification`)
- WebKit - Imported but not actively used in current code
- Carbon - Global keyboard shortcut registration (`EventHotKeyRef`, `RegisterEventHotKey`)

**Testing:**
- None configured. No test files or test framework present.

**Build/Dev:**
- `swiftc` (Swift compiler) - Direct compilation without Xcode project
- `lipo` - Universal binary creation (arm64 + x86_64)
- `codesign` - Code signing (Developer ID or ad-hoc fallback)
- `hdiutil` - DMG creation for distribution
- `iconutil` / `sips` - Icon generation from PNG source

## Key Dependencies

**Critical:**
- `URLSession` (Foundation) - HTTP client for Claude.ai API requests
- `UserDefaults` - Local persistence for session cookie, settings, notification thresholds
- `NSStatusBar` / `NSStatusItem` - Menu bar presence
- `NSUserNotification` (deprecated API) - Desktop notifications without permission prompts

**Infrastructure:**
- No external dependencies. Entire app uses only Apple system frameworks.

## Configuration

**Environment:**
- No `.env` files or environment variables
- All configuration stored in macOS `UserDefaults` with these keys:
  - `claude_session_cookie` - Full browser cookie string for claude.ai authentication
  - `notifications_enabled` / `has_set_notifications` - Notification preferences
  - `open_at_login` - Launch at login toggle
  - `shortcut_enabled` - Cmd+U global hotkey toggle
  - `last_notified_threshold` - Tracks last usage threshold that triggered a notification

**Build:**
- `app/Info.plist` - App bundle configuration (bundle ID: `com.claude.usagebar`, version 1.0)
- `app/build.sh` - Build script, targets `arm64-apple-macos12.0` and `x86_64-apple-macos12.0`
- Code signing identity: `Developer ID Application: Linkko Technology Pte Ltd (Q467HQ5432)` (falls back to ad-hoc)

**App Transport Security:**
- Configured in `app/Info.plist` to allow HTTPS only
- Exception domain: `anthropic.com` (includes subdomains, requires forward secrecy)

## Platform Requirements

**Development:**
- macOS with Xcode Command Line Tools (provides `swiftc`, `lipo`, `codesign`)
- No Xcode project required -- builds from command line
- Build command: `cd app && ./build.sh`

**Production:**
- macOS 12.0 (Monterey) or later
- Universal binary supports both Apple Silicon (M1/M2/M3) and Intel Macs
- `LSUIElement: true` -- runs as menu bar app only (no Dock icon)
- Optional: Accessibility permission for global Cmd+U keyboard shortcut

**Distribution:**
- DMG installer created via `app/create_dmg.sh`
- GitHub Releases at `github.com/Artzainnn/ClaudeUsageBar/releases`

## Website Stack

**Hosting:**
- Static site at `claudeusagebar.com`
- Single HTML file with inline CSS and JavaScript (`website/index.html`)

**External Resources:**
- Google Fonts: Inter (400/500/600/700), JetBrains Mono (500)
- DataFast analytics: `datafa.st/js/script.js` (website ID: `dfid_HMM2pp9guqaCpR3KioH9l`)
- GitHub API: `api.github.com/repos/Artzainnn/ClaudeUsageBar` (fetches star count)

---

*Stack analysis: 2026-03-05*
