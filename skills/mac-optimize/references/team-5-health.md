# Team 5: system-health (system health)

This file is the body of Team 5's prompt. Core principles, Data-loss rules, and the security keyword list from SKILL.md are pasted into the same prompt, and the prompt states the scope in its first line; where this file seems to allow more, they and the scope win.

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
