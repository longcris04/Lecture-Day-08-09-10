# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| `policy_refund_v4` | CSV export từ DB nội bộ | Nội dung stale "14 ngày làm việc" bị inject; chunk bắt đầu `!!!` | E3 `refund_no_stale_14d_window`; E8 `no_corrupted_content_marker` |
| `hr_leave_policy` | CSV export từ DB nội bộ | Bản HR 2025 được export lại với ngày mới (conflict version "10 ngày phép năm") | E6 `hr_leave_no_stale_10d_annual`; rule `stale_hr_policy_effective_date` |
| `sla_p1_2026` | CSV export từ hệ thống SLA | Doc_id không khớp allowlist nếu version mới được đặt tên khác | E1 `min_one_row`; quarantine reason `unknown_doc_id` |
| `it_helpdesk_faq` | CSV export từ hệ thống helpdesk | Chunk ngắn, effective_date sai format (DD/MM/YYYY) | E4 `chunk_min_length_8`; E5 `effective_date_iso_yyyy_mm_dd` |
| `access_control_sop` | CSV export từ IT Security | Bị thiếu khỏi ALLOWED_DOC_IDS ban đầu → toàn bộ quarantine nhầm | E7 `access_control_sop_present`; quarantine reason `unknown_doc_id` |

**Doc_id trong raw CSV nhưng KHÔNG thuộc allowlist (bị quarantine toàn bộ):**

| Doc_id | Số record | Lý do không cho phép |
|--------|-----------|----------------------|
| `data_privacy_guideline` | 29 | Chưa có trong catalog Lab Day 10 |
| `security_policy` | 18 | Chưa có trong catalog Lab Day 10 |
| `legacy_catalog_xyz_zzz` | 31 | Doc legacy, không còn hiệu lực |
| `invalid_doc_*` (25 loại) | 25 | Tên doc_id ngẫu nhiên / lỗi export |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| `chunk_id` | string | Có | SHA-256 hash của `doc_id\|chunk_text\|seq` — stable, idempotent |
| `doc_id` | string | Có | Phải thuộc `ALLOWED_DOC_IDS` trong `cleaning_rules.py` |
| `chunk_text` | string | Có | Tối thiểu 8 ký tự; không bắt đầu `!!!`; không chứa câu lặp >= 3 lần |
| `effective_date` | date (YYYY-MM-DD) | Có | Chuẩn hoá từ DD/MM/YYYY nếu cần; reject nếu không parse được |
| `exported_at` | datetime | Không | Dùng cho freshness check SLA 24h |

---

## 3. Quy tắc quarantine vs drop

| Reason | Nguồn gốc | Hành động | Ai review |
|--------|-----------|-----------|-----------|
| `unknown_doc_id` | Doc_id không thuộc allowlist | Quarantine CSV | Data engineer cập nhật `ALLOWED_DOC_IDS` |
| `missing_effective_date` | Ô effective_date rỗng | Quarantine CSV | Data owner bổ sung ngày |
| `invalid_effective_date_format` | Format không parse được | Quarantine CSV | Data engineer mở rộng parser |
| `stale_hr_policy_effective_date` | HR policy effective_date < 2026-01-01 | Quarantine CSV | HR team xác nhận version hiện hành |
| `missing_chunk_text` | chunk_text rỗng | Quarantine CSV | Data owner bổ sung nội dung |
| `stale_hr_content_marker` | HR chunk chứa "10 ngày phép năm" | Quarantine CSV | HR team cập nhật nội dung lên 12 ngày |
| `corrupted_content_marker` | chunk_text bắt đầu `!!!` | Quarantine CSV | Kiểm tra hệ thống export, loại bỏ marker |
| `excessive_repetition` | Cùng câu lặp >= 3 lần | Quarantine CSV | Data owner deduplicate nội dung nguồn |
| `duplicate_chunk_text` | Nội dung trùng với chunk đã xử lý | Quarantine CSV | Không cần review — bản đầu được giữ lại |

**Không có record nào bị xóa vĩnh viễn** — tất cả đều có trong `artifacts/quarantine/` để audit.

---

## 4. Phiên bản & canonical

| Tài liệu | Version hiện hành | Source of truth | Ghi chú |
|---------|-------------------|----------------|---------|
| Chính sách hoàn tiền | `policy_refund_v4` | `data/docs/policy_refund_v4.txt` | Cửa sổ đúng: **7 ngày làm việc** (không phải 14) |
| Chính sách nghỉ phép | `hr_leave_policy` | `data/docs/hr_leave_policy.txt` | Số ngày đúng: **12 ngày phép năm** (không phải 10) |
| SLA Priority 1 | `sla_p1_2026` | `data/docs/sla_p1_2026.txt` | Hiệu lực từ 2026 |
| FAQ Helpdesk | `it_helpdesk_faq` | `data/docs/it_helpdesk_faq.txt` | — |
| Access Control SOP | `access_control_sop` | `data/docs/access_control_sop.txt` | Level 4: phê duyệt IT Manager + CISO |

---

## 5. Sprint 1 — Kết quả phân tích ban đầu

### Doc_id unique trong raw CSV (247 records)

| Doc_id | Records | Trong allowlist? |
|--------|---------|-----------------|
| `hr_leave_policy` | 40 | Có |
| `policy_refund_v4` | 33 | Có |
| `sla_p1_2026` | 31 | Có |
| `legacy_catalog_xyz_zzz` | 31 | Không — quarantine |
| `data_privacy_guideline` | 29 | Không — quarantine |
| `it_helpdesk_faq` | 26 | Có |
| `security_policy` | 18 | Không — quarantine |
| `access_control_sop` | 8 | **Thiếu → đã thêm vào allowlist** |
| `invalid_doc_*` (25 loại) | 25 | Không — quarantine |

### Nguyên nhân pipeline HALT ban đầu

```
expectation[hr_leave_no_stale_10d_annual] FAIL (halt) :: violations=2
```

2 chunk `hr_leave_policy` chứa `"10 ngày phép năm"` với `effective_date >= 2026` lọt qua date filter → cần thêm content-based rule.

### Sửa đổi trong Sprint 1

**`transform/cleaning_rules.py`:**
- Thêm `access_control_sop` vào `ALLOWED_DOC_IDS`
- Rule 1: `stale_hr_content_marker` — quarantine HR chunk chứa "10 ngày phép năm"
- Rule 2: `corrupted_content_marker` — quarantine chunk bắt đầu `!!!`
- Rule 3: `excessive_repetition` — quarantine chunk có câu lặp >= 3 lần

**`quality/expectations.py`:**
- E7: `access_control_sop_present` (halt)
- E8: `no_corrupted_content_marker` (halt)

### Kết quả sau fix

```
raw_records=247 | cleaned_records=42 | quarantine_records=205
Tất cả 8 expectations PASS → PIPELINE_OK (exit 0)
```
