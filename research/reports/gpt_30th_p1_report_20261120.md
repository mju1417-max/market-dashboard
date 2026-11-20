# GPT 제30차 P1 후속 보고서 — KR·US 운영 자가발전 · US stale 게이트 · 3시장 H=30/60/120

- **작업 ID**: `30th-round / P1 (⑥ P1-E · ⑦ P1-C/P1-B · ⑧ P1-F)`
- **작성·실행**: 2026-11-20 22:20 ~ 23:10 KST (모든 수치는 이 구간 실측)
- **데이터 기준**: 업비트 `2026-11-20 13:17 UTC`(22:17 KST) · 한국 `2026-11-20 15:44 KST` · 미국 `2026-11-20 22:06 KST`. 지식 원장(rotation·prod)은 22:52 KST 스냅숏.
- **공개 저장소**: 이 보고서와 비식별 JSON 요약만 게시했다. 비밀키·PAT·개인정보·원본 DB·절대경로는 넣지 않았다.
- **트랙 경계**: P0-A(한국 `surge_miner --stage`·`strategy_lab_KR`·`KR_S16`·`KR_VCP`)와 P0-B(WAL 복제·잠금 계측)의 산출물은 **읽기만** 했다. 이번 P1은 `research/production_connected.py`(신규)·`research/horizon_lab.py`(신규)·`research/research_universe.py`(신규)·`research/signal_timing_metrics.py`(신규)·`shadow_engine.py`(시장 일반화)·`forward_tracker.py`(stale 게이트)·`monitor_research_20261120.py`(연결 상태)·`🔬-자가발전-패턴-연구/check.sh`(단계 추가)를 다룬다.

---

## 0. 결론 요약

| # | 요구 | 결과 | 실측 증거 |
|---|---|---|---|
| ⑥ | KR·US **운영 자가발전** 연결 + `production_connected` 구분 | **연결·완주.** 30분 액션 회차마다 KR→US 순환으로 한 시장 한 단계가 실제로 돈다. **KR·US 모두 자기 세대를 처음 생성**(KR-G001 91,704 DEV행 · US-G001 102,882 DEV행, 각 272 격자 평가·**통과 0**)하고 60종목 패널을 유지한다. `e2e_connected`와 `production_connected`를 `monitor/latest.json` `research.markets.<시장>.connection`에 분리 표기 | §1 |
| ⑥ | **진짜 미래 Shadow**(historical replay 아님) | **가동.** 등록 **이후** 봉에서만 신호를 만들고, 실제 수신·판단 시각과 봉 마감 대비 지연을 기록. UP 기준선 참조 Shadow에서 **8신호·라벨 4·TP 0·FP 4·상승폭 MAE 4.703%p·순손익 −2.97%/건** 실측 | §1-4 |
| ⑦ | US `stale_max_sec` 게이트 & 늦은 봉 분리 | **완료.** 사전 동결 420초, 초과 시 신호 대신 `expired`(누락 수치 보존). 대안 규칙은 `US_GEN16_STALEFILL`·`US_WAVE_RANK_STALEFILL` **미래 전용** ID로만 등록. 검증 **13/13 PASS** | §2 |
| ⑦ | `true_early_TP`·`late_event`·`no_event`·`decision_to_event_seconds`·`entry_to_event_seconds`·`execution_net` 분리 | **완료.** `US_GEN16` = 사전포착 **0** / 늦은 관측 **3** / 미도달 **10**(판단→사건 평균 **−282초**) · `US_WAVE_RANK` = 늦은 관측 1(−46초) · `UPBIT_GEN3` = 사전포착 11 / 늦은 2 / 미도달 346(+1,809초) | §2-3 |
| ⑧ | 150종목 컷오프 고정 유니버스 + H=30/60/120 독립 연구 + Bonferroni 7.4e-5 | **완료.** 3시장 × 3지평 실측. **UP H=60만 게이트 통과 가설 5개**(최상 `ret60≥+1.71% & vcontract≥1.278`, DEV 45.3%·VAL 35%·HOLD 60%·+1.04%). **KR 0/675, US 0/675**(모든 지평). 신규 가설군은 DEV 통과 후 VAL/HOLD에서 무너져 **탐색적 기록**으로만 남김 | §3 |
| — | 테스트·스케줄 | `verify_stale_gate` **13/13** · `verify_production_connected` **10/10** · `verify_horizon_lab` **6/6** · `🔬` `scheduled-action check` **quiet(exit 1) 25초** | §4 |

한 줄 결론: **"3시장이 같은 계약으로 운영 스케줄에서 실제로 돌고 있다"는 사실은 이제 증거로 말할 수 있게 됐고, 성과는 여전히 UP H=60 한 곳에서만 보인다.** KR은 신호는 있으나(ACCEL 정밀도 17.5%, 리프트 13.9) 왕복 0.41% 비용 뒤에서 전부 음수, US는 10세션 표본에 +5% 사건이 27~53건뿐이라 발견 자체가 불가능하다.

