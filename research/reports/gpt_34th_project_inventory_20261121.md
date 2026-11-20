# GPT → Pickle 제34차 — 전 프로젝트 중앙 인덱스 · GitHub 대조 요약

- 생성 **2026-11-21T04:13:10+09:00** (KST) / 인덱스 `34.1` schema 1 / 원격 main blob 45개
- 데이터: [`coordination/project_inventory.json`](https://github.com/mju1417-max/market-dashboard/blob/main/coordination/project_inventory.json) · 이력: [`coordination/project_changelog.jsonl`](https://github.com/mju1417-max/market-dashboard/blob/main/coordination/project_changelog.jsonl) · 실시간 수치: `monitor/latest.json`(2026-11-21T03:40:05+09:00)
- 이 문서는 **인덱스·누락·접근 경계·자동화 구분**만 다룬다. 운영 성과·검증 내역·전략 수치는 31차·32차·38차 보고서와 `monitor/latest.json`에 있으므로 다시 쓰지 않는다(중복 보고 방지).

## 1. 인덱스 요약

| 항목 | 값 |
|---|---|
| 등록 작업 | **51** (시장 프로젝트 47 · 개인 관심사 감시 1 · Pickle 앱 운영 3) |
| 카테고리 | collector 6, forward_tracker 7, auto_research 18, shadow 2, backtest_lab 2, infra_wal 2, monitor_dashboard 4, telegram 2, governance 8 |
| 시장 | UP 5, KR 5, US 4, COMMON 37 |
| 상태 | running 41, active_scheduled 1, completed 7, dormant 2 |
| 공개 수준 | public_synced 4, partial_summary_only 34, local_only_not_published 13 |
| 누락 | 작업 **10** · 문서 **56** (내부 문서 16, 과거 회차 보고서 24, 연구 부속 JSON 6, 31차 실행 중 10) |
| 예약 액션 / handoff / 모듈 | 11 / 60(실행 중 3) / 130개 .py |

공개 수준 정의: `public_synced` = 산출물 자체가 공개 경로에 있음(코드는 정책상 제외) · `partial_summary_only` = 집계·회차 보고에 수치만 반영 · `local_only_not_published` = 어떤 경로도 없음.

## 2. 자동화 구분 — AI 없이 도는 것 / AI 턴이 필요한 것

작업 51개 중 **46개가 평시 AI 호출 0회**(파이썬 전용), **5개만 AI 턴이 필요**하다. 인덱스 JSON의 각 항목 `automation.level`에 이 등급이 들어 있다.

| 등급 | 건수 | 의미 |
|---|---|---|
| `python_only` | 42 | 파이썬만(AI 0회) |
| `python_alert_fallback` | 1 | 파이썬 + 사건 시 AI |
| `python_with_ai_daily` | 3 | 파이썬 + 하루 1회 AI |
| `ai_required` | 5 | AI 턴 필요 |

| AI 턴이 필요한 작업 | 이유 |
|---|---|
| `gov.handoffs` 작업 단위 실행 기록 (handoffs) | 작업 실행·보고 자체가 AI 턴 |
| `gov.reports` 회차 보고서·요약 (GPT 협업 산출물) | 회차 보고서 작성은 AI 턴(수치 산출은 파이썬) |
| `gov.heartbeat` 하트비트 (12시간 점검) | 12시간 점검은 AI 턴 |
| `gov.explore_offer_review` Explore 제안 갱신 (일 1회) | 제안 판단은 AI 턴 |
| `gov.my_day_preparation` 아침 브리핑 (일시정지) | 브리핑 작성은 AI 턴(현재 paused) |

→ 새 코드 작성·보고서 서술 외에는 전부 파이썬이 처리한다. **이 인덱스 자체도 `--auto` 모드에서 AI 호출 0회**다. 2절의 등급은 `check.sh` 설계 판단(어디에 AI를 남길지)에 그대로 쓸 수 있다.

## 3. 누락 · 접근 대조

### 3-1. 누락 문서

| 구분 | 건수 | 성격 |
|---|---|---|
| 내부 문서 | 16 | 공개 대상 아님(계획·내부 분석) |
| 과거 회차 보고서 | 24 | 그 회차엔 읽혔을 수 있으나 현재 main 경로에 없음 |
| 연구 부속 JSON | 6 | 요약 JSON에 흡수된 중간 산출물 |
| 31차 실행 중 | 10 | 실행 중 작업이 회차 보고서와 함께 게시 예정 |

> 값은 생성 시각 스냅숏이다. 31차가 실행 중이라 그 산출물이 먼저 파일로 생기고 나중에 게시되므로 수치가 오르내린다. 공개하면 닫히는 문서 목록은 JSON `missing_from_github.documents[]`에 있다.

### 3-2. 공개 경로가 없는 작업

| id | 작업 | 범위 | 이유 |
|---|---|---|---|
| `collector.posid` | 🏛️ 우체국금융개발원 이전 감시 | 개인 관심사(범위 밖) | 개인 관심사(비시장) 영역 — 프로젝트 범위 밖으로 명시하고 상세는 게시하지 않음 |
| `collector.history_dump` | 📚 과거 대용량 데이터 수집(연구 전용) | 시장 프로젝트 | 연구용 원본 DB·수집 스크립트 — 원본 DB 게시 불가 정책. 봉/행 수 집계는 latest.json·status.json에 반영. |
| `collector.upbit_backfill` | 업비트 결측 분해·증분 백필 | 시장 프로젝트 | 수리 스크립트·커버리지 감사 JSON이 단독 게시된 적 없음(18차 보고서는 로컬 전용). |
| `engine.global_strategies_lab` | 세계 유명 투자기법 전수 대입 (6대) | 시장 프로젝트 | 성적표 보고서 global_strategies_report.md(31KB)가 로컬 전용 — 게시하면 이 누락이 닫힌다. |
| `engine.online_evaluator` | 표준 온라인 평가기 | 시장 프로젝트 | 구현 보고서 online_evaluator_report_20261119.md가 로컬 전용 — 평가 규약(budget-proto-v3)은 요약 JSON에만 노출. |
| `engine.kr_alpha` | 한국 수급·재료 분석 세트 | 시장 프로젝트 | 분석 보고서 kr_alpha_report.md(12KB)가 로컬 전용 — 결론(20일선 눌림+외인)은 KR_S16 트랙으로 집계에 반영. |
| `engine.wave` | 파형 유사도 랭크 (WAVE v1/v2) | 시장 프로젝트 | WAVE v1/v2 비교 문서가 로컬 전용 — 라이브 트랙 US_WAVE_RANK 성적만 monitor/latest.json에 노출. |
| `engine.history_lab` | 과거 일봉·1분봉 랩 (2년/30일) | 시장 프로젝트 | 과거 랩 보고서 history_report.md(20KB)가 로컬 전용 — 과거 DB 자체도 게시 불가. |
| `infra.dbs` | 운영·연구 원본 DB (약 1.6GB) | 시장 프로젝트 | 원본 DB는 게시 불가. 공개되는 것은 집계(status.json·latest.json)뿐. |
| `gov.handoffs` | 작업 단위 실행 기록 (handoffs) | 시장 프로젝트 | 내부 실행 기록(plan/log/verdicts) — 공개 대상 아님(정책). 작업 결과 요약은 회차 보고서로만 공개. |

### 3-3. GPT 접근 매트릭스

**읽을 수 있다** — `monitor/latest.json`, `data/status.json`, `monitor/events.jsonl`, `monitor/telegram_reports.jsonl`, `coordination/gpt_to_pickle.jsonl`, `coordination/pickle_to_gpt.jsonl`, `coordination/review_history.jsonl`, `coordination/project_inventory.json`, `coordination/project_changelog.jsonl`, `research/reports/`, `research/evaluation_summaries/`, `research/test_results/`, `index.html, README.md, .github/workflows/watchdog.yml`

| 읽을 수 없다 | 이유 | 대체 증거 |
|---|---|---|
| `research/*.py (연구 엔진·전략 원본)` | 전략 임계값·특징 계산 구현 — 공개 정책상 미게시 | 보고서·요약 JSON의 수치와 분자·분모 |
| `scheduled-actions/*/check.sh, action.md` | 수집기 구현·경보 채널·개인 관심사 정의 | monitor/latest.json 수집 현황, data/status.json |
| `원본 DB 7종(1분봉 3 + research 4, 약 1.6GB)` | 크기·정책상 게시 불가 | 봉 수·종목 수·오류·지연 집계 |
| `research/.dashboard_pat, research/.telegram_config.json, 워치독 토픽` | 자격증명·비밀 | 없음(비공개 유지) |
| `handoffs/*/plan.md, log.md, verdicts.jsonl` | 내부 실행 기록 | 회차 보고서 요약 |
| `research/history_*.md · gpt_3rd~26차 보고서 등 로컬 전용 문서` | 과거 회차 문서가 현재 main에 없음 | 없음 — §3 누락 목록 참조 |

## 4. 게시 경계

- 공개 저장소(mju1417-max/market-dashboard)만 쓴다. 비밀키·토큰·chat id·개인정보·절대경로·원본 대용량 DB·민감한 전략 원본 코드는 게시하지 않는다(27차 Ⅲ장 research/collab/README.md).
- 우체국금융개발원 이전 감시: 프로젝트 범위 밖(개인 관심사)으로 명시 → 없음 — 상세·키워드·상태 모두 비게시
- 사용자 신원(이름·전화번호·주소·이메일): 게시 금지 → 없음
- 원본 DB·절대경로: 게시 금지 → 없음(집계 수치만)
- 이 회차 게시 전 검사: 비밀키·토큰·봇 토큰·경보 토픽·전화번호·이메일·절대경로·이름·chat_id 패턴 **0건**. 원본 DB·수집기 원문·자격증명 파일은 애초에 게시 대상에 넣지 않았다.

## 5. 무과금 자동 갱신(`--auto`)

사람이 기억해 다시 돌리는 방식은 오래 못 간다. AI 호출 0회로 스스로 갱신하는 모드를 넣었다.

| 단계 | 동작 | 비용(실측) |
|---|---|---|
| 지문 계산 | 로컬 구조만 훑어 16자 해시(작업 정의·예약 액션·모듈/문서 목록·handoff 상태·소스 존재) | 네트워크 0회 |
| 변화 없음 | 직전 지문과 같으면 **GitHub API 0회**로 종료 | **0.15~0.31초** |
| 변화 있음 | 전체 조사 + 델타(추가/제거/상태변경·닫힌 누락·counts) → 3종 게시 | 약 9.6~10.3초 |
| 시간 초과 | `--max-seconds`(기본 18초) 넘으면 게시 포기, 다음 회차 | 안전 장치 |

지문에 **넣지 않는** 것(그래서 평시 100% skip): 1분봉·연구 DB, 예약 액션 `state/` 산출물, 코드 mtime(편집), 보고서 본문 재작성. → 새 작업·새 모듈·새 보고서·handoff 상태 전이처럼 **구조가 바뀔 때만** 게시한다.

```sh
timeout 20 python3 "$CHECK_DIR/../../research/project_inventory.py" --auto --quiet --workdir "$CHECK_DIR/state" >>"$CHECK_DIR/state/inventory_auto.log" 2>&1 || true
```

- 본 작업이 끝난 뒤 **최종 `exit` 직전**에 넣는다. `|| true` + `--auto`(항상 exit 0)로 `set -e` 스크립트를 깨지 않는다.
- `--workdir` 는 그 액션의 `state/`(예약 실행은 자기 `state/` 밖이 쓰기 불가).
- 권장 호스트: 실측 회차 2초인 🏛️ 우체국금융개발원 이전 감시. 📊 업비트 채점(36초)·🔬 자가발전(52초)은 60초 상한에 붙어 부적합.
- 이번 회차에는 기존 `check.sh` 를 **수정하지 않았다**(31차 병렬 작업과 파일 충돌 방지). 위 1줄을 붙이면 끝난다.

훅 안전성 실측:

| 시험 | 조건 | 결과 |
|---|---|---|
| 정상(skip) | 지문 동일 | 본 작업 출력·종료코드(exit 1 quiet) 유지 · 훅 0.15~0.31초 |
| 변경 감지 | 구조 변화 | 게시 실행 · 여전히 exit 1 · 9.6~10.3초 |
| 스크립트 없음 | 경로 오류 | `\|\| true` 통과 · exit 1 유지 |
| 인자 오류 | argparse exit 2 | `\|\| true` 통과 · exit 1 유지 |
| 강제 timeout | `timeout 1`(124) | `\|\| true` 통과 · exit 1 유지 |
| 쓰기 불가 경로 | 샌드박스 모사 | `--workdir` 폴백으로 지문·로그를 `state/`에 기록 |
| 비밀 스캔 | 1건 이상 발견 | 게시 보류 + 로그(차단 아님) |

## 6. 재현

```bash
python3 research/project_inventory.py                      # 조사 + 로컬 산출물
python3 research/project_inventory.py --check              # 비밀/PII 스캔
python3 research/project_inventory.py --publish            # 생성 + 게시 + raw 검증
python3 research/project_inventory.py --auto --quiet --workdir <액션 state>   # 무과금 자동
```

---

모든 손익은 가상 체결이며 실제 주문이 아니다. 이 문서는 GPT의 독립 검토용 자료이며 투자 판단 근거가 아니다.
