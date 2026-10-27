# Paperless Cheque OCR API — Project Knowledge

Full working knowledge of the Paperless Cheque project: what it does, how the code is
organised, every business rule, how to run and test it, and the decisions taken so far.
Use this as the starting context for any work on the codebase.

- **Main repo (UAT):** https://github.com/shithindev-1/paperless_cheque_dev
- **Daily backup repo:** https://github.com/shithindev-1/papperless-backup
- **This knowledge repo:** https://github.com/shithindev-1/paperless-knowledge
- **Local path:** `D:\softspace\Desktop\Paperless_cheque`
- **Owner:** Shithin Devadasan (Softspace)

---

## 1. What the API does

A FastAPI service for UAE property-leasing cheques.

1. **Extract** (`POST /api/v1/cheques/extract`): upload cheque images/PDFs, get back for
   every cheque: date, amount, installment number, service, overlap error.
2. **Match** (`POST /api/v1/cheques/match`): send cheque file URLs (Azure Blob) plus the
   expected payment schedule from the backend; the API OCRs every file and marks each
   payment as matched / not matched against a cheque.
3. **Health** (`GET /api/v1/health`): `{"status": "ok", "version": "1.0.0"}` only.

OCR is done by a Google Gemini vision model through the `google-genai` SDK with
structured JSON output. Model name and key come only from `.env`.

---

## 2. Code structure

```
app/
├── main.py                      app factory: CORS, router, error handlers, docs toggle,
│                                Swagger file-picker patch (format: binary)
├── api/
│   ├── deps.py                  get_cheque_extractor() (lru_cache singleton)
│   ├── errors.py                every error -> {"error": "<message>"}; 422 message differs for /match
│   └── v1/router.py, endpoints/{health,extraction,matching}.py   thin HTTP layer only
├── core/
│   ├── config.py                Settings (pydantic-settings, .env). gemini_api_key + gemini_model REQUIRED
│   ├── exceptions.py            AppError, InvalidRequestError(400), FileValidationError(400),
│   │                            FileDownloadError(400), ExtractionError(502); `.message` is client-safe
│   ├── logging.py               DailyFileHandler, DualTimezoneFormatter, log_result()
│   └── timezones.py             UAE (+4) and IST (+5:30)
├── prompts/cheque_extraction.py SYSTEM_INSTRUCTION + EXTRACTION_PROMPT
├── schemas/
│   ├── cheque.py                ChequeEntity (also the model's response schema), ServiceType, ChequeExtraction
│   ├── matching.py              RentPayment (extra="allow"), MatchRequest
│   └── response.py              TokenUsage (+ .total()), FileResult, BatchResponse
├── services/
│   ├── extraction_service.py    use case for /extract (validate uploads, extract, log)
│   ├── matching_service.py      use case for /match (load -> extract -> match -> log)
│   ├── cheque_extractor.py      ChequeExtractor: per-file model call, semaphore concurrency,
│   │                            infer_installments()
│   ├── payment_matcher.py       pure matching rules + OcrResponse messages
│   ├── file_loader.py           Azure https download (host allow-list, size cap) / local files
│   └── gemini/                  client.py (genai.Client), generation.py (config, retries, usage)
└── utils/
    ├── dates.py                 parse_date(): lenient, day-first
    └── files.py                 magic-byte sniffing, validate_file(), read_and_validate()
tests/                           pytest, no model/network calls
postman/Paperless_Cheque.postman_collection.json
```

Layering rule: **endpoints -> services (use cases) -> pure logic / utils**. Endpoints hold
no logic. `payment_matcher.py` and `utils/` have no I/O so they are unit-testable.

---

## 3. Business rules

### 3.1 Extraction
- Fields per cheque: `cheque_date`, `cheque_amount`, `installment_number`, `service`, `error`.
- `cheque_date` always returned as **DD-MM-YYYY** (validator normalises any format via `parse_date`).
- `cheque_amount`: number only; if figures and words disagree the prompt says use words.
- `installment_number`: from notes like "1st Inst", "Inst 2/4", "2 of 4". If none written,
  `infer_installments()` fills it by **date order within the same service** in the request
  (only groups of 2+ dated cheques; never overwrites a written number).
- `service`: one of `Property Rent`, `Security Deposit`, `VAT on Rent`, `Handling Fee`, or null.
- `error` (per cheque): null, or a message when the cheque is **physically overlapped**
  (partly covered by another cheque/paper/object). Visible fields still extracted,
  covered fields null.
- Per-file `error` (on FileResult) is separate: whole file failed (bad type, too big, model error).
- Allowed types: JPG, PNG, WEBP, HEIC/HEIF, PDF. Max 15 MB, max 20 files, 5 concurrent model calls.

