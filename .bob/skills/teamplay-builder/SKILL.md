---
name: teamplay-builder
description: "팀플 해결사" 해커톤 프로젝트의 Python 도구(find_common_slots, backward_schedule)를 만들고 테스트하고 watsonx Orchestrate에 등록할 때, 그리고 agents/*.md 문구로 에이전트를 만들 때 사용한다. 이 폴더에서 도구, 테스트, 도구 등록, 에이전트 만들기 요청이 오면 항상 이 스킬을 따른다.
---

# teamplay-builder

팀플을 돕는 watsonx Orchestrate 멀티 에이전트(총괄 tp_supervisor + topic_helper_1, tp_plan, tp_track, tp_message) 중 tp_plan이 쓸 Python 도구 2개를 만들고 등록한다. 사용자가 요청하면 `agents/*.md` 문구 그대로 에이전트도 만든다 (아래 "에이전트 만들기").

## 이름 규칙
- 에이전트 이름: tp_supervisor, topic_helper_1, tp_plan, tp_track, tp_message. 이름은 각 `agents/*.md` 첫 줄에 적힌 것 그대로.
- 도구 이름(find_common_slots, backward_schedule)은 그대로. 이미 우리가 등록한 도구를 고칠 때는 같은 명령으로 다시 import해서 덮어쓴다 (오류가 나면 `orchestrate tools import --help`로 확인하고, 그래도 안 되면 멈추고 알린다).

## 파일 위치
- 샘플 데이터: `data/availability.json`(JSON), `data/availability.txt`(글 형식), `data/strengths.json`, `data/meeting_1.txt`, `data/meeting_2.txt`, `data/kakao_chat.txt`
- 에이전트 문구: `agents/*_agent.md`, `agents/supervisor.md` (에이전트의 원본. 수정하지 않는다)
- 정답표: `answers.md`, 데모 입력: `demo_inputs.txt`
- 만들 코드: `tools/` (도구), `tests/` (테스트)
- 비밀 값: `key.txt`, `.env` (읽지 않고, 저장소에 올리지 않음)

## 환경 (Windows 노트북)
- 이 폴더는 행사 실습 때 Bob의 **watsonx Orchestrate ADK 확장**이 만든 워크스페이스다 (venv/, workspace_config.yaml, key.txt, models/ 등). 상태표시줄에 `ADK: ✅`가 보이면 Orchestrate에 연결된 상태다.
- 가상환경은 확장이 만든 `venv` 폴더를 그대로 쓴다. 새 가상환경을 만들거나 Python을 따로 설치하지 않는다. `orchestrate` 명령이 안 잡히면 혼자 설치하지 말고 사용자에게 알린다.
- 연결과 API 키는 확장이 관리한다. `orchestrate env add`/`env activate`를 실행하지 않는다. `key.txt`와 API 키는 읽거나 출력하지 않는다.
- orchestrate 명령이 인증 오류를 내면 더 시도하지 말고 멈춘 뒤 "ADK 확장에서 API 키를 다시 넣어 주세요"라고 알린다.
- 모델 id는 사용자가 `orchestrate models list`에서 고른 이름을 그대로 쓴다. 직접 지어내지 않는다.
- 실습에서 만든 다른 파일(예: email_helper 에이전트)은 건드리지 않는다. 이 프로젝트 요청에는 다른 스킬(wxo-builder 등)보다 이 스킬을 우선한다.

## 도구 만드는 방식
- Python 도구로 만든다. 파일 하나에 도구 하나: `tools/find_common_slots.py`, `tools/backward_schedule.py`.
- `from ibm_watsonx_orchestrate.agent_builder.tools import tool` 후 함수에 `@tool`.
- 모든 인자와 반환값에 타입 힌트. Google 스타일 docstring(요약 한 줄 + Args: + Returns:)을 한국어로 쓴다. 요약이 도구 설명이 되고 Args가 인자 설명이 된다.
- 계산은 표준 라이브러리만. 외부 패키지를 쓰지 않으므로 `requirements.txt`는 빈 파일로 둔다.
- 테스트가 `@tool` 없이 돌아가도록 계산 함수(예: `compute_common_slots`)를 따로 두고 `@tool` 함수가 그것을 부른다.
- 도구 파일 하나로 완결한다: 계산 함수도 같은 파일 안에 두고, 다른 로컬 파일(공통 모듈 등)을 import하지 않는다. 다른 파일을 import하면 등록할 때 `-p`(package_root) 옵션과 상대 import가 필요해져 등록이 복잡해진다.
- 반환 타입은 항상 하나로 고정한다(아래 각 도구의 `-> dict`). Orchestrate는 반환 타입 힌트로 출력 형식을 정하므로, 성공일 때 목록·실패일 때 문자열처럼 섞어 돌려주지 않는다. 실패도 같은 dict 안의 `message`로 알린다.
- 테스트는 표준 라이브러리 `unittest`로 `tests/test_tools.py`에 쓰고 `python -m unittest discover tests`로 돌린다 (pytest를 설치하지 않는다). tools/는 패키지가 아니므로 테스트 파일 맨 위에서 `sys.path`에 tools 폴더를 넣고 import한다.

## 도구1: find_common_slots
- 입력: availability (str), min_minutes (int, 기본 60), top_n (int, 기본 3)
- availability는 두 형식을 모두 받는다: (1) `data/availability.json` 형식 JSON 문자열 (2) `data/availability.txt` 형식 글 ("가은: 월 13~18시, 화 10~12시" 한 줄에 한 명, "13:30~15시"처럼 분이 있어도 해석).
- 요일별로 모든 팀원의 가능 시간이 겹치는 구간을 찾는다. min_minutes보다 짧으면 제외. 긴 순서, 같으면 월→금.
- 출력(`-> dict`, 항상 이 모양): `{"slots": [{"요일", "시작", "끝", "분"}, ...], "table": str, "message": str}`
  - 정상: slots에 Top N, message는 빈 문자열
  - table: 주간 가능 시간표 마크다운 문자열. 첫 줄 `| 요일 | 10시 | 11시 | … |` (가장 이른 시작 시각부터 가장 늦은 끝 시각 직전까지 1시간 단위, "10시" = 10:00-11:00), 행은 월~금. 칸은 그 1시간을 통째로 가능한 인원 수 n으로 `🟩n`(전원) / `🟨n`(전원-1) / `⬜n`(그 외). 표 마지막 줄 다음에 빈 줄 하나(`\n\n`)를 두고 범례 `🟩 전원 가능 · 🟨 1명 빠짐 · ⬜ 2명 이상 빠짐`를 쓴다 (빈 줄이 없으면 화면에서 범례가 표의 한 줄로 들어가 첫 칸에 세로로 꺾여 보인다). 겹치는 구간이 없을 때도 table은 만든다. 해석할 수 없을 때만 빈 문자열.
  - 겹치는 구간이 없음: slots는 [], message에 "모두 되는 시간이 없습니다. 가장 많은 인원(N명)이 되는 시간: …"
  - 해석할 수 없음: 예외를 내지 않고 slots는 [], message에 "가능 시간 형식을 확인해 주세요 (예: 가은: 월 13~18시)"
- 정답 (두 형식 모두): 1위 수 18:00-20:00(120분), 2위 월 16:00-17:00(60분), 3위 목 20:00-21:00(60분).
- table 정답: `answers.md`의 "데모 1-0 주간 가능 시간표"와 칸마다 같아야 한다 (🟩는 월 16시, 수 18·19시, 목 20시, 금 14시 다섯 칸뿐).

## 도구2: backward_schedule
- 입력: start_date (str), deadline (str), 둘 다 YYYY-MM-DD, research_ratio (float, 기본 0.4, 0.2~0.6 밖이면 schedule [] + message)
- 발표 전날 = "발표 리허설"(1일). 시작일~리허설 전날 일수 N. "자료조사" = 반올림(N*research_ratio), "발표 초안" = 반올림(N*0.25), "PPT 제작" = 나머지. 이 순서로 날짜 배정.
- 반올림은 `int(x + 0.5)`로 한다. Python의 `round()`는 2.5를 2로 만들기 때문에 쓰지 않는다 (심사위원이 다른 날짜로 시험해도 상식대로 나오게).
- 출력(`-> dict`, 항상 이 모양): `{"schedule": [{"작업", "시작일", "종료일", "일수"}, ...], "message": str}`
  - 날짜 형식이 틀렸거나 N이 3보다 작거나, 계산한 작업 중 하루도 안 되는 것이 있으면: 예외 없이 schedule은 [], message에 이유
- 정답 (2026-11-27 → 2026-12-10): 자료조사 11/27~12/1(5일), 발표 초안 12/2~12/4(3일), PPT 제작 12/5~12/8(4일), 발표 리허설 12/9(1일)
- 정답 (같은 날짜, research_ratio=0.5): 자료조사 11/27~12/2(6일), 발표 초안 12/3~12/5(3일), PPT 제작 12/6~12/8(3일), 발표 리허설 12/9(1일)

## 에이전트 만들기 (사용자가 요청할 때만)
- 에이전트 하나당 New task 하나. 한 task에서 여러 에이전트를 만들지 않는다.
- 이름, Description, Instructions는 해당 `agents/*.md`의 첫 줄 이름, `## Description` 아래, `## Instructions` 아래를 글자 그대로 쓴다. 줄이거나 다듬지 않는다.
- 모델: `groq/openai/gpt-oss-120b`
- 연결할 것: tp_plan은 도구 find_common_slots·backward_schedule / tp_track은 Knowledge에 `data/course_guide.txt` / tp_supervisor는 협력 에이전트 topic_helper_1, tp_plan, tp_track, tp_message (이 4개가 먼저 있어야 함) / topic_helper_1, tp_message는 연결 없음
- 같은 이름이 이미 있으면 덮어쓰지 말고 멈춘 뒤 물어본다. import 명령이나 YAML 형식이 헷갈리면 `orchestrate agents import --help`로 확인한다.
- 만든 뒤 `orchestrate agents list`로 확인하고, 화면 Draft Preview에서 `demo_inputs.txt`의 해당 데모로 시험해 보라고 사용자에게 알린다.

## 명령어 (공식 문서 기준)
- 도구 등록: `orchestrate tools import -k python -f tools/find_common_slots.py -r requirements.txt` (도구마다 한 번, `-r requirements.txt` 항상 붙임)
- 확인: `orchestrate tools list`
- 명령 옵션이 다르다는 오류가 나면 `<명령> --help`로 확인해서 맞춘다.

## 각 단계 끝에
무엇을 만들었는지 3줄 요약만 한다. git commit·push는 사용자가 요청할 때만 한다 (이 폴더는 git 저장소가 아닐 수 있다).
