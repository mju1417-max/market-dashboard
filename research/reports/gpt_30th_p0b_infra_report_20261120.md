# GPT 제30차 P0-B · P0-C · P0-E 구현·검증 보고서 (트랙 2)

- 작성: 2026-11-20 22:00 KST (Pickle, 트랙 2 — 안전 읽기·잠금 계측·모니터 복구)
- 대상: GPT 제30차 통합 지시서(`#8549`) 중 **P0-B(WAL 안전 읽기) · P0-C(원장 잠금) · P0-E(연구 모니터)** + 6절 원장 ACK
- 범위 밖(트랙 1 담당, 이 보고서는 건드리지 않음): `surge_miner.py` · `strategy_lab_KR.json` · `KR_S16` · `KR_VCP` · `🇰🇷-한국장-일간-재탐색`
- 운영 DB·원본 코드는 공개 저장소에 올리지 않았다. 아래 수치는 전부 실측이며, 미측정·미구현은 null 로 남겼다.

## 0. 결론 요약

| 항목 | 상태 | 실측 증거 |
|---|---|---|
| P0-B WAL 일관 복제본 게시 | **완료** | `verify_wal_backup_20261120.py` 6/6 PASS, 운영 복제본 1,556,035행(22:18 KST)·`quick_check`/해시/바이트 정합 |
| P0-B 읽기 경로 전환 | **완료** | 채점기 heartbeat `db=replica`(실측), 대시보드 `read_source=replica`, `safe_ro` 바이트 복사 제거 |
| P0-C 잠금 계측 | **완료** | BUSY(5) 1건에 대기 400/800/1200ms·재시도 2·WAL 53,592B·checkpoint 상태·보유자 pid 기록 확인 |
| P0-C busy_skip / no_new_data 분리 | **완료** | busy_skip 1건(저널), no_new_data 11건(run_key 보유·new_rows 0), 회차 표에 섞이지 않음 |
| P0-C 누락 데이터 재수집 보장 | **완료** | 체크포인트 30분 되돌림 + 패널 336행 삭제 → 다음 회차가 336행 재적재·체크포인트 복귀 |
| P0-E `research={}` 원인·수정 | **완료** | 원인 코드 경로 확정(아래 3절), 원격 raw GET 에서 3시장·34필드 확인 |
| 6절 원장 ACK | **완료** | `coordination/gpt_to_pickle.jsonl` 에 `G30-P0-B`·`G30-P0-C`·`G30-P0-E`·`G30-VI` 4행 |
| 전체 구조 검사(`quick_check`)의 매회 실행 | **미완(의도적 분리)** | 콜드 캐시 실측 106초 → 5분 회차에 넣을 수 없어 light(매회)/deep(테스트·정기)로 분리 |

## 1. P0-B — SQLite WAL **일관 복제본**으로 전환

### 1.1 지적은 실측으로 확인됐다

GPT 지적(“db+-wal+-shm 순차 바이트 복사는 단일 시점 일관성을 보장하지 않는다”)을 그대로 재현했다.

| 재현 | 결과 |
|---|---|
| WAL 모드 DB(200행 커밋, 체크포인트 전)에서 **본체 파일만** 복사 | `naive_file_copy_rows = error:OperationalError` — 읽히지도 않는다 |
| 같은 DB 를 `Connection.backup()` 으로 복제 | 200행 전부 보존, `quick_check=ok`, `data_hash_equal=true`, `bytes == pages×page_size` |

즉 WAL 에만 있는 커밋분은 파일 복사로는 **사라지거나 깨진 DB**가 되고, `-shm` 사본은 일관성 증거가 될 수 없다.

### 1.2 설계 — 씨앗(backup) + 5분 증분 동기화

이 환경에서 242MB 전체 복사는 **한가할 때 4초, 수집기 자신의 쓰기와 겹치면 47~106초**였다(같은 명령 3회 실측: 4.14s / 3.91s / 47.37s, `quick_check` 콜드 106s). 5분 회차(60초 상한, 평시 46~49초 소요)에 매번 전체 복사를 넣는 것은 불가능하다. 그래서 두 단계로 나눴다.

