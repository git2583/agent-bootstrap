# CLAUDE.md — Agent Bootstrap 프로젝트 지침

## 프로젝트 개요

Claude Code에게 던지면 AI가 직접 1:1로 안내하면서 사용자 컴퓨터에 자동화 에이전트를 설치해주는 셋업 키트.
사용자는 토큰 복붙과 클릭만 하면 되고, 터미널 명령·파일 생성·권한 설정은 모두 Claude가 처리한다.

---

## 시작 방법

이 폴더 안에서 Claude Code를 열고 아래 한 줄을 받으면 셋업이 시작된다:

```
이 폴더의 SKILL.md를 읽고 셋업을 시작해줘.
```

`SKILL.md`가 전체 진행 로직을 담고 있다. 반드시 먼저 읽고 시작할 것.

---

## 폴더 구조

```
agent-bootstrap/
├── CLAUDE.md               ← 지금 이 파일 (Claude Code 지침)
├── SKILL.md                ← AI 가이드 본체 (셋업 로직 전체)
├── README.md               ← 사용자용 소개
├── .env.example            ← 환경변수 템플릿
├── worklog.md              ← 설치 작업 이력
├── modules/                ← 모듈별 단계 안내 (AI가 읽음)
│   ├── 01-slack-bridge.md
│   ├── 02-folder-trigger.md
│   ├── 03-bridge-trigger.md
│   └── 04-notion-archive.md
└── templates/
    ├── lib/                ← 검증된 라이브러리 (수정 금지)
    │   ├── notion.py
    │   ├── slack_mrkdwn.py
    │   └── md_to_notion.py
    ├── hooks/              ← Claude Code Stop hook (수정 금지)
    │   ├── slack-session-summary.sh
    │   └── append_turn_raw.py
    ├── scripts/slack-jipsa/
    │   └── daemon.py       ← Slack Socket Mode daemon (수정 금지)
    ├── launchd-daemon.plist.tmpl
    ├── launchd-folder-watch.plist.tmpl
    └── systemd-*.tmpl
```

---

## 핵심 규칙

### 1. templates/ 코드는 절대 수정하지 말 것

`templates/lib/`, `templates/hooks/`, `templates/scripts/` 안의 파일은
운영 환경에서 검증된 코드다. 사용자 환경 결합은 전부 `.env` 변수로 처리된다.

- **그대로 복사**: `notion.py`, `slack_mrkdwn.py`, `md_to_notion.py`, `daemon.py`, `slack-session-summary.sh`, `append_turn_raw.py`
- **변수 치환 후 사용**: `*.tmpl` 파일 (`{USERNAME}`, `{HOME}`, `{WATCH_FOLDER}` 등)
- **AI가 직접 생성**: Windows PowerShell·Task Scheduler 코드, Linux systemd 코드

### 2. 시크릿 안전

- 모든 토큰은 `~/.claude/secrets/` 에 저장
- GitHub·로그에 절대 노출 금지
- Windows: `%USERPROFILE%\.claude\secrets\` (ACL 본인만 읽기)
- macOS/Linux: `chmod 600`

### 3. 한 단계씩 진행

사용자에게 여러 단계를 한꺼번에 던지지 말 것. 각 단계마다 "됐어요?" 확인 후 다음 단계로.

### 4. 에러는 AI가 진단

사용자가 에러 메시지를 붙여넣으면 AI가 직접 원인을 분석하고 해결한다.
사용자에게 에러를 해석하게 만들지 말 것.

---

## OS별 처리 방식

| 구분 | macOS | Windows | Linux |
|------|-------|---------|-------|
| 자동 시작 | launchd (.plist) | Task Scheduler | systemd user |
| 폴더 감지 | launchd WatchPaths | PowerShell 5초 폴링 | systemd .path unit |
| 시크릿 폴더 | `~/.claude/secrets/` chmod 600 | `%USERPROFILE%\.claude\secrets\` icacls | `~/.claude/secrets/` chmod 600 |
| Python | `/usr/bin/python3` | `C:\PythonXXX\python.exe` | `/usr/bin/python3` |
| Shell | bash/zsh | PowerShell | bash |

### Windows 특이사항 (이 설치 환경: Windows 10, 사용자 a)

- PowerShell 실행 정책: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`
- claude.exe 전체 경로: `C:\Users\a\AppData\Roaming\npm\node_modules\@anthropic-ai\claude-code\bin\claude.exe`
- Python 경로: `C:\Python312\python.exe`
- Task Scheduler로 SlackJipsa, FolderWatch 등록
- Git Bash curl 한국어 인코딩: `~/bin/curl` 래퍼 사용 (`--data-binary @tmpfile`)
- jq: `~/bin/jq` (winget 설치 후 복사)
- claude 비대화형 실행 시 `--dangerously-skip-permissions` 필요

