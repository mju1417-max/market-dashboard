# GPT 제27차 통합 지시서 — P0-A~F + GitHub 협업 원장 구축 실행 보고

- **작업 ID**: `27th-round / P0-A~F / III`
- **작성·실행 시각**: 2026-11-20 17:36 ~ 18:3x KST (모든 수치는 이 구간 실측)
- **대상 시장**: UP(업비트) 중심, DB·대시보드·원장은 3시장 공통
- **적용 시장/공통 구분**: P0-A·P0-F·Ⅲ장 = 공통 인프라, P0-B~E = 업비트 연구 트랙(공통 모듈 경유)
- **데이터 기준 시점**: 업비트 1분봉 `2026-11-09 07:02 ~ 2026-11-20 18:2x KST` · 연구 패널 DEV `11-09 07:00~11-19 00:00`, HOLD `11-19 00:00~11-20 00:00`
- **DEV / 검증 / HOLD / Shadow 구분**: 본 보고의 성능 수치는 **DEV·HOLD 재생(백테스트)** 이고, Shadow 미래 성능은 **동결 후보 0건 → 표본 0** 이다. 실측이라 부르는 것은 전진 장부의 수신 지연뿐이다.
- **코드 버전**: `research/export_dashboard_json.py`·`push_dashboard_api.py`·`failure_hypothesis.py`·`leading_accel.py`·`shadow_engine.py`·`verify_auto_researcher.py`·`collab_ledger.py`(신규)·`scheduled-actions/*/check.sh` 수정, `sparse_shards.py` 무변경(리더만 사용)
- **실행 환경**: Python 3 / SQLite 3 (WAL), 5분 무과금 파이프라인 4개 가동 중

---

## 0. 요약 (결론 먼저)

| 항목 | 결론 | 근거 |
|---|---|---|
| **P0-A** 업비트 `db_ok=false` | **원본 DB 무손실.** 원인은 상태 생성기가 읽기 실패를 **0건으로 덮어쓰고 사유를 남기지 않던 로직** + 다른 액션(샌드박스)에서 WAL DB를 밖에서 읽는 경로. **수집기가 자기 state/에 통계 1장을 남기고 대시보드가 그것만 읽도록** 바꿔 우회·복구. 18:22·18:26 게시본 `db_ok=true`, `db_source=collector_stats` | §1 |
| **P0-B** 트랙2 직접 접근 | 두 스크립트 3곳 모두 `sparse_shards.open_panel(readonly=True)` → `panel_view_all` 로 전환. **프루닝 전/후 패널 4,653,960행·내용해시 `76980985…` 동일**, DEV/HOLD 평가·실패가설·선행가속 결과 **1:1 동일**. 그 뒤 프루닝 실행(활성 4,653,960→866,880행, 파일 403MB→67MB) | §2 |
| **P0-C** 예산 102건 | **합법.** 예산일이 **11개**(11-09~11-19)이고 상한은 11×10=110. 확정(의사결정) 시각 기준 하루 최대 10건을 넘지 않았다. 100→102는 KST 자정을 넘긴 9~12건이 **다음 예산일**로 옮겨간 결과 | §3 |
| **P0-D** 순손익 차이 | 두 숫자는 **같은 100건·같은 규약**이다. −90.92%p 는 **라벨 확정 88건의 합**, −103.3%p 는 **평균(−1.033) × 선택 100건**이라는 표기 산술 오류. 검열 12건 × 1.033 = 12.40%p 만큼 부풀려졌다 | §4 |
| **P0-E** Shadow 시각 | `shadow_signals` 에 `first_received_exact`·`latency_source` 열 추가. 재생 신호는 전부 `exact=0`, 지연 표본은 **관측 시작 이전 것만** 사용(미래 지연 사후 보정 금지). 합성 데모: 측정 0 / 추정 26, 지연 `assumed:default_60s` | §5 |
| **P0-F** 운영 안정성 | 수정된 스냅숏 실측 **1.99초/58MB**, 동시 read-only 20회 **실패 0·최대 52ms**, 수집기 회차 간섭 0. 다만 **직전 구현은 4분 초과**였고(원인: 전체 복사 후 96만 행 DELETE), 필요 행만 복사하는 방식으로 다시 고쳤다. 종료코드 규약(0/1/2) **12/12 검증 PASS** | §6 |
| **Ⅲ장** 협업 원장 | 공개 저장소(`mju1417-max/market-dashboard`, **PUBLIC 확인**)에 `monitor/`·`coordination/`·`research/` 구조 구축, `monitor/latest.json` 5분 자동 갱신 연결. 비밀키·개인정보·원본 DB·절대경로 미포함 | §7 |
| **검증** | `verify_sparse_grid 65/65` · `verify_18th 21/21` · `verify_429_counter 13/13` · 신규 P0 검증 4종 전부 PASS (`verify_shadow_and_evolution` 결과는 §8) | §8 |

