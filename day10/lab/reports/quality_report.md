# Quality report — Lab Day 10 (nhóm)

**run_id:** `2026-06-10T18-08Z`  
**Ngày:** 2026-06-11

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (baseline / inject-bad) | Sau (pipeline chuẩn) | Ghi chú |
|--------|-------------------------------|----------------------|---------|
| raw_records | 247 | 247 | Dataset cố định |
| cleaned_records | 40 (baseline, thiếu access_control_sop) | **42** | +6 access_control_sop, -4 new rules |
| quarantine_records | 207 (baseline) | **205** | 2 chunk access_control_sop giải phóng |
| Expectation halt? | **CÓ** — E6 FAIL `violations=2` | **KHÔNG** — 8/8 halt PASS | Nguyên nhân: stale HR content marker |
| Inject-bad cleaned | 42 | 42 | cleaned count không đổi (Rule 2 vẫn chạy) |
| Inject-bad E3 | FAIL `violations=2` | PASS `violations=0` | `--no-refund-fix` tắt replace 14→7 ngày |

**Quarantine breakdown (run chuẩn):**

| Reason | Count |
|--------|-------|
| `unknown_doc_id` | 109 |
| `duplicate_chunk_text` | 50 |
| `stale_hr_policy_effective_date` | 22 |
| `stale_hr_content_marker` | 8 |
| `missing_chunk_text` | 8 |
| `missing_effective_date` | 6 |
| `excessive_repetition` | 1 |
| `corrupted_content_marker` | 1 |
| **TOTAL** | **205** |

---

## 2. Before / after retrieval (bắt buộc)

**Artifacts:** `artifacts/eval/after_inject_bad.csv` (before) vs `artifacts/eval/after_fix_eval.csv` (after)

### Câu hỏi then chốt: refund window (`gq_d10_01`)

> "Khách hàng có tối đa bao nhiêu ngày làm việc để gửi yêu cầu hoàn tiền?"  
> `must_contain_any: ["7 ngày"]` | `must_not_contain: ["14 ngày", "14 ngày làm việc"]`

**Trước (inject-bad — `--no-refund-fix`):**
```
top1_doc_id=policy_refund_v4 | contains_expected=yes | hits_forbidden=yes | top1_doc_expected=yes
```
E3 phát hiện: `violations=2` (2 chunk policy_refund_v4 có "14 ngày làm việc" trong cleaned).  
`hits_forbidden=yes` do top-k chứa chunk có "14 ngày" (context khác, không phải "14 ngày làm việc").

**Sau (pipeline chuẩn — refund fix ON):**
```
top1_doc_id=policy_refund_v4 | contains_expected=yes | hits_forbidden=yes | top1_doc_expected=yes
```
E3 PASS: `violations=0` (không còn "14 ngày làm việc" trong cleaned).  
`hits_forbidden=yes` còn lại do 1 chunk policy_refund_v4 có "14 ngày" trong ngữ cảnh khác — xem mục 5.

**Kết luận:** Expectation E3 phân biệt rõ 2 trạng thái (FAIL vs PASS). Retrieval metric `hits_forbidden` không phân biệt được vì lý do residual chunk — chứng minh expectation quan trọng hơn eval metric trong trường hợp này.

---

### HR leave version (`gq_d10_09`)

> "Nhân viên dưới 3 năm được bao nhiêu ngày phép năm theo chính sách HR 2026?"  
> `must_contain_any: ["12 ngày"]` | `must_not_contain: ["10 ngày phép năm"]`

**Trước (baseline — trước khi thêm Rule 1):**
- E6 FAIL: `violations=2` — chunk "10 ngày phép năm" lọt vào cleaned
- Nếu embed → `hits_forbidden=yes`, `contains_expected` không đảm bảo

**Sau (pipeline chuẩn — Rule 1 active):**
```
top1_doc_id=hr_leave_policy | contains_expected=yes | hits_forbidden=no | top1_doc_expected=yes
```
Rule 1 quarantine 8 chunk HR stale → E6 PASS `violations=0` → "12 ngày" là top-1, không còn "10 ngày" trong top-k.

---

### Tổng hợp 10 câu hỏi

| State | contains=yes | hits_forbidden=no | top1_ok=yes | Fully clean |
|-------|-------------|-------------------|-------------|-------------|
| after_fix | 9/10 | 9/10 | 10/10 | **8/10** |
| after_inject_bad | 9/10 | 9/10 | 10/10 | **8/10** |

`gq_d10_06` (`contains=no`): "10 phút auto escalate" — chunk sla_p1_2026 có thông tin nhưng không rank top-3.  
`gq_d10_01` (`hits_forbidden=yes`): residual "14 ngày" chunk trong policy_refund_v4 — xem mục 5.

