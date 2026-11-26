---
name: teamplay-builder
description: "팀플 해결사" 해커톤 프로젝트의 Python 도구(find_common_slots, backward_schedule)를 만들고 테스트하고, watsonx Orchestrate에 등록하고, 에이전트(tp_schedule_agent, tp_supervisor)를 만들고 배포할 때 사용한다. 이 폴더에서 도구, 테스트, 에이전트 YAML, 등록, 배포 요청이 오면 항상 이 스킬을 따른다.
---

# teamplay-builder

팀원 가능 시간과 회의 기록을 받아 팀플을 돕는 watsonx Orchestrate 멀티 에이전트를 만든다.

## 이름 규칙
- 에이전트 이름에는 팀 접두어 `tp_`: tp_schedule_agent, tp_meeting_agent, tp_supervisor.
- 도구 이름(find_common_slots, backward_schedule)은 그대로. 같은 이름의 도구가 이미 있다고 나오면 앞에 `tp_`를 붙이고 에이전트 YAML과 instructions의 도구 이름도 같이 바꾼다.
- 에이전트 이름이 이미 있다고 나오면(다른 팀이 같은 이름을 씁) 덮어쓰지 말고 멈춘 뒤 사용자에게 알린다.

## 파일 위치
- 샘플 데이터: `data/availability.json`(JSON), `data/availability.txt`(글 형식), `data/strengths.json`, `data/meeting_1.txt`, `data/meeting_2.txt`, `data/kakao_chat.txt`
- 에이전트 문구: `agents/schedule_agent.md`, `agents/meeting_agent.md`, `agents/supervisor.md` — Description과 Instructions를 바꾸지 말고 그대로 쓴다
- 정답표: `answers.md`, 데모 입력: `demo_inputs.txt`
- 만들 코드: `tools/` (도구), `tests/` (테스트), `agents/*.yaml` (에이전트)
- 비밀 값: `.env` (저장소에 올리지 않음)

## 환경 (Windows 노트북)
- 환경 설정은 행사 실습 안내를 따른다. 가상환경은 프로젝트 폴더의 `.venv` (Python 3.12). Orchestrate에서 도구가 실행되는 환경이 Python 3.12이므로 로컬도 3.12를 쓴다. 명령을 실행하기 전에 `.venv\Scripts\activate` 상태인지 확인한다.
- `orchestrate` 명령이 없으면 혼자 설치하지 말고, `.venv`에 `pip install ibm-watsonx-orchestrate`로 설치해도 되는지 사용자에게 먼저 묻는다.
- Orchestrate 연결(`orchestrate env add`, `orchestrate env activate`)은 사용자가 직접 터미널에서 한다. API 키를 묻거나 출력하지 않는다.
- 원격 연결은 2시간이 지나면 만료된다. orchestrate 명령이 인증 오류를 내면 사용자에게 "`orchestrate env activate hack --api-key <키>`를 다시 실행해 주세요"라고 알린다.
- 모델 id는 사용자가 `orchestrate models list`에서 고른 이름을 그대로 쓴다. 직접 지어내지 않는다.

## 도구 만드는 방식
- Python 도구로 만든다. 파일 하나에 도구 하나: `tools/find_common_slots.py`, `tools/backward_schedule.py`.
- `from ibm_watsonx_orchestrate.agent_builder.tools import tool` 후 함수에 `@tool`.
- 모든 인자와 반환값에 타입 힌트. Google 스타일 docstring(요약 한 줄 + Args: + Returns:)을 한국어로 쓴다. 요약이 도구 설명이 되고 Args가 인자 설명이 된다.
- 계산은 표준 라이브러리만. 외부 패키지를 쓰지 않으므로 `requirements.txt`는 빈 파일로 둔다.
- 테스트가 `@tool` 없이 돌아가도록 계산 함수(예: `compute_common_slots`)를 따로 두고 `@tool` 함수가 그것을 부른다.

