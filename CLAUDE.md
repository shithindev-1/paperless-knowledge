# Paperless Cheque OCR API — Project Knowledge

Full working knowledge of the Paperless Cheque project: what it does, how the code is
organised, every business rule, how to run, test and deploy it, and the decisions taken so far.
Use this as the starting context for any work on the codebase.

- **Main repo (UAT):** https://github.com/shithindev-1/paperless_cheque_dev
- **Daily backup repo:** https://github.com/shithindev-1/papperless-backup
- **This knowledge repo:** https://github.com/shithindev-1/paperless-knowledge
- **Local path:** `D:\softspace\Desktop\Paperless_cheque`
- **Owner:** Shithin Devadasan (Softspace)

---

## 1. What the API does

A FastAPI service for UAE property-leasing cheques.

1. **Extract** (`POST /api/v1/cheques/extract`): upload cheque images/PDFs (multipart `files`),
   get back for every cheque: date, amount, installment number, service, overlap error. No cropping.
2. **Match** (`POST /api/v1/cheques/match`): send cheque file URLs (Azure Blob) plus the
   expected payment schedule from the backend; the API OCRs every file, marks each payment
   as matched / not matched, and crops/rotates the **matched** cheques into `_ocr.pdf` files.
3. **Health** (`GET /api/v1/health`): `{"status": "ok", "version": "1.0.0"}` only. No auth.

OCR is done by a Google Gemini vision model (`google-genai` SDK, structured JSON output).
The same call also returns the cheque layout (page, bounding box, rotation, needs_crop).
Model name lives only in `config/settings.<env>.json` (gitignored); key only in `.env`.

---

## 2. Code structure

```
app/
├── main.py                      app factory: CORS, router, error handlers, docs toggle,
│                                Swagger file-picker patch (format: binary)
├── api/
│   ├── deps.py                  get_cheque_extractor() (lru_cache singleton)
│   ├── security.py              require_api_key: X-API-Key header, hmac.compare_digest
│   ├── errors.py                every error -> {"error": "<message>"}; 422 message differs for /match
│   └── v1/router.py, endpoints/{health,extraction,matching}.py   thin HTTP layer only
│                                (extract + match routers have Depends(require_api_key))
├── core/
│   ├── config.py                Settings: JSON config files + .env + env vars (see §4)
│   ├── exceptions.py            AppError, AuthenticationError(401), InvalidRequestError(400),
│   │                            FileValidationError, FileDownloadError(400), ExtractionError(502)
│   ├── logging.py               DailyFileHandler, DualTimezoneFormatter, log_result()
│   └── timezones.py             UAE (+4) and IST (+5:30)
├── prompts/cheque_extraction.py fields + layout (page, bounding_box, rotation_degrees, needs_crop)
├── schemas/
│   ├── cheque.py                ChequeEntity (public), ChequeOcr (adds layout, never serialised),
│   │                            ServiceType, ChequeExtraction
│   ├── matching.py              RentPayment (extra="allow"), MatchRequest
│   └── response.py              TokenUsage, FileResult (cheque_files internal), BatchResponse
├── services/
│   ├── extraction_service.py    /extract use case
│   ├── matching_service.py      /match use case: load -> extract -> match -> crop matched -> log
│   ├── cheque_extractor.py      model call per file, semaphore concurrency, infer_installments()
│   ├── payment_matcher.py       pure matching rules + OcrResponse; returns list[Match]
│   ├── file_loader.py           blob (connection string) / allowed https / local files; redact()
│   ├── blob_storage.py          BlobStorage: blob_name(url), url_for (optional SAS), download, upload
│   ├── image_processor.py       OpenCV: needs_processing, load_page, crop, rotate, encode_pdf
│   ├── cheque_imaging.py        attach_cheque_files(only=...), processed_name(), save to blob/local
│   └── gemini/                  client.py, generation.py (config, retries, usage)
└── utils/
    ├── dates.py                 parse_date(): lenient, day-first
    └── files.py                 magic-byte sniffing, validate_file()
config/settings.json             shared defaults (committed)
config/settings.example.json     template for settings.<env>.json (committed)
tests/                           pytest, no model/network calls
postman/Paperless_Cheque.postman_collection.json
```