| 단계 | 파일·경로 | 방식 | 실측 |
|---|---|---|---|
| 씨앗/재구축 | `wal_replica.py --publish` | `Connection.backup()` → 임시파일 → `PRAGMA journal_mode=DELETE`(=-wal/-shm 없이 단독 읽기) → `fsync` → `os.replace` (원자) | 7.49초(1.55M행, light) |
| 5분 갱신 | `wal_replica.py --sync` | 수집기 state/ 안에서 **트랜잭션 증분**(`INSERT OR IGNORE`, 종목별 마지막 봉−15분 구간 + `fetch_log` 전량, 60종목 단위 commit, 회전 커서) | 302종목 **2.7초**(22:08 실측) · 한가할 때 0.24~0.9초 |
| 원자성 | — | 씨앗은 `tmp+fsync+os.replace`, 증분은 SQLite 트랜잭션(독자는 완결된 상태만 본다) | — |

수집기는 `INSERT OR IGNORE` 로만 봉을 적재하므로(기존 행을 고치지 않는다) 종목별 마지막 봉 이후만 옮기면 내용이 같다. `fetch_log`(종목 1행)는 매회 전량 교체한다.

### 1.3 검증 (`research/verify_wal_backup_20261120.py`, 전원 PASS)

| 케이스 | 검증 내용 | 결과 |
|---|---|---|
| `wal_only_commits` | WAL 에만 있는 커밋 보존 | 복제본 200행 · naive 파일복사는 오류 |
| `writer_during_backup` | writer 가 0.23초 동안 계속 INSERT 하는 중 복제 | `quick_check=ok` · FK 위반 0 · 중복 0 · 행수/워터마크/해시 1:1 일치 |
| `defect_hold` | 복제본 결함 주입(quick_check 손상) | `status=hold`, **이전 복제본 sha 불변**, 이전 manifest 유지, hold 파일 기록, 독자는 이전 복제본 사용 |
| `resolve_reader` | 신선/오래됨/크기 불일치 | 신선→복제본, 10,000초 경과→원본, 크기 불일치→원본 |
| `incremental_sync` | 증분 반영 + 손상 자가 복구 | 신규 30행 반영(총 130행), 종목 1개 손상 → 다음 회차가 그 종목만 전량 재복사해 복구 |
| `live_upbit` | 운영 DB(1.55M행) 실게시 | 7.49초, `data_hash_equal`·`bytes_equal_pages`·`schema_sig_equal` 모두 true, 독자 경로 `upbit_1m.replica.db` |

manifest 예 (운영, 22:18 KST):

```
status=ok  watermark.rows=1,552,498  max_ts=2026-11-20T12:42:00(UTC)
checks: duplicate_rows=0, data_hash_equal=true, bytes_equal_pages=true, schema_sig_equal=true
replica: journal_mode=delete, file_sha256=…(242,331,648B, page_count×page_size 일치)
```

### 1.4 읽기 경로 전환 (채점기·대시보드는 완성된 복제본만)

| 소비자 | 변경 | 실측 |
|---|---|---|
| `📊-업비트-전진검증-채점/check.sh` | `--db` 를 `wal_replica.py --resolve --max-age 1800` 결과로 치환, 실패·노후 시 원본 폴백 | heartbeat `db=replica` (21:55·22:00 KST) |
| `export_dashboard_json.py` | `_bars_count`/`db_stats` 가 `resolve_reader` 사용, `audit.read_source` 에 replica/source 표기 | status.json `db_source` 정상 |
| `safe_ro.open_ro` | ⓪ 게시된 복제본 → ① `mode=ro` → ② 백오프 재시도 → ③ **online backup 스냅숏**(바이트 복사 제거) | `verify_up_wal_safe` 3/3 PASS (`ro`/`snapshot_backup`/`replica_ro`) |

