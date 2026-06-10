# Kiến trúc pipeline — Lab Day 10

**Nhóm:** _______________  
**Cập nhật:** 2026-06-11

---

## 1. Sơ đồ luồng

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ETL PIPELINE — DAY 10                                │
└─────────────────────────────────────────────────────────────────────────────┘

data/raw/
policy_export_dirty.csv
        │
        ▼
┌──────────────┐   run_id, raw_records
│   INGEST     │──────────────────────────► artifacts/logs/run_<run_id>.log
│ load_raw_csv │
└──────┬───────┘
       │ 247 rows
       ▼
┌──────────────────┐   quarantine_records
│   TRANSFORM      │──────────────────────► artifacts/quarantine/quarantine_<run_id>.csv
│  clean_rows()    │   (205 rows, ghi reason)
│                  │
│  Rules (9):      │
│  - allowlist     │
│  - date parse    │
│  - stale hr date │
│  - empty text    │
│  - stale hr txt  │ ← Rule 1 (mới)
│  - corrupt !!!   │ ← Rule 2 (mới)
│  - excessive rep │ ← Rule 3 (mới)
│  - dedup         │
│  - refund fix    │
└──────┬───────────┘
       │ 42 rows cleaned
       ▼
┌────────────────────┐   cleaned_records
│   VALIDATE         │──────────────────────► PIPELINE_HALT (exit 2) nếu halt fail
│ run_expectations() │
│                    │
│  E1 min_one_row    │ halt
│  E2 no_empty_docid │ halt
│  E3 refund_14d     │ halt
│  E4 min_length_8   │ warn
│  E5 date_iso       │ halt
│  E6 hr_stale_10d   │ halt
│  E7 acsop_present  │ halt ← (mới)
│  E8 no_corrupt_!!! │ halt ← (mới)
└──────┬─────────────┘
       │ all halt PASS
       ▼
┌────────────────────┐
│     EMBED          │──────────────────────► chroma_db/day10_kb
│ cmd_embed_internal │   upsert chunk_id (idempotent)
│                    │   prune old chunk_ids
└──────┬─────────────┘   embed_upsert count=42
       │
       ▼
┌────────────────────┐
│    MANIFEST        │──────────────────────► artifacts/manifests/manifest_<run_id>.json
│  + FRESHNESS CHECK │   PASS / WARN / FAIL
└────────────────────┘

                         ▼
                 PIPELINE_OK (exit 0)
```

---

## 2. Ranh giới trách nhiệm

| Thành phần | Input | Output | File chính |
|------------|-------|--------|------------|
| **Ingest** | `data/raw/policy_export_dirty.csv` | `List[Dict]` 247 rows | `transform/cleaning_rules.py::load_raw_csv` |
| **Transform** | 247 raw rows | `cleaned` (42) + `quarantine` (205) | `transform/cleaning_rules.py::clean_rows` |
| **Quality** | 42 cleaned rows | `List[ExpectationResult]`, `should_halt: bool` | `quality/expectations.py::run_expectations` |
| **Embed** | `artifacts/cleaned/*.csv` | ChromaDB collection `day10_kb` | `etl_pipeline.py::cmd_embed_internal` |
| **Monitor** | `artifacts/manifests/*.json` | `PASS/WARN/FAIL` + age_hours | `monitoring/freshness_check.py` |

---

## 3. Idempotency & rerun

**Cơ chế:** Mỗi chunk có `chunk_id = SHA-256(doc_id|chunk_text|seq)[:16]` — stable, deterministic.

```python
col.upsert(ids=ids, documents=documents, metadatas=metadatas)
```

Upsert theo `chunk_id`: chạy lại 2 lần với cùng data → không tạo duplicate.

**Prune:** Sau mỗi run, xóa `chunk_id` cũ không còn trong cleaned hiện tại:
```
prev_ids - current_ids → col.delete(ids=drop)
```

Đảm bảo ChromaDB = snapshot chính xác của cleaned CSV hiện tại.  
Minh chứng: Sau khi restore từ inject-bad → `embed_prune_removed=2`.

**Rerun an toàn:** `python etl_pipeline.py run` có thể chạy nhiều lần — kết quả cuối luôn nhất quán.

---

## 4. Liên hệ Day 09

| Điểm so sánh | Day 09 | Day 10 |
|-------------|--------|--------|
| Nguồn data | Embed trực tiếp từ `data/docs/*.txt` | Ingest từ CSV export → clean → validate → embed |
| Collection | `day09_kb` (hoặc default) | `day10_kb` |
| Embedding model | `all-MiniLM-L6-v2` | `all-MiniLM-L6-v2` (giống) |
| Corpus docs | `data/docs/` | `data/docs/` (giống) |

Day 10 bổ sung lớp ETL trước embedding — xử lý dữ liệu dirty từ export hệ thống. Multi-agent Day 09 có thể dùng lại bằng cách trỏ `CHROMA_COLLECTION=day10_kb`.

---

## 5. Rủi ro đã biết

| Rủi ro | Mức độ | Trạng thái |
|--------|--------|-----------|
| `gq_d10_06` — "10 phút escalate" không rank top-5 với query tiếng Việt | Trung bình | Còn tồn tại |
| `gq_d10_01` — residual "14 ngày" chunk trong top-k (eval top-3) | Thấp | Không ảnh hưởng grading (top-5 PASS) |
| Freshness FAIL — dataset lab fixed (exported 2026-04-07) | Thấp (lab) | Bình thường |
| Format `exported_at` dùng `/` thay `-` | Thấp | Đã fix trong `freshness_check.py` |
| `--skip-validate` cho phép bypass halt | Chỉ dùng demo | Documented, không dùng production |
