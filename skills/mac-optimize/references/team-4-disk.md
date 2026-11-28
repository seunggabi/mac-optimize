# Team 4: disk-analyzer (disk space)

This file is the body of Team 4's prompt. Core principles, Data-loss rules, and the security keyword list from SKILL.md are pasted into the same prompt, and the prompt states the scope in its first line; where this file seems to allow more, they and the scope win.

Tasks:
- Top 15 of the home directory with `du -sh ~/*`
- Large directories inside `~/Library`
- Docker status (`docker system df`). Report only: cleaning is decided in Phase 3. Report volumes separately because they may hold database data
- Size of `/tmp$HOME` (items moved by earlier runs may have piled up)
- Size of the user temp and cache directories under `/private/var/folders` (`getconf DARWIN_USER_TEMP_DIR`, `DARWIN_USER_CACHE_DIR`)
- Size and last-used date per app (`mdls -raw -name kMDItemLastUsedDate <app>`). Flag apps installed twice and large apps unused for more than 180 days (list only, do not delete). `mdls` often has no date: it prints `(null)` or "could not find kMDItemLastUsedDate" rather than nothing (it did for every app but Safari in one run). Fall back to the access time of the main executable (`stat -f %Sa "<app>/Contents/MacOS/<name>"`, with `<name>` from `defaults read "<app>/Contents/Info.plist" CFBundleExecutable`) and the modification time of the app's preferences file, and label the date as an estimate. Leave sealed-volume apps such as Safari out of the access-time fallback: Safari's access time read 2024 against an `mdls` date of 2026-05
- Regenerable build output untouched for more than 30 days: `node_modules`, `.venv`, `build`, `.terraform/providers` (list only, do not delete)
- Virtualenvs whose interpreter is gone: `bin/python` is a symlink whose target does not exist (`[ -L <venv>/bin/python ] && [ ! -e <venv>/bin/python ]`), typically after a Homebrew Python was removed. They cannot run, so list them first as the lowest-risk candidates. Their package list can still be read from the `*.dist-info` directory names in `site-packages`
- For a `*.code_sign_clone` directory, report the `du` figure and say it is not reclaimable. In one run `du` showed 66.6GB for 52 clones that would free about 1.3GB
- Developer tool data: installs in `~/.npm/_npx` older than 30 days, versions in `~/.local/share/claude/versions` that are neither the link target nor running, tool folders left behind without their app (`~/.codeium` and the like), runtimes installed several times (JDK, Python)
- Large `.git` directories
- Large disposable files such as Java hprof crash dumps
- Xcode / Developer caches
