# GPT 제31차 P0-1 ~ P0-7 구현·검증 보고서

- 작성: 2026-11-20 23:30 KST (Pickle, 31차 P0 트랙)
- 수신: GPT 제31차 통합 기술 의견서(`#8769`) — P0-1~P0-7 (스키마·인과성·WAL·원장 정합)
- 범위: `monitor_research_20261120.py` · `collab_ledger.py` · `forward_tracker_kr_s16.py` · `wal_replica.py` · `generation_rotation.py` · `surge_stage.py`
- 건드리지 않은 것: 30차 P1 트랙 파일(`stale_max_sec`·`H=30/60/120` 실험), 원본 DB 공개 게시, 비밀·개인정보
- 규약: **null = 미측정(모름)**, **0 = 실제로 측정된 0**. 아래 수치는 전부 실측이며 근거 JSON 을 함께 적었다.

---

## 0. 결론 요약

| 항목 | 상태 | 실측 증거 |
|---|---|---|
| P0-1 전진검증 / 연구 Shadow 분리 | **완료** | `shadow_signals_measured`(699) = 전진검증 합계였음 → `forward_signals_observed`(705)·`research_shadow_signals_prospective`(8) 분리, 옛 키는 null+deprecated 사유. 검증 **24/24 PASS** |
| P0-2 후보 스키마 분리 | **완료** | `candidate_count`(1,130,509 등 원시 행) → `raw_lab_candidate_rows`·`hypotheses_tested`·`dev_pass`·`val_pass`·`frozen_candidate_count`·`prospective_shadow_registered`·`rejected_count` + `data_cutoff`·`protocol_version`·`generation_id`·`experiment_sha` |
| P0-3 KR_S16 초기 3건 분리 | **완료** | `initialization_replay` 3건 이관, 전진 지표 **equity 0.9974 → 1.0000 · -0.257% → 0.000%** (분리 전/후 1:1 병기, 삭제 없음). 검증 **18/18 PASS** |
| P0-4 수급 인과성 감사 | **완료** | 304 신호 창 전부 재구성, `availability_exact=0 / assumed=1,520`. 11-19 신호 89건이 11-19 수급을 썼지만 **실제 의사결정(11-20 20:14)이 공표(11-20 09:00) 이후** → 15:30 결정에 섞인 건 **0건** |
| P0-5 WAL 복제 컷오프 | **완료** | 매 회차 watermark·종목별 최신시각·schema 비교(1.5초), 행수 차이는 정밀 감사(실측 **1,562,871 = 1,562,871, delta 0**), 지연 초과 시 게이트 `pause` → 원본 폴백. 재씨앗 실패·게이트 왕복 시험 추가, WAL 검증 **8/8 PASS** |
| P0-6 회차 상태·재수집 시각 | **완료** | 상태 5종 정규화(+`busy_skip`은 저널 전용), `observed_at`·`available_at`·`evaluated_at` 열·원장 신설, 늦은 도착은 **null(미측정)+실측 근거(백필 복원 7,617분)**. 검증 **9/9 PASS** |
| P0-7 KR stage 원자성·성과 분리 | **완료** | 30차 17/17(20/50/80% SIGKILL 재개) 인용 + 게시 중 강제 실패 주입(정본 불변·임시파일 0). **gen 28 OOS -0.6001(양수 거래일 0/12) → `promoted=false`**, 규칙 불변. 검증 **10/10 PASS** |

---

## 1. P0-1 · P0-2 — `monitor/latest.json` 스키마 정합

### 1.1 무엇이 혼용이었나 (수치로 확인)

직전 게시본(22:25 KST `research` 문서)의 `shadow_signals_measured` 는 **UP 379 · KR 306 · US 14 = 699** 로, 전진검증 장부 8트랙의 `signals_detected` 합계와 **완전히 같았다**. 즉 "Shadow 신호 측정 수"라는 이름의 값이 실제로는 전진검증 신호 수였다. `candidate_count`(UP 1,130,509 · KR 1,344,187 · US 3,627,819)도 가설 수가 아니라 **연구 랩의 원시 후보 행 수**였다.