Layering rule: **endpoints -> services (use cases) -> pure logic / utils**. Endpoints hold
no logic. `payment_matcher.py`, `image_processor.py` and `utils/` have no I/O.

---

## 3. Business rules

### 3.1 Extraction
- Fields per cheque: `cheque_date`, `cheque_amount`, `installment_number`, `service`, `error`.
- `cheque_date` always **DD-MM-YYYY**.
- `cheque_amount`: number only; if figures and words disagree, words win.
- `installment_number`: from notes ("1st Inst", "2 of 4"); if none, `infer_installments()` fills
  by **date order within the same service** (groups of 2+; never overwrites a written number).
- `service`: `Property Rent`, `Security Deposit`, `VAT on Rent`, `Handling Fee`, or null.
- `error` (per cheque): message when the cheque is **physically overlapped**; visible fields
  still extracted, covered fields null.
- Allowed types: JPG, PNG, WEBP, HEIC/HEIF, PDF. Max 15 MB, max 20 files, 5 concurrent model calls.

### 3.2 Matching (`/cheques/match`)
Request: `{"file_urls": [...], "payments": [ {RentPaymentId, Installment, ServiceId,
ServiceDescription, Amount, DueDate, ...} ]}`. Response: the **payment list only**; extra keys
the caller sends are returned unchanged. Four keys filled:

| Key | Matched | Not matched |
|---|---|---|
| `MatchedWithOcr` | `true` | `false` |
| `MatchedOcrFileURL` | original URL of the cheque's file | original URL of the **closest** cheque's file; null if none |
| `CroppedOCRFileURL` | `..._ocr.pdf` URL if cropped/rotated, else null | **always null** (never processed) |
| `OcrResponse` | `"Matched"` | short reason (below) |

- **Match = same Amount (±0.01) AND same date.** Installment and service NOT checked
  (backend sends correct installments; check was built then removed).
- Payments in request order; **each cheque used at most once**.
- `OcrResponse` reasons show **only the cheque's value**, never the expected one:
  - `Date doesn't match. Cheque date is 01-10-2026.`
  - `Amount doesn't match. Cheque amount is 5,000.`
  - `Amount and date don't match. Cheque shows 5,000 dated 01-10-2026.`
  - `No matching cheque. All cheques are already matched to other payments.`
  - `None of the cheque files could be processed.` / `No cheques found in the uploaded files.`
  - appends ` Some cheques are overlapped and could not be fully read.` if an unused cheque is overlapped.
- Closest cheque: same amount -> nearest date; same date -> nearest amount; else nearest date then amount.
- Input values for the four filled keys are ignored. Empty `payments` / `file_urls` -> 400;
  unparseable `DueDate` -> 422; >20 files -> 400.
- Key names history: `MatchedOcrFile` renamed to `MatchedOcrFileURL` and `CroppedOCRFileURL`
  added on 2026-10-28 (breaking change for frontend).

### 3.3 Crop / rotation (matched cheques only)
- Order: extract all -> match -> `_crop_matched()` processes **only matched cheques**.
- A cheque is processed when its file holds **more than one cheque**, or the model says
  `needs_crop`, or rotation ≥ 1°. Otherwise `CroppedOCRFileURL` = null.
- OpenCV: render PDF page (200 dpi) -> crop to box (0-1000 scale, 2% padding) -> quarter turns
  + tilt (white fill) -> saved as a **one-page PDF** (JPEG q92 inside, page sized at render dpi).