---

## 1. P0-A — 업비트 `data/status.json` `db_ok=false` 조사·복구

### 1-1. 무엇이 보였나 (GPT 관측 재현)

GitHub `data/status.json` 커밋 `f48aebb`(2026-11-20 17:16:43 KST 생성)를 직접 내려받아 확인했다.

```json
"markets": { "upbit": { "universe": null, "collect_cycles": 290,
  "collector_last_run_at": "2026-11-20T17:12:58", "collector_errors": 0,
  "symbols_with_candles": 0, "candles_1m": 0, "candle_from": null, "candle_to": null,
  "db_ok": false, "fetch_ok": 0, "fetch_rows": 0, "last_fetched_at": null,
  "surges": {"total": 84, "ge5_pct": 65, "last_detected_at": "2026-11-20T17:08:27"} } }
```

같은 시각 KR·US 는 정상(`db_ok=true`, 4,678,981 / 4,563,630봉)이었다. 즉 **한 시장만** 0으로 떨어졌고, `collector_last_run_at`·`surges` 는 같은 폴더에서 정상적으로 읽혔다 → **폴더·경로 문제가 아니라 DB 읽기만 실패**한 형태다.

### 1-2. 실제 운영 DB는 정상이었다 (원본 손실 아님)

| 확인 항목 | 실측값 |
|---|---|
| 파일 | `scheduled-actions/📈-업비트-모의투자-분석/state/upbit_1m.db` · 238,682,112 B (228MB) |
| 봉 수 / 종목 수 | **1,515,732 봉 / 302 종목** (조회 0.08초) |
| 최초/최종 봉 | `2026-11-09T07:02:00` ~ `2026-11-20T17:32:00` |
| fetch_log | 302행 전부 `ok`, 마지막 수집 `17:33:09` |
| 저널 모드 | **WAL** (11-20 트랙2 잠금 사고 수리 때 `busy_timeout 30s` 와 함께 켜짐) |
| 무결성 | `PRAGMA quick_check = ok`, `page_count×page_size == 파일 크기` |

→ **1분봉 151만 건은 그대로 있었고, 손실은 없었다.** GPT 지적대로 "원본 DB 손실"로 단정하지 않는다.

### 1-3. 언제부터·얼마나 자주 (커밋 이력 실측)

`data/status.json` 커밋 경로를 13:00~18:40 KST 전부 훑어 `markets.upbit.db_ok` 를 뽑았다.

| 시각(KST) | db_ok | 비고 |
|---|---|---|
| 13:01 ~ 16:51 (46회) | true | 정상 |
| **16:56** | **false** | 첫 실패 |
| 17:02 | true | 일시 회복 |
| **17:06, 17:11, 17:16, 17:21, 17:26, 17:31** | **false** | 6회 연속 |
| 17:36, 17:41, 17:46, 17:51, 17:56, 18:01, 18:06 … | true | 회복(§1-5 조치 후 `collector_stats`) |

→ **간헐적**이며 25~30분짜리 구간에서 연속 실패했다. 같은 시간에 수집기는 `collector_errors=0`, 46~50초로 정상이었고 전진검증 채점도 정상(`last_status.ok`)이었다.

### 1-4. 원인 — 왜 0으로 나갔고 왜 사유가 없었나

1. **상태 생성기의 실패 처리 결함(확정)**: `export_dashboard_json.db_stats()` 는 예외를 잡아 `available=False` 만 돌려주고 **예외 메시지를 버렸고**, `build_status()` 는 그 값을 `candles_1m=0 · universe=null · db_ok=false` 로 그대로 게시했다. 즉 **"읽지 못했다"가 "0건이다"로 둔갑**했고, 사유는 어디에도 남지 않았다. 이것이 GPT가 6번 항목으로 지적한 "상태 생성 실패 시 0건으로 표시하는 로직"의 실체다.
2. **실패의 기전(재현 확인)**: 업비트 DB는 **WAL 모드**다. WAL DB를 **다른 액션의 샌드박스**(대시보드 푸시는 `📊-업비트-전진검증-채점` 에서 돈다 — 자기 `state/` 밖은 쓰기 금지)에서 read-only로 열면, `-shm` 파일을 만들지 못해 SQLite가 아래 오류를 낸다.

   ```
   sqlite3.OperationalError: attempt to write a readonly database
   ```
   실제로 같은 형태(신선한 WAL DB + 쓰기 금지 디렉터리)를 만들어 확인했고, `-shm` 이 이미 있으면 같은 조건에서 정상적으로 읽혔다. 이 오류가 나면 위 1항에 의해 **정확히 관측된 그 0-패턴**이 만들어진다.
   *한계*: 그 순간의 실제 예외 문자열은 옛 코드가 버려서 **남아 있지 않다**(그래서 1항을 수리했다). 또한 같은 액션의 `forward_tracker` 는 같은 DB를 문제없이 읽고 있어, "샌드박스가 모든 경로를 막는다"고 단정할 수는 없다. 따라서 아래 조치는 **기전과 무관하게 성립하는 구조적 우회**로 설계했다.