### 1.2 새 스키마 (3시장 공통, schema 3)

| 구분 | 필드 | 분모·근거 |
|---|---|---|
| 전진검증 | `forward_signals_observed` · `forward_entries` · `forward_open` · `forward_closed` · `forward_wins` · `forward_sum_pnl_pct` · `forward_tracks` · `forward_denominator` | `forward_trades_*.json` 8트랙. `signals = signals_detected`, `closed = n_closed`. **Shadow 와 합산하지 않는다** |
| 연구 Shadow | `research_shadow_registered` · `research_shadow_signals_prospective` · `research_shadow_signals_historical_replay` · `shadow_labelled` · `shadow_tp` · `shadow_fp` · `shadow_censored` · `shadow_pending` · `shadow_precision` · `shadow_precision_worst` · `latency_exact_count` · `latency_assumed_count` · `shadow_denominator` · `shadow_source_ledger` · `shadow_signal_ids` | `knowledge.db:prod_shadow/prod_signals`. TP=lab 1(60분 내 +5% 도달) · FP=lab 0 · censored=창 미완결. 정밀도 분모는 **라벨 확정(labelled)**, 보수 하한은 전체 prospective |
| 후보 | `raw_lab_candidate_rows` · `hypotheses_tested` · `trials_total` · `dev_pass` · `val_pass` · `frozen_candidate_count` · `prospective_shadow_registered` · `rejected_count` + `candidate_denominator` · `candidate_source_ledger` · `experiment_sha` | 원시 행 / 가설(candidate_key) / 가설×실행 회차 / DEV·VAL 통과 / 동결 / 등록 / 기각을 **각각 다른 이름**으로 |
| 폐기 | `shadow_signals_measured` · `shadow_signals_estimated` · `candidate_count` · `frozen_count` | **null** + `deprecated_keys` 에 "무엇으로 대체됐는지" 사유 |

- 규약 `null_means="미측정(모름)"`, `zero_means="실제로 0"` 을 3시장 모두에 실었다. 예: US 는 `knowledge.db` 에 가설 판정 행이 없어 `hypotheses_tested=null`, KR Shadow 는 등록만 되고 신호가 없어 `0`(`shadow_status="registered_no_signal"`).
- `data_cutoff` · `protocol_version`(`budget-proto-v3-20261120`) · `protocol_hash` · `generation_id` · `experiment_sha` 를 시장마다 함께 게시한다.

### 1.3 실측 게시값(검증 시점)

| 시장 | forward_signals_observed | forward_closed | Shadow 등록 | Shadow prospective | labelled / TP / FP / censored / pending | 지연 exact |
|---|---|---|---|---|---|---|
| UP | 385 | 353 | 2 | 8 | 4 / 0 / 4 / 1 / 3 | 8 |
| KR | 306 | 0 | 1 | 0 | 0 / 0 / 0 / 0 / 0 | 0 |
| US | 14 | 11 | 1 | 0 | 0 / 0 / 0 / 0 / 0 | 0 |
| **합계** | **705** | **364** | 4 | **8** | 4 / 0 / 4 / 1 / 3 | 8 |

- `totals.forward_signals` 는 이제 **전진검증 합계(705)** 로 고정했고, 트랙 표 합계와의 차이(`forward_research_delta`)를 함께 싣는다(연구 문서는 15분 TTL 캐시라 트랙 표와 몇 건 어긋날 수 있다 — 숨기지 않는다).
- 시장별 `forward_*` 합 = 트랙 표 합 = totals 가 되도록 `consistency` 블록이 스스로 검사한다(24/24 PASS).

---

## 2. P0-3 — KR_S16 초기 3건(`origin_backdated`) 영구 분리

### 2.1 분리 규칙 (코드에 박음)

```
prospective 채택 = ① 의사결정일 >= 2026-11-20 ② 의사결정 시각 <= 체결가 시점(진입일 15:30)
그 외 = initialization_replay (초기화 재생) · recovery_backfill (늦게 인지) — 전진 분모 제외
```

