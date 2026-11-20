# GPT 제29차 통합 지시서 — P0-1~P0-7 (운영 복구·정합성·GitHub 스키마 보강) 실행 보고

- **작업 ID**: `29th-round / P0-1~P0-7 / Track A`
- **작성·실행 시각**: 2026-11-20 20:10 ~ 21:0x KST (모든 수치는 이 구간 실측)
- **대상 시장**: UP(업비트)·KR(한국)·US(미국) 3시장
- **트랙 구분**: 이 문서는 **트랙 A(운영 복구·정합성)**. `research/market_common.py`·`market_adapters.py`·`gpt_29th_p1_report_20261120.md` 는 트랙 B 소관으로 **건드리지 않았다**.
- **데이터 기준**: UP 1분봉 `2026-11-09 07:02 ~ 2026-11-20 20:5x` · KR 1분봉 `~2026-11-20 15:44` · KR 일봉 `2026-11-20` · US 1분봉 7세션 `2026-11-19`(KST 11-19 23:30~11-20 06:00)
- **성능 수치의 성격**: 본 보고의 전진검증 숫자는 **모의·재생 결과**이며 실거래가 아니다. `sum_trade_return_pct_points` 는 여러 가상 거래의 수익률 합이다.
- **공개 저장소**: `mju1417-max/market-dashboard` (PUBLIC). 비밀키·PAT·chat_id·개인정보·원본 DB·절대경로는 게시하지 않았다.

---

## 0. 요약 (결론 먼저)

| 항목 | 결론 | 실측 근거 |
|---|---|---|
| **P0-1** KR_S16 평가 정지 | **복구 완료.** 평가일을 자기 DB 최대일에서 유도하던 자기잠금 제거 → 거래소 캘린더+종가확정 시각으로 목표일 결정, 결측일 게이트·소급 적재 구현. 11-19·11-20 일봉 **1,248종목** 소급 적재(커버리지 99.9%/100%), 보유 3종목 손상 없이 유지, 신호일 11-18/11-19/11-20 재평가, **중복 0** | §1 |
| **P0-2** KR_VCP 일봉 복구 | **복구 완료.** 원인은 ①심볼 접미사(`.KS/.KQ`)로 네이버 요청 전건 실패 ②SQLite 연결 `check_same_thread=True` 로 워커 INSERT 전건 예외(조용히 삼켜짐) ③4회 재시도 상한. 청크·체크포인트·자동 재개·스톨 가드로 교체 → 11-19 **981/981**, 11-20 **980/981**, 미체결 신호(지어소프트) 규칙가 체결 | §2 |
| **P0-6** KR daily_rerun 예약 | **준비 완료(+테스트).** `scheduled-actions/🇰🇷-한국장-일간-재탐색/` 신설. 워터마크 확인 후 16:05~19:30 창에서 회차당 한 단계(`lab`→`report`→`surge`), 단계별 자체 타임아웃·이어하기·오류 복구, **항상 exit 1/2 Python-only**. 랩 단계 8/8 세션 완주 확인. **한계: `surge_miner` 는 183초·체크포인트 없음 → 60초 창에서 완주 불가(정직하게 not_finished 기록)** | §3 |
| **P0-5** US strategy_lab 280초 초과 | **해소.** 세션별 독립 청크(npz+manifest, `data_cutoff`·`feature_version`·`experiment_id`·세션기준 고정) + 재개 + 원자적 병합. **미국 7/7 완주**(data_cutoff `2026-11-19`, 후보 3,627,819). 일괄 vs 분할 정합성 **US·UP 모두 오차 0.0** | §4 |
| **P0-3** UP rotation 3회 오류 | **원인 단정 대신 기록·재현으로 전환.** 4회(17:29·18:59·19:29·19:59) 모두 **14.8~15.3초 = 원장 `busy_timeout 15000ms`** 에서 `status=error`. 예외 원문은 **단계 4를 'skipped' 로 덮어쓰던 버그** 때문에 사라졌다 → append-only `rotation_errors` 원장·폴백 파일로 원문·`sqlite_errorcode`·단계·시각·프로세스/WAL 상태 기록. 통제 실험으로 **SQLITE_BUSY(5)** 재현. **5분 수집과의 동시 중첩은 로그상 없음**(수집이 0~32초 먼저 끝남). busy 30초·짧은 트랜잭션·백오프 재시도·비차단 실행락·스케줄 지터 적용. SIGKILL 복구 시험 통과 | §5 |
| **P0-4** GEN3 WAL `-shm` | **안전 읽기 경로 구현·검증.** `mode=ro` 실패 시 백오프 재시도 → **바이트 스냅숏(db+wal+shm)** 사본으로 읽기. **`immutable=1` 무검증 우회는 쓰지 않음.** 잠금 재현(직접 ro 실패) → 재시도 성공·장기 잠금 시 스냅숏으로 실제 봉·시가 읽기 확인. 채점·대시보드 양쪽에 적용, 실패는 rc=2 로 표면화 | §6 |
| **P0-7** 24,480봉·research={}·TKST | **규명·수정.** 차이는 데이터 손실이 아니라 **정의 차이**(`COUNT(*)` vs `SUM(fetch_log.candles_saved)`) + 시점 차이: **15,725(UP) + 9,369(KR) − 614(US) = 24,480**. 상태에 두 수치·쿼리·DB 식별자를 함께 싣도록 수정. `research={}` → 3시장 × 22필드(null 은 사유 표기). `TKST+09:00` 생산 코드 수정 + 기존 31행 정규화·검증 | §7 |
| **검증** | `verify_kr_recovery` **14/14 PASS** · `verify_up_wal_safe` **PASS** · `verify_rotation_recovery` **PASS** · US/UP 분할 정합성 **오차 0.0** · US 랩 7/7 · KR 랩 8/8 | §8 |

