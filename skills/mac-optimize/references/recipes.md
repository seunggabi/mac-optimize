# Shell recipes for every team

Pasted into every team prompt, and read by the main agent before its own moves in Phase 3. Each pattern went wrong in a real run.

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