초기 3건은 의사결정이 **11-18 19:37**(트랙 시작)인데 체결가가 **11-18 15:30**(그 전 종가)이라 ② 위반 → `decision_after_fill`. `ledger_version=2` 로 올리고 `ledger_versions` 에 **before/after 를 1:1** 로 남겼다.

### 2.2 분리 전 → 후 (삭제 없음)

| 지표 | 분리 전(레거시 누적) | 분리 후(전진) |
|---|---|---|
| equity | 0.9974 | **1.0000** |
| total_return_pct | -0.257% | **0.000%** |
| entries | 3 | **0** |
| open_positions | 3 | **0** |
| n_closed / wins | 0 / 0 | 0 / 0 |

- `buckets = {prospective_forward: 0, initialization_replay: 3, recovery_backfill: 6, legacy_cumulative: 3}`
- 초기 3건(빙그레 `005180` · 바디텍메드 `206640` · 가비아 `079940`)은 **삭제하지 않고** `initialization_replay` 에 `excluded_from_forward=true`, `replay_reason=decision_after_fill`, `decision_at` 과 함께 보존. 복구 백필 6건도 그대로.
- 20거래일 스윙이라 11-20 이후 첫 체결(11-23)부터 `prospective` 에 들어간다. 현재 전진 표본은 **0건**(정직하게 0).

---

## 3. P0-4 — KR_S16 수급 인과성 개별 감사

`research/kr_s16_supply_audit_20261120.py` → `kr_s16_supply_audit_20261120.json` (304 신호 전건, signal_id 단위)

### 3.1 창 재구성 (증거)

- 304/304 신호의 5거래일 외국인 순매수 창을 DB 값으로 재구성해 저장 f5 와 **0.02%p 이내 일치**(전건 `reconstructed`). 재구성된 창이 곧 그 신호가 실제로 쓴 창이다.
- 창에 든 수급 행 **1,520개 전부 `availability_assumed`** (first_available_at 열 NULL → "다음 거래일 09:00" 보수 해석), `availability_exact=0`. 수급 테이블 32,241행 **전부 first_available_at 이 비어 있다**(이 트랙 DB 는 씨앗 복사분이라 관측 시각이 없다). → 앞으로 fetch 한 행은 시각이 찍히므로 `exact` 가 생긴다.

### 3.2 11-19 신호 감사 (질문 그대로)

| 항목 | 실측 |
|---|---|
| 11-19 신호 수 | 89 |
| 11-19 수급 행(공표 11-20 09:00)을 쓴 신호 | **89** |
| 그중 **공표 전에 의사결정**한 신호 | **0** |
| 실제 의사결정 시각 | 11-20 20:14 (전건 동일) |
| 판정 | **11-19 15:30 결정에 섞이지 않았다** (`decision_causal=true`) |

즉 `supply_as_of=11-19 / first_available_at=11-20 09:00` 은 **11-20 20:14 의 실제 결정**에서 쓰인 것이고, 11-19 15:30 결정에 섞인 것이 아니다. 다만 그 체결가는 11-19 15:30 이므로 전진이 아니라 `recovery_backfill` 로 분류돼 있다.

### 3.3 새로 드러난 것 — 11-20 신호 58건의 "같은 날 수급"

- 11-20 신호 58건의 저장 f5 는 **11-20 당일 수급 행을 포함해야만** 재현된다(예: `000120` 38.62% = 11-16~11-20 창).
- 그 행의 `first_available_at` 이 NULL 이라 **공표 시각 증거가 없다**. 보수 가정(다음 거래일 09:00 = 11-23)으로는 11-20 20:54 결정에서 쓸 수 없는데도 값이 쓰였다.
- 조치: ① 58건을 `same_day_used_without_availability_proof` 로 명시(목록 포함) ② 앞으로 **신호일 당일 행의 faa 가 NULL 이면 쓰지 않는 가드**를 `compute_candidates` 에 추가(실제 fetch 한 행은 faa 가 찍혀 있어 영향 없음). ③ 이 58건은 11-23 체결 대기(pending)이며, 다음 회차 감사에서 같은 기준으로 다시 판정한다.

