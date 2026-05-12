---
description: Bidirectional sync — pull remote changes, then push local changes (pull → push in one shot).
---

You are running a **bidirectional sync** of Claude Code auto-memory between this machine and the remote GitLab repo.

Order: **pull first (remote → local), then push (local → remote)**. The pull step intentionally overwrites local files with remote content for files that exist on both sides — anything genuinely new on this machine survives because it does not exist in the remote yet.

> WARNING: if you have actively edited memory on this machine AND the same files were updated on another machine since the last sync, the local edits will be lost when pull overwrites them. Use `/memory-push` directly if you want to avoid that, or run `/memory-status` first to inspect.

Execute the steps below. Stop on any non-zero exit.

---

## Step 1 — Run /memory-pull logic

Execute the same Step 1–4 sequence from `commands/memory-pull.md`:

1. Initialize config (idempotent)
2. Resolve `SLUG`, `LOGICAL_KEY`, `MEM_DIR`, `SYNC_GLOBAL`
3. Clone or update cache (`git fetch` + `git reset --hard origin/<branch>`)
4. Copy `cache/global/` + `cache/projects/<key>/` into `$MEM_DIR` (rsync; MEMORY.md merge)

Print "PULL COMPLETE — at commit <hash>" when done.

## Step 2 — Run /memory-push logic

Execute the same Step 2–4 sequence from `commands/memory-push.md`:

1. Re-fetch + rebase cache against `origin/<branch>` (no-op if Step 1 just finished, but safe)
2. `rsync -a --delete "$MEM_DIR/" "$CACHE_DIR/projects/$LOGICAL_KEY/"`
3. If no diff in cache → "No changes to push." and exit 0
4. Otherwise `git commit` (message: `memory sync: <key> @ <host> @ <utc-ts>`) and `git push origin <branch>`

Print "PUSH COMPLETE — at commit <hash>" or "No changes to push."

## Step 3 — Combined report

Tell the user, in this order:

1. Pull source commit (the remote commit that was pulled in)
2. Push result (new commit hash, or "no changes")
3. Logical key used + sync direction summary in plain language

If conflicts occurred in either direction, stop after the failing step and ask the user to resolve in `~/.claude-memory-sync/cache` manually. Do not attempt to auto-resolve memory conflicts — they require human judgment about which version is correct.