---

## 1. P0-1 — KR_S16 평가 정지 복구

### 1-1. 무엇이 멈춰 있었나 (수리 전 실측)

| 항목 | 수리 전 값 |
|---|---|
| `forward_trades_kr_s16.json` `last_status.as_of` | **2026-11-18** (오늘 11-20) |
| `kr_s16_data.db` daily 최대일 | 2026-11-18 (1,942종목) |
| supply 최대일 | 2026-11-17 |
| `bars_target` / `bars_fetched` | 0 / 0 (0.7초 no-op) |
| `signal_date` | 2026-11-17 (11-18 이후 미평가) |

원인은 코드 한 줄이다. `run_daily()` 가 `T = args.force_date or days[-1]` 로 **평가일을 자기 일봉 DB 의 최대일**에서 유도했다. 하루 수집이 밀리면 T 도 같이 멈추고, 수집 대상도 그 옛 날짜(`fetch_daily_bars(..., target_date=T)`)라서 새 봉을 영원히 받지 않는 **자기잠금**이 된다.

### 1-2. 무엇을 바꿨나

1. **거래소 캘린더 신설** — `research/krx_calendar.py`(신규, KR_VCP 와 공용): 주말+공휴일(2025~2027, 설·추석·대체·임시공휴일 포함). `target_eval_date()` = 평일 15:46(KST) 이후면 오늘, 그 전·휴장일이면 직전 영업일.
2. **목표일과 데이터 준비를 분리** — 목표일은 캘린더에서, 실제 데이터는 **완전성 게이트**(유니버스 종목의 90% 이상 봉 보유)로 판정. 캘린더에 없는 임시휴장은 "그날 봉이 0건"이 확인되면 `meta.nontrading:<날짜>` 로 남기고 하루 물러난다.
3. **소급 적재(재개 가능)** — `(DB 최대일, 목표일]` 의 거래일을 모두 대상으로, 그중 하나라도 봉이 없는 종목만 종목당 1회(8거래일치) 요청. 남은 종목은 **DB 상태에서 다시 계산**되므로 예산이 모자라면 다음 회차가 이어받는다.
4. **결측 표시** — 커버리지가 게이트 미만인 날은 `data_gate=incomplete/pending`, `gap_dates`, `state.data_gaps[{date, coverage_pct, have, target, action:backfill_pending}]` 로 남기고 그날 신호 평가를 **보류**한다.
5. **신호일 따라잡기** — 마지막 처리일 이후 확정 거래일을 순서대로(회당 최대 4일) 평가. 첫 실행은 과거를 소급하지 않고 평가일 하나만 본다(원 설계 유지).
6. **수급 미공표일 보류** — 외인 순매수는 마감 후 다음 날 공표된다. 가격 조건을 통과한 종목의 20% 이상이 수급 결측이면 그날을 `deferred(supply_not_published)` 로 두고 **처리 완료로 찍지 않는다**(11-20 이 이 경우였다).
7. `check.sh` — KR_S16 을 하루 1회(15:46~16:15)에서 **15:46 이후 매 회차**로 바꿨다(끝난 뒤 실행은 0.3초 no-op). 남은 시간에서 예산을 깎아 60초 상한을 지킨다.

### 1-3. 수리 후 실측 (11-19·11-20 소급 적재)

```
backfill bars(2026-11-19,2026-11-20): 대상 1248 · 수집 1248 · 미완 0 (58.5s)
signals 2026-11-18: 가격조건 183 → f5>0 85 (수급 보강 183)
signals 2026-11-19: 가격조건 198 → f5>0 89 (수급 보강 90)
signals 2026-11-20: 가격조건 131 → f5>0 58 (수급 보강 131)   ← 수급 공표 후 재평가
entries: 6건 체결
```

