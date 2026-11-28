# Team 1: cache-cleaner (caches and temp files)

This file is the body of Team 1's prompt. Core principles, Data-loss rules, and the security keyword list from SKILL.md are pasted into the same prompt; where this file seems to allow more, they win.

Tasks:
- `pip cache purge`
- `npm cache clean --force`
- Move log files older than 30 days in `~/Library/Logs` to `/tmp/`
- Clean browser caches (Whale, Chrome, and so on) by moving them to `/tmp/`
- Clean the safe parts of JetBrains caches (backup, download, temp). Leave `~/Library/Caches/JetBrains/Toolbox/backup` out in a run where Toolbox updated an IDE: it holds the previous builds for rollback
- In `~/Library/Logs/JetBrains`, move log directories of versions lower than the installed one, and of products that are not installed, to `/tmp/` (read the installed version with `defaults read "<app>/Contents/Info.plist" CFBundleShortVersionString`)
- In the user temp directory (`getconf DARWIN_USER_TEMP_DIR`), list the largest top-level items with their size and modification time. Report only; do not move anything there (sockets, `TemporaryItems`, `com.apple.*`, and `*.code_sign_clone` entries can belong to live daemons or updaters)
- Measure size before and after each task

Notes:
- Keep core IDE caches (indexes and the like)
- Do not run brew commands (they collide with Team 2's upgrade lock; Team 2 runs cleanup after the upgrade)
- Before calling something an "old version cache", check whether the app is installed (`/Applications`, `~/Applications`). The cache of an installed version is not old
