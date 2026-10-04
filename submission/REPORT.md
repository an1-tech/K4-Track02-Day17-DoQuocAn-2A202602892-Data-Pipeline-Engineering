# K4 Track 02 Day 17 — Report cá nhân

**Họ tên / MSSV:** Đỗ Quốc An / 2A202602892.

**Repo bài nộp do học viên cung cấp:** https://github.com/an1-tech/K4-Track02-Day17-DoQuocAn-2A202602892-Data-Pipeline-Engineering

**Tên repo bài nộp cần dùng:** `K4-Track02-Day17-DoQuocAn-2A202602892-DataPipelineEngineering`.
Học viên đã đổi tên repo với hậu tố `Data-Pipeline-Engineering`; hậu tố này khác
chuỗi `DataPipelineEngineering` trong SUBMISSION.md. Dùng đúng URL học viên cung cấp.

**Commit mã nguồn và bằng chứng đã kiểm chứng:** `0e0ba84`.
REPORT được bổ sung ở commit tiếp theo; xem commit mới nhất trên nhánh `main` khi nộp.

**AI đã dùng:** ChatGPT/Codex hỗ trợ đọc đề, phân tích lỗi, viết code, thực thi
kiểm tra và soạn báo cáo/thiết kế. Học viên cần review diff và giải thích các thay đổi.

