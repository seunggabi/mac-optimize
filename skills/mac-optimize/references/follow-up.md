# Phase 3 in detail: collect results and follow up

Read this file before Phase 3 step 1. SKILL.md lists the steps in order; this file says how to carry out each one. The Data-loss rules in SKILL.md apply to every step ("Before any signal" is one of them).

After the agents finish:

1. **Java hprof files**: find them with `find ~ -maxdepth 1 -name 'java_error_in_*.hprof'` and move them to `/tmp/` (JVM crash dumps, safe to delete)
2. **brew upgrade lock conflicts**: if a package failed on a lock, report it and the PID holding the lock (`pgrep -x brew`). Do not kill it
3. **Trash**: if the Trash is large, recommend emptying it (do not empty it automatically)
4. **Decisions for the user**: show the lists the agents only reported (running services, Docker, idle coding-agent sessions, stale build output, unused apps, duplicate runtimes, dead auto-start items, scheduled cleaners, plaintext secrets) in one question and ask which to act on. No team collects keychain leftovers: Phase 3 collects that list itself, and only when the user asks for it. Act only on what they pick, and apply Core principles in SKILL.md (skip repositories with a live process; remove managed apps through their manager). How to carry out the common picks:
   - **Docker images**: `docker image prune -a --filter "until=720h" -f` removes unused images created more than 30 days ago. `until` filters on the creation date, not the pull date, so an old image pulled today is removed too; such images need a re-pull or rebuild. Never add `--volumes` and never run `docker volume prune`. Build cache follows the same age rule: `docker builder prune --filter "until=720h" -f`
   - **Docker Desktop**: quit it only when no container is running, then verify (see "Quitting Docker Desktop" below)
   - **Idle agent sessions**: "idle" is the user's call. Show each session on its own row with its working directory and uptime, and end only the root PIDs the user names. Never the session running this skill or its ancestors (walk up the parent links from the current shell's PID). A session waiting for user input also has no child other than its MCP servers, so that check does not prove a session is abandoned. For a named session, confirm it has no child other than its MCP servers, send `kill -TERM` to the session root, and check that the children exited. Leave sessions started within the last day unless the user names them
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
8. **Post-check**: run every check in `references/post-check.md`
9. **Re-setup list**: for everything removed, tell the user what they need to run before using it again (`terraform init`, `npm install`, recreate the virtualenv, re-pull Docker images, cold-boot the emulator)

## Recipes for the picks

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

### Stopping a service for good (only after the user names it)
```bash
# brew services stop removes the plist, so the service does not come back after a reboot
brew services stop kafka

# Verify
ls ~/Library/LaunchAgents/homebrew.mxcl.kafka.plist  # should not exist
```
