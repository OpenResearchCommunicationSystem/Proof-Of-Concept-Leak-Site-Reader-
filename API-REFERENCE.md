# Leak Data Reader PoC — API Reference

All endpoints are mounted under the `/api` prefix by Express (`app.use("/api", router)`).

**Authentication note:** The backend enforces JWT verification only on
`POST /api/auth/change-password` and `GET /api/auth/me`. All other endpoints accept
requests without a token. The login requirement is enforced by the frontend's `AuthGate`
component. In any multi-tenant or publicly accessible deployment these routes should be
protected server-side.

Base URL (development): `https://<replit-dev-domain>/api-server/api`

---

## Health

### GET /api/healthz

Service health check. Returns immediately with `{ "status": "ok" }`. No authentication
required. Used by the Replit platform to verify the API server is running.

**Response `200`:**
```json
{ "status": "ok" }
```

**curl:**
```bash
curl https://<host>/api/healthz
```

---

## Auth

### POST /api/auth/login

Authenticate with username and password. Returns a JWT.

**Request body:**
```json
{ "username": "wikiran_agent", "password": "YourPassword" }
```

**Response `200`:**
```json
{
  "token": "<jwt>",
  "mustChangePassword": false,
  "username": "wikiran_agent"
}
```

**Response `401`:** Invalid credentials.

**curl:**
```bash
curl -X POST https://<host>/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"wikiran_agent","password":"YourPassword"}'
```

---

### POST /api/auth/change-password

Change the authenticated user's password. Issues a new token with `mustChangePassword: false`.

**Headers:** `Authorization: Bearer <jwt>`

**Request body:**
```json
{ "newPassword": "NewPassword123" }
```

**Response `200`:** Same shape as login response — new token.

**curl:**
```bash
curl -X POST https://<host>/api/auth/change-password \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"newPassword":"NewPassword123"}'
```

---

### GET /api/auth/me

Return the authenticated user's profile from the JWT payload.

**Response `200`:**
```json
{ "id": 1, "username": "wikiran_agent", "mustChangePassword": false }
```

**curl:**
```bash
curl https://<host>/api/auth/me -H "Authorization: Bearer $TOKEN"
```

---

## Datasets

### GET /api/datasets

List all datasets with their email and thread counts.

**Response `200`:**
```json
[
  {
    "id": 1,
    "name": "IFC Leak",
    "description": "...",
    "email_count": 562634,
    "thread_count": 12300,
    "created_at": "2026-01-15T10:00:00.000Z"
  }
]
```

**curl:**
```bash
curl https://<host>/api/datasets -H "Authorization: Bearer $TOKEN"
```

---

## Emails

### GET /api/emails

List emails with filtering, search, and pagination.

**Query params:**

| Param | Type | Description |
|---|---|---|
| `dataset_id` | integer | Filter to a specific dataset |
| `thread_id` | integer | Filter to a specific thread |
| `search` | string | Full-text search across subject, body, translation, from address, and attachment OCR text |
| `sender` | string | Case-insensitive substring match on `from_address` |
| `date_from` | ISO 8601 date | Earliest email date |
| `date_to` | ISO 8601 date | Latest email date |
| `has_attachments` | `"true"` | Only emails with attachments |
| `language` | string | Language code: `"fa"`, `"en"`, etc. |
| `is_junk` | `"true"` / `"false"` | Filter by junk flag |
| `page` | integer | Page number (default `1`) |
| `limit` | integer | Results per page (default `50`, max `200`) |

**Response `200`:**
```json
{
  "items": [
    {
      "id": 42,
      "message_id": "<abc@example.ir>",
      "subject": "Re: حواله",
      "from_address": "order@sadafexchange.com",
      "from_name": "Sadaf",
      "to_addresses": ["treasury@tejaratbank.ir"],
      "date": "2022-12-20T10:29:00.000Z",
      "has_attachments": true,
      "attachment_count": 1,
      "language": "fa",
      "thread_id": 7,
      "dataset_id": 1,
      "is_junk": false,
      "body_preview": "First 200 chars of body...",
      "matched_in": ["translation", "ocr"]
    }
  ],
  "total": 1842,
  "page": 1,
  "limit": 50,
  "pages": 37
}
```

`matched_in` is always present (empty array when no `search` is provided). Values: `"translation"`,
`"ocr"`, `"ocr_translation"`.

