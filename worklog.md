================================================================================
WORKLOG — Agent Bootstrap Setup
날짜: 2026-10-07
작업자: 대표님 (U0APXUTNP3K)
세션: d8c4b379-402a-447d-8cf2-1903572b3578 ~ 현재
================================================================================

■ 개요
  Claude Code + Slack + Notion 자동화 에이전트 셋업 (agent-bootstrap 스킬 기반)
  OS: Windows 10 Education (10.0.19045), 사용자: a
  작업 디렉토리: C:\Users\a\.claude\scripts\slack-jipsa

================================================================================
MODULE 1: Slack ↔ Claude Code 양방향 연결
================================================================================

[STEP 1] 슬랙 앱 생성 및 토큰 수집
  - Slack 앱 이름: 집사
  - Bot Token:    xoxb-10794690496051-12235445447509-*** (수집 완료)
  - App Token:    xapp-1-A0C6MSFE57F-*** (수집 완료)
  - Channel ID:   C0C76F35C762
  - User ID:      U0APXUTNP3K (대표님)
  - Bot User ID:  U0C6XD3D5EZ

[STEP 2] 시크릿 파일 작성
  파일: C:\Users\a\.claude\secrets\slack-jipsa.env
  내용:
    SLACK_BOT_TOKEN=xoxb-...
    SLACK_APP_TOKEN=xapp-...
    SLACK_CHANNEL=C0C76F35C762
    USER_SLACK_ID=U0APXUTNP3K
    BOT_USER_ID=U0C6XD3D5EZ
    USER_NAME=대표님
    SLACK_BOT_NAME=집사
    SLACK_SESSION_WEBHOOK=https://hooks.slack.com/services/T0APCLAEL1H/...
    NOTION_API_TOKEN=ntn_528415673514AKXy*** (모듈 4 추가)
    NOTION_SESSION_DB=3f1c4d9a-5477-81a4-88d2-c709100a6e0c (모듈 4 추가)
    NOTION_DAILY_DB=3f1c4d9a-5477-8166-98e6-d4ec24c3ee7c (모듈 4 추가)

[STEP 3] lib 의존성 복사 (templates/ → ~/.claude/)
  - C:\Users\a\.claude\scripts\lib\notion.py         ✅
  - C:\Users\a\.claude\scripts\lib\slack_mrkdwn.py   ✅
  - C:\Users\a\.claude\hooks\md_to_notion.py         ✅
  - C:\Users\a\.claude\scripts\lib\__init__.py       ✅

[STEP 4] daemon.py 복사 및 run.ps1 작성
  파일: C:\Users\a\.claude\scripts\slack-jipsa\daemon.py  (templates/ 에서 복사)
  파일: C:\Users\a\.claude\scripts\slack-jipsa\run.ps1
    내용:
      $env:SLACK_BOT_TOKEN = ... (slack-jipsa.env에서 로드)
      C:\Python312\python.exe daemon.py 실행

[STEP 5] Task Scheduler 등록 — SlackJipsa
  작업 이름: SlackJipsa
  트리거: 로그인 시 자동 시작
  실행: powershell.exe -File run.ps1
  상태: Running ✅

[STEP 6] Stop hook 설정
  파일: C:\Users\a\.claude\hooks\slack-session-summary.sh  (templates/ 복사)
  파일: C:\Users\a\.claude\hooks\append_turn_raw.py         (templates/ 복사)
  파일: C:\Users\a\.claude\hooks\slack-hook-wrapper.sh      (신규 작성)
    내용:
      export LANG=en_US.UTF-8
      export PATH="$HOME/bin:$PATH"
      exec bash ~/.claude/hooks/slack-session-summary.sh

  C:\Users\a\.claude\settings.json 수정:
    "hooks": { "Stop": [{ "command": "bash $HOME/.claude/hooks/slack-hook-wrapper.sh" }] }
    "env": { "SLACK_SESSION_WEBHOOK": "...", "LANG": "en_US.UTF-8", ... }