3. **왜 간헐적인가**: `-shm` 파일은 마지막 쓰기 연결(수집기)이 닫을 때 사라진다. 다른 프로세스가 그 사이에 DB를 열어 `-shm` 을 남겨 두면 성공하고, 없으면 실패한다 → 5분 주기 중 어느 시점에 열리느냐에 따라 갈린다(관측된 실패/성공 교대와 일치).

### 1-5. 조치 (원인 보존 + 안전 복구 + 구조적 우회)

| # | 조치 | 파일 |
|---|---|---|
| ① | **수집기가 자기 state/에 `dashboard_stats.json` 을 매 회차 기록** (봉 수·종목 수·fetch 302/302·최초/최종 봉·오류 수·DB 바이트). 비용은 302행 집계 1회(무시 가능) | `📈-업비트-모의투자-분석/check.sh` |
| ② | 대시보드는 **그 파일(30분 이내)을 우선 사용**, 없으면 직접 읽기(2회 재시도) | `export_dashboard_json.db_stats()` |
| ③ | 직접 읽기 실패 시 **`db_error`(형식+메시지) 를 status.json 에 남기고**, `dashboard_export.log`(채점 액션 state, 쓰기 가능)에 기록 | 동일 |
| ④ | 그래도 실패하면 **마지막 성공값(last-good)** 을 쓰고 `db_stale=true`·`db_stale_since` 로 표시 → 0건으로 보이지 않게 | 동일 |
| ⑤ | 푸시 스크립트가 실패를 **exit 2** 로 알리고, 채점 check.sh 는 **3회 연속**일 때만 오류로 올린다(일시 장애 1회는 quiet) | `push_dashboard_api.py`, `📊-업비트-전진검증-채점/check.sh` |

**복구 확인**: 17:51 게시본부터 `db_source=collector_stats`, `db_ok=true`, `candles_1m=1,516,657` → 18:22 `1,518,611` → 18:26 `1,518,927`. 수집기 정상 여부(`collector_errors=0`, 회차 46~49초)와 대시보드 조회 성공 여부(`db_source`)를 **분리해** status.json 에 남긴다.

### 1-6. 남은 것
- 기전이 "샌드박스 쓰기 금지"인지 "일시 잠금"인지 **확정하지 못했다**(예외 문자열 미보존). 이제 `dashboard_export.log` 가 남으므로 재발 시 즉시 확정된다. 현재는 ①의 파일 경로로 DB를 아예 건드리지 않으므로 재발 여지가 없다.
- `dashboard_stats.json` 은 업비트만 만든다. KR·US 는 직접 읽기가 정상 동작(비-WAL)이라 그대로 두었다. 통일하려면 같은 3줄을 각 수집기에 넣으면 된다(미구현).

---

## 2. P0-B — 트랙2 직접 DB 접근 → 트랙1 통합 리더 전환

### 2-1. 바꾼 곳 (3곳)

| 파일 | 함수 | 이전 | 이후 |
|---|---|---|---|
| `research/failure_hypothesis.py` | `analyse_sparse()` | `sqlite3.connect(sparse_grid.db) → from pg_panel` | `SH.open_panel(readonly=True)` → `from panel_view_all where market=? and "window"='dev'` |
| `research/failure_hypothesis.py` | `validate_sparse()` | 동일 | 동일 |
| `research/leading_accel.py` | `build_B()` | `pg_panel` 직접 | `panel_view_all` (x_r5·x_p15·x_zvol·net 포함) |

`sparse_grid.db` 를 직접 여는 코드는 이제 두 스크립트에 없다(`grep -n "pg_panel" research/failure_hypothesis.py research/leading_accel.py` → 0건).

### 2-2. 프루닝 전 1:1 동등성 (통합 리더 vs 활성 표 직접)

`research/verify_p0b_panel_parity.py` 로 같은 분석을 (a) 통합 리더 (b) 활성 표만(구 방식과 동일한 행 집합) 두 번 돌려 비교했다.

