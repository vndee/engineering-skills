---
name: disk-cleanup
description: Use when the user wants to free disk space, investigate disk usage, clean caches, remove unused artifacts, or when disk capacity is high. Also use when asked to "clean up", "free space", "disk full", or "what's using disk space". Always uses AskUserQuestion for every deletion decision.
---

# Interactive Disk Cleanup

## Overview

Systematically scan macOS disk usage across all known bloat categories, present findings, and let the user pick exactly what to delete using AskUserQuestion. Never delete without explicit user approval per item.

**Core principle:** Scan everything, present clearly, let the user decide. Use AskUserQuestion tool for EVERY deletion decision — never assume.

## Scan Order

Run all scans in parallel where possible for speed.

### Phase 1: Overview

```bash
# Overall disk state
df -h /
diskutil apfs list | grep -E "Capacity|Size" | head -6

# Top-level user directories
du -sh ~/Library/Caches ~/Library/Developer ~/Library/Application\ Support ~/Library/Containers ~/Library/Logs ~/Downloads ~/Documents ~/Desktop ~/Movies ~/Pictures 2>/dev/null
```

### Phase 2: Known Bloat Categories

Scan these categories in parallel. Present results as a summary table with sizes.

#### 1. Caches (~5-20 GB typical)
```bash
du -sh ~/Library/Caches/* 2>/dev/null | sort -rh | head -15
```
**Common large caches:** go-build, browser caches, Playwright, app updater ShipIt caches, gopls, node-gyp

#### 2. Docker (~5-20 GB typical)
```bash
docker system df 2>/dev/null
```
**Look for:** reclaimable images, unused volumes (often 90%+ reclaimable), build cache

#### 3. Xcode/Developer (~5-15 GB typical)
```bash
du -sh ~/Library/Developer/Xcode/DerivedData ~/Library/Developer/Xcode/iOS\ DeviceSupport ~/Library/Developer/CoreSimulator 2>/dev/null
xcrun simctl runtime list 2>/dev/null
```
**Look for:** DerivedData (always safe to clear), old iOS DeviceSupport, unused simulator runtimes (often 15-20 GB each)

#### 4. Homebrew (~5-25 GB typical)
```bash
# Package sizes
brew list --formula | while read f; do echo "$(du -sh $(brew --prefix)/Cellar/$f 2>/dev/null)"; done | sort -rh | head -20
# Top-level packages (not dependencies)
brew leaves
# What depends on large packages
brew uses --installed <package>
```
**Look for:** Unused large packages (qt, llvm, opencv chains), old Python versions, packages for tools no longer used

#### 5. Application Support (~10-60 GB typical)
```bash
ls -d ~/Library/"Application Support"/*/ 2>/dev/null | while read d; do du -sh "$d" 2>/dev/null; done | sort -rh | head -20
```
**Look for:** IDE data (Cursor, VS Code), chat apps (Zalo, Slack, Lark), browser profiles (Arc), screen recorders (Cap)

#### 6. Containers (~5-25 GB typical)
```bash
du -sh ~/Library/Containers/* 2>/dev/null | sort -rh | head -10
```
**Look for:** Docker container data, old app containers

#### 7. Dev Tool Directories (~5-15 GB typical)
```bash
du -sh ~/.cargo ~/.rustup ~/.npm ~/.nvm ~/.bun ~/.pyenv ~/go ~/.gradle ~/.m2 ~/miniconda3 ~/.conda ~/.cache ~/.local 2>/dev/null
```
**Look for:** Unused language toolchains (Rust, old Node versions), stale caches

#### 8. Project Artifacts (often largest — 10-50 GB)
```bash
# node_modules across all projects
find ~/Documents -maxdepth 4 -name "node_modules" -type d -exec du -sh {} \; 2>/dev/null | sort -rh
# Python venvs
find ~/Documents -maxdepth 4 -type d \( -name ".venv" -o -name "venv" \) -exec du -sh {} \; 2>/dev/null | sort -rh
# Large .git directories
find ~/Documents -maxdepth 3 -name ".git" -type d -exec du -sh {} \; 2>/dev/null | sort -rh | head -15
```
**node_modules and venvs are always safe to delete** — restorable with `npm install` / `pip install`.