## 도구1: find_common_slots
- 입력: availability (str), min_minutes (int, 기본 60), top_n (int, 기본 3)
- availability는 두 형식을 모두 받는다: (1) `data/availability.json` 형식 JSON 문자열 (2) `data/availability.txt` 형식 글 ("가은: 월 13~18시, 화 10~12시" 한 줄에 한 명, "13:30~15시"처럼 분이 있어도 해석).
- 요일별로 모든 팀원의 가능 시간이 겹치는 구간을 찾는다. min_minutes보다 짧으면 제외. 긴 순서, 같으면 월→금.
- 출력: [{"요일", "시작", "끝", "분"}]. 겹치는 구간이 없으면 빈 목록과 함께 가장 많은 인원이 가능한 시간.
- 해석할 수 없으면 예외 대신 "가능 시간 형식을 확인해 주세요 (예: 가은: 월 13~18시)"를 돌려준다.
- 정답 (두 형식 모두): 1위 수 18:00-20:00(120분), 2위 월 16:00-17:00(60분), 3위 목 20:00-21:00(60분).

## 도구2: backward_schedule
- 입력: start_date (str), deadline (str), 둘 다 YYYY-MM-DD
- 발표 전날 = "발표 리허설"(1일). 시작일~리허설 전날 일수 N. "자료조사" = round(N*0.4), "발표 초안" = round(N*0.25), "PPT 제작" = 나머지. 이 순서로 날짜 배정.
- 출력: [{"작업", "시작일", "종료일", "일수"}]
- 정답 (2026-11-27 → 2026-12-10): 자료조사 11/27~12/1(5일), 발표 초안 12/2~12/4(3일), PPT 제작 12/5~12/8(4일), 발표 리허설 12/9(1일)

## 에이전트 YAML 형식
```yaml
spec_version: v1
kind: native
name: tp_schedule_agent
description: |
  (agents/schedule_agent.md의 Description 그대로)
instructions: |
  (agents/schedule_agent.md의 Instructions 그대로)
llm: <사용자가 models list에서 고른 모델 id>
style: react_core
tools:
  - find_common_slots
  - backward_schedule
```
총괄은 tools 대신 `collaborators:`에 두 에이전트의 **실제 name**을 넣는다.

## tp_meeting_agent의 실제 name 찾기 (총괄 만들기 전에 반드시)
- tp_meeting_agent는 팀원이 Orchestrate 화면에서 만든다. 화면에 보이는 이름과 내부 name이 다를 수 있다.
- `orchestrate agents list -v` 를 실행한다 (모든 에이전트의 상세 정보가 JSON으로 나온다).
- JSON에서 tp_meeting_agent에 해당하는 항목의 `"name"` 값을 그대로 collaborators에 쓴다. 추측하지 않는다.
- 그다음 백업으로 `orchestrate agents export -n <그 name> -k native -o agents/tp_meeting_agent.yaml --agent-only` 를 실행해 저장소에 남긴다.

## 명령어 (공식 문서 기준)
- 도구 등록: `orchestrate tools import -k python -f tools/find_common_slots.py -r requirements.txt` (도구마다 한 번, `-r requirements.txt` 항상 붙임)
- 확인: `orchestrate tools list`
- 에이전트 등록: `orchestrate agents import -f agents/tp_schedule_agent.yaml` (등록하면 초안 상태)
- 수정: 같은 name으로 YAML을 고쳐 `orchestrate agents import -f` 를 다시 실행하면 덮어쓴다.
- 확인: `orchestrate agents list` (자세히: `orchestrate agents list -v`)
- 배포: `orchestrate agents deploy -n <에이전트 name>` (배포해야 웹 채팅에 나옴. Developer Edition이면 배포 없이 초안으로 나오고 deploy는 오류)
- 명령 옵션이 다르다는 오류가 나면 `<명령> --help`로 확인해서 맞춘다.

## 각 단계 끝에
무엇을 만들었는지 3줄 요약, 그다음 git commit 후 push (.env 제외).
