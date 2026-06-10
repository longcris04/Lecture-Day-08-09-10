# Runbook — Lab Day 10 (incident tối giản)

---

## Incident 1: Chatbot trả lời sai chính sách hoàn tiền ("14 ngày" thay vì "7 ngày")

### Symptom
Người dùng / agent nhận được câu trả lời: *"Yêu cầu hoàn tiền phải gửi trong vòng 14 ngày làm việc"*  
Đúng phải là: **7 ngày làm việc**.

### Detection

| Metric | Dấu hiệu |
|--------|---------|
| Expectation `refund_no_stale_14d_window` | `FAIL (halt) :: violations=N` trong log |
| Eval `hits_forbidden` | `yes` cho `gq_d10_01` trong `artifacts/eval/*.csv` |
| Freshness | `FAIL reason=freshness_sla_exceeded` — data quá cũ, có thể bản stale được re-export |

### Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Đọc `artifacts/manifests/manifest_<run_id>.json` — kiểm tra `no_refund_fix: false` | Nếu `true` → pipeline chạy với `--no-refund-fix` |
| 2 | Đọc `artifacts/quarantine/<run_id>.csv` — lọc `doc_id=policy_refund_v4` | Xem reason — nếu không có `stale_refund_window` → rule fix bị tắt |
| 3 | Kiểm tra log `artifacts/logs/run_<run_id>.log` | `expectation[refund_no_stale_14d_window] FAIL` |
| 4 | Chạy `python eval_retrieval.py --questions data/grading_questions.json --out artifacts/eval/debug.csv` | `gq_d10_01 hits_forbidden=yes` |

### Mitigation

```bash
# Rerun pipeline chuẩn (không có --no-refund-fix)
python etl_pipeline.py run

# Xác nhận fix
python eval_retrieval.py --questions data/grading_questions.json --out artifacts/eval/after_fix.csv
```

Kiểm tra log: `expectation[refund_no_stale_14d_window] OK (halt) :: violations=0`

### Prevention
- Không dùng `--no-refund-fix` trong production (chỉ dùng cho Sprint 3 demo)
- Thêm CI check: nếu expectation fail → block merge

---

## Incident 2: Pipeline HALT — không embed được

### Symptom
```
PIPELINE_HALT: expectation suite failed (halt).
```
Exit code 2. ChromaDB không được cập nhật.

### Detection
Log `artifacts/logs/run_<run_id>.log` chứa dòng `FAIL (halt)`.

### Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Đọc log → tìm dòng `FAIL (halt)` | Biết expectation nào fail |
| 2 | Đọc quarantine CSV của run đó | Xem distribution `reason` — bất thường ở đâu? |
| 3 | So sánh `raw_records` vs `cleaned_records` | Tỉ lệ quarantine đột ngột tăng? |

**Mapping expectation → nguyên nhân thường gặp:**

| Expectation FAIL | Nguyên nhân |
|-----------------|-------------|
| `hr_leave_no_stale_10d_annual` | Chunk HR stale lọt qua cleaning — kiểm tra Rule 1 |
| `refund_no_stale_14d_window` | `--no-refund-fix` bị bật, hoặc rule fix bị xóa |
| `access_control_sop_present` | `access_control_sop` không còn trong allowlist hoặc data rỗng |
| `no_corrupted_content_marker` | Rule 2 bị xóa, hoặc data mới có `!!!` prefix |
| `effective_date_iso_yyyy_mm_dd` | Format ngày mới trong data source không được parser xử lý |

### Mitigation

```bash
# Giữ nguyên ChromaDB cũ (vẫn phục vụ được từ run trước)
# Fix rule / data → rerun
python etl_pipeline.py run
```

### Prevention
- Không xóa cleaning rules đã có — mỗi rule có expectation tương ứng bảo vệ
- Khi thêm data source mới: cập nhật `ALLOWED_DOC_IDS` + thêm expectation

---

## Incident 3: Freshness WARN/FAIL

### Symptom
```
freshness_check=WARN {"reason": "no_timestamp_in_manifest"}
freshness_check=FAIL {"reason": "freshness_sla_exceeded", "age_hours": 1554}
```

### Detection
Log cuối mỗi pipeline run, hoặc chạy:
```bash
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run_id>.json
```

### Giải thích PASS / WARN / FAIL

| Kết quả | Điều kiện | Hành động |
|---------|-----------|-----------|
| `PASS` | `age_hours <= sla_hours (24)` | Không cần làm gì |
| `WARN` | Không parse được `latest_exported_at` | Kiểm tra format timestamp trong CSV nguồn (phải là `YYYY-MM-DDTHH:MM:SS` hoặc `YYYY/MM/DDTHH:MM:SS`) |
| `FAIL` | `age_hours > sla_hours` | Trigger lại pipeline với data mới từ upstream |

### Diagnosis

```bash
# Đọc manifest để xem exported_at
python -c "import json; d=json.load(open('artifacts/manifests/manifest_<run_id>.json')); print(d['latest_exported_at'])"
```

### Mitigation

```bash
# Lấy data mới từ nguồn và rerun
python etl_pipeline.py run --raw data/raw/policy_export_new.csv
```

### Prevention
- Đảm bảo hệ thống export upstream dùng format `YYYY-MM-DDTHH:MM:SS`
- Cài scheduler chạy pipeline định kỳ (cron / Airflow) để freshness không vượt SLA

---

## Quick reference

```bash
# Chạy pipeline đầy đủ
python etl_pipeline.py run

# Kiểm tra freshness manifest cụ thể
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run_id>.json

# Đánh giá retrieval
python eval_retrieval.py --questions data/grading_questions.json --out artifacts/eval/check.csv

# Grading chính thức
python grading_run.py --out artifacts/eval/grading_run.jsonl

# Inject corruption (chỉ demo Sprint 3)
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
```