### 3.2 Matching (`/cheques/match`)
Request: `{"file_urls": [...], "payments": [ {RentPaymentId, Installment, ServiceId,
ServiceDescription, Amount, DueDate, ...} ]}`. Response: the **payment list only**, each item
with three keys filled; any extra keys the caller sends are returned unchanged.

- **Match = same Amount (±0.01) AND same date.** Installment and service are NOT checked —
  the backend always sends correct installments (installment check was built then removed
  on user request).
- Payments are processed in request order; **each cheque is used at most once**.
- `MatchedWithOcr`: true / false.
- `MatchedOcrFile`: matched -> URL of the matching file; unmatched -> URL of the **closest
  cheque's file** (the one quoted in OcrResponse); null only when there is no cheque to point
  to (no cheques read, all files failed, all cheques already used). URL returned exactly as sent.
- `OcrResponse`: `"Matched"`, otherwise a short, user-friendly reason showing **only the value
  read from the cheque** (never the expected value, no file name):
  - amount matches, date differs -> `Date doesn't match. Cheque date is 01-10-2026.`
  - date matches, amount differs -> `Amount doesn't match. Cheque amount is 5,000.`
  - both differ -> `Amount and date don't match. Cheque shows 5,000 dated 01-10-2026.`
  - `No matching cheque. All cheques are already matched to other payments.`
  - `None of the cheque files could be processed.` / `No cheques found in the uploaded files.`
  - appends ` Some cheques are overlapped and could not be fully read.` if any unused cheque is overlapped.
- Closest cheque: same-amount -> nearest date; same-date -> nearest amount; else nearest
  date then amount. Only unused cheques considered.
- Input values for MatchedWithOcr / MatchedOcrFile / OcrResponse are ignored (validator).
- Empty `payments` -> 400 "The payment list is empty."; empty `file_urls` -> 400.
- `DueDate` that cannot be parsed -> 422.

### 3.3 Dates (`utils/dates.py`)
`parse_date()` accepts ISO (`2026-11-05`, `...T00:00:00`, `...Z`, `20261105`), numeric
(`05/11/2026`, `05-11-2026`, `05.11.2026`, `5/11/26`, `2026/11/05`), month names
(`5 Nov 2026`, `5-November-2026`, `Nov 5, 2026`, `5th Nov 2026`), trailing times, and .NET
`/Date(ms)/`. **Ambiguous numeric dates are day-first (UAE)**; month-first only when
day-first is impossible (`11/25/2026`). Returns None if unreadable.

### 3.4 File loading (`services/file_loader.py`)
- `https://` URLs: host must match `ALLOWED_FILE_HOSTS` (default `.blob.core.windows.net`),
  http rejected, no redirects, streamed with size cap.
- Anything else = local file name, **only when `APP_ENV=local` and `LOCAL_FILES_DIR` set**;
  must resolve inside that folder (path traversal blocked). UAT/prod: "Only file URLs are accepted."
- `redact()` strips query strings (SAS tokens) before logging.

---

## 4. Configuration (`.env`)

Required: `GEMINI_API_KEY`, `GEMINI_MODEL` (no defaults in code). Current local model:
`gemini-3.8-flash` (in local `.env` only, never committed).

| Key | Notes |
|---|---|
| `APP_ENV` | local / uat / prod (default local) |
| `DOCS_ENABLED` | Swagger/ReDoc/openapi.json. **Keep ON in UAT** (user decision) |
| `GEMINI_TEMPERATURE` 0.0, `TOP_P` 0.95, `TOP_K` 40, `MAX_OUTPUT_TOKENS` 4096, `SEED` | sampling |
| `GEMINI_THINKING_BUDGET` (2.5 models) / `GEMINI_THINKING_LEVEL` (3+ models) | set one. With gemini-3.8-flash some thinking tokens appear even with budget 0 |
| `GEMINI_TIMEOUT_SECONDS` 120, `GEMINI_MAX_RETRIES` 2 | retries on 408/429/5xx with backoff |
| `MAX_FILES_PER_REQUEST` 20, `MAX_FILE_SIZE_MB` 15, `MAX_CONCURRENT_REQUESTS` 5 | limits |
| `ALLOWED_FILE_HOSTS`, `DOWNLOAD_TIMEOUT_SECONDS` 60 | /match downloads |
| `LOCAL_FILES_DIR` | local testing, e.g. `D:\softspace\Desktop\Paperless_cheque\samples` |
| `CORS_ORIGINS`, `LOG_LEVEL`, `LOGS_DIR` | |

---

## 5. Logging

- `logs/app/YYYY-MM-DD.log`: every line `"<UAE time> UAE | <IST time> IST | LEVEL | logger | msg"`.
  Per file: cheques found, tokens (prompt/output/thinking/total), ms. Per payment (match):
  matched, file, OcrResponse. Summary line: matched count, files ok, total tokens, total ms.