**curl:**
```bash
curl "https://<host>/api/emails?dataset_id=1&search=حواله&page=1" \
  -H "Authorization: Bearer $TOKEN"
```

---

### GET /api/emails/:id

Return a single email with full body, entities, and attachments.

**Response `200`:**
```json
{
  "id": 42,
  "message_id": "...",
  "subject": "...",
  "from_address": "...",
  "from_name": "...",
  "to_addresses": [],
  "cc_addresses": [],
  "date": "...",
  "body_text": "...",
  "body_html": null,
  "body_translation": "English translation here",
  "language": "fa",
  "has_attachments": true,
  "attachments": [
    {
      "id": 5,
      "filename": "invoice.pdf",
      "content_type": "application/pdf",
      "size_bytes": 48000,
      "content_hash": "abc123...",
      "is_duplicate": false,
      "ocr_text": "...",
      "ocr_translation": "..."
    }
  ],
  "entities": [
    {
      "id": 10,
      "entity_type": "PERSON",
      "value": "رضا حسینی",
      "confidence": 0.93,
      "email_id": 42,
      "value_translation": "Reza Hosseini"
    }
  ],
  "thread_id": 7,
  "dataset_id": 1,
  "is_junk": false,
  "content_hash": "sha256hex..."
}
```

**curl:**
```bash
curl https://<host>/api/emails/42 -H "Authorization: Bearer $TOKEN"
```

---

### POST /api/emails/:id/translate

Translate the email body to English using the AI. Idempotent: if a translation already
exists it is returned without calling the AI again.

**Response `200`:**
```json
{
  "email_id": 42,
  "original_language": "fa",
  "translation": "English translation..."
}
```

**curl:**
```bash
curl -X POST https://<host>/api/emails/42/translate \
  -H "Authorization: Bearer $TOKEN"
```

---

## Threads

### GET /api/threads

List email threads with optional filtering.

**Query params:**

| Param | Type | Description |
|---|---|---|
| `dataset_id` | integer | Filter to threads belonging to a dataset (includes cross-dataset threads) |
| `search` | string | Case-insensitive substring match on subject |
| `page` | integer | Default `1` |
| `limit` | integer | Default `50`, max `200` |

**Response `200`:**
```json
{
  "items": [
    {
      "id": 7,
      "subject": "Re: حواله",
      "email_count": 4,
      "participant_count": 3,
      "first_date": "2022-12-15T08:00:00.000Z",
      "last_date": "2022-12-20T10:29:00.000Z",
      "dataset_id": 1,
      "summary": "Exchange house staff discussing a wire transfer reversal."
    }
  ],
  "total": 240,
  "page": 1,
  "limit": 50,
  "pages": 5
}
```

**curl:**
```bash
curl "https://<host>/api/threads?dataset_id=1" -H "Authorization: Bearer $TOKEN"
```

---

### GET /api/threads/:id

Return a single thread with all its emails, participants, entities, and attachments.

**Response `200`:** Thread object with embedded `emails[]`, `participants[]` (string array
of email addresses), `entities[]`, and per-email `attachments[]`. Duplicate attachments
within the thread (same `content_hash`) have their `ocr_text` and `ocr_translation` set
to `null` in subsequent occurrences.

**curl:**
```bash
curl https://<host>/api/threads/7 -H "Authorization: Bearer $TOKEN"
```

---

### POST /api/threads/:id/summarize

Run AI analysis on a thread. Stores the result in the database and returns it.
Subsequent calls re-run the analysis (not idempotent).

**Request body (all optional, default `true` / `false` as shown):**
```json
{
  "include_original": true,
  "include_translation": false,
  "include_ocr": false,
  "include_ocr_translation": false
}
```

**Response `200`:**
```json
{
  "thread_id": 7,
  "summary": "Two-sentence summary...",
  "key_people": ["order@sadafexchange.com", "Mikayelian"],
  "key_topics": ["wire transfer reversal", "حواله ابطال"],
  "investigative_leads": [
    "Lead 1: Reverse wire with no stated reason — possible layering.",
    "Lead 2: ..."
  ]
}
```

**curl:**
```bash
curl -X POST https://<host>/api/threads/7/summarize \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"include_translation":true}'
```

---

## Entities

### GET /api/entities

List extracted named entities with filtering.

**Query params:**

