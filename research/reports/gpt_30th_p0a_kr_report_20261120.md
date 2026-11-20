# GPT 제30차 통합 지시서 — P0-A (한국 `surge_miner --stage` · `strategy_lab_KR` 정본 동기화 · 소급 체결 인과성 분리)

- **작업 ID**: `30th-round / P0-A / 한국`
- **작성·실행 시각**: 2026-11-20 21:15 ~ 22:15 KST (모든 수치는 이 구간 실측)
- **데이터 기준**: 한국 1분봉 `~2026-11-20 15:44` · 일봉 `2026-11-20` · 랩 8세션(`20768,20769,20770,20773,20774,20775,20776,20777`)
- **트랙 경계**: 트랙 2 소관(`safe_ro.py`, SQLite backup, `generation_rotation.py`, `export_dashboard_json.py`, `push_dashboard_api.py`, `monitor/latest.json`)와 트랙 B(`market_common.py`, `market_adapters.py`)는 **건드리지 않았다**.
- **공개 저장소**: 푸시하지 않았다(지시대로 보고서 파일만 남긴다). 비밀키·PAT·개인정보·원본 DB는 문서에 넣지 않았다.

---

## 0. 요약 (결론 먼저)

| 항목 | 결론 | 실측 근거 |
|---|---|---|
| **작업 1** `surge_miner --stage` | **완성·가동.** `PENDING→PREPARED→FEATURES_DONE→SEARCH_DONE→EVALUATED→VERIFIED→PUBLISHED` 상태머신, 40초 회차로 분할·재개, 체크포인트 17키 + 단계별 산출물 해시 기록. **일괄 실행과 결과가 한 자리도 다르지 않음**(실데이터 6,452개 값 중 불일치 **0**) | §1 |
| 한국 폭등공식 일일 자동 갱신 | **29차의 미완결이 해소됨.** 오늘(11-20) 실제로 6회차(40초 창)에 걸쳐 완주해 `state/evolved_rules_kr.json` **generation 28** 로 게시. 25회 상한·`partial` 진행 기록·검증 실패 시 게시 차단 | §1-6 |
| **작업 2** 랩 정본 동기화 | **완료.** 구 정본(SHA-256 `f6320781…`, 11-18 18:51, 구 규약) `research/archive/` 보존 → 8/8 세션·후보 **1,344,187** 원장 검증 → 원자적 교체 → 재독 해시·8/8 일치. `protocol_version=lab_protocol_v2_session_cooldown` 각인 | §2 |
| **작업 3** 소급 체결 인과성 | **분리 완료.** S16 체결 9건 중 **6건**(11-18 신호→11-19 종가 3, 11-19 신호→11-20 종가 3)은 체결 시각보다 **35~53시간 늦게** 신호를 처음 인지 → `recovery_backfill` 로 이동(삭제 없음). VCP 지어소프트는 체결 14시간 **전**에 인지 → `forward` 유지. S16 수급에 `first_available_at` 기록·차단 구현 | §3 |
| **검증** | `verify_kr_surge_miner_stage.py` **17/17 PASS**(합성 120종목 + 실데이터 파리티) · `verify_kr_recovery_20261120.py` **14/14 PASS** · `forward_tracker_kr_s16/vcp --self-test` **PASS** · 두 KR 예약 액션 `check` **quiet(exit 1)** | §4 |

---

## 1. 작업 1 — 한국 `surge_miner --stage` 완전 자동화

### 1-1. 왜 필요했나 (29차의 정확한 실패 지점)

2세대 엔진은 한국 실측 **125초**(레짐 18.4s + 피처 80s + 탐색·평가 26s)이고 체크포인트가 없었다. 60초 예약 창에서는 원리적으로 완주할 수 없어, 29차는 "45초에서 잘림 → `not_finished`" 를 정직하게 남기는 것이 최선이었다. 즉 **폭등공식의 일일 자동 갱신이 비어 있었다**.

### 1-2. 상태머신과 단계

`research/surge_stage.py`(신규) + `surge_miner.py --stage`:

```
PENDING ─ plan ─→ PREPARED ─ regime ─ features(배치 N) ─ merge ─ prep ─→ FEATURES_DONE
   ─ search:1 / score / search:2:2 / search:2:5 ─→ SEARCH_DONE
   ─ stage1 / stage2 / diag / evmodel / prevgen / assemble ─→ EVALUATED
   ─ verify ─→ VERIFIED ─ publish ─→ PUBLISHED
중단·예산소진 → INCOMPLETE(부분 기록 유지) · 예외 → ERROR(마지막 정상 산출물 보존)
```

CLI(지시서 그대로):

```sh
python3 research/surge_miner.py --market KR --stage next --max-seconds 40 --experiment-id <id>
python3 research/surge_miner.py --market KR --stage verify   --workspace <dir>
python3 research/surge_miner.py --market KR --stage finalize --workspace <dir>
```

### 1-3. 40초를 넘는 단계를 어떻게 나눴나 (실측 기준)

| 단위 | 일괄 실행 실측 | 분할 방식 | 실효 소요(부하 시) |
|---|---|---|---|
| `plan` | — | 유니버스·분할일·서명 계산 | 6.7~11.9s |
| `regime` | 18.4s | **종목 배치**(평면 npz) + **360분 블록**(`REG_CHUNK` 경계 고정) | 3회차 13~35s |
| `features` | **80s** | **종목 배치** — 종목 경계에서만 끊고(중간 절단 없음) 마감 3초 전 정지, 커서 저장 | 4회차 20~34s |
| `search:*` | 1단 12s | **규칙 격자 배치** — pool 순서·(beam,pool) 순서·정렬 안정성을 보존하며 40개 단위로 끊고 재개 | 1회차 12.3s |
| `merge`·`prep`·평가·조립 | 1~6s | 단일 단위(예산 초과 시 커서 재개) | 각 0.3~7.4s |
| `verify`·`publish` | 5~11s | 단일 | 10.6s / 0.1s |

- 배치 크기는 **측정 속도 × 22~25초**로 스스로 조정한다(부하가 걸려 느려지면 자동으로 작아진다). 한 회차가 예산에 닿으면 `partial` 로 커서만 남기고 끝난다.
- 상한 사고 방지: 배치 memmap 상한을 `max(종목별 봉수) × 배치종목수`로 잡아 "유지 행 ≤ 후보 분 ≤ 봉" 상한을 보장하고, 배치 종목수를 900개로 제한했다(파일당 512MB `ulimit` 안).

### 1-4. 체크포인트 키 (전부 기록)

`state/surge_stage/checkpoint.json`:

```json
{
 "market":"kr", "experiment_id":"kr_20261120_226ddf89", "data_cutoff":"2026-11-20T15:44+09:00",
 "last_complete_session":"2026-11-20", "universe_hash":"d4f4d20d…18ce0", "source_db_id":"2971e4d759e372ce",
 "feature_version":"5e737ff25e393165", "label_version":"51ae8058c20bb07e", "cost_version":"be5ae7b7cbcb49d9",
 "session_calendar_version":"3536121d243ae654", "hypothesis_family_hash":"…", "code_sha":"…",
 "chunk_id":"publish", "completed_count":3156, "total_count":3156, "file_hash":"…",
 "started_at":"2026-11-20T22:08:2x+09:00", "completed_at":"2026-11-20T22:11:45+09:00",
 "stage":"PUBLISHED", "status":"COMPLETE", "units":{…}, "db_read":{"mode":"ro","query_only":true}, "history":[…]
}
```

- **입력 서명이 바뀌면 이전 청크를 재사용하지 않는다.** 매 회차 `cheap_sig`(DB 크기·mtime·최대 봉)를 비교하고, 바뀌었으면 이전 산출물을 `branches/<실험>_<시각>/` 로 옮긴 뒤 **새 실험 ID** 로 분기한다(PENDING 부터). `data_cutoff`·`universe_hash`·`feature_version`·`hypothesis_family_hash`·`code_sha` 가 실험 ID 재료다.
- 산출물은 전부 `임시파일 → fsync → os.replace` 로만 쓴다. 체크포인트는 `checkpoint.json` + `checkpoint.bak.json` 이중 보관하고, 손상 시 `.bak` 으로 복구한다.
- 중복 실행은 `flock` 비차단 — 이미 도는 회차가 있으면 즉시 `busy_skip`(exit 5)으로 양보한다.

