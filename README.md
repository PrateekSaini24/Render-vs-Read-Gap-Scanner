# Gap Scanner — Render-vs-Read Document Firewall

Detects text that **machines read but humans don't see** inside PDF, DOCX, and HTML files — hidden prompt injection in resumes, invoices, and contracts aimed at LLM/RAG pipelines — and returns a calibrated risk score, highlighted evidence, and sanitized text that's safe to ingest.

## Why

LLM and RAG pipelines often ingest untrusted documents. An attacker can hide instructions in a PDF using white-on-white text, tiny fonts, off-page placement, zero-width Unicode, homoglyphs, or metadata — invisible to a human reviewer but fully readable by a parser. Gap Scanner finds the gap between what a human sees and what a machine reads, and flags or strips it.

## Quick start

```bash
sudo apt install tesseract-ocr poppler-utils   # macOS: brew install tesseract poppler
pip install -r requirements.txt
python text_clf.py && python app.py            # UI: http://localhost:5000
```

Batch / CLI usage:

```bash
python cli.py samples/ --csv out.csv --sarif out.sarif
# exits with code 2 if any file is scored MALICIOUS
```

Run the test suite:

```bash
make test
```

### Windows

1. Python 3.10+ from python.org (tick "Add to PATH").
2. Tesseract: UB-Mannheim installer (default path auto-detected, or `set TESSERACT_CMD=...\tesseract.exe`).
3. Poppler: unzip `oschwartz10612/poppler-windows`, then `set POPPLER_PATH=C:\poppler\Library\bin`.
4. ```
   python -m venv venv && venv\Scripts\activate
   pip install -r requirements.txt
   set PYTHONUTF8=1
   python text_clf.py && python app.py
   ```

## Detection coverage

| Layer | Techniques |
|---|---|
| PDF render-vs-read | white/low-contrast text, tiny font, off-page, covered by shape, render mode 3, overlapping/glyph-mismatch text |
| PDF structure | metadata/annotation/form-value injection, embedded files, JavaScript, default-off layers, XFA, incremental-update recovery (deleted content) |
| Unicode | zero-width/tag characters, homoglyph (Cyrillic/Greek) confusables |
| Images (`deep` mode) | near-invisible text inside embedded images (contrast-amplified OCR diff) |
| DOCX / HTML | vanish/white/tiny text runs, comments, metadata; `display:none`, off-screen, zero-font, opacity 0, white text, alt/meta text |
| Text content | regex + trained TF-IDF/logistic-regression classifier (optional DeBERTa via `INJ_MODEL`); document-level scoring catches payloads split across lines/pages |
| Scoring | calibrated GradientBoosting model (11 features) with a per-feature explanation |

## Platform / API

- `POST /api/scan`, async `/api/jobs` (+ poll), `/api/sanitize`, `/api/feedback` (allow/deny), `/api/drift` (PSI), `/api/audit`, `/api/stats`, `/metrics`, `/health`
- Multi-tenant API keys with per-tenant thresholds:
  ```bash
  export GAP_KEYS='{"k1":{"tenant":"acme","suspicious":25,"malicious":60}}'
  # or a single key: export GAP_API_KEY=...
  ```
- Isolated scan subprocess with timeout, input guards, rate limiting, security headers, audit log, Prometheus metrics (`monitoring/grafana_dashboard.json`)
- Docker (non-root, with healthcheck), GitHub Actions CI, reusable GitHub Action (`.github/actions/gap-scan`) that emits SARIF for code scanning
- Integrations: `integrations/safe_loader.py` (a RAG document loader with quarantine), `mcp_server.py` (an MCP tool, `safe_read_document`, for AI agents)

## ML & evaluation workflow

```bash
make data train         # dataset.py → train.py: 5-fold CV, bootstrap CIs, model.pkl, report.md
make redteam             # adaptive attacker; --harvest adds evaders back into training
make compare              # vs. a naive text-layer baseline (appends to report.md)
make eval-adv               # evasion + tricky-benign robustness
make real-eval DIR=/path/to/your/clean/pdfs  # false-positive rate on real PDFs + recall after injecting attacks
make retrain-feedback            # active learning from analyst feedback
```

`finetune_deberta.py` and `vlm_check.py` are provided but untested in the build sandbox (they need `torch` / an API key).

## Project layout

| Path | Purpose |
|---|---|
| `app.py` | Flask web UI + API |
| `cli.py` | Batch / CLI scanning |
| `scanner.py` | Core detection engine |
| `text_clf.py`, `train.py`, `model.pkl`, `textclf.pkl` | ML classifier training + artifacts |
| `redteam.py`, `eval_adversarial.py`, `real_eval.py` | Evaluation and adversarial testing |
| `integrations/safe_loader.py` | RAG-pipeline loader with quarantine |
| `mcp_server.py` | MCP tool for AI agents |
| `samples/` | Clean and adversarial sample documents |
| `tests/` | Test suite |
| `docs/ARCHITECTURE.md`, `docs/WRITEUP.md`, `docs/BENCHMARK_CARD.md` | Design docs, write-up, benchmark card |

## Honest limitations

Training data is synthetic, so cross-validation numbers are optimistic — run `real_eval.py` against your own real documents for a trustworthy false-positive rate. Not built: a browser extension, an email gateway, billing, or a Postgres/Celery backend (SQLite + a thread pool are used instead). VLM-based image-injection reading is a hook only, and glyph-remap fonts beyond the overlap/mismatch check aren't covered.

## License

Add a license of your choice (e.g. MIT) before making this repository public.