| 비교 항목 | 결과 |
|---|---|
| 패널 행수 | **4,653,960 = 4,653,960** |
| 키 중복(`market@minute@window` distinct) | 4,653,960 → 중복 0 |
| 일별·창별 구성(dev 4,221,960 / hold 432,000) | 동일 |
| 내용 합계(종가·체결가·거래대금·no_trade·검열·lab·x_r5·x_zvol) | **전 항목 동일**(부동소수 1e-9 허용) |
| 실패가설 T3 발견/검증 | 동일 |
| 선행가속 B(dev·hold, ACCEL·ZVOL) | 동일(`no_candidate`, 유효 후보 0) |

### 2-3. 프루닝 실행과 사후 동등성

- 사전 스냅숏: `research/.p0b_preprune_snapshot_20261120.db` (403MB, 롤백용 보관)
- 프루닝: `research/p0b_prune_verified.py` — `sparse_shards.prune_range()` 만 호출(재빌드 없음). 각 샤드의 **내용해시를 지우기 직전에 재확인**하고 불일치면 아무것도 지우지 않는 가드를 그대로 통과했다.
- 결과: 11-09~11-17 **9일 3,787,080행** 삭제, 활성 표 4,653,960 → **866,880행**(11-18·11-19 잔류).
- **프루닝 후 통합 리더 패널: 4,653,960행, 해시 `76980985d2ae0f74be1aa5aa114eabf04263500c5e32483b86c6de6ea9acd204` — 프루닝 전과 동일.** 두 분석 결과도 1:1 동일(§2-2와 같은 비교, `analysis_diff=None`).
- VACUUM 으로 파일 회수: **403MB → 67MB** (512MB 상한의 12.5%, 여유 470MB). 삭제만으로는 free page 가 남아 파일이 줄지 않는다(프루닝 직후 `freelist 79,618페이지` 확인).
- `verify_sparse_grid.py` **65/65 PASS** (샤드 무결성·중복 0·중단재개·프루닝 가드·복원 왕복·MANIFEST 해시 포함).

### 2-4. 복원(--restore) 비교
`verify_sparse_grid` S6 이 프루닝 후 **복원 왕복 시 해시 일치**를 이미 검사한다(65/65 안에 포함). 별도 전면 복원은 하지 않았다 — 활성 표를 다시 4.6M 행으로 되돌리면 방금 회수한 용량을 다시 잃기 때문이다. 필요 시 `python3 research/sparse_shards.py --restore all` 1명령으로 가능(샤드 파일은 삭제하지 않았다).

---

## 3. P0-C — Causal FIFO 예산 102건 해명

`research/p0c_budget_audit_20261120.py` (신규 감사기) 로 실제 선택 이력을 다시 계산했다.

### 3-1. 규칙
- A(연구용) 예산일 = **봉 시각(bar_ts)의 KST 날짜**.
- B(실행 가능 Causal) 예산일 = **의사결정 가능 시각(decision_at)의 KST 날짜** — `common_ledger.select()` 의 `day=(time_ts+9h)//86400`. 확정 전에는 예산을 쓰지 않는다.
- `decision_epoch`: `check_grid` = 봉 마감 뒤 처음 오는 5분 체크 경계, `measured` = 봉 마감 + 실측 지연분포 재생.

### 3-2. 실제 KST 예산일 (ACCEL · measured)

| 예산일 | 선택(확정 기준) | 같은 신호의 봉 날짜 기준 | 상한 |
|---|---|---|---|
| 11-09 | 10 | 11 | 10 |
| 11-10 | 10 | 11 | 10 |
| 11-11 | 10 | 9 | 10 |
| 11-12 | 10 | 9 | 10 |
| 11-13 | 10 | 10 | 10 |
| 11-14 | 10 | 10 | 10 |
| 11-15 | 10 | 10 | 10 |
| 11-16 | 10 | **12** | 10 |
| 11-17 | 10 | 9 | 10 |
| 11-18 | 10 | 11 | 10 |
| **11-19** | **2** | 0 | 10 |
| **합계** | **102** | 100 | 11×10 = **110** |

### 3-3. 결론
- **예산일이 10개가 아니라 11개**다(11-09~11-19). DEV 창이 `11-09 07:00 ~ 11-19 00:00` 이므로, 11-18 23:5x 봉의 확정 시각이 11-19 00:0x 로 넘어가면 **11번째 예산일**이 열린다. 그날 후보가 2건 → 10×10 + 2 = **102**.
- **확정 시각 기준으로 하루 10건을 넘은 날은 0일**이다(`per_decision_day_max=10`). 봉 시각으로 세면 11-09·11-10·11-16·11-18 이 11~12건으로 보이는데, 이는 자정을 넘긴 9~12건이 다음 날로 옮겨간 결과다(예: `KRW-META2` 봉 11-18 23:59 → 확정 11-19 00:05).
- **수정할 것은 없다.** 다만 보고서에 "10일 × 10건 = 100" 이라고 적은 것은 **예산일 수를 잘못 센 표기 오류**다(11일 × 10 = 110 상한).
- 부수 확인: `measured` 재생은 **실측 지연 표본에 따라 선택 수가 ±1 흔들린다**(18:0x 표본 n=153 → 102, 16:30 스냅숏도 102). 규약 위반이 아니라 재생 입력이 갱신된 결과이며, 상한 110 안에서만 움직인다.

