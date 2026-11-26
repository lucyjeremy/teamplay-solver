---
name: teamplay-builder
description: "팀플 해결사" 해커톤 프로젝트의 Python 도구(find_common_slots, backward_schedule)를 만들고 테스트하고, watsonx Orchestrate에 연결·등록하고, 에이전트(schedule_agent, teamplay_supervisor)를 만들 때 사용한다. 이 폴더에서 도구, 테스트, Orchestrate 연결, 에이전트 YAML, 등록 요청이 오면 항상 이 스킬을 따른다.
---

# teamplay-builder

팀원 가능 시간과 회의록을 받아 팀플을 돕는 watsonx Orchestrate 멀티 에이전트를 만든다.

## 파일 위치
- 샘플 데이터: `data/availability.json`, `data/strengths.json`, `data/meeting_1.txt`, `data/meeting_2.txt`
- 에이전트 문구(description, instructions): `agents/schedule_agent.md`, `agents/meeting_agent.md`, `agents/supervisor.md` — 문구를 바꾸지 말고 그대로 쓴다
- 정답표: `answers.md`
- 만들 코드: `tools/` (도구), `tests/` (테스트), `agents/*.yaml` (에이전트)
- 비밀 값: `.env` (저장소에 올리지 않음, 형식은 `.env.example`)

## 도구 만드는 방식 (중요)
- **Python 도구로 만든다.** 로컬 MCP 서버는 클라우드 Orchestrate에서 접속할 수 없을 수 있다. 해커톤 실습이 로컬(개발자용) Orchestrate 환경을 쓰고 MCP 방식을 안내한 경우에만 MCP를 쓴다.
- `tools/find_common_slots.py`, `tools/backward_schedule.py` 파일 하나에 도구 하나.
- `from ibm_watsonx_orchestrate.agent_builder.tools import tool` 후 함수에 `@tool`을 붙인다.
- 모든 인자에 타입 힌트(str, int)를 달고, 함수에 한국어 docstring을 쓴다: 무엇을 하는지 한 줄 + Args(각 인자 설명과 예시) + Returns. docstring이 없으면 에이전트가 도구를 언제 쓸지 모른다.
- 계산은 표준 라이브러리만 쓴다. 반환값은 list/dict (JSON으로 바꿀 수 있게).
- 테스트는 `@tool` 없이도 돌아가게, 계산 함수를 따로 두고 `@tool` 함수가 그것을 부르는 구조로 만든다.

## 도구1: find_common_slots
- 입력: availability_json (str, `data/availability.json` 형식의 JSON 문자열), min_minutes (int, 기본 60), top_n (int, 기본 3)
- 요일별로 모든 팀원의 가능 시간이 겹치는 구간을 찾는다. min_minutes보다 짧으면 제외.
- 긴 순서, 같으면 요일 순서(월, 화, 수, 목, 금).
- 출력: [{"요일", "시작", "끝", "분"}]. 겹치는 구간이 없으면 빈 목록과 함께 가장 많은 인원이 가능한 시간.
- JSON 문자열이 잘못되면 에러 대신 "가능 시간 형식을 확인해 주세요" 메시지를 돌려준다.
- 정답: 1위 수 18:00-20:00(120분), 2위 월 16:00-17:00(60분), 3위 목 20:00-21:00(60분)

## 도구2: backward_schedule
- 입력: start_date (str), deadline (str), 둘 다 YYYY-MM-DD
- 발표 전날 = "발표 리허설"(1일)
- 시작일~리허설 전날 일수 N. "자료조사" = round(N*0.4), "발표 초안" = round(N*0.25), "PPT 제작" = 나머지. 이 순서로 날짜 배정.
- 출력: [{"작업", "시작일", "종료일", "일수"}]
- 정답 (2026-11-27 → 2026-12-10): 자료조사 11/27~12/1(5일), 발표 초안 12/2~12/4(3일), PPT 제작 12/5~12/8(4일), 발표 리허설 12/9(1일)

## 진행 순서
1. Plan: 위 스펙으로 계획을 세우고 사용자 승인을 받는다.
2. Agent: 도구와 테스트를 만들고, 테스트가 통과할 때까지 고친 뒤 결과를 표로 보여준다.
3. 연결: 가상환경에 ADK(`ibm-watsonx-orchestrate`)를 설치하고, `.env`의 주소와 API 키로 `orchestrate env` 명령을 써서 해커톤 Orchestrate 환경에 연결하고 활성화한다. 명령 사용법은 --help로 먼저 확인. Windows PowerShell에서 가상환경 활성화가 막히면 원인을 설명하고 현재 사용자 범위에서만 해결한다.
4. 등록: `orchestrate tools import -k python -f tools/<파일>.py` (필요하면 -r requirements.txt). 등록 후 `orchestrate tools list`로 확인.
5. 에이전트: `agents/schedule_agent.md` 문구로 `agents/schedule_agent.yaml`(spec_version v1, kind native)을 만들고 `orchestrate agents import`로 등록. llm은 사용자가 알려준 모델.
6. 총괄: 먼저 `orchestrate agents list`로 meeting_agent의 실제 내부 이름을 확인한 뒤, `agents/supervisor.md` 문구로 `agents/teamplay_supervisor.yaml`(collaborators: schedule_agent, <meeting_agent의 실제 이름>)을 만들고 등록.
7. 배포: 필요하면 세 에이전트를 배포(deploy)해서 채팅 화면에서 쓸 수 있게 한다.
8. 각 단계가 끝나면 무엇을 만들었는지 3줄로 요약한다.