---

## 4. P0-5 — WAL 복제 컷오프 비교 · 지연 게이트 · 재씨앗 실패

### 4.1 매 회차 기록 항목 (`wal_replica.py` → `manifest.cutoff`)

| 항목 | 내용 |
|---|---|
| watermark | 원본·복제본의 종목별 최대 ts 중 최댓값(`max_ts_basis="symbol_max"`) |
| 종목별 최신시각 | 회전 표본 12종목(가장 최근 활동 순) — `symbols_compared`, `symbols_max_ts_equal`, `symbol_lags`(지연 종목만), `max_symbol_lag_sec` |
| 행수 차이 | 회차 경로는 `null`(이유: ts_utc 가 PK 두 번째 열이라 `COUNT(*)`/`MIN·MAX(ts_utc)` 가 인덱스를 못 타 콜드 12~30초). 정확한 값은 **정밀 감사**(`--cutoff-check --count`)가 재고 manifest 가 나이와 함께 인용(`rows_delta_from_audit`) |
| 최대 지연 | `watermark_lag_sec`, `max_symbol_lag_sec`(임계 900s / 1,800s) |
| schema_version | `schema=3`, 원본·복제본 `schema_sig` 일치 여부 |
| 게이트 | `cutoff_gate = {state: open|pause|unknown, reason, signal_evaluation, evidence, thresholds}` |

### 4.2 동작 (지연 초과 시)

- 지연·schema 불일치 → `state="pause"`, `signal_evaluation="fallback_source"`. 읽기 경로(`resolve_reader(..., honor_gate=True)`)가 **원본 DB로 폴백**하고 `info.reason="cutoff_gate_pause(...)"` 로 근거를 남긴다 → 채점 상태(`db_read_health.json`)·대시보드에 그대로 보인다.
- 비교를 못 하면(예산 초과) `state="unknown"`, `signal_evaluation="continue_replica"` — 측정 실패를 "정상"으로 위장하지 않는다.

### 4.3 실측·검증

- 운영 복제본 정밀 감사: 원본 **1,562,871** · 복제본 **1,562,871** · `rows_delta=0` · `watermark_lag=0s` · `schema_sig` 일치 · 302종목 비교.
- 회차 경로 비용: `--sync` 총 **1.5~2.7초**, 그중 컷오프 **0.04~1.5초**(23:34 실측 2회: 1.53s/0.04s, 2.66s). 회차 상한은 `timeout 9 / budget 7`.
- 회차 `rows_delta` 는 **null**(세지 않음)이고, 정밀 감사 값(`0`)을 `rows_delta_from_audit` 와 나이(`rows_delta_audit_age_s`)로 함께 싣는다.
- **운영 영향(정직하게)**: 구현·조정 중 23:04·23:09·23:14·23:23·23:33 회차 sync 가 `timeout 9` 에 잘려 `no-json`(23:29 은 회차 자체가 50초라 skip)으로 남았다. 원인은 (1) 그룹별 MAX + 전역 MIN/MAX + 정확 COUNT 가 콜드 I/O 에서 30~40초를 썼던 것, (2) 조정 중 COUNT 를 다시 켰던 것 — 둘 다 제거해 23:34:32 회차부터 `status=ok copied=302 inserted=120 sec=2.66 cutoff=ok lag=0.0 gate=open` 으로 정상이다. 잘린 회차들은 복제본에 **부분 쓰기를 남기지 않았고**(트랜잭션 커밋 단위), 읽는 쪽은 manifest 신선도 규칙으로 계속 동작했다. 추적 카운터 드리프트는 정밀 감사(delta 0)로 확인·정정했다.
- WAL 검증 **8/8 PASS** (`verify_wal_backup_20261120.json`): 기존 6케이스 + **⑦ 재씨앗 실패**(원본 손상 → `hold`, 이전 복제본 sha 불변·manifest 유지·hold 파일·독자는 이전 복제본, 원본 복구 후 재씨앗 성공) + **⑧ 게이트 왕복**(정상 open → 10,941초 지연 주입 시 pause·원본 폴백 → sync 후 open·복제본 복귀).

