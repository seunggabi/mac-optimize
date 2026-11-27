---
name: mac-optimize
description: "Full macOS system check and cleanup. Analyzes memory, CPU, and disk usage, cleans caches/logs/temp files, updates Homebrew packages, stops unneeded background services, and reviews auto-start items. Use when the user mentions 'mac cleanup', 'system optimize', 'mac is slow', 'disk cleanup', 'out of memory', 'clear cache', 'system check', or in Korean '컴퓨터 최적화', '시스템 정리', '맥 느려', '디스크 정리', '메모리 부족', '캐시 정리', '시스템 점검'."
---

# macOS System Optimize

A workflow that analyzes resource usage on macOS, cleans up unneeded files and services, and reports the result with before/after numbers.

## Core principles

- **Safety first**: never use `rm -rf`. Prefer a cache manager's own purge/clean command; otherwise move the target to `/tmp/`.
- **Never touch security software**: do not stop or change corporate security software (Fortinet, AhnLab, Jiran, PNPSecure, Wizvera, and the like).
- **Run in parallel**: independent checks and cleanups go to parallel agents.
- **Before/After reporting**: measure every action before and after and report the numbers.
- **Verify real free space with `df`**: a `du` figure is not always reclaimable space. Anything moved to `/tmp` stays on the same volume, and APFS clones (for example `/private/var/folders/*/*/X/*.code_sign_clone`, which `du` reports as tens of GB) share blocks with the original, so deleting them frees nothing.
- **Report instead of acting when evidence conflicts**: if signs disagree about which copy is in use (for example the Dock and a CLI launcher point at different copies), or the item is settings or data, do not move it. Hand it to the user with the evidence.
- **Re-measure only when run recently**: if the last run was the same day, do not repeat finished cleanups. Re-run Phase 1 and give agents only the areas not yet examined.
- **An old modification time does not mean unused**: a `.venv` or `node_modules` untouched for 30 days can still be executed every day. Even when the user approves deleting build output, leave out any repository that has a live process working in it (a terminal session, an IDE, a coding agent), or name those repositories and ask again.
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

# Top memory processes
ps -eo pid,rss,comm -r | head -20

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

Launch 5 agents **at the same time** in the background:

#### Team 1: cache-cleaner (caches and temp files)

Tasks:
- `pip cache purge`
- `npm cache clean --force`
- Move log files older than 30 days in `~/Library/Logs` to `/tmp/`
- Clean browser caches (Whale, Chrome, and so on) by moving them to `/tmp/`
- Clean the safe parts of JetBrains caches (backup, download, temp)
- In `~/Library/Logs/JetBrains`, move log directories of versions lower than the installed one, and of products that are not installed, to `/tmp/` (read the installed version with `defaults read "<app>/Contents/Info.plist" CFBundleShortVersionString`)
- In the user temp directory (`getconf DARWIN_USER_TEMP_DIR`), move top-level items whose modification and access times are both older than 3 days and that are not open to `/tmp/`
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
- Run `HOMEBREW_NO_AUTO_UPDATE=1 brew upgrade --formula`
- Run `HOMEBREW_NO_AUTO_UPDATE=1 brew autoremove`
- Run `HOMEBREW_NO_AUTO_UPDATE=1 brew cleanup --prune=all` (also clears the download cache)
- Diagnose with `HOMEBREW_NO_AUTO_UPDATE=1 brew doctor`
- Fix what can be fixed safely

Notes:
- Always set `HOMEBREW_NO_AUTO_UPDATE=1` (otherwise brew update tries to clone homebrew-core and gets very slow)
- On a lock conflict, `pkill -f "brew upgrade"` and retry
- For packages that need a major version change, only list them
- Do not upgrade casks; only list them (a running app may restart)
- Leave out services that are running or hold data (MySQL, MariaDB, Kafka, and so on); only list them
- Do not run the `rm -rf` or `sudo` commands that `brew doctor` suggests; report them

#### Team 3: service-optimizer (services)

Tasks:
- Check status with `brew services list`
- Stop development services (Kafka, Redis, Elasticsearch, and so on) that do not need to run all the time
- Stop with `brew services stop <service>` (this also removes the plist, so it will not come back after a reboot)
- Check for abnormal CPU use by FortiClient and similar
- Check services that are not running (Nginx, Docker, and so on)
- Aggregate long-running processes: count, total memory, and uptime per process name (`ps -eo pid,rss,etime,comm`). Dozens of terminal sessions or IDE windows often pile up and hold memory
- Find dead auto-start items: flag LaunchAgents and LaunchDaemons plists whose executable (`plutil -extract ProgramArguments.0 raw <plist>`) no longer exists. Items under `/Library` need admin rights to remove, so only report them

Notes:
- Only list user processes (terminal sessions, IDEs, and so on); end them only when the user asks. Before ending one, check that its child processes have no work in progress, and use `kill -TERM` (never `-KILL`)
- Never change security software (Fortinet, AhnLab, Jiran, PNPSecure, Wizvera, CrossEx, and so on)
- Keep databases the user may want, such as MySQL, by default
- `brew services stop` removes the LaunchAgent plist, so it is safe across reboots

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
- List `node_modules` (list only, do not delete)
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
- Swap usage: `sysctl vm.swapusage`
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
2. **brew upgrade lock conflicts**: if a package failed, kill the stale process and retry
3. **Trash**: if the Trash is large, recommend emptying it (do not empty it automatically)
4. **Items moved to `/tmp`**: `/tmp` is on the same volume, so moving alone does not increase free space. Report the moved size and the "real free space gained" (difference in `df` before and after) separately, and ask the user whether to empty `/tmp$HOME`. When emptying, delete only the paths this run moved and leave files other work has put there
5. **sudo commands**: give the user the DNS flush and `purge` command to run themselves
   `sudo purge && sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder`
6. **macOS updates**: report the list and highlight security updates. Re-measure swap after the cleanup before recommending a restart: swap can drop on its own once memory pressure eases (10GB to under 1GB in one real run), so recommend a restart for swap only if it is still high
7. **Post-check**: confirm nothing the user relies on broke.
   - Development commands still resolve: `for c in python3 node git docker java; do command -v $c >/dev/null || echo "missing: $c"; done`
   - CLI launchers do not point at a removed app: `grep -l "\.app/" /usr/local/bin/* 2>/dev/null` and check that each referenced path exists
   - Removed build output does not sit in a directory where a process is working: compare the removed paths with `lsof -a -p <pid> -d cwd -Fn` of terminal, IDE, and agent processes, and report any overlap
8. **Re-setup list**: for everything removed, tell the user what they need to run before using it again (`terraform init`, `npm install`, recreate the virtualenv, re-pull Docker images, cold-boot the emulator)

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

### 5. Further recommendations (manual)
| Priority | Item | Expected effect |
|----------|------|-----------------|
| ... | ... | ... |

### 6. Before you use it again
| Removed | Run this first |
|---------|----------------|
| ... | ... |

### 7. Not verified
- ...
```

## Key patterns

### Safe deletion
```bash
# Prefer the cache manager's command
pip cache purge
npm cache clean --force
brew cleanup

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
# Always disable auto-update
HOMEBREW_NO_AUTO_UPDATE=1 brew upgrade
HOMEBREW_NO_AUTO_UPDATE=1 brew cleanup
HOMEBREW_NO_AUTO_UPDATE=1 brew outdated
HOMEBREW_NO_AUTO_UPDATE=1 brew doctor
```

### Stopping a service for good
```bash
# brew services stop removes the plist, so it is safe across reboots
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
