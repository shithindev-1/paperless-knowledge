# Project Memory — Paperless Cheque OCR API

Short, durable memory for anyone (or any assistant) picking up this project. The full
technical reference is [CLAUDE.md](CLAUDE.md). Update both after every change.

_Last updated: 2026-11-10_

---

## Current state

| Item | State |
|---|---|
| Main repo `paperless_cheque_dev` | `a2ff35d` (2026-11-10) — top-level `cheque_info`, response object |
| Backup repo `papperless-backup` | `a2ff35d` |
| UAT VM | `10a4eff` (case study, deployed + verified 2026-11-06, tag `backup-before-10a4eff`). `a2ff35d` deploy pending; `194.233.75.55:8007`, folder `D:\AI-project\papperless-cheque` |

### ▶ Resume here (next session)
1. Deploy `a2ff35d` to the VM **together with the .NET change** (response is now
   `{cheque_info, payments}`; `cheque_info` only at the top level, per-payment -> 422).
2. After deploy: one `/match` with top-level `cheque_info`, check `CaseStudy/` and `logs\results`.
3. Pending user decisions: payee one-letter misreads (A/B/C); Arabic letter variants;
   alteration second check and confident year misread (both on hold with the team).

---

## Decisions (and why)

| Decision | Why |
|---|---|
| Match = Amount + DueDate (+ any name/IBAN sent) + cheque has no issues | Business rule from user |
| Installment and service are **not** checked | Backend always sends correct installments |
| `OcrResponse` shows only what the cheque says, short and friendly | Read by non-technical users |
| `MatchedOcrFileURL` = original file; `CroppedOCRFileURL` = processed PDF or null | Frontend needs both |
| Only **matched** cheques are cropped/rotated | No wasted work or blob files for unmatched |
| Cropped output = one-page PDF `<name>_ocr.pdf` / `_ocr_N.pdf` | User's naming standard |
| Cropped folder `ApplicationFiles/ChequeMatchOcr/Cropped/<request-id>/` | One GUID per request: no overwrites on concurrent / repeated hits |
| Name / IBAN checks are **strict**, and only when a value is sent | User asked for strict; empty = skip |
| `cheque_info` once at the request top level, checked for every payment; inside a payment -> 422; response `{cheque_info, payments}` | User 2026-11-10: one tenant's cheques share payee / holder / IBAN; per-payment overrides too complicated |
| Case study: failed cheques saved after the response to `ApplicationFiles/CaseStudy/<request-id>/` (crop **or** original + case JSON), daily index, no clearing | User wants to study why matches fail without slowing the main pipeline |
| Case study duplicates: fingerprint = file content hash + cheque index; repeats only indexed | Amount / date / OcrResponse are not unique (user) |
| Names ignore case, spaces and punctuation (`MS.` = `MS`) | User: a missing dot must not fail the match |
| Names may be English or Arabic; same script strict, other script by sound; `ChequeHolderNameAr` key | User: .NET may send either; check both ways and report mismatches |
| Doubtful date part named (`Year of the cheque date is not clearly visible.`); one doubtful digit = unclear | Model read handwritten 2026 as 2024; user: don't guess, say exactly what is unclear |
| Stale-date check removed | User: DueDate comparison already flags wrong dates; avoid duplicate checks |
| One trace dict per request in `logs/results` + `X-Request-ID` | User wanted every timing/count for a hit in one identifiable record |
| `ChequeAmountInWords` removed | Words vs figures is checked by the backend on every cheque |
| Signature missing, words ≠ figures, damaged are checked **first** | User's priority list |
| Other checks (alteration, payee issue, mixed language, illegible, incomplete) kept | User: "keep it like this" |
| Unclear fields are never guessed; ask for a clear copy | User request |
| Backend nulls any field the model marks unclear; no comparison on IMAGE_INCOMPLETE / ILLEGIBLE | Model still returned guessed dates (e.g. 09-10-2023) on blurred images |
| Sharpness gate (Laplacian variance < `min_image_sharpness` 25) drops all values | Model invented dates on a low-res image without flagging it; real cheques 80–590, blurred 1.7 / low-res 6.3 |
| Bad image -> one message: "Cheque image is incomplete or unclear. Please upload a clear, full copy." | User wanted short, friendly messages, not lists of consequences |
| Model details > 45 chars dropped from messages | Same |
| Alteration (strike-through / overwriting, even initialled) -> "...UAE banks do not accept altered cheques. Please provide a new cheque." | UAE banks reject altered cheques |
| Orientation from model as `text_top` (up/right/down/left); fine tilt measured by OpenCV; corners trimmed | Model confused rotation direction; rotated output must be human-viewable |
| Temperature 1.0 (was 0.0) + retry on cut-off / invalid JSON | At 0.0 the model looped on digits (`000…`/`001001…`) until max tokens -> "None of the cheque files could be processed." |
| Local processed output in `samples\processed\<request-id>\` | User wants a separate local folder to view results |
| Config in JSON files, secrets only in `.env` | Clean structure; prod differs from UAT |
| X-API-Key auth; shared key with leasing OCR OK for UAT, separate for prod | Security guidance given |

---

## Rules

- Main and backup repos: **no mention of Claude** (no CLAUDE.md, no co-author lines); no secrets,
  no model name in code, Swagger, `/health` or errors.
- Secrets (Gemini key, storage connection string, API keys) only in `.env`. The storage key was
  shared in chat: **rotate before prod**.
- Ask before running tests, starting servers or editing code. Local port 8080 (never 8000).
- Use `python -m ...` (Application Control blocks pip `.exe` launchers); venv with `py -3.12`.
- Summaries: short title + one-line description.

---

## Integration notes (frontend / .NET)

- `POST /api/v1/cheques/match`, headers `Content-Type: application/json`, `X-API-Key`.
- `file_urls`: JSON array of plain URL strings (one item per URL). List key: `payments`.
- Optional **top-level** `cheque_info`: `ChequepayeeName`, `ChequeHolderName`, `ChequeHolderNameAr`,
  `ChequeHolderIBAN` (names in English or Arabic; Arabic-Indic digits accepted in DueDate / IBAN).
  Not allowed inside a payment (422). Response: `{"cheque_info": ..., "payments": [...]}`.
- `POST /api/v1/cheques/match/upload`: form-data `files`, `payments` (JSON), optional `cheque_info` (JSON).
- Every response has `X-Request-ID` (same id as the log trace and the cropped folder).
- Always use the returned URLs; fall back to `MatchedOcrFileURL` when `CroppedOCRFileURL` is null.
- Client timeout ≥ 120 s. Call from the backend, not the browser.
- 422 debugging: VM log `logs\app\<date>.log`, search "invalid request" — `loc` shows the bad field.

---

## Test data

`sctrd.jpg` (UAT blob `ApplicationFiles/OCR/Temp/sctrd.jpg`, 4 cheques):

| Cheque | Amount | Date | Payee / holder / IBAN printed? |
|---|---|---|---|
| A | 10,000 | 27-09-2026 | payee, holder, IBAN |
| B | 5,333 | 05-08-2026 | payee, holder, IBAN (spaced format) |
| C | 5,000 | 01-10-2026 | holder |
| D | 6,667 | 05-03-2027 | payee |

Real names / IBANs are deliberately **not stored here** (personal data). Get them by running
`/cheques/extract` on the file.

Generated scenarios (local only, gitignored, contain real names/IBANs — never commit):
`samples/generated/` — 01–14 cheque checks, 20–26 crop/rotation/layout, 30–40 request-only,
50–52 Arabic cheque images + 53–60 Arabic request-only (source `samples/cheque_5_arabic.png`, ADIB,
6,250 @ 10-11-2026 in Arabic-Indic digits);
`SCENARIOS.md` (expected responses), `requests/*.json`, `Cheque_Test_Scenarios.postman_collection.json`,
`_overview.jpg`. Regenerate: `.venv\Scripts\python.exe samples\tools\generate_test_cheques.py`.

Local samples: `samples/cheque_1..4.pdf` (10000 @ 27-09-2024, 5333 @ 05-08-2026, 5000 @ 01-10-2026, 6667 @ 05-03-2027).

---

## Open items

- Deploy `a2ff35d` to VM with the .NET change; tell frontend about top-level `cheque_info`, the response object, `/cheques/match/upload` and new messages.
- ON HOLD: model misses strike-through ALTERATION (cheque_3); altered cheque can be Matched. Focused second check proposed, not built.
- Watch UAT for strict-name false negatives and over/under-strict "unclear" / issue calls.
- Bank statements not handled yet.
- Decide SAS for returned URLs (if container is private); set UAT CORS origins.
- Prod: HTTPS, separate API key, rotated storage key, Azure lifecycle rule on `ChequeMatchOcr/Cropped/`.
- Delete local `.env.backup`.
