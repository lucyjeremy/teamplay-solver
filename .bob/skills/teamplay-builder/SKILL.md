---
name: teamplay-builder
description: "팀플 해결사" 해커톤 프로젝트의 Python 도구(find_common_slots, backward_schedule)를 만들고 테스트하고, watsonx Orchestrate 에이전트(schedule_agent, teamplay_supervisor)를 등록할 때 사용한다. 이 폴더에서 도구, 테스트, 에이전트 YAML, Orchestrate 등록 요청이 오면 항상 이 스킬을 따른다.
---

# teamplay-builder

팀원 가능 시간과 회의록을 받아 팀플을 돕는 watsonx Orchestrate 멀티 에이전트를 만든다.

## 파일 위치
- 샘플 데이터: `data/availability.json`, `data/strengths.json`, `data/meeting_1.txt`, `data/meeting_2.txt`
- 에이전트 문구(description, instructions): `agents/schedule_agent.md`, `agents/meeting_agent.md`, `agents/supervisor.md` — 문구를 바꾸지 말고 그대로 쓴다
- 정답표: `answers.md`
- 만들 코드: `tools/` 폴더 (도구), `tests/` 폴더 (테스트), `agents/*.yaml` (에이전트)

## 도구1: find_common_slots
- 입력: availability_json (문자열, `data/availability.json` 형식), min_minutes (정수, 기본 60), top_n (정수, 기본 3)
- 요일별로 모든 팀원의 가능 시간이 겹치는 구간을 찾는다. min_minutes보다 짧으면 제외한다.
- 긴 순서로 정렬하고, 길이가 같으면 요일 순서(월, 화, 수, 목, 금).
- 출력: [{"요일", "시작", "끝", "분"}] 목록. 겹치는 구간이 없으면 빈 목록과 함께 가장 많은 인원이 가능한 시간을 돌려준다.
- 정답: 1위 수 18:00-20:00(120분), 2위 월 16:00-17:00(60분), 3위 목 20:00-21:00(60분)

## 도구2: backward_schedule
- 입력: start_date, deadline (YYYY-MM-DD 문자열)
- 발표 전날 = "발표 리허설"(1일)
- 시작일~리허설 전날 일수 N. "자료조사" = round(N*0.4), "발표 초안" = round(N*0.25), "PPT 제작" = 나머지. 이 순서로 날짜를 배정한다.
- 출력: [{"작업", "시작일", "종료일", "일수"}]
- 정답 (2026-11-27 → 2026-12-10): 자료조사 11/27~12/1(5일), 발표 초안 12/2~12/4(3일), PPT 제작 12/5~12/8(4일), 발표 리허설 12/9(1일)

## 진행 순서
1. Plan: 위 스펙으로 계획을 세우고 사용자 승인을 받는다.
2. Agent: `tools/`에 도구를 만들고 `tests/`에 위 정답으로 테스트를 만든다. 테스트가 통과할 때까지 고친 뒤 결과를 표로 보여준다.
3. 등록: 해커톤 실습과 같은 방식으로 Orchestrate에 도구를 등록한다. orchestrate 명령어는 먼저 --help로 확인한다. watsonx Orchestrate ADK MCP 서버나 스킬이 있으면 그것을 우선 쓴다.
4. 에이전트: `agents/schedule_agent.md` 문구로 `agents/schedule_agent.yaml`을 만들고 등록한다. 총괄은 `agents/supervisor.md`로 `agents/teamplay_supervisor.yaml`(collaborators: schedule_agent, meeting_agent).
5. 각 단계가 끝나면 무엇을 만들었는지 3줄로 요약한다.
