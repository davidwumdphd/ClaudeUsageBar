# ClaudeUsageBar

## Build

- Build script: `cd app && bash build.sh` (compiles universal binary, signs, launches)
- Build output: `app/build/ClaudeUsageBar.app` (only source of truth — no root-level build dir)
- Install: `cp -r app/build/ClaudeUsageBar.app /Applications/`
- Ad-hoc signed unless Developer ID cert is available on the machine