| 검증 항목 | 결과 |
|---|---|
| 일봉 커버리지 | 11-19 **1,247/1,248 (99.9%)**, 11-20 **1,248/1,248 (100.0%)** → `data_gate=complete` |
| 평가일 | `as_of` **2026-11-20**, `target_date` 2026-11-20 (DB 최대일과 무관하게 전진) |
| 기존 보유 3종목 | 빙그레 78,400 / 바디텍메드 9,620 / 가비아 48,600 — **가격·진입일(11-18)·청산예정일(2026-12-16) 그대로**, 재계산 후에도 동일 |
| 신규 체결 | 11-18 신호 → 11-19 종가 3건(`late_fill_days=1`), 11-19 신호 → 11-20 종가 3건. 진입가는 **규칙가(D+1 종가)** 그대로 |
| 보유/대기 | 보유 9 · 대기 58(11-23 진입 예정) · 청산 0 |
| 중복 | 포지션 id 중복 **0**, 종목 중복 **0** |
| 멱등 | 같은 상태 재실행 **0.3초**, 신호·체결 증가 0 |

`--self-test` OK, `verify_kr_recovery_20261120.py` 항목 A·B·C 전부 PASS(§8).

---

## 2. P0-2 — KR_VCP 일봉 복구

### 2-1. 무엇이 1종목이었나

`kr_vcp_data.db` 의 날짜별 종목 수: **11-18 988종목 → 11-19 1종목 → 11-20 1종목**. `meta.attempts:2026-11-20 = 4`(상한 소진), `as_of` 는 1종목짜리 날(11-20), `last_evaluated_date` 는 11-18 에 묶여 있었다.

### 2-2. 원인 3개 (추정 아님, 재현으로 확인)

1. **심볼 접미사** — 이 트랙 DB 의 심볼은 씨앗(`research/history_data.db`)에서 온 야후식 `000020.KS` / `051160.KQ` 인데, 수집 URL 은 네이버 fchart 로 **6자리 코드**만 받는다. 접미사가 붙은 채 요청해서 **980종목 전건 실패**했다(`naver_code()` 로 접두 6자리만 보내도록 수정).
2. **SQLite 연결 스레드 제약** — 수집은 워커 스레드가 하는데 연결이 `check_same_thread=True` 였다. 모든 INSERT 가 `ProgrammingError` 로 죽었고, `Fetcher.run` 의 `except Exception: ok=False` 가 그 예외를 **완전히 삼켜** heartbeat 에는 "완료 0 / 남음 980" 만 남았다(`check_same_thread=False` + 쓰기 직렬화 락, 그리고 실패 원문 표본을 `Fetcher.errs` 에 남기도록 수정).
3. **4회 재시도 상한 + `as_of=latest_date`** — 상한에 걸리면 그날 더 이상 시도하지 않고, 종목 1개짜리 날을 `as_of` 로 잡아 평가 기준일이 흔들렸다.

### 2-3. 무엇으로 바꿨나

- **커버리지 기준 확정일** — `last_complete_date()` / `complete_dates()`: 종목 수가 `max(400, 유니버스×0.90)` 이상인 날만 확정 거래일. 1종목짜리 날은 `as_of` 가 되지 않는다.
- **증분·청크·체크포인트 수집** — 목표일을 캘린더에서 정하고, 빠진 날짜를 모두 대상으로 종목 단위 배치를 돈다. 날짜별 진행은 `meta.collect:<날짜>` = `{attempts, have, target, coverage, stall, status}` 로 남기고, 남은 종목은 **다음 회차가 DB 에서 다시 계산**해 이어받는다.
- **스톨 가드** — 진전 없는 시도가 3회 연속이면 `status=stalled`, 응답 자체가 없으면 `nontrading:<날짜>` 로 판정(무한 재시도 금지). 상한은 4회 → **24회**(5분 회차 2시간).
- **미완결 일봉 진입 금지 유지** — 커버리지 < 90% 인 날은 신호 평가에서 제외하고 `low_coverage_dates` 에 남긴다.

### 2-4. 수리 후 실측

```
daily bars(2026-11-19): 대상 980 · 완료 980 · 남음 0
daily bars(2026-11-20): 대상 1 · 완료 1 · 남음 0
scan 2026-11-19: 980종목 중 VCP 0건 / 2026-11-20: 980종목 중 VCP 1건
KR_VCP: as_of 2026-11-20 · 신호 1 · 체결 1 · 청산 0 · 대기 1 · 보유 1 (51.4s)
```

| 검증 항목 | 결과 |
|---|---|
| 커버리지 | 11-19 **981/981 (100%)**, 11-20 **980/981 (99.9%)** → `collect:*` status=complete |
| 미체결 신호 후속 처리 | 지어소프트(`051160.KQ`) 신호 11-18 → **11-19 시가 15,300원 체결**(규칙가 그대로) |
| 신규 신호 | 377450 (11-20) 1건 — 11-23 시가 진입 대기 |
| 중복 진입 | 보유 1 · 대기 1 · 중복 **0**, 재실행 0.8초 no-op |
| 테스트 | 커버리지 가드·미체결 후속·중복 방지 3항목 PASS(§8) |

