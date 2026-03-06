# External Integrations

**Analysis Date:** 2026-03-05

## APIs & External Services

**Claude.ai (Primary -- macOS app):**
- Purpose: Fetches usage data for the authenticated user's Claude.ai account
- Endpoints:
  - `GET https://claude.ai/api/bootstrap` - Retrieves organization ID from account data (fallback when `lastActiveOrg` cookie is missing)
  - `GET https://claude.ai/api/organizations/{orgId}/usage` - Returns session and weekly usage utilization percentages
- SDK/Client: `URLSession` (Apple Foundation framework), no third-party HTTP client
- Auth: Browser session cookie (full cookie string stored in `UserDefaults` key `claude_session_cookie`)
- Headers sent: Cookie, Accept, Content-Type, Origin, Referer, User-Agent (Chrome spoof), authority
- Implementation: `app/ClaudeUsageBar.swift` lines 398-600 (`fetchOrganizationId`, `fetchUsage`, `fetchUsageWithOrgId`, `parseUsageData`)
- Refresh interval: Every 300 seconds (5 minutes) via `Timer.scheduledTimer`

**Claude.ai API Response Format:**
```json
{
  "five_hour": {
    "utilization": 45.0,
    "resets_at": "2026-03-05T12:00:00.000Z"
  },
  "seven_day": {
    "utilization": 30.0,
    "resets_at": "2026-03-10T00:00:00.000Z"
  },
  "seven_day_sonnet": {
    "utilization": 20.0,
    "resets_at": "2026-03-10T00:00:00.000Z"
  }
}
```
- `five_hour` - Session usage (always present)
- `seven_day` - Weekly usage (always present)
- `seven_day_sonnet` - Weekly Sonnet usage (Pro plan only, optional)

**GitHub API (Website only):**
- Purpose: Displays star count on marketing website
- Endpoint: `GET https://api.github.com/repos/Artzainnn/ClaudeUsageBar`
- Auth: None (unauthenticated, subject to rate limiting)
- Implementation: `website/index.html` line 1425 (`fetchStarCount` function)

**Stripe (Donation link only):**
- Purpose: "Buy Dev a Coffee" donation link
- URL: `https://donate.stripe.com/3cIcN5b5H7Q8ay8bIDfIs02`
- Implementation: Link in macOS app (`app/ClaudeUsageBar.swift` line 984) and website (`website/index.html` lines 1052, 1353)
- No Stripe SDK integration -- opens external browser to Stripe-hosted donation page

**DataFast Analytics (Website only):**
- Purpose: Website analytics for `claudeusagebar.com`
- Script: `https://datafa.st/js/script.js`
- Website ID: `dfid_HMM2pp9guqaCpR3KioH9l`
- Domain: `claudeusagebar.com`
- Goal tracking events: `download_macos`, `view_github`, `buy_coffee`
- Implementation: `website/index.html` lines 1003-1014, `data-fast-goal` attributes on links

## Data Storage

**Databases:**
- None. No database used.

**Local Persistence:**
- macOS `UserDefaults` (plist-backed key-value store)
  - Keys: `claude_session_cookie`, `notifications_enabled`, `has_set_notifications`, `open_at_login`, `shortcut_enabled`, `last_notified_threshold`
  - Implementation: `app/ClaudeUsageBar.swift` (`loadSessionCookie`, `loadSettings`, `saveSettings`, `saveSessionCookie`, `clearSessionCookie`)

**File Storage:**
- Not applicable. No file storage used.

**Caching:**
- None. Usage data is re-fetched on each interval. In-memory `@Published` properties on `UsageManager` hold current state.

## Authentication & Identity

**Auth Approach:**
- Cookie-based authentication against `claude.ai`
- User manually extracts full cookie string from browser DevTools (Network tab, Cookie request header)
- Cookie string includes `sessionKey`, `lastActiveOrg`, and other Claude.ai cookies
- Organization ID extracted from `lastActiveOrg` cookie value, or fetched from `/api/bootstrap` as fallback
- No OAuth, no API keys, no user accounts

**Cookie Flow:**
1. User copies cookie from browser DevTools
2. Pastes into app's "Set Session Cookie" text field
3. Saved to `UserDefaults` as `claude_session_cookie`
4. Sent as `Cookie` header on all API requests to `claude.ai`
5. Expires when Claude.ai session expires (user must re-paste)

## Monitoring & Observability

**Error Tracking:**
- None. No external error tracking service.
- Errors displayed in-app via `usageManager.errorMessage` (shown in popover UI)

**Logs:**
- `NSLog` statements throughout `app/ClaudeUsageBar.swift`
- Viewable via macOS Console.app
- Log prefixes used: checkmark, cross, clipboard, satellite, package, bell, clock, warning emojis

## CI/CD & Deployment

**macOS App:**
- No CI/CD pipeline configured
- Manual build via `app/build.sh`
- Manual DMG creation via `app/create_dmg.sh`
- Distribution via GitHub Releases (`github.com/Artzainnn/ClaudeUsageBar/releases`)

**Website:**
- Static HTML file (`website/index.html`)
- Hosted at `claudeusagebar.com` (hosting provider not specified in codebase)
- No build step required

## Environment Configuration

**Required env vars:**
- None. The app has no environment variable dependencies.

**Secrets:**
- Claude.ai session cookie stored in macOS `UserDefaults` (local to user's machine)
- Code signing identity hardcoded in `app/build.sh` (falls back to ad-hoc if not available)
- DataFast website ID hardcoded in `website/index.html`

## Webhooks & Callbacks

**Incoming:**
- None

**Outgoing:**
- None

## Important Notes

- The Claude.ai API endpoints used (`/api/bootstrap`, `/api/organizations/{orgId}/usage`) are **internal/undocumented** endpoints. They may change without notice as noted in `app/README.md` disclaimer.
- The app spoofs a Chrome User-Agent header to mimic browser requests.
- `NSUserNotification` (used for desktop alerts) is deprecated by Apple but chosen deliberately because it works without requiring notification permissions for unsigned apps.

---

*Integration audit: 2026-03-05*