**Nguồn tham khảo:** tài liệu đề trong `docs/`; [DuckDB MERGE](https://duckdb.org/docs/stable/sql/statements/merge_into);
[dbt lookback](https://docs.getdbt.com/reference/resource-configs/lookback).

**Ngày thực hiện:** 04/10/2026, UTC+7. Đây là ngày thực hành, không khẳng định deadline của lớp.

## 1. Ba lỗi

Baseline đạt 8/18 checks; 9 tests fail. Bằng chứng gốc ở `submission/evidence/baseline-*.txt`.

| | Silver | Late data | CDC delete |
|---|---|---|---|
| Triệu chứng | 24 hàng/12 ticket; T-91 có 3 trạng thái | u05 ngày 12/08 có (2 events, 0 down); feature lệch full recompute | T-97 còn ở snapshot mới nhất và có 2 RAG chunks |
| Nguyên nhân | Dedup trong batch rồi INSERT nối tiếp | LOOKBACK_DAYS=0, không tính lại ngày event cũ | Chỉ lấy khóa từ after; delete after=null bị loại |
| Cách sửa | `silver.py`: MERGE theo ticket_id, update khi LSN mới hơn | `config.py`: LOOKBACK_DAYS=3 từ số đo Bronze | `staging.py`: coalesce khóa after/before; các cột PII vẫn lấy after |
| Khái niệm | Keyed upsert; replay idempotent; LSN guard | Event time; lateness; overwrite partition | Debezium delete; Silver tombstone; xóa lan xuống Gold |

## 2. Các con số

- Bronze: 43 event records; P50=0.00, P95=2.90, P99=3.00 ngày; lookback=ceil(P99)=3.
- Verify 18/18; pytest 34 passed; dbt PASS=19; parity PARITY.
- Rerun PASS: C0=C1=C2=C3=`39e115c510ecdf526800eac227158a4f`.

## 3. Lựa chọn công cụ / kỹ thuật

- MERGE giữ một trạng thái/ticket; overwrite partition tính lại aggregate, tránh cộng trùng khi replay.
- Tombstone giữ khóa/LSN, chặn batch cũ hồi sinh ticket; đánh đổi là phải quản lý hàng đánh dấu xóa.
- Snapshot as-of Bronze và feedback đã ingest giữ point-in-time; version cũ không nhận thông tin tương lai.
- DuckDB phù hợp dữ liệu local nhỏ; dbt quản lý SQL/dependency/contracts. Chưa cần Spark phân tán.

## 4. Hai câu hỏi suy ngẫm

1. Lab giữ snapshot cũ để tái lập. Thực tế cần quy trình xóa có kiểm soát qua Bronze,
   snapshot, transcript, cache, index và backup; thu hồi version cũ, tạo bản sạch,
   giữ metadata kiểm toán phù hợp thay vì giữ plaintext chỉ để tái lập.
2. Đặt gate PII trước Silver/Gold, kết hợp regex và nhận diện tên/địa chỉ tiếng Việt.
   Hạn chế quyền Bronze, rà soát log/cache; đo precision/recall theo loại PII trên
   tập gán nhãn và tỷ lệ rò rỉ sau gate, gồm cả redact nhầm.

## 5. Output thực tế

Output dưới đây lấy từ các lần chạy trên mã nguồn đã sửa. Mã màu ANSI của dbt
được bỏ khi trình bày; log gốc giữ tại [evidence/](evidence/).

### Verify

```text
$ .\.venv\Scripts\python.exe -m scripts.verify
=== verify.py — Day 17 pipeline contracts ===
  [OK ] Bronze  every daily batch landed as Parquet (7 days x 3 sources)
  [OK ] Bronze  re-landing a batch is a no-op (append-only, no duplicate file)
  [OK ] Bronze  Bronze keeps the raw truth: Kafka tombstone + redelivered events are still there
  [OK ] Silver  silver_tickets has exactly one row per ticket_id
  [OK ] Silver  T-91 shows its latest state: high / closed / bug
  [OK ] Silver  deleted ticket T-97 is a tombstone: is_deleted and no personal data left
  [OK ] Silver  no email / phone number survives past Bronze
  [OK ] Silver  silver_events has one row per event_id (Kafka redeliveries removed)
  [OK ] Silver  2 malformed events quarantined with a reason; the run did not halt
  [OK ] Gold    gold_feature_daily reconciles with a full recompute from Silver
  [OK ] Gold    u05's offline events of 08-12 (arrived 08-15) are counted on 08-12
  [OK ] Gold    LOOKBACK_DAYS covers measured P99 lateness (p99=3.00 days)
  [OK ] Gold    training set uses point-in-time priority (T-91 created as 'low')
  [OK ] Gold    late feedback creates a NEW snapshot version; the old one is untouched
  [OK ] Gold    latest training snapshot excludes the deleted ticket T-97
  [OK ] Gold    deletes propagate to the RAG index: no chunk of T-97
  [OK ] Gold    gold_doc_chunks: one row per chunk, and a re-run embeds 0 new chunks
  [OK ] Rerun   re-run 2026-08-12 three times -> Gold checksum identical to a fresh build

RESULT: 18/18 checks — ALL PASS
re-run checksums written to submission/checksums.txt
```

### Pytest

```text
$ .\.venv\Scripts\python.exe -m pytest
..................................                                       [100%]
34 passed in 4.60s
```

### Rerun

```text
$ .\.venv\Scripts\python.exe -m scripts.rerun_check
# Lab 17 — re-run check for 2026-08-12

run                     gold_feature_daily    gold_training_set     gold_doc_chunks       gold (combined)
fresh build             8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #1 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #2 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #3 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f

RESULT: PASS — 3 re-runs, identical checksums
```

### Lateness

```text
$ .\.venv\Scripts\python.exe main.py --lateness
event lateness over 43 Bronze records (calendar days): p50=0.00 p95=2.90 p99=3.00 max=3
-> lookback must be >= ceil(p99) = 3 day(s); config.LOOKBACK_DAYS = 3
```

### dbt build

```text
$ D:\Track2_AI_Data_Infrastructure\K4-Track02-Day17-Data-Pipeline-Engineering\.venv\Scripts\dbt.exe build --profiles-dir . --event-time-start 2026-08-10 --event-time-end 2026-08-17
11:43:04  Running with dbt=1.12.5
11:43:04  Registered adapter: duckdb=1.11.0
11:43:05  Found 5 models, 13 data tests, 2 sources, 502 macros, 1 unit test
11:43:05  
11:43:05  Concurrency: 1 threads (target='dev')
11:43:05  
11:43:05  1 of 19 START sql view model main.stg_events ................................... [RUN]
11:43:06  1 of 19 OK created sql view model main.stg_events .............................. [OK in 0.08s]
11:43:06  2 of 19 START sql view model main.stg_ticket_changes ........................... [RUN]
11:43:06  2 of 19 OK created sql view model main.stg_ticket_changes ...................... [OK in 0.03s]
11:43:06  3 of 19 START sql incremental model main.silver_events ......................... [RUN]
11:43:06  3 of 19 OK created sql incremental model main.silver_events .................... [OK in 0.12s]
11:43:06  4 of 19 START unit_test silver_tickets::silver_tickets_latest_change_wins_and_delete_is_tombstone  [RUN]
11:43:06  4 of 19 PASS silver_tickets::silver_tickets_latest_change_wins_and_delete_is_tombstone  [PASS in 0.17s]
11:43:06  8 of 19 START sql incremental model main.silver_tickets ........................ [RUN]
11:43:06  8 of 19 OK created sql incremental model main.silver_tickets ................... [OK in 0.07s]
11:43:06  5 of 19 START test not_null_silver_events_event_id ............................. [RUN]
11:43:06  5 of 19 PASS not_null_silver_events_event_id ................................... [PASS in 0.04s]
11:43:06  6 of 19 START test not_null_silver_events_user_id .............................. [RUN]
11:43:06  6 of 19 PASS not_null_silver_events_user_id .................................... [PASS in 0.02s]
11:43:06  7 of 19 START test unique_silver_events_event_id ............................... [RUN]
11:43:06  7 of 19 PASS unique_silver_events_event_id ..................................... [PASS in 0.02s]
11:43:06  9 of 19 START test accepted_values_silver_tickets_category__bug__billing__other  [RUN]
11:43:06  9 of 19 PASS accepted_values_silver_tickets_category__bug__billing__other ...... [PASS in 0.07s]
11:43:06  10 of 19 START test accepted_values_silver_tickets_priority__low__medium__high . [RUN]
11:43:06  10 of 19 PASS accepted_values_silver_tickets_priority__low__medium__high ....... [PASS in 0.04s]
11:43:06  11 of 19 START test accepted_values_silver_tickets_status__open__pending__closed  [RUN]
11:43:06  11 of 19 PASS accepted_values_silver_tickets_status__open__pending__closed ..... [PASS in 0.03s]
11:43:06  12 of 19 START test not_null_silver_tickets__lsn ............................... [RUN]
11:43:06  12 of 19 PASS not_null_silver_tickets__lsn ..................................... [PASS in 0.02s]
11:43:06  13 of 19 START test not_null_silver_tickets_is_deleted ......................... [RUN]
11:43:06  13 of 19 PASS not_null_silver_tickets_is_deleted ............................... [PASS in 0.01s]
11:43:06  14 of 19 START test not_null_silver_tickets_ticket_id .......................... [RUN]
11:43:06  14 of 19 PASS not_null_silver_tickets_ticket_id ................................ [PASS in 0.03s]
11:43:06  15 of 19 START test unique_silver_tickets_ticket_id ............................ [RUN]
11:43:06  15 of 19 PASS unique_silver_tickets_ticket_id .................................. [PASS in 0.01s]
11:43:06  16 of 19 START sql microbatch model main.gold_feature_daily .................... [RUN]
11:43:06  Batch 1 of 7 START batch 2026-08-10 of main.gold_feature_daily ....................... [RUN]
11:43:06  Batch 1 of 7 OK created batch 2026-08-10 of main.gold_feature_daily .................. [OK in 0.02s]
11:43:06  Batch 2 of 7 START batch 2026-08-11 of main.gold_feature_daily ....................... [RUN]
11:43:06  Batch 2 of 7 OK created batch 2026-08-11 of main.gold_feature_daily .................. [OK in 0.05s]
11:43:06  Batch 3 of 7 START batch 2026-08-12 of main.gold_feature_daily ....................... [RUN]
11:43:06  Batch 3 of 7 OK created batch 2026-08-12 of main.gold_feature_daily .................. [OK in 0.06s]
11:43:06  Batch 4 of 7 START batch 2026-08-13 of main.gold_feature_daily ....................... [RUN]
11:43:06  Batch 4 of 7 OK created batch 2026-08-13 of main.gold_feature_daily .................. [OK in 0.08s]
11:43:06  Batch 5 of 7 START batch 2026-08-14 of main.gold_feature_daily ....................... [RUN]
11:43:07  Batch 5 of 7 OK created batch 2026-08-14 of main.gold_feature_daily .................. [OK in 0.08s]
11:43:07  Batch 6 of 7 START batch 2026-08-15 of main.gold_feature_daily ....................... [RUN]
11:43:07  Batch 6 of 7 OK created batch 2026-08-15 of main.gold_feature_daily .................. [OK in 0.05s]
11:43:07  Batch 7 of 7 START batch 2026-08-16 of main.gold_feature_daily ....................... [RUN]
11:43:07  Batch 7 of 7 OK created batch 2026-08-16 of main.gold_feature_daily .................. [OK in 0.03s]
11:43:07  16 of 19 OK created sql microbatch model main.gold_feature_daily ............... [SUCCESS in 0.42s]
11:43:07  17 of 19 START test dbt_utils_free_unique_combination_gold_feature_daily_user_id__event_date  [RUN]
11:43:07  17 of 19 PASS dbt_utils_free_unique_combination_gold_feature_daily_user_id__event_date  [PASS in 0.03s]
11:43:07  18 of 19 START test not_null_gold_feature_daily_event_date ..................... [RUN]
11:43:07  18 of 19 PASS not_null_gold_feature_daily_event_date ........................... [PASS in 0.01s]
11:43:07  19 of 19 START test not_null_gold_feature_daily_user_id ........................ [RUN]
11:43:07  19 of 19 PASS not_null_gold_feature_daily_user_id .............................. [PASS in 0.03s]
11:43:07  
11:43:07  Finished running 3 incremental models, 13 data tests, 1 unit test, 2 view models in 0 hours 0 minutes and 1.53 seconds (1.53s).
11:43:07  
11:43:07  Completed successfully
11:43:07  
11:43:07  Done. PASS=19 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=19
```

### Parity

```text
$ .\.venv\Scripts\python.exe -m scripts.parity
=== parity: lite pipeline vs dbt ===
  [OK ] silver_tickets       lite 3c15dfd43701  dbt 3c15dfd43701
  [OK ] gold_feature_daily   lite 8630e04a61d1  dbt 8630e04a61d1
RESULT: PARITY — both implementations agree
```

### Bonus LLM

```text
$ .\.venv\Scripts\python.exe -m scripts.bonus_llm
=== bonus: LLM labelling of 11 live tickets ===
  cost estimate before running: ~484 tokens = $0.0010 per full run
  cache-miss cost estimate before running: ~484 tokens = $0.0010
  cache-miss cost estimate before running: ~0 tokens = $0.0000
  cache-miss cost estimate before running: ~484 tokens = $0.0010
  [OK ] first run labels every live ticket
  [OK ] re-run with same model + prompt makes 0 LLM calls
  [OK ] every Gold label is bug / billing / other
  [OK ] off-schema answers go to llm_label_quarantine
  [OK ] new prompt version re-labels on purpose
  [OK ] labels carry their prompt version
BONUS PASS
```

### Extension Flywheel

```text
$ .\.venv\Scripts\python.exe -m extensions.flywheel
=== Day 17 flywheel: agent traces -> datasets ===
  spans landed in Bronze   : 21 (from 8 traces)
  eval golden rows         : 2  -> eval_golden.jsonl
  preference pairs (raw)   : 3
  preference pairs (clean) : 1  -> preference_pairs.jsonl  (2 dropped by decontamination)

Per-trace summary (the analyst view):
trace_id                                  user_input                                                                                     agent_output outcome  total_tokens  latency_ms  n_spans
   t0001 Can I return a widget I bought 10 days ago? Yes. Widgets can be returned within 30 days for a full refund, so a 10-day-old widget qualifies.      ok         578.0        1840        3
   t0002           What warranty does a gadget have?                          Gadgets carry a 90-day limited warranty covering manufacturing defects.      ok         534.0        2110        3
   t0003 Can I return a widget I bought 10 days ago?                 Sorry, I could not look up the returns policy right now. Please try again later.   error           0.0         980        2
   t0004                   Are sprockets returnable?                                 No. Sprockets are final sale and cannot be returned once opened.      ok         524.0        1520        3
   t0005           What warranty does a gadget have?                                                                 I am not able to help with that.   error         499.0         760        2
   t0006            How long is the gadget warranty?                        A gadget has a 90-day limited warranty that covers manufacturing defects.      ok         548.0        1990        3
   t0007  Do you accept returns on opened sprockets?                                No. Once opened, sprockets are final sale and cannot be returned.      ok         530.0        1610        3
   t0008  Do you accept returns on opened sprockets?                                   Sure! You can return opened sprockets any time within 30 days.   error         511.0         890        2

Point-in-time features (ASOF, correct) vs naive (leaky):
user_id            event_ts  spend_at_event  spend_leaky
   u100 2026-06-01 10:00:00            50.0        300.0
   u100 2026-06-03 09:00:00           120.0        300.0
   u101 2026-06-02 12:00:00            20.0         20.0

  rows where the naive join LEAKED a future value: 2
```

### Extension Knowledge Graph

```text
$ .\.venv\Scripts\python.exe -m extensions.kg_demo
=== Day 17 bonus: Knowledge Graph vs Vector RAG ===
  nodes (entities): 5
    (accessory) -[SHIPS_FROM]-> (hanoi fulfillment center)
    (gadget) -[IS_A]-> (accessory)
    (gadget) -[HAS_WARRANTY]-> (90 days)
    (industrial-part) -[SHIPS_FROM]-> (hai phong fulfillment center)
    (sprocket) -[IS_A]-> (industrial-part)
    (sprocket) -[NON_RETURNABLE]-> (final sale)
    (widget) -[IS_A]-> (accessory)
    (widget) -[RETURNABLE_WITHIN]-> (30 days)

  Q: 'Where does a widget ship from?'

  [Vector RAG] flat chunk retrieval:
    chunk mentioning 'widget' : 'Widgets and gadgets belong to the accessory category.'
    chunk mentioning 'hanoi'  : 'Accessories ship from the Hanoi fulfillment center.'
    one chunk that answers it (has BOTH): False  => flat retrieval CANNOT bridge the two facts

  [Knowledge graph] multi-hop traverse:
    widget  ->  accessory  ->  hanoi fulfillment center   (2 hops)  => hanoi fulfillment center

  1-hop  widget / RETURNABLE_WITHIN : [('RETURNABLE_WITHIN', '30 days')]
  multi-node 'what is returnable?'  : ['widget']
```


## 6. Bonus và extensions

- **B1:** cache SHA-256(input)+model+prompt version, validate JSON label, cache cả
  câu trả lời sai, quarantine idempotent và ước tính chi phí trước call. Checker
  đạt BONUS PASS. Kiểm tra bổ sung đổi model/input, nội dung trùng và schema sai
  được lưu ở [bonus-edge-cases.txt](evidence/bonus-edge-cases.txt).
- **B2:** chọn brainstorm; [bonus/DESIGN.md](../bonus/DESIGN.md) có 1.585 từ theo
  whitespace, 6 quyết định cùng đánh đổi, phương án bị loại và sơ đồ kiến trúc.
  Quy mô/SLA/ngân sách là giả định thiết kế. Không nộp bằng chứng Airflow:
  Docker Engine trên máy chưa chạy; hai lựa chọn B2 không cộng dồn.
- **Flywheel:** 21 spans/8 traces, 2 eval rows; 3 preference pairs → 1 sau
  decontamination; naive join rò rỉ giá trị tương lai ở 2 hàng.
- **KG:** traversal 2-hop widget→accessory→hanoi fulfillment center; không có
  một chunk chứa cả hai đầu chuỗi trong demo. Đây là demo luật, chưa đánh giá RAG thực.

## 7. Checkpoint và trạng thái nộp

| Checkpoint | Sản phẩm / bằng chứng |
|---|---|
| CP1 | Baseline run/verify/pytest/lateness trong `evidence/baseline-*.txt` |
| CP2 | Keyed MERGE + LSN guard; 2 test chọn lọc pass trong `cp2-silver.txt` |
| CP3 | P99 đo từ Bronze, lookback=3; 3 test chọn lọc pass trong `cp3-lateness.txt` |
| CP4 | Delete tombstone; verify 18/18, pytest 34 pass, checksums PASS |
| CP5 | dbt PASS=19, parity PARITY, log trong phần 5 |
| CP6 | REPORT, DESIGN và bằng chứng đã commit/push; repo public, remote commit và checksum đã kiểm tra; nộp LMS chưa hoàn tất |