---

## 4. P0-D — 두 트랙 순손익 합계 차이 해명

`research/p0d_net_audit_20261120.py` 로 같은 신호 집합을 다시 계산해 대조했다.

| 구분 | ACCEL | ZVOL2.5 |
|---|---|---|
| 선택 건수 | 100 | 100 |
| **라벨 확정** | **88** | **86** |
| 검열(censored, 기간 끝) | 12 | 14 |
| 원장 `net.sum` (확정 88/86건 합) | **−90.9177%p** | **−54.7363%p** |
| `net.mean` (확정 88/86건 평균) | −1.0332%p | −0.6365%p |
| **평균 × 선택 100건** | **−103.32%p** | **−63.65%p** |
| 부풀림 = 검열 × 평균 | −12.3979 (= 12×1.0332) | −8.9106 (= 14×0.6365) |
| 트랙1 `sparse_v2` 보고값과 일치 | ✅ (`sum −90.9177`, `mean −1.0332`) | ✅ (`sum −54.7363`, `mean −0.6365`) |

### 결론
- **두 트랙의 숫자는 애초에 같다.** 트랙1 `sparse v2 dev @market` 의 원장값은 ACCEL −90.9177 / ZVOL −54.7363 이고, Causal FIFO 보고의 **A행**도 같은 값이다.
- 차이는 **표기 산술**이다. `mean_net` 은 **라벨 확정 건수(88/86)의 평균**인데, 보고 표에서 여기에 **선택 건수 100** 을 곱해 "합계"로 적었다. 검열 12·14건은 평균·합계 어디에도 들어가지 않는다.
- 평가 규약도 동일하다: 진입 = 신호 봉 **다음 봉 시가**(패널 규칙), 청산 = +5%/−3%/60분, `net` = 왕복 **0.20%p 차감 후**. 두 보고 사이에 체결가·지연·비용·재평가 버전의 차이는 없다.
- A(봉 시각) vs B(확정 시각) 선택 집합은 실제로 다르다: 100 vs 102, 교집합 80, A만 20, B만 22 (지연 재생 표본 n=153 시점). 이 차이는 **평가 규약 차이**이지 계산 오류가 아니며, 승격 판단은 B를 기준으로 한다(프로토콜 v3).

---

## 5. P0-E — Shadow 시각 인과성

### 5-1. 이전 상태(문제)
- `shadow_engine.scan()` 은 재생 신호마다 `first_received_ts = bar_ts + 60`, `decision_at = first_received + 지연` 을 넣었는데, **어디에도 "이건 추정"이라는 표시가 없었다**(`decision_latency_sec` 만 저장).
- 지연은 `online_evaluator.live_receive_lags()` 의 **전체 기간** 중앙값을 썼다 → 검증 구간보다 **미래** 관측으로 과거를 보정할 수 있는 구조였다.

### 5-2. 지금
| 항목 | 내용 |
|---|---|
| 스키마 | `shadow_signals` 에 `first_received_exact`(1=실측/0=재생·가정), `latency_source` 추가(기존 원장 자동 마이그레이션, 값 보존) |
| 기록 | 재생 신호는 **항상 `first_received_exact=0`** — "실제 관측 시각"으로 부르지 않는다 |
| 지연 표본 | `decision_latency_sec(cutoff)` — **cutoff(그 shadow 의 관측 시작/마지막 스캔) 이전에 관측된 지연만** 사용. 없으면 60초 가정 + `assumed:default_60s` |
| 평가 표시 | `shadow_evaluations.metrics_json.time_basis` 에 `n_measured / n_estimated / latency_sources` |
| 합성 데모 실측 | 신호 26건 전부 `딜레이 60초 · exact=0 · assumed:default_60s`, 평가 `n_measured=0 / n_estimated=26` |

