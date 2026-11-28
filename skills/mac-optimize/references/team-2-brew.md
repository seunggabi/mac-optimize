# Team 2: brew-updater (Homebrew)

This file is the body of Team 2's prompt. Core principles, Data-loss rules, and the security keyword list from SKILL.md are pasted into the same prompt; where this file seems to allow more, they win.

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
- Do not run the `rm -rf` or `sudo` commands that `brew doctor` suggests; report them

brew commands:
```bash
# Always disable auto-update, and always name the formulae to upgrade
HOMEBREW_NO_AUTO_UPDATE=1 brew outdated --formula
HOMEBREW_NO_AUTO_UPDATE=1 brew upgrade --formula <names>
HOMEBREW_NO_AUTO_UPDATE=1 brew cleanup --prune=all
HOMEBREW_NO_AUTO_UPDATE=1 brew doctor
```