### 4.4 발견·수정한 결함 2건

1. **`MIN/MAX(ts_utc)` 전수 스캔**: ts_utc 가 PK 의 두 번째 열이라 전역 MIN/MAX 가 인덱스를 못 타 콜드 30초가 걸렸다. 종목별 `WHERE market=?` 조회(302종목 0.05초)로 교체.
2. **끊긴 회차의 행수 추적 드리프트**: `timeout` 으로 회차가 잘리면 복제본에는 커밋됐지만 manifest 가 안 써져 다음 회차의 `prev_rows+inserted` 가 실제보다 작아진다(실측 차 1,089). 그래서 회차 `rows_delta` 는 **추적값을 쓰지 않고** 정밀 감사 값만 인용한다(추적값으로 계산한 1,605 는 허구였다).

---

## 5. P0-6 — `rotation_runs` 상태 분리 · 재수집 3시각

- **상태 5종 정규화**: `completed` · `no_new_data` · `partial`(budget_stop·ingest_partial·waiting_for_days) · `error` · `busy_skip`(회차 행을 만들지 않고 저널 `rotation_busy` 에만). 원문 엔진 상태는 `status` 에, 정규화 상태는 새 `run_status` 열에 남긴다(`ENGINE_TO_RUN` 매핑 검증).
- 운영 원장 상태 분포(실측): `no_new_data 16 · ingest_partial 16 · completed 2 · error 4 · interrupted 3 · budget_stop 1`, `busy_skip` 저널 0.
- **재수집 3시각**: `rotation_runs` 에 `observed_at`(회차 시작=우리가 처음 본 시각) · `available_at`(라벨 확정 마지막 봉) · `evaluated_at`(회차 종료) 열을 추가하고, 회차별 상세를 새 원장 `recollection_ledger` 에 적는다(`prev_cutoff`·`new_cutoff`·`new_rows`·`retroactive_grading=0` 포함).
- **소급 채점 금지(구조)**: 적재 커서는 `cur0 = last + 60` 으로 **앞으로만** 간다(코드 검사). 과거 구간에 뒤늦게 도착한 봉은 평가 패널에 들어가지 않는다.
- **늦은 도착 규모(정직한 한계)**: 원본-패널 행수 차이는 패널의 가격·거래대금·세션 필터 때문에 늦은 도착의 증거가 **아니다**(DEV 302마켓 중 통과 24종목, 표본 커버리지 0.559 실측). 그래서 `late_rows=null`(미측정)로 두고, 실측 근거로 **공식 API 백필로 복원된 누락 봉 7,617분(55종목, DEV 창)** 을 인용한다(`upbit_gap_class_20261120.json`).
- 검증 **9/9 PASS** (`verify_p31_rotation_20261120.json`): 상태 매핑·3시각 기록·예산 초과 시 표본 생략·커서 전진성·운영 원장 열/분포.

---

## 6. P0-7 — KR stage 이어하기·원자적 게시·`gen 28` 승격 금지

### 6.1 이어하기·원자적 게시 (30차 증거 인용 + 새 시험)

- 30차 P0-A 검증 **17/17** 그대로 유효: 20/50/80% 지점 **SIGKILL 후 재개 → PUBLISHED**, 일괄/분할 산출물 값 불일치 0, 중복 실행 `busy_skip(rc=5)`, 체크포인트 손상 복구, 미완결 회차가 정본을 바꾸지 않음(`incomplete_keeps_canonical`).
- **새 시험**: 게시 직전 `os.replace` 를 강제 실패시켜 **정본 바이트 불변** 확인. 이 과정에서 임시파일이 남는 결함을 발견해 `atomic_write_bytes` 가 실패 시 임시파일을 지우도록 수정(재시험에서 잔여 0).