---

## 3. P0-6 — KR daily_rerun 독립 Python 예약

### 3-1. 왜 별도 액션인가

지금까지 일일 재탐색은 **AI 보고 회차가 열려야만** 돌았다(`daily_rerun.py`, 한국 401초 = 폭등공식 183.8 + 랩 153.7 + 요약 64.3). 보고가 빠진 날은 재탐색이 통째로 비었다.

### 3-2. 만든 것

- `scheduled-actions/🇰🇷-한국장-일간-재탐색/action.md` + `check.sh`(신규, 5분 간격·항상 exit 1/2)
- `research/kr_daily_step.py`(신규) — 회차당 **한 단계**만 실행하는 러너
- `research/daily_rerun.py`·`strategy_lab.py` — 산출물 경로를 env(`DAILY_OUT`·`LAB_OUT_DIR`·`LAB_CHUNK_ROOT`·`LAB_DIR`)로 돌릴 수 있게 수정(샌드박스는 자기 `state/` 밖이 읽기전용)

| 단계 | 하는 일 | 자체 타임아웃 | 이어하기 |
|---|---|---|---|
| `lab` | `strategy_lab.py --market KR --session-chunks --finalize` (세션별 청크) | 엔진 예산 22~30초 / 프로세스 44~52초 | 청크 단위, 8/8 까지 여러 회차 |
| `report` | `daily_rerun.py --market kr --skip-run` → `state/daily/kr_latest.md` | 30초 | 단계 완료 후 재실행 안 함 |
| `surge` | `surge_miner.py kr` | 45초 | **하루 1회 시도**, 미완결이면 `not_finished` |

가드: 평일 16:05~19:30 창, `kr_1m.db` 마지막 봉 날짜가 오늘일 때만, 같은 단계 3회 연속 실패에서만 `exit 2`(그 외 `exit 1`).

### 3-3. 테스트 결과 (실제 실행)

| 확인 | 결과 |
|---|---|
| 창 밖 회차 | `outside_window`, 0초, exit 1 |
| 워터마크 | `kr_1m.db` 마지막 봉 `2026-11-20` 확인 |
| `lab` | 6회차에 걸쳐 **8/8 세션 완주**, `state/strategy_lab_KR.json` = `experiment_id lab_KR_20261120_3f1a339d`, data_cutoff 2026-11-20, 후보 **1,344,187**, 회차당 25~34초(잘림 없음) |
| `report` | 0.3초, `state/daily/kr_latest.md` 생성(폭등공식 선택·랩 OOS 통과 0개·전진 장부 요약) |
| `surge` | 실행되지만 45초에서 잘림 → `not_finished`(183초 엔진·체크포인트 없음). 실행 중 발견한 **`ulimit -f 512MB` 초과(OSError 27)** 는 직전 세대 파일을 state 로 복사하고 `SURGE_TMP` 를 state 로 돌려 해결 |

**남은 한계(정직하게 기록)**: 폭등공식(`surge_miner`)은 자체 체크포인트가 없어 60초 예약 창에서 완주할 수 없다. 지금은 하루 1회 시도 후 `not_finished` 로 남기며, **폭등공식의 일일 자동 갱신은 아직 완결되지 않았다**. 다음 수리 과제: `surge_miner` 에 단계 체크포인트(`--stage`)를 넣어 창 단위로 나누는 것.

등록은 메인 에이전트가 수행한다: `scheduled-action register scheduled-actions/🇰🇷-한국장-일간-재탐색` (테스트: `scheduled-action check …` → **quiet, exit 1**, 광고된 폴더 경로 그대로).

---

## 4. P0-5 — US strategy_lab 280초 초과 해소

### 4-1. 원인 확인

`strategy_lab.py` 는 세션 전체를 한 번에 돌리고 **마지막에만** JSON 을 썼다. 6세션 실행이 126초였고, 7세션·2,072종목이 되자 280초 셸 상한을 넘겨 **산출물 0**. `strategy_lab_US.json` 은 6세션(nd=6) 상태로 남아 최종 성적처럼 보였다.

### 4-2. 바꾼 구조