---

## 1. 작업 ⑥ (P1-E) — KR·US 운영 자가발전 & 진짜 미래 Shadow

### 1-1. `e2e_connected` 와 `production_connected` 의 정의 (엄격 구분)

| 구분 | 뜻 | 이번 회차 증거 |
|---|---|---|
| `e2e_connected` | 29차에서 3시장이 같은 7계층 계약(`MarketAdapter`…`FailureLedger`)으로 **격리 디렉터리에서 한 번** end-to-end 순환을 완주했다 | `research/.p1_e2e_test/` + `verify_three_market_e2e.py` 28/28 (기존 유지) |
| `production_connected` | **운영 스케줄(30분 액션)에서 그 시장의 세대교체가 실제로 실행된 기록**이 있고, Shadow 관측 단계도 실제로 호출됐다 | `rotation_runs`(KR 8회·US 7회) + `prod_runs`(KR 9·US 7) + `prod_shadow`(KR 1·US 1) |

두 값은 `monitor/latest.json` → `research.markets.{upbit,kr,us}.connection` 에 **별도 필드**로 실린다. `production_connected` 는 `rotation_runs` 행 존재 + `prod_runs` 호출 기록 둘 다로 판정하며, 신호 수(`production_signals`)는 0이어도 "연결됨"과 "측정됨"을 섞지 않는다.

### 1-2. 운영 스케줄에 어떻게 붙였나 (60초 상한 준수)

`scheduled-actions/🔬-자가발전-패턴-연구/check.sh` (30분 주기, 원래 UP 세대교체 1회):

```
① 0~4초 지연(다른 5분 파이프라인과 겹침 회피)
②-0 production_connected.py --stage next --budget 11   (timeout 14)   ← 신규
     KR → US 순환으로 한 시장 한 단계
②   generation_rotation.py --market upbit --budget 28  (timeout 38)   ← 기존(예산 32→28)
③ 하트비트 1줄에 prod=시장/단계/상태 sig=.. exp=.. settled=.. 추가
```

- 실측 회차: **25~26초**(22:52 `prod=KR/shadow_scan/ok`). 최악의 경우에도 14+38+요약 < 60초.
- UP 예산을 28초로 줄인 뒤에도 UP 회차는 `no_new_data`/`ingest` 정상(22:29·22:47·22:52 모두 `ok`).
- 모든 단계는 커서(`prod_shadow.last_scanned_ts`·`symbol_cursor`, `next_stage`)를 원장에 남겨 예산에 잘려도 다음 회차가 이어받는다.
- `scheduled-action check` 결과: **quiet(exit 1)**, 25초. `check.sh`는 어떤 경우에도 AI를 깨우지 않는다.

### 1-3. 단계 설계 (`research/production_connected.py`)

| 단계 | 하는 일 | 경계 |
|---|---|---|
| `rotate` | `generation_rotation.rotate()` — 그 시장 패널 적재 → DEV/HOLD 자동 분할 → 272 격자 평가 → 동결 → Shadow 등록 → 실패 원장 | 운영 1분봉 DB `mode=ro`. 유니버스는 **컷오프 고정 상위 60종목**(`research_universe`)으로 제한해 30분 회차 예산 안에 들어오게 함 |
| `shadow_sync` | 동결 후보 + 사전 등록 가설을 `prod_shadow`로 등록(등록 시각 이후만 관측) | 후보 0건이면 0건 그대로(기준 완화 없음) |
| `shadow_scan` | 등록 이후 새 봉에서 조건 통과 봉을 찾아 신호 기록 | **실제 실행 시각**을 `first_received_ts`·`decision_at`으로 저장. 봉 마감 대비 `staleness_sec` 계산. US는 420초 초과 시 `expired` |
| `shadow_settle` | 봉 마감+60분이 **실제로** 지난 신호만 라벨 확정 | 세션 경계를 넘으면 `censored`(프리/애프터 가격으로 +5%를 세지 않음). 데이터 공백이면 `data_missing` censored |
| `shadow_evaluate` | 등록 시 고정한 검증계획(최소 20라벨·2일, 정밀도 0.15 미만·평균 −0.50% 미만이면 기각)으로 유지/기각 | **자동 기각만**. 승격·알림 없음 |

`shadow_engine.py` 는 UP 전용(`market` 컬럼·ISO 시각·`compute_feats("UP")`)이어서 3시장으로 일반화했다: 시장별 DB·심볼 컬럼·시각 형식(`text|int`)·세션 게이트(KR·US 정규장만)·비용(KR 0.41%) 분리. 이 수정 없이 KR 후보를 등록하면 UP DB를 읽어 **다른 시장 가격으로 KR 신호를 만드는** 사고가 날 수 있었다(등록 후보 0건이라 아직 발현 전이었음).

### 1-4. 실측 — KR·US 실제 가동, 그리고 UP 진짜 미래 Shadow

