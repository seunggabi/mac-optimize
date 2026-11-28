---
name: mac-optimize
description: "Full macOS system check and cleanup. Analyzes memory, CPU, and disk usage, cleans caches/logs/temp files, updates Homebrew packages, stops unneeded background services, and reviews auto-start items. Use when the user mentions 'mac cleanup', 'system optimize', 'mac is slow', 'disk cleanup', 'out of memory', 'clear cache', 'system check', or in Korean '컴퓨터 최적화', '시스템 정리', '맥 느려', '디스크 정리', '메모리 부족', '캐시 정리', '시스템 점검'."
---

# macOS System Optimize

A workflow that analyzes resource usage on macOS, cleans up unneeded files and services, and reports the result with before/after numbers.

## Core principles

- **Safety first**: never use `rm -rf` on an original location. Prefer a cache manager's own purge/clean command; otherwise move the target to `/tmp/`. A moved item stays recoverable only until `/tmp` is cleaned (a reboot, or macOS purging old files after about 3 days). The one allowed deletion is emptying paths this run already moved under `/tmp`, after the user approves.
- **Never touch security software**: do not stop or change corporate security software (Fortinet, AhnLab, Jiran, PNPSecure, Wizvera, and the like).
- **Run in parallel**: independent checks and cleanups go to parallel agents.
- **Before/After reporting**: measure every action before and after and report the numbers.
- **Verify real free space with `df`**: a `du` figure is not always reclaimable space. Anything moved to `/tmp` stays on the same volume, and APFS clones (for example `/private/var/folders/*/*/X/*.code_sign_clone`, which `du` reports as tens of GB) share blocks with the original, so deleting them frees nothing.
- **Report instead of acting when evidence conflicts**: if signs disagree about which copy is in use (for example the Dock and a CLI launcher point at different copies), or the item is settings or data, do not move it. Hand it to the user with the evidence.
- **Compare before restoring**: before any restore that replaces a live directory, compare the two (key files present, counts, sizes, modification times) and show the user the table. When the live copy is newer or more complete, do not swap. Preserve the trashed copy from expiry instead: clone it into the run's record directory with `cp -Rc <trashed copy> ~/.mac-optimize/rebuild/<YYYYMMDD>/` (an APFS clone, which costs no space on the same volume) and say where it is. Clone it the same way before the cleaner is upgraded, reinstalled, or started again: its trash keeps its own expiry clock, every run of it purges what has expired, and a new version can re-stamp or shorten the clock (mac-ops 1.5.0 began expiring items that 1.4 had kept indefinitely). A cleaner's restore that refuses because the original path exists again is working as intended: compare in place, and never move the live copy aside to make the restore go through. In one run a browser profile in a cleaner's trash looked like the thing to restore, but it had no `Login Data`, `History`, or `Cookies` (it was moved while the browser was running and only part of it arrived), 5 extensions against 18, and the same 386 bookmarks the live profile had already synced back.
- **Re-measure only when run recently**: if the last run was the same day, do not repeat finished cleanups. Re-run Phase 1 and give agents only the areas not yet examined.
- **An old modification time does not mean unused**: a `.venv` or `node_modules` untouched for 30 days can still be executed every day. Even when the user approves deleting build output, leave out any repository that has a live process working in it (a terminal session, an IDE, a coding agent), or name those repositories and ask again.
- **Record how to rebuild before removing**: "regenerable" is only true if the recipe exists. Before moving a virtualenv, save its Python version (`pyvenv.cfg`) and its package list to a file kept with the list of moved paths. Take the list with `uv pip freeze --python <venv>/bin/python`, or `<venv>/bin/python -m pip freeze` when uv is not installed (a virtualenv created by uv has no pip, so `<venv>/bin/pip` does not exist there). An empty list from a non-empty virtualenv means the command failed, not that nothing is installed. If there is no lock file or requirements file and the list cannot be taken, do not move it. In one real run a virtualenv with no requirements file could not be rebuilt at all, and two others came back at 14MB instead of 219MB because hand-installed packages were not recorded anywhere.
- **Uninstall managed apps through their manager**: an app installed by JetBrains Toolbox, Homebrew, or a similar manager should be removed with that manager. Removing the bundle by hand leaves a dead entry in the manager, and the manager may delete the CLI launcher it generated (for example `/usr/local/bin/pycharm`).
- **Opening a manager is an action**: a manager window can start work nobody asked for, and a cancelled auto-update is not a stopped auto-update. Opening JetBrains Toolbox to uninstall one app started major-version update downloads for every installed IDE; they restarted after being cancelled, and within ten minutes all four IDEs were installed at the next major version (2025.3 to 2026.3), which the user had not asked for and which can fall outside a perpetual licence. Before opening a manager, read whether its auto-update is on (if you cannot read the setting, ask the user) and tell the user what opening it will do. If the user does not want updates, have them turn auto-update off first, or do not open it. Record each app's version before (`defaults read "<app>/Contents/Info.plist" CFBundleShortVersionString`), compare afterwards, and report any change and anything you could not stop. JetBrains Toolbox keeps the previous builds under `~/Library/Caches/JetBrains/Toolbox/backup` (12.5GB in that run) for rollback, so do not list that folder as a cache to clean in the same run.
- **Keep the record outside `/tmp`**: write the list of moved paths and the rebuild recipes to `~/.mac-optimize/rebuild/<YYYYMMDD>/` as you go. A scratch directory under `/tmp` is emptied by the same reboot that empties the moved files, which leaves nothing to restore from.
- **Admin rights come from a password dialog, and only for what the user picked**: `sudo` cannot read a password without a terminal. For an admin action the user chose in Phase 3, write the commands to a script, show the user the script, and then run it with `osascript -e 'do shell script "<script>" with prompt "<what and why>" with administrator privileges'`. Never use the dialog for anything the user did not pick. Phase 2 agents never use the dialog, and the commands in the "sudo commands" step stay with the user.
- **Two macOS path traps**: `/tmp` is a symlink to `/private/tmp`, so `find /tmp ...` reports the link and scans nothing; search `/private/tmp` (or `/tmp/`). APFS is case-insensitive by default, so `Foo` and `foo` name the same file: lowercase both sides (`${name:l}` in zsh) before deciding that a name is protected, installed, or different.

