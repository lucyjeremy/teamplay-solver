# 팀플 해결사 프로젝트 규칙

- 항상 한국어로 답하고, 설명은 짧게 한다. 사용자는 개발 입문자다. 전문 용어는 처음 나올 때 한 줄로 풀어준다.
- 도구의 계산 코드는 Python 표준 라이브러리만 쓴다. Orchestrate ADK(ibm-watsonx-orchestrate)는 예외.
- 팀원 이름(가은, 나은, 다은, 라은), 작업 이름(자료조사, 발표 초안, PPT 제작, 발표 리허설), 요일 표기(월~금), 시간 표기("18:00-20:00")는 바꾸지 않는다.
- 코드를 바꾸면 테스트를 다시 돌리고 `answers.md` 정답과 비교한다.
- `data/`, `agents/*.md`, `answers.md`는 사용자가 요청할 때만 수정한다.
- skr_schedule_agent와 skr_supervisor는 YAML로만 고치고 다시 import한다. skr_meeting_agent는 팀원이 Orchestrate 화면에서 관리하므로 건드리지 않는다.
- API 키, 인스턴스 주소 같은 비밀 값은 코드나 YAML에 직접 쓰지 않고 `.env`에만 둔다. `.env`는 절대 커밋하지 않는다.
- 파일을 지우거나 큰 구조를 바꿀 때는 먼저 물어본다.