- **세션 = 청크**: `research/lab_chunks/US/<experiment_id>/sess_XX_<day>.npz` + `.npz.json` manifest. manifest 에 `data_cutoff`·`feature_version`·`experiment_id`·`session_basis(W, trade_col, tz_off, min_day_frac, cost, liq)`·`n_candidates`·`seconds` 고정.
- **재개**: 이미 있는 청크는 건너뛴다. 회차 예산(`--max-seconds`) 안에서 다음 세션을 시작할지 판단(직전 세션 실측 시간으로 예상).
- **원자적 병합**: `--finalize` 는 **모든 세션이 같은 실험 ID 로 완료된 경우에만** 임시파일 → `os.replace` 로 `strategy_lab_US.json` 교체. 미완결이면 아무것도 쓰지 않고 이전 검증본을 유지한다.
- **격자 캐시도 재개 가능**: 3,000종목 격자 생성이 회차 상한을 넘던 문제 → 종목 단위 진행 파일(`<MKT>_grid_progress.json`)로 이어 만들기. 세션 구성(새 거래일·종목)이 바뀌면 캐시를 자동 무효화.
- **세션 기준 고정(정합성 전제)**: 쿨다운이 `col = day*W + smin` 기반이라 **세션 경계를 넘어 30분 쿨다운이 붙어 있었다**(전일 마지막 신호가 다음날 첫 신호를 지움). 쿨다운을 `(종목, 세션)` 단위로 리셋하도록 고쳤다 — 일괄 실행과 분할 실행이 **같은 정의**를 쓰게 되어 정합성 검증이 성립한다. (결과 수치가 이전 게시본과 미세하게 달라지는 이유다.)

### 4-3. 미국 7/7 완주 (최신 data_cutoff)

| 항목 | 값 |
|---|---|
| experiment_id | `lab_US_20261119_11d1253a` |
| data_cutoff | **2026-11-19** (마지막 완결 세션, 20776) |
| 세션 | 7/7 — 20768·20769·20770·20773·20774·20775·20776 |
| 후보 | **3,627,819** (세션별 547,443 / 571,876 / 577,058 / 555,388 / 552,624 / 448,594 / 374,836, 각 33~41초) |
| 최종 병합 | 6초, `strategy_lab_US.json` `complete=true`·`provenance.completeness="7/7"` |
| 기준선 f240 | IS n=85,174 승39.2% ev−0.2538 / OOS n=51,661 승31.4% ev−0.3456 |

### 4-4. 일괄 vs 분할 정합성

| 시장 | 세션 | 일괄 후보 | 분할 후보 | 비교 셀 | 최대 ev 차이 |
|---|---|---|---|---|---|
| US | 20775·20776 | 823,430 | 448,594 + 374,836 | **3,801** | **0.0** |
| UP | 20766·20767 | 195,924 | 99,980 + 95,944 | **3,465** | **0.0** |

→ 세션 분할 집계가 일괄 집계와 **수치적으로 동일**(n·승률·기대값 전부 일치, 허용오차 2e-3 미사용).

---

## 5. P0-3 — UP generation_rotation 3회(실제 4회) 오류

### 5-1. 관측과 첫 번째 결함

`rotation_runs` 실측: **17:29:29(14.91s) · 18:59:24(14.81s) · 19:29:23(15.25s) · 19:59:12(15.14s)** 네 회차가 `status=error`, `new_rows=0`, `panel=None`, heartbeat `sec=None`. 14.8~15.3초는 모든 원장 연결의 **`busy_timeout=15000ms`** 와 정확히 일치한다.

그런데 **예외 원문이 원장에 없었다.** 이유: 엔진 예외 시 단계 4 를 `error` 로 쓴 뒤, 아래 분기(`elif status != "no_new_data"`)가 같은 기본키로 **`skipped` 를 덮어썼다**(`insert or replace`). 그래서 "무엇이 막혔는지"를 사후에 알 수 없었다 — GPT 지적 그대로다.

### 5-2. 재현 실험 (가정 금지)

| 실험 | 조건 | 결과 |
|---|---|---|
| 기준선 | 경합 없음(사본 원장) | `no_new_data`, 18.4초, 오류 0 |
| 원본 DB 쓰기 잠금 | `upbit_1m.db` 사본에 `BEGIN IMMEDIATE` 유지 | **성공**(7.2초) — 원본 읽기는 막히지 않음 |
| 원장 쓰기 잠금 | `knowledge.db` 사본에 `BEGIN IMMEDIATE` 유지 | **`sqlite3.OperationalError: database is locked`, `sqlite_errorcode=5(SQLITE_BUSY)`, 16.3초**, phase `insert_run` |
| 읽기 스냅숏 유지 | `mode=ro` 연결로 `BEGIN`+장기 읽기 | **성공** — 읽기 잠금은 쓰기를 막지 않음 |
| 두 회차 동시 | 같은 원장에 동시 실행 | 첫 회차 성공·둘째 `busy_skip`(수리 후) |

→ **원장 쓰기 잠금만이 15초 실패 서명을 만든다.** 수리 후에는 그 사실이 `rotation_errors` 에 남는다.

### 5-3. 5분 수집과의 실제 중첩 (로그 전수 대조)

각 액션의 `check_heartbeat.log` 로 시작·종료 시각을 복원해 실패 회차와 겹치는 구간을 확인했다.