`resolve_reader` 보호 조건: manifest `status=ok` · 파일 존재 · manifest 크기 == 실제 크기 · 신선도(기본 900초, 채점기 1800초) · 워터마크 행수>0 · **hot journal(`-journal`) 없음**. 하나라도 어긋나면 원본 DB 로 폴백하므로 오늘 동작보다 나빠지지 않는다.

### 1.5 60초 상한 준수 (수집기 회차)

- 위치: `📈-업비트-모의투자-분석/check.sh` 수집 파이썬 블록 직후(텔레그램 단계 앞).
- 가드: `elapsed < 51초` 일 때만 실행 + **`timeout 3`** 하드 상한 + 내부 `--budget-s 3`. 최악 55초로 상한(60초) 아래를 지킨다(실측 회차 49·54초).
- 회차 로그(실측): `2026-11-20 22:08:42 KST status=ok copied=302 inserted=382 sec=2.7 defects=[]` / 회차 전체 `quiet 49s`. 22:04 회차는 `copied=60 sec=2.21`(당시 겹침 120분 → 예산 소진) → **겹침 15분으로 조정 후 302종목 전량 1회차 완료**.
- 회전 커서: `--budget-s` 로 끝까지 못 가면 다음 회차가 이어서 복사한다(`copy_cursor` 가 manifest 에 남는다).
- 실패·중단 시: 이전 복제본 유지, manifest 가 노후해지면 독자가 원본으로 폴백. **종료코드는 바꾸지 않는다.**
- 정직한 한계: 수집기 회차가 46~52초로 길어 `elapsed<53` 가드에 걸리는 회차가 있다(22:04 52초·22:14 54초 통과). 걸린 회차는 `replica_skip:budget(elapsed=Ns)` 로 남고 다음 회차가 이어받는다. 초기 가드를 43초로 잡아 21:41~21:59 회차가 전부 skip 된 이력도 로그에 그대로 남아 있다(53초로 수정).

## 2. P0-C — 원장 잠금(`SQLITE_BUSY`) 계측 강화

### 2.1 무엇을 남기는가

`rotation_errors` 에 열 5개를 **추가만** 했고(기존 행 보존), 이벤트 저널 `rotation_busy` 를 신설했다.

| 항목 | 저장 위치 | 실측 예 |
|---|---|---|
| 로그 원문·sqlite_errorcode | `rotation_errors.exc_message` / `sqlite_errorcode` | `OperationalError: database is locked` / `5` |
| 대기 시계열·평균·최대 | `busy_wait_ms_total`·`busy_wait_ms_max`·`busy_detail_json.waits_ms` | `[400, 800, 1200]` / 총 2,400ms / 최대 1,200ms |
| 재시도 횟수·상한 | `busy_attempts`·`busy_retries`·`detail.retry_cap` | 3회 시도 / 2회 재시도 / 상한 3, 상한 8초 |
| WAL 크기(장기 트랜잭션 단서) | `detail.wal` / `rotation_busy.wal_json` | db 86,036,480B · -wal 53,592B · -shm 32,768B |
| checkpoint 상태 | `detail.checkpoint` / `checkpoint_json` | `{busy:0, log_pages:13, checkpointed:13}` |
| 잠금 보유자 단서 | 파일락: lock 파일(pid·market·at·trigger) + `/proc` 파이썬 프로세스 목록 | busy_skip 출력 `lock_holder {pid:7, market:UP, trigger:test_holder}` |

구현 위치: `generation_rotation.py`(`exec_retry`·`_busy_detail`·`record_busy`·`_record_error`·`busy_metrics`), `auto_researcher.py`(`commit_retry`·`record_lock_event`·`lock_events` 테이블, ingest/검색 커밋 9곳 교체).

