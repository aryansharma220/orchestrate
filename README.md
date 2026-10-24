# HackerRank Orchestrate - October 2026

## Ghost Data Still Exists

Build an AI agent that audits whether a user's personal data was actually deleted.

A deletion workflow may report **Complete** while data still exists in databases, logs, support attachments, exports, search indexes, archives, or backups. Sync and restore processes may also bring deleted data back.

For each request, the agent must first determine whether deletion was actually requested. It then returns `COMPLETE`, `INCOMPLETE`, or `NOT_APPLICABLE`, reports any remaining data or recreation risk, and produces the ordered remediation and verification steps required to finish an authorized deletion safely.

Read [`problem_statement.md`](./problem_statement.md) for the complete challenge contract.

---

## Implemented Baseline

The Python implementation is a conservative starting point, **not a submission-ready solution**. It parses participant evidence, resolves historical identifiers, applies deletion policy, and emits deterministic audit steps. It does not execute deletions or modify the dataset.

- `code/evidence.py`: JSON/service-export, sandboxed SQLite, text and trace-correlated log ingestion; time-bounded identity/resource links and scoped snapshot absence evidence.
- `code/audit.py`: request-intent baseline, policy treatment, per-replica state reconciliation and ordered findings/remediation/verification.
- `code/contract.py`: exact output schema, canonical ordering, evidence cutoff and remediation/verification validation.
- `code/documents.py`: local PDF text extraction and Tesseract OCR adapter; word coordinates, confidence checks and explicit business-layout parsers.
- `code/evaluation/main.py`: regression tests and public-sample metrics. Sample answers are used only here, never by the prediction pipeline.

### Setup And Run