#### 9. System Data
```bash
# iOS Simulator runtimes (often 15-20 GB EACH)
xcrun simctl runtime list
# Nix store (if unused)
du -sh /nix 2>/dev/null
# Time Machine snapshots
tmutil listlocalsnapshots /
# Temp directories
du -sh /private/var/folders/*/*/* 2>/dev/null | sort -rh | head -10
# Claude Code temp (can grow to 30+ GB from runaway tasks)
du -sh /private/tmp/claude-* 2>/dev/null
```

#### 10. Stale App Data
Cross-reference `ls /Applications/` against data in `~/Library/Application Support/`, `~/Library/Caches/`, `~/Library/Containers/` to find remnants from uninstalled apps.

```bash
ls /Applications/ | sort
# Then check for data from apps NOT in /Applications/
# Common: PyCharm, IntelliJ, WebStorm, GoLand remnants
find ~/Library -maxdepth 3 -iname "*pycharm*" -o -iname "*intellij*" -o -iname "*webstorm*" -o -iname "*goland*" 2>/dev/null
```

## Presentation Format

After scanning, present a summary table:

```
| Category              | Size   | Recoverable | Notes                          |
|-----------------------|--------|-------------|--------------------------------|
| Docker volumes        | 8.5 GB | ~8 GB       | 93% reclaimable                |
| Xcode DerivedData     | 4.7 GB | ~4.7 GB     | Always safe, rebuilds on demand|
| iOS Simulator 17.0.1  | 15 GB  | ~15 GB      | Old runtime                    |
| node_modules (12 dirs)| 9.4 GB | ~9.4 GB     | Always restorable              |
| ...                   |        |             |                                |
```

## Deletion Flow

**CRITICAL: Always use AskUserQuestion for deletion decisions.**

### Safe bulk operations (still ask first)
For categories where everything is safe to delete (caches, DerivedData), ask once:
```
AskUserQuestion: "Clear all Xcode DerivedData? (4.7 GB — always rebuilds on demand)"
Options: Delete / Keep
```

### Per-item decisions
For project artifacts (node_modules, venvs) and app data, ask per item. Batch up to 4 questions per AskUserQuestion call:
```
AskUserQuestion (4 questions):
1. "Delete NewAI/project-a/node_modules? (1.2 GB)"
2. "Delete NewAI/project-b/node_modules? (800 MB)"
3. "Delete indie-hacker/project-c/.venv? (500 MB)"
4. "Delete indie-hacker/project-d/.venv? (300 MB)"
Each with: Delete / Keep options
```

### After each batch
- Execute deletions for approved items immediately
- Move to next batch

### After all deletions
Run `df -h /` and show before/after comparison.

## Cleanup Commands Reference

| Category | Command | Needs sudo? |
|---|---|---|
| Docker full prune | `docker system prune -a --volumes -f` | No |
| Go build cache | `go clean -cache` | No |
| Xcode DerivedData | `rm -rf ~/Library/Developer/Xcode/DerivedData/*` | No |
| iOS DeviceSupport | `rm -rf ~/Library/Developer/Xcode/iOS\ DeviceSupport/*` | No |
| Simulator runtimes | `xcrun simctl runtime delete <name>` | No |
| Unavailable simulators | `xcrun simctl delete unavailable` | No |
| Homebrew cleanup | `brew cleanup --prune=all && brew autoremove` | No |
| Homebrew uninstall | `brew uninstall <pkg>` (check `brew uses --installed <pkg>` first) | No |
| Playwright cache | `rm -rf ~/Library/Caches/ms-playwright*` | No |
| Rust full remove | `rustup self uninstall -y` | No |
| Nix volume delete | `diskutil apfs deleteVolume <disk>` | May need sudo |
| Claude Code temp | `rm -rf /private/tmp/claude-*` or specific large `.output` files | May need sudo |
| Cursor sandbox cache | `rm -rf /private/var/folders/.../T/cursor-sandbox-cache` | May need sudo |

## Common Mistakes

- **Deleting Claude Desktop vm_bundles** (~11 GB) — it's the active runtime VM, will just re-download
- **Deleting Cursor state.vscdb** — active database, will corrupt Cursor
- **Not checking `brew uses --installed`** before uninstalling — may break dependencies
- **Force-deleting /nix** — it's on the read-only system volume, use `diskutil apfs deleteVolume` for the Nix Store volume instead
- **Skipping /private/tmp/claude-*/** — runaway Claude Code background tasks can create 30+ GB output files
