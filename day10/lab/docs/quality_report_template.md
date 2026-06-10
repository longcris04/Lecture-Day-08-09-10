# Quality report — Lab Day 10 (nhóm)

**run_id:** `2026-06-10T18-08Z`  
**Ngày:** 2026-06-11

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (baseline) | Sau (pipeline chuẩn) | Ghi chú |
|--------|-----------------|----------------------|---------|
| raw_records | 247 | 247 | Dataset cố định |
| cleaned_records | 40 | **42** | +6 access_control_sop, -4 new rules |
| quarantine_records | 207 | **205** | 2 chunk access_control_sop hợp lệ |
| Expectation halt? | **CÓ** — E6 FAIL `violations=2` | **KHÔNG** — 8/8 halt PASS | Thêm Rule 1 + access_control_sop vào allowlist |

**Quarantine phân loại (run chuẩn):**

| Reason | Count |
|--------|-------|
| `unknown_doc_id` | 109 |
| `duplicate_chunk_text` | 50 |
| `stale_hr_policy_effective_date` | 22 |
| `stale_hr_content_marker` *(Rule 1 — mới)* | 8 |
| `missing_chunk_text` | 8 |
| `missing_effective_date` | 6 |
| `excessive_repetition` *(Rule 3 — mới)* | 1 |
| `corrupted_content_marker` *(Rule 2 — mới)* | 1 |
| **TOTAL** | **205** |

---

## 2. Before / after retrieval (bắt buộc)

**Files:** `artifacts/eval/after_inject_bad.csv` (before) vs `artifacts/eval/after_fix_eval.csv` (after)  
**Grading chính thức:** `artifacts/eval/grading_run.jsonl` (top_k=5)

---

### Câu hỏi then chốt: refund window (`gq_d10_01`)

> "Khách hàng có tối đa bao nhiêu ngày làm việc để gửi yêu cầu hoàn tiền?"  
> `must_contain_any: ["7 ngày"]` | `must_not_contain: ["14 ngày", "14 ngày làm việc"]`

**Trước (inject-bad — `--no-refund-fix`, E3 FAIL):**
```
gq_d10_01,top1=policy_refund_v4,contains_expected=yes,hits_forbidden=yes,top1_doc_expected=yes
```
E3 phát hiện: `violations=2` — 2 chunk có "14 ngày làm việc" trong cleaned.

**Sau (pipeline chuẩn — refund fix ON, E3 PASS):**
```
gq_d10_01,top1=policy_refund_v4,contains_expected=True,hits_forbidden=False,top1_doc_matches=True
```
E3: `violations=0`. "14 ngày làm việc" đã được replace thành "7 ngày làm việc [cleaned: stale_refund_window]".

---

### Merit: HR leave versioning (`gq_d10_09`)

> "Nhân viên dưới 3 năm được bao nhiêu ngày phép năm theo chính sách HR 2026?"  
> `must_contain_any: ["12 ngày"]` | `must_not_contain: ["10 ngày phép năm"]`

**Trước (baseline — trước Rule 1, E6 FAIL):**

E6 FAIL `violations=2` → pipeline HALT, không embed → nếu bypass: chunk "10 ngày phép năm" lọt vào ChromaDB → `hits_forbidden=yes`.

**Sau (Rule 1 active — E6 PASS):**
```
gq_d10_09,top1=hr_leave_policy,contains_expected=True,hits_forbidden=False,top1_doc_matches=True
```
8 chunk HR stale quarantined. "12 ngày" là top-1, không còn "10 ngày" trong top-k.

---

### Tổng hợp grading 10 câu (grading_run.jsonl — top_k=5)

| Question | contains | forbidden | top1_ok | Kết quả |
|----------|---------|-----------|---------|---------|
| gq_d10_01 | True | False | True | **PASS** |
| gq_d10_02 | True | False | True | **PASS** |
| gq_d10_03 | True | False | True | **PASS** |
| gq_d10_04 | True | False | True | **PASS** |
| gq_d10_05 | True | False | True | **PASS** |
| gq_d10_06 | **False** | False | True | **FAIL** |
| gq_d10_07 | True | False | True | **PASS** |
| gq_d10_08 | True | False | True | **PASS** |
| gq_d10_09 | True | False | True | **PASS** |
| gq_d10_10 | True | False | True | **PASS** |

**9/10 PASS.** `gq_d10_06` fail: chunk "10 phút escalate" có trong ChromaDB nhưng không rank top-5 với Vietnamese query.

---

## 3. Freshness & monitor

**SLA chọn:** 24 giờ (`FRESHNESS_SLA_HOURS=24`).

**Kết quả manifest `2026-06-10T18-08Z`:**
```
freshness_check=FAIL
reason=freshness_sla_exceeded
latest_exported_at=2026/04/07T00:00:00
age_hours=1554.1
sla_hours=24.0
```

**Giải thích 3 trạng thái:**

| Trạng thái | Điều kiện | Ý nghĩa |
|-----------|-----------|---------|
| `PASS` | `age_hours <= 24` | Data tươi, trong SLA — không cần làm gì |
| `WARN` | Không parse được timestamp | Bug format `exported_at` — đã fix: `ts.replace("/", "-", 2)` |
| `FAIL` | `age_hours > 24` | Data quá cũ — trigger lại pipeline với data mới |

**Bug đã fix:** `monitoring/freshness_check.py` không parse được `2026/04/07T...` (dùng `/` thay `-`). Trả về `WARN` nhầm thay vì `FAIL`. Fix bằng normalize format trước khi parse.

**Cơ chế idempotent:** Mỗi run, `embed_prune` xóa chunk_id cũ không còn trong cleaned. Minh chứng: restore sau inject-bad → `embed_prune_removed=2`.

---

## 4. Corruption inject (Sprint 3)

**Lệnh inject:**
```bash
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
```

**Loại corruption:**

| Kiểu | Cơ chế | Kết quả |
|------|--------|---------|
| Stale refund window | `--no-refund-fix` tắt replace "14 ngày" → "7 ngày" | 2 chunk policy_refund_v4 có "14 ngày làm việc" trong cleaned |
| Bypass halt | `--skip-validate` bỏ qua E3 FAIL | Pipeline embed dù expectation phát hiện lỗi |

**Log chứng minh (`artifacts/logs/run_inject-bad.log`):**
```
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=2
WARN: expectation failed but --skip-validate -> tiep tuc embed
embed_upsert count=42 collection=day10_kb
PIPELINE_OK
```

**Phát hiện:** E3 bắt chính xác `violations=2`. Expectation phát hiện lỗi ngay cả khi retrieval eval (top-k metric) không thay đổi — chứng minh expectation suite quan trọng hơn eval metric đơn thuần.

**Restore:**
```bash
python etl_pipeline.py run   # embed_prune_removed=2 → ChromaDB sạch
```

---

## 5. Hạn chế & việc chưa làm

| Vấn đề | Mức độ | Hướng xử lý |
|--------|--------|-------------|
| `gq_d10_06` fail: "10 phút escalate" không rank top-5 với Vietnamese query | Trung bình | Tăng top_k, hoặc thêm expectation kiểm tra sla_p1_2026 có chunk escalation |
| Freshness FAIL do dataset lab cố định (exported 2026-04-07) | Thấp (lab) | Bình thường — production cần scheduler trigger pipeline theo lịch |
| `reports/individual/[ten].md` chưa điền thông tin thật của từng thành viên | Chưa làm | Mỗi thành viên tự điền theo template `reports/individual/member_sample.md` |
