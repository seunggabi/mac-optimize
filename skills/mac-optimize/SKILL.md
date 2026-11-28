---
name: mac-optimize
description: "Full macOS system check and cleanup. Analyzes memory, CPU, and disk usage, cleans caches/logs/temp files, updates Homebrew packages, stops unneeded background services, and reviews auto-start items. Use when the user mentions 'mac cleanup', 'system optimize', 'mac is slow', 'disk cleanup', 'out of memory', 'clear cache', 'system check', or in Korean '컴퓨터 최적화', '시스템 정리', '맥 느려', '디스크 정리', '메모리 부족', '캐시 정리', '시스템 점검'."
---

# macOS System Optimize

A workflow that analyzes resource usage on macOS, cleans up unneeded files and services, and reports the result with before/after numbers.

## Invocation

The text after `/mac-optimize`, or the request that triggered the skill, sets the scope. Decide it before Phase 1 and state it in one line.

- **Nothing**: the full workflow
- **`help` or `usage`** (or the same request in Korean): print the usage summary below and stop. Run nothing that changes the machine
- **A focus word**: Phase 1, then only the matching team, then Phase 3 and Phase 4 for that scope. `disk` is Teams 1 and 4, `memory` and `services` are Team 3, `brew` is Team 2, and `security` or `health` is Team 5. `score` runs Phase 1 and all five teams with every cleanup task removed, then prints only the section 9 score table and what blocks a higher score
- **`report` or `dry-run`**: Phase 1 and all five teams with every cleanup task removed from their prompts (no purge, move, upgrade, cleanup, or autoremove: they list what they would do), then the Phase 4 report. No Phase 3 picks
- **Anything else**: a free-text request. Say how you read it in one line. If it is a question or asks for documentation rather than a cleanup, answer it and start no cleanup

A second run on the same day follows "Re-measure only when run recently", whatever the arguments.

Usage summary:
```
/mac-optimize              diagnose, five parallel teams, your picks, report with a 100-point score
/mac-optimize <focus>      disk | memory | brew | services | security | health | score
/mac-optimize report       diagnose and report only, change nothing (same: dry-run)
/mac-optimize help         this summary
Asks before: stopping services, Docker cleanup, ending agent sessions, removing build
output or apps, admin actions, emptying what it moved to /tmp
Records: ~/.mac-optimize/rebuild/<YYYYMMDD>/ (moved paths, rebuild recipes)
Unattended hourly cleanup: mac-ops (https://github.com/seunggabi/mac-ops)
```

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

## Data-loss rules

Each rule here prevents losing data, settings, work in progress, or a licence. They apply in every phase, go into every team prompt together with Core principles, and win over any step that seems to say otherwise.

Files and caches:
- Move app caches (`Cache`, `Code Cache`, `GPUCache`, `Service Worker/CacheStorage`) only when the app is not running (check with `ps -eo comm`)
- Do not move anything a running process holds open (`lsof +D <dir>` lists the holders; a sqlite database in use is the usual case)
- Some caches are not caches: leave `ms-playwright` (browser binaries that running tools use), `ms-playwright-mcp` (can hold a browser profile), and `JetBrains` (indexes) in `~/Library/Caches`. `~/Library/HTTPStorages` (cookies, so logins) and `~/Library/WebKit` (local storage, IndexedDB) are data, not caches
- Leave `~/Library/Caches/JetBrains/Toolbox/backup` out of a run in which Toolbox updated an IDE: it holds the previous builds for rollback. Keep core IDE caches (indexes and the like)
- For the uv cache use `uv cache prune` (unused entries only). `uv cache clean` removes environments that a running `uvx` tool is executing from
- `find` may reject relative dates: where `find` is `bfs`, `-newermt "3 days ago"` fails with "Invalid timestamp" (`/usr/bin/find` accepted both forms). Pass an absolute time (`-newermt "$(date -v-3d '+%Y-%m-%d %H:%M:%S')"`) and do not send stderr to `/dev/null` on a test that decides what gets moved: a failed test reads as "nothing recent"
- The user temp directory (`getconf DARWIN_USER_TEMP_DIR`) is report-only: sockets, `TemporaryItems`, `com.apple.*`, and `*.code_sign_clone` entries can belong to live daemons or updaters
- A data directory is never moved. Before calling a leftover directory "empty", count its entries (`ls -A <dir> | wc -l`), not its `du` size
- Under `/tmp`, delete only the exact paths this run moved, one by one from the recorded list, and only after the user approves; leave everything else under `/tmp$HOME` alone. Never empty the macOS Trash; recommend it
- Never delete Time Machine snapshots and never run `softwareupdate --install`

Homebrew:
- Never run bare `brew upgrade` (it also upgrades casks, and a running app may restart). Only list outdated casks
- Upgrade only the explicit list of formulae that Team 2 builds; run `brew autoremove` only when its `--dry-run` list holds nothing the user would call directly; skip `brew cleanup` while a running process executes from a keg that is not the linked version
- Never `pkill -f "brew upgrade"`, never kill the PID holding a brew lock, and never run the `rm -rf` or `sudo` commands that `brew doctor` suggests