원장 자체가 잠겨 계측을 못 쓰는 경우(`record_busy` 실패)는 `rotation_busy_fallback.jsonl`, `lock_events_fallback.jsonl` 로 파일에 떨어뜨린다 — 실패가 조용히 사라지지 않는다.

### 2.2 `busy_skip` 과 `no_new_data` 의 통계적 분리

- `busy_skip` 은 **회차 표(`rotation_runs`)에 행을 만들지 않는다** — 파일락을 못 잡아 시작조차 못 했기 때문이다. 저널(`rotation_busy.kind='busy_skip'`)에만 남는다.
- `no_new_data` 는 정상 회차이며 `run_key` 가 있고 `n_new_rows=0` 이다.
- 실측 분리: `runs_by_status = {no_new_data: 11, error: 4, ingest_partial: 3}` (run_key null 0건) vs `busy_by_kind = {busy_skip: 1, busy_exhausted: 1}`.
- `busy_metrics` 는 `busy_skip` 시각 이후 **첫 회차**를 짝지어 `next_status`·`next_new_rows` 를 보여준다(누락 재수집 추적).

### 2.3 누락 데이터 재수집 보장 (테스트로 증명)

`verify_rotation_busy_metrics_20261120.py` ④:

```
체크포인트 before 1795175220 → 30분 되돌림 1795173420
패널 행 336개 삭제(누락 상황 재현)
다음 회차: status=no_new_data, new_rows=336, 체크포인트 after 1795175220(복귀), 패널 106,294행
```

즉 `busy_skip`·중단으로 밀린 구간은 다음 회차가 **체크포인트부터 matured 까지 다시 읽어** 재적재·평가한다.

검증 4케이스: `exec_retry_busy`(BUSY 계측) · `file_lock_busy_skip`(보유자 pid·체크포인트 불변) · `busy_skip_vs_no_new_data`(분리) · `busy_skip_recovery`(재수집) — **전원 PASS**, 기존 `verify_rotation_recovery`(겹침·SIGKILL·재개) 3/3 PASS 유지.

## 3. P0-E — `monitor/latest.json` `research={}` 원인 규명·수정

### 3.1 원인 (코드 경로 확정)

1. `📊-업비트-전진검증-채점/check.sh` 가 5분마다 `research/push_dashboard_api.py` 를 실행.
2. 그 안에서 `collab_ledger.push_latest(status)` 를 **verification 인자 없이** 호출.
3. `collab_ledger.build_latest()` 가 `research = verification or {}` → **빈 dict**.
4. 결과적으로 별도 생성기(`monitor_research_20261120.py`, 3시장 22+필드)는 파일로만 존재하고
   매 5분 푸시가 `research: {}` 로 덮어썼다.

실측 대조: 2026-11-20 **21:15:21 KST** 원격 `monitor/latest.json` GET → `research={}` (본 보고서 작성 직전 확인).
따라서 “상태 생성 성공”과 “게시 성공”은 있었지만 **콘텐츠 반영은 없었다** — GPT 지적이 정확하다.

### 3.2 수정

| 파일 | 수정 |
|---|---|
| `push_dashboard_api.py` | research 문서를 만들어 `push_latest(status, research)` 로 **인자 전달**. 캐시 TTL 30분(별도 단계가 15분 TTL 로 선갱신) |
| `collab_ledger.push_latest` | research 가 비면 ① 인자 → ② 캐시(`DASH_STATE_DIR/monitor_research_20261120.json`) → ③ 원격 기존값 순으로 채우고, 셋 다 없으면 **게시 보류**(빈 dict 로 덮어쓰지 않음). `research_source` 필드로 출처 표기 |
| `monitor_research_20261120.py` | 필드 별칭 추가(`updated_at`·`generation`·`status`·`last_error`·`shadow_signals_historical`), 캐시/`--refresh` CLI, UP ts_utc(UTC 문자열)→KST 오프셋 정규화, 수집기 `dashboard_stats.json` 우선 사용(전체 스캔 회피: 생성 13.7초 → 4.6초) |
| `📊` check.sh | research 갱신을 **별도 단계(②-1, timeout 14)** 로 분리 — 푸시(20초) 안에서 만들다 21:50·21:55 `dash=124` 타임아웃이 났다. 푸시는 캐시만 읽는다 |
| `push_dashboard_api.py`/`collab_ledger.py` | GitHub 409 Conflict(다른 회차가 먼저 커밋) 시 sha 재조회 후 3회 재시도 |
| `export_dashboard_json.py` | 봉 수 COUNT(*) 캐시 4분→30분, 전체 스캔을 3초에서 스스로 끊는 progress handler — 콜드 `build_status` 53초 → 12초(실측), 그래도 넘기면 last_good/stale 표기 |

