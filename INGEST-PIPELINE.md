# Leak Data Reader PoC — Ingest Pipeline

This document traces the full path from file upload to indexed, searchable, entity-enriched
email records in the database.

---

## Pipeline Overview

```
 ┌───────────────────────────────────────────────────────────────────┐
 │  Client uploads .mbox or .eml file via POST /api/upload           │
 └─────────────────────────────┬─────────────────────────────────────┘
                               │
                               ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │  1. RECEIVE & PARSE                                                 │
 │     multer buffers file in memory (max 100 MB)                      │
 │     • .mbox  → splitMbox() → [raw message strings]                 │
 │     • .eml   → single raw message string                           │
 │     mailparser.simpleParser(raw) → ParsedEmail objects             │
 └─────────────────────────────┬───────────────────────────────────────┘
                               │  ParsedEmail[]
                               ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │  2. CREATE DATASET ROW                                              │
 │     INSERT INTO datasets (name, description)                       │
 └─────────────────────────────┬───────────────────────────────────────┘
                               │  dataset.id
                               ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │  3. DEDUPLICATION CHECK (per dataset)                              │
 │     Load all existing content_hash values for this dataset into    │
 │     a Set<string> in memory.                                       │
 └─────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼ (for each ParsedEmail)
 ┌─────────────────────────────────────────────────────────────────────┐
 │  4. HASH & INSERT EMAIL                                             │
 │     SHA-256(messageId + subject + bodyText + fromAddress)          │
 │     • hash in Set → is_duplicate = true                            │
 │     • hash new   → is_duplicate = false                            │
 │     detectLanguageSimple(bodyText) → language code                 │
 │     INSERT INTO emails (all fields)                                │
 └──────────────┬──────────────────────────────┬───────────────────────┘
                │                              │
                │ (non-duplicate, has attachments)
                ▼                              ▼
 ┌──────────────────────┐     ┌────────────────────────────────────────┐
 │  5. ATTACHMENTS      │     │  6. ENTITY EXTRACTION (AI)             │
 │  hashBuffer(content) │     │  extractEntitiesFromText(bodyText)     │
 │  INSERT INTO         │     │  → [{ entity_type, value, confidence}] │
 │  attachments         │     │  INSERT INTO entities                  │
 │  (binary in bytea)   │     │  (skipped if bodyText < 10 chars)      │
 └──────────────────────┘     └────────────────────────────────────────┘
                               │
                               ▼ (after all emails processed)
 ┌─────────────────────────────────────────────────────────────────────┐
 │  7. THREAD LINKING                                                  │
 │     buildThreads(datasetId)                                        │
 │     → fetch all emails with thread_id IS NULL                      │
 │     → normalize subject (strip Re:/Fwd:, lowercase, trim, 100 ch) │
 │     → group by normalized subject                                  │
 │     → INSERT INTO email_threads (subject, counts, date range)     │
 │     → UPDATE emails SET thread_id = <new thread id>               │
 └─────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │  8. IDENTITY CLUSTER UPDATE (lightweight pass)                      │
 │     updateIdentityClusters(datasetId)                              │
 │     → count emails per from_address                                │
 │     → skip addresses already in identities.email_addresses         │
 │     → INSERT new identity rows (one per unseen address)            │
 └─────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │  RESPONSE: { dataset_id, emails_ingested, duplicates_skipped,       │
 │              threads_created }                                      │
 └─────────────────────────────────────────────────────────────────────┘
```

---

## Stage-by-Stage Detail

### 1. Receive & Parse (`routes/upload.ts`)

