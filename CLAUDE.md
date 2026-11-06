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
   get back for every cheque: date, amount (figures), amount in words, installment, service,
   payee name, holder name, IBAN, `unclear_fields`, `issues`, `error`. No cropping.
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
ServiceDescription, Amount, DueDate, ChequepayeeName?, ChequeHolderName?, ChequeHolderNameAr?,
ChequeHolderIBAN?, ...} ]}`.
`file_urls` must be a JSON **array of plain URL strings** (not objects, not comma-separated);
the list key is `payments` (not `payment_services`) — the .NET team hit 422 on both (2026-10-29).
Response: the **payment list only**; extra keys the caller sends are returned unchanged. Four keys filled:

| Key | Matched | Not matched |
|---|---|---|
| `MatchedWithOcr` | `true` | `false` |
| `MatchedOcrFileURL` | original URL of the cheque's file | original URL of the **closest** cheque's file; null if none |
| `CroppedOCRFileURL` | `..._ocr.pdf` URL if cropped/rotated, else null | **always null** (never processed) |
| `OcrResponse` | `"Matched"` | short reason (below) |

- **Match = same Amount (±0.01) AND same date AND every name/IBAN sent matches AND the cheque
  has no issues (§3.7) and no unclear core field.** Installment and service NOT checked.
- **Name / IBAN (2026-11-02):** `ChequepayeeName` -> `payee_name`, `ChequeHolderName` ->
  `holder_name`, `ChequeHolderIBAN` -> `iban`. Checked **only when a non-empty value is sent**.
  Same script: names ignore case, spaces and **punctuation** (2026-11-02 late: `MS.` = `MS`,
  `L.L.C.` = `LLC`) but every letter must match (`Ahmad` ≠ `Ahmed`); IBAN ignores spaces (and
  Arabic digits are converted). Messages: `Payee name doesn't match. Cheque shows X.`,
  `IBAN not found on the cheque.` (sent but not printed). IBAN is often not printed on UAE cheques.
