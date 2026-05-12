# labiter-memory-sync

Claude Code 플러그인 — Claude의 auto-memory 디렉토리를 GitLab repo와 양방향 동기화한다.

플러그인 자체는 **슬래시 커맨드 4개만** 제공한다. 메모리 본체는 원격 git repo에 살고, 머신마다의 `~/.claude/projects/<slug>/memory/`는 그 repo의 캐시본을 받아 Claude Code auto-memory가 자동 로드하는 형태로 사용한다.

```
[원격 GitLab repo]
        ↕  git fetch / push
[로컬 캐시 ~/.claude-memory-sync/cache/]
        ↕  rsync
[Claude memory ~/.claude/projects/<slug>/memory/]
        ↑  Claude Code가 매 대화마다 자동 로드
```

## 원격 repo

```
https://gitlab.superstart.co.kr/labiter/labiter-testing-memory.git
```

## 슬래시 커맨드

| Command | 방향 | 동작 |
|---|---|---|
| `/memory-pull` | remote → local | 캐시 갱신 후 로컬 memory 디렉토리 덮어쓰기 |
| `/memory-push` | local → remote | 로컬 memory 디렉토리를 캐시에 미러링, rebase 후 push |
| `/memory-sync` | 양방향 | pull → push 순차 실행 |
| `/memory-status` | read-only | 캐시/로컬 상태, ahead/behind, diff 요약 |

## 디렉토리 레이아웃

### 원격 repo

```
labiter-testing-memory/
├── README.md
├── global/                              # (선택) 모든 프로젝트 공통
│   ├── MEMORY.md
│   └── user_*.md / feedback_*.md
└── projects/
    ├── playwright-run-test/             # logical key = 보통 basename(프로젝트 경로)
    │   ├── MEMORY.md
    │   ├── feedback_*.md
    │   ├── project_*.md
    │   ├── reference_*.md
    │   └── user_*.md
    └── playwright-hub/
        └── ...
```

### 머신 로컬

```
~/.claude-memory-sync/
├── config.json                          # 머신마다 다름. git에 들어가지 않음
└── cache/                               # 원격 repo의 git working tree (영구 보존)
    └── (원격 repo와 동일 구조)

~/.claude/projects/<slug>/memory/        # Claude Code가 실제로 읽는 곳
├── MEMORY.md                            # global + project 인덱스 머지본
└── *.md                                 # 메모리 본문 파일들
```

**`slug`**: 현재 작업 디렉토리의 절대 경로에서 `/`를 `-`로 치환한 문자열. 예: `/Users/labiter/projects/foo` → `-Users-labiter-projects-foo`.

**`logical_key`**: 원격 repo의 `projects/` 하위 폴더 이름. 기본값은 `basename($(pwd))`. 머신마다 절대 경로가 달라도 같은 logical key로 매핑하려면 `config.json`의 `projects` 필드로 명시한다.

## config.json

`~/.claude-memory-sync/config.json` — `/memory-pull`을 처음 실행할 때 기본값으로 자동 생성된다.

머신마다 슬러그가 달라지므로 명시 매핑이 필요할 때만 `projects` 필드를 채운다.

```json
{
  "remote": "https://gitlab.superstart.co.kr/labiter/labiter-testing-memory.git",
  "branch": "main",
  "sync_global": true,
  "projects": {
    "-Users-labiter-projects-playwright-run-test": "playwright-run-test"
  }
}
```

전체 예시는 [`examples/config.example.json`](examples/config.example.json) 참고.

## 설치

### 1) GitLab repo 준비 (최초 1회, 어느 머신에서든)

```bash
# 빈 repo를 GitLab에서 만든 뒤
git clone https://gitlab.superstart.co.kr/labiter/labiter-testing-memory.git
cd labiter-testing-memory
mkdir -p projects global
echo "# labiter memory" > README.md
echo "MEMORY.md merge=union" > .gitattributes
git add -A && git commit -m "init" && git push -u origin main
```

> `.gitattributes`의 `MEMORY.md merge=union` 설정은 두 머신이 인덱스에 서로 다른 줄을 추가했을 때 자동 합쳐주기 위한 것.

### 2) Claude Code에 플러그인 등록

이 디렉토리(`labiter-memory-sync-plugin`)를 Claude Code 플러그인으로 인식시키는 방법은 Claude Code 설치 방식에 따라 다르다:

- **로컬 개발 / 단일 머신**: `~/.claude/plugins/labiter-memory-sync/` 로 심볼릭 링크
  ```bash
  ln -s /Users/labiter/projects/labiter-memory-sync-plugin ~/.claude/plugins/labiter-memory-sync
  ```
- **팀 배포**: 이 디렉토리 자체를 git repo로 publish한 뒤 `/plugin install <url>` 또는 `settings.json`의 `plugins` 필드에 등록.