| 시장 | 세대교체(회차) | 마지막 상태 | 패널 | 격자 | 동결 | 운영 Shadow 등록 | 측정 신호 | 라벨 | TP/FP | 상승폭 MAE | 순손익/건 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| UP | 13회 `no_new_data` + 기존 | `no_new_data` gen 3 | 106,931행·76종목 | 0(변화 없음) | 0 | **2** (사전등록 가설 + 기준선 참조) | **8** | 4 | 0 / 4 | **4.703%p** | **−2.97%** |
| KR | **8회**(1 완주·7 적재) | **`completed` gen 1** | **106,907행·60종목** | **272** | 0 | 1 | 0 | 0 | — | — | — |
| US | **9회**(8 적재·1 완주) | **`completed` gen 1** | **127,954행·60종목** | **272** | 0 | 1 | 0 | 0 | — | — | — |

읽는 법(정직):

- **KR은 처음으로 운영 원장에 자기 세대(generation 1)를 만들었다.** 272개 격자를 실제 KR 데이터(11-10~11-20, 60종목)로 평가했고 **통과 0건**이다. 이는 29차 격리 e2e(KR DEV 통과 0)와 같은 방향의 결과이며, 이제는 **운영 스케줄에서** 반복된다.
- **US도 완주했다**(9회). 60종목 백필이 회차당 10~21k행씩 진행돼 127,954행이 됐고, 22:59 회차에 `US-G001`(DEV 11-10~11-17 · HOLD 11-18~11-19 · DEV 102,882행)이 생성되며 **272 격자 평가·통과 0**으로 끝났다. 적재 중(`ingest_partial`·`budget_stop`)에는 단계 순환이 `rotate`를 건너뛰지 않도록 고정해, 백필이 다른 단계 뒤로 밀리지 않게 했다.
- 즉 **세 시장이 모두 운영 원장에 자기 세대를 가진다**: UP-G003(기존) · KR-G001 · US-G001. 셋 다 272 격자에서 통과 0건이다 — 29차 격리 e2e에서 UP만 후보가 나왔던 것과 같은 방향이며, 이제는 **운영 스케줄에서** 반복된다.
- **UP의 8신호는 '첫 관측 catch-up'** 이다. 기준선 참조 Shadow(`ZVOL2.5`)를 등록하면서 시작 시각을 90분 과거로 두었고, 그래서 `staleness_sec` 이 3,300~5,100초로 찍혔다. 이 값은 숨기지 않고 그대로 남겼다: **이 8건은 사전포착이 아니라 늦은 관측**이며, 라벨·순손익만 유효하다. 이후 회차부터는 30분마다 새 봉만 본다.
- 사전 등록 가설 `PREREG_H60_MOMVOL`(UP H=60 최상 가설)은 UP·KR·US 모두에 등록했고 아직 신호 0건이다(조건이 드물다). KR·US는 장 마감 시간대라 오늘 신호가 없는 것이 정상이다.

---

## 2. 작업 ⑦ (P1-C & P1-B) — US `stale_max_sec` 게이트 & 사전포착/손익 분리

### 2-1. 게이트는 "사전 동결" 값으로만 움직인다

`research/stale_gate_freeze_20261120.json` (해시 `f0c02b728ca0c7c1`):

| 항목 | 값 |
|---|---|
| 값 | 미국 `stale_max_sec = 420` (US_GEN16 · US_WAVE_RANK) |
| 근거 | 5분 수집 주기 300초 + 수집·계산 여유 120초. 이보다 오래된 봉은 "지금 진입 가능한 가격"이 아니다 |
| 동결 시각 | 2026-11-20 22:30 KST |
| 변경 규칙 | 관측 중 변경 금지. 바꾸려면 새 전략 ID로 등록하고 미래 데이터로만 검증 |

`forward_tracker.py` 의 `_scan_symbol` 은 신호 후보 봉마다 `staleness = 판단시각 − (봉시각+60)` 을 계산해 420초를 넘으면:

1. **신호를 만들지 않는다**(`pending`/`open`/`closed` 어디에도 안 들어감),
2. `state.expired[]` 에 `would_be`(그때의 조건값·신호봉 종가·기준가)와 `staleness_sec`·`stale_max_sec` 를 남긴다 — **누락 수치는 보존**,
3. `stats.expired_stale` 을 올리고, 더 오래된 봉은 더 stale 하므로 그 종목 스캔을 끝낸다.

정상 신호에는 `bar_close_epoch · first_received_epoch · analysis_available_epoch · decision_epoch · entry_available_epoch · staleness_sec · stale_max_sec` 를 **모두** 남긴다(봉 종료 → 최초 수신 → 분석 가능 → 진입 가능 시각).

### 2-2. 대안("최신 봉 재평가")은 새 전략 ID로만

`research/stale_fill_strategies_20261120.json`:

| 전략 ID | 상태 | 라이브 연결 | 검증 시작 | 가설 |
|---|---|---|---|---|
| `US_GEN16_STALEFILL` | `future_only` | **false** | 2026-11-20 22:30 | 늦은 관측을 최신 봉 재평가로 구제하면 순손익이 개선되는가 |
| `US_WAVE_RANK_STALEFILL` | `future_only` | **false** | 2026-11-20 22:30 | 466초 지연 사례(SEAT)를 재평가로 구제할 수 있는가 |

두 ID는 `TRACKS` 에 없고 어떤 스캔에도 연결되지 않는다(테스트 S4가 이를 검사). 등록 시각 이후 신규 데이터로만 검증한다.

### 2-3. 지표 분리 실측 (`signal_timing_metrics.py`)

| 트랙 | 표본 | **true_early_TP** | **late_event** | **no_event** | censored | decision→event 평균 | entry→event 평균 | execution_net | 상승폭 MAE |
|---|---|---|---|---|---|---|---|---|---|
| `US_GEN16` | 13 | **0** | **3** | 10 | 0 | **−282.3초** | −269.6초 | +1.61%/건 | 4.62%p |
| `US_WAVE_RANK` | 1 | **0** | **1** | 0 | 0 | **−46.0초** | −40.8초 | +4.90% | 8.88%p |
| `UPBIT_GEN3` | 359 | 11 | 2 | 346 | 0 | +1,809.4초 | +2,049.0초 | −0.22%/건 | 3.46%p |

- `ex_post_hit_rate`(사건이 있었는가) ≠ `early_capture_rate`(사건 **전에** 판단했는가). US_GEN16은 ex-post 3/13(23.1%)이지만 **사전포착 0/13(0%)** 이다. 29차 결론을 지표로 재확인했다.
- `US_WAVE_RANK` 1건(SEAT)은 판단이 사건보다 46초 늦었다 → **'늦은 관측'** 이며 사전포착 사례로 인용 금지.
- `UPBIT_GEN3` 는 사전포착 11건이 있지만 평균 순손익이 −0.22%/건이다(사전포착 ≠ 수익). 두 지표를 섞으면 안 되는 이유를 스스로 보여준다.

---

## 3. 작업 ⑧ (P1-F) — 3시장 H=30/60/120 독립 연구 & 150종목 유니버스

### 3-1. 유니버스 — 컷오프 시점 고정, 미래 정보 0

`research/research_universe.py` (검증 P1: 컷오프 이후 거래대금 폭증 봉을 넣어도 순위·해시 불변):

| 시장 | cutoff | 후보 풀 | 150종목 | 제외 | 해시 | 소스 |
|---|---|---|---|---|---|---|
| UP | 2026-11-20 22:23 KST | 302 | 150 | 가격 0 · 봉부족 0 | `79a5bbe513c7e4bc` | `upbit_1m.db` |
| KR | 2026-11-20 15:30 KST | 3,091 | 150 | 가격 10 · 봉부족 55 | `50da33668c026f54` | `kr_1m.db` |
| US | 2026-11-21 06:00 KST | 1,998 | 150 | 가격 52 · 봉부족 26 | `b094c9ce8873eca8` | `us_1m.db` |

- 순위 = 컷오프 **이전** lookback(UP 10일·KR/US 12일) 누적 거래대금(close×volume). 컷오프 이후 봉은 한 행도 읽지 않는다.
- 가격 하한(10원·1,000원·$1)은 컷오프 시점 종가로 판정. 상장 생존정보(지금 살아있는지)는 쓰지 않았다 — 컷오프 시점에 봉이 있었다는 사실만 사용.
- 지평 실험은 이 150종목·최근 10세션(UP 200,703행 / KR 82,720행 / US 84,023행, 5분 격자)에서 수행.

### 3-2. 지평별 분리 — 미래 가격·세션 경계·중복 사건 purge

| 시장 | H | 원 표본 | purge 후 | 제거 | censored | +5% 사건 | 사건률 |
|---|---|---|---|---|---|---|---|
| UP | 30 | 200,703 | 53,598 | 147,105 | 43,283 | 424 | 4.11% |
| UP | 60 | 200,703 | 29,820 | 170,883 | 25,899 | 348 | 8.88% |
| UP | 120 | 200,703 | 16,004 | 184,699 | 14,538 | 230 | 15.69% |
| KR | 30 | 82,720 | 14,308 | 68,412 | 1,462 | 170 | 1.32% |
| KR | 60 | 82,720 | 7,355 | 75,365 | 1,280 | 188 | 3.09% |
| KR | 120 | 82,720 | 4,031 | 78,689 | 1,231 | 198 | 7.07% |
| US | 30 | 84,023 | 14,735 | 69,288 | 1,147 | 27 | 0.20% |
| US | 60 | 84,023 | 7,502 | 76,521 | 1,141 | 42 | 0.66% |
| US | 120 | 84,023 | 4,143 | 79,880 | 1,137 | 53 | 1.76% |