## Companion tool: mac-ops

[mac-ops](https://github.com/seunggabi/mac-ops) does the unattended side of this job: a Homebrew CLI whose LaunchAgent cleans caches, temp files, logs, and leftovers of removed apps every hour, into its own trash with a 72-hour restore. This skill is the interactive pass: diagnosis, decisions, and anything that needs the user.

- **When it is installed** (`command -v mac-ops`, or Team 3 finds `com.mac-ops.cleanup`): read `mac-ops version` first; `runs`, `restore --run`, and the health lines of `status` exist from 1.5.0. In Phase 1 run `mac-ops status` (read-only; an `[ALERT]` first line means the last scheduled run failed or is overdue, and a BROKEN or STALE agent line comes with the command that fixes it) and `mac-ops config` (every setting with its source). Report every setting that moves settings or ends processes: `DataAction` set to `trash`, an `Action` set to `term` or `prune`, or the `brew` module enabled
- **Undoing one of its runs**: `mac-ops runs` lists each run with how many settings and app-data items it moved. After "Compare before restoring", and only for what the user picks: `mac-ops restore --run <run-id> --dry-run`, then the same without `--dry-run`
- **Never without a pick**: `mac-ops run` (it also purges expired trash), `purge`, `restore`, `install`, `uninstall`. Never delete `~/.mac-ops/.trash` by hand
- **When to recommend installing it** (Phase 4, further recommendations): the caches and logs this run cleaned will grow back, the user wants that upkeep without asking, and no other scheduled cleaner is active (two cleaners make every finding ambiguous). Give the steps: `brew install seunggabi/tap/mac-ops`, `mac-ops run --dry-run` and read the list, then `mac-ops install`. By default it moves only caches, temp files, logs, and the caches and saved window state of removed apps; their settings and data folders are reported, not moved

## Workflow

### Phase 1: Diagnose system state (do this yourself)

Get a quick picture of the system. Run these **in parallel**:

```bash
# CPU / memory summary
top -l 1 -n 0 | head -12

# Disk usage
df -h / /System/Volumes/Data

# Total RAM and VM stats
echo "Total RAM: $(( $(sysctl -n hw.memsize) / 1073741824 )) GB"
vm_stat | head -10
sysctl vm.swapusage
memory_pressure | tail -1
vm_stat | grep -E "Anonymous|File-backed|occupied by compressor"   # app memory / file cache / compressed

# Top memory processes (-m sorts by memory; -r would sort by CPU)
ps -eo pid,rss,comm -m | head -20

# Top CPU processes (ps %CPU is a lifetime average; use top for the current load)
top -l 2 -n 10 -o cpu -stats pid,cpu,mem,command | tail -11

# Cache directory sizes
du -sh ~/Library/Caches/*/ 2>/dev/null | sort -hr | head -15

# Login items
osascript -e 'tell application "System Events" to get the name of every login item'

# LaunchAgents
ls ~/Library/LaunchAgents/ 2>/dev/null
ls /Library/LaunchAgents/ 2>/dev/null
ls /Library/LaunchDaemons/ 2>/dev/null

# Running non-Apple services
launchctl list | grep -v "com.apple" | grep -v "^\-" | head -30
```

Summarize the diagnosis for the user as tables:
- CPU: idle %, load average
- Memory: total, app memory (anonymous pages), file cache (file-backed pages), compressor size, swap used, free percentage from `memory_pressure`. "Used" in `top` includes the file cache, which macOS gives back on demand: 24GB "used" on a 32GB machine was 7.8GB of cache with pressure normal. Call memory a problem only when the free percentage is low or swap is growing
- Disk: total, used, free
- Top 5 resource consumers
- Largest caches

### Phase 2: Parallel agent teams

Launch 5 agents **at the same time** in the background.

An agent sees only the prompt you write for it. Paste the Core principles and the security-software keyword list (end of this file) into every agent prompt, together with its team section.

Give every agent a hard limit for its final report (for example "under 600 words, tables only, at most 15 rows"). An analysis agent with no limit can exceed the output cap and fail with nothing returned.

#### Team 1: cache-cleaner (caches and temp files)

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
- No `rm -rf`; move to `/tmp/` instead
- Prefer the purge/clean command a cache manager provides
- Keep core IDE caches (indexes and the like)
- Do not run brew commands (they collide with Team 2's upgrade lock; Team 2 runs cleanup after the upgrade)
- Move app caches (`Cache`, `Code Cache`, `GPUCache`, `Service Worker/CacheStorage`) only when the app is not running (check with `ps -eo comm`)
- Do not move anything a running process holds open (`lsof +D <dir>` lists the holders; a sqlite database in use is the usual case)
- Some caches are not caches: leave `ms-playwright` (browser binaries that running tools use), `ms-playwright-mcp` (can hold a browser profile), and `JetBrains` (indexes) in `~/Library/Caches`. `~/Library/HTTPStorages` (cookies, so logins) and `~/Library/WebKit` (local storage, IndexedDB) are data, not caches
- Before calling something an "old version cache", check whether the app is installed (`/Applications`, `~/Applications`). The cache of an installed version is not old
- For the uv cache use `uv cache prune` (unused entries only). `uv cache clean` removes environments that a running `uvx` tool is executing from
- `find` may reject relative dates: where `find` is `bfs`, `-newermt "3 days ago"` fails with "Invalid timestamp" (`/usr/bin/find` accepted both forms). Pass an absolute time (`-newermt "$(date -v-3d '+%Y-%m-%d %H:%M:%S')"`) and do not send stderr to `/dev/null` on a test that decides what gets moved: a failed test reads as "nothing recent"

#### Team 2: brew-updater (Homebrew)

Tasks:
- List with `HOMEBREW_NO_AUTO_UPDATE=1 brew outdated --formula` and `--cask`
- Build the upgrade list: take the names from `brew outdated --formula`, drop every major version change, every formula that appears in `brew services list`, every language runtime a running process executes from (`ps -eo comm | grep Cellar`), and every formula such a runtime links against (`brew deps <runtime>`). Run `HOMEBREW_NO_AUTO_UPDATE=1 brew upgrade --formula --dry-run <names>` and drop anything that would build from source or pull a dropped formula along. Then run `HOMEBREW_NO_AUTO_UPDATE=1 brew upgrade --formula <names>` with that explicit list. If the list is empty, do not run upgrade at all
- If `brew doctor` says the macOS version is no longer supported, expect almost nothing to upgrade: bottles are not built for it, so nearly every formula is a source build. Report that as the cause instead of forcing the builds (in one run 1 of 73 outdated formulae was upgradable)
- Run `HOMEBREW_NO_AUTO_UPDATE=1 brew autoremove --dry-run` first. Run the real `brew autoremove` only if the dry-run list holds nothing the user would call directly (runtimes, databases, CLIs); otherwise report the list
- Run `HOMEBREW_NO_AUTO_UPDATE=1 brew cleanup --prune=all` (also clears the download cache). Skip it and report why when a running process executes from a keg that is not the linked version of its formula (a `ps -eo comm` path under `Cellar/<formula>/<version>/` whose version differs from the target of `$(brew --prefix)/opt/<formula>`): cleanup deletes that keg under the running process
- Diagnose with `HOMEBREW_NO_AUTO_UPDATE=1 brew doctor` and report the findings. Fix none of them

Notes:
- Always set `HOMEBREW_NO_AUTO_UPDATE=1` (otherwise brew update tries to clone homebrew-core and gets very slow)
- On a lock conflict, wait 30 seconds and retry once, then report. Do not `pkill -f "brew upgrade"`: it matches any command line containing that text, including your own shell and a brew the user is running in another terminal
- Never run bare `brew upgrade` (it also upgrades casks, and a running app may restart). Only list outdated casks
- Do not run the `rm -rf` or `sudo` commands that `brew doctor` suggests; report them

#### Team 3: service-optimizer (services)

Tasks:
- Check status with `brew services list`
- List running development services (Kafka, Redis, Elasticsearch, and so on) with their memory use. Do not stop any of them. The user decides in Phase 3; a service is stopped only after they name it, with `brew services stop <service>`
- Check for abnormal CPU use by FortiClient and similar (report only)
- Check services that are not running (Nginx, Docker, and so on)
- Aggregate long-running processes: count, total memory, and uptime per process name (`ps -eo pid,rss,etime,comm`). Dozens of terminal sessions or IDE windows often pile up and hold memory
- Group coding-agent sessions by their root process (`ps -eo pid,ppid,rss,etime,command`, then walk the parent links). Each session starts its own set of MCP servers, so `npm` and `node` rows that look like orphans usually belong to a live session: in one run 56 of them belonged to 7 sessions, about 1GB per session. Report per session: root PID, working directory (`lsof -a -p <pid> -d cwd -Fn`), uptime, child count, total memory. List every session on its own row, not as a total. Mark the session running this skill and its ancestors as "this session" (walk up the parent links from the current shell's PID with `ps -o ppid= -p <pid>`): it is never ended. "Idle" is the user's call: a session that is waiting for user input looks the same as an abandoned one, so the user decides from the working directory and uptime. Flag as orphans only processes whose parent is PID 1, and not all of those: launchd jobs (PID listed in `launchctl list`), helpers inside a `.app`, processes with children, and daemons started on purpose (ssh-agent, gpg-agent, tmux, databases, local model servers, tunnels) have parent PID 1 by design
- Docker Desktop: report the container count (`docker ps -q | wc -l`) and the memory of its processes including the VM (`com.apple.Virtualization.VirtualMachine`). Prove the VM is Docker's with `lsof -p <pid> | grep Docker.raw`, because other apps run VMs under the same process name
- Find dead auto-start items: for each LaunchAgents and LaunchDaemons plist, read the executable with `plutil -extract ProgramArguments.0 raw <plist>` and, if that key is missing, `plutil -extract Program raw <plist>`. Flag the plist as dead only when a path was read and that file does not exist. If neither key can be read, say "cannot tell". Skip every plist that matches the security-software keywords. Report only; items under `/Library` need admin rights to remove
- List scheduled cleaners, because another cleanup tool can be the cause of a finding: LaunchAgents, LaunchDaemons, and cron entries whose label or program suggests cleaning (clean, cleanup, purge, optimize, maid, sweep, tidy, or a known tool such as mac-ops, CleanMyMac, Onyx, mac-cleanup). Read plists with `plutil -p` (they can be binary): `find ~/Library/LaunchAgents /Library/LaunchAgents /Library/LaunchDaemons -maxdepth 1 -name '*.plist' | while read -r p; do plutil -p "$p" | grep -q -i -E 'clean|purge|optimi[sz]e|maid|sweep|tidy|mac-ops|onyx' && echo "$p"; done`, and `crontab -l | grep -i -E` with the same pattern. For each one found, read its log and its trash listing for entries under `~/Library/Application Support`, `Preferences`, and `Containers` in the last 90 days, and report what it moved and when. Report only, and skip every plist that matches the security-software keywords. In one run a second cleanup tool ran hourly from a LaunchAgent: it had moved `~/Library/Application Support/JetBrains` to its own trash as "orphan app data" four times in five months and purged it after 72 hours, and it did the same to a browser profile and to iTerm2 and Garmin data
- If that cleaner is mac-ops, read `mac-ops version` first. On 1.5.0 and later, start with its read-only commands, `mac-ops status`, `mac-ops runs`, and `mac-ops list-trash`, then read its log for anything older (run manifests cover the last 200 runs, the trash only 72 hours). On an older version, report the version and read only its log and trash listing
- Judge what a cleaner will do next from its preview, read by name: on mac-ops 1.5.0 or later run `mac-ops run --dry-run` (moves and signals nothing; if it says another run holds the lock, it previewed nothing) and read every path it would move. A selection rule that silently matches everything looks normal as a count and wrong only in the names: that is how the cleaner above took live settings folders. Never run a cleaner's test suite or scripts on this machine to check it: a test whose scan roots are not pointed at fixtures runs a real cleanup. Read the code instead
- Before calling a leftover directory "empty", count its entries (`ls -A <dir> | wc -l`), not its `du` size. A `/opt/homebrew/var/mongodb` reported as empty held 24 database files. A data directory is never moved

Notes:
- Only list user processes (terminal sessions, IDEs, and so on); end them only when the user asks. Before ending one, check that its child processes have no work in progress, and use `kill -TERM` (never `-KILL`)
- Never change security software (Fortinet, AhnLab, Jiran, PNPSecure, Wizvera, CrossEx, and so on)
- Keep databases the user may want, such as MySQL, by default
- Match processes by executable (`ps -eo comm | grep -i <name>`), not by full command line. `pgrep -f zoom` matched an Electron app whose arguments contain `popoutZoom`, and `pgrep -f android` matched `androidEmulator` in the same arguments

#### Team 4: disk-analyzer (disk space)

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

#### Team 5: system-health (system health)

Tasks:
- List macOS software updates with `softwareupdate -l` (list only, do not install)
- Verify both volumes: `diskutil verifyVolume /` checks only the sealed System volume, so also run `diskutil verifyVolume /System/Volumes/Data` for the user's data. Run them last (minutes, brief I/O stalls). A live check can print "found to be corrupt", then "appears to be OK" with exit code 0 and "A repair action has been taken": quote the lines as they are, and recommend First Aid from Recovery before an OS update instead of calling the volume healthy or broken
- Backup and firewall: `tmutil destinationinfo` ("No destinations configured" means no backup; say so before recommending an OS update) and `/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate`
- Automatic-update settings and stealth mode: `for k in AutomaticCheckEnabled AutomaticDownload CriticalUpdateInstall ConfigDataInstall; do defaults read /Library/Preferences/com.apple.SoftwareUpdate "$k"; done` (1 means on; report only) and `/usr/libexec/ApplicationFirewall/socketfilterfw --getstealthmode`. Read the values before offering to turn anything on: in one run all four were already on
- Plaintext secrets in shell startup files: `grep -n -i -E "(token|secret|api_?key|password)[A-Z_]*=" ~/.zshrc ~/.zshenv ~/.zprofile 2>/dev/null | sed -E 's/=.*/=<masked>/'`. The `sed` masks the value before it is printed; never run the `grep` without it. Report the file, line number, and variable name only. Never print the value
- Uptime: `uptime` (swap is not cleared when the machine runs for a long time without a reboot)
- `.DS_Store` files (first level of the home directory only)
- Trash size: `du -sh ~/.Trash/ 2>/dev/null`
- Time Machine local snapshots: `tmutil listlocalsnapshots /`
- System Integrity Protection: `csrutil status`
- Open file limits: `sysctl kern.maxfiles kern.maxfilesperproc`
- Disk and battery health: `diskutil info / | grep SMART`, `system_profiler SPPowerDataType | grep -E "Cycle Count|Condition|Maximum Capacity"`
- Protection status: `fdesetup status`, `spctl --status`

Notes:
- Do not run `softwareupdate --install`; only list
- Do not delete Time Machine snapshots; only report their size
- Do not run `sudo` (an agent cannot type a password). Give the DNS flush and `purge` commands to the user in the Phase 4 recommendations

### Phase 3: Collect results and follow up

After the agents finish:

1. **Java hprof files**: find them with `find ~ -maxdepth 1 -name 'java_error_in_*.hprof'` and move them to `/tmp/` (JVM crash dumps, safe to delete)
2. **brew upgrade lock conflicts**: if a package failed on a lock, report it and the PID holding the lock (`pgrep -x brew`). Do not kill it
3. **Trash**: if the Trash is large, recommend emptying it (do not empty it automatically)
4. **Decisions for the user**: show the lists the agents only reported (running services, Docker, idle coding-agent sessions, stale build output, unused apps, duplicate runtimes, dead auto-start items, scheduled cleaners, plaintext secrets) in one question and ask which to act on. No team collects keychain leftovers: Phase 3 collects that list itself, and only when the user asks for it. Act only on what they pick, and apply the principles above (skip repositories with a live process; remove managed apps through their manager). How to carry out the common picks:
   - **Docker images**: `docker image prune -a --filter "until=720h" -f` removes unused images created more than 30 days ago. `until` filters on the creation date, not the pull date, so an old image pulled today is removed too; such images need a re-pull or rebuild. Never add `--volumes` and never run `docker volume prune`. Build cache follows the same age rule: `docker builder prune --filter "until=720h" -f`
   - **Docker Desktop**: quit it only when no container is running, then verify (see "Quitting Docker Desktop" below)
   - **Idle agent sessions**: "idle" is the user's call. Show each session on its own row with its working directory and uptime, and end only the root PIDs the user names. Never the session running this skill or its ancestors (walk up the parent links from the current shell's PID). A session waiting for user input also has no child other than its MCP servers, so that check does not prove a session is abandoned. For a named session, confirm it has no child other than its MCP servers, send `kill -TERM` to the session root, and check that the children exited. Leave sessions started within the last day unless the user names them
   - **Before any signal**: re-check that the PID still exists and is still the same command (`ps -o command= -p <pid>`) right before sending it. Sessions listed minutes earlier can be gone or replaced (two were, in one run)
   - **A service**: for Redis run `redis-cli save` first, then `brew services stop redis`
   - **Dead auto-start items under `/Library`**: check that the app is absent and nothing of it is running, `launchctl bootout system/<label>` for a loaded item, then move the plist and its helper in `/Library/PrivilegedHelperTools` to `/tmp/Library/...` through the password dialog. A dead login item is removed with `osascript -e 'tell application "System Events" to delete login item "<name>"'`. `/usr/local/bin` is usually owned by root, so broken launcher symlinks are moved to `/tmp/usr/local/bin/` through the password dialog too, and only links that are both `-L` and not `-e` (`[ -L <link> ] && [ ! -e <link> ]`)
   - **A scheduled cleaner**: stopping it is the user's pick. Use the tool's own command (for example `<tool> uninstall`), not `launchctl` by hand. To check a scheduled job after a fix or an upgrade, and only when the user picks it (it runs a real cleanup and purges expired trash), start it through launchd: `launchctl kickstart gui/$(id -u)/<label>`, then read its log and exit status (`launchctl print gui/$(id -u)/<label>`). A run from the terminal proves little: launchd gives the job a different PATH and its own privacy permissions (Full Disk Access)
   - **A tap of the user's own that brew reports as untrusted**: `brew trust <user>/<tap>`, only for a tap the user owns and picks. Never trust a third-party tap. In one run every `brew outdated` call printed "Refusing to load ... from untrusted tap" until this was done
   - **Keychain leftovers**: list names and dates only with `security dump-keychain | grep -E '"(svce|srvr|acct|mdat)"|^keychain:|^class:'` (never `-d`, which prints secrets; without the filter the dump prints thousands of attribute lines). Offer entries of apps that are no longer installed; put everything else under "confirm". Delete only what the user picks, with `security delete-generic-password -s <service> -a <account>` or `security delete-internet-password -s <server> -a <account>` (without `-a` the first match is deleted). Deletion cannot be undone; say so in the question. Items in the System keychain need admin rights. Saved Wi-Fi passwords are removed from the known-networks list in System Settings, not from the keychain
   - **A plaintext secret**: ask whether it is still used. If it is, only report it. If not, delete the line and tell the user to revoke the token at the provider. Check the edited file with `zsh -n` before replacing the original, and keep a backup of the original with the value replaced by `<removed>`, never a copy that still holds the secret
5. **Items moved to `/tmp`**: `/tmp` is on the same volume, so moving alone does not increase free space. Report the moved size and the "real free space gained" (difference in `df` before and after) separately, and ask the user whether to empty them. When they approve, delete only the exact paths this run moved, one by one from the recorded list, and leave everything else under `/tmp$HOME` alone
6. **sudo commands**: give the user the DNS flush and `purge` command to run themselves
   `sudo purge && sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder`
7. **macOS updates**: report the list and highlight security updates. Re-measure swap after the cleanup before recommending a restart: swap can drop on its own once memory pressure eases (10GB to under 1GB in one real run), so recommend a restart for swap only if it is still high. If the user starts the OS update themselves, do not start `softwareupdate` as well. While an OS update is downloading, `df` free space drops by the size of the download (about 5GB in one run); say so when comparing before and after
8. **Post-check**: confirm nothing the user relies on broke.
   - Development commands still resolve: `for c in python3 node git docker java; do command -v $c >/dev/null || echo "missing: $c"; done`
   - CLI launchers do not point at a removed app: `find /usr/local/bin -maxdepth 1 -type f -exec grep -l "\.app/" {} +` and check that each referenced path exists (use `find`, because a glob with no match is an error in zsh). Launchers are often symlinks, which `-type f` skips, so also list broken ones: `find /usr/local/bin -maxdepth 1 -type l ! -exec test -e {} \; -print`
   - Removed build output does not sit in a directory where a process is working: compare the removed paths with `lsof -a -p <pid> -d cwd -Fn` of terminal, IDE, and agent processes, and report any overlap
   - Apps whose caches were removed still start. Ask the user first, because this opens apps on their screen. For each app that is not already running: `open -g -j -a "<app>"`, wait about 25 seconds, check that the process is alive and that no new file for it appeared in `~/Library/Logs/DiagnosticReports`, then quit it with `osascript -e 'tell application "<app>" to quit'`. Skip an app that is already running. If an app does not quit (an open dialog), leave it running and tell the user; never force-quit
   - Auto-start items survived the upgrade: after any `brew upgrade`, re-run Team 3's dead auto-start check for plists whose program path contains `/Cellar/`. A versioned path dies with the old version: after a tool went from 1.4.0 to 1.4.2 its LaunchAgent still pointed at `/opt/homebrew/Cellar/<tool>/1.4.0/...`, which no longer existed, and the job failed silently every five minutes. Re-register with the tool's own setup command, not by editing the plist
   - User data or settings are missing (a settings folder is gone, an app starts as if new): read the logs and trash listings of the scheduled cleaners Team 3 listed before writing "cause unknown". If a cleaner's trash still holds the copy, apply "Compare before restoring"
   - The user reports that something broke during the run: first list what the run changed that could affect it and check each against the live state. When nothing the run changed is involved, say so plainly and give the user one command that shows the actual behaviour, instead of guessing a cause. For a key problem: run `cat -v` in a shell, press the key, and read the bytes. In one run a terminal shortcut was reported broken while the terminal's Option-key setting from an earlier fix was still in place, the multiplexer and shell bindings were untouched, and the terminal app had not restarted
9. **Re-setup list**: for everything removed, tell the user what they need to run before using it again (`terraform init`, `npm install`, recreate the virtualenv, re-pull Docker images, cold-boot the emulator)

### Phase 4: Final report

When everything is done, write the final report in this format:

```
## Final system optimization report

### 1. Disk cleanup
| Action | Size | Counted in real free space |
|--------|------|----------------------------|
| ... | ... | yes / waiting in /tmp |
| **Real free space gained (by df)** | **XXX GB** | |

### 2. Homebrew packages
- Upgraded: N
- Not done (manual): ...
- brew doctor: ...

### 3. Services
| Service | Action |
|---------|--------|
| ... | ... |

### 4. System state (Before → After)
| Item | Before | After |
|------|--------|-------|
| CPU idle | X% | Y% |
| Load Avg | X | Y |
| Free memory / swap used | X GB | Y GB |
| Disk free | X GB | Y GB |

### 5. System health
| Check | Result |
|-------|--------|
| macOS updates / volume / SMART / battery / uptime | ... |

### 6. Further recommendations (manual)
| Priority | Item | Expected effect |
|----------|------|-----------------|
| ... | ... | ... |

### 7. Before you use it again
| Removed | Run this first |
|---------|----------------|
| ... | ... |

### 8. Not verified
- ...

### 9. Score (the skill's own estimate)
| Area | Weight | Score | The one fact that set it |
|------|--------|-------|--------------------------|
| Disk | 20 | ... | ... |
| Memory | 20 | ... | ... |
| Security | 20 | ... | ... |
| Backup and recovery | 15 | ... | ... |
| Auto-start and leftovers | 15 | ... | ... |
| Development environment | 10 | ... | ... |
| **Total** | **100** | **...** | |

What blocks a higher score, and who has to do it:
- ...
```

The weights in section 9 are fixed; do not change them between runs. Under the table, name each thing that blocks a higher score and who has to do it (for example: no backup destination needs an external disk from the user; an OS update needs a restart). Some points cannot be earned by the skill at all: in one run three of the missing points were of that kind.

## Key patterns

### Safe deletion
```bash
# Prefer the cache manager's command (brew cleanup belongs to Team 2 only)
pip cache purge
npm cache clean --force

# No manager: move to /tmp
mkdir -p "/tmp$HOME/Library/Caches"
mv ~/Library/Caches/SomeApp "/tmp$HOME/Library/Caches/SomeApp"

# Avoid collisions (add a timestamp when the target exists)
mv ~/some/path "/tmp$HOME/some/path_$(date +%Y%m%d_%H%M%S)"
```

### Looping over a list in zsh
```bash
# zsh does not word-split $VAR. Use an array for lists
T=(101 102 103); for p in $T; do kill -TERM $p; done   # correct
T="101 102 103"; for p in $T; do kill -TERM $p; done   # loops once and fails
```

### Checking whether something is running
```bash
# Inside a loop, ps | grep "$name" matches the grep itself, so everything looks "running".
# Build the running list once first, then compare against it
RUN=$(ps -eo command | grep -v grep | grep -o "_npx/[a-f0-9]\{8,\}" | sort -u)
case "$RUN" in *"$h"*) echo "skip $h";; esac
```

### Checking that something exists in zsh
```bash
# A glob with no match aborts the whole command, so `ls /Applications/Foo* /Applications/Bar.app`
# prints an error and checks nothing. Test each path on its own
for a in "zoom.us.app" "Charles.app"; do
  for d in /Applications "$HOME/Applications"; do [ -e "$d/$a" ] && echo "installed: $d/$a"; done
done
```

### Quitting Docker Desktop
```bash
# Stop if containers cannot be listed. If the daemon is already unreachable the container
# count is unknown: ask the user instead of quitting or killing
quit_ok=0
docker ps -q >/dev/null || { echo "cannot list containers: stop"; return; }

# Quit only when the list is empty. The listing is checked again on this line because
# outside a function bash prints an error at `return` and continues.
# `quit_ok` lives only within one shell call: run the whole block in one call. If it is
# empty at the kill line, ask the user instead of killing by hand
c=$(docker ps -q) && [ -z "$c" ] && quit_ok=1 && osascript -e 'tell application "Docker" to quit'

# Verify after 30 seconds. The quit request can be ignored: in one run the engine restarted twice
# instead, and the VM grew from 1.6GB to 8.5GB
ps -eo pid,rss,comm | grep -E "Docker.app|com.docker|Virtualization.VirtualMachine"

# Still there: end the main process with TERM, never KILL, and check again. The kill runs only
# when the list was empty above, a PID was found, its parent is PID 1, and no container is listed now.
# A `docker serve` or `com.docker.dev-envs` helper can be left behind with parent PID 1
p=$(pgrep -x Docker | head -1); [ "$quit_ok" = 1 ] && [ -n "$p" ] && [ "$(ps -o ppid= -p "$p" | tr -d ' ')" = 1 ] && [ -z "$(docker ps -q 2>/dev/null)" ] && kill -TERM "$p"
```

### Admin action through the password dialog (only for what the user picked)
```bash
# Put the commands in a script, show it to the user, then let macOS ask for the password
mkdir -p "$HOME/.mac-optimize/rebuild/$(date +%Y%m%d)"
cat > "$HOME/.mac-optimize/rebuild/$(date +%Y%m%d)/admin.sh" <<'EOF'
#!/bin/sh
mkdir -p /tmp/Library/LaunchDaemons
launchctl bootout system/com.example.dead 2>&1 || true
mv /Library/LaunchDaemons/com.example.dead.plist /tmp/Library/LaunchDaemons/
EOF
chmod 755 "$HOME/.mac-optimize/rebuild/$(date +%Y%m%d)/admin.sh"
osascript -e "do shell script \"$HOME/.mac-optimize/rebuild/$(date +%Y%m%d)/admin.sh\" with prompt \"Move the dead auto-start items to /tmp\" with administrator privileges"
```

### brew commands
```bash
# Always disable auto-update, and always name the formulae to upgrade
HOMEBREW_NO_AUTO_UPDATE=1 brew outdated --formula
HOMEBREW_NO_AUTO_UPDATE=1 brew upgrade --formula <names>
HOMEBREW_NO_AUTO_UPDATE=1 brew cleanup --prune=all
HOMEBREW_NO_AUTO_UPDATE=1 brew doctor
```

### Stopping a service for good (only after the user names it)
```bash
# brew services stop removes the plist, so the service does not come back after a reboot
brew services stop kafka

# Verify
ls ~/Library/LaunchAgents/homebrew.mxcl.kafka.plist  # should not exist
```

### Identifying security software
A service whose name contains one of these keywords is likely required by corporate security policy. Never change it:
- fortinet, forticlient (VPN / security)
- ahnlab, astx (antivirus)
- jiran, jkok (DLP / security)
- pnpsecure, pcassist (security)
- wizvera, delfino, veraport (certificate tools)
- iniline, crossex, initech, inisafe (certificate tools)
- nprotect (game / security)
- raon, touchen (certificate tools)
