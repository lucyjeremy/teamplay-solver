# 팀플 해결사 (Teamplay Solver)

숙명여대 AI센터 x IBM Campus Agent Hackathon (2026-11-27) · 팀 숙련자들

팀 프로젝트를 하는 사람과 팀을 지도·평가하는 사람(교수, 팀장 등) 누구나 쓰는 watsonx Orchestrate 멀티 에이전트. 주제 추천부터 회의 시간·역할·일정, 회의록·카톡 기반 현재 팀 기여도 리포트, 독촉 메시지까지 만든다.

## 구조 (총괄 1 + 기능 에이전트 4)
- tp_supervisor (총괄): 요청을 보고 아래 4개에게 맡김
- tp_topic: 팀플 주제 추천
- tp_plan: 회의 시간·역할·일정 (도구 find_common_slots, backward_schedule, Bob으로 Python 제작)
- tp_track: 회의 정리 + 현재 팀 기여도 리포트 (Knowledge: data/course_guide.txt)
- tp_message: 일을 안 한 팀원에게 보낼 메시지

## 폴더
- data/ : 가상 팀원 4명(가은·나은·다은·라은) 자료, 과제 안내(course_guide.txt)
- agents/ : 각 에이전트의 description·instructions (Orchestrate 화면에 그대로 붙여 넣기)
- demo_inputs.txt : 데모 입력 문장
- answers.md : 테스트 정답표