Python 3.11 or newer is required. The audit uses deterministic rules, not language models, embeddings, online translation or provider APIs. No API key is needed. Install PyMuPDF in a virtual environment for local PDF extraction:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r code/requirements.txt
.\.venv\Scripts\python.exe code/evaluation/main.py -v
.\.venv\Scripts\python.exe code/main.py --allow-partial
```

On macOS/Linux use `.venv/bin/python` in place of `.venv\Scripts\python.exe`.

Paths default to the repository root, regardless of the working directory. Explicit relative CLI paths are relative to the working directory. Output and report paths cannot be inside the input dataset.

When any case or evidence file remains unresolved, the default command exits with code **2** without writing a final output. `--allow-partial` writes `output.partial.csv`, never a misleading final submission. Provisional predictions may still omit facts from unparsed evidence. See `evaluation/run_report.md` for case-level blockers. Existing output files are not deleted when a later run fails; check the current run report before using any artifact.

New run reports include a UTC run ID, request filename, elapsed time and blocker counts. The matching usage report carries the same run ID. A custom report such as `evaluation/sample_run.md` writes `evaluation/sample_run_usage.md`, preserving the full-run `evaluation/usage_report.md`. Reports without a matching run ID must not be assumed to describe the current implementation.

### Local OCR Setup

The Python package `pytesseract` is only a wrapper: it does not install the Tesseract OCR engine. OpenCV, Pillow and NumPy do not supply that engine either. On Windows, run the following installer command yourself; it may request administrator approval:

```powershell
winget install --id UB-Mannheim.TesseractOCR --exact --source winget
```

Images, scanned PDF pages and embedded images on text-bearing PDF pages need a separately installed Tesseract executable and English (`eng`) traineddata. Install these before running the audit; extraction performs no runtime downloads. On Windows, install Tesseract locally and add it to PATH, or set its executable path:

```powershell
$env:TESSERACT_CMD = 'C:\Program Files\Tesseract-OCR\tesseract.exe'
& $env:TESSERACT_CMD --version
& $env:TESSERACT_CMD --list-langs
```

Validate the environment before attempting OCR:

```powershell
.\.venv\Scripts\python.exe code/main.py --check-environment
```

This read-only command reports the actual Python interpreter, PyMuPDF version, engine path/version and availability of English OCR data. It exits with code 2 when a dependency is missing and does not modify predictions or reports. The adapter also checks standard Windows installation paths when PATH has not refreshed.

On macOS/Linux, install Tesseract with English language data using your system package manager and ensure `tesseract` is on PATH (or set `TESSERACT_CMD`). OCR recognition weights are used only for local text recognition, never language-model inference. The adapter requests TSV word boxes with a 60-second subprocess timeout. Critical words below the provisional 0.85 confidence threshold remain unresolved; this threshold is not yet calibrated across all artifacts.

The current validation environment has Python 3.14.7, PyMuPDF 1.28.2 and Tesseract 5.4.0.20240606 with English traineddata. Live OCR passes the marketing-profile and support-ticket screenshot regressions, including label-aligned owner identity. Two supplied PDF layouts are verified: the delivery correction attachment and security compliance journal. Account, search, export, backup, security and other screenshot layouts remain unsupported or confidence-blocked; their identifiers are not silently corrected or accepted.

Raster OCR uses 2x scaling and maps word boxes back to source-image coordinates. PDFs with native text also OCR their embedded image regions. PDF boxes use unrotated page coordinates for both native and raster text. Eleven focused document tests pass, including real marketing/support screenshots, a rotated mixed-content PDF and scanned-page coordinate conversion. A full-suite completion summary is still unverified.

### Evaluation And Limitations

```powershell
.\.venv\Scripts\python.exe code/main.py --requests dataset/sample_deletion_requests.csv --output evaluation/sample_predictions.csv --allow-partial --report evaluation/sample_run.md
.\.venv\Scripts\python.exe code/evaluation/main.py --predictions evaluation/sample_predictions.partial.csv
```

The latest verified offline sample run after ownership filtering produced 4 of 32 provisional answers: 12.5% status accuracy and 6.25% exact-row accuracy across all samples, counting unresolved cases as failures. It parsed 2,880 observations and left 11 artifacts unparsed. See `evaluation/ownership_filter_sample_run.md` and `evaluation/ownership_filter_sample_evaluation.md`. These are public-sample measurements, not hidden-dataset accuracy.

Security retention now requires a supplied unexpired deadline, and legal retention requires an explicitly active hold. Support messages require explicit shared-context evidence before redaction or deletion; support attachments require sole-owner evidence and an uploader matching the subject. Structured tombstone/analytics/telemetry records are checked for nested identifiers; a redacted tombstone must preserve its stable reference. The complete regression suite passes: 49 tests in 25.655 seconds, including offline integration and OCR. This does not establish completeness of every policy condition or evidence format.

Structured profile responses distinguish valid 404/empty reads from errors or contradictory payloads. Zero-hit search responses establish absence only when their document list is empty; positive responses require the requested resource among the hits. Other hits and the query are retained separately as provenance, not used as that resource's ownership. Unidentified search copies can be discovered through explicit references to an identified account, but foreign-owner, shared-support and future observations are not attached by this reverse traversal.

Identity resolution indexes current and historical email owners before reconstructing their time-bounded aliases. It does not scan unrelated identities for each request, and still rejects shared-email ambiguity and future-only aliases. Authorization tests distinguish "no longer use my account" from "no longer want you to delete my account"; the latter remains unresolved.

The latest completed full diagnostic run is in `evaluation/ownership_filter_full_run.md` with matching usage report `evaluation/ownership_filter_full_run_usage.md` (Run ID `2026-10-24T16:24:00.698216+00:00`). It processed 518 requests in 42.411 seconds and produced 52 provisional answers, with 466 unresolved and 11 unparsed artifacts. Blockers include missing account/tombstone proof (333 cases), unresolved policy conditions (125), ambiguous intent (4) and no attributable evidence (4). These are coverage counts, not hidden-answer accuracy. This partial run is not a submission artifact.

Remaining work includes broader identity/resource linking with ownership guards, unsupported document and screenshot layouts, complete verification coverage, multilingual authorization, condition-aware policy handling, and explicit downstream-before-source dependency validation where canonical sorting is insufficient. Offline intent heuristics are not a production authorization mechanism. A parsed source is not proof of exhaustive system coverage. Complete snapshot absence is limited to declared system scopes or the supplied native full-export format, never arbitrary empty tables. Candidate links from activity do not turn operation success into state proof.

`evaluation/usage_report.md` describes the latest baseline run, not a final submission run. The required final reports must be regenerated after all 518 cases and evidence formats are handled. Do not package the partial output as a completed submission.

---

## Quick Start

Clone the repository and move into the project directory:

```bash
git clone https://github.com/interviewstreet/hackerrank-orchestrate-october26.git
cd hackerrank-orchestrate-october26
```

Start your AI coding tool from the **repository root** so it automatically reads [`AGENTS.md`](./AGENTS.md) and logs your session turns to `log.txt` in the root directory.

Build your solution in `code/main.py`, or use another language and document its setup and run command clearly in your README.

Your solution must:

- Read the input files from `dataset/`
- Generate one prediction row for every request in `dataset/deletion_requests.csv`
- Write the final predictions to `output.csv` in the repository root

Run the starter Python entry point with:

```bash
python3 code/main.py
```

The blank template at `dataset/output.csv` is provided as a reference. Your final generated file for submission must be the root-level `output.csv`.


## Dataset

All participant-facing files are in `dataset/`:

| File | Contents |
| --- | --- |
| `deletion_requests.csv` | 518 requests |
| `sample_deletion_requests.csv` | 32 sample requests |
| `sample_output.csv` | Correct sample answers |
| `user_identities.csv` | Current user identities |
| `user_audit_log.csv` | Historical identity changes |
| `system_activity.csv` | Activity across services and data processes |
| `deletion_rules.csv` | Required treatment for each data category |
| `evidence_files.csv` | Manifest of supplied evidence files |
| `output.csv` | Blank output template |

Evidence is stored under `dataset/evidence/` and may include PDFs, images, logs, JSON, plain text, or SQL snapshot exports. SQL captures may contain a primary database, regional replicas, a reporting replica, or a failover replica so replication drift can be investigated directly. The `.sql` files are readable as text and executable with SQLite.

Only `deletion_requests.csv` requires answers. The other files provide evidence and context.

---

## Output

Create `output.csv` with these columns in this exact order:

```text
case_id,deletion_status,steps_json
```

Produce exactly one row per request. `deletion_status` is `COMPLETE`, `INCOMPLETE`, or `NOT_APPLICABLE`. `steps_json` is an ordered JSON array containing findings, remediation actions, and verification checks. Use an empty array (`[]`) for `NOT_APPLICABLE`. Evidence references may be a `file_id`, `activity_id`, or `audit_id`. Determine the resource state at `audit_at` from the available evidence; reported workflow statuses and operation results are claims. Use `dataset/sample_output.csv` as the format reference.

The final file must contain 518 data rows sorted by `case_id` (plus the header row). The grader parses `steps_json` as JSON, so object-key order and whitespace do not matter; array order does. Order findings, remediation, and verification in that phase order. Within a phase, follow the operation order in the problem statement, then sort by `system_id` and `resource_id`.

---

## Evaluation

`output.csv` will be compared with hidden canonical answers. Scoring considers:

- deletion assessment accuracy;
- current and historical identity resolution;
- discovery of remaining data and recreation risks;
- correct policy application and evidence references; and
- completeness and ordering of remediation and verification steps.

Your `code.zip` must include:

```text
evaluation/evaluation_report.md
evaluation/usage_report.md
```

- `evaluation_report.md` must cover case, row, field, and sequence accuracy, at least three additional evaluation cases created by you, failures found, and improvements made.
- `usage_report.md` must cover the final full-dataset run, including model providers and names, model calls, input and output tokens, total and average tokens per request, and estimated total and per-request cost. Include per-model and overall totals when using multiple models.

**Important:** Place the reports in the root `evaluation/` directory, as required by `AGENTS.md`; the evaluator implementation remains in `code/evaluation/main.py`. Write metrics, custom cases and token/cost breakdowns directly in the Markdown reports, and keep prompts in source code or Markdown. Automated code evaluation ignores raw data files (`.csv`, `.tsv`, `.json`, `.jsonl`, `.sql`, `.log`), so content stored only in those formats will not be visible to the code judge or AI interviewer.

---

## Chat Transcript

This repository includes [`AGENTS.md`](./AGENTS.md), which instructs compatible AI coding tools to append conversation summaries to `log.txt` in the repository root:

| Platform | Path |
| --- | --- |
| macOS / Linux | `<repo root>/log.txt` |
| Windows | `<repo root>\log.txt` |

- Always launch your AI coding agent from the repository root (or explicitly ask it to read `./AGENTS.md` first) so every turn is logged to `log.txt` with the required `User Prompt`, `Agent Response Summary`, `Actions`, and `Context` sections.
- Upload `log.txt` directly as your `chat_transcript` submission on HackerRank.
- Do not include API keys, tokens, credentials, or secrets in prompts, code, configuration, or `log.txt`.

---

## Packaging `code.zip` and Submission

| Item | Description |
| --- | --- |
| `code.zip` | Runnable solution, prompts, configuration, `README.md`, and `evaluation/` reports |
| `output.csv` | One answer row for each of the 518 requests in `dataset/deletion_requests.csv` |
| `chat_transcript` | The root `log.txt` file generated by your coding agent |
| AI interview | Live explanation of your architecture, approach, and evaluation |

You may use any programming language, model, framework, or retrieval approach. Include clear setup and execution instructions in `README.md` inside `code.zip`.

To package `code.zip` from the repository root without bundling the dataset:

**macOS / Linux:**

```bash
zip -r code.zip README.md code evaluation -x "*/__pycache__/*" "*/.venv/*" "*/venv/*" "*/node_modules/*" "*/.git/*" "*/.DS_Store" "evaluation/*.csv"
```

**Windows (PowerShell):**

```powershell
Compress-Archive -Path README.md, code, evaluation -DestinationPath code.zip -Force
```

When packaging `code.zip`:

- Include root `evaluation/` so both required reports appear at the contract's archive paths. Exclude provisional CSV files and diagnostic-only reports from the submission archive.
- Do **not** include `dataset/`, virtual environments (`.venv`, `venv`, `node_modules`), or build caches (`__pycache__`). On Windows, keep `.venv` at the repository root (outside `code/`) and remove any `code/__pycache__` folders before running `Compress-Archive`.
- Do **not** include nested archives (`.zip`, `.tar`, `.gz`, `.tgz`, `.7z`, `.rar`) inside `code.zip` — submissions containing nested archive files will be rejected.

Before submitting, confirm that:

- `output.csv` has the exact header `case_id,deletion_status,steps_json` and contains all 518 data rows sorted by `case_id`;
- every `steps_json` value is a valid JSON array with consecutive step numbers starting at `1`;
- findings cite valid evidence IDs, and `COMPLETE` verification steps cite evidence proving the current state;
- proposed remediation and future verification steps use an empty `evidence_id` (`""`);
- `code.zip` includes your runnable code, `README.md`, `evaluation/evaluation_report.md`, and `evaluation/usage_report.md`;
- `log.txt` follows the `AGENTS.md` turn format and is uploaded to the `chat_transcript` slot; and
- no API keys, credentials, real personal data, or sensitive configuration values are included.
