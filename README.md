# labiter-memory-sync

A Claude Code plugin that syncs Claude's auto-memory directory bidirectionally with a private GitLab repository.

The plugin itself ships **only four slash commands**. The memory data lives in a remote git repo, and each machine's `~/.claude/projects/<slug>/memory/` is treated as a local working copy that Claude Code's auto-memory system automatically loads on every conversation.

```
[remote GitLab repo]
        ↕  git fetch / push
[local cache ~/.claude-memory-sync/cache/]
        ↕  rsync
[Claude memory ~/.claude/projects/<slug>/memory/]
        ↑  auto-loaded by Claude Code on every conversation
```

## Remote repo

```
ssh://git@gitlab.superstart.co.kr:13022/labiter/labiter-testing-memory.git
```

> SSH access requires your machine's public key to be registered with GitLab. If you only have HTTPS access, override `remote` in `~/.claude-memory-sync/config.json` to the HTTPS URL.

## Slash commands

| Command | Direction | Behavior |
|---|---|---|
| `/memory-pull` | remote → local | Update cache, then overwrite the local memory directory |
| `/memory-push` | local → remote | Mirror the local memory directory into the cache, rebase, then push |
| `/memory-sync` | bidirectional | Run pull first, then push |
| `/memory-status` | read-only | Show cache state, ahead/behind counts, and a per-file diff summary |

## Directory layout

### Remote repo

```
labiter-testing-memory/
├── README.md
├── global/                              # (optional) shared across all projects
│   ├── MEMORY.md
│   └── user_*.md / feedback_*.md
└── projects/
    ├── playwright-run-test/             # logical_key, usually basename(project path)
    │   ├── MEMORY.md
    │   ├── feedback_*.md
    │   ├── project_*.md
    │   ├── reference_*.md
    │   └── user_*.md
    └── playwright-hub/
        └── ...
```

### Local (per-machine)

```
~/.claude-memory-sync/
├── config.json                          # per-machine. NOT committed to the remote repo
└── cache/                               # local working tree for the remote repo (kept permanently)
    └── (same layout as the remote repo)

~/.claude/projects/<slug>/memory/        # what Claude Code actually reads
├── MEMORY.md                            # merged index (global + project)
└── *.md                                 # memory detail files
```

**`slug`**: the absolute path of the current working directory with `/` replaced by `-`. Example: `/Users/labiter/projects/foo` → `-Users-labiter-projects-foo`.

**`logical_key`**: the folder name under `projects/` in the remote repo. Defaults to `basename($(pwd))`. When the same project lives at different absolute paths on different machines, set an explicit mapping in `config.json` so they all sync to the same logical key.

## config.json

`~/.claude-memory-sync/config.json` — created automatically the first time you run `/memory-pull`.

Because the slug differs per machine, only fill in the `projects` map when the default basename-based mapping is wrong.

```json
{
  "remote": "ssh://git@gitlab.superstart.co.kr:13022/labiter/labiter-testing-memory.git",
  "branch": "main",
  "sync_global": true,
  "projects": {
    "-Users-labiter-projects-playwright-run-test": "playwright-run-test"
  }
}
```

See [`examples/config.example.json`](examples/config.example.json) for a fuller example.

## Installation

### 1) Initialize the remote repo (one-time, from any machine)

```bash
# After creating an empty repo on GitLab:
git clone ssh://git@gitlab.superstart.co.kr:13022/labiter/labiter-testing-memory.git
cd labiter-testing-memory
mkdir -p projects global
echo "# labiter memory" > README.md
echo "MEMORY.md merge=union" > .gitattributes
git add -A && git commit -m "init" && git push -u origin main
```

> The `MEMORY.md merge=union` rule in `.gitattributes` lets git automatically union-merge index lines when two machines add different bullets concurrently.

### 2) Register the plugin in Claude Code

```bash
claude plugin marketplace add https://github.com/ColdzyBoy/claude-playwright-setup.git
claude plugin install labiter-memory-sync@labiter-memory-sync
```

Restart Claude Code so the slash commands get picked up.

### 3) Bootstrap on a new machine

```text
1. Install the plugin (step 2 above)
2. Restart Claude Code
3. cd into your project directory
4. Run /memory-pull
   → config.json is auto-created
   → the remote repo is cloned into the cache
   → ~/.claude/projects/<slug>/memory/ is populated
5. From the next conversation onward, auto-memory loads automatically
```

### 4) Mapping a project that lives at different absolute paths on different machines

```bash
# Example: a laptop where the same project lives at /Users/jaewon/code/playwright-run-test
python3 -c "
import json, pathlib
p = pathlib.Path.home() / '.claude-memory-sync' / 'config.json'
cfg = json.loads(p.read_text())
cfg.setdefault('projects', {})['-Users-jaewon-code-playwright-run-test'] = 'playwright-run-test'
p.write_text(json.dumps(cfg, indent=2, ensure_ascii=False))
"
```

Or edit `config.json` directly, using [`examples/config.example.json`](examples/config.example.json) as a reference.

## Typical workflow

```
[morning]   /memory-pull       # pick up what other machines pushed yesterday
[working]   (Claude Code accumulates new memory files as you go)
[evening]   /memory-push       # commit & push today's work to the remote
```

Or, in one call:

```
[before work]  /memory-sync
[after work]   /memory-sync
```

## Conflict behavior

- `/memory-pull`: uses `git fetch + reset --hard origin/<branch>` to force the cache in sync with the remote. The cache is only ever touched by this plugin, so origin is always treated as the source of truth — no conflicts at the cache layer.
- `/memory-push`: runs `git rebase origin/<branch>` before pushing. If another machine has already pushed sync commits that touched the same files you have local edits in, rebase will fail. The command **stops** and asks you to resolve manually inside `~/.claude-memory-sync/cache` (`git status`, resolve conflicts, `git rebase --continue`).
- `/memory-sync`: pull overwrites the local memory directory, so files that only exist locally survive but locally **modified** files may be lost. To stay safe, run `/memory-status` first to inspect the diff, or use `/memory-push` alone.

## Security & hygiene

- The remote repo **must be private**. Claude's auto-memory captures conversational context — internal system names, account references, ticket numbers, etc.
- Prefer token-based HTTPS auth (GitLab Personal Access Token in your OS keychain) or SSH keys.
- `~/.claude-memory-sync/cache` is not re-cloned every run; it's a permanent working copy. Disk usage is typically just a few MB.
- `~/.claude-memory-sync/config.json` is a per-machine setting and must **not** be committed to the remote repo.

## Troubleshooting

| Symptom | Cause / Resolution |
|---|---|
| `MEMORY.md` is empty after `/memory-pull` | The remote repo has no `projects/<logical_key>/` (or `global/`) folder yet. Let auto-memory accumulate files locally, then run `/memory-push` to seed the remote. |
| `git push` returns 403 / Authentication failed | No GitLab Personal Access Token or SSH key is registered. Run `git -C ~/.claude-memory-sync/cache push` directly to see the auth prompt. |
| Same project syncs to different folders on different machines | basenames differed, or the slug differs and you haven't mapped it. Add the slug → logical_key mapping in `config.json`'s `projects` field. |
| `/memory-push` stops during rebase | Two machines edited the same file. Resolve manually: enter `~/.claude-memory-sync/cache`, fix conflicts, run `git rebase --continue && git push`. |

## File layout (this repo)

```
labiter-memory-sync-plugin/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── commands/
│   ├── memory-pull.md
│   ├── memory-push.md
│   ├── memory-sync.md
│   └── memory-status.md
├── examples/
│   └── config.example.json
└── README.md
```