### 5-3. 실측 지연과의 대조 (그대로 보고)
- 전진 장부에서 **정확히 관측된** 지연(exact=1, ≤360초): n=153, p10 **151초**, 중앙 **216초**, p90 **306초**, 최대 350초(18:1x 기준). 27차 지시서의 p10 159 / 중앙 230 / p90 418 과 같은 자릿수다(모집단·시점 차이).
- 따라서 `check_grid`(5분 격자) 가정은 **낙관적 하한**이고, `measured` 재생도 실측보다 이른 경우가 60.8%(격리검증 1항)다. 이 사실은 프로토콜 v3 `protocol.fifo.executable.claim` 에 "재생"으로 명시돼 있다.
- **동결 후보가 0건**이라 실제 미래 성능 표본은 아직 0건이다. 이 보고에서 "실측 성능"이라고 부를 수 있는 Shadow 수치는 없다.

---

## 6. P0-F — 운영 안정성·종료 코드

### 6-1. 스냅숏 실측 (수집기 유휴 구간 1회)
- 시각: 2026-11-20 18:29:26 KST (직전 수집 회차 18:28:44 종료, 46~49초 정상)
- 측정: `research/p0f_snapshot_probe_20261120.py` — 스냅숏 실행 중 0.1초 간격으로 **대시보드와 같은 read-only 열기**를 20회 시도
- 결과: **소요 1.99초**, 산출 58,781,696 B(58MB), read-only 실패 **0/20**, 최대 지연 **0.052초**
- 수집기 영향: 이후 회차 `18:29`·`18:33` 모두 `quiet` 46~49초, `collector_errors=0`

### 6-2. 측정 중 발견한 결함과 재수리
- **직전 구현은 4분을 넘겼다.** `src.backup(out)` 로 239MB 전체를 복사한 뒤 `delete from candles_1m where ts_utc < FLOOR` 로 96만 행을 지우는 구조였고, 삭제가 파일 페이지 대부분을 다시 써서 이 환경에서 분 단위가 걸렸다(인덱스·`journal_mode=OFF` 로도 동일). 16:52 의 `.ar_test/probe_snap.db`(25MB, -journal 잔존)가 그 흔적이다.
- **다시 고침**: 필요 행만 `CREATE TABLE AS SELECT ... WHERE` 로 복사(스키마·인덱스는 `sqlite_master` 정의 재생성, 원본은 `mode=ro`). 실측 **전체 1.99초 / T1 1.32초 / T2 1.26초**, `quick_check=ok`, 행수·구간 정확(전체 603,272행 / T1 371,740 / T2 388,566).
- 결론: 이제 검증 스냅숏은 **운영 DB에 사실상 영향 없는 2초 작업**이다. 다만 원칙은 유지한다 — **장중 전체 `verify_auto_researcher` 반복 실행은 하지 않는다.**

### 6-3. 종료 코드 규약 (0=AI, 1=quiet, 2=error)
| 액션 | 이전 | 지금 |
|---|---|---|
| `📈-업비트-모의투자-분석`(수집) | 0/1/2 (그대로) | 0/1/2 (변경 없음) |
| `📊-업비트-전진검증-채점` | **항상 1** (ft-fail·푸시 실패가 조용히 삼켜짐) | 정상 1 · **트랙 실패/텔레그램 오류/대시보드 3회 연속 실패 = 2** |
| `push_dashboard_api.py` | 실패해도 0 | 실패 시 **2** (메시지 출력) |

검증: `research/p0f_exitcode_verify_20261120.py` — 셸에서 종료 판정부를 떼어내 6가지 조합 실행 + 문서·코드 일치 검사, **12/12 PASS**. `action.md` 의 "항상 exit 1" 문구도 규약에 맞게 고쳤다(불일치 해소).

---

## 7. Ⅲ장 — GitHub 협업 원장

### 7-1. 저장소 확인
- 대상: **`mju1417-max/market-dashboard` — PUBLIC 저장소**(`gh repo view` 로 확인). GPT가 읽을 수 있는 공개 저장소라는 전제가 맞다.
- 게시 원칙(구현에 반영): 비밀키·토큰·PAT·텔레그램 chat id·개인정보·대용량 DB·절대경로 금지. `status.json` 은 원래부터 시장 지표만 담는다.

### 7-2. 구조 (게시 완료)
```
monitor/latest.json            3시장 최신 상태 요약(DB·봉 수·전진 트랙·검증 결과) — 5분 자동 갱신
monitor/events.jsonl           주요 사건(구현·수리·검증·프루닝·세대교체) append-only
monitor/telegram_reports.jsonl 비식별 텔레그램 발송 이력(시각·유형·성공여부만, 본문 없음)
monitor/evidence/README.md     증거 파일 규칙
coordination/pickle_to_gpt.jsonl  GPT 검토 요청·반론(의견 ID)
coordination/gpt_to_pickle.jsonl  27차 의견 접수·처리 상태
coordination/review_history.jsonl 교차검증 이력
research/reports/gpt_27th_p0_report_20261120.md  이 보고서
research/evaluation_summaries/p0_eval_summary_20261120.json
research/test_results/verify_results_20261120.json
```
- 갱신 연결: `research/collab_ledger.py` 가 `monitor/latest.json` 을 만들고, `push_dashboard_api.py` 가 status.json 푸시 직후 이를 호출한다(**내용이 같으면 커밋하지 않음** — `generated_at` 제외 비교).
- 푸시는 기존과 같은 Contents API + PAT(`research/.dashboard_pat`)를 쓴다. 새 인증정보를 만들지 않았다.

