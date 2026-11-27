# mac-optimize

A Claude Code skill that checks and cleans up a macOS system.

Five parallel agents handle cache cleanup, Homebrew updates, service checks, disk analysis, and system health, then report before/after numbers in tables.

## Install

```bash
npx skills add seunggabi/mac-optimize      # into the current project
npx skills add seunggabi/mac-optimize -g   # for your user, all projects
```

### Manual install

```bash
git clone https://github.com/seunggabi/mac-optimize.git
mkdir -p ~/.claude/skills
ln -s "$(pwd)/mac-optimize/skills/mac-optimize" ~/.claude/skills/mac-optimize
```

## Usage

```
/mac-optimize
```

It also triggers on phrases like "mac is slow", "disk cleanup", or "system check".

## What it does

| Phase | What happens |
|---|---|
| Phase 1 | Diagnose CPU, memory, swap, disk, caches, and auto-start items |
| Phase 2 | Run 5 agents in parallel |
| Phase 3 | Collect results, report items moved to `/tmp` and commands that need sudo |
| Phase 4 | Final Before → After report |

| Agent | Job |
|---|---|
| cache-cleaner | pip and npm caches, old logs, browser and IDE caches |
| brew-updater | formula upgrade, autoremove, cleanup, doctor |
| service-optimizer | brew services, long-running processes, dead auto-start items |
| disk-analyzer | large folders, Docker, node_modules, large files (read-only) |
| system-health | macOS updates, volume check, swap, uptime, battery (read-only) |

## Safety rules

- Never uses `rm -rf`. It runs a cache manager's own command or moves the target to `/tmp$HOME/...`.
- Never touches security software (VPN, antivirus, DLP, certificate tools).
- Does not run `sudo`, install macOS updates, upgrade casks, stop services, prune Docker, or end processes on its own. It lists them and acts only on what you pick.
- `/tmp` is on the same volume, so moving files there does not free space. It asks before emptying it.
- Free space is verified with `df`, because a `du` figure is not always reclaimable (APFS clones).

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=seunggabi/mac-optimize&type=Date)](https://star-history.com/#seunggabi/mac-optimize&Date)

## License

MIT