### 1-5. 검증 5종 (`research/verify_kr_surge_miner_stage.py`, **17/17 PASS**)

합성 한국장 DB(결정적 seed, 120종목×3일)로 같은 코드 경로를 검증하고, 실데이터 파리티는 따로 판정한다.

| 묶음 | 확인 | 결과 |
|---|---|---|
| (a) 일괄 vs 분할 파리티 | 동일 입력에서 일괄 1회 vs **3배치** 분할 — 페이로드 리프 3,917개 **불일치 0**, 후보 수 129,960 동일, TP/FP/검열 `{n:364, tp:1, fp:178, censored:185}` 동일, 1단 규칙 4개 **순서까지** 일치 | PASS |
| (a') 실데이터 파리티 | 실제 한국 DB 일괄 결과 vs 분할 게시 결과 **6,452개 값 중 불일치 0** (후보 2,428,914 · Test 775,000행 · 규칙 148개 · 선택 규칙 `2단5%#4 / tp5_sl3_60` · Test n=28 ev=−1.6147) | PASS |
| (a'') 비교에서 뺀 것 | 실행 **소요시간**(`v2/regime/elapsed`)과 실행 **형태**(`provenance.*`, 청크 수·실험 ID)만. 결과 수치·순서·통계는 전부 비교 | — |
| (b) 20%·50%·80% 강제 중단 후 재개 | 각 지점에서 **SIGKILL** → 재개 → 일괄과 **불일치 0**(FEATURES_DONE/SEARCH_DONE/EVALUATED 에서 중단) | PASS ×3 |
| (c) 중복 실행·SIGKILL·손상 방어 | ① 동시 실행 → `busy_skip`(5) ② `checkpoint.json` 을 깨뜨림 → `.bak` 복구 후 계속 ③ 배치 파일을 깨뜨림 → 해시 불일치 감지·재계산(오류로 죽지 않음) | PASS ×3 |
| (d) 불완전 결과는 이전 정본 보존 | 부분 진행 회차에서 `SURGE_OUT` 해시 불변·`PUBLISHED` 아님 | PASS |
| (e) 수집 DB 경합 없음 | 다른 프로세스가 `BEGIN IMMEDIATE` 쓰기 트랜잭션을 쥔 채로 분할 실행 → 성공, `db_read.mode="ro"`, DB mtime 불변, `-wal`/`-shm` 새 파일 **0** | PASS |

검증 산출물: `research/verify_kr_surge_miner_stage_20261120.json` (17/17, 항목별 상세).

### 1-6. `kr_daily_step.py`·`check.sh` 연결과 **오늘 실제 완주**

- `kr_daily_step.py` 의 `surge` 단계가 `surge_miner.py --market KR --stage next --max-seconds 32 --workspace <state>/surge_stage` 를 부르도록 교체. 로그 마지막 JSON 줄에서 `stage` 를 읽어
  - `PUBLISHED` → `ok`(그날 완료)
  - 진행 중 → `partial`(다음 회차가 이어받는다 — 시도 횟수로 벌하지 않음)
  - 25회를 넘기면 → `not_finished`(하루 상한)
  로 기록한다. `check.sh`·창(16:05~19:30)·워터마크 가드는 그대로다.
- **오늘 22:08~22:11 실측 완주**(창 밖이지만 수동 catch-up): 6회차에 걸쳐
  `plan(11.9s) → regime(13.3s) → features(4배치) → merge(6.1s) → prep → search:1(12.3s) → score → search:2:2/2:5 → stage1/stage2/diag/evmodel/prevgen/assemble → verify(10.6s, 8/8 PASS) → publish`.
  결과 `state/evolved_rules_kr.json` **generation 28**(직전 27), `data_cutoff 2026-11-20T15:44+09:00`, 선택 `2단5%#4 / tp5_sl3_60` (Test n=28 승25.0% ev −1.6147 — **일괄 실행과 동일 수치**), `provenance` 에 실험 ID·서명·청크 수 기록.
- `scheduled-action check scheduled-actions/🇰🇷-한국장-일간-재탐색` → **quiet(exit 1)**.

### 1-7. 부하 상황에서의 정직한 기록

같은 날 21:30~22:10 사이 다른 5분 파이프라인이 겹치며(load average 1.5/2코어) 레짐·피처가 일괄 실행의 2~3배 느려졌다. 그래서 "40초 창 1회 완주" 가 아니라 **여러 회차 누적 완주**로 설계했고, 그 조건에서 실제로 6회차에 완주했다. 부하가 없을 때의 예상은 6~7회차(약 4분)와 같다. 남은 창(16:05~19:30, 5분 간격 41회차)에서 lab(6~8회차)·report(1회차)·surge(6~8회차)를 모두 소화할 수 있다.

---

## 2. 작업 2 — `research/strategy_lab_KR.json` 정본 동기화

도구: `research/sync_strategy_lab_canonical.py` (+ 규약 정의 `research/lab_protocol.py`). 원장: `research/reports/strategy_lab_KR_sync_ledger_20261120.json`.

### 2-1. 구 정본 보존 (versioned archive)

| 항목 | 값 |
|---|---|
| 구 정본 | `research/archive/strategy_lab_KR.f6320781_20261118_185123.json` |
| SHA-256 | `f6320781ab59da07…` (복사 후 재해시 일치 확인) |
| 수정시각·크기 | 2026-11-18 18:51 · 2,161,464 B |
| 구 규약 | `provenance` **없음**, `protocol_version` 없음(전역 쿨다운 시절), 세션 6 · 후보 1,038,415 |
| 원장 | `research/archive/strategy_lab_KR.f6320781_20261118_185123.archive.json` 에 "구판 OOS 와 신판 OOS 를 직접 비교하지 않는다" 는 사유까지 기록 |

### 2-2. 원장(ledger) 검증 — 12개 항목 전부 통과

| 검사 | 결과 |
|---|---|
| state `complete=true`·provenance `complete=true` | PASS |
| experiment_id `lab_KR_20261120_3f1a339d` · data_cutoff `2026-11-20` | PASS |
| 세션 청크 `state/lab_chunks/KR/<eid>/` **manifest 8개 = 세션 8개** | PASS |
| 8개 manifest 의 `experiment_id`·`data_cutoff`·`feature_version`·`session_basis`(cost 0.41 포함) **전부 동일** | PASS |
| manifest 날짜 = `meta.days` (`20768·20769·20770·20773·20774·20775·20776·20777`) | PASS |
| 후보 합계 = manifest 합 = 파일 값 = **1,344,187** | PASS |
| `provenance.completeness="8/8"` · `cooldown_scope="session"` | PASS |
| `protocol_version` = `lab_protocol_v2_session_cooldown` (세션 경계 30분 쿨다운 리셋) | PASS |

### 2-3. 원자적 교체와 재독

- 같은 파일시스템 임시파일(`strategy_lab_KR.json.tmp<pid>`) → `fsync` → `os.replace`.
- 재독 검증: SHA-256 `4de2d508…`(쓴 값과 동일) · 세션 **8/8** · `complete=true` · 후보 **1,344,187** · 규약 `lab_protocol_v2_session_cooldown` **전부 일치**. 하나라도 어긋나면 `strategy_lab_KR.json.before_sync` 로 **복구**하도록 만들었다(이번에는 불필요).
- **구판 OOS vs 신판 OOS 단순 성능 비교는 하지 않았다.** 원장에 `protocol_change(from=v1_global_cooldown, to=v2_session_cooldown)` 와 "비교 금지" 정책을 명시했다. 이후 랩 실행도 `provenance.protocol_version` 을 남기도록 `strategy_lab.py` 에 연결했다(다음 실행부터).

---

## 3. 작업 3 — `KR_S16`·`KR_VCP` 소급 체결 인과성 분리

도구: `research/kr_forward_causality_audit.py` → `research/reports/kr_forward_causality_20261120.json`(감사 원장). 장부 백업: `state/archive/forward_trades_kr_s16_pre_causality_20261120_*.json`.

### 3-1. 신호별 대조표 (실측)

| 트랙 | 종목 | 신호일 | 신호봉 | 데이터 공표 | 첫 인지 | 체결 규칙 | 판정 |
|---|---|---|---|---|---|---|---|
| KR_S16 | 빙그레(005180) | 2026-11-17 | 2026-11-17 15:30 KST | 2026-11-18T09:00:00+09:00 | 2026-11-18T19:37:18+09:00 | 2026-11-18T15:30:00+09:00 | **origin_backdated** |
| KR_S16 | 바디텍메드(206640) | 2026-11-17 | 2026-11-17 15:30 KST | 2026-11-18T09:00:00+09:00 | 2026-11-18T19:37:18+09:00 | 2026-11-18T15:30:00+09:00 | **origin_backdated** |
| KR_S16 | 가비아(079940) | 2026-11-17 | 2026-11-17 15:30 KST | 2026-11-18T09:00:00+09:00 | 2026-11-18T19:37:18+09:00 | 2026-11-18T15:30:00+09:00 | **origin_backdated** |
| KR_S16 | 케이뱅크(279570) | 2026-11-18 | 2026-11-18 15:30 KST | 2026-11-19T09:00:00+09:00 | 2026-11-20T20:14:12+09:00 | 2026-11-19T15:30:00+09:00 | **recovery_backfill** |
| KR_S16 | 경동나비엔(009450) | 2026-11-18 | 2026-11-18 15:30 KST | 2026-11-19T09:00:00+09:00 | 2026-11-20T20:14:12+09:00 | 2026-11-19T15:30:00+09:00 | **recovery_backfill** |
| KR_S16 | NICE(034310) | 2026-11-18 | 2026-11-18 15:30 KST | 2026-11-19T09:00:00+09:00 | 2026-11-20T20:14:12+09:00 | 2026-11-19T15:30:00+09:00 | **recovery_backfill** |
| KR_S16 | 오리온홀딩스(001800) | 2026-11-19 | 2026-11-19 15:30 KST | 2026-11-20T09:00:00+09:00 | 2026-11-20T20:14:16+09:00 | 2026-11-20T15:30:00+09:00 | **recovery_backfill** |
| KR_S16 | 종근당홀딩스(001630) | 2026-11-19 | 2026-11-19 15:30 KST | 2026-11-20T09:00:00+09:00 | 2026-11-20T20:14:16+09:00 | 2026-11-20T15:30:00+09:00 | **recovery_backfill** |
| KR_S16 | 롯데칠성(005300) | 2026-11-19 | 2026-11-19 15:30 KST | 2026-11-20T09:00:00+09:00 | 2026-11-20T20:14:16+09:00 | 2026-11-20T15:30:00+09:00 | **recovery_backfill** |
| KR_VCP | 지어소프트(051160.KQ) | 2026-11-18 | 2026-11-18 15:30 KST | 2026-11-18T15:46:00+09:00 | 2026-11-18T19:44:50+09:00 | 2026-11-19T09:00:00+09:00 | **forward** |

읽는 법:

- **S16 6건** — 신호 기록(`created_at`)이 `2026-11-20 20:14` 인데 체결 규칙 시각은 11-19 15:30 / 11-20 15:30. 즉 시스템이 신호를 **35~53시간 늦게** 알았다. 29차 P0-1 복구가 11-19·11-20 일봉을 소급 적재한 뒤 신호일 11-18·11-19를 재평가하면서 만들어진 **사후 재구성 진입**이다.
- **VCP 1건(지어소프트)** — 신호를 11-18 19:44 에 이미 기록했고 체결 규칙 시각(11-19 09:00 시가)보다 13.3시간 **앞선다**. 실제로 인지한 뒤 규칙대로 체결된 **정상 전진 진입**으로 판정(29차의 "미체결 신호 후속 체결" 은 가격 확정이 늦었을 뿐 인지 시점은 11-18 이다).
- **기존 3보유(빙그레·바디텍메드·가비아)** — 트랙 시작(11-18 19:37) 때 만든 초기 보유라 체결가(11-18 15:30 종가)가 시작 **4시간 전**이다. 지시대로 기존 보유는 성과 합산에 남기되 `bucket="forward_origin_backdated"` 로 **표시**했다(값·진입일은 그대로, 숨기지 않는다).

### 3-2. 장부 분리 결과

| 항목 | 분리 전 | 분리 후 |
|---|---|---|
| S16 보유(open) | 9 | **3** (빙그레·바디텍메드·가비아, `forward_origin_backdated`) |
| S16 `recovery_backfill` | — | **6** (케이뱅크·경동나비엔·NICE·오리온홀딩스·종근당홀딩스·롯데칠성, `excluded_from_forward=true`, `causality` 전문 포함) |
| S16 통계 entries / 총수익 / equity | 9건 / −1.168% / 0.9883 | **3건 / −0.257% / 0.9974** (자본곡선 재계산) |
| S16 대기(pending) | 58 | 58 (11-20 신호, 11-23 종가 진입 예정 — 정상 전진) |
| VCP 보유 | 1 | 1 (변경 없음, `forward`) |

체결 레코드·신호 로그는 **삭제하지 않았다**. 옮긴 체결에는 `bucket`, `excluded_from_forward`, `replay_reason="fill_before_first_decision"`, `causality`(위 표의 6필드)를 붙였고, 장부에 `causality_audit` 원장 한 줄(이동 목록·규칙)을 남겼다.

### 3-3. `KR_S16` 수급 특징 `first_available_at`

- `kr_s16_data.db` `supply` 테이블에 `first_available_at` 열 추가(기존 DB는 `ALTER TABLE` 자동).
- 수집(`fetch_supply`)은 **처음 확인한 시각**을 박는다(`ON CONFLICT … COALESCE(supply.first_available_at, excluded.first_available_at)` — 최초 시각 유지).
- 기존 32,241행(faa NULL)은 **다음 거래일 09:00** 으로 보수 해석한다. 근거는 실측이다: as_of 11-20 에 11-20 수급 행이 **131종목**만, 11-19 행 315종목, 11-17 행 1,265종목 — 그날 값이 그날 확정 전에 다 차 있지 않다.
- 신호 생성 시 `first_available_at > decision_at` 인 행은 **쓰지 않고**, 최근 5거래일 창을 그만큼 뒤로 밀어 5일을 채운다(규칙의 '5일 순매수 강도' 정의 유지). 채울 수 없으면 `no_supply` 로 제외(deferred 사유 기록). 각 신호에는 `supply_as_of`, `supply_first_available_at`, `decision_at`, `signal_bar_at` 이 함께 저장된다.
- 샌드박스 실측(11-19 재평가): `가격조건 198 → f5>0 89`, 신호마다 `supply_as_of=2026-11-19`, `supply_first_available_at=2026-11-20T09:00+09:00` — **공표 전 값 없이** 같은 89건이 재현됐다.
- `--self-test` 에 차단 규칙 단위검사 4개를 추가했고 전부 통과(`supply_row_usable` : 01-05 저녁에 01-05 수급 불가 / 익일 09:30 가능 / 명시 시각 이전 불가).

---

## 4. 검증 목록

| 스크립트 | 항목 | 결과 |
|---|---|---|
| `research/verify_kr_surge_miner_stage.py` | (a) 파리티 합성·실데이터 · (b) 20/50/80% 중단재개 · (c) 중복·SIGKILL·손상 · (d) 정본 보존 · (e) DB 경합 | **17/17 PASS** (→ `research/verify_kr_surge_miner_stage_20261120.json`) |
| `research/verify_kr_recovery_20261120.py` | 캘린더 5 · S16 게이트/결측/멱등 · VCP 커버리지·후속체결·중복 | **14/14 PASS** |
| `surge_stage` 자체 검증(`--stage verify`) | 청크 전수·행수·서명 불변·기준선/규칙 재계산·DB 읽기전용·조립 존재 | **8/8 PASS** (운영 게시 전 실행) |
| `forward_tracker_kr_s16.py --self-test` | 규칙·체결·정산·공표 차단 단위검사 | **PASS** |
| `forward_tracker_kr_vcp.py --self-test` | 규칙·트레일링·미청산 유지 | **PASS** |
| `sync_strategy_lab_canonical.py` | 원장 12항목 + 재독 | **12/12 PASS** |
| `scheduled-action check` | 🇰🇷한국주식-패턴-분석 · 🇰🇷한국장-일간-재탐색 | **quiet(exit 1)** |

---

## 5. 남은 것 (숨기지 않고 기록)

1. **기존 3보유의 원점 백데이팅** — 트랙이 11-18 19:37 에 시작해 11-18 15:30 종가를 진입가로 쓴 것은 사실이다(표의 `origin_backdated`). 지시대로 성과에는 남겼지만, 엄밀히는 "시작 전 가격" 이므로 앞으로 전진 표본을 해석할 때 이 3건을 **별도 표기**로 읽어야 한다.
2. **수급 `first_available_at` 의 과거 행** — 32,241행이 NULL(보수적으로 익일 09:00). 앞으로 수집되는 행은 실제 확인 시각이 박히므로, 며칠 지나면 실측 기반으로 바뀐다. 과거 행의 실제 공표 시각은 복원할 수 없다(당시 기록 없음).
3. **`surge_miner` 의 레짐 단위** — 현재 13~35초로 40초 안에 들어오지만, 종목이 더 늘면 `REG_CHUNK` 블록 커서로만 쪼개진다(종목 채우기는 이미 배치). 시장이 2배가 되면 `regime` 도 별도 예산 배분이 필요하다.
4. **부하 의존 소요** — 완주에 6~8회차(30~50분) 걸린다. 창(16:05~19:30)에는 충분하지만, `lab` 이 8회차를 다 쓰는 날은 `surge` 가 창 끝에 걸릴 수 있다. 그때는 다음 날 첫 회차가 `inputs_changed` 로 분기해 새 실험을 시작한다(이전 산출물 보존, 게시는 되지 않음).
5. **TP/FP/검열·calibration 실현 라벨** — 여전히 3시장 모두 `null + not_measured`(29차 기록 그대로). 이번 회차 범위 밖.
6. **공개 저장소** — 지시대로 푸시하지 않았다. 게시가 필요하면 이 보고서와 `research/surge_stage.py`·`verify_kr_surge_miner_stage.py`·`sync_strategy_lab_canonical.py`·`lab_protocol.py`·`kr_forward_causality_audit.py` 만 커밋하면 된다(비밀키·DB·절대경로 없음).

---

## 6. 변경·신규 파일

- **신규**: `research/surge_stage.py` · `research/verify_kr_surge_miner_stage.py` · `research/lab_protocol.py` · `research/sync_strategy_lab_canonical.py` · `research/kr_forward_causality_audit.py`
- **수정**: `research/surge_miner.py`(`--stage` 분기, `process_market` 종목 배치/마감, `market_db` env) · `research/kr_daily_step.py`(surge 단계 → `--stage`) · `scheduled-actions/🇰🇷-한국장-일간-재탐색/action.md`(단계 실행기 설명) · `research/strategy_lab.py`(`protocol_version` 기록) · `research/forward_tracker_kr_s16.py`(수급 `first_available_at`·차단·창 이동·self-test)
- **교체**: `research/strategy_lab_KR.json`(8/8 · 1,344,187 · v2 규약) — 구 정본은 `research/archive/` 에 보존
- **장부 변경**: `scheduled-actions/🇰🇷-한국주식-패턴-분석/state/forward_trades_kr_s16.json`(open 9→3, `recovery_backfill` 6) · `state/evolved_rules_kr.json`(generation 28)
- **무변경(트랙 2·B)**: `research/safe_ro.py` · `generation_rotation.py` · `export_dashboard_json.py` · `push_dashboard_api.py` · `monitor/latest.json` · `research/market_common.py` · `research/market_adapters.py`
