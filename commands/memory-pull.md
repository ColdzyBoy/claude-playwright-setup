---
description: Pull Claude memory from the remote GitLab repo into this machine's local Claude memory directory for the current project (remote → local, overwrites local).
---

You are syncing Claude Code auto-memory **from a remote GitLab repository down to this machine's local memory directory** for the current project.

Direction: **remote → local**. Local files will be overwritten with remote content.

Execute every Bash block below in order. Stop and report to the user if any step fails.

---

## Step 1 — Initialize machine-local config (idempotent)

```bash
set -e
CONFIG_DIR="$HOME/.claude-memory-sync"
CONFIG_FILE="$CONFIG_DIR/config.json"
CACHE_DIR="$CONFIG_DIR/cache"
DEFAULT_REMOTE="ssh://git@gitlab.superstart.co.kr:13022/labiter/labiter-testing-memory.git"
DEFAULT_BRANCH="main"

mkdir -p "$CONFIG_DIR"
if [ ! -f "$CONFIG_FILE" ]; then
  cat > "$CONFIG_FILE" <<EOF
{
  "remote": "$DEFAULT_REMOTE",
  "branch": "$DEFAULT_BRANCH",
  "projects": {},
  "sync_global": true
}
EOF
  echo "Initialized config: $CONFIG_FILE"
fi

REMOTE_URL=$(python3 -c "import json; print(json.load(open('$CONFIG_FILE'))['remote'])")
BRANCH=$(python3 -c "import json; print(json.load(open('$CONFIG_FILE'))['branch'])")
```

## Step 2 — Resolve slug and logical key

```bash
SLUG=$(pwd | sed 's|/|-|g')
DEFAULT_KEY=$(basename "$(pwd)")
LOGICAL_KEY=$(python3 - <<PY
import json
cfg = json.load(open("$CONFIG_FILE"))
print(cfg.get("projects", {}).get("$SLUG", "$DEFAULT_KEY"))
PY
)
SYNC_GLOBAL=$(python3 -c "import json; print(json.load(open('$CONFIG_FILE')).get('sync_global', True))")
MEM_DIR="$HOME/.claude/projects/$SLUG/memory"

echo "slug:        $SLUG"
echo "logical_key: $LOGICAL_KEY"
echo "mem_dir:     $MEM_DIR"
echo "sync_global: $SYNC_GLOBAL"
```

## Step 3 — Clone or update cache

```bash
if [ ! -d "$CACHE_DIR/.git" ]; then
  echo "Cloning $REMOTE_URL → $CACHE_DIR"
  git clone --branch "$BRANCH" "$REMOTE_URL" "$CACHE_DIR"
else
  echo "Updating cache from origin/$BRANCH"
  git -C "$CACHE_DIR" fetch origin "$BRANCH"
  git -C "$CACHE_DIR" reset --hard "origin/$BRANCH"
fi
COMMIT=$(git -C "$CACHE_DIR" rev-parse --short HEAD)
echo "Cache at commit: $COMMIT"
```

## Step 4 — Copy cache → local Claude memory dir

```bash
mkdir -p "$MEM_DIR"
PROJECT_DIR="$CACHE_DIR/projects/$LOGICAL_KEY"

if [ "$SYNC_GLOBAL" = "True" ] && [ -d "$CACHE_DIR/global" ]; then
  rsync -a --exclude=MEMORY.md "$CACHE_DIR/global/" "$MEM_DIR/"
fi

if [ -d "$PROJECT_DIR" ]; then
  rsync -a --exclude=MEMORY.md "$PROJECT_DIR/" "$MEM_DIR/"
fi

{
  [ -f "$CACHE_DIR/global/MEMORY.md" ] && cat "$CACHE_DIR/global/MEMORY.md"
  [ -f "$PROJECT_DIR/MEMORY.md" ] && cat "$PROJECT_DIR/MEMORY.md"
} 2>/dev/null | awk 'NF && !seen[$0]++' > "$MEM_DIR/MEMORY.md"

[ -s "$MEM_DIR/MEMORY.md" ] || : > "$MEM_DIR/MEMORY.md"
```

## Step 5 — Report

Run:

```bash
echo "--- Synced memory files ---"
ls -1 "$MEM_DIR"
echo ""
echo "--- MEMORY.md preview (first 15 lines) ---"
head -15 "$MEM_DIR/MEMORY.md" 2>/dev/null || echo "(empty)"
echo ""
echo "--- Source ---"
echo "commit: $COMMIT"
echo "logical_key: $LOGICAL_KEY"
[ -d "$PROJECT_DIR" ] || echo "NOTE: no remote folder for logical_key '$LOGICAL_KEY' yet — push first to create it."
```

Then summarize to the user:

- Where memory landed: `$MEM_DIR`
- Logical key resolved (auto from basename, or override from config?)
- Number of files synced + commit hash
- If `projects/$LOGICAL_KEY` did not exist in the remote, tell the user that this is a first-time project. They should populate memory via normal Claude usage, then run `/memory-push` to seed the remote.
