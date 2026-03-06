# Codebase Structure

**Analysis Date:** 2026-03-05

## Directory Layout

```
ClaudeUsageBar/
├── app/                        # macOS menu bar application
│   ├── ClaudeUsageBar.swift    # Entire app source (1138 lines)
│   ├── build.sh                # Universal binary build script
│   ├── create_dmg.sh           # DMG installer creation script
│   ├── make_app_icon.sh        # Icon generation from PNG
│   ├── Info.plist               # macOS app bundle config
│   ├── ClaudeUsageBar.icns     # App icon (pre-built)
│   ├── claudeusagebar-icon.png # Source icon PNG
│   ├── LICENSE                 # MIT license (app copy)
│   └── README.md               # App-specific documentation
├── website/                    # Landing page (claudeusagebar.com)
│   ├── index.html              # Single-page site with inline CSS (1446 lines)
│   ├── favicon.svg             # SVG favicon
│   ├── apple-touch-icon.png    # iOS home screen icon
│   ├── og-image.png            # Open Graph social preview
│   ├── .gitignore              # Vercel/env exclusions
│   └── README.md               # Website documentation
├── .planning/                  # GSD planning directory
│   └── codebase/               # Codebase analysis documents
├── .gitignore                  # Root gitignore
├── LICENSE                     # MIT license (root)
├── README.md                   # Project overview/readme
├── og-image.png                # Open Graph image (root copy)
└── setup-guide.png             # Setup instructions screenshot
```

## Directory Purposes

**app/:**
- Purpose: Complete macOS menu bar application
- Contains: Single Swift source file, shell build scripts, app bundle metadata, icon assets
- Key files: `ClaudeUsageBar.swift` (all application code), `build.sh` (compilation), `Info.plist` (bundle config)

**website/:**
- Purpose: Static landing/marketing page deployed to claudeusagebar.com via Vercel
- Contains: Single self-contained HTML file with inline CSS, favicon/OG assets
- Key files: `index.html` (entire site)

**.planning/codebase/:**
- Purpose: GSD codebase analysis documents
- Contains: Architecture and structure analysis markdown files
- Generated: Yes (by GSD tooling)
- Committed: Yes

## Key File Locations

**Entry Points:**
- `app/ClaudeUsageBar.swift`: `@main struct Main` at line 287 -- the sole application entry point

**Configuration:**
- `app/Info.plist`: macOS bundle identifier (`com.claude.usagebar`), minimum OS version (12.0), `LSUIElement` (menu bar only), App Transport Security exceptions for anthropic.com
- `.gitignore`: Excludes `app/build/`, `.dmg`, `.iconset/`, IDE files, `node_modules/`, `.env`

**Core Logic:**
- `app/ClaudeUsageBar.swift` lines 7-271: `AppDelegate` -- app lifecycle, status bar, popover, hotkey
- `app/ClaudeUsageBar.swift` lines 298-674: `UsageManager` -- API client, state, persistence, notifications
- `app/ClaudeUsageBar.swift` lines 677-807: Custom text input components (paste support)
- `app/ClaudeUsageBar.swift` lines 809-1138: `UsageView` -- SwiftUI popover UI

**Build/Distribution:**
- `app/build.sh`: Compiles universal binary (arm64 + x86_64), creates `.app` bundle, code-signs
- `app/create_dmg.sh`: Packages `.app` into DMG installer with Applications symlink
- `app/make_app_icon.sh`: Generates `.icns` from source PNG using `sips` and `iconutil`

**Testing:**
- None. No test files exist in the project.

## Naming Conventions

**Files:**
- Swift source: PascalCase matching app name (`ClaudeUsageBar.swift`)
- Shell scripts: snake_case (`build.sh`, `create_dmg.sh`, `make_app_icon.sh`)
- Config files: Standard names (`Info.plist`, `.gitignore`)
- Images: kebab-case (`claudeusagebar-icon.png`, `og-image.png`, `setup-guide.png`)

**Swift Types:**
- Classes: PascalCase (`AppDelegate`, `UsageManager`, `CustomTextField`, `PasteableNSTextView`)
- Structs: PascalCase (`Main`, `UsageView`, `PasteableTextField`)
- Properties: camelCase (`sessionCookie`, `weeklyResetsAt`, `hasFetchedData`)
- Functions: camelCase (`fetchUsage()`, `parseUsageData()`, `updateStatusBar()`)

**Directories:**
- Lowercase (`app/`, `website/`)

## Where to Add New Code

**New Feature (in the app):**
- Primary code: `app/ClaudeUsageBar.swift` -- add new classes/structs in the appropriate section (data logic near `UsageManager`, views near `UsageView`)
- Consider: The single-file architecture means all new code goes into `app/ClaudeUsageBar.swift`. If the file grows significantly, consider splitting into multiple Swift files and updating `build.sh` to compile them all.

**New SwiftUI View:**
- Implementation: Add struct conforming to `View` in `app/ClaudeUsageBar.swift` near the existing `UsageView` (after line 809)
- Wire into popover or as a sub-view of `UsageView`

**New Build Script:**
- Location: `app/` directory alongside existing `.sh` files
- Follow convention: `snake_case.sh`, make executable (`chmod +x`)

**New Website Page:**
- Location: `website/` directory
- The current site is a single HTML file; add new pages as separate HTML files or refactor to a static site generator

**New Settings/Preferences:**
- Add `@Published` property to `UsageManager` class
- Persist via `UserDefaults` in `loadSettings()` / `saveSettings()`
- Add UI toggle in `UsageView`'s settings section (inside `if showingSettings` block)

## Special Directories

**app/build/ (gitignored):**
- Purpose: Contains compiled `.app` bundle after running `build.sh`
- Generated: Yes
- Committed: No

**website/.vercel (gitignored):**
- Purpose: Vercel deployment cache
- Generated: Yes
- Committed: No

---

*Structure analysis: 2026-03-05*
