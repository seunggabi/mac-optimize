# mac-optimize

A Claude Code skill that checks and cleans up a macOS system with you: it diagnoses memory, CPU, disk, services and auto-start items, cleans what is safe to clean, asks you about everything else, and ends with a before/after report and a 100-point score.

## Install

```bash
npx -y skills add seunggabi/mac-optimize -g -a claude-code -y   # for your user, all projects
npx skills add seunggabi/mac-optimize                            # into the current project only
```

### Manual install

```bash
git clone https://github.com/seunggabi/mac-optimize.git
mkdir -p ~/.claude/skills
ln -s "$(pwd)/mac-optimize/skills/mac-optimize" ~/.claude/skills/mac-optimize
```

The link points at the whole `skills/mac-optimize` folder, so `references/` comes with it. When copying by hand, copy the folder, not `SKILL.md` alone.

### Layout

```
skills/mac-optimize/
|-- SKILL.md                    # always loaded (about 24 KB): invocation, principles, data-loss rules, phases, report format
`-- references/                 # loaded only at the step that needs it
    |-- team-1-cache.md         # prompt body of each Phase 2 team
    |-- team-2-brew.md
    |-- team-3-services.md
    |-- team-4-disk.md
    |-- team-5-health.md
    |-- recipes.md              # shell patterns pasted into every team prompt
    |-- follow-up.md            # Phase 3: how to carry out each step and each pick
    |-- post-check.md           # Phase 3 step 8
    `-- mac-ops.md              # only when mac-ops is installed
```

Every rule that prevents data loss is in `SKILL.md` itself, in the Core principles and the Data-loss rules block. The reference files repeat some of them at the step where they apply, and otherwise hold only the details of actions that cannot be reached without reading them (a team's tasks, the recipe for a pick).

## Command usage

Type `/mac-optimize`, or just describe the problem: phrases such as "mac is slow", "disk cleanup", "out of memory", "clear cache" or "system check" start it too, and so do the Korean ones in [SKILL.md](skills/mac-optimize/SKILL.md) (for example 맥 느려, 디스크 정리, 시스템 점검).

| Invocation | What it does | Example |
|---|---|---|
| `/mac-optimize` | Full workflow: diagnose, five parallel teams, your picks, final report | `/mac-optimize` |
| `/mac-optimize <focus>` | Diagnose, then only the matching team and its follow-up: `disk`, `memory`, `brew`, `services`, `security` / `health` | `/mac-optimize disk` |
| `/mac-optimize score` | Re-measure and print only the score table and what blocks a higher score; cleans nothing | `/mac-optimize score` |
| `/mac-optimize report` | Diagnose and report what it would do; changes nothing (same: `dry-run`) | `/mac-optimize report` |
| `/mac-optimize help` | Print the usage summary and stop (same: `usage`, or the same request in Korean) | `/mac-optimize help` |
| `/mac-optimize <anything else>` | Read as a free-text request; a question gets an answer, not a cleanup | `/mac-optimize why is swap so high?` |

## What it does

| Phase | What happens |
|---|---|
| 1. Diagnose | CPU, memory, swap, disk, caches and auto-start items, as tables |
| 2. Five teams in parallel | cache-cleaner, brew-updater, service-optimizer, disk-analyzer (read-only), system-health (read-only) |
| 3. Your picks | One question with everything that needs your decision, then only what you picked |
| 4. Report | Before/after numbers, what to run before using removed things again, what was not verified, and the score |

## Usage examples

**Start a full check** when the Mac feels slow or the disk is nearly full:
```
/mac-optimize
```

**Answer the follow-up picks.** After the teams finish, it lists what it only reported (running services, Docker, idle coding-agent sessions, old build output, unused apps, duplicate runtimes, dead auto-start items, scheduled cleaners, plaintext secrets) in one question. Name what to act on and leave the rest:
```
Stop redis. Prune Docker images older than 30 days. Remove the node_modules of the two
archived projects. Leave the agent sessions and the apps alone.
```

**Look without changing anything**, for example on a work machine:
```
/mac-optimize report
```

**Find your records.** Every run keeps the list of moved paths and the rebuild recipes (Python version and package list of each removed virtualenv) in `~/.mac-optimize/rebuild/<YYYYMMDD>/`, outside `/tmp`, so they survive a reboot:
```bash
ls ~/.mac-optimize/rebuild/
```

**Run it again later.** The score weights are fixed, so the section 9 tables of two reports can be compared line by line. A second run on the same day does not repeat finished cleanups; it re-measures and looks only at what was not examined yet:
```
/mac-optimize score
```

## Sample report (example)

The last part of a final report. The numbers are invented; the structure is what the skill prints.

```
### 9. Score (the skill's own estimate)
| Area                     | Weight | Score | The one fact that set it                     |
|--------------------------|--------|-------|----------------------------------------------|
| Disk                     | 20     | 17    | 58 GB free after cleanup (df)                |
| Memory                   | 20     | 16    | Pressure normal, swap 1.2 GB                 |
| Security                 | 20     | 15    | Firewall off                                 |
| Backup and recovery      | 15     | 5     | No Time Machine destination                  |
| Auto-start and leftovers | 15     | 13    | 2 dead LaunchAgents left for the user        |
| Development environment  | 10     | 9     | All dev commands resolve                     |
| **Total**                | **100**| **75**|                                              |

What blocks a higher score, and who has to do it:
- No backup destination: needs an external disk from you
- Firewall: turn it on in System Settings (your decision)
```

## Safety rules

- Never uses `rm -rf`. It runs a cache manager's own command or moves the target to `/tmp$HOME/...`.
- Never touches security software (VPN, antivirus, DLP, certificate tools).
- Does not run `sudo`, install macOS updates, upgrade casks, stop services, prune Docker, or end processes on its own. It lists them and acts only on what you pick. An admin action you picked runs through a macOS password dialog, after it shows you the script. A plaintext secret is reported with its value masked.
- Keeps the list of moved paths and the rebuild recipes in `~/.mac-optimize/rebuild/<date>/`, so they survive the reboot that empties `/tmp`.
- `/tmp` is on the same volume, so moving files there does not free space. It asks before emptying it.
- Free space is verified with `df`, because a `du` figure is not always reclaimable (APFS clones).

## Related

[mac-ops](https://github.com/seunggabi/mac-ops) is the companion tool for unattended upkeep: a zsh CLI that cleans caches, temp files, logs, and leftovers of removed apps every hour from launchd, into its own trash with a 72-hour restore. This skill does the interactive part: a one-off diagnosis, the decisions only you can make, and a before/after report. When mac-ops is installed, the skill reads its status, settings, and run history, and recommends installing it when you want the cleanup to keep happening without asking.

```bash
brew install seunggabi/tap/mac-ops
```

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=seunggabi/mac-optimize&type=Date)](https://star-history.com/#seunggabi/mac-optimize&Date)

## License

MIT
