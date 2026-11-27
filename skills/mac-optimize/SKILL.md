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
- **Re-measure only when run recently**: if the last run was the same day, do not repeat finished cleanups. Re-run Phase 1 and give agents only the areas not yet examined.
- **An old modification time does not mean unused**: a `.venv` or `node_modules` untouched for 30 days can still be executed every day. Even when the user approves deleting build output, leave out any repository that has a live process working in it (a terminal session, an IDE, a coding agent), or name those repositories and ask again.
- **Record how to rebuild before removing**: "regenerable" is only true if the recipe exists. Before moving a virtualenv, save its Python version (`pyvenv.cfg`) and its package list to a file kept with the list of moved paths. Take the list with `uv pip freeze --python <venv>/bin/python`, or `<venv>/bin/python -m pip freeze` when uv is not installed (a virtualenv created by uv has no pip, so `<venv>/bin/pip` does not exist there). An empty list from a non-empty virtualenv means the command failed, not that nothing is installed. If there is no lock file or requirements file and the list cannot be taken, do not move it. In one real run a virtualenv with no requirements file could not be rebuilt at all, and two others came back at 14MB instead of 219MB because hand-installed packages were not recorded anywhere.
- **Uninstall managed apps through their manager**: an app installed by JetBrains Toolbox, Homebrew, or a similar manager should be removed with that manager. Removing the bundle by hand leaves a dead entry in the manager, and the manager may delete the CLI launcher it generated (for example `/usr/local/bin/pycharm`).

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
- Memory: total, used, compressor size, swap used
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
- Clean the safe parts of JetBrains caches (backup, download, temp)
- In `~/Library/Logs/JetBrains`, move log directories of versions lower than the installed one, and of products that are not installed, to `/tmp/` (read the installed version with `defaults read "<app>/Contents/Info.plist" CFBundleShortVersionString`)
- In the user temp directory (`getconf DARWIN_USER_TEMP_DIR`), list the largest top-level items with their size and modification time. Report only; do not move anything there (sockets, `TemporaryItems`, `com.apple.*`, and `*.code_sign_clone` entries can belong to live daemons or updaters)
- Measure size before and after each task