The route uses [multer](https://github.com/expressjs/multer) with `memoryStorage` to buffer
the entire uploaded file in RAM before processing. The 100 MB limit is configured in the
multer options.

**mbox format detection:** If the file ends in `.mbox` or the first line begins with
`"From "`, `splitMbox()` is called to split the raw buffer on the Unix mbox separator
(`\nFrom `). Each segment is passed to `simpleParser` individually.

**eml format:** A file that is not an mbox is passed directly to `simpleParser` as a
`Buffer`. Any parse error returns HTTP 400.

**`ParsedEmail` interface** (defined in `lib/ingest.ts`):
```typescript
interface ParsedEmail {
  messageId?: string;
  subject?: string;
  fromAddress?: string;
  fromName?: string;
  toAddresses: string[];
  ccAddresses: string[];
  date?: Date;
  bodyText?: string;
  bodyHtml?: string;
  attachments: Array<{
    filename?: string;
    contentType?: string;
    sizeBytes?: number;
    content?: Buffer;
  }>;
}
```

### 2. Create Dataset Row

A single `INSERT INTO datasets` is executed before any email processing. The dataset
receives the `dataset_name` form field as its name, falling back to the original filename.
All subsequent rows reference this `dataset.id`.

### 3. Deduplication Check

Before the per-email loop, a single query loads all `content_hash` values for the target
dataset into a `Set<string>`. This avoids an individual SELECT per email, keeping the
deduplication check O(1) per email regardless of dataset size.

### 4. Hash & Insert Email

For each `ParsedEmail`:

1. **Content hash** — SHA-256 of the concatenation:
   `"<messageId>||<subject>||<bodyText>||<fromAddress>"`. The `||` separator prevents
   collisions between empty fields.
2. **Duplicate detection** — If the hash is already in the pre-loaded `Set`, `is_duplicate`
   is set to `true`. Either way the hash is added to the `Set` so duplicates within the
   same upload are also caught.
3. **Language detection** — `detectLanguageSimple()` applies Unicode range tests in order:
   - `[\u0600-\u06FF]` matched first → `"fa"`. Because this block covers both Persian and
     core Arabic characters, Arabic text will almost always be tagged `"fa"` as well.
   - `[\u0600-\u06FF\u0750-\u077F]` matched next → `"ar"`. In practice this branch is
     only reached for text consisting solely of Arabic Extended-A characters (U+0750–U+077F)
     with no core Arabic/Persian block characters.
   - `[\u0400-\u04FF]` → `"ru"` (Cyrillic)
   - `[\u4E00-\u9FFF]` → `"zh"` (Chinese)
   - Otherwise → `"en"`
4. **Insert** — A single `INSERT INTO emails` with all parsed fields. The row is inserted
   regardless of duplicate status (the `is_duplicate` flag marks it).

### 5. Attachments

For each attachment in the `ParsedEmail.attachments` array:

- `hashBuffer(content)` computes a SHA-256 of the raw `Buffer`.
- One `INSERT INTO attachments` stores `filename`, `contentType`, `sizeBytes`,
  and `contentHash`. **The raw binary content is not persisted** — the `file_data bytea`
  column exists in the schema but ingest does not write to it. The `POST /api/attachments/:id/ocr`
  route will return `422 "No file data stored"` for any attachment ingested via the standard
  pipeline.
- `is_duplicate` is hardcoded `false` at ingest time — cross-email duplicate detection
  happens at the thread level when `GET /api/threads/:id` is called (it skips repeating
  `ocr_text` for attachments with matching hashes).

**OCR and OCR translation are not run at ingest time.** They are on-demand: the analyst
manually triggers `POST /api/attachments/:id/ocr` and then
`POST /api/attachments/:id/ocr/translate` from the UI.

### 6. Entity Extraction (AI, inline)

For each non-duplicate email with `bodyText` longer than 10 characters, the ingest loop
calls `extractEntitiesFromText(bodyText)` from `lib/ai.ts`. This is a synchronous AI call
in the request handler — it blocks until the API returns.

The function sends the first 2,000 characters of the body to GPT-5-mini with a NER prompt
and parses the JSON response. On failure the error is swallowed (`catch (_) {}`) and the
email is still inserted without entities.

**Recognized entity types at ingest:** `PERSON`, `ORG`, `EMAIL`, `PHONE`, `LOCATION`,
`DATE`.

**Crypto wallet scanning is separate.** It is triggered manually from the Crypto Wallets
page via `POST /api/crypto/scan`. It uses regex patterns in `lib/crypto.ts`, not the AI.

### 7. Thread Linking (`buildThreads`)

After all emails are inserted, `buildThreads(datasetId)` runs:

1. Fetch all emails for the dataset where `thread_id IS NULL`.
2. Normalize subject: strip leading `Re:`, `Fwd:`, `Fwd:`, `AW:`, `FWD:` (case-insensitive),
   trim, lowercase, truncate to 100 characters.
3. Emails with no subject get the key `"__no_subject_<email.id>"` — each goes into its
   own thread.
4. For each subject group: `INSERT INTO email_threads` with the thread's subject, email
   count, participant count (union of all `from_address` and `to_addresses`), and date range.
5. `UPDATE emails SET thread_id = <thread.id>` for all emails in the group.

The return value is the count of newly created threads, which is included in the upload
response.

### 8. Identity Cluster Update (`updateIdentityClusters`)

A lightweight pass that creates one identity row per previously-unseen sender address:

1. Fetch all `from_address` / `from_name` pairs for the dataset.
2. Load all `email_addresses` arrays from existing `identities` rows for the dataset.
   Flatten them into a `Set<string>` of already-processed addresses.
3. For each new address: derive `canonical_name` from the **first observed** `from_name` for
   that address (insertion order in a `Set`), or the local part of the email address if no
   name was ever seen.
4. `INSERT INTO identities` with `canonicalName`, `aliases` (all observed display names),
   `emailAddresses: [address]`, and `emailCount`.

This is a basic structural extraction. The AI-powered `POST /api/identities/extract`
endpoint (triggered manually) provides a far richer extraction by reading email content.

---

## Async Patterns (and why they are absent)

The ingest pipeline runs **entirely synchronously within the HTTP request handler**. There
is no job queue, background worker, or message broker. This means:

- Large mbox files (thousands of emails) will result in a long-running HTTP request.
- AI entity extraction failures are silently ignored to keep the pipeline moving.
- The browser upload UI may time out for very large files.

If dataset size grows to tens of thousands of emails, the AI extraction call per email
should be moved to a background queue (e.g. BullMQ with Redis) and triggered post-ingest.

---

## Relevant Source Files

| File | Role |
|---|---|
| `artifacts/api-server/src/routes/upload.ts` | mbox/eml parsing, dataset creation, calls `ingestParsedEmails` |
| `artifacts/api-server/src/lib/ingest.ts` | Core ingest loop: dedup, insert, entity extraction, thread linking, identity update |
| `artifacts/api-server/src/lib/ai.ts` | `extractEntitiesFromText()` — NER AI call |
| `artifacts/api-server/src/lib/crypto.ts` | Regex wallet scanner (not part of ingest; triggered separately) |
| `lib/db/src/schema/emails.ts` | emails table definition |
| `lib/db/src/schema/attachments.ts` | attachments table definition |
| `lib/db/src/schema/entities.ts` | entities table definition |
| `lib/db/src/schema/threads.ts` | email_threads table definition |
| `lib/db/src/schema/identities.ts` | identities table definition |
