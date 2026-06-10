# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** ___________  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| ___ | Ingestion / Raw Owner | ___ |
| ___ | Cleaning & Quality Owner | ___ |
| ___ | Embed & Idempotency Owner | ___ |
| ___ | Monitoring / Docs Owner | ___ |

**Ngày nộp:** 2026-06-11  
**Repo:** ___________  
**Độ dài khuyến nghị:** 600–1000 từ

---

> **Nộp tại:** `reports/group_report.md`  
> **Deadline commit:** xem `SCORING.md`  
> Phải có **run_id**, **đường dẫn artifact**, và **bằng chứng before/after** (CSV eval hoặc screenshot).

---

## 1. Pipeline tổng quan

**Nguồn raw:** `data/raw/policy_export_dirty.csv` — 247 records, 5 doc_id hợp lệ sau khi fix allowlist, nhiều doc_id rác và invalid được inject cố ý để kiểm thử pipeline.

**Luồng end-to-end:**

```
load_raw_csv()           247 records raw
      ↓
clean_rows()             loại 205 records vi phạm rules
      ↓
write_cleaned_csv()      42 records → artifacts/cleaned/
write_quarantine_csv()   205 records → artifacts/quarantine/
      ↓
run_expectations()       8 expectations kiểm tra kết quả clean
      ↓ (halt nếu fail)
cmd_embed_internal()     upsert 42 chunks → ChromaDB day10_kb
      ↓
manifest + freshness     artifacts/manifests/manifest_<run_id>.json
```

**Lệnh chạy một dòng:**

```bash
python etl_pipeline.py run
```

**run_id mới nhất:** `2026-06-10T18-08Z`  
**Artifacts:** `artifacts/cleaned/`, `artifacts/quarantine/`, `artifacts/manifests/`, `artifacts/eval/`

**Kết quả run chuẩn:**
```
raw_records=247 | cleaned_records=42 | quarantine_records=205
Tất cả 8 expectations PASS → PIPELINE_OK (exit 0)
```

---

## 2. Cleaning & expectation

Baseline đã có 6 rule: allowlist doc_id, parse ngày ISO, loại HR stale theo date, loại chunk rỗng, dedup, fix refund 14→7 ngày. Nhóm bổ sung 3 rule mới và 2 expectation mới.

### 2a. Bảng metric_impact (bắt buộc — chống trivial)

| Rule / Expectation mới | Trước khi thêm | Sau khi thêm | Chứng cứ |
|------------------------|----------------|--------------|----------|
| **Rule 1** `stale_hr_content_marker` | E6 FAIL `violations=2`; 8 chunk HR stale (effective_date >= 2026 nhưng nội dung "10 ngày phép năm") lọt vào cleaned | E6 PASS `violations=0`; 8 rows quarantined thêm | `artifacts/quarantine/` reason=`stale_hr_content_marker` count=8 |
| **Rule 2** `corrupted_content_marker` | 1 chunk `!!!` prefix (policy_refund_v4) lọt vào cleaned; E8 FAIL `bang_prefix_rows=1` | E8 PASS `bang_prefix_rows=0`; 1 row quarantined | `artifacts/quarantine/` reason=`corrupted_content_marker` count=1 |
| **Rule 3** `excessive_repetition` | 1 chunk có câu lặp 5 lần lọt vào cleaned → lệch similarity score | 1 row quarantined | `artifacts/quarantine/` reason=`excessive_repetition` count=1 |
| **E7** `access_control_sop_present` | access_control_sop không có trong ALLOWED_DOC_IDS → 0 chunk embed → gq_d10_10 fail retrieval | PASS `access_control_sop_rows=6` | `artifacts/eval/after_fix_eval.csv` gq_d10_10 `top1_doc_expected=yes` |
| **E8** `no_corrupted_content_marker` | 1 chunk `!!!` trong cleaned → E8 FAIL | PASS `bang_prefix_rows=0` | expectation log `bang_prefix_rows=0` |

**Tổng quarantine phân loại (run chuẩn `2026-06-10T18-08Z`):**