| 실패 회차 | 5분 수집(📈) 구간 | 관계 |
|---|---|---|
| 17:29:28 | 17:28:08 ~ 17:28:56 | 수집이 **32초 먼저 끝남** |
| 18:59:23 | 18:58:12 ~ 18:59:00 | 23초 먼저 끝남 |
| 19:29:22 | 19:28:32 ~ 19:29:21 | **1초 전 종료** |
| 19:59:10 | 19:58:22 ~ 19:59:10 | **동시 종료** |

→ **동시 중첩은 없었다.** 5분 수집은 다른 DB(`upbit_1m.db`)에 쓰고 원장(`knowledge.db`)에는 쓰지 않는다. 따라서 "5분 수집과의 SQLite 락 충돌"로 단정하지 않는다. 실패가 수집 회차 **직후**에 몰린 것은 사실이므로, 원인 후보로 남기고 **회차마다 원문·프로세스 목록·WAL/-shm 상태를 원장에 남겨 다음 발생 시 확정**한다.

### 5-4. 적용한 수정

1. **원장(append-only)** `rotation_errors`: `rotation_id, phase, step_no, exc_type, exc_message, sqlite_errorcode, sqlite_errorname, db_role, db_path, started_at, failed_at, elapsed_s, note(ctx=프로세스 목록·WAL/-shm 크기)`. 원장 자체가 잠겨 못 쓰면 `rotation_errors_fallback.jsonl` 로 떨어진다.
2. **덮어쓰기 금지**: 단계 4 가 `error` 면 아래 분기가 건너뛴다.
3. **대기·재시도**: 원장 `busy_timeout` 15s → **30s**(`generation_rotation`·`auto_researcher`), 쓰기는 `exec_retry`(BEGIN IMMEDIATE + 백오프 3회).
4. **중복 실행 방지**: 원장별 비차단 파일 락(`flock`). 이미 도는 회차가 있으면 `busy_skip`(15초 대기 대신 즉시 종료).
5. **좀비 정리**: 이전에 SIGKILL/timeout 으로 남은 `running` 행은 다음 회차가 `interrupted` 로 정리(삭제 없음).
6. **스케줄 충돌 완화**: `check.sh` 에 0~4초 지터, 잠금성 실패는 `lock-skip` 으로 heartbeat 에 남기고 **exit 1(quiet)** — AI 호출 낭비 방지. 실제 오류만 2.

### 5-5. 복구·중복 실행 안정성 증명 (사본 원장)

| 시험 | 결과 |
|---|---|
| 두 회차 겹침 | 1번째 `no_new_data`, 2번째 **`busy_skip`** (15초 대기 없음) |
| 실행 1.5초 뒤 SIGKILL | `integrity_check=ok`, 패널 행수 **105,226 그대로**, `rotation_runs` 중복 0, `running` 0 |
| 이어서 정상 회차 | `no_new_data`, runs 19, `running=0`, integrity ok |
| 운영 적용 후 | heartbeat `status=no_new_data … rot=no_new_data` (13초), 오류 0 |

---

## 6. P0-4 — GEN3 채점의 WAL `-shm` 안전 읽기

### 6-1. 문제

채점(`📊-업비트-전진검증-채점`)은 `forward_tracker.py` 로 `upbit_1m.db` 를 `mode=ro` 로 연다. WAL DB 는 `-shm` 접근이 필요한데, 예약 check 샌드박스는 자기 `state/` 밖이 읽기전용이라 `-shm` 생성·복구가 막히면 `unable to open database file` / `database is locked` 로 **그 회차 채점을 통째로 잃는다**(2026-11-20 관측).

### 6-2. 안전 읽기 경로 (`research/safe_ro.py` 신규)

1. `mode=ro` 로 열고 `PRAGMA query_only=1` + 최소 조회로 실제 읽힘을 확인
2. 실패하면 **0.2/0.5/1.0/2.0초 백오프 재시도**(WAL 재시작·체크포인트 경합은 대개 1초 내 해소)
3. 그래도 실패하면 **바이트 스냅숏**: `db + -wal + -shm` 을 쓰기 가능한 `state/db_snapshots/` 로 복사해 그 사본을 연다(실제 봉·시가를 그대로 읽는다. 통계 JSON 대체 아님)
4. 결과(경로·재시도 횟수·마지막 오류·스냅숏 여부)를 `db_read_health.json` 에 누적
5. **`immutable=1` 은 쓰지 않는다** — 파일이 변하지 않는다고 가정해 잠금·`-shm` 검증을 건너뛰는 무검증 우회다.

적용 위치: `forward_tracker.CandleDB`(GEN3·ZVOL·GEN2 채점) · `export_dashboard_json._bars_count`(대시보드) · `monitor_research`/`reconcile`. 열기 실패는 이제 **rc=2 로 표면화**한다(예전엔 조용히 0).