| Param | Type | Description |
|---|---|---|
| `dataset_id` | integer | Filter to entities from emails in this dataset |
| `entity_type` | string | Exact match: `PERSON`, `ORG`, `EMAIL`, `PHONE`, `LOCATION`, `DATE`, `CRYPTO_*` |
| `search` | string | Case-insensitive substring match on `value` or `value_translation` |
| `page` | integer | Default `1` |
| `limit` | integer | Default `100`, max `500` |

**Response `200`:**
```json
{
  "items": [
    {
      "id": 10,
      "entity_type": "PERSON",
      "value": "رضا حسینی",
      "confidence": 0.93,
      "email_id": 42,
      "value_translation": "Reza Hosseini"
    }
  ],
  "total": 8400,
  "page": 1,
  "limit": 100,
  "pages": 84
}
```

**curl:**
```bash
curl "https://<host>/api/entities?dataset_id=1&entity_type=PERSON" \
  -H "Authorization: Bearer $TOKEN"
```

---

### PATCH /api/entities/:id

Update the `value_translation` field of an entity.

**Request body:**
```json
{ "value_translation": "Reza Hosseini Moghaddam" }
```

**Response `200`:** Updated entity object.

**curl:**
```bash
curl -X PATCH https://<host>/api/entities/10 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"value_translation":"Reza Hosseini Moghaddam"}'
```

---

### POST /api/entities/:id/suggest

AI-generate a translation for a single entity and save it to the database.

**Response `200`:** Updated entity object with `value_translation` filled in.

**curl:**
```bash
curl -X POST https://<host>/api/entities/10/suggest \
  -H "Authorization: Bearer $TOKEN"
```

---

### POST /api/emails/:id/entities/translate

Translate all PERSON, ORG, and LOCATION entities for a given email in a single AI call.
Saves results to the database.

**Response `200`:**
```json
{ "entities": [ /* all entities for this email, with translations */ ] }
```

**curl:**
```bash
curl -X POST https://<host>/api/emails/42/entities/translate \
  -H "Authorization: Bearer $TOKEN"
```

---

## Identities

### GET /api/identities

List identity cluster records.

**Query params:**

| Param | Type | Description |
|---|---|---|
| `dataset_id` | integer | Filter to a dataset |
| `search` | string | Case-insensitive match on `canonical_name` or `email_addresses` |
| `risk_level` | string | Exact match: `critical`, `high`, `medium`, `low` |
| `page` | integer | Default `1` |
| `limit` | integer | Default `50`, max `200` |

Results are sorted by `email_count` descending (highest-volume actors first).

**Response `200`:**
```json
{
  "items": [
    {
      "id": 3,
      "canonical_name": "Mohammad Reza Hosseini",
      "aliases": ["M.R. Hosseini"],
      "email_addresses": ["mrhosseini@example.gov.ir"],
      "email_count": 142,
      "dataset_id": 1,
      "phone_numbers": ["+98-21-1234567"],
      "physical_addresses": ["Tehran, Iran"],
      "organizations": [{ "name": "Ministry of Oil", "role": "Director" }],
      "risk_level": "high",
      "notes": "Senior procurement official."
    }
  ],
  "total": 87,
  "page": 1,
  "limit": 50,
  "pages": 2
}
```

**curl:**
```bash
curl "https://<host>/api/identities?dataset_id=1&risk_level=critical" \
  -H "Authorization: Bearer $TOKEN"
```

---

### POST /api/identities

Create a new identity record manually.

**Request body:**
```json
{
  "dataset_id": 1,
  "canonical_name": "Jane Doe",
  "aliases": ["J. Doe"],
  "email_addresses": ["jane@example.ir"],
  "phone_numbers": [],
  "physical_addresses": [],
  "organizations": [{ "name": "Example Corp", "role": "CFO" }],
  "risk_level": "medium",
  "notes": "Analyst note."
}
```

All fields are optional. `email_count` is derived from the length of `email_addresses`.

**Response `201`:** Created identity object.

**curl:**
```bash
curl -X POST https://<host>/api/identities \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"canonical_name":"Jane Doe","email_addresses":["jane@example.ir"],"risk_level":"low"}'
```

---

### PATCH /api/identities/:id

Partially update an identity record. Only fields present in the request body are updated.

**Request body:** Same shape as `POST /api/identities`. Any subset of fields.

**Response `200`:** Updated identity object.