- `logs/results/YYYY-MM-DD.jsonl`: one JSON per request with `timestamp_uae`, `timestamp_ist`,
  `request_id`, `type` (extract/match), `model`, totals, `usage`, `processing_time_ms`, full results.
- Files named by UAE date, roll at UAE midnight. Model name appears in logs only (logs never committed).

---

## 6. Running locally (Windows)

```powershell
cd D:\softspace\Desktop\Paperless_cheque
.venv\Scripts\activate
python -m uvicorn app.main:app --reload --port 8080
```

Machine quirks (important):
- `python` is NOT on PATH -> use `py -3.12` to create venvs. Python 3.12 at
  `C:\Users\Lenovo\AppData\Local\Programs\Python\Python312\`.
- **Windows Application Control blocks pip-generated `.exe` launchers** (`uvicorn.exe`,
  `pip.exe`). Always use `python -m uvicorn`, `python -m pip`, `python -m pytest`.
- Project was moved from OneDrive; the venv was recreated. If "No pyvenv.cfg" or
  "Unable to create process" appears, recreate `.venv`.
- Use port **8080** (not 8000). Use 8081 for throwaway test servers.

Tests: `python -m pip install -r requirements-dev.txt` then `python -m pytest`.

---

## 7. Postman

Collection `postman/Paperless_Cheque.postman_collection.json`, variable
`baseUrl = http://localhost:8080/api/v1`. Requests: Health, Extract single/multiple,
Match - cheques vs payments, Match - empty payments (400), two negative extract tests.
Import with Ctrl+O or drag-and-drop. User tests mainly via Postman and Swagger (`/docs`;
multiple files via "Add string item").

Sample data (local only, gitignored): `samples/cheque_1..4.pdf` and `samples/test_match_request.json`.
OCR of samples: cheque_1 10000 @ 27-09-2024, cheque_2 5333 @ 05-08-2026,
cheque_3 5000 @ 01-10-2026, cheque_4 6667 @ 05-03-2027 (service null on all).

---

## 8. Repos & workflow

- **Main repo** `paperless_cheque_dev` (remote `origin`): what ships to UAT. User deploys
  UAT manually. Must stay clean and professional.
- **Backup repo** `papperless-backup` (remote `backup`): daily push of the same code.
  Daily: `git add -A; git commit -m "<summary>"; git push origin main; git push backup main`.
- **Knowledge repo** `paperless-knowledge` (this file), local folder
  `D:\softspace\Desktop\paperless-knowledge`. Update after every change.

Rules for main + backup repos:
- **No mention of Claude anywhere**: no CLAUDE.md, no `Co-Authored-By: Claude`, no
  "Generated with Claude Code" in commits/PRs.
- **No secrets or model details**: `.env`, `logs/`, `samples/`, `.venv/` are gitignored;
  `.env.example` has placeholders only; `/health` shows no model; Swagger and error
  messages never mention the model/provider. Scan staged files before every commit.
- Git identity (repo-local): Shithin Devadasan <shithin.devadasan@softspace.ae>.

---

## 9. Working preferences

- **Ask before running tests, starting servers, or editing files** unless the user asked
  for that action. Explicit requests ("make it", "push", "test it") count as permission
  for that task only.
- Keep answers short; the user likes "short title + one line description" summaries.
- Response/message wording should be short and friendly for non-technical users.

---

## 10. Change history

| Date | Change |
|---|---|
| 2026-10-20 | Postman collection; venv recreated after move; port 8080; Swagger multi-upload guidance |
| 2026-10-20 | Dates returned as DD-MM-YYYY |
| 2026-10-22 | Installment inferred from date order (installment_source added then removed) |
| 2026-10-22 | Per-cheque `error` for overlapped cheques |
| 2026-10-23 | `/cheques/match` endpoint (Azure URLs + payment list), local-folder testing, match logging (UAE/IST, tokens) |
| 2026-10-23 | Any-format dates; `Message` -> `OcrResponse` ("Matched" or reason) |
| 2026-10-23 | Installment check added, then removed on 2026-10-27 (backend sends correct installments) |
| 2026-10-27 | OcrResponse shows only the cheque's value, short and friendly |
| 2026-10-27 | Professional restructure (thin endpoints, use-case services, file_loader, APP_ENV, DOCS_ENABLED, tests, no leaks); `/health` no longer shows model |
| 2026-10-27 | `MatchedOcrFile` also set for unmatched payments (closest cheque's file) |
| 2026-10-27 | Pushed to main + backup repos; knowledge repo created |

## 11. Open items
- Unit tests and the restructured app have **not yet been run** (awaiting user permission).
- Re-test `/match` in Postman after the restructure.
- Deployment files (Docker / App Service) not added; user deploys UAT manually.