### 7-3. 공개/비공개 분리 검토 (GPT Ⅲ장 요청)
- 공개 저장소에는 **요약·사건·검증 결과**만 올린다. 연구 원본 코드·패널 DB·전략 임계값은 올리지 않는다.
- 더 깊은 코드 단위 검토가 필요해지면 **비공개 연구 저장소**를 따로 만드는 편이 맞다(이번 회차에는 만들지 않았다 — 저장소 생성은 사용자 계정에 남는 결정이라 사용자 확인 후 진행).

---

## 8. 검증 결과 (실행 명령 포함)

| 스크립트 | 결과 | 명령 |
|---|---|---|
| `verify_sparse_grid.py` | **65/65 PASS** (프루닝·VACUUM 후) | `python3 research/verify_sparse_grid.py` |
| `verify_18th.py` | **21/21 PASS** | `python3 research/verify_18th.py` |
| `verify_429_counter_20261120.py` | **13/13 PASS** | `python3 research/verify_429_counter_20261120.py` |
| `verify_shadow_and_evolution.py` | **8/8 PASS** (166.5초, §8-1) | `python3 research/verify_shadow_and_evolution.py` |
| `verify_p0b_panel_parity.py` | PASS (프루닝 전 both / 후 integrated) | `python3 research/verify_p0b_panel_parity.py` |
| `p0c_budget_audit_20261120.py` | PASS (4개 규칙×모드) | `python3 research/p0c_budget_audit_20261120.py` |
| `p0d_net_audit_20261120.py` | PASS (`agrees_with_track1=true`) | `python3 research/p0d_net_audit_20261120.py` |
| `p0f_exitcode_verify_20261120.py` | **12/12 PASS** | `python3 research/p0f_exitcode_verify_20261120.py` |
| `p0f_snapshot_probe_20261120.py` | 1.99초 · 실패 0/20 | `python3 research/p0f_snapshot_probe_20261120.py` |

### 8-1. `verify_shadow_and_evolution` 상세 (2026-11-20 18:4x, 166.5초)

| 항목 | 결과 | 근거 |
|---|---|---|
| S1 Shadow 합성 end-to-end(격리) | PASS | 신호 26 · 판정 reject(정밀도 0.115<0.15) · 하루 최대 10건 · 예산 위반 0 |
| S2 Shadow 실등록 0건 | PASS | 운영 원장 후보 0 (동결 0건이 정상) |
| R1 자동 세대교체(격리) | PASS | gen [1,2,3] → [1,2,3,4], 7단계 전부 ok, tested 272 |
| R2 같은 입력 재실행 | PASS | `no_new_data`, trials 5168→5168 |
| R3 운영 스케줄 실측 | PASS | 18:29:19 KST `no_new_data`·`ok`·11초·research=ro |
| H1 실패 원장 가설 생성 | PASS | 4유형(T1 inconclusive·T2 validated·T4 refuted·T3 refuted), 구간분리 유지 |
| L1 선행 가속 정합 | PASS | A=66 B=236, parity·trailing·budget |
| A1 기존 `verify_auto_researcher` 7/7 | PASS | 18:29 이전 결과 파일 확인 |

부수 수리: 이 검증기의 `_prepare_rotation_env()` 가 격리 패널에서 **10만 행 DELETE** 를 해 1회 준비에 183초가 걸렸다(전체 실행이 셸 상한을 넘겨 중단됨). 남길 행만 골라 표를 새로 만드는 방식으로 바꿔 **35.6초**로 줄였다(결과 데이터 동일: UP 패널 22,842행·최대 day_i 20775). 검증 대상이 아니라 **검증기를 돌리기 위한 준비 코드**만 바꿨다.

---

## 9. 수정 전후 비교 (한눈에)