---

## 현재 설치 상태 (2026-10-07 기준)

| 모듈 | 상태 | 비고 |
|------|------|------|
| Module 1: Slack ↔ Claude Code | ✅ 완료 | SlackJipsa Task Scheduler 실행 중 |
| Module 2: 폴더 트리거 | ✅ 완료 | FolderWatch Task Scheduler 실행 중 |
| Module 3: Slack + 폴더 합치기 | ✅ 완료 | 시작·완료 알림 + 요약 본문 Slack 전송 |
| Module 4: 노션 자동 적재 | ✅ 완료 | Notion DB 생성 완료, Stop hook 연동 |

**설치된 파일 위치:**
- 시크릿: `C:\Users\a\.claude\secrets\slack-jipsa.env`
- 설정: `C:\Users\a\.claude\settings.json`
- Stop hook: `C:\Users\a\.claude\hooks\slack-hook-wrapper.sh`
- Daemon: `C:\Users\a\.claude\scripts\slack-jipsa\daemon.py`
- 폴더 감시: `C:\Users\a\.claude\scripts\folder-watch\folder-watch.ps1`
- 감시 폴더: `C:\Users\a\Documents\claude-inbox\`

**Notion DB:**
- 턴 로그 DB ID: `3f1c4d9a-5477-81a4-88d2-c709100a6e0c`
- 일일 통합 DB ID: `3f1c4d9a-5477-8166-98e6-d4ec24c3ee7c`
- 부모 페이지 ID: `3f1c4d9a-5477-803f-8c0c-d7c83f06010e`

---

## 자주 발생하는 문제

| 증상 | 원인 | 해결 |
|------|------|------|
| Slack 메시지 한글 깨짐 | Git Bash curl 인코딩 | `~/bin/curl` 래퍼 확인 |
| Stop hook 미발동 | settings.json hooks 형식 오류 | matcher + hooks 배열 구조 확인 |
| claude.exe not found | Task Scheduler PATH 미인식 | 전체 경로 하드코딩 |
| 파일 읽기 권한 차단 | 비대화형 실행 시 권한 프롬프트 | `--dangerously-skip-permissions` 추가 |
| SLACK_BOT_TOKEN 0chars | Task Scheduler .env 파싱 실패 | 스크립트 상단 직접 하드코딩 |
| chat.postMessage channel_not_found | Bot Token 채널 미가입 | Incoming Webhook URL로 전환 |
| 폴더워치 한글 출력 깨짐 | PowerShell 파이프 CP949 해석 | ProcessStartInfo + StandardOutputEncoding=UTF8 |
| Notion row 미생성 | NOTION_SESSION_DB 미설정 | settings.json env 섹션 확인 |

---

## 로그 위치

- 폴더 감시 로그: `C:\Users\a\.claude\scripts\folder-watch\logs\YYYY-MM-DD.log`
- Stop hook 로그: `/tmp/slack-session-summary.log`
- Daemon 로그: `C:\Users\a\.claude\scripts\slack-jipsa\logs\`
