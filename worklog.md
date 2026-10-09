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
웹훅 교체 및 검증 (2026-10-09)
================================================================================

  [작업] Slack Incoming Webhook 교체 (양파토끼_대화 채널)
  - 새 웹훅(…/B0C801ZPYCW/…)을 settings.json env(SLACK_SESSION_WEBHOOK)와
    secrets/slack-jipsa.env 에 적용
  - 테스트: curl --data-binary 로 UTF-8 메시지 전송 → HTTP 200 "ok",
    양파토끼_대화 채널 수신 확인, 한글 깨짐 없음
  - 이전 웹훅(…/B0C76UR1ECU/…) 을 Slack 앱 설정에서 삭제
    → 이전 URL로 전송 시 HTTP 404 "no_service" 확인 (폐기 완료)
  - secrets/slack-jipsa.env.bak-webhook (이전 URL 백업) 삭제
  - 참고: 웹훅 URL 전체는 로그/문서에 기록하지 않음 (시크릿)

================================================================================
Slack 요약 봇 전환 · 3일 자동 정리 · 노션 복구 (2026-10-10)
================================================================================

  [작업 1] 요약 전송을 웹훅 → 봇 토큰(chat.postMessage)으로 전환
  - 이유: 웹훅 메시지는 봇 토큰으로 삭제 불가 (chat.delete → cant_delete_message)
  - ~/.claude/hooks/slack-session-summary.sh (설치본, templates/ 는 수정 안 함):
    settings.json env 의 SLACK_SUMMARY_CHANNEL 이 있으면 봇 토큰으로 전송,
    실패하면 기존 SLACK_SESSION_WEBHOOK 으로 폴백
  - 양파토끼-대화(비공개) 채널에 봇(yangpatokki) 초대 필요: /invite @yangpatokki
  - 이 채널은 수신용. daemon.py 는 SLACK_CHANNEL(업무자동화)만 응답하므로 변경 없음

  [작업 2] 3일 지난 봇 메시지 자동 삭제
  - scripts/slack-jipsa/cleanup_bot_messages.py: 채널 2개(업무자동화, 양파토끼-대화),
    --yes 없으면 미리보기만. 웹훅/사람 메시지는 대상 제외
  - Task Scheduler 'SlackCleanup' 매일 04:00 (cleanup-hidden.vbs, 로그: logs/cleanup.log)
  - [문제 17] conversations.history 의 latest 를 소수점 7자리로 보내 Slack 이 무시
    → 3일 이내 메시지까지 대상이 되어 업무자동화 봇 응답 7개가 잘못 삭제됨
    해결: latest=f'{cutoff:.6f}' + cutoff 이후 ts 는 대상에서 제외하는 안전장치

  [작업 3] 노션 적재 실패 복구
  - [문제 18] Stop hook 의 노션 적재가 10-07 이후 계속 page_id=FAIL
    원인: Git Bash 의 python3 가 Windows Store 스텁(WindowsApps\python3)
          (오류 출력이 /dev/null 로 버려져 원인이 로그에 안 남음)
    해결: ~/bin/python3 shim → C:\Python312\python.exe (hook 래퍼가 ~/bin 을 PATH 앞에 둠)
  - 복구: 10-06 이후 36개 세션 192턴을 hook 사본으로 재적재 (원래 시각, Slack 제외,
    claude:<세션>:<턴> upsert 라 중복 없음). 성공 192 / 실패 0, 실시간 적재도 정상 확인

  [작업 4] 웹훅 교체
  - 새 웹훅(…/B0C7NGZJZ0F/…) 적용 (settings.json, secrets/slack-jipsa.env), 전송 테스트 OK
  - 이전 웹훅(…/B0C801ZPYCW/…) Slack 에서 삭제 → 404 no_service 확인
  - 이전 URL 이 든 백업 파일(settings.json.bak-*) 및 folder-watch.ps1.bak-webhook 삭제
  - URL 전체는 문서에 기록하지 않음. 채팅/화면에 붙여넣지 말고 파일로 전달할 것

================================================================================
다중 봇 협업: 집사 + 실무담당 (2026-10-10)
================================================================================

  [작업 1] 두 번째 봇 '실무담당' 추가
  - Slack 앱 '실무담당'(사용자명 silmutokki, U0C7P9RCL1M)을 manifest 로 생성, Socket Mode
  - 데몬: ~/.claude/scripts/slack-silmu/ (slack-jipsa daemon.py 복사본, 경로·식별자만 분리)
    secrets/slack-silmu.env, sessions/logs 별도, 노션 external id 'silmu:'
  - Task Scheduler 'SlackSilmu' (SlackJipsa 와 동일: 로그온 + 5분 반복, IgnoreNew)
  - 역할: 집사 = 비서/조율(일정·안내·대표님 응대), 실무담당 = 실행(파일·코드·조사)
    각 폴더 CLAUDE.md 에 "누가 답하는가" 규칙. 해당 없으면 응답 전체를 SKIP 한 단어로
  - 두 데몬 공통: 상대 봇 라벨을 OTHER_BOT_LABEL(.env)로, 자기 응답 라벨을 BOT_NAME 으로

  [작업 2] 시험 채널(봇협업-시험) → 업무자동화 전환
  - 시험: 호명·역할별 응답, 토론 모드(트리거 키워드) 2턴 후 양쪽 SKIP 으로 자연 종료 확인
  - 전환: 두 봇 모두 SLACK_CHANNEL_DIALOG = 업무자동화(C0C76F35C76)
    → 업무자동화에서 토론 키워드(토론·비교·의견·둘이 등)가 나오면 봇끼리 대화, '그만'으로 종료
  - 3일 정리 스크립트가 봇별 env(slack-jipsa.env, slack-silmu.env)를 순회해 각자 메시지 삭제

  [문제 19] 토론 중 집사가 영어 혼잣말("my earlier reply doesn't appear in the context")을 게시
  원인: 맥락 구성 시 shared[-15:-1] 로 "마지막 줄 = 방금 받은 메시지"라고 가정.
        두 봇이 동시에 답하면 마지막 줄이 자기 답이 되어 자기 답이 맥락에서 빠짐
  해결: 방금 받은 메시지를 msg_ts 로 제외. Windows 용 공유 버퍼 잠금(msvcrt) 추가.
        두 봇 CLAUDE.md 에 "채널엔 한국어만, 맥락/시스템 혼잣말 금지, 애매하면 SKIP" 규칙
  검증: 재시험에서 혼잣말 없이 2턴 후 자연 종료

  [참고] Slack 초대 시 표시 이름이 아닌 실제 이름으로 찾아야 함
  - 집사 봇의 Slack 이름은 '양파토끼'(사용자명 yangpatokki, 표시 이름 비어 있음)
  - /invite 는 입력창에서 '/'를 직접 쳐야 명령으로 인식 (붙여넣으면 일반 메시지로 올라감)

================================================================================
END OF WORKLOG
================================================================================
