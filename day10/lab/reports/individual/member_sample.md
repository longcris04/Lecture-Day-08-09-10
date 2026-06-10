# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** [Điền tên của bạn]
**Vai trò:** Cleaning & Quality Owner
**Ngày nộp:** 2026-06-11
**Độ dài yêu cầu:** 400–650 từ

---

## 1. Tôi phụ trách phần nào?

**File / module:**

- `transform/cleaning_rules.py` — thêm Rule 1, 2, 3 và cập nhật `ALLOWED_DOC_IDS`
- `quality/expectations.py` — thêm E7 (`access_control_sop_present`) và E8 (`no_corrupted_content_marker`)

**Kết nối với thành viên khác:**

Rule 1-3 trong cleaning_rules.py là input trực tiếp cho expectations.py — nếu rule không quarantine đúng chunk, expectation sẽ FAIL và block toàn bộ pipeline (halt). Phần embed của thành viên Embed Owner phụ thuộc vào cleaning PASS hết expectations trước.

**Bằng chứng:**

- `transform/cleaning_rules.py` dòng 24: `"access_control_sop"` thêm vào `ALLOWED_DOC_IDS`
- `transform/cleaning_rules.py` dòng 126-146: Rule 1, 2, 3 với comment `metric_impact`
- `quality/expectations.py` dòng 115-138: E7, E8

---

## 2. Một quyết định kỹ thuật

**Quyết định: đặt Rule 1 (stale_hr_content_marker) TRƯỚC dedup, không phải sau.**

Rule 1 quarantine chunk hr_leave_policy chứa "10 ngày phép năm" bất kể effective_date. Nếu đặt sau dedup, một số chunk stale đã bị dedup (không còn trong cleaned) — E6 vẫn có thể PASS tình cờ mà không phải vì rule đúng.

Đặt Rule 1 TRƯỚC dedup đảm bảo: mọi chunk stale đều bị quarantine với `reason=stale_hr_content_marker`, không phụ thuộc vào thứ tự xuất hiện trong CSV. Điều này làm quarantine_records chính xác hơn (8 rows với reason cụ thể thay vì một số bị "nuốt" vào `duplicate_chunk_text`).

---

## 3. Một lỗi đã xử lý

**Triệu chứng:** Pipeline HALT ngay từ lần chạy đầu tiên sau khi thêm access_control_sop vào allowlist:

```
expectation[hr_leave_no_stale_10d_annual] FAIL (halt) :: violations=2
```

**Metric phát hiện:** E6 FAIL với `violations=2` — 2 chunk hr_leave_policy có "10 ngày phép năm" lọt vào cleaned dù đã có date filter (vì effective_date = 2026-03-30, >= 2026-01-01).

**Root cause:** Date filter chỉ lọc theo ngày, không kiểm tra nội dung. Row 6 (date 2026-03-30) và row 36 (date 2026-02-10) là bản HR 2025 bị export lại với ngày mới — date hợp lệ nhưng content stale.

**Fix:** Thêm Rule 1 content-based:
```python
if doc_id == "hr_leave_policy" and "10 ngày phép năm" in text:
    quarantine(..., reason="stale_hr_content_marker")
```

**Kết quả:** E6 PASS `violations=0`. 8 rows bị quarantine với reason đúng.

---

## 4. Bằng chứng trước / sau

**run_id:** `2026-06-10T18-08Z`

**Trước (baseline — E6 FAIL):**
```
expectation[hr_leave_no_stale_10d_annual] FAIL (halt) :: violations=2
PIPELINE_HALT: expectation suite failed (halt).
```

**Sau (Rule 1 added — E6 PASS):**
```
expectation[hr_leave_no_stale_10d_annual] OK (halt) :: violations=0
embed_upsert count=42 collection=day10_kb
PIPELINE_OK
```

Grading: `gq_d10_09` — `contains_expected=True`, `hits_forbidden=False`, `top1_doc_matches=True`  
(artifacts/eval/grading_run.jsonl)

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ: fix `gq_d10_06` bằng cách thêm chunk escalation vào `sla_p1_2026` với nội dung rõ ràng hơn về "10 phút", hoặc thêm một expectation kiểm tra `sla_p1_2026` phải có chunk chứa keyword "escalate" + "10 phút" — tương tự cách E7 đảm bảo access_control_sop có data. Điều này sẽ phát hiện sớm khi chunk escalation bị mất hoặc không rank được.
