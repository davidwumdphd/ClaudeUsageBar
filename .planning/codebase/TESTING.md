# Testing Patterns

**Analysis Date:** 2026-03-05

## Test Framework

**Runner:** None

No test framework is configured. No test files exist anywhere in the codebase.

**Assertion Library:** None

**Run Commands:**
```bash
# No test commands available
# The app is compiled directly with swiftc, not through Xcode or Swift Package Manager
```

## Test File Organization

**Location:** No test files exist.

No files matching `*Test.swift`, `*Tests.swift`, `*.test.*`, or `*.spec.*` were found in the repository.

## Current Build/Verification

The only verification available is compilation:

```bash
cd /Users/dwu/Code/ClaudeUsageBar/app && ./build.sh
```

This compiles `app/ClaudeUsageBar.swift` with `swiftc` for both arm64 and x86_64, creates a universal binary, signs it, and launches the app. A successful build is the only automated check.

## Why There Are No Tests

The project has characteristics that make traditional testing difficult:

1. **Single-file architecture:** All 1,138 lines live in `app/ClaudeUsageBar.swift` with no module boundaries
2. **No Swift Package Manager:** The app compiles via raw `swiftc` in `app/build.sh`, so there is no `Package.swift` to add test targets to
3. **Heavy AppKit/SwiftUI coupling:** Most logic is intertwined with UI framework classes (`NSStatusItem`, `NSPopover`, `URLSession`, `UserDefaults`)
4. **No dependency injection:** `UsageManager` directly calls `URLSession.shared`, `UserDefaults.standard`, and `NSUserNotificationCenter.default`

## What Could Be Tested

If tests were to be added, these areas have extractable logic:

**Pure logic in `app/ClaudeUsageBar.swift`:**
- `parseUsageData()` (line 522) - JSON parsing of API response into usage values
- `checkNotificationThresholds()` (line 612) - Threshold detection logic
- `formatResetTime()` (line 1114) - Date formatting
- `colorForPercentage()` (line 1128) - Color threshold mapping
- `updatePercentages()` (line 669) - Percentage calculation

**To enable testing, these would need to be:**
1. Extracted into a testable module or standalone functions
2. Given injectable dependencies (protocol-based URLSession, UserDefaults wrapper)
3. Built via Swift Package Manager with a test target

## Coverage

**Requirements:** None enforced

**Current coverage:** 0% - no tests exist

## Test Types

**Unit Tests:** Not present

**Integration Tests:** Not present

**E2E Tests:** Not present

**Manual Testing:** The primary verification method. The build script (`app/build.sh`) automatically launches the app after a successful build, enabling manual verification.

## Recommended Test Setup (If Adding Tests)

To introduce testing to this project:

1. **Create `Package.swift`** at the repo root or in `app/` with a library target for testable logic and an executable target for the app
2. **Extract testable logic** from `UsageManager` into pure functions or a protocol-backed service
3. **Use XCTest** (the standard Swift testing framework):

```swift
// Example structure if tests were added:
// app/Tests/UsageParsingTests.swift

import XCTest
@testable import ClaudeUsageBarLib

final class UsageParsingTests: XCTestCase {
    func testParseUsageData_withValidJSON_setsSessionUsage() {
        let json = """
        {"five_hour": {"utilization": 42.0, "resets_at": "2026-03-05T12:00:00.000Z"}}
        """.data(using: .utf8)!

        let manager = UsageManager(statusItem: nil)
        manager.parseUsageData(json)

        XCTAssertEqual(manager.sessionUsage, 42)
    }

    func testCheckNotificationThresholds_at75Percent_sendsNotification() {
        // ...
    }
}
```

4. **Run with:** `swift test` (requires Package.swift)

## Mocking

**Framework:** None (XCTest's built-in mocking or a library like `swift-testing` would be needed)

**Pattern for injectable dependencies (not yet implemented):**
```swift
protocol URLSessionProtocol {
    func dataTask(with request: URLRequest, completionHandler: @escaping (Data?, URLResponse?, Error?) -> Void) -> URLSessionDataTask
}

extension URLSession: URLSessionProtocol {}

class UsageManager: ObservableObject {
    private let session: URLSessionProtocol

    init(statusItem: NSStatusItem?, session: URLSessionProtocol = URLSession.shared) {
        self.session = session
    }
}
```

---

*Testing analysis: 2026-03-05*