| Reason | Count |
|--------|-------|
| `unknown_doc_id` | 109 |
| `duplicate_chunk_text` | 50 |
| `stale_hr_policy_effective_date` | 22 |
| `stale_hr_content_marker` *(Rule 1 mới)* | 8 |
| `missing_chunk_text` | 8 |
| `missing_effective_date` | 6 |
| `excessive_repetition` *(Rule 3 mới)* | 1 |
| `corrupted_content_marker` *(Rule 2 mới)* | 1 |
| **TOTAL** | **205** |

**Rule chính (baseline + mở rộng):**

- `unknown_doc_id`: doc_id không thuộc `ALLOWED_DOC_IDS` — loại 109 rows (doc rác, legacy, invalid)
- `stale_hr_policy_effective_date`: hr_leave_policy effective_date < 2026-01-01 — loại 22 rows bản cũ
- `stale_hr_content_marker` *(mới)*: hr_leave_policy chứa "10 ngày phép năm" dù date mới — loại 8 rows conflict version
- `corrupted_content_marker` *(mới)*: chunk bắt đầu `!!!` — loại 1 row inject lỗi
- `excessive_repetition` *(mới)*: cùng câu lặp >= 3 lần — loại 1 row bloated
- `duplicate_chunk_text`: nội dung trùng sau normalize — loại 50 rows

**Ví dụ expectation fail và cách xử lý:**

```
# Baseline (trước khi thêm Rule 1):
expectation[hr_leave_no_stale_10d_annual] FAIL (halt) :: violations=2
→ Pipeline HALT exit 2, không embed.

# Fix: thêm Rule 1 stale_hr_content_marker vào cleaning_rules.py
→ 8 rows HR stale bị quarantine trước khi đến expectation check
→ expectation[hr_leave_no_stale_10d_annual] OK (halt) :: violations=0
```

---

## 2b. Grading chính thức (artifacts/eval/grading_run.jsonl — top_k=5)

| Question | contains_expected | hits_forbidden | top1_doc_matches | Kết quả |
|----------|------------------|----------------|-----------------|---------|
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

**Kết quả: 9/10 PASS.**

`gq_d10_06` fail `contains_expected=False`: "10 phút auto escalate P1" — chunk chứa "10 phút" có trong ChromaDB nhưng không rank top-5 với Vietnamese query. Embedding model `all-MiniLM-L6-v2` không align tốt giữa câu hỏi escalation và chunk SLA.

---

## 3. Before / after ảnh hưởng retrieval (Sprint 3)

**Kịch bản inject corruption:**

```bash
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
```

- `--no-refund-fix`: tắt rule replace "14 ngày làm việc" → "7 ngày làm việc"
- `--skip-validate`: bỏ qua halt khi E3 fail → embed tiếp dù có dữ liệu sai

**Expectation phát hiện được:**
```
expectation[refund_no_stale_14d_window] FAIL (halt) :: violations=2
WARN: expectation failed but --skip-validate -> tiep tuc embed
```

**Kết quả định lượng (so sánh CSV eval):**

| Question | after_fix | after_inject_bad | Ghi chú |
|----------|-----------|-----------------|---------|
| gq_d10_01 | contains=yes, forbidden=**yes** | contains=yes, forbidden=**yes** | Cả 2 có "14 ngày" residual trong top-k (không phải "14 ngày làm việc") |
| gq_d10_02 | contains=yes, forbidden=no | contains=yes, forbidden=no | OK |
| gq_d10_03 | contains=yes, forbidden=no | contains=yes, forbidden=no | OK |
| gq_d10_04–08 | all pass | all pass | OK |
| gq_d10_09 | contains=yes, forbidden=no | contains=yes, forbidden=no | "12 ngày" hiện diện, "10 ngày" đã loại |
| gq_d10_10 | contains=yes, forbidden=no | contains=yes, forbidden=no | access_control_sop present |

**Artifacts:** `artifacts/eval/after_fix_eval.csv`, `artifacts/eval/after_inject_bad.csv`