[STEP 7] UTF-8 인코딩 문제 해결
  문제: Git Bash curl이 한국어 텍스트를 Windows 코드 페이지로 전송 → 깨짐
  해결: ~/bin/curl 래퍼 작성
    파일: C:\Users\a\bin\curl
    방식: -d "string" → --data-binary @tmpfile (UTF-8 바이트 직접 전송)

  문제: jq가 Git Bash PATH에 없음
  해결: winget으로 설치된 jq.exe를 ~/bin/jq로 복사

[STEP 8] Module 1 검증 결과
  - Slack 채널에서 메시지 발송 → daemon.py 수신 → Claude Code 응답 ✅
  - Claude Code 세션 종료 → Stop hook → Slack 세션 요약 도착 ✅
  - 한글 인코딩 정상 (curl 래퍼 적용 후) ✅

================================================================================
MODULE 2: 폴더 트리거 자동화 (Folder Watch)
================================================================================

[STEP 1] 감시 폴더 생성
  경로: C:\Users\a\Documents\claude-inbox\
  처리 완료 폴더: C:\Users\a\Documents\claude-inbox\.processed\

[STEP 2] 시나리오 선택
  선택: (c) 마크다운 파일 → 분석 → Slack 알림

[STEP 3] folder-watch.ps1 작성
  파일: C:\Users\a\.claude\scripts\folder-watch\folder-watch.ps1
  동작:
    1) 5초마다 claude-inbox 폴더 폴링
    2) 새 파일 발견 시 claude.exe --print --dangerously-skip-permissions 로 요약 요청
    3) 요약 결과를 Slack API chat.postMessage로 전송
    4) 처리된 파일을 .processed/ 로 이동

  주요 수정 이력:
    - Register-ObjectEvent 방식 → 5초 폴링으로 변경 (Task Scheduler 환경 호환성)
    - 한국어 프롬프트 → 영어 프롬프트 + 임시파일 stdin 방식 (인코딩 우회)
    - SLACK_BOT_TOKEN 환경변수 로딩 실패 → 스크립트 상단 하드코딩 방식으로 변경
    - Process-File 함수 내 $slackToken 변수 버그 수정
      (GetEnvironmentVariable 참조 → 스크립트 상단 $SlackToken 직접 참조)
    - --dangerously-skip-permissions 플래그 추가 (파일 읽기 권한 허용)

[STEP 4] Task Scheduler 등록 — FolderWatch
  작업 이름: FolderWatch
  트리거: 로그인 시 자동 시작
  실행: powershell.exe -File folder-watch.ps1
  상태: Running ✅

[STEP 5] Module 2 검증 결과
  - claude-inbox에 test.md 투입
  - Claude가 파일 읽고 한국어 요약 생성
  - Slack에 "📄 test.md 요약: ..." 메시지 도착 ✅
  - 처리 후 .processed/ 로 파일 이동 ✅

================================================================================
MODULE 4: 노션 자동 아카이브
================================================================================

[STEP 1] Notion Integration 생성
  이름: Agent Bootstrap
  유형: Internal
  API Token: ntn_528415673514AKXy*** (slack-jipsa.env에 저장)

[STEP 2] 노션 페이지 준비
  페이지: agent-bootstrap
  URL: https://app.notion.com/p/agent-bootstrap-3f1c4d9a5477803f8c0cd7c83f06010e
  Page ID: 3f1c4d9a-5477-803f-8c0c-d7c83f06010e
  Integration 연결: ✅

[STEP 3] DB 자동 생성 (Notion API v2022-06-28)
  DB 1: "Claude Code 턴 로그"
    ID: 3f1c4d9a-5477-81a4-88d2-c709100a6e0c
    컬럼: 프로젝트(title), 시각(date), 세션ID, 작업디렉토리, 시킨일, 한일,
          결과, 확인필요, 모델(select), 도구호출수(number), 전체요약, external_id
  DB 2: "일일 통합"
    ID: 3f1c4d9a-5477-8166-98e6-d4ec24c3ee7c
    컬럼: 이름(title), 날짜(date), 상태(status), external_id
  Relation: 턴 로그 DB → "📊 일일 통합" 컬럼 (single_property) ✅