**curl:**
```bash
curl -X PATCH https://<host>/api/identities/3 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"risk_level":"critical","notes":"Confirmed sanctions link."}'
```

---

### POST /api/identities/extract

Use AI to extract identity clusters from up to 500 emails in a dataset. Upserts
results: updates existing records matched by `canonical_name + dataset_id`, inserts new ones.

**Request body:**
```json
{ "dataset_id": 1 }
```

**Response `200`:**
```json
{ "created": 12, "updated": 5 }
```

**curl:**
```bash
curl -X POST https://<host>/api/identities/extract \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"dataset_id":1}'
```

---

### GET /api/identities/:id/emails

Return emails sent from any address in an identity's `email_addresses` list. Up to 200
results, sorted newest first.

**Response `200`:** Array of email summary objects (same shape as `GET /api/emails` items,
plus `body_preview`).

**curl:**
```bash
curl https://<host>/api/identities/3/emails -H "Authorization: Bearer $TOKEN"
```

---

## Crypto Wallets

### POST /api/crypto/scan

Scan all emails in a dataset (or all datasets if `dataset_id` is omitted) for crypto wallet
addresses using regex patterns. Writes results to the `entities` table with `CRYPTO_*` types.
Clears and re-writes existing `CRYPTO_*` entities for scanned emails before inserting.

**Request body:**
```json
{ "dataset_id": 1 }
```

**Response `200`:**
```json
{ "emailsScanned": 1200, "walletCount": 4, "keywordCount": 8 }
```

**curl:**
```bash
curl -X POST https://<host>/api/crypto/scan \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"dataset_id":1}'
```

---

### GET /api/crypto/wallets

Return all detected wallet addresses grouped by address, with occurrence metadata.

**Query params:**

| Param | Type | Description |
|---|---|---|
| `type` | string | Filter to one wallet type (e.g. `CRYPTO_ETH`) |

**Response `200`:**
```json
{
  "wallets": [
    {
      "address": "0xAbCd...",
      "type": "CRYPTO_ETH",
      "confidence": 0.97,
      "cases": ["IFC Leak"],
      "occurrences": [
        {
          "emailId": 42,
          "subject": "payment confirmed",
          "fromAddress": "sender@example.ir",
          "date": "2022-06-01T00:00:00.000Z",
          "datasetId": 1,
          "datasetName": "IFC Leak"
        }
      ]
    }
  ],
  "keywords": [
    {
      "value": "tether, crypto",
      "emailId": 55,
      "subject": "...",
      "fromAddress": "...",
      "date": "...",
      "datasetName": "IFC Leak"
    }
  ]
}
```

**curl:**
```bash
curl https://<host>/api/crypto/wallets -H "Authorization: Bearer $TOKEN"
```

---

### GET /api/crypto/summary

Aggregate crypto entity counts by type.

**Response `200`:**
```json
{
  "byType": [
    { "type": "CRYPTO_ETH", "count": 3 },
    { "type": "CRYPTO_TRX", "count": 1 }
  ],
  "uniqueAddresses": 4
}
```

**curl:**
```bash
curl https://<host>/api/crypto/summary -H "Authorization: Bearer $TOKEN"
```

---

### GET /api/crypto/by-dataset

Per-dataset breakdown of crypto findings.

**Response `200`:** Array of objects:
```json
[
  {
    "datasetId": 1,
    "datasetName": "IFC Leak",
    "totalEmails": 1200,
    "uniqueAddresses": 4,
    "keywordHits": 8,
    "byType": { "CRYPTO_ETH": 3, "CRYPTO_TRX": 1 },
    "topAddresses": ["0xAbCd...", "T9XYZ..."]
  }
]
```

**curl:**
```bash
curl https://<host>/api/crypto/by-dataset -H "Authorization: Bearer $TOKEN"
```

---

## Attachments

### GET /api/emails/:emailId/attachments

List metadata for all attachments of an email. Does not return binary data.

**Response `200`:**
```json
{
  "attachments": [
    {
      "id": 5,
      "filename": "invoice.pdf",
      "contentType": "application/pdf",
      "sizeBytes": 48000,
      "ocrText": null,
      "ocrTranslation": null,
      "isDuplicate": false
    }
  ]
}
```

**curl:**
```bash
curl https://<host>/api/emails/42/attachments -H "Authorization: Bearer $TOKEN"
```

---

### GET /api/attachments/:id/file