| 항목 | 전 | 후 |
|---|---|---|
| 업비트 대시보드 상태 | 간헐 `db_ok=false`, 0건, 사유 없음 | 수집기 통계 파일 경유 `db_ok=true`, 실패 시 사유·stale 표시 |
| 트랙2 패널 읽기 | `sparse_grid.db` 직접(프루닝되면 깨짐) | 통합 리더(활성+샤드), 프루닝 후에도 동일 |
| 연구 패널 파일 | 403MB (상한 512MB의 79%) | **67MB (12.5%)** |
| 활성 표 행 | 4,653,960 | 866,880 (샤드 11개에 보존) |
| Shadow 시각 | 추정·실측 구분 없음 | `first_received_exact`·`latency_source` 필수 기록 |
| 지연 표본 | 전체 기간(미래 포함) | 관측 시작 이전만 |
| 검증 스냅숏 | 4분 초과(수집기 잠금 사고 이력) | **1.99초**, 읽기 지연 최대 52ms |
| 채점 액션 종료 | 항상 1 | 1(정상)/2(오류) |

---

## 10. 미구현·미측정·한계 (그대로)

1. **[미확정]** P0-A 의 그 순간 예외 문자열 — 옛 코드가 버려서 복원 불가. 로그를 새로 남기게 했으므로 재발 시 확정된다.
2. **[미구현]** KR·US 수집기의 `dashboard_stats.json` — 직접 읽기가 정상이라 넣지 않았다.
3. **[미측정]** Shadow 실제 미래 성능 — 동결 후보 0건, 표본 0. 승격·알림 없음.
4. **[변동성]** `measured` 재생의 선택 수는 실측 지연 표본이 늘면 ±1 흔들린다(102 ↔ 101). 상한(예산일×10) 안에서만 움직인다.
5. **[미실행]** 전면 `--restore` 비교 — S6 왕복 검사로 대체(용량 재손실 방지).
6. **[남은 위험]** 대시보드 공개 저장소에는 여전히 전진 트랙 성적·종목명이 실린다(사용자가 이미 공개 중인 범위). 전략 임계값·원본 코드는 올리지 않았지만, 더 엄격히 하려면 비공개 저장소 분리가 필요하다.

---

## 11. 다음 작업 / GPT 검토 요청

**GPT에게 묻고 싶은 것**
1. P0-C 의 "예산일 11개" 해석이 Ⅳ장 P0-C 의도와 맞는가? (10일 표기 정정으로 충분한지, 아니면 DEV 창을 11-19 00:00 에서 자르는 규약을 바꿔야 하는지)
2. P0-D 는 "표기 산술 오류"로 결론냈는데, `mean × n_selected` 를 쓰는 표를 **금지**하는 규약을 프로토콜에 넣는 게 좋은지.
3. P0-E 의 "관측 시작 이전 지연만 사용" 규칙이 과도하게 보수적인지(라이브에서는 등록 이후 신규 지연이 계속 쌓이는데, 그걸 다음 shadow 부터 쓰는 방식).
4. P0-A 의 수집기 통계 파일 경유 방식이 "대시보드가 DB를 직접 읽어야 한다"는 요구와 충돌하는지.
5. Ⅲ장 공개/비공개 분리 — 공개 저장소에는 요약만 두고 코드 검토용 비공개 저장소를 만들지 여부.

**다음 회차(우선순위)**
1. 11-20 완결 후 첫 **자동 세대교체 운영 관찰**(gen 4 행 생성 확인).
2. 새 DEV/HOLD 에서 T2 가설 재검증 → 유지 시 동결 후보 → Shadow 등록(이때부터 실측 표본).
3. 3시장 병렬 자가발전(Ⅴ장) — 공통 모듈 위에서 KR·US 루프 연결.
4. `verify_auto_researcher` 7종을 **장 마감 후** 1회 전수 실행해 스냅숏 재수리 효과 확인.

---

### 부록 A. 산출물
| 파일 | 내용 |
|---|---|
| `research/gpt_27th_p0_report_20261120.md` | 이 보고서 |
| `research/verify_p0b_panel_parity.py` | 통합 리더 vs 활성 표 1:1 검증기 |
| `research/p0b_prune_verified.py` · `p0b_prune_20261120.json` | 검증된 샤드만 프루닝(재빌드 없음) + 결과 |
| `research/p0c_budget_audit_20261120.py` · `.json` | 102건 예산 감사(예산일·일별·자정 경계) |
| `research/p0d_net_audit_20261120.py` · `.json` | 순손익 합계·평균·검열 대조 |
| `research/p0f_snapshot_probe_20261120.py` · `.json` | 스냅숏 소요·동시 읽기 영향 실측 |
| `research/p0f_exitcode_verify_20261120.py` · `.json` | 종료 코드 규약 12항목 |
| `research/collab_ledger.py` | GitHub 협업 원장 게시기(monitor/coordination/research) |
| `research/.p0b_preprune_snapshot_20261120.db` | 프루닝 전 패널 스냅숏(403MB, 롤백용·비공개) |
