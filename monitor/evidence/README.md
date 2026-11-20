# monitor/evidence — 증거 파일 규칙

이 폴더는 Pickle(정욱님의 개인 비서)과 GPT가 **같은 원장을 보고** 검토하기 위한 공개 협업 공간입니다.

## 무엇이 올라가는가
- `monitor/latest.json` — 3시장(업비트·한국·미국) 최신 운영·연구·검증 상태 요약 (5분 자동 갱신)
- `monitor/events.jsonl` — 구현·수리·검증·프루닝·세대교체 같은 주요 사건 (append-only)
- `monitor/telegram_reports.jsonl` — 텔레그램 발송 이력의 **비식별** 기록(시각·유형·성공 여부만)
- `coordination/gpt_to_pickle.jsonl` — 사용자가 전달한 GPT 의견의 ID·접수 시각·처리 상태
- `coordination/pickle_to_gpt.jsonl` — GPT에 보내는 검토 요청·반론
- `research/reports/` — 회차 보고서, `research/test_results/` — 검증 결과, `research/evaluation_summaries/` — 집계 요약

## 무엇이 올라가지 않는가 (공개 저장소)
- 비밀키·토큰·PAT·계정 인증정보, 텔레그램 chat id
- 개인정보(이름·전화번호·주소·이메일)
- 원본 대용량 DB(1분봉·연구 패널), 절대경로
- 민감한 전략 원본 코드(임계값·특징 계산 구현). 요약 수치와 검증 결과만 공개합니다.

## 재현
각 파일의 `evidence` 필드는 같은 저장소의 보고서 절을 가리킵니다.
수치의 분자·분모와 DEV/HOLD/Shadow 구분은 보고서에 함께 적혀 있습니다.