Download the raw binary content of an attachment. Returns the file with appropriate
`Content-Type` and `Content-Disposition: inline` headers. JPEG detection overrides
incorrect MIME types.

**Response `200`:** Binary file data.

**curl:**
```bash
curl https://<host>/api/attachments/5/file \
  -H "Authorization: Bearer $TOKEN" \
  -o invoice.pdf
```

---

### POST /api/attachments/:id/ocr

Run OCR on an attachment. Supports PDF (text extraction via pdf-parse, then GPT-4o
cleanup) and images (GPT-4o vision). Saves result to `attachments.ocr_text` and clears
any existing translation.

**Response `200`:**
```json
{ "id": 5, "ocrText": "Extracted and cleaned text..." }
```

**curl:**
```bash
curl -X POST https://<host>/api/attachments/5/ocr \
  -H "Authorization: Bearer $TOKEN"
```

---

### POST /api/attachments/:id/ocr/translate

Translate the stored `ocr_text` to English. Requires OCR to have been run first.
Saves result to `attachments.ocr_translation`.

**Response `200`:**
```json
{ "id": 5, "ocrTranslation": "English translation of OCR text..." }
```

**curl:**
```bash
curl -X POST https://<host>/api/attachments/5/ocr/translate \
  -H "Authorization: Bearer $TOKEN"
```

---

## Stats

### GET /api/stats

Aggregate counts for the whole system or a single dataset.

**Query params:**

| Param | Type | Description |
|---|---|---|
| `dataset_id` | integer | Scope to a dataset |

**Response `200`:**
```json
{
  "total_emails": 5421,
  "total_threads": 312,
  "total_entities": 18400,
  "total_identities": 87,
  "duplicate_emails": 142,
  "total_attachments": 203,
  "languages_detected": 2
}
```

**curl:**
```bash
curl "https://<host>/api/stats?dataset_id=1" -H "Authorization: Bearer $TOKEN"
```

---

### GET /api/stats/timeline

Email volume over time.

**Query params:**

| Param | Values | Description |
|---|---|---|
| `dataset_id` | integer | Scope to a dataset |
| `granularity` | `"day"`, `"week"`, `"month"` | Time bucket size (default `"month"`) |

**Response `200`:** `[{ "date": "2022-12-01", "count": 143 }, ...]`

**curl:**
```bash
curl "https://<host>/api/stats/timeline?dataset_id=1&granularity=month" \
  -H "Authorization: Bearer $TOKEN"
```

---

### GET /api/stats/top-senders

Most prolific email senders.

**Query params:** `dataset_id`, `limit` (default `20`, max `100`).

**Response `200`:** `[{ "address": "...", "name": "...", "count": 142 }, ...]`

**curl:**
```bash
curl "https://<host>/api/stats/top-senders?dataset_id=1&limit=10" \
  -H "Authorization: Bearer $TOKEN"
```

---

### GET /api/stats/languages

Language distribution across emails.

**Query params:** `dataset_id`.

**Response `200`:** `[{ "language": "fa", "count": 4800 }, ...]`

**curl:**
```bash
curl "https://<host>/api/stats/languages?dataset_id=1" \
  -H "Authorization: Bearer $TOKEN"
```

---

### GET /api/stats/entity-types

Entity type distribution.

**Query params:** `dataset_id`.

**Response `200`:** `[{ "entity_type": "PERSON", "count": 1200 }, ...]`

**curl:**
```bash
curl "https://<host>/api/stats/entity-types?dataset_id=1" \
  -H "Authorization: Bearer $TOKEN"
```

---

## Upload

### POST /api/upload

Upload an email file (`.eml` or `.mbox`) and ingest it. Creates a new dataset row
automatically.

**Request:** `multipart/form-data`

| Field | Type | Description |
|---|---|---|
| `file` | file | `.mbox` or `.eml` file (max 100 MB) |
| `dataset_name` | string | Optional name for the new dataset |

**Response `200`:**
```json
{
  "dataset_id": 4,
  "emails_ingested": 142,
  "duplicates_skipped": 3,
  "threads_created": 27,
  "message": "Successfully ingested 142 emails into dataset \"IFC Leak\" (3 duplicates skipped, 27 threads created)"
}
```

**curl:**
```bash
curl -X POST https://<host>/api/upload \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@ifc.mbox" \
  -F "dataset_name=IFC Leak"
```