---

## 3. Freshness & monitor

**SLA chọn:** 24 giờ (default `FRESHNESS_SLA_HOURS=24`).

**Kết quả manifest `2026-06-10T18-08Z`:**
```
freshness_check=FAIL
reason=freshness_sla_exceeded
latest_exported_at=2026/04/07T00:00:00
age_hours=1554.1  (>65 ngày)
sla_hours=24.0
```

**Ý nghĩa:**

| Trạng thái | Điều kiện | Ý nghĩa |
|-----------|-----------|---------|
| `PASS` | age_hours ≤ 24 | Data tươi, trong SLA |
| `WARN` | Không parse được timestamp | Bug format — đã fix (`/` → `-`) |
| `FAIL` | age_hours > 24 | Data quá cũ, cần trigger pipeline mới |

**Bug đã fix trong `monitoring/freshness_check.py`:**  
Format `2026/04/07T00:00:00` dùng `/` thay `-` → `datetime.fromisoformat()` không parse được → trả về `WARN` nhầm. Fix: `ts.replace("/", "-", 2)` trước parse. Sau fix → trả về `FAIL` đúng.

**Cơ chế idempotent:** Mỗi run, `cmd_embed_internal()` prune chunk_id cũ không còn trong cleaned.  
Minh chứng: Sau khi restore từ inject-bad → `embed_prune_removed=2` (2 chunk "14 ngày làm việc" bị xóa khỏi ChromaDB).

---

## 4. Corruption inject (Sprint 3)

**Lệnh inject:**
```bash
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
```

**Kiểu corruption inject:**

| Loại | Cơ chế | Kết quả trong cleaned |
|------|--------|----------------------|
| Stale refund window | `--no-refund-fix` tắt replace "14 ngày làm việc" → "7 ngày làm việc" | 2 chunk policy_refund_v4 có "14 ngày làm việc" |
| Bypass halt | `--skip-validate` bỏ qua E3 FAIL | Pipeline embed dù có violations |

**Log chứng minh (artifacts/logs/run_inject-bad.log):**
```
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=2
WARN: expectation failed but --skip-validate -> tiep tuc embed
embed_upsert count=42 collection=day10_kb
PIPELINE_OK
```

**Phát hiện bằng expectation:**  
E3 `refund_no_stale_14d_window` bắt chính xác `violations=2` — pipeline BIẾT có lỗi nhưng bị buộc tiếp tục do `--skip-validate`. Đây là lý do flag này chỉ tồn tại để demo, không dùng production.

**So sánh eval before/after inject:**

| File | Fully clean (contains=yes + forbidden=no) | E3 status |
|------|------------------------------------------|-----------|
| `artifacts/eval/after_inject_bad.csv` | 8/10 | **FAIL** violations=2 |
| `artifacts/eval/after_fix_eval.csv` | 8/10 | **PASS** violations=0 |

Lưu ý: retrieval metric không thay đổi vì 2 chunk "14 ngày làm việc" không rank cao đủ trong top-3. **Expectation là lớp phát hiện đáng tin cậy hơn eval metric** trong trường hợp này.

**Restore về clean state:**
```bash
python etl_pipeline.py run
# embed_prune_removed=2  ← 2 chunk inject-bad bị xóa
# PIPELINE_OK
```

---

## 5. Hạn chế & việc chưa làm

| Vấn đề | Mức độ | Hướng xử lý đề xuất |
|--------|--------|---------------------|
| `gq_d10_01` `hits_forbidden=yes` sau fix: chunk policy_refund_v4 có "14 ngày" (không phải "14 ngày làm việc") vẫn trong top-k | Trung bình | Thêm rule quarantine chunk có standalone "14 ngày" trong policy_refund_v4; hoặc mở rộng replace regex |
| `gq_d10_06` `contains_expected=no`: "10 phút auto escalate" không rank top-3 | Trung bình | Kiểm tra chunk sla_p1_2026 có đầy đủ nội dung escalation; có thể cần tăng `top_k` |
| Freshness FAIL do dataset lab cố định (exported 2026-04-07, ~65 ngày trước) | Thấp | Bình thường với dataset lab; production cần scheduler trigger pipeline theo lịch |
| `grading_run.py` chưa chạy để xác nhận điểm grading chính thức | Chưa làm | `python grading_run.py --out artifacts/eval/grading_run.jsonl` |
| `data/test_questions.json` chưa được dùng trong eval (dùng grading_questions.json thay thế) | Thấp | Kiểm tra xem 2 file có nội dung khác nhau không |