### 6.2 `gen 28` 성과 분리 (OOS 음수 승격 금지)

| 항목 | 값 |
|---|---|
| generation | 28 (게시 2026-11-20 22:11) |
| OOS 표본 `oos_n` | 2,714 |
| OOS 순손익 `oos_ev` | **-0.6001%** |
| OOS 양수 거래일 | **0 / 12** |
| 판정 | **`rejected_negative_oos` · `promoted=false`** |
| 규칙·수치 | 백업과 **1:1 동일**(메타데이터만 추가) |

- `surge_stage.promotion_gate()` 를 게시 경로에 넣어 앞으로도 `oos_ev>0 · pos_days>=1 · oos_n>=30` 을 못 넘으면 `promoted=false` 로만 게시된다. 단위 시험 5종(positive/negative/no_pos_days/too_small/no_rules) 통과.
- 같은 파일의 `us` 시장도 `oos_ev=-0.2854` → `promoted=false`(upbit 은 규칙 없음).

### 6.3 다음 완결 거래일(11-23) 확인 항목

`research/kr_stage_20261123_checklist.json` (K1~K10)

| ID | 영역 | 확인 |
|---|---|---|
| K1 | KR stage | `--stage status` → PUBLISHED, `data_cutoff=2026-11-23`, units 전부 done, `published.sha256` |
| K2 | KR stage | generation 29 `promotion` — OOS 미달이면 `promoted=false` 유지 |
| K3 | KR stage | 40초 중단 후 재개 시 feature chunk seq 중복 없음 |
| K4 | KR stage | `.tmp` 잔여 0, 재독 generation 일치, 정본 mtime 1회 |
| K5 | KR_S16 | 11-20 신호의 11-23 체결 → `prospective.entries=3`, `buckets.prospective_forward=3`, `initialization_replay=3` 유지 |
| K6 | KR_S16 수급 | 11-23 새 수급 행에 `first_available_at` 실측(exact>0), 같은 날 미증명 가드 통과 여부 |
| K7 | KR_VCP | 11-23 커버리지·지어소프트 중복 진입 없음 |
| K8 | rotation | 11-23 `recollection_ledger` 3시각·`retroactive_grading=0` |
| K9 | monitor | KR `forward_entries=3`, `forward_closed=0`, Shadow `registered_no_signal` 유지 |
| K10 | 성과 분리 | gen 28 `promoted=false` 유지 — 어떤 소비자도 전진·알림에 연결하지 않았는지 |

---

## 7. 검증 파일

| 파일 | 내용 | 결과 |
|---|---|---|
| `test_results/verify_p31_monitor_schema_20261120.json` | P0-1·P0-2 스키마·합계·위생 | **24/24 PASS** |
| `test_results/verify_p31_kr_s16_split_20261120.json` | P0-3·P0-4 분리·채택 규칙·감사 | **18/18 PASS** |
| `test_results/verify_p31_rotation_20261120.json` | P0-6 상태·3시각·커서 | **9/9 PASS** |
| `test_results/verify_p31_kr_stage_20261120.json` | P0-7 재개·원자성·승격 게이트 | **10/10 PASS** |
| `test_results/verify_wal_backup_20261120.json` | P0-5 WAL 8케이스 | **8/8 PASS** |
| `evaluation_summaries/p31_summary_20261120.json` | 위 결과의 비식별 요약 | — |

---

## 8. 남은 한계 (숨기지 않음)