- **Arabic (2026-11-03):** new optional key `ChequeHolderNameAr` (holder name in Arabic, null if not
  sent; label in messages "Arabic holder name"). Any name key may be English or Arabic. The model
  also returns internal `holder_name_ar` (Arabic holder printed beside English) and
  `payee_name_latin` / `holder_name_latin` (English spelling of an Arabic name) — never in responses.
  `_names_match`: if the cheque shows the name in the **same script** as sent -> strict letters;
  only if it has it **only in the other script** -> compared **by sound** (`_sound`: vowels, y/w,
  doubled letters, final h ignored; Arabic converted by `utils/arabic.arabic_to_latin`, plus the
  model's latin spelling). Mismatch shows the cheque name(s), `A / B` if both scripts, plus
  `(English spelling)` for an Arabic-only name when English was sent.
  Arabic letter variants (ا/أ/إ, ة/ه, ى/ي) are still strict within Arabic — **user decision pending**.
- `ChequeAmountInWords` request key was added then **removed** (2026-11-02): the backend checks
  words vs figures on every cheque instead.
- Payments in request order; **each cheque used at most once**.
- `OcrResponse` order: cheque issues (Signature / words≠figures / damaged first, then the rest)
  -> amount/date -> name/IBAN -> "X is not clearly visible." -> "Please upload a clear copy."
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
- OpenCV: render PDF page (200 dpi) -> crop to box (0-1000 scale, 2% padding) -> quarter turn
  from model's `text_top` (up 0 / right -90 / down 180 / left 90, clockwise) -> fine tilt
  **measured by OpenCV** (`measure_tilt`: Hough lines within ±20°, length-weighted median;
  falls back to model `tilt_degrees`) -> `_trim_to_cheque` removes background corners ->
  saved as a **one-page PDF** (JPEG q92 inside). `rotation_degrees` is now a computed property.
  (Before 2026-11-02 the model returned a signed `rotation_degrees` and confused directions.)
- Local testing: `settings.local.json` `processed_folder: "processed"` -> `samples\processed\<request-id>\`.
- Naming (suffix `processed_file_suffix` = `_ocr`), always `.pdf`:
  - 1 cheque in file: `<name>_ocr.pdf`
  - several cheques: `<name>_ocr_<n>.pdf` (n = cheque's position in the file; only matched ones created)
- **Folder (2026-10-30/11-02):** `processed_folder` = `ApplicationFiles/ChequeMatchOcr/Cropped`
  (default in committed `settings.json`, relative to the container) ->
  `.../ChequeMatchOcr/Cropped/<request-id>/<name>_ocr.pdf`. **One 32-char GUID folder per request**
  (same id as in logs), so concurrent / repeated hits never overwrite. Written directly, no move;
  original untouched. `processed_folder: null` = old behaviour (next to the original).
  Storage grows per call -> suggest an Azure lifecycle rule on `ChequeMatchOcr/Cropped/`.
- Blob: written only if the source URL is inside `azure_blob_base_url` (connection string). Other
  URLs are read-only -> original kept. Local mode: written under `local_files_dir`.
- Any failure -> original kept, logged.
- Limits: overlapped crops may include part of the covering cheque; hidden parts can't be
  recovered; no perspective (keystone) correction; quality depends on model's box.

### 3.4 Dates (`utils/dates.py`)
`parse_date()` accepts ISO, numeric (`05/11/2026`, `5/11/26`, `2026/11/05`), month names
(`5 Nov 2026`, `Nov 5, 2026`, `5th Nov 2026`), trailing times, .NET `/Date(ms)/`.
**Ambiguous numeric dates are day-first (UAE)**. Returns None if unreadable.
Arabic-Indic digits (`١٠/١١/٢٠٢٦`) are converted first (`utils/arabic.western_digits`), also for
amounts (`٦٬٢٥٠٫٠٠` -> 6250; before 2026-11-03 this read as **6**) and DueDate. Arabic month names not handled.

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

### 3.7 Cheque checks (every cheque, 2026-11-02)
OCR returns `issues: [{code, detail}]` and `unclear_fields`; `ChequeOcr` validator adds backend
checks and writes all problems into `error`. Any issue -> payment unmatched, reason in `OcrResponse`.

**Checked first (user's priority list, always first in OcrResponse):**
| Code | Message |
|---|---|
| SIGNATURE_MISSING | `Signature missing.` |
| AMOUNT_MISMATCH | `Amount in words (Five Thousand) does not match figures (8,000).` — **backend decides**: `utils/amount_words.words_to_number(amount_in_words)` vs `cheque_amount` (English words, lakh/crore, fils, fuzzy spelling); model's flag dropped if numbers agree. Arabic words not converted -> model's flag used |
| DAMAGED | `Cheque appears damaged near the MICR line.` (location from OCR) + clear-copy request |

**Also checked:** ALTERATION, AMOUNT_MISSING, DATE_ISSUE (only missing / impossible date; the
**stale-date check was removed 2026-11-02** — a wrong or old date fails the DueDate comparison
anyway), PAYEE_ISSUE (skipped when `ChequepayeeName` is sent), MIXED_LANGUAGE, ILLEGIBLE, IMAGE_INCOMPLETE.
User rule: don't add checks already covered by the payment keys.

**ALTERATION** (any strike-through / overwritten character / correction, even initialled) is 4th in
priority: `Overwriting or strike-through found on the <field>. UAE banks do not accept altered
cheques. Please provide a new cheque.`

**Bad images (2026-11-02 evening):**
- Backend nulls every field listed in `unclear_fields` (the model sometimes returns a guess anyway).
- `IMAGE_INCOMPLETE` / `ILLEGIBLE` = `unreadable`: no amount/date/name comparison, no derived
  issues, and the whole OcrResponse is just `Cheque image is incomplete or unclear. Please upload a clear, full copy.`
- **Sharpness gate** in `ChequeExtractor._drop_unreadable`: `image_processor.sharpness()` (Laplacian
  variance at 1000 px width) per page; below `min_image_sharpness` (25, config) the cheque becomes
  `as_unreadable()` — all values dropped. Calibrated: real cheques 80–590, blurred 1.7, low-res 6.3.
- Model details longer than 45 chars (`MAX_DETAIL_LENGTH`) are dropped from messages.

**Unclear fields:** fields present but not clearly readable are returned null, never guessed or
compared -> `Payee name is not clearly visible. Please upload a clear copy.` Core fields (date,
amount, words, payee) always count; holder name / IBAN only when the caller sent them.
A field simply not printed is not "unclear". Issues already explained by an unclear field are not repeated.
**Doubt rule (2026-11-03):** a field is unclear if even one character could be read as another
(4/6, 1/7, 3/8…). For dates the model lists the doubtful part (`date_day` / `date_month` /
`date_year`); the whole date is nulled and the message names it: `Year of the cheque date is not
clearly visible. Please upload a clear copy.` Label "Amount" became `Amount in figures`.
Trigger: in the 4-cheque photo (scenario 24) the model read 27-09-2026 as 2024 (handwritten 6 looks like 4).
A *confidently* wrong read is still possible -> proposed fix (not approved): second independent read
of each cheque from a full-resolution crop; disagreement -> unclear (option A: multi-cheque files only).

**Model output loops (fixed 2026-11-02):** numbers (`cheque_amount`, `installment_number`, `page`,
`tilt_degrees`) are requested as **strings, maxLength 12** and parsed back (`_parse_number`);
`amount_in_words_value` removed (backend converts the words). `generate_structured` retries on
invalid / cut-off JSON and traces wasted tokens.

---

## 4. Configuration

Layered (later wins): `config/settings.json` -> `config/settings.<APP_ENV>.json` -> `.env` -> env vars.
`APP_CONFIG_DIR` can override the config folder. Config files are grouped in sections
(app, ocr_model, uploads, file_urls, blob_storage, image_processing); unknown keys or secrets
in a config file stop startup.

- **`.env` (secrets only, gitignored):** `APP_ENV`, `API_KEYS`, `GEMINI_API_KEY`, `AZURE_STORAGE_CONNECTION_STRING`.
- **`settings.<env>.json` (gitignored):** `gemini_model` (`gemini-3.8-flash`), `azure_blob_base_url`,
  `cors_origins`, `docs_enabled`, `local_files_dir`.
- **Shared defaults:** sampling (temp **1.0** since 2026-11-02, was 0; top_p 0.95, top_k 40, 4096 tokens), retries 2, timeout 120 s,
  limits (20 files, 15 MB, 5 concurrent), `azure_sas_expiry_hours` 0, image processing
  (enabled, `_ocr`, `processed_folder` = `ApplicationFiles/ChequeMatchOcr/Cropped`, padding 2%,
  200 dpi, JPEG q92).
- UAT blob base: `https://softspacestorage.blob.core.windows.net/realestateapp-applicationfiles-uat/`
  (files under `ApplicationFiles/OCR/Temp/<guid>`).

---

## 5. Logging

- `logs/app/YYYY-MM-DD.log`: `"<UAE> UAE | <IST> IST | LEVEL | logger | msg"`. Per file: cheques,
  tokens, ms. Per processed cheque: page, rotation, crop, output. Per payment: matched, file,
  cropped, OcrResponse. Summary line with totals.
- `logs/results/YYYY-MM-DD.jsonl`: **one trace dict per request** (`app/core/tracing.py`, middleware
  in `main.py`, 2026-11-02): request_id (= `X-Request-ID` header, = cropped folder GUID), endpoint,
  started/finished {uae, ist}, total_ms, status_code, outcome, client {ip, user_agent, api_key_id =
  sha256[:10]}, counts, `stages_ms` (load / extraction / matching / cropping), usage, `files[]`
  (load_ms, queue_wait_ms, model_ms, quality_check_ms, sharpness, tokens, cheques summary — no names),
  `cheques_processed[]` (process_ms, save_ms, rotation, size_kb), `payments[]` (`checks_sent` = key
  names only, never values), issues_summary, events (retries with wasted tokens), errors.
  App log gets one `[id] SUMMARY {...}` line per request.
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

| 2026-10-29 | Diagnosed .NET 422 from VM log (file_urls objects, `payment_services`); file_urls = array; concurrency/overwrite review |
| 2026-10-30 | Cropped files -> `ChequeMatchOcr/Cropped/<request-id>/` (`processed_folder`, full GUID request id) |
| 2026-11-02 | `ChequepayeeName` / `ChequeHolderName` / `ChequeHolderIBAN` strict checks; `/extract` returns payee/holder/IBAN/words |
| 2026-11-02 | Unclear fields never guessed -> "not clearly visible. Please upload a clear copy."; cheque issue checks (signature, words≠figures, damaged first; alteration, stale date, etc.); `ChequeAmountInWords` added then removed |
| 2026-11-02 | Alteration message (bank rejects); short messages; never-guess (null unclear, sharpness gate); text_top + OpenCV tilt + trim; temperature 1.0 + JSON retry; test cheque generator (local). Committed `736041e`, pushed to **backup only** |

| 2026-11-02 | Request trace (one dict per request, X-Request-ID); stale check removed; names ignore punctuation; numbers as strings + words converter (loops fixed: requests 0 retries, 4–10 s vs 74 s) |
| 2026-11-03 | All scenarios 01–40 tested by user; doubtful date part named (day/month/year), one-digit-doubt rule; Arabic: `ChequeHolderNameAr`, English↔Arabic name matching by sound, Arabic-Indic digits; Arabic test scenarios 50–60. pytest 122 passed. Committed `cdba8cb`, pushed to **main + backup** (user approved) |

| 2026-11-04 | `/cheques/match/upload` (form-data files + payments JSON, same checks); payee / holder / IBAN keys grouped in `cheque_info` (flat keys still accepted); rotation from MICR line position (model text_top mixed up left/right; fallback only); one re-read when a cheque comes back with no date and no amount; closest-cheque ranking (unreadable before clearly different); trace cheque index 0-based. Regression 01/03/20-24/26/50 passed. pytest 152. Commit `f8baa0e` -> backup 2026-11-04, main 2026-11-06 (user approved) |

## 12. Open items
- **Main + backup at `f8baa0e` (2026-11-06).** VM: tag `backup-before-f8baa0e` + folder copy, `git pull`, `nssm restart PaperlessChequeApi`.
- **ALTERATION missed (on hold by user):** since ~2026-11-04 the model no longer flags the cheque_3 payee strike-through (scenarios 14 / 24), also on the old deployed code -> an altered cheque can come back Matched. Proposed (not built): focused second alteration check on matched cheques.
- Tell .NET: `cheque_info`, `/cheques/match/upload`, X-Request-ID; client timeout >= 120 s (empty-read retry can add ~19 s).
- Regression of the stricter doubt prompt (01, 06, 23, 24, 26, 32) and Arabic scenarios 50–60 not
  yet confirmed against the real model at push time — check UAT results.
- Decide: Arabic letter variants equal or not; Arabic month names; Arabic amount-in-words converter;
  second-read verification (option A) for confident misreads.
- Tell frontend/.NET: `ChequeHolderNameAr`, new OcrResponse texts ("Amount in figures…",
  "Year of the cheque date…", "Arabic holder name…"), `X-Request-ID` header, cropped URL path.
- Watch in UAT: strict name matching false negatives; model's "unclear"/issue calls too strict or lax.
- Bank statements not handled (fields read from cheques only).
- `sctrd.jpg` test (4 cheques): A 10,000 @ 27-09-2026, B 5,333 @ 05-08-2026, C 5,000 @ 01-10-2026,
  D 6,667 @ 05-03-2027. Names / IBANs are personal data: never store them in this repo; read them
  with `/cheques/extract` when needed.
- Rotate the storage account key before prod; delete local `.env.backup`.
- Decide `azure_sas_expiry_hours` (is the container private? returned URLs need SAS if so).
- Set UAT `cors_origins` to the frontend origin.
- Prod: HTTPS in front of the API; separate API key per service.
- Test real overlapping / multi-cheque / tilted samples and tune `crop_padding_percent`.