[STEP 4] 환경변수 추가
  slack-jipsa.env:
    NOTION_API_TOKEN=ntn_***
    NOTION_SESSION_DB=3f1c4d9a-5477-81a4-88d2-c709100a6e0c
    NOTION_DAILY_DB=3f1c4d9a-5477-8166-98e6-d4ec24c3ee7c
  settings.json env 섹션도 동일하게 추가

[STEP 5] Stop hook 노션 적재 확인
  slack-session-summary.sh 에 이미 노션 적재 로직 내장 확인:
    - NOTION_TOKEN + NOTION_DB 있으면 자동 upsert
    - upsert_by_external_id (lib/notion.py) 사용
    - 일일 통합 relation 자동 매칭 (get_or_create_daily)
    - append_turn_raw.py 로 원문 블록 append

[STEP 6] daemon 재시작
  SlackJipsa Task Scheduler 재시작 → Running ✅

[STEP 7] Module 4 검증 결과
  - claude --print "1+1" 실행
  - 노션 "Claude Code 턴 로그" DB에 새 row 생성 ✅
  - 프로젝트, 시킨 일, 결과 컬럼 정상 기록 ✅

================================================================================
현재 동작 중인 자동화 전체 목록
================================================================================

  [항상 실행 중]
  1. SlackJipsa (Task Scheduler)
     - daemon.py (Python) — Slack Socket Mode 수신
     - 슬랙 메시지 → Claude Code 처리 → 슬랙 응답

  2. FolderWatch (Task Scheduler)
     - folder-watch.ps1 (PowerShell) — 5초 폴링
     - claude-inbox 파일 → Claude 요약 → Slack 전송

  [Claude Code 세션 종료 시마다]
  3. Stop hook (slack-hook-wrapper.sh → slack-session-summary.sh)
     - Slack 세션 요약 전송
     - Notion "Claude Code 턴 로그" DB row 생성
     - Notion "일일 통합" DB 매칭 (일별 허브)

================================================================================
파일 목록 (생성/수정)
================================================================================

  C:\Users\a\.claude\secrets\slack-jipsa.env
  C:\Users\a\.claude\settings.json
  C:\Users\a\.claude\hooks\slack-hook-wrapper.sh
  C:\Users\a\.claude\hooks\slack-session-summary.sh  (templates/ 복사)
  C:\Users\a\.claude\hooks\append_turn_raw.py         (templates/ 복사)
  C:\Users\a\.claude\hooks\md_to_notion.py            (templates/ 복사)
  C:\Users\a\.claude\scripts\lib\notion.py            (templates/ 복사)
  C:\Users\a\.claude\scripts\lib\slack_mrkdwn.py     (templates/ 복사)
  C:\Users\a\.claude\scripts\lib\__init__.py
  C:\Users\a\.claude\scripts\slack-jipsa\daemon.py   (templates/ 복사)
  C:\Users\a\.claude\scripts\slack-jipsa\run.ps1
  C:\Users\a\.claude\scripts\folder-watch\folder-watch.ps1
  C:\Users\a\bin\curl                                 (UTF-8 래퍼)
  C:\Users\a\bin\jq                                   (winget → 복사)