Processes:
- Only list user processes (terminal sessions, IDEs, and so on); end them only when the user asks. Before ending one, check that its child processes have no work in progress, and use `kill -TERM` (never `-KILL`)
- **Before any signal**: re-check that the PID still exists and is still the same command (`ps -o command= -p <pid>`) right before sending it. Sessions listed minutes earlier can be gone or replaced (two were, in one run)
- Match processes by executable (`ps -eo comm | grep -i <name>`), not by full command line. `pgrep -f zoom` matched an Electron app whose arguments contain `popoutZoom`, and `pgrep -f android` matched `androidEmulator` in the same arguments
- Never end the session running this skill or its ancestors (walk up the parent links from the current shell's PID). End only the session root PIDs the user names: "idle" is the user's call. Leave sessions started within the last day unless the user names them
- Quit Docker Desktop only when `docker ps -q` lists no container, with the recipe in `references/follow-up.md`. Never force-quit an app
- Run `redis-cli save` before stopping Redis. Keep databases the user may want, such as MySQL, by default

Docker, keychain, secrets:
- Never add `--volumes` to a prune and never run `docker volume prune`: volumes may hold database data. `until` filters on the image creation date, so an old image pulled today is pruned too
- Keychain: never run `security dump-keychain -d` (it prints secrets). Delete only the entries the user picks, always with `-a <account>` (without it the first match is deleted). Deletion cannot be undone; say so in the question
- Never print a secret's value: mask it with `sed` before it reaches the output. A backup of an edited startup file holds `<removed>` in place of the value, never a copy that still holds the secret. Check the edited file with `zsh -n` before replacing the original

Other tools and admin actions:
- **mac-ops, never without a pick**: `mac-ops run` (it also purges expired trash), `purge`, `restore`, `install`, `uninstall`. Never delete `~/.mac-ops/.trash` by hand
- Never run a cleaner's test suite or scripts on this machine to check it: a test whose scan roots are not pointed at fixtures runs a real cleanup. Start a scheduled job with `launchctl kickstart` only when the user picks it: it runs a real cleanup and purges expired trash
- Remove a dead auto-start item only after checking that its app is absent and nothing of it is running. Move a broken launcher symlink only when it is both `-L` and not `-e`. Flag an auto-start item as dead only when a program path was read and that file does not exist
- Security software, admin rights, restores, and virtualenvs follow Core principles: never touch security software (keywords at the end of this file); use the password dialog only for what the user picked; compare before restoring; move a virtualenv only with its recorded package list

## Companion tool: mac-ops

[mac-ops](https://github.com/seunggabi/mac-ops) does the unattended side of this job: hourly cleanup from launchd into its own trash with a 72-hour restore.
When `command -v mac-ops` succeeds, or Team 3 finds `com.mac-ops.cleanup`, read `references/mac-ops.md` before you summarize Phase 1 and run the Phase 1 commands it lists.

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
- mac-ops, when installed: the health lines of `mac-ops status` and every setting that `references/mac-ops.md` says to report

### Phase 2: Parallel agent teams

Launch 5 agents **at the same time** in the background.

An agent sees only the prompt you write for it, so a team prompt is built from files, never from the one-line purposes below. Right before launching team N, read its file and build the prompt from, in this order:
1. the whole of `references/team-N-*.md` (table below)
2. Core principles and Data-loss rules from this file
3. the security keyword list (end of this file)
4. the whole of `references/recipes.md`
5. the scope from Invocation (for `report`, `dry-run`, and `score`, remove every cleanup task and say so in the prompt), and a hard limit for the final report

If a team file cannot be read, do not launch that team; say which file is missing.

Give every agent a hard limit for its final report (for example "under 600 words, tables only, at most 15 rows"). An analysis agent with no limit can exceed the output cap and fail with nothing returned.

| Team | File | Purpose |
|---|---|---|
| 1 cache-cleaner | `references/team-1-cache.md` | Caches, old logs, JetBrains leftovers; lists the user temp directory |
| 2 brew-updater | `references/team-2-brew.md` | Safe formula upgrade, autoremove, cleanup, doctor |
| 3 service-optimizer | `references/team-3-services.md` | Services, long-running processes, agent sessions, Docker memory, dead auto-start items, scheduled cleaners (report only) |
| 4 disk-analyzer | `references/team-4-disk.md` | Large folders, Docker disk use, unused apps, stale build output, broken virtualenvs (read-only) |
| 5 system-health | `references/team-5-health.md` | Updates, volumes, backup, firewall, secrets in startup files, uptime, battery (read-only) |

### Phase 3: Collect results and follow up

After the agents finish, read `references/follow-up.md` before step 1. The lines below give the order; that file says how to carry out each step, and a step is done only when it is done as written there.

1. **Java hprof files**: move JVM crash dumps in the home directory to `/tmp/`
2. **brew upgrade lock conflicts**: report the PID holding the lock; do not kill it
3. **Trash**: recommend emptying a large Trash; do not empty it
4. **Decisions for the user**: one question with every list the agents only reported; act only on the picks, each as `references/follow-up.md` describes it
5. **Items moved to `/tmp`**: report the moved size and the `df` gain separately, and empty only with approval
6. **sudo commands**: give the user the DNS flush and `purge` command to run themselves
7. **macOS updates**: report the list, highlight security updates, and re-measure swap before recommending a restart
8. **Post-check**: read `references/post-check.md` and run every check in it
9. **Re-setup list**: for everything removed, what to run before using it again

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

## Security software keywords

A service whose name contains one of these keywords is likely required by corporate security policy. Never change it:
- fortinet, forticlient (VPN / security)
- ahnlab, astx (antivirus)
- jiran, jkok (DLP / security)
- pnpsecure, pcassist (security)
- wizvera, delfino, veraport (certificate tools)
- iniline, crossex, initech, inisafe (certificate tools)
- nprotect (game / security)
- raon, touchen (certificate tools)
