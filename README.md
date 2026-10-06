# Agent Bootstrap

> **AI가 직접 깔아주는 "내 컴퓨터에 사는 에이전트" 셋업 키트**

이 키트를 Claude Code에 던지면, Claude가 직접 1:1로 안내하면서 슬랙 봇·자동화·폴더 트리거를 깔아줍니다. 사용자는 토큰 복붙과 클릭만 하면 됩니다.

---

## 무엇을 깔아주나요?

이 키트는 4개 모듈로 구성됩니다. 필요한 것만 골라서 셋업하면 됩니다.

| 모듈 | 무엇 | 시간 | 난이도 |
|------|------|------|--------|
| **1. 슬랙 ↔ 클로드 코드** | 슬랙 채널에서 클로드 코드와 대화. 메시지 → 즉시 응답. 세션 끝나면 슬랙으로 자동 보고 | 30분 | ★★ |
| **2. 폴더 트리거 자동화** | 특정 폴더에 파일 떨어뜨리면 Claude가 자동 처리 (분석/이동/요약 등) | 20분 | ★ |
| **3. 슬랙 + 폴더 합치기** | 폴더 변화 → 슬랙 알림 + 자동 처리. 위 두 모듈 결합 | 10분 | ★ |
| **4. 노션 자동 적재** (선택) | Claude Code 모든 세션 + 슬랙 대화를 노션 DB에 자동 누적. 검색·회고·아카이브 | 30분 | ★★ |

---

## 실제 설치 사례 (Windows 10, 2026-10-07)

아래는 이 키트로 Module 1·2·4를 실제 셋업한 결과입니다.

### 동작 중인 자동화

| # | 이름 | 방식 | 동작 |
|---|------|------|------|
| 1 | **SlackJipsa** | Task Scheduler + Python daemon | Slack 메시지 → Claude Code 처리 → Slack 응답 |
| 2 | **FolderWatch** | Task Scheduler + PowerShell 5초 폴링 | `claude-inbox/` 파일 투입 → Claude 요약 → Slack 전송 |
| 3 | **Stop hook** | Claude Code hooks.Stop | 세션 종료 → Slack 요약 + Notion DB row 자동 생성 |

### Slack 세션 요약 예시

```
🤖 system32
⏰ 17:03 KST · 세션 cfc44558
🎯 시킨 일: Read the file at '...\test.md' and summarize...
📝 한 일: Read
🧠 결과: "테스트 중입니다"라는 한 줄짜리 테스트용 문서입니다.
```

### Notion 자동 적재 결과

- **Claude Code 턴 로그** DB: 세션마다 프로젝트·시킨일·결과·모델·도구호출수 자동 기록
- **일일 통합** DB: 날짜별 허브 row — 해당 날의 모든 턴 로그를 relation으로 묶음

### Windows 셋업 시 해결한 주요 이슈

| 문제 | 해결 |
|------|------|
| Git Bash curl 한국어 깨짐 | `~/bin/curl` 래퍼 — `-d` → `--data-binary @tmpfile` |
| Task Scheduler에서 claude.exe 미인식 | npm 전역 경로 하드코딩 |
| Stop hook PATH 누락 | `slack-hook-wrapper.sh`에 `export PATH="$HOME/bin:$PATH"` 추가 |
| 비대화형 환경 파일 읽기 차단 | `--dangerously-skip-permissions` 플래그 추가 |
| .env 토큰 파싱 실패 (BOM) | 스크립트 상단 직접 하드코딩으로 우회 |

> 전체 작업 이력은 [worklog.md](./worklog.md) 참고.

---

## 진짜 핵심 — "AI가 깔아준다"는 게 뭔가요?

기존 가이드(PDF·노션 문서):
- 사람이 글을 읽고 따라함
- 막히면 끝. 에러 메시지 받아도 다음 단계 모름

이 키트:
- Claude Code에 폴더를 던지면, Claude가 **읽고** 단계별 안내 시작
- 사용자 환경(맥/윈도우)을 묻고, 환경에 맞는 명령어 생성
- 사용자가 "토큰 받았어" 하면 Claude가 **`.env` 직접 작성**
- launchd plist, stop hook 스크립트 모두 Claude가 **파일 시스템에 직접 만듦**
- 사용자는 클릭·복붙·"됐어" 답변만

---

## 사용법

### 1. 키트 다운로드

```bash
git clone https://github.com/git2583/agent-bootstrap.git
cd agent-bootstrap
```

또는 GitHub에서 **Code → Download ZIP**.

### 2. Claude Code 실행

