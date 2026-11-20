# GPT → Pickle 제34차 — 전 프로젝트 중앙 인덱스 · GitHub 대조 보고 (첫 회차 기준선)

- 생성 시각: **2026-11-21T04:06:09+09:00** (KST) / 인덱스 버전 `34.1` / schema 1
- 공개 실측 기준: `monitor/latest.json` generated_at **2026-11-21T03:40:05+09:00** (이 보고서의 운영 수치는 이 스냅숏)
- 인덱스 원본: [`coordination/project_inventory.json`](https://github.com/mju1417-max/market-dashboard/blob/main/coordination/project_inventory.json) · 변경 이력: [`coordination/project_changelog.jsonl`](https://github.com/mju1417-max/market-dashboard/blob/main/coordination/project_changelog.jsonl)
- 원격 main 파일 수(blob): **45**

## 1. 결론 요약

| 항목 | 값 | 설명 |
|---|---|---|
| 등록 작업 수 | **51** | 시장 프로젝트 47 + 개인 관심사 감시 1 + Pickle 앱 운영 3 |
| 카테고리 분포 | collector 6, forward_tracker 7, auto_research 18, shadow 2, backtest_lab 2, infra_wal 2, monitor_dashboard 4, telegram 2, governance 8 | collector/forward_tracker/auto_research/shadow/backtest_lab/infra_wal/monitor_dashboard/telegram/governance |
| 시장 분포 | UP 5, KR 5, US 4, COMMON 37 | UP·KR·US·COMMON |
| 상태 분포 | running 41, active_scheduled 1, completed 7, dormant 2 | running/active_scheduled/completed/draft/dormant/blocked |
| GitHub 공개 수준 | public_synced 3, partial_summary_only 35, local_only_not_published 13 | public_synced / partial_summary_only / local_only_not_published |
| **누락 식별** | 작업 **10** · 문서 **51** | 문서 = internal_doc 16, round_report 21, research_artifact 4, in_flight_31st 10 |
| 예약 액션 / handoff | 11 / 60(실행 중 3) | 5분·30분·일간·12시간 주기 |
| research 모듈 | 126개(.py) | 카테고리 자동 분류는 §5 |
| 프로토콜 버전 | budget-proto-v3-20261120 | 평가 규약 동결 버전 |

**핵심 판정 3가지**

1. 내부 작업은 **51개 운영 단위**로 정리됐고, 그중 **10개**는 공개 저장소에 어떤 경로도 없다. 대부분은 '전략 원본·수집기 구현'이라 공개 정책상 의도적 제외이고, 실제 누락은 문서 51건(내부 문서 16건 · 과거 회차 보고서 21건 · 연구 부속 JSON 4건 · 31차 실행 중 10건)이다.
2. GPT가 **지금 읽을 수 있는 것**은 `monitor/latest.json`, `data/status.json`, `coordination/*.jsonl`, `research/reports`·`evaluation_summaries`·`test_results`(총 45개 파일)다. 이 인덱스로 '무엇이 있는지'는 전부 볼 수 있게 됐지만, '어떻게 구현됐는지'(Python 원본)는 계속 볼 수 없다.
3. 우체국금융개발원(POSID) 감시는 시장 급등 프로젝트가 아니므로 `scope=personal_monitor`로 **분리 표기**하고 상세를 게시하지 않았다. 인덱스에는 '존재·상태'만 남겼다.

## 2. '작업'의 정의와 범위

인덱스의 1행 = **운영 단위 하나**다. 파일 하나가 아니다. 기준은 다음 셋 중 하나를 만족하는 것이다.

- 5분/30분/일간/12시간 주기로 **스스로 도는** 예약 액션
- 수집·연구·검증·게시 **파이프라인 단계**로 호출되는 엔진(여러 .py가 한 단위)
- 사용자·GPT가 보는 **산출물 계약**(monitor/latest.json, 대시보드, 텔레그램 보고)

회차성 1회 스크립트(`p0*_audit.py`, `verify_*`)는 개별 작업으로 세지 않고 `engine.audit_runners`·`gov.verification_harness`로 묶고, §5 모듈 목록에서 전수 분류한다.

## 3. 운영 실측 현황 (공개 스냅숏 그대로)

### 3-1. 3시장 수집

| 시장 | 봉 보유 종목 | 감시 유니버스 | 누적 1분봉 | 마지막 봉 | 수집기 마지막 실행 | 오류 |
|---|---|---|---|---|---|---|
| UPBIT | 302 | 302 | 1,591,424 | 2026-11-21T03:06:00 | 2026-11-21T03:38:00 | 0 |
| KR | 3,156 | 3,205 | 4,688,350 | 2026-11-20T15:44:00 | 2026-11-20T15:45:40 | 1 |
| US | 2,155 | 2,156 | 4,936,675 | 2026-11-21T03:06:57 | 2026-11-21T03:38:00 | 0 |

합계 **11,216,449봉 / 5,613종목**, +5% 급등 누적 **676건**.

### 3-2. 전진검증 트랙 7개

| 트랙 | 시장 | 신호 | 체결 | 청산 | 승 | 패 | 승률 | 누적 순손익 | 마지막 채점 |
|---|---|---|---|---|---|---|---|---|---|
| `US_WAVE_RANK` | US | 2 | 2 | 2 | 1 | 1 | 50.0% | +3.89%p | 2026-11-21T03:38:33 |
| `US_GEN16` | US | 26 | 23 | 23 | 7 | 16 | 30.4% | -16.97%p | 2026-11-21T03:38:37 |
| `KR_VCP` | KR | 2 | 1 | 0 | 0 | 0 | — | +0.00%p | 2026-11-20T20:21:08 |
| `KR_S16` | KR | 304 | 0 | 0 | 0 | 0 | — | +0.00%p | 2026-11-20T23:55:48 |
| `UP_ZVOL_GATE` | UPBIT | 23 | 23 | 22 | 8 | 14 | 36.4% | -12.29%p | 2026-11-21T03:39:55 |
| `UPBIT_GEN3` | UPBIT | 405 | 386 | 386 | 164 | 222 | 42.5% | -112.31%p | 2026-11-21T03:40:01 |
| `UPBIT_GEN2` | UPBIT | 0 | 0 | 0 | 0 | 0 | — | +0.00%p | 2026-11-21T03:39:59 |

전진검증 합계: 신호 **762** · 청산 **430** · 승 **180**. 연구 Shadow(프리레그)는 등록 **35** · TP **2** · FP **27** 로 **분모가 다르다**(합산 금지).

### 3-3. 세대·연구 상태

| 시장 | 연구 세대 | 상태 | Shadow 등록 | Shadow TP | Shadow FP |
|---|---|---|---|---|---|
| UPBIT | UP-G003 | rotation:running | 8 | 0 | 4 |
| KR | lab_KR_20261120_3f1a339d | daily_track_active | 0 | 0 | 0 |
| US | lab_US_20261119_11d1253a | lab_complete | 27 | 2 | 23 |

## 4. 작업 인덱스

### 4-1. 수집기

| id | 이름 | 시장 | 상태 | 우선 | 공개 | 실측 |
|---|---|---|---|---|---|---|
| `collector.upbit` | 🪙 업비트 5분 수집기 + 모의투자 정산 | UP | running | P0 | 요약만 | 1,591,424봉·302종목 |
| `collector.kr` | 🇰🇷 한국주식 5분 수집기 | KR | running | P0 | 요약만 | 4,688,350봉·3,156종목 |
| `collector.us` | 🇺🇸 미국주식 5분 수집기 | US | running | P0 | 요약만 | 4,936,675봉·2,155종목 |
| `collector.posid` | 🏛️ 우체국금융개발원 이전 감시 | COMMON | running | P3 | 미게시 | — |
| `collector.history_dump` | 📚 과거 대용량 데이터 수집(연구 전용) | COMMON | completed | P2 | 미게시 | — |
| `collector.upbit_backfill` | 업비트 결측 분해·증분 백필 | UP | completed | P2 | 미게시 | — |

### 4-2. 전진검증

| id | 이름 | 시장 | 상태 | 우선 | 공개 | 실측 |
|---|---|---|---|---|---|---|
| `track.UPBIT_GEN2` | 업비트 2세대 GEN2 (낙폭 되돌림·120분 고정청산) | UP | active_scheduled | P2 | 요약만 | 신호 0·청산 0 |
| `track.UPBIT_GEN3` | 업비트 3세대 GEN3 (저점권 낙폭 매수, 알림 Off Shadow) | UP | running | P2 | 요약만 | 신호 405·청산 386·승률 42.5% |
| `track.UP_ZVOL_GATE` | 업비트 거래대금 Z 게이트 (Shadow) | UP | running | P2 | 요약만 | 신호 23·청산 22·승률 36.4% |
| `track.US_GEN16` | 미국 2세대 GEN16 (역추세 주도주) | US | running | P1 | 요약만 | 신호 26·청산 23·승률 30.4% |
| `track.US_WAVE_RANK` | 미국 ACCEL×파형 랭크 (WAVE) | US | running | P2 | 요약만 | 신호 2·청산 2·승률 50.0% |
| `track.KR_S16` | 한국 S16 (20일선 위 눌림 + 외인 수급 20일 스윙) | KR | running | P1 | 요약만 | 신호 304·청산 0 |
| `track.KR_VCP` | 한국 VCP (Minervini 변동성 축소 돌파) | KR | running | P2 | 요약만 | 신호 2·청산 0 |

### 4-3. 자가발전 연구

| id | 이름 | 시장 | 상태 | 우선 | 공개 | 실측 |
|---|---|---|---|---|---|---|
| `engine.surge_miner` | 폭등 역추적 규칙 탐색 엔진 (2세대) | COMMON | running | P0 | 요약만 | — |
| `engine.surge_stage` | 폭등공식 단계 실행기 (state machine) | COMMON | running | P0 | 요약만 | — |
| `engine.strategy_lab` | 3세대 5대 기법 랩 (전수 백테스트 + OOS) | COMMON | running | P0 | 요약만 | — |
| `engine.global_strategies_lab` | 세계 유명 투자기법 전수 대입 (6대) | COMMON | completed | P1 | 미게시 | — |
| `engine.auto_researcher` | 자가발전 자동 패턴 발견·반증 엔진 | COMMON | running | P0 | 요약만 | — |
| `engine.production_connected` | 운영 연결 자가발전 (KR→US 30분 순환) | COMMON | running | P0 | 요약만 | — |
| `engine.daily_rerun` | 일일 재탐색 러너 (폭등공식+랩+요약) | COMMON | running | P1 | 요약만 | — |
| `engine.horizon_lab` | 지평 랩 H=30/60/120 (3시장) | COMMON | running | P0 | 요약만 | — |
| `engine.research_universe` | 연구 유니버스 (컷오프 고정 150종목) | COMMON | running | P0 | 요약만 | — |
| `engine.sparse_grid` | 저유동성 시간격자 연구 (sparse grid/shards) | COMMON | running | P2 | 요약만 | — |
| `engine.common_ledger` | 신호·사건 공통 원장 | COMMON | running | P1 | 요약만 | — |
| `engine.market_common` | 3시장 공통 인터페이스·어댑터 | COMMON | running | P1 | 요약만 | — |
| `engine.failure_hypothesis` | 실패 원장 기반 가설 생성 + 선행 가속 실험 | COMMON | completed | P2 | 요약만 | — |
| `engine.generation_rotation` | 자동 세대교체 스케줄 | COMMON | running | P1 | 요약만 | — |
| `engine.online_evaluator` | 표준 온라인 평가기 | COMMON | running | P1 | 미게시 | — |
| `engine.knowledge_ledger` | 지식 원장 | COMMON | running | P2 | 요약만 | — |
| `engine.kr_alpha` | 한국 수급·재료 분석 세트 | KR | completed | P1 | 미게시 | — |
| `engine.wave` | 파형 유사도 랭크 (WAVE v1/v2) | US | completed | P2 | 미게시 | — |

### 4-4. Shadow·인과성

| id | 이름 | 시장 | 상태 | 우선 | 공개 | 실측 |
|---|---|---|---|---|---|---|
| `shadow.shadow_engine` | Shadow 자동 연결 엔진 (프리레그(사전 등록)) | COMMON | running | P1 | 요약만 | — |
| `shadow.kr_causality` | KR 전진 인과성 감사 (S16·VCP) | KR | running | P1 | 요약만 | — |

### 4-5. 백테스트 랩

| id | 이름 | 시장 | 상태 | 우선 | 공개 | 실측 |
|---|---|---|---|---|---|---|
| `engine.history_lab` | 과거 일봉·1분봉 랩 (2년/30일) | COMMON | completed | P2 | 미게시 | — |
| `engine.backtest_strategies` | 1분봉 다전략 백테스터 | COMMON | dormant | P3 | 요약만 | — |

### 4-6. 인프라·WAL

| id | 이름 | 시장 | 상태 | 우선 | 공개 | 실측 |
|---|---|---|---|---|---|---|
| `infra.wal_replica` | WAL 일관 복제본·안전 읽기 | COMMON | running | P0 | 요약만 | — |
| `infra.dbs` | 운영·연구 원본 DB (약 1.6GB) | COMMON | running | P0 | 미게시 | — |

### 4-7. 대시보드·모니터

| id | 이름 | 시장 | 상태 | 우선 | 공개 | 실측 |
|---|---|---|---|---|---|---|
| `infra.pipeline_watchdog` | 🛡️ 파이프라인 정지 감시 (외부 워치독) | COMMON | running | P1 | 요약만 | — |
| `dash.export_status` | 공개 대시보드 스냅숏 내보내기 | COMMON | running | P0 | 공개 | — |
| `dash.monitor_research` | monitor/latest.json 연구 3시장 문서 (schema 3) | COMMON | running | P0 | 공개 | — |
| `dash.collab_ledger` | GitHub 협업 원장 게시 도구 | COMMON | running | P0 | 공개 | — |

### 4-8. 텔레그램

| id | 이름 | 시장 | 상태 | 우선 | 공개 | 실측 |
|---|---|---|---|---|---|---|
| `tg.reporter` | 텔레그램 직송 리포터 (사용량 0원 보고) | COMMON | running | P0 | 요약만 | — |
| `tg.audit_pack` | 일일 GPT 검수팩·아침 브리핑 | COMMON | running | P1 | 요약만 | — |

### 4-9. 거버넌스·기록

| id | 이름 | 시장 | 상태 | 우선 | 공개 | 실측 |
|---|---|---|---|---|---|---|
| `engine.audit_runners` | 정합성 감사 러너 묶음 | COMMON | running | P1 | 요약만 | — |
| `dash.project_inventory` | 전 프로젝트 중앙 인덱스 (34차) | COMMON | running | P1 | 요약만 | — |
| `gov.handoffs` | 작업 단위 실행 기록 (handoffs) | COMMON | running | P1 | 미게시 | — |
| `gov.verification_harness` | 검증 하네스 (verify_*.py 33개) | COMMON | running | P1 | 요약만 | — |
| `gov.reports` | 회차 보고서·요약 (GPT 협업 산출물) | COMMON | running | P0 | 요약만 | — |
| `gov.heartbeat` | 하트비트 (12시간 점검) | COMMON | running | P3 | 미게시 | — |
| `gov.explore_offer_review` | Explore 제안 갱신 (일 1회) | COMMON | running | P3 | 미게시 | — |
| `gov.my_day_preparation` | 아침 브리핑 (일시정지) | COMMON | dormant | P3 | 미게시 | — |

## 5. research 모듈 전수 분류

| 분류 | 파일 수 | 대표 파일 |
|---|---|---|
| 연구 엔진 | 36 | ar_failure_analysis.py, auto_researcher.py, common_ledger.py, daily_rerun.py, failure_hypothesis.py, generation_rotation.py … |
| 검증 | 25 | p0f_exitcode_verify_20261120.py, p0f_snapshot_probe_20261120.py, verify_18th.py, verify_429_counter_20261120.py, verify_auto_researcher.py, verify_gen16_us.py … |
| 백테스트·감사 | 15 | backtest_strategies.py, budget_gen23_audit_20261120.py, causal_fifo_isolation_20261120.py, coverage_audit_upbit.py, dev_learning_loop_audit.py, dev_repanel_experiment.py … |
| 규약·파리티 | 13 | audit_hot_movers_fix.py, freeze_eval_protocol_20261120.py, gen3_forward_parity.py, kr_gen28_promotion_gate_20261120.py, kr_gen2_probe.py, krx_calendar.py … |
| 게시·요약 | 13 | build_collab_ledger_20261120.py, build_p1_summary_20261120.py, build_p1_summary_30th.py, build_p31_summary_20261120.py, build_three_market_summary_20261120.py, build_wave_template_us.py … |
| 인프라·게시 | 7 | collab_ledger.py, export_dashboard_json.py, monitor_research_20261120.py, push_dashboard_api.py, safe_ro.py, telegram_reporter.py … |
| 감사·규명 | 7 | kr_s16_supply_audit_20261120.py, p0_23_audit.py, p0_universe_causality_20261121.py, p0b_prune_verified.py, p0c_budget_audit_20261120.py, p0d_net_audit_20261120.py … |
| 수집 | 5 | collector_audit_20261120.py, collector_dedup_audit_20261120.py, history_collect.py, kr_alpha_collect.py, upbit_backfill.py |
| 전진·Shadow | 5 | forward_tracker.py, forward_tracker_kr_s16.py, forward_tracker_kr_vcp.py, shadow_engine.py, vcp_watch.py |

전체 **126개** .py. 파일별 1줄 설명은 인덱스 JSON `modules.items[].doc`에 있다.

## 6. GitHub 공개 현황과 누락

### 6-1. 누락 문서 (파일명 기준 원격에 없음) — 51건

> 이 수치는 생성 시각 기준 스냅숏이다. 31차 독립검증 작업이 **지금 실행 중**이라 그 산출물이 먼저 파일로 생기고 회차 보고서와 함께 나중에 게시되므로, 실행 중에는 늘었다 줄었다 한다.
| 구분 | 건수 | 대표 파일 |
|---|---|---|
| 과거 회차 보고서 | 21 | `global_strategies_revalidation_20261118.md`, `gpt_10th_response_20261119.md`, `gpt_11th_final_plan_20261119.md`, `gpt_14th_evidence_report_20261119.md`, `gpt_16th_evidence_report_20261119.md`, `gpt_18th_evidence_report_20261120.md` … |
| 연구 부속 JSON | 4 | `horizon_olduni_gated_UP_20261121.json`, `horizon_predev_KR_20261121.json`, `horizon_predev_UP_20261121.json`, `horizon_predev_US_20261121.json` |
| 31차 실행 중 산출물 | 10 | `kr_forward_causality_20261120.json`, `p0_universe_causality_20261121.json`, `strategy_lab_KR_sync_ledger_20261120.json`, `timing_metrics_20261120.json`, `universe_KR_150_20261120.json`, `universe_UP_150_20261120.json` … |
| 내부 계획·분석 문서 | 16 | `DAILY_RERUN.md`, `auto_researcher_report_20261119.md`, `global_strategies_report.md`, `history_report.md`, `integrated_benchmark_20261118.md`, `kr_alpha_report.md` … |

판정: **과거 회차 보고서**는 그 회차에 GPT가 읽었을 수 있으나 현재 main 경로에 없다 — 재현·역사 추적이 끊긴다. **연구 부속 JSON**은 요약 JSON에 흡수되어 단독 게시되지 않았다. **31차 산출물**은 실행 중인 작업이 자기 회차 보고서와 함께 게시할 몫이다.

### 6-2. 공개 경로가 없는 작업

| id | 작업 | 카테고리 | 범위 | 이유 |
|---|---|---|---|---|
| `collector.posid` | 🏛️ 우체국금융개발원 이전 감시 | collector | 개인 관심사(프로젝트 범위 밖) | 개인 관심사(비시장) 영역 — 프로젝트 범위 밖으로 명시하고 상세는 게시하지 않음 |
| `collector.history_dump` | 📚 과거 대용량 데이터 수집(연구 전용) | collector | 시장 프로젝트 | 연구용 원본 DB·수집 스크립트 — 원본 DB 게시 불가 정책. 봉/행 수 집계는 latest.json·status.json에 반영. |
| `collector.upbit_backfill` | 업비트 결측 분해·증분 백필 | collector | 시장 프로젝트 | 수리 스크립트·커버리지 감사 JSON이 단독 게시된 적 없음(18차 보고서는 로컬 전용). |
| `engine.global_strategies_lab` | 세계 유명 투자기법 전수 대입 (6대) | auto_research | 시장 프로젝트 | 성적표 보고서 global_strategies_report.md(31KB)가 로컬 전용 — 게시하면 이 누락이 닫힌다. |
| `engine.online_evaluator` | 표준 온라인 평가기 | auto_research | 시장 프로젝트 | 구현 보고서 online_evaluator_report_20261119.md가 로컬 전용 — 평가 규약(budget-proto-v3)은 요약 JSON에만 노출. |
| `engine.kr_alpha` | 한국 수급·재료 분석 세트 | auto_research | 시장 프로젝트 | 분석 보고서 kr_alpha_report.md(12KB)가 로컬 전용 — 결론(20일선 눌림+외인)은 KR_S16 트랙으로 집계에 반영. |
| `engine.wave` | 파형 유사도 랭크 (WAVE v1/v2) | auto_research | 시장 프로젝트 | WAVE v1/v2 비교 문서가 로컬 전용 — 라이브 트랙 US_WAVE_RANK 성적만 monitor/latest.json에 노출. |
| `engine.history_lab` | 과거 일봉·1분봉 랩 (2년/30일) | backtest_lab | 시장 프로젝트 | 과거 랩 보고서 history_report.md(20KB)가 로컬 전용 — 과거 DB 자체도 게시 불가. |
| `infra.dbs` | 운영·연구 원본 DB (약 1.6GB) | infra_wal | 시장 프로젝트 | 원본 DB는 게시 불가. 공개되는 것은 집계(status.json·latest.json)뿐. |
| `gov.handoffs` | 작업 단위 실행 기록 (handoffs) | governance | 시장 프로젝트 | 내부 실행 기록(plan/log/verdicts) — 공개 대상 아님(정책). 작업 결과 요약은 회차 보고서로만 공개. |

Pickle 앱 자체 운영 작업(하트비트·Explore 제안·아침 브리핑)은 프로젝트 산출물이 아니므로 이 목록에서 뺐다(인덱스에는 `scope=pickle_runtime`으로 3건 남아 있다).

## 7. GPT 접근 매트릭스

### 7-1. 지금 읽을 수 있다

| 경로 | 내용 | 갱신 | 비고 |
|---|---|---|---|
| `monitor/latest.json` | 3시장 전진검증·연구 Shadow·세대 상태 요약(schema 3) | 5분 | 전진과 Shadow를 분리한 단일 화면 |
| `data/status.json` | 수집 현황·트랙 성적·최근 급등 | 5분 | 대시보드가 60초마다 다시 읽는 파일 |
| `monitor/events.jsonl` | 구현·수리·검증·세대교체 사건(append-only) | — |  |
| `monitor/telegram_reports.jsonl` | 발송 이력의 비식별 기록(시각·유형·성공 여부) | — |  |
| `coordination/gpt_to_pickle.jsonl` | GPT 의견 접수·처리 상태 | — |  |
| `coordination/pickle_to_gpt.jsonl` | Pickle의 답변·검토 요청 | — |  |
| `coordination/review_history.jsonl` | 회차 게시 이력 | — |  |
| `coordination/project_inventory.json` | 이 인덱스(작업·누락·접근 경계) | — |  |
| `coordination/project_changelog.jsonl` | 인덱스 변경 이력(append-only) | — |  |
| `research/reports/` | 회차 보고서(27·29·30·31차, 3시장 현황, horizon JSON) | — |  |
| `research/evaluation_summaries/` | 집계 요약 JSON | — |  |
| `research/test_results/` | 검증 결과 JSON | — |  |
| `index.html, README.md, .github/workflows/watchdog.yml` | 공개 대시보드 페이지·설명·외부 워치독 | — |  |

### 7-2. 읽을 수 없다 (이유 · 대체 증거)

| 경로 | 이유 | 대체 증거 |
|---|---|---|
| `research/*.py (연구 엔진·전략 원본)` | 전략 임계값·특징 계산 구현 — 공개 정책상 미게시 | 보고서·요약 JSON의 수치와 분자·분모 |
| `scheduled-actions/*/check.sh, action.md` | 수집기 구현·경보 채널·개인 관심사 정의 | monitor/latest.json 수집 현황, data/status.json |
| `원본 DB 7종(1분봉 3 + research 4, 약 1.6GB)` | 크기·정책상 게시 불가 | 봉 수·종목 수·오류·지연 집계 |
| `research/.dashboard_pat, research/.telegram_config.json, 워치독 토픽` | 자격증명·비밀 | 없음(비공개 유지) |
| `handoffs/*/plan.md, log.md, verdicts.jsonl` | 내부 실행 기록 | 회차 보고서 요약 |
| `research/history_*.md · gpt_3rd~26차 보고서 등 로컬 전용 문서` | 과거 회차 문서가 현재 main에 없음 | 없음 — §6 누락 목록 참조 |

## 8. 게시 경계 — 공개한 것과 공개하지 않은 것

공개 저장소(mju1417-max/market-dashboard)만 쓴다. 비밀키·토큰·chat id·개인정보·절대경로·원본 대용량 DB·민감한 전략 원본 코드는 게시하지 않는다(27차 Ⅲ장 research/collab/README.md).

| 대상 | 결정 | 게시 |
|---|---|---|
| 우체국금융개발원 이전 감시 | 프로젝트 범위 밖(개인 관심사)으로 명시 | 없음 — 상세·키워드·상태 모두 비게시 |
| 사용자 신원(이름·전화번호·주소·이메일) | 게시 금지 | 없음 |
| 원본 DB·절대경로 | 게시 금지 | 없음(집계 수치만) |

이 회차에서 게시 전 수행한 검사: 비밀키·토큰(`github_pat_`/`ghp_`)·텔레그램 봇 토큰·경보 토픽·전화번호·이메일·절대경로·이름·chat_id 패턴 **0건**. 원본 DB·수집기 원문·자격증명 파일은 애초에 게시 대상에 넣지 않았다.

## 9. 다음 단계

1. **정기 갱신**: 회차 보고 때마다 `python3 research/project_inventory.py --publish` 1회. 내용이 바뀌면 `coordination/project_changelog.jsonl`에 append되고, 파일 내용이 같으면 커밋하지 않는다.
2. **남은 결정(사용자 몫)**: 아래 9-1의 로컬 전용 문서를 공개 저장소로 옮길지. 공개 범위가 넓어지는 일이라 Pickle이 임의로 게시하지 않았다. 옮기면 §6-1의 누락과 해당 작업의 '공개 경로 없음'이 함께 닫힌다.
3. **31차 연결**: 실행 중인 31차 독립검증 결과가 게시되면 `engine.horizon_lab`·`engine.research_universe`·`shadow.shadow_engine` 항목의 `verification_status`가 갱신된다(현재 `owner` 필드로 그 작업을 가리킨다).
4. **구조 개선 후보**: `research/collab_ledger.py`의 409 재시도는 sha만 갱신하고 내용 병합은 하지 않는다 — append-only 파일을 두 작업이 동시에 게시하면 덮어쓸 수 있다. 병렬 회차가 늘었으므로 병합 로직이 필요하다.

### 9-1. 공개하면 닫히는 로컬 전용 문서

| 문서 | 크기 | 닫히는 공백 |
|---|---|---|
| `research/global_strategies_report.md` | 31.2KB | `engine.global_strategies_lab`(6대 기법 성적표) 근거 |
| `research/global_strategies_revalidation_20261118.md` | 33.3KB | 같은 기법의 독립 재계산 검증 근거 |
| `research/history_report.md` | 20.9KB | `engine.history_lab`(과거 2년 일봉·30일 1분봉) 근거 |
| `research/integrated_benchmark_20261118.md` | 26.2KB | 초기 3시장 벤치마크 기준선 |
| `research/kr_alpha_report.md` | 12.9KB | `engine.kr_alpha`(한국 수급·재료 결론) 근거 |
| `research/online_evaluator_report_20261119.md` | 14.2KB | `engine.online_evaluator`(평가 규약 v1) 근거 |
| `research/operational_verification_20261118.md` | 32.4KB | 운영 능력 6대 항목 실측 |
| `research/reliability_improvement_plan_20261118.md` | 56.4KB | 신뢰성 개선 계획·한계 기록 |
| `research/technical_audit_20261118.md` | 51.6KB | 10부 종합 기술 감사 원문(현재 GPT는 요약만 봄) |

과거 회차 보고서 21건은 회차별 이력이라 별도 판단이 필요하다(합계 수십 MB, 일부는 이미 지난 회차에서 읽힌 내용).

## 10. 재현

```bash
python3 research/project_inventory.py            # 조사 + 로컬 산출물
python3 research/project_inventory.py --check    # 비밀/PII 스캔
python3 research/project_inventory.py --publish  # 생성 + GitHub 게시 + raw 검증
```

---

모든 손익은 가상 체결이며 실제 주문이 아니다. 이 문서는 GPT의 독립 검토용 자료이며 투자 판단 근거가 아니다.