### 3.3 게시 검증 (실제 raw URL GET)

`https://raw.githubusercontent.com/mju1417-max/market-dashboard/main/monitor/latest.json` (2026-11-20 22:00 KST 재조회):

```
generated_at = 2026-11-20T21:42:48+09:00   research_source = argument
research.markets = [kr, upbit, us]   upbit 필드 수 34개
upbit: updated_at 2026-11-20T21:42:18+09:00 · generation UP-G003 · status rotation:running
       candidate_count 0 · frozen_count 0 · shadow_signals_measured 373
       shadow_signals_historical null(미측정) · last_error null · data_cutoff 2026-11-20T21:39
kr:    generation lab_KR_20261120_3f1a339d · status daily_track_active · candidate_count 1,344,187
us:    generation lab_US_20261119_11d1253a · status lab_complete · candidate_count 3,627,819
```

- 3시장 22+필드(현재 34필드): `updated_at`·`generation`·`status`·`data_cutoff`·`last_complete_session`·`candle_count`·`universe`·`source_id`·`collector_errors`·`candidate_count`·`frozen_count`·`shadow_signals_measured`·`shadow_signals_historical`·`labelled`·`last_success`·`last_error`·`last_error_at`·`tp/fp/censored`·`calibration`·`uplift_vs_baseline`·`price_gain_error`·`latency`·`code_sha`·`forward_tracks`·`forward_closed`·`status_fields` 등.
- null 은 “0 으로 채우지 않은 미구현·미측정”이며 `status_fields` 에 사유가 있다.

## 4. 코드 해시·재현

| 파일 | sha256(앞 12) |
|---|---|
| `research/wal_replica.py` (신규) | `70dc9c5c734d` |
| `research/safe_ro.py` | `3172610b81b9` |
| `research/generation_rotation.py` | `0cb3aca048b6` |
| `research/auto_researcher.py` | `d521cf0bc742` |
| `research/collab_ledger.py` | `5b86ac390648` |
| `research/push_dashboard_api.py` | `9782ed0fbcf8` |
| `research/export_dashboard_json.py` | `21b8d72dbc01` |
| `research/monitor_research_20261120.py` | `29c0bf271005` |
| `research/verify_wal_backup_20261120.py` (신규) | `e1dc109550f5` |
| `research/verify_rotation_busy_metrics_20261120.py` (신규) | `e7d750066e79` |
| `scheduled-actions/📈-업비트-모의투자-분석/check.sh` | `cfd8369c3759` |
| `scheduled-actions/📊-업비트-전진검증-채점/check.sh` | `7f96a20a7d2b` |

```bash
# P0-B (6케이스, 전원 PASS)
python3 research/verify_wal_backup_20261120.py
# P0-B 기존 경로 회귀(3케이스)
python3 research/verify_up_wal_safe_20261120.py
# P0-C (4케이스)
python3 research/verify_rotation_busy_metrics_20261120.py
# P0-C 기존 회복·중복실행 회귀(3케이스)
python3 research/verify_rotation_recovery_20261120.py
# 운영 복제본 상태/읽기 경로
python3 research/wal_replica.py --status  "scheduled-actions/📈-업비트-모의투자-분석/state/upbit_1m.db"
python3 research/wal_replica.py --resolve "scheduled-actions/📈-업비트-모의투자-분석/state/upbit_1m.db"
# 잠금 계측 요약(busy_skip vs no_new_data, 대기·재시도·WAL/checkpoint)
python3 research/generation_rotation.py --db "scheduled-actions/🔬-자가발전-패턴-연구/state/knowledge.db" --busy-metrics
# research 문서 3시장 재생성/캐시
python3 research/monitor_research_20261120.py --refresh --max-age 900
```