- **purge**: 같은 종목에서 라벨 창(지평 H분)이 겹치는 표본은 시간순 첫 표본만 남긴다(겹치는 관측은 독립 표본이 아니다). 지평이 길수록 더 많이 제거된다.
- **세션 경계**: US는 정규장(09:30–16:00 ET)만, KR은 09:00–15:30만. H가 세션 종료를 넘으면 `censored`(프리/애프터 가격으로 +5%를 세지 않음).
- 미래 가격은 지평마다 **그 지평의** 고가·종가만 쓴다(H=30 결과로 H=120을 채우지 않음).

### 3-3. 675가설 격자 — Bonferroni α = 0.05/675 = **7.407e-5** (FDR 미사용)

| 시장 | H | 시험 | 최소표본 미달 | α 통과(게이트 전) | **게이트 통과** | VAL 통과 | HOLD 통과 |
|---|---|---|---|---|---|---|---|
| UP | 30 | 675 | 329 | 9 | **3** | 0 | 3 |
| UP | 60 | 675 | 452 | 26 | **5** | 4 | 3 |
| UP | 120 | 675 | 569 | 8 | **1** | 1 | 1 |
| KR | 30/60/120 | 675 | 285/376/463 | 23/15/3 | **0 / 0 / 0** | 0 | 0 |
| US | 30/60/120 | 675 | 311/384/455 | 96/77/51 | **0 / 0 / 0** | 0 | 0 |

게이트(α 통과만으로 승격하지 않기): 라벨 ≥50 · TP ≥3 · 리프트 ≥1.15 · 비용후 순손익 > 0. **표본이 작으면 기저율이 0에 가까워 p가 기계적으로 작아지기 때문에**(US 사건 27건) 이 하한을 두었다. 그 결과 US의 "α 통과 96개"는 전부 표본 부족/비용후 음수로 걸러졌다.

**UP H=60 상위(전부 VAL·HOLD까지 통과)**:

| 가설 | DEV 표본/TP/정밀도/p | DEV 순손익 | VAL p / 순손익 | HOLD p / 순손익 |
|---|---|---|---|---|
| `ret60≥+1.71% & vcontract≥1.278` | 53 / 24 / **45.3%** / 1.49e-14 | +0.38% | 0.0050 / +0.43% | 4.3e-7 / **+1.04%** |
| `ret60≥+1.71% & vcontract≥1.278 & ret60_btc≥+1.71%` | 53 / 24 / 45.3% / 1.49e-14 | +0.40% | 0.0010 / +0.77% | 4.3e-7 / +1.04% |
| `vcontract≥1.278 & ret60_btc≥+1.71%` | 54 / 21 / 38.9% / 2.3e-11 | +0.22% | 0.020 / −0.01% | 4.3e-7 / +1.18% |

읽는 법: **변동성 확장(vcontract 상위) + 60분 모멘텀**이 UP에서 가장 재현성 있는 선행 조건이다. 29차 UP H=60 최상 가설(`ret60≥1.63% & range_pos≤0.469 & z_vol≤−0.365`)과는 다른 방향(압축이 아니라 확장)이며, 표본·게이트·유니버스가 달라진 결과다. HOLD 표본은 10건/일 상한으로 20건뿐이라 **승격 근거가 아니라 관찰 근거**다.

### 3-4. 기준선(무작위·ACCEL·ZVOL2.5) — 같은 지평·같은 선택 규칙

| 시장 | H | ACCEL DEV 정밀도/순손익/p | ACCEL HOLD 순손익 | ZVOL DEV 정밀도/순손익/p | ZVOL HOLD 순손익 |
|---|---|---|---|---|---|
| UP | 30 | 24.0% / −0.37% / 1.9e-8 | +0.29% | 23.3% / +0.17% / 1.9e-9 | +1.67% |
| UP | 60 | 43.8% / +0.32% / 8.0e-9 | −0.01% | 37.5% / −0.47% / 3.0e-8 | +1.59% |
| UP | 120 | 38.5% / −0.05% / 0.017 | +0.80% | 27.8% / −0.32% / 0.067 | +2.95% |
| KR | 30 | 17.5% / **−0.92%** / 6.5e-7 | −0.19% | 14.0% / **−0.60%** / 1.6e-5 | −0.08% |
| KR | 60 | 17.5% / **−1.10%** / 3.4e-4 | −0.63% | 17.1% / **−0.71%** / 4.0e-4 | −0.93% |
| KR | 120 | 25.0% / **−1.41%** / 1.3e-3 | −1.09% | 22.0% / **−0.82%** / 5.8e-3 | −1.08% |
| US | 30 | 11.1% / −0.43% / 2.1e-3 | −0.31% | 13.8% / +0.52% / 4.4e-6 | −0.24% |
| US | 60 | 36.4% / +0.31% / 7.7e-6 | −0.93% | 21.4% / +0.65% / 1.2e-6 | −0.51% |
| US | 120 | 36.4% / +0.47% / 3.2e-4 | −1.19% | 25.0% / +1.34% / 2.7e-5 | −1.44% |

