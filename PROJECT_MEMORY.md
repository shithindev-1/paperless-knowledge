# Project Memory — Paperless Cheque OCR API

Short, durable memory for anyone (or any assistant) picking up this project. The full
technical reference is [CLAUDE.md](CLAUDE.md). Update both after every change.

_Last updated: 2026-11-02 (evening)_

---

## Current state

| Item | State |
|---|---|
| Main repo `paperless_cheque_dev` | `f746a4d` (2026-10-28) — **do not push until testing is finished** (user, 2026-11-02) |
| Backup repo `papperless-backup` | `736041e` (2026-11-02, WIP — all changes below) |
| UAT VM | `f746a4d`; `194.233.75.55:8007`, NSSM `PaperlessChequeApi`, folder `D:\AI-project\papperless-cheque` |

### ▶ Resume here (next session)
1. Local Postman testing with `samples/generated/` scenarios: **01–21 tested on 2026-11-02**.
   **Continue from `22_tilted_on_desk.jpg`** (22–26 layout, then 30–40 request-only).
   Restart the server first (config changed: temperature 1.0, local `processed_folder`).
2. Check processed output in `samples\processed\<request-id>\` (local only).
3. Run `python -m pytest` (not run since many changes; never run on the 736041e code).
4. Only then, with the user's OK: push main -> VM `git pull` + `nssm restart PaperlessChequeApi`.
5. Tell the .NET/frontend team about new optional keys and new OcrResponse messages.

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
| `ChequeAmountInWords` removed | Words vs figures is checked by the backend on every cheque |
| Signature missing, words ≠ figures, damaged are checked **first** | User's priority list |
| Other checks (alteration, stale > 6 months, payee issue, mixed language, illegible, incomplete) kept | User: "keep it like this" |
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
- `file_urls`: JSON array of plain URL strings. List key: `payments`.
- Optional per payment: `ChequepayeeName`, `ChequeHolderName`, `ChequeHolderIBAN`.
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
`samples/generated/` — 01–14 cheque checks, 20–26 crop/rotation/layout, 30–40 request-only;
`SCENARIOS.md` (expected responses), `requests/*.json`, `Cheque_Test_Scenarios.postman_collection.json`,
`_overview.jpg`. Regenerate: `.venv\Scripts\python.exe samples\tools\generate_test_cheques.py`.

Local samples: `samples/cheque_1..4.pdf` (10000 @ 27-09-2024, 5333 @ 05-08-2026, 5000 @ 01-10-2026, 6667 @ 05-03-2027).

---

## Open items

- Test + push the uncommitted changes; deploy to VM; tell frontend about the new keys and messages.
- Watch UAT for strict-name false negatives and over/under-strict "unclear" / issue calls.
- Bank statements not handled yet.
- Decide SAS for returned URLs (if container is private); set UAT CORS origins.
- Prod: HTTPS, separate API key, rotated storage key, Azure lifecycle rule on `ChequeMatchOcr/Cropped/`.
- Delete local `.env.backup`.