Notes:
- No `rm -rf`; move to `/tmp/` instead
- Prefer the purge/clean command a cache manager provides
- Keep core IDE caches (indexes and the like)
- Do not run brew commands (they collide with Team 2's upgrade lock; Team 2 runs cleanup after the upgrade)
- Move app caches (`Cache`, `Code Cache`, `GPUCache`, `Service Worker/CacheStorage`) only when the app is not running (check with `ps -eo comm`)
- Before calling something an "old version cache", check whether the app is installed (`/Applications`, `~/Applications`). The cache of an installed version is not old

#### Team 2: brew-updater (Homebrew)

Tasks:
- List with `HOMEBREW_NO_AUTO_UPDATE=1 brew outdated --formula` and `--cask`
- Build the upgrade list: take the names from `brew outdated --formula`, drop every major version change and every formula that appears in `brew services list`. Run `HOMEBREW_NO_AUTO_UPDATE=1 brew upgrade --formula <names>` with that explicit list. If the list is empty, do not run upgrade at all
- Run `HOMEBREW_NO_AUTO_UPDATE=1 brew autoremove --dry-run` first. Run the real `brew autoremove` only if the dry-run list holds nothing the user would call directly (runtimes, databases, CLIs); otherwise report the list
- Run `HOMEBREW_NO_AUTO_UPDATE=1 brew cleanup --prune=all` (also clears the download cache)
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
- Find dead auto-start items: for each LaunchAgents and LaunchDaemons plist, read the executable with `plutil -extract ProgramArguments.0 raw <plist>` and, if that key is missing, `plutil -extract Program raw <plist>`. Flag the plist as dead only when a path was read and that file does not exist. If neither key can be read, say "cannot tell". Skip every plist that matches the security-software keywords. Report only; items under `/Library` need admin rights to remove

Notes:
- Only list user processes (terminal sessions, IDEs, and so on); end them only when the user asks. Before ending one, check that its child processes have no work in progress, and use `kill -TERM` (never `-KILL`)
- Never change security software (Fortinet, AhnLab, Jiran, PNPSecure, Wizvera, CrossEx, and so on)
- Keep databases the user may want, such as MySQL, by default

#### Team 4: disk-analyzer (disk space)

Tasks:
- Top 15 of the home directory with `du -sh ~/*`
- Large directories inside `~/Library`
- Docker status (`docker system df`). Clean only when the user asks, with `docker system prune -a`; ask separately about volumes because they may hold database data
- Size of `/tmp$HOME` (items moved by earlier runs may have piled up)
- Size of the user temp and cache directories under `/private/var/folders` (`getconf DARWIN_USER_TEMP_DIR`, `DARWIN_USER_CACHE_DIR`)
- Size and last-used date per app (`mdls -raw -name kMDItemLastUsedDate <app>`). Flag apps installed twice and large apps unused for more than 180 days (list only, do not delete)
- Regenerable build output untouched for more than 30 days: `node_modules`, `.venv`, `build`, `.terraform/providers` (list only, do not delete)
- Developer tool data: installs in `~/.npm/_npx` older than 30 days, versions in `~/.local/share/claude/versions` that are neither the link target nor running, tool folders left behind without their app (`~/.codeium` and the like), runtimes installed several times (JDK, Python)
- Large `.git` directories
- Large disposable files such as Java hprof crash dumps
- Xcode / Developer caches

#### Team 5: system-health (system health)

Tasks:
- List macOS software updates with `softwareupdate -l` (list only, do not install)
- Verify the volume with `diskutil verifyVolume /`
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
4. **Decisions for the user**: show the lists the agents only reported (running services, Docker, stale build output, unused apps, duplicate runtimes) in one question and ask which to act on. Act only on what they pick, and apply the principles above (skip repositories with a live process; remove managed apps through their manager)
5. **Items moved to `/tmp`**: `/tmp` is on the same volume, so moving alone does not increase free space. Report the moved size and the "real free space gained" (difference in `df` before and after) separately, and ask the user whether to empty them. When they approve, delete only the exact paths this run moved, one by one from the recorded list, and leave everything else under `/tmp$HOME` alone
6. **sudo commands**: give the user the DNS flush and `purge` command to run themselves
   `sudo purge && sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder`
7. **macOS updates**: report the list and highlight security updates. Re-measure swap after the cleanup before recommending a restart: swap can drop on its own once memory pressure eases (10GB to under 1GB in one real run), so recommend a restart for swap only if it is still high
8. **Post-check**: confirm nothing the user relies on broke.
   - Development commands still resolve: `for c in python3 node git docker java; do command -v $c >/dev/null || echo "missing: $c"; done`
   - CLI launchers do not point at a removed app: `find /usr/local/bin -maxdepth 1 -type f -exec grep -l "\.app/" {} +` and check that each referenced path exists (use `find`, because a glob with no match is an error in zsh)
   - Removed build output does not sit in a directory where a process is working: compare the removed paths with `lsof -a -p <pid> -d cwd -Fn` of terminal, IDE, and agent processes, and report any overlap
   - Apps whose caches were removed still start. Ask the user first, because this opens apps on their screen. For each app that is not already running: `open -g -j -a "<app>"`, wait about 25 seconds, check that the process is alive and that no new file for it appeared in `~/Library/Logs/DiagnosticReports`, then quit it with `osascript -e 'tell application "<app>" to quit'`. Skip an app that is already running. If an app does not quit (an open dialog), leave it running and tell the user; never force-quit
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
```

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
- iniline, crossex (certificate tools)
- nprotect (game / security)
- raon, touchen (certificate tools)