- **KR**: ACCEL·ZVOL 모두 사건을 잘 잡는다(정밀도 14~25%, 리프트 5~14배). 그런데 **모든 지평·구간에서 비용후 음수**다(왕복 0.41%). "신호가 없다"가 아니라 "비용을 못 넘는다"는 뜻이며, 29차 KR 결론과 일치한다.
- **US**: DEV에서 좋아 보이는 값이 VAL·HOLD에서 0 TP로 붕괴한다(사건 27~53건). 표본이 결론을 지탱하지 못한다.

### 3-5. 신규 선행 가설군 (사전 등록 · 탐색적)

| 가설군 | 시장 | 정의 | DEV (n/TP/정밀도/p/순손익) | VAL | HOLD | 판정 |
|---|---|---|---|---|---|---|
| `VCOMP_TVACCEL` | KR | 변동성 압축(하위25%) + 거래대금 가속(z_vol·vol_ratio 상위25%) | 50/5/10.0%/4.2e-4/**−0.41%** | −0.15% | −0.77% | **불채택**(비용후 음수) |
| `NIGHT_GAP` | KR | 야간 갭 상위10% + 변동성 압축 | 40/5/12.5%/1.4e-4/**−0.37%** | −1.01% | +0.56% | **불채택**(일관성 없음) |
| `SUPPLY_F5` | KR | 외인 5일 순매수>0(공표 후) + 압축 | 표본 부족(공표 커버리지 제한) | — | — | **미측정**(정직 기록) |
| `VCOMP_TVACCEL` | UP | 같은 정의 | 60/0/0%/1.0/**−0.24%** | +0.05% | −0.22% | **불채택** |
| `BTC_REL` | UP | BTC 대비 60분 상대강도 상위10% + 압축 | 60/5/8.3%/0.032/−0.50% | −0.20% | +0.63% | **불채택**(DEV 음수) |
| `BTC_REL` | UP H=120 | 같은 정의 | 42/20/47.6%/**3.2e-8**/**+0.46%** | +0.69% | **−0.29%** | DEV 통과·HOLD 음수 → **탐색적 기록만** |
| `VCOMP_TVACCEL` | US | 같은 정의 | 40/3/7.5%/4.7e-4/+0.31% | 0 TP | 0 TP | **불채택**(VAL 붕괴) |

- 가족 크기(시장·지평별 시험 수)로 Bonferroni를 따로 걸었다(UP·KR 3가설 → α 0.0167, US 2가설 → α 0.025). 675 격자의 α 7.4e-5와 **섞지 않는다**.
- `SUPPLY_F5` 는 `kr_s16_data.db supply.first_available_at`(P0-A)을 지켜 공표 이후 값만 특징으로 썼지만, 60종목 표본에서 커버리지가 낮아 판정 표본을 못 채웠다. 0으로 채우지 않고 **미측정**으로 남겼다.
- 결론: "변동성 압축 + 거래대금 가속"은 **이 표본·이 비용 가정에서 살아남지 못했다**. 사전에 세운 가설이 죽은 것을 그대로 기록하는 것이 이 실험의 목적이다.

---

## 4. 검증 (전부 PASS)

| 스크립트 | 항목 | 결과 |
|---|---|---|
| `research/verify_stale_gate_20261120.py` | 동결값/해시 · 신선 신호 · stale expired(누락 수치) · 시각 순서 · 대안 ID 미연결 | **13/13 PASS** → `verify_stale_gate_20261120.json` |
| `research/verify_production_connected_20261120.py` | 유니버스 인과성 · e2e/production 분리 · 등록 이전 봉 무시 · 실제 시각 · US stale · 60분 라벨 · 예산 가드 · 실제 원장/스케줄 증거 | **10/10 PASS** → `verify_production_connected_20261120.json` |
| `research/verify_horizon_lab_20261120.py` | 지평 독립(H=30 vs 60 vs 120) · 세션 censored · purge · α=0.05/675 · 결정성 · 실제 산출물 | **6/6 PASS** → `verify_horizon_lab_20261120.json` |
| `forward_tracker.py --track US_GEN16 --self-test` | 규칙·체결·정산 자체검증 | `{"ok": true, "fails": []}` |
| `scheduled-action check scheduled-actions/🔬-자가발전-패턴-연구` | 60초 상한·quiet | **quiet(exit 1), 25초** (`prod=KR/shadow_scan/ok` 하트비트) |
| `python3 research/horizon_lab.py --market {UP,KR,US}` | 실제 3시장 산출 | 3시장 × 3지평 JSON 생성(각 17~47초) |

재현:

```bash
python3 research/research_universe.py --market KR --n 150
python3 research/horizon_lab.py --market UP --horizons 30,60,120 --symbols 150 --out research/reports/horizon_UP_20261120.json
python3 research/signal_timing_metrics.py --tracks US_GEN16,US_WAVE_RANK,UPBIT_GEN3 --prod-db "scheduled-actions/🔬-자가발전-패턴-연구/state/knowledge.db" --out research/reports/timing_metrics_20261120.json
python3 research/production_connected.py --db "scheduled-actions/🔬-자가발전-패턴-연구/state/knowledge.db" --stage status
python3 research/verify_stale_gate_20261120.py && python3 research/verify_production_connected_20261120.py && python3 research/verify_horizon_lab_20261120.py
```