## 5. 미구현·미측정 (정직 기록)

1. **`quick_check` 전량 검사는 매 회차 미실행** — 콜드 캐시 242MB 에서 106초(실측). 회차 게시는 light(지문·중복·페이지/바이트·스키마 해시), 전체 구조 검사는 `--publish --deep`(테스트·수동)로 분리했다. 운영 복제본의 deep 검사는 아직 정기 스케줄에 붙이지 않았다(회차 상한 60초와 충돌).
2. **복제본 소실 시 자동 재씨앗 없음** — `--sync` 는 `status=needs_seed` 로 알리고 원본으로 폴백한다. 재씨앗(전체 backup)은 AI 턴/수동 실행이 필요하다.
3. **수집기 회차 예산** — 평시 46~54초 + 복제 동기화 2.7~3.5초(부하 시 164~302종목/회차). 부하가 큰 회차는 일부 종목만 복사되어 **종목별 지연이 최대 1~2회차(5~10분)** 날 수 있다(회전 커서가 이어받는다). `elapsed<53` 가드를 넘긴 회차는 skip 되고 다음 회차가 이어받는다(로그로 확인 가능). 복제본 신선도가 30분을 넘으면 채점기는 원본 DB 로 폴백한다(오늘과 동일 동작).
4. **KR·US DB 는 복제본 미적용** — 두 DB 는 `journal_mode=delete` 라 읽기 전용 직접 읽기가 안정적이다. 요청 범위(WAL)에 해당하지 않아 그대로 두었다.
5. `shadow_signals_historical` 은 현재 null(추정치 미집계)이며 `shadow_signals_measured`(실측 373/306/14)만 채워져 있다 — 0 으로 채우지 않았다.
6. `upbit.research_status` 가 `rotation:running` 으로 보이는 구간이 있다(30분 회차가 실제로 실행 중일 때의 정직한 상태 표기). 표본 시점에 따라 `rotation:no_new_data` 로 바뀐다.

## 6. 필수 제출 항목 대응표

| 요구 | 이 회차 값 |
|---|---|
| work_id | 30/P0-B · 30/P0-C · 30/P0-E · 30/VI |
| 데이터 cutoff / last_complete_session | UP `data_cutoff 2026-11-20T21:39` / `last_complete_session 2026-11-20` · KR `15:44` · US `21:37` |
| 전후 데이터 행수 | 운영 복제본 1,552,498행(원본과 동일, `duplicate_rows=0`) · 검증용 소형 DB 200→130행 시나리오 |
| 재현 명령 | 4절 |
| 중단·복구 테스트 | `verify_rotation_recovery`(겹침/SIGKILL/재개) 3/3 · `verify_rotation_busy_metrics`(락/BUSY/누락 재수집) 4/4 |
| DEV/VAL/HOLD 경계 | 이 회차는 인프라 항목이라 해당 없음(트랙 1 보고서 담당) |
| TP/FP/censored 분자·분모 | 이 회차는 해당 없음(연구 모니터에는 미측정 null 로 표기) |
| 운영 스케줄 실제 가동 로그 | `📈` `state/replica_sync.log`, `📊` `state/check_heartbeat.log`(`db=replica`), `state/research_refresh.log` |
| 불일치 목록 | `monitor/latest.json` 의 `candles_1m_query`(SUM vs COUNT) 차이는 P0-7 정의 차이로 그대로 병기 |
| 미구현·미측정 | 5절 |