### 6-3. 재현·검증 (`verify_up_wal_safe_20261120.py`)

| 시나리오 | 결과 |
|---|---|
| 다른 프로세스가 `locking_mode=EXCLUSIVE`+쓰기 트랜잭션(1초) | 직접 `mode=ro` → **`OperationalError: database is locked`** / safe_ro → **재시도 1회 후 성공**, 50행 정상 |
| 같은 잠금 6초 유지 | safe_ro → **스냅숏 모드**, 50행·min_open 100.0·max_close 149.5 정상 |
| 운영 DB 정상 경로 | `mode=ro`, 봉 수 실측, 마지막 봉 `KRW-AI 2026-11-20T11:22Z open 17.9 / close 17.8` |
| 채점 액션 실제 실행 | 3트랙 모두 `mode=ro`, `ledger ok_total=245`, `consecutive_fail=0` |

---

## 7. P0-7 — `24,480봉` 규명 · `research={}` 보강 · `TKST+09:00` 정규화

### 7-1. 24,480봉 차이 — 재현과 산술

두 수치는 **다른 정의**를 세고 있었다.

- 19:45 연구 스냅숏 = `SELECT COUNT(*) FROM candles_1m` (PK 중복 제거된 실제 행)
- 19:55 `monitor/latest.json` = `SELECT SUM(candles_saved) FROM fetch_log` (수집기가 "받아 저장했다"고 센 누적 카운터)

`research/candle_count_reconcile_20261120.json` (20:48 KST 실측)

| 시장 | COUNT(*) (bars) | SUM(candles_saved) | Δ | 19:45 발표와 일치 | 19:55 발표와 일치 |
|---|---|---|---|---|---|
| UP | 1,545,203 | 1,525,843 | **+19,360** | COUNT(*) (1,538,894) | SUM (1,523,169) |
| KR | **4,688,350** | **4,678,981** | **+9,369** | COUNT(*) — **정확히 일치** | SUM — **정확히 일치** |
| US | 4,577,946 | 4,577,946 | **0** | COUNT(*) | SUM(시점차) |

**24,480 = 15,725(UP 정의차) + 9,369(KR 정의차) − 614(US 시점차).** 즉 **원본 소실이 아니라 정의·시점 차이**다. KR 은 두 발표값이 각각 두 정의와 **한 자리도 틀리지 않고** 일치해 규명이 확정적이다. UP·KR 에서 SUM 이 작은 이유는 `fetch_log.candles_saved` 가 심볼별 누적 카운터로 재설정·절단되며 실제 저장 행보다 적게 세기 때문이고, US 는 두 값이 같아 차이가 없다.

**수정**: 상태(=`data/status.json`, `monitor/latest.json`)에 두 수치를 함께 싣는다 — `candles_1m`(headline), `candles_1m_query`, `candles_1m_saved`, `candle_count_delta`, 그리고 `candle_count_audit{db_file, db_size, db_mtime, universe_def, bars_query, saved_query, bars_actual, generated_at}`. 이제 어떤 값이 어떤 쿼리·시점·DB 에서 나왔는지 게시물만 보고 재현할 수 있다.

### 7-2. `monitor/latest.json` `research={}` → 3시장 Ⅴ.2 필드

`research/monitor_research_20261120.py`(신규)가 실제 DB·원장에서 22개 필드를 채운다. 미구현 항목은 **null + 사유**(`status_fields`)로 남긴다.

| 필드 | UP | KR | US |
|---|---|---|---|
| data_cutoff | 1분봉 최종 봉 | 2026-11-20T15:44 | 1분봉 최종 봉 |
| last_complete_session | 최종 봉 날짜 | 2026-11-20 (S16 as_of) | **2026-11-19** |
| candle_count · universe | COUNT(*) · fetch_log | 동일 | 동일 |
| generation_id | research_generations 최신 | `lab_KR_20261120_3f1a339d` | `lab_US_20261119_11d1253a` |
| research_status | `rotation:<상태>` | `daily_track_active` | `lab_complete` |
| candidate_count | strategy_registry (candidate) | 1,344,187 (랩) | 3,627,819 (랩) |
| shadow measured/estimated | 장부 기반 / null | 248 / null | 장부 기반 / null |
| TP/FP/censored · calibration · uplift · price_gain_error · latency | **null** + `not_measured/not_implemented` 사유 | 동일 | 동일 |
| code_sha | auto_researcher+rotation 해시 | KR 트래커 해시 | strategy_lab `code_sha` |

### 7-3. `TKST+09:00` 정규화

