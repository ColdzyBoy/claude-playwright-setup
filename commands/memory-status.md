---
description: Show sync status — local vs remote diff, cache state, resolved logical key, and config snapshot. Read-only.
---

You will inspect the current Claude memory sync state **without making any changes**. Read-only.

Execute every Bash block. Then summarize.

---

## Step 1 — Show config and resolution

```bash
set -e
CONFIG_DIR="$HOME/.claude-memory-sync"
CONFIG_FILE="$CONFIG_DIR/config.json"
CACHE_DIR="$CONFIG_DIR/cache"

echo "=== config ==="
if [ -f "$CONFIG_FILE" ]; then
  cat "$CONFIG_FILE"
else
  echo "(missing — run /memory-pull to initialize)"
fi
echo ""

SLUG=$(pwd | sed 's|/|-|g')
DEFAULT_KEY=$(basename "$(pwd)")
if [ -f "$CONFIG_FILE" ]; then
  LOGICAL_KEY=$(python3 - <<PY
import json
cfg = json.load(open("$CONFIG_FILE"))
print(cfg.get("projects", {}).get("$SLUG", "$DEFAULT_KEY"))
PY
)
else
  LOGICAL_KEY="$DEFAULT_KEY"
fi
MEM_DIR="$HOME/.claude/projects/$SLUG/memory"

echo "=== resolution ==="
echo "cwd:          $(pwd)"
echo "slug:         $SLUG"
echo "logical_key:  $LOGICAL_KEY"
echo "mem_dir:      $MEM_DIR"
echo ""
```

## Step 2 — Local memory directory state

```bash
echo "=== local memory dir ==="
if [ -d "$MEM_DIR" ]; then
  echo "files: $(ls -1 "$MEM_DIR" | wc -l | tr -d ' ')"
  ls -la "$MEM_DIR"
else
  echo "(does not exist)"
fi
echo ""
```

## Step 3 — Cache state

```bash
echo "=== cache ==="
if [ -d "$CACHE_DIR/.git" ]; then
  echo "remote:  $(git -C "$CACHE_DIR" config --get remote.origin.url)"
  echo "branch:  $(git -C "$CACHE_DIR" rev-parse --abbrev-ref HEAD)"
  echo "head:    $(git -C "$CACHE_DIR" rev-parse --short HEAD)"
  git -C "$CACHE_DIR" fetch --quiet origin 2>/dev/null || true
  AHEAD=$(git -C "$CACHE_DIR" rev-list --count "@{u}..HEAD" 2>/dev/null || echo "?")
  BEHIND=$(git -C "$CACHE_DIR" rev-list --count "HEAD..@{u}" 2>/dev/null || echo "?")
  echo "ahead:   $AHEAD  (commits in cache not yet pushed)"
  echo "behind:  $BEHIND  (commits on remote not yet pulled)"
  PROJECT_DIR="$CACHE_DIR/projects/$LOGICAL_KEY"
  if [ -d "$PROJECT_DIR" ]; then
    echo "remote project folder: present ($(ls -1 "$PROJECT_DIR" | wc -l | tr -d ' ') files)"
  else
    echo "remote project folder: MISSING (first-time push will create projects/$LOGICAL_KEY/)"
  fi
else
  echo "(cache not cloned yet — run /memory-pull)"
fi
echo ""
```

## Step 4 — Diff: local vs cache

```bash
echo "=== diff: local vs cache (project scope only) ==="
PROJECT_DIR="$CACHE_DIR/projects/$LOGICAL_KEY"
if [ -d "$MEM_DIR" ] && [ -d "$PROJECT_DIR" ]; then
  diff -rq "$MEM_DIR" "$PROJECT_DIR" || true
else
  echo "(skipped — one or both directories missing)"
fi
```

## Step 5 — Summary

After running the blocks, give the user a short summary:

1. Whether config is initialized
2. Logical key (and whether it was auto-derived from basename or set explicitly)
3. Cache `ahead`/`behind` counts → recommend `/memory-pull` if behind, `/memory-push` if ahead, `/memory-sync` if both
4. Whether `projects/$LOGICAL_KEY` exists on the remote yet
5. File-level diff summary from Step 4 (count of differing/new files, no full diff body)

Do NOT modify any files in this command. It is purely a read.
