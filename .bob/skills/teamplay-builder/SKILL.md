---
name: teamplay-builder
description: "팀플 해결사" 해커톤 프로젝트의 Python 도구(find_common_slots, backward_schedule)를 만들고 테스트하고, watsonx Orchestrate에 연결·등록하고, 에이전트(skr_schedule_agent, skr_supervisor)를 만들 때 사용한다. 이 폴더에서 도구, 테스트, Orchestrate 연결, 에이전트 YAML, 등록 요청이 오면 항상 이 스킬을 따른다.
---

# teamplay-builder

팀원 가능 시간과 회의 기록을 받아 팀플을 돕는 watsonx Orchestrate 멀티 에이전트를 만든다.

## 이름 규칙
- 다른 팀과 이름이 겹치지 않게 에이전트 이름에는 팀 접두어 `skr_`를 붙인다: skr_schedule_agent, skr_meeting_agent, skr_supervisor (예비: skr_schedule_backup).
- 도구 이름(find_common_slots, backward_schedule)은 그대로 둔다. 같은 이름의 도구가 이미 있다고 나오면 앞에 `skr_`를 붙이고 에이전트 YAML과 instructions의 도구 이름도 같이 바꾼다.

## 파일 위치
- 샘플 데이터: `data/availability.json`(JSON), `data/availability.txt`(사람이 치는 글 형식), `data/strengths.json`, `data/meeting_1.txt`, `data/meeting_2.txt`, `data/kakao_chat.txt`
- 에이전트 문구(description, instructions): `agents/schedule_agent.md`, `agents/meeting_agent.md`, `agents/supervisor.md`, `agents/schedule_backup.md` — 문구를 바꾸지 말고 그대로 쓴다
- 정답표: `answers.md`, 데모 입력: `demo_inputs.txt`
- 만들 코드: `tools/` (도구), `tests/` (테스트), `agents/*.yaml` (에이전트)
- 비밀 값: `.env` (저장소에 올리지 않음, 형식은 `.env.example`)

## 환경 (Windows 노트북)
- ADK(`ibm-watsonx-orchestrate`)는 Python 3.11 이상 3.15 미만에서만 설치된다. 먼저 `py -0`으로 설치된 버전을 확인하고, 3.13이나 3.14로 가상환경을 만든다: `py -3.13 -m venv .venv`. 3.15 이상뿐이면 사용자에게 3.13 설치를 요청한다.
- `python`을 입력했는데 Microsoft Store가 열리면 진짜 Python이 아니다. `py` 런처를 쓴다.
- `orchestrate env` 명령이 API 키를 화면에서 물어보는(대화형) 경우 Bob의 명령 실행이 멈춘 것처럼 보일 수 있다. --help에 API 키를 인자로 넘기는 방법이 있으면 `.env` 값으로 그걸 쓰고, 없으면 사용자에게 "이 명령은 직접 터미널에서 실행해 달라"고 정확한 명령을 알려준다.
- 와이파이가 느려 pip 설치가 오래 걸릴 수 있다. 설치는 한 번만 하고 다시 하지 않는다.

## 도구 만드는 방식 (중요)
- **Python 도구로 만든다.** 로컬 MCP 서버는 클라우드 Orchestrate에서 접속할 수 없을 수 있다. 해커톤 실습이 로컬(개발자용) Orchestrate 환경을 쓰고 MCP 방식을 안내한 경우에만 MCP를 쓴다.
- `tools/find_common_slots.py`, `tools/backward_schedule.py` 파일 하나에 도구 하나.
- `from ibm_watsonx_orchestrate.agent_builder.tools import tool` 후 함수에 `@tool`을 붙인다.
- 모든 인자에 타입 힌트(str, int)를 달고, 함수에 한국어 docstring을 쓴다: 무엇을 하는지 한 줄 + Args(각 인자 설명과 예시) + Returns. docstring이 없으면 에이전트가 도구를 언제 쓸지 모른다.
- 계산은 표준 라이브러리만 쓴다. 반환값은 list/dict (JSON으로 바꿀 수 있게).
- 테스트는 `@tool` 없이도 돌아가게, 계산 함수를 따로 두고 `@tool` 함수가 그것을 부르는 구조로 만든다.

