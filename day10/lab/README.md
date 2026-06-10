# Lab Day 10 — ETL Pipeline & Data Observability

## Mot lenh chay ca pipeline

```bash
python etl_pipeline.py run
```

Ket qua mong doi:
```
raw_records=247 | cleaned_records=42 | quarantine_records=205
expectation[...] OK (halt/warn) x8
embed_upsert count=42 collection=day10_kb
PIPELINE_OK
```

---

## Cai dat

```bash
pip install -r requirements.txt
cp .env.example .env
python etl_pipeline.py run
```

---

## Cac lenh thuong dung

```bash
# Chay pipeline chuan
python etl_pipeline.py run

# Kiem tra freshness
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run_id>.json

# Danh gia retrieval
python eval_retrieval.py --questions data/grading_questions.json --out artifacts/eval/check.csv

# Grading chinh thuc (10 cau)
python grading_run.py --out artifacts/eval/grading_run.jsonl

# Inject corruption -- chi dung Sprint 3 demo
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
```

---

## Cau truc thu muc

```
day10/lab/
├── etl_pipeline.py          # Entrypoint chinh
├── transform/
│   └── cleaning_rules.py    # 9 cleaning rules + ALLOWED_DOC_IDS
├── quality/
│   └── expectations.py      # 8 expectations (E1-E8)
├── monitoring/
│   └── freshness_check.py   # SLA freshness check
├── data/
│   ├── raw/                 # CSV dirty input
│   ├── docs/                # Source documents
│   └── grading_questions.json
├── artifacts/
│   ├── cleaned/             # Cleaned CSV sau moi run
│   ├── quarantine/          # Rows bi loai + reason
│   ├── manifests/           # Pipeline lineage JSON
│   ├── logs/                # Run logs
│   └── eval/                # Retrieval eval results
├── docs/
│   ├── pipeline_architecture.md
│   ├── data_contract.md
│   └── runbook.md
└── reports/
    ├── group_report.md
    ├── quality_report.md
    └── individual/
```

---

## Exit codes

| Code | Nghia |
|------|-------|
| 0 | PIPELINE_OK |
| 1 | File raw khong ton tai |
| 2 | Expectation halt FAIL |
| 3 | Embed that bai |

---

## Docs

- [Pipeline architecture](docs/pipeline_architecture.md)
- [Data contract](docs/data_contract.md)
- [Runbook](docs/runbook.md)
- [Group report](reports/group_report.md)
- [Quality report](reports/quality_report.md)