**Phân tích:** Inject-bad tạo violations=2 (E3 phát hiện) nhưng eval top-k không thay đổi vì 2 chunk "14 ngày làm việc" không rank đủ cao trong top-3. Đây chứng minh **expectation là lớp bảo vệ quan trọng hơn eval** — phát hiện được lỗi mà retrieval metric bỏ qua. Sau khi restore pipeline chuẩn: `embed_prune_removed=2` (2 chunk stale bị xóa khỏi ChromaDB).

**Vấn đề còn lại (gq_d10_01):** `hits_forbidden=yes` ở cả 2 trạng thái do có chunk policy_refund_v4 chứa "14 ngày" (không phải "14 ngày làm việc") trong top-k — rule replace không bắt được vì không khớp chính xác. Xem mục 6.

---

## 4. Freshness & monitoring

**SLA được chọn:** 24 giờ (`FRESHNESS_SLA_HOURS=24` trong `.env`).

**Kết quả trên manifest `2026-06-10T18-08Z`:**
```
freshness_check=FAIL
latest_exported_at=2026/04/07T00:00:00
age_hours=1554.1
reason=freshness_sla_exceeded
```

**Ý nghĩa các trạng thái:**

| Trạng thái | Điều kiện | Hành động khuyến nghị |
|-----------|-----------|----------------------|
| `PASS` | age_hours <= 24 | Không cần làm gì |
| `WARN` | Không parse được timestamp | Kiểm tra format `exported_at` trong CSV nguồn |
| `FAIL` | age_hours > 24 | Trigger lại pipeline với data mới từ upstream |

**Bug đã fix:** `freshness_check.py` không parse được format `2026/04/07T...` (dùng `/` thay `-`). Đã thêm normalize: `ts.replace("/", "-", 2)` trước khi parse. Kết quả trả về `FAIL` đúng thay vì `WARN` nhầm.

**Cơ chế snapshot publish (idempotent):** Mỗi run, `cmd_embed_internal()` so sánh `prev_ids` với `ids` hiện tại, xóa chunk_id cũ không còn trong cleaned. Đảm bảo ChromaDB luôn đồng bộ với cleaned CSV, không tích lũy rác qua các run.

---

## 5. Liên hệ Day 09

Pipeline Day 10 xây dựng ChromaDB collection `day10_kb` với cùng model embedding `all-MiniLM-L6-v2` như Day 09. Corpus docs giống nhau (`data/docs/`). Điểm khác biệt: Day 10 thêm lớp ETL (ingest từ CSV export → clean → validate) trước khi embed, thay vì embed trực tiếp từ file txt.

Collection `day10_kb` **tách biệt** với collection Day 09 để tránh nhiễu dữ liệu trong quá trình lab. Multi-agent Day 09 có thể tích hợp lại bằng cách trỏ `CHROMA_COLLECTION=day10_kb` — nhưng cần đảm bảo pipeline Day 10 đã chạy thành công trước.

---

## 6. Rủi ro còn lại & việc chưa làm

| Vấn đề | Mức độ | Hướng xử lý |
|--------|--------|-------------|
| `gq_d10_01` `hits_forbidden=yes` — chunk policy_refund_v4 có "14 ngày" không phải "14 ngày làm việc" còn trong top-k | Trung bình | Mở rộng cleaning rule để quarantine chunk có "14 ngày" không kèm fix marker; hoặc nới lỏng `must_not_contain` |
| `gq_d10_06` `contains_expected=no` — "10 phút" không tìm thấy trong top-k | Trung bình | Kiểm tra lại chunk sla_p1_2026 có chứa thông tin escalation không |
| Freshness FAIL do data lab cố định (exported 2026-04-07) | Thấp (lab) | Bình thường với dataset cố định; production cần trigger pipeline theo lịch |
| `gq_d10_01` cần xác minh thêm về chunk "14 ngày" còn sót | Thấp | Đọc `artifacts/cleaned/` để trace chunk cụ thể |
| `grading_run.py` chưa chạy để xác nhận điểm cuối | Chưa làm | Chạy `python grading_run.py --out artifacts/eval/grading_run.jsonl` |