설치되면 Claude Code 재시작 후 `/memory-pull` 등의 커맨드가 잡힌다.

### 3) 새 머신에서 부트스트랩

```text
1. 플러그인 설치 (위 2번)
2. Claude Code에서 프로젝트 디렉토리로 진입
3. /memory-pull 실행
   → config.json 자동 생성
   → cache 디렉토리에 clone
   → ~/.claude/projects/<slug>/memory/ 에 복사
4. 이후 대화부터 auto-memory가 자동 로드됨
```

### 4) 다른 머신에서 절대 경로가 달라 logical key를 명시해야 할 때

```bash
# 예: 회사 노트북에서 같은 logical key 사용
python3 -c "
import json, pathlib
p = pathlib.Path.home() / '.claude-memory-sync' / 'config.json'
cfg = json.loads(p.read_text())
cfg.setdefault('projects', {})['-Users-jaewon-code-playwright-run-test'] = 'playwright-run-test'
p.write_text(json.dumps(cfg, indent=2, ensure_ascii=False))
"
```

또는 `examples/config.example.json`을 참고해 직접 편집.

## 일반적인 사용 흐름

```
[아침] /memory-pull       # 어제 다른 머신에서 push한 메모리 받기
[작업 중] (Claude Code가 auto-memory에 새 파일을 쓰면서 진행)
[저녁] /memory-push       # 오늘 작업분 원격에 commit & push
```

또는 한 번에:

```
[작업 시작 전] /memory-sync
[작업 종료 후] /memory-sync
```

## 충돌 시 동작

- `/memory-pull`: `git fetch + reset --hard origin/<branch>` 으로 캐시를 강제 동기화한다. 캐시 단계에서는 충돌이 발생하지 않는다 (캐시는 sync 도구만 쓰는 영역이므로 항상 origin이 truth).
- `/memory-push`: push 전에 `git rebase origin/<branch>` 가 먼저 돈다. 캐시에 다른 머신의 sync 커밋이 같은 파일을 건드린 적이 있고 로컬에서도 같은 파일을 바꿨다면 rebase가 깨진다. 이 경우 커맨드는 **중단**되고 사용자가 `~/.claude-memory-sync/cache`에서 직접 해결해야 한다 (`git status` 후 conflict 해결 → `git rebase --continue`).
- `/memory-sync`: pull이 로컬 memory 디렉토리를 덮어쓰므로, 그 사이 로컬에서만 추가된 새 파일은 살아남지만 **수정**된 파일은 손실될 수 있다. 안전하게 가려면 `/memory-status`로 먼저 diff를 확인하거나, `/memory-push` 단방향만 사용.

## 보안 / 위생

- repo는 **private 필수**. Claude의 auto-memory에는 작업 맥락(내부 시스템명, 계정, 티켓 번호 등)이 그대로 박혀 있다.
- 토큰을 쓰는 HTTPS 인증을 권장 (GitLab Personal Access Token + 1Password / 키체인). SSH 키도 가능.
- `~/.claude-memory-sync/cache`는 매번 clone하지 않는다 (영구 캐시). 디스크 점유는 보통 수 MB 수준.
- `~/.claude-memory-sync/config.json`은 머신 로컬 설정이므로 git에 들어가서는 안 된다 — repo의 `.gitignore`에 명시할 필요는 없으나, 실수로 commit하지 않도록 주의.

## 트러블슈팅

| 증상 | 원인 / 해결 |
|---|---|
| `/memory-pull` 후 `MEMORY.md`가 비어 있음 | 원격 repo에 `projects/<logical_key>/` 또는 `global/` 폴더가 아직 없음. 작업하면서 메모리가 쌓이면 `/memory-push`로 첫 seed를 만든다. |
| `git push` 시 403 / Authentication failed | GitLab Personal Access Token이나 SSH 키가 등록되지 않음. `git -C ~/.claude-memory-sync/cache push` 를 직접 실행해 인증 다이얼로그 확인. |
| 머신 간 같은 프로젝트인데 다른 폴더로 sync됨 | basename이 달랐거나 slug가 다른데 매핑을 안 했음. `config.json`의 `projects`에 logical_key 매핑 추가. |
| `/memory-push`가 rebase에서 멈춤 | 두 머신이 같은 파일을 수정 → 수동 해결 필요. `~/.claude-memory-sync/cache`에서 직접 conflict 해소 후 `git rebase --continue && git push`. |

## 파일 구조

```
labiter-memory-sync-plugin/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── memory-pull.md
│   ├── memory-push.md
│   ├── memory-sync.md
│   └── memory-status.md
├── examples/
│   └── config.example.json
└── README.md
```