키트 폴더 안에서:

```bash
claude
```

### 3. 다음 한 줄을 Claude에게 보내기

```
이 폴더의 SKILL.md를 읽고 셋업을 시작해줘.
```

Claude가 자동으로:
- 어떤 모듈부터 깔지 물어봄
- 환경 확인 (맥/윈, Claude 버전, 시스템 정보)
- 단계별 안내 시작
- 파일 생성, 권한 설정, Task Scheduler / launchd 등록 등 직접 처리

---

## 필수 준비물

| 항목 | 비용 | 필요한 모듈 |
|------|------|------------|
| **Claude Code 구독** | 유료 | 모든 모듈 |
| **슬랙 워크스페이스** (개인용 무료) | 무료 | 1, 2, 3 |
| **노션 계정** (무료) | 무료 | 4 |
| **Python 3.x** | 무료 | 1, 4 |
| **OS** | — | macOS / Windows / Linux 모두 지원 |

> 모든 모듈이 macOS·Windows·Linux 모두에서 작동합니다. AI가 사용자 OS를 묻고 자동으로 분기 처리합니다.
>
> - macOS: launchd
> - Windows: Task Scheduler + PowerShell
> - Linux: systemd

---

## 이 키트의 디자인 원칙

1. **AI가 리드한다** — 사용자가 가이드를 읽는 게 아니라 AI가 가이드를 읽고 사용자를 끌고 감
2. **터미널 명령은 AI가 생성** — 사용자는 복붙만
3. **막혔을 때 환경별 분기** — 에러 메시지를 AI에게 보여주면 다음 스텝 안내
4. **시크릿은 로컬에만** — 모든 토큰은 `~/.claude/secrets/` 안에 저장. GitHub에 절대 안 올라감

---

## 폴더 구조

```
agent-bootstrap/
├── README.md              ← 지금 이 파일
├── CLAUDE.md              ← Claude Code 프로젝트 지침
├── SKILL.md               ← AI 가이드 본체 (Claude가 읽음)
├── .env.example           ← 환경변수 템플릿
├── worklog.md             ← 실제 설치 작업 이력
├── modules/               ← 모듈별 단계별 안내 (AI가 읽음)
│   ├── 01-slack-bridge.md
│   ├── 02-folder-trigger.md
│   ├── 03-bridge-trigger.md
│   └── 04-notion-archive.md
└── templates/
    ├── lib/               ← 검증된 코드 (그대로 카피)
    │   ├── notion.py
    │   ├── slack_mrkdwn.py
    │   └── md_to_notion.py
    ├── hooks/             ← Claude Code Stop hook (그대로 카피)
    │   ├── append_turn_raw.py
    │   └── slack-session-summary.sh
    ├── scripts/slack-jipsa/
    │   └── daemon.py      ← 슬랙 ↔ 클코 daemon (그대로 카피)
    ├── launchd-daemon.plist.tmpl       ← macOS daemon 자동 시작
    ├── launchd-folder-watch.plist.tmpl ← macOS 폴더 감지
    └── systemd-*.tmpl                  ← Linux systemd 동등 파일
```

---

## 코드 출처

`templates/lib/`, `templates/hooks/`, `templates/scripts/` 안의 파일은 **운영 환경에서 검증된 코드 그대로**입니다. 사용자 환경에 맞춘 결합은 모두 `.env` 변수로 처리되어 코드 자체는 수정할 필요가 없습니다.

폴더 트리거 watcher (`launchd WatchPaths`, PowerShell 폴링, systemd `.path` unit) 코드는 모듈 2·3 문서 안에 inline으로 들어있습니다. 이 부분은 generation 코드이므로 환경에 따라 AI가 분기 처리합니다.

---

## Windows 사용자 참고

이 키트의 검증 코드는 macOS·Linux 기준입니다 (bash/Python/launchd). Windows에서는 AI가 검증 코드 패턴을 보고 PowerShell · Task Scheduler로 번역하면서 안내합니다. Python 부분(daemon.py, lib/*)은 OS 독립이라 그대로 작동합니다.

**Windows 설치 시 추가로 필요한 것:**
- Git Bash (bash 실행 환경)
- Python 3.x (`winget install Python.Python.3.12`)
- jq (`winget install jqlang.jq`)

---

## 만든 사람

**미시소 — 미리의 시스템 연구소** ([misiso.com](https://misiso.com))

2026-05-14 라이브 "나와 대화하는 에이전트" 무료 자료로 공개.

---

## 라이선스

MIT. 자유롭게 가져다 쓰세요.