## 도구1: find_common_slots
- 입력: availability (str), min_minutes (int, 기본 60), top_n (int, 기본 3)
- availability는 두 형식을 모두 받는다. (1) `data/availability.json` 형식의 JSON 문자열 (2) `data/availability.txt` 형식의 글 ("가은: 월 13~18시, 화 10~12시" 한 줄에 한 명, "13:30~15시"처럼 분이 있어도 해석). 에이전트가 어느 형식으로 넣어도 돌아가게 한다.
- 요일별로 모든 팀원의 가능 시간이 겹치는 구간을 찾는다. min_minutes보다 짧으면 제외.
- 긴 순서, 같으면 요일 순서(월, 화, 수, 목, 금).
- 출력: [{"요일", "시작", "끝", "분"}]. 겹치는 구간이 없으면 빈 목록과 함께 가장 많은 인원이 가능한 시간.
- 해석할 수 없으면 에러 대신 "가능 시간 형식을 확인해 주세요 (예: 가은: 월 13~18시)" 메시지를 돌려준다.
- 정답 (두 형식 모두): 1위 수 18:00-20:00(120분), 2위 월 16:00-17:00(60분), 3위 목 20:00-21:00(60분). 테스트는 두 형식으로 각각 만든다.

## 도구2: backward_schedule
- 입력: start_date (str), deadline (str), 둘 다 YYYY-MM-DD
- 발표 전날 = "발표 리허설"(1일)
- 시작일~리허설 전날 일수 N. "자료조사" = round(N*0.4), "발표 초안" = round(N*0.25), "PPT 제작" = 나머지. 이 순서로 날짜 배정.
- 출력: [{"작업", "시작일", "종료일", "일수"}]
- 정답 (2026-11-27 → 2026-12-10): 자료조사 11/27~12/1(5일), 발표 초안 12/2~12/4(3일), PPT 제작 12/5~12/8(4일), 발표 리허설 12/9(1일)

## 진행 순서
1. Plan: 위 스펙으로 계획을 세우고 사용자 승인을 받는다. 계획은 짧게(파일·함수·테스트 목록 위주).
2. Agent: 도구와 테스트를 만들고, 테스트가 통과할 때까지 고친 뒤 결과를 표로 보여준다.
3. 연결: 위 "환경" 규칙대로 가상환경을 만들고 ADK를 설치한 뒤, `.env`의 주소와 API 키로 `orchestrate env` 명령을 써서 해커톤 Orchestrate 환경에 연결하고 활성화한다. 명령 사용법은 --help로 먼저 확인. PowerShell에서 가상환경 활성화가 막히면 원인을 설명하고 현재 사용자 범위에서만 해결한다.
4. 등록: `orchestrate tools import -k python -f tools/<파일>.py` (필요하면 -r requirements.txt). 등록 후 `orchestrate tools list`로 확인.
5. 에이전트: `agents/schedule_agent.md` 문구로 `agents/skr_schedule_agent.yaml`(spec_version v1, kind native)을 만들고 `orchestrate agents import`로 등록. llm은 사용자가 알려준 모델.
6. 총괄: 먼저 `orchestrate agents list`로 skr_meeting_agent의 실제 내부 이름을 확인한 뒤, `agents/supervisor.md` 문구로 `agents/skr_supervisor.yaml`(collaborators: skr_schedule_agent, <skr_meeting_agent의 실제 이름>)을 만들고 등록.
7. 배포: 필요하면 세 에이전트를 배포(deploy)해서 채팅 화면에서 쓸 수 있게 한다.
8. 각 단계가 끝나면 무엇을 만들었는지 3줄로 요약하고, git commit 후 push한다 (.env 제외).