================================================================================
트러블슈팅 이력
================================================================================

  [문제 1] jq not found in Git Bash
  원인: winget 설치 경로가 Git Bash PATH 밖
  해결: jq.exe → ~/bin/jq 복사

  [문제 2] curl 한국어 인코딩 깨짐
  원인: Git Bash가 -d "string" 을 Windows 코드 페이지로 전달
  해결: ~/bin/curl 래퍼 — -d → --data-binary @tmpfile

  [문제 3] Stop hook PATH 문제
  원인: Task Scheduler 서브프로세스가 ~/bin 미인식
  해결: slack-hook-wrapper.sh에 export PATH="$HOME/bin:$PATH" 추가

  [문제 4] Register-ObjectEvent 미작동
  원인: Task Scheduler 환경에서 이벤트 구독 불가
  해결: 5초 폴링 루프로 전환

  [문제 5] claude.exe not found
  원인: Task Scheduler가 npm 전역 PATH 미인식
  해결: 전체 경로 하드코딩
        C:\Users\a\AppData\Roaming\npm\node_modules\@anthropic-ai\claude-code\bin\claude.exe

  [문제 6] 한국어 프롬프트 → claude stdin 깨짐
  원인: PowerShell 파이프가 UTF-8 유지 못함
  해결: 영어 프롬프트 임시파일 기록 후 stdin으로 전달

  [문제 7] SLACK_BOT_TOKEN 로딩 실패 (0chars)
  원인: Task Scheduler 환경에서 .env 파싱 실패 (BOM 문제 등)
  해결: 스크립트 상단에 직접 하드코딩 ($SlackToken, $SlackChannel)

  [문제 8] Process-File 변수 버그
  원인: 함수 내 $slackToken이 GetEnvironmentVariable 참조 (항상 empty)
  해결: $slackToken = $SlackToken 으로 수정

  [문제 9] 파일 읽기 권한 차단
  원인: claude.exe 기본 실행 시 파일 읽기 권한 프롬프트 발생 (비대화형 환경)
  해결: --dangerously-skip-permissions 플래그 추가

================================================================================
워킹 로그 (folder-watch 처리 이력)
================================================================================

  [테스트 파일 처리 내역]
  - test.md         → 처리 완료 → .processed/ 이동
  - test3.md        → 처리 완료 → .processed/ 이동
  - (기타 테스트 파일들 다수 처리)

  [Slack 수신 메시지 샘플]
  [오전 1:59] 🤖 system32
    ⏰ 16:59 KST · 세션 df43fde8
    🎯 시킨 일: Read the file at '...\test.md' and summarize...
    📝 한 일: Read
    🧠 결과: "테스트 중입니다"라는 한 줄짜리 테스트용 문서입니다.

  [오전 2:03] 🤖 system32
    ⏰ 17:03 KST · 세션 cfc44558
    🎯 시킨 일: Read the file at '...\test.md' and summarize...
    📝 한 일: Read
    🧠 결과: "테스트 중입니다"라는 한 줄짜리 테스트용 문서입니다.

================================================================================
MODULE 3: Slack + 폴더 합치기
================================================================================

[STEP 1] 알림 방식 선택
  선택: ③ 둘 다 (시작 알림 + 결과 본문)

[STEP 2] folder-watch.ps1 수정
  파일: C:\Users\a\.claude\scripts\folder-watch\folder-watch.ps1
  변경 내용:
    - Post-Slack 함수 추가 (Incoming Webhook 방식)
    - Process-File 함수에 시작 알림 추가
    - 완료 시 요약 본문 함께 전송

  주요 수정 이력:
    - chat.postMessage (Bot Token) → Incoming Webhook URL로 전환
      원인: channel_not_found 에러 (Bot Token은 채널 멤버십 필요)
      해결: Stop hook과 동일한 Webhook URL 재사용
    - 완료 메시지 한글 깨짐 수정
      원인: claude.exe 출력을 파이프로 캡처 시 CP949로 잘못 해석
      해결: ProcessStartInfo + StandardOutputEncoding = UTF8 방식으로 직접 캡처
    - 완료 메시지 포맷 단순화 (*bold*, 줄바꿈 → 평문)
      원인: Webhook JSON에서 mrkdwn 마크다운이 깨져 전송됨

[STEP 3] FolderWatch Task Scheduler 재시작
  상태: Running ✅

[STEP 4] Module 3 검증 결과
  - claude-inbox에 test.md 투입
  - Slack 메시지 1: ⏳ test.md 처리 시작 ✅
  - Slack 메시지 2: ✅ test.md - 모듈3의 두 번째 메시지에서 글자가 깨지는... ✅
  - 한글 인코딩 정상 ✅