---

## 5. `monitor/latest.json` — 3시장 `research` 최신 상태

`research/monitor_research_20261120.py` 를 schema 3으로 올려, 각 시장에 **`connection`** 을 추가했다:

```
research.markets.<upbit|kr|us>.connection = {
  e2e_connected: true,                        // 29차 격리 계약 실증
  production_connected: true|false,           // 운영 스케줄 실행 기록 + Shadow 단계 호출
  rotation_status, rotation_at, rotation_generation,
  prod_stages, production_signals, expired_stale, shadow_registered,
  stale_max_sec, measured_at
}
research.stale_gate = { values: {US: 420}, hash: "f0c02b728ca0c7c1", basis, frozen_at }
```

- 운영 실측이 있는 항목만 `tp`·`fp`·`censored`·`price_gain_error`(MAE)·`latency`(decision→event·entry→event)·`execution_net` 을 숫자로 채우고, `status_fields` 에 `measured:` 로 표기한다. 미측정은 **null + 사유**로 남긴다(0으로 채우지 않음).
- 전진 장부 요약에 `expired_stale`·`stale_max_sec` 를 추가했다 — US 장부에서 **stale 로 버려진 신호 수**를 볼 수 있다.
- 5분 푸시는 P0-B에서 고친 경로(`push_dashboard_api.py` → `collab_ledger.push_latest(status, research)`)를 그대로 탄다. 이 보고서 작성 후 실제 raw GET으로 3시장·`connection` 필드를 확인했다(§7).

---

## 6. 변경·신규 파일

**신규**
- `research/production_connected.py` — 운영 연결 자기발전(단계 상태머신·진짜 미래 Shadow·stale 게이트·사전 등록)
- `research/research_universe.py` — 컷오프 고정 150종목 유니버스
- `research/horizon_lab.py` — 3시장 × H=30/60/120 독립 연구(675 격자·purge·기준선·신규 가설군)
- `research/signal_timing_metrics.py` — `true_early_TP`/`late_event`/`no_event`/시각/손익/상승폭 오차
- `research/verify_stale_gate_20261120.py` · `verify_production_connected_20261120.py` · `verify_horizon_lab_20261120.py`
- `research/stale_gate_freeze_20261120.json` · `research/stale_fill_strategies_20261120.json`
- `research/reports/horizon_{UP,KR,US}_20261120.json` · `universe_{UP,KR,US}_150_20261120.json` · `timing_metrics_20261120.json`
- `research/reports/gpt_30th_p1_report_20261120.md`(이 문서) · `research/evaluation_summaries/gpt_30th_p1_summary_20261120.json`

**수정**
- `research/shadow_engine.py` — 3시장 일반화(시장별 DB·컬럼·시각형식·세션·비용). KR·US는 `trade_price` 없음 → close×volume.
- `research/auto_researcher.py` — `AR_UNIVERSE_FILE`(없으면 기존과 동일) 유니버스 제한. UP 경로 무변경.
- `research/forward_tracker.py` — US 트랙 `stale_max_sec=420` 동결·expired 기록·시각 4종 보존. `--self-test` PASS.
- `research/monitor_research_20261120.py` — schema 3, `connection`·`stale_gate`·`expired_stale`.
- `scheduled-actions/🔬-자가발전-패턴-연구/check.sh` — P1-E 단계(14초)·UP 예산 28초·하트비트 prod 열.
- `scheduled-actions/🔬-자가발전-패턴-연구/action.md` — 새 단계·상한·구분 설명.

---

## 7. 게시 검증

- 이 보고서: `https://raw.githubusercontent.com/mju1417-max/market-dashboard/main/research/reports/gpt_30th_p1_report_20261120.md` — HTTP **200** (아래 실행 기록)
- 요약 JSON: `https://raw.githubusercontent.com/mju1417-max/market-dashboard/main/research/evaluation_summaries/gpt_30th_p1_summary_20261120.json` — HTTP **200**
- `monitor/latest.json` — **GitHub Contents API로 원격 실물 확인**(2026-11-20 23:05 KST 게시분):
  `research.schema=3`, `research.stale_gate={values:{US:420}, hash:"f0c02b728ca0c7c1"}`,
  `research.markets.{upbit,kr,us}.connection` 각각 `{production_connected:true, e2e_connected:true, rotation:{status:no_new_data|completed|completed}, signals_prospective:8|0|0, price_gain_error:{mae_pct:4.7031}|null|null}`.
  `research_source=argument`(빈 dict 로 덮이지 않음).
  참고 1: `raw.githubusercontent.com` 은 CDN 캐시 때문에 새 커밋 직후 조회에서 직전 개정이 보일 수 있다(같은 순간 Contents API 는 새 문서 반환). HTTP 코드는 200 이다.
  참고 2: 게시 파이프라인은 `DASH_STATE_DIR` 캐시를 15~30분 TTL 로 읽는다. 스키마 변경 직후에는 **캐시가 옛 문서를 계속 실어 보낼 수 있어**, 이 회차에서는 채점 액션의 캐시를 현재 모듈로 한 번 갱신한 뒤 푸시해 확인했다(23:05 게시분 schema 3·`connection` 3시장).