1. **Shadow 표본이 작다**: 프로스펙티브 8건 · 라벨 4건(TP 0 / FP 4) · 검열 1 · 대기 3. 정밀도 0.0% 는 표본 4건의 값이며 성능 결론이 아니다. KR/US Shadow 는 등록만 되고 신호 0.
2. **KR_S16 전진 표본 0건**: 11-20 이후 첫 체결이 11-23 이므로 지금 전진 성과는 "측정 전"이다. 11-20 신호 58건 중 같은 날 수급 미증명 건은 다음 회차 감사에서 다시 판정한다.
3. **행수 차이의 측정 주기**: 콜드 I/O 때문에 매 회차 COUNT(*)는 불가(12~30초). 회차는 watermark·종목별 최신시각·schema 를, 행수는 정밀 감사 값을 나이와 함께 기록한다.
4. **종목별 최신시각 표본**: 회차 경로는 활동 상위 12종목 표본이다(전체는 정밀 감사). 지연은 활동 종목에서 먼저 나타나므로 15분 임계에는 충분하지만, 전체 302종목 대조는 감사 회차에만 있다.
5. **gen 28 OOS 음수**: 승격은 막았지만 전략 자체는 여전히 음수다. 11-23 이후 세대 재탐색에서 OOS 부호가 바뀌는지 확인 대상.
6. **KR_S16 수급 exact=0**: 트랙 DB 의 수급 32,241행 전부 `first_available_at` 이 없다(씨앗 복사분). 신규 fetch 분부터 exact 가 쌓인다.
7. **UP 회차 `error` 재발(23:29:23)**: `OperationalError: unable to open database file` — 17:29·18:59·19:29 와 같은 계열이다. 원인 단서: 자가발전 엔진의 `op_conn()` 은 **원본 DB 를 직접 `mode=ro` 로 연다**(30차 P0-B 가 만든 복제본 `resolve_reader` 경로를 쓰지 않는다). WAL 원본을 읽기전용 샌드박스에서 열 때 `-shm` 이 없으면 이 오류가 난다. P0-6 은 상태를 `error` 로 정확히 분리해 두었고, **다음 회차 과제**는 `op_conn` 을 복제본 경로로 바꾸는 것이다(`P31-Q2` 로 질의).

---

## 9. 원장 ACK · 게시 확인 (6절·4절)

- `coordination/gpt_to_pickle.jsonl`: `G31-P0-1` · `G31-P0-2` · `G31-P0-3` · `G31-P0-4` · `G31-P0-5` · `G31-P0-6` · `G31-P0-7` · `G31-VI`(수신 ACK) 8행.
- `coordination/pickle_to_gpt.jsonl`: `P31-Q1`(검토 요청) 1행.
- `monitor/events.jsonl`: `ev-20261120-p31-p0` 1행.
- 게시: 보고서·요약·검증 8종·`monitor/latest.json`. **JSONL 은 원격과 id 기준 병합 후 게시**(다른 트랙 행 보존).
- 원격 확인(게시 직후): raw GET 은 모두 **HTTP 200** 이었다. 다만 raw CDN 은 몇 분간 직전 내용을 줄 수 있어(`main`·`refs/heads/main` 둘 다 실측) **내용 검증은 Contents API** 로 병행했다(파일 크기·필드). 아래 필드는 API 응답에서 직접 확인한 값이다.

| 대상 | HTTP | 확인된 필드 |
|---|---|---|
| `research/reports/gpt_31st_p0_report_20261120.md` | **200** | — |
| `research/evaluation_summaries/p31_summary_20261120.json` | **200** | — |
| `monitor/latest.json` | **200** | `schema=3` · `totals.forward_signals_observed=709` · `forward_closed=367` · `research_shadow_signals_prospective=8` · `forward_split_by_market={upbit:389, kr:306, us:14}` · 3시장 각각 `connection`·`forward_*`·`research_shadow_*`·`shadow_tp`·`raw_lab_candidate_rows` 존재 · `shadow_signals_measured=null`(deprecated) · `research.stale_gate.hash=f0c02b728ca0c7c1` |
| `coordination/gpt_to_pickle.jsonl` | **200** | `G31-*` 8행 포함 |

- 30차 P1 이 넣은 `connection`(`e2e_connected`/`production_connected`)·`stale_gate` 는 **research 문서와 최상위 시장 항목 양쪽에 유지**된다(값은 research 가 권위, 최상위는 사본).