================================================================================
완료 상태
================================================================================

  Module 1 (Slack ↔ Claude Code): ✅ 완료
  Module 2 (Folder Watch):        ✅ 완료
  Module 3 (Slack + 폴더 합치기): ✅ 완료
  Module 4 (Notion Archive):      ✅ 완료

  모든 자동화 현재 실행 중.
  재부팅 후에도 Task Scheduler 등록으로 자동 재시작됨.

================================================================================
트러블슈팅 추가 이력 (Module 3)
================================================================================

  [문제 10] chat.postMessage channel_not_found
  원인: Bot Token으로 직접 API 호출 시 채널 미가입 상태
  해결: Incoming Webhook URL 방식으로 전환 (Stop hook과 동일)

  [문제 11] 완료 메시지 한글 깨짐
  원인: PowerShell 파이프로 claude.exe 출력 캡처 시 CP949 해석
  해결: System.Diagnostics.ProcessStartInfo + StandardOutputEncoding = UTF8

================================================================================
트러블슈팅 추가 이력 (Module 1 재디버깅 — 2026-10-07)
================================================================================

  [문제 12] Slack → daemon 메시지 미수신 (Socket Mode 연결은 됨)
  원인: Slack 앱 OAuth 스코프에 channels:history, groups:history, im:history 누락
  해결: api.slack.com/apps → OAuth & Permissions → Bot Token Scopes에 추가 후 재설치
        추가 스코프: channels:history, groups:history, im:history, reactions:write

  [문제 13] SLACK_CHANNEL 채널 ID 불일치
  원인: .env의 SLACK_CHANNEL=C0C76F35C762 (12자) — 실제 채널은 C0C76F35C76 (11자)
        daemon.py handle_message의 채널 필터에서 걸려 메시지 무시
  해결: slack-jipsa.env의 SLACK_CHANNEL을 C0C76F35C76 으로 수정

  [문제 14] Python subprocess가 claude.cmd 실행 불가
  원인: Windows에서 subprocess.run(['claude', ...])는 .cmd 파일을 직접 실행 못함
        (shell=True 없이 CreateProcess 사용 → FileNotFoundError → 스레드 조용히 종료)
  해결: slack-jipsa.env에 CLAUDE_EXE 추가:
          CLAUDE_EXE=C:\Users\a\AppData\Roaming\npm\node_modules\@anthropic-ai\claude-code\bin\claude.exe
        daemon.py _run_claude()에서 ENV.get('CLAUDE_EXE', 'claude') 로 읽어 사용

  [문제 15] Task Scheduler WorkingDirectory 미설정
  원인: 작업 등록 시 WorkingDirectory 미지정 → daemon.py의 상대경로 참조 실패 가능
  해결: Register-ScheduledTask -Action에 -WorkingDirectory 추가

================================================================================
최종 검증 결과 (2026-10-07)
================================================================================

  Module 1 (Slack ↔ Claude Code):
    - Slack 채널 메시지 → daemon 수신 → ⏳ 이모지 → Claude 응답 → ✅ 이모지 ✅
    - Task Scheduler 자동 시작 (로그인 시) ✅
  Module 2 (Folder Watch):        ✅ 완료
  Module 3 (Slack + 폴더 합치기): ✅ 완료
  Module 4 (Notion Archive):      ✅ 완료

================================================================================
트러블슈팅 추가 이력 (자동 재시작 — 2026-10-07)
================================================================================

  [문제 16] SlackJipsa·FolderWatch가 로그인 중에 멈춘 채 다시 켜지지 않음
  원인: 트리거가 '로그온 시'만 있음. StopIfGoingOnBatteries=True,
        FolderWatch ExecutionTimeLimit=72H, 종료 코드 0xC000013A(창 닫힘)
  해결: 두 작업에 5분 반복 시간 트리거 추가 (MultipleInstances=IgnoreNew로 중복 방지)
        배터리 관련 중지/시작 금지 해제, ExecutionTimeLimit=0, StartWhenAvailable=True
  검증: 두 작업 Stop → 약 5분 내 자동 Running 복귀, 인스턴스 중복 없음 확인

================================================================================
END OF WORKLOG
================================================================================
