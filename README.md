# 팀플 해결사 (Teamplay Solver)

숙명여대 AI센터 x IBM Campus Agent Hackathon (2026-11-27) · 팀 숙련자들

팀원 가능 시간과 회의록을 넣으면 회의 시간·역할·일정을 짜고, 기여도를 추적해 독촉 메시지와 교수님 제출용 기여도 보고서까지 써주는 watsonx Orchestrate 멀티 에이전트.

## 구조
- tp_supervisor (총괄) → tp_schedule_agent, tp_meeting_agent에 일을 나눔
- tp_schedule_agent: 도구 find_common_slots, backward_schedule (Bob으로 Python 제작)
- tp_meeting_agent: 지시문만으로 회의 정리·기여도·독촉 메시지·보고서 (노코드)

## 폴더
- data/ : 가상 팀원 4명(가은·나은·다은·라은) 가능 시간, 강점, 과제 공지, 회의록 2개
- agents/ : 각 에이전트의 description·instructions (Orchestrate에 그대로 붙여 넣기)
- demo_inputs.txt : 데모 입력 문장
- answers.md : 테스트 정답표