---

## 8. 하지 않은 것 · 검증하지 못한 것 (정직 기록)

1. **US 세대교체는 완주했지만 표본이 짧다** — US-G001 의 DEV 102,882행은 11-10~11-17 8세션이다. 272 격자에서 통과 0건이라는 사실은 "이 표본에서 후보가 없다"는 뜻이며, 더 긴 이력이 쌓이면 다시 평가해야 한다.
2. **KR·US 운영 Shadow 신호 0건** — KR 장 마감·US 정규장 전(프리마켓)이었고, 사전 등록 가설 조건이 드물다. 경로는 테스트 10/10과 UP 실측 8신호로 증명했지만, **KR·US에서 실제 신호가 나온 것은 아직 아니다**.
3. **UP 8신호는 catch-up(90분 과거 시작)** 이다. `staleness_sec` 3,300~5,100초를 그대로 남겼고, 사전포착 통계에 넣지 않았다. 이 8건으로 전략 성과를 말하지 않는다.
4. **`SUPPLY_F5` 가설은 미측정**이다(공표 후 커버리지 부족). 0으로 채우지 않았다.
5. **H=120 사건률(UP 15.7%)은 purge 후 표본(16,004행) 기준**이다. 지평이 길수록 겹침 제거가 커져 표본이 급감한다 — 지평 간 사건률을 직접 비교할 때는 표본 열을 함께 봐야 한다.
6. **US 사건 27~53건**으로는 어떤 가설도 검정력이 없다. 10세션 표본을 늘리거나(더 긴 이력) 유니버스를 넓히지 않으면 US 결론은 계속 "판정 불가"다.
7. **`product_connected`의 '연결'은 실행 증거**이지 성과가 아니다. 회전이 돌았다는 사실과 전략이 돈을 번다는 사실을 분리해 읽어야 한다.
8. **게이트의 실제 결과: 미국 두 트랙은 앞으로 신호가 거의 안 나올 수 있다.** 실측 수집 지연이 466~770초인데 상한이 420초이므로, 그런 회차의 후보는 전부 `expired` 로 남는다. 이것은 의도된 정직한 결과(늦은 관측을 신호로 세지 않음)지만, **미국 전진검증 표본 증가가 사실상 멈춘다**는 뜻이기도 하다. 표본을 늘리려면 수집 지연을 420초 안으로 줄이거나(수집기 개선, 다른 트랙 소관) `stale_max_sec` 를 새 전략 ID로 다시 동결해야 한다. 지금 상태로 "미국 트랙 성과"를 기대하면 안 된다.
9. 실주문·실거래·알림 없음. 모든 수치는 모의·연구 기록이다.

---

## 9. 필수 제출 항목 대응

| 요구 | 이 회차 값 |
|---|---|
| work_id | 30/P1-E · 30/P1-C · 30/P1-B · 30/P1-F |
| 데이터 cutoff | UP 2026-11-20T22:23+09:00 · KR 2026-11-20T15:30+09:00 · US 2026-11-21T06:00+09:00 (유니버스 컷오프) |
| 전후 데이터 행수 | KR 운영 패널 0 → 106,907행 · US 운영 패널 0 → 127,954행 · 연구 표본 UP 200,703 / KR 82,720 / US 84,023행(각 3지평) |
| 재현 명령 | §4 |
| 중단·복구 | 단계 커서(`prod_state`·`last_scanned_ts`·`symbol_cursor`)로 예산 중단 후 재개(P7). 회전은 `busy_skip` 시 다음 회차 |
| DEV/VAL/HOLD 경계 | UP H=60: DEV 11-10~11-17 · VAL 11-18~11-19 · HOLD 11-20 (KR·US 동일 규칙, §3-2 표의 일자) |
| TP/FP/censored | §3-2 표(사건률·censored) + §1-4 운영 Shadow(TP 0·FP 4·censored 1) |
| 운영 스케줄 실제 가동 로그 | `🔬` `state/check_heartbeat.log`(`prod=KR/shadow_scan/ok`), `state/knowledge.db` `rotation_runs`(KR 8·US 7)·`prod_runs`(KR 9·US 7) |
| 불일치 목록 | `monitor/latest.json` `candles_1m_query`(SUM vs COUNT) 차이는 P0-7 정의 차이로 병기 유지. KR monitor `forward_tracks` 의 `open 9` 는 P0-A 분리 전 스냅숏(3 forward + 6 recovery)이라 원장과 표기가 다르다 — P0-A 소관 |
| 미구현·미측정 | §8 |