- Naming (suffix `processed_file_suffix` = `_ocr`), always `.pdf`, saved **next to the original**:
  - 1 cheque in file: `<name>_ocr.pdf`
  - several cheques: `<name>_ocr_<n>.pdf` (n = cheque's position in the file; only matched ones created)
- Blob: written only if the URL is inside `azure_blob_base_url` (connection string). Other URLs
  are read-only -> original kept. Local mode: written into `local_files_dir`.
- Re-sending the same file overwrites. Any failure -> original kept, logged.
- Limits: overlapped crops may include part of the covering cheque; hidden parts can't be
  recovered; no perspective (keystone) correction; quality depends on model's box.

### 3.4 Dates (`utils/dates.py`)
`parse_date()` accepts ISO, numeric (`05/11/2026`, `5/11/26`, `2026/11/05`), month names
(`5 Nov 2026`, `Nov 5, 2026`, `5th Nov 2026`), trailing times, .NET `/Date(ms)/`.
**Ambiguous numeric dates are day-first (UAE)**. Returns None if unreadable.

### 3.5 File loading
- URL inside `azure_blob_base_url` -> read via `BlobStorage` (connection string; works for private container).
- Other `https://` URLs: host must match `allowed_file_hosts` (`.blob.core.windows.net`), no redirects, size cap.
- Plain names: local files only when `APP_ENV=local` (inside `local_files_dir`, traversal blocked).
- `redact()` strips query strings (SAS) before logging.

### 3.6 Authentication
- `X-API-Key` header on `/extract` and `/match`; `/health` open.
- Keys in `.env` `API_KEYS` (comma-separated -> rotation without downtime). Required when
  `APP_ENV` is uat/prod; optional locally. Wrong/missing -> 401 `{"error": "Invalid or missing API key."}`.
- Guidance given: one key per service in prod (user asked about sharing with leasing OCR key;
  OK for UAT, separate for prod). Call from backend, not browser.

---

## 4. Configuration

Layered (later wins): `config/settings.json` -> `config/settings.<APP_ENV>.json` -> `.env` -> env vars.
`APP_CONFIG_DIR` can override the config folder. Config files are grouped in sections
(app, ocr_model, uploads, file_urls, blob_storage, image_processing); unknown keys or secrets
in a config file stop startup.

- **`.env` (secrets only, gitignored):** `APP_ENV`, `API_KEYS`, `GEMINI_API_KEY`, `AZURE_STORAGE_CONNECTION_STRING`.
- **`settings.<env>.json` (gitignored):** `gemini_model` (`gemini-3.8-flash`), `azure_blob_base_url`,
  `cors_origins`, `docs_enabled`, `local_files_dir`.
- **Shared defaults:** sampling (temp 0, top_p 0.95, top_k 40, 4096 tokens), retries 2, timeout 120 s,
  limits (20 files, 15 MB, 5 concurrent), `azure_sas_expiry_hours` 0, image processing
  (enabled, `_ocr`, padding 2%, 200 dpi, JPEG q92).
- UAT blob base: `https://softspacestorage.blob.core.windows.net/realestateapp-applicationfiles-uat/`
  (files under `ApplicationFiles/OCR/Temp/<guid>`).

---

## 5. Logging

- `logs/app/YYYY-MM-DD.log`: `"<UAE> UAE | <IST> IST | LEVEL | logger | msg"`. Per file: cheques,
  tokens, ms. Per processed cheque: page, rotation, crop, output. Per payment: matched, file,
  cropped, OcrResponse. Summary line with totals.
- `logs/results/YYYY-MM-DD.jsonl`: one JSON per request (timestamps, model, totals, usage, results).
- Roll at UAE midnight; SAS redacted; no auto cleanup. On VM also NSSM stdout/stderr logs.

---

## 6. Running locally (Windows)

```powershell
cd D:\softspace\Desktop\Paperless_cheque
.venv\Scripts\activate
python -m uvicorn app.main:app --reload --port 8080
```
Base URL `http://localhost:8080/api/v1`, Swagger `http://localhost:8080/docs` (Authorize -> key).

Machine quirks:
- `python` not on PATH -> `py -3.12 -m venv .venv`.
- **Application Control blocks pip `.exe` launchers** -> always `python -m uvicorn / pip / pytest`.
- Port **8080** (never 8000); 8081 for throwaway servers.

Tests: `python -m pip install -r requirements-dev.txt`, `python -m pytest`.

---

## 7. Postman

Collection `postman/Paperless_Cheque.postman_collection.json`; collection auth = API Key
`X-API-Key: {{apiKey}}`; variables `baseUrl`, `apiKey`. User has environments for local and VM.
Requests: Health, Extract single/multiple, Match, Match empty (400), negative extract tests,
Auth missing key (401). Local `file_urls` can be sample names (`cheque_1.pdf`); VM needs blob URLs.

Samples (gitignored): `samples/cheque_1..4.pdf`, `samples/test_match_request.json`.
OCR: cheque_1 10000 @ 27-09-2024, cheque_2 5333 @ 05-08-2026, cheque_3 5000 @ 01-10-2026,
cheque_4 6667 @ 05-03-2027.

---

## 8. UAT deployment (VM)

- VM `194.233.75.55`, folder `D:\AI-project\papperless-cheque`, port **8007**.
- NSSM service `PaperlessChequeApi`, display name "Paperless Cheque API".
  App: `.venv\Scripts\python.exe -m uvicorn app.main:app --host 0.0.0.0 --port 8007`.
- venv must be **Python 3.12** (`py -3.12 -m venv .venv`; VM default is 3.14).
- Firewall rule "Paperless Cheque API 8007" (inbound TCP 8007) required for external access.
- VM needs its own `.env` (APP_ENV=uat, API_KEYS, keys, connection string) and
  `config\settings.uat.json` (real `gemini_model`, blob base URL).
- URLs: health `http://194.233.75.55:8007/api/v1/health`, Swagger `http://194.233.75.55:8007/docs`.
- Update: `git pull origin main` -> `nssm restart PaperlessChequeApi` (pip install only if requirements change).
- Issues hit: missing venv (SERVICE_STOPPED); missing `.env`/`settings.uat.json` (Field required);
  no firewall rule (external timeout); placeholder `gemini_model` -> model 400 -> "None of the cheque files could be processed."

---

## 9. Repos & workflow

- **Main repo** (`origin`): ships to UAT. Clean and professional.
- **Backup repo** (`backup`): same code, `git push backup main`.
- **Knowledge repo** (this file), `D:\softspace\Desktop\paperless-knowledge`. Update after every change.

Rules for main + backup repos:
- **No mention of Claude anywhere**: no CLAUDE.md, no Co-Authored-By/Generated-with lines. Claude must not
  appear in contributors.
- **No secrets or model details**: `.env*`, `config/settings.*.json` (except example), `logs/`, `samples/`,
  `.venv/` gitignored; `/health`, Swagger, errors never mention the model. Scan diff before every commit.
- Storage connection string/account key never in code, repos or this file (only `.env`). It was shared
  in chat -> **rotate before prod**.
- Git identity (repo-local): Shithin Devadasan <shithin.devadasan@softspace.ae>.

---

## 10. Working preferences

- **Ask before running tests, starting servers, or editing files** unless the user asked for that action.
- Keep answers short; user likes "short title + one line description" summaries.
- Messages to end users: short and friendly.

---

## 11. Change history

| Date | Change |
|---|---|
| 2026-10-20 | Postman collection; venv recreated; port 8080; DD-MM-YYYY dates |
| 2026-10-22 | Installment inferred from date order; per-cheque overlap `error` |
| 2026-10-23 | `/cheques/match`, local-folder testing, UAE/IST + token logging; any-format dates; `OcrResponse` |
| 2026-10-27 | Installment check removed; friendly OcrResponse; professional restructure; MatchedOcrFile for unmatched; repos created |
| 2026-10-27 | Crop/rotation (OpenCV + model layout), blob read/write via connection string |
| 2026-10-27 | Config split: JSON settings files + secrets-only `.env`; X-API-Key auth |
| 2026-10-27 | Pushed `e4b8357`; deployed to UAT VM (NSSM, port 8007); working with blob URLs |
| 2026-10-28 | `MatchedOcrFile` -> `MatchedOcrFileURL`; new `CroppedOCRFileURL`; processed files saved as `_ocr.pdf` / `_ocr_N.pdf`; only matched cheques are cropped. Pushed `f746a4d` to main + backup (tests not run before push) |

## 12. Open items
- Deploy `f746a4d` to VM and verify; notify frontend of renamed keys.
- Run `python -m pytest` on the latest code.
- Rotate the storage account key before prod; delete local `.env.backup`.
- Decide `azure_sas_expiry_hours` (is the container private? returned URLs need SAS if so).
- Set UAT `cors_origins` to the frontend origin.
- Prod: HTTPS in front of the API; separate API key per service.
- Test real overlapping / multi-cheque / tilted samples and tune `crop_padding_percent`.