- **생산 코드 결함**: `build_collab_ledger_20261120.py` 의 `at=parts[0].replace(" ", "T") + "+09:00"` — 입력이 `"2026-11-20 04:04:55 KST"` 라 `" KST"` 를 지우지 않아 **`2026-11-20T04:04:55TKST+09:00`** 가 나갔다. `replace(" KST","")` 후 `T` 치환으로 수정.
- **기존 게시물**: `monitor/telegram_reports.jsonl` 31행 전부 같은 결함 → 각 행의 `at` 을 `YYYY-MM-DDTHH:MM:SS+09:00` 으로 정규화하고 **전 행 ISO8601 파싱 검증**(§9 게시물 참조). 본문·id·유형·성공여부는 그대로 보존(삭제 없음).

---

## 8. 검증 목록

| 스크립트 | 항목 | 결과 |
|---|---|---|
| `research/verify_kr_recovery_20261120.py` | 캘린더 5 · S16 자기잠금/게이트/결측표시 3 · S16 멱등 2 · VCP 커버리지 가드 1 · VCP 후속체결/중복방지 3 | **14/14 PASS** |
| `research/verify_up_wal_safe_20261120.py` | 짧은 잠금 재시도 · 장기 잠금 스냅숏 · 운영 DB 실제 값 | **PASS** |
| `research/verify_rotation_recovery_20261120.py` | 겹침 busy_skip · SIGKILL 무결성/행보존/중복0 · 이어서 완주 | **PASS** |
| `strategy_lab.py --parity` | US 2세션 · UP 2세션 | **오차 0.0** |
| `forward_tracker_kr_s16.py --self-test` / `forward_tracker_kr_vcp.py --self-test` | 규칙·체결·정산·자기검증 | **PASS / PASS** |
| `scheduled-action check` | 🇰🇷한국주식 · 🔬자가발전 · 📊업비트채점 · 🇰🇷한국장-일간-재탐색 | 전부 **quiet(exit 1)** |

## 9. 남은 것 (숨기지 않고 기록)

1. **`surge_miner` 폭등공식 일일 자동 갱신 미완결** — 183초·체크포인트 없음. 예약 창(60초)에서 완주 불가. 단계 체크포인트(`--stage`) 구현이 다음 과제. 현재는 `not_finished` 로 정직하게 기록.
2. **UP rotation 4회 실패의 최종 원인 미확정** — 원장 쓰기 잠금이 15초 실패를 만든다는 것과, 5분 수집과 **동시 중첩이 없었다**는 것까지 실측했다. 실제 잠금 보유자는 다음 발생 시 `rotation_errors.ctx`(프로세스 목록·WAL/-shm 크기)로 확정한다. 완화(30초 대기·재시도·실행락·지터)는 적용했다.
3. **TP/FP/censored·calibration·uplift·price_gain_error·latency** — 3시장 모두 `null` + `not_measured/not_implemented`. 실현 라벨 집계가 아직 없다.
4. **UP 랩(`strategy_lab_UP.json`)·UP GEN 계보** — 이번 회차는 US·KR 만 재채점했다. UP 랩은 이전 산출물(provenance 없음)이라 `generation_id` 가 null 이다.
5. **쿨다운 세션 스코프 변경** — 정합성 검증을 위해 의미를 고쳤다(세션 경계 리셋). US 는 새 정의로 7/7 재계산했고, KR 랩도 새 정의(8세션)로 계산했지만 `research/strategy_lab_KR.json`(정본)은 아직 이전 파일이라 `state/` 사본을 우선 인용했다. 정본 동기화는 다음 회차.
6. `KR_VCP` 는 상시 수집이 5분 창에서 계속 이어지는 구조라, 11-20 이후 날짜도 같은 방식으로 자동 복구된다(상한 24회·스톨 가드).

## 10. 변경 파일

- 신규: `research/krx_calendar.py` · `research/safe_ro.py` · `research/kr_daily_step.py` · `research/reconcile_candle_counts_20261120.py` · `research/monitor_research_20261120.py` · `research/verify_kr_recovery_20261120.py` · `research/verify_up_wal_safe_20261120.py` · `research/verify_rotation_recovery_20261120.py` · `research/rotation_lock_probe_20261120.py` · `scheduled-actions/🇰🇷-한국장-일간-재탐색/{action.md,check.sh}`
- 수정: `research/forward_tracker_kr_s16.py` · `research/forward_tracker_kr_vcp.py` · `research/strategy_lab.py` · `research/generation_rotation.py` · `research/auto_researcher.py` · `research/forward_tracker.py` · `research/export_dashboard_json.py` · `research/daily_rerun.py` · `research/surge_miner.py` · `research/collab_ledger.py` · `research/build_collab_ledger_20261120.py` · `scheduled-actions/🇰🇷-한국주식-패턴-분석/check.sh` · `scheduled-actions/🔬-자가발전-패턴-연구/check.sh`
- 무변경(트랙 B): `research/market_common.py` · `research/market_adapters.py` · `research/gpt_29th_p1_report_20261120.md`
