---
description: Push local Claude memory changes to the remote GitLab repo (local → remote, commit & push).
---

You are syncing Claude Code auto-memory **from this machine's local memory directory up to the remote GitLab repository** for the current project.

Direction: **local → remote**. Local changes are committed to the cache and pushed.

Always rebase first to avoid clobbering work from other machines. If a conflict occurs, stop and ask the user.

Execute every Bash block below in order.

---

## Step 1 — Load config and resolve paths

```bash
set -e
CONFIG_DIR="$HOME/.claude-memory-sync"
CONFIG_FILE="$CONFIG_DIR/config.json"
CACHE_DIR="$CONFIG_DIR/cache"

if [ ! -f "$CONFIG_FILE" ]; then
  echo "ERROR: $CONFIG_FILE not found. Run /memory-pull first to initialize." >&2
  exit 1
fi

REMOTE_URL=$(python3 -c "import json; print(json.load(open('$CONFIG_FILE'))['remote'])")
BRANCH=$(python3 -c "import json; print(json.load(open('$CONFIG_FILE'))['branch'])")

SLUG=$(pwd | sed 's|/|-|g')
DEFAULT_KEY=$(basename "$(pwd)")
LOGICAL_KEY=$(python3 - <<PY
import json
cfg = json.load(open("$CONFIG_FILE"))
print(cfg.get("projects", {}).get("$SLUG", "$DEFAULT_KEY"))
PY
)
MEM_DIR="$HOME/.claude/projects/$SLUG/memory"

echo "slug:        $SLUG"
echo "logical_key: $LOGICAL_KEY"
echo "mem_dir:     $MEM_DIR"

if [ ! -d "$MEM_DIR" ]; then
  echo "ERROR: $MEM_DIR does not exist. Nothing to push." >&2
  exit 1
fi
```

## Step 2 — Ensure cache exists and is up-to-date with remote

```bash
if [ ! -d "$CACHE_DIR/.git" ]; then
  echo "Cache missing. Cloning..."
  git clone --branch "$BRANCH" "$REMOTE_URL" "$CACHE_DIR"
fi

git -C "$CACHE_DIR" fetch origin "$BRANCH"
git -C "$CACHE_DIR" checkout "$BRANCH"

# Rebase any local cache commits on top of origin (safe: cache only holds sync commits)
if ! git -C "$CACHE_DIR" rebase "origin/$BRANCH"; then
  echo "ERROR: rebase failed in $CACHE_DIR. Resolve manually, then re-run /memory-push." >&2
  exit 2
fi
```

## Step 3 — Copy local memory → cache (project folder only; global is read-only by default)

```bash
PROJECT_DIR="$CACHE_DIR/projects/$LOGICAL_KEY"
mkdir -p "$PROJECT_DIR"

# Mirror local memory dir into the project folder (delete files removed locally too)
rsync -a --delete "$MEM_DIR/" "$PROJECT_DIR/"
```

## Step 4 — Commit and push if there are changes

```bash
cd "$CACHE_DIR"

git add -A "projects/$LOGICAL_KEY"

if git diff --cached --quiet; then
  echo "No changes to push."
  exit 0
fi

MACHINE=$(hostname -s 2>/dev/null || hostname)
TIMESTAMP=$(date -u +%FT%TZ)
COMMIT_MSG="memory sync: $LOGICAL_KEY @ $MACHINE @ $TIMESTAMP"

git commit -m "$COMMIT_MSG"
git push origin "$BRANCH"

NEW_COMMIT=$(git rev-parse --short HEAD)
echo "Pushed: $NEW_COMMIT"
echo "Message: $COMMIT_MSG"
```

## Step 5 — Report to user

Tell the user:

- Logical key + machine name used in the commit message
- New commit hash (short)
- File counts before/after (use `ls -1 "$PROJECT_DIR" | wc -l`)
- If "No changes to push." was printed, say so clearly — nothing was committed.

If Step 2 or 3 failed with a conflict / rsync error, do NOT retry. Explain the failure and ask the user to inspect `$CACHE_DIR` manually.
