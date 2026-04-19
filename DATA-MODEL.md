# Leak Data Reader PoC — Database Data Model

All schema definitions live in `lib/db/src/schema/`. The ORM is
[Drizzle ORM](https://orm.drizzle.team/) with a PostgreSQL backend.
Run `pnpm --filter @workspace/db run push` to sync schema changes to the database.

---

## Entity-Relationship Diagram

```
datasets
  │id (PK)
  │
  ├─── emails (dataset_id FK)
  │      │id (PK)
  │      │thread_id FK ──────► email_threads (id PK)
  │      │                        │dataset_id FK → datasets
  │      │
  │      ├─── attachments (email_id FK)
  │      │      │id (PK)
  │      │
  │      └─── entities (email_id FK)
  │             │id (PK)
  │
  └─── identities (dataset_id FK)
         │id (PK)

users (standalone — no FK relationships)

conversations
  │id (PK)
  │
  └─── messages (conversation_id FK, cascade delete)
         │id (PK)

  [conversations and messages are scaffold-generated tables;
   not connected to the application data model above]
```

---

## Table Definitions

### `datasets`

One row per imported email collection. Created automatically on file upload or by the seed
script when pulling from wikiran.org.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | serial PK | No | Auto-increment |
| `name` | text | No | Human-readable dataset name (e.g. "IFC Leak 2024") |
| `description` | text | Yes | Free-text description; auto-set to `"Uploaded from <filename>"` on upload |
| `created_at` | timestamp | No | Row creation time; default `now()` |

No indexes beyond the PK. All other tables reference `datasets.id`.

---

### `emails`

Core table. One row per parsed email message.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | serial PK | No | |
| `dataset_id` | integer FK → datasets | No | Which dataset this email belongs to |
| `thread_id` | integer FK → email_threads | Yes | Assigned during thread-linking; null until `buildThreads()` runs |
| `message_id` | text | Yes | RFC 2822 Message-ID header value |
| `subject` | text | Yes | Email subject line |
| `from_address` | text | Yes | Sender email address (bare address, not display name) |
| `from_name` | text | Yes | Sender display name |
| `to_addresses` | jsonb `string[]` | Yes | List of recipient addresses; default `[]` |
| `cc_addresses` | jsonb `string[]` | Yes | List of CC addresses; default `[]` |
| `date` | timestamp | Yes | Email send date parsed from headers |
| `body_text` | text | Yes | Plain-text body |
| `body_html` | text | Yes | HTML body (stored but not displayed in UI) |
| `body_translation` | text | Yes | English translation produced by `translateEmailBody()` |
| `language` | text | Yes | Detected language code: `"fa"`, `"en"`, `"ar"`, `"ru"`, `"zh"` |
| `has_attachments` | boolean | No | `true` if `attachment_count > 0`; default `false` |
| `attachment_count` | integer | No | Number of attachments; default `0` |
| `content_hash` | text | Yes | SHA-256 of `messageId + subject + bodyText + fromAddress`; used for dedup |
| `is_duplicate` | boolean | No | `true` if `content_hash` already existed in this dataset at ingest time |
| `is_junk` | boolean | No | Manual junk flag (not auto-set by ingest); default `false` |
| `created_at` | timestamp | No | Row creation time |

**Indexes:**

| Index name | Column | Purpose |
|---|---|---|
| `idx_emails_dataset` | `dataset_id` | Filter by dataset |
| `idx_emails_thread` | `thread_id` | Join to threads |
| `idx_emails_from` | `from_address` | Sender filter / identity lookup |
| `idx_emails_hash` | `content_hash` | Duplicate detection |
| `idx_emails_date` | `date` | Timeline sorting and date range queries |

**Why JSONB for arrays?** `to_addresses` and `cc_addresses` are stored as `jsonb string[]`
rather than a junction table because email recipients are append-only data that is always
read as a unit. The tradeoff is that you cannot efficiently query "all emails addressed to
X" with a simple indexed lookup — the API uses `ANY(...)` array operators for this.

---

### `email_threads`

Groups emails that share the same (normalized) subject line within a dataset.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | serial PK | No | |
| `dataset_id` | integer FK → datasets | Yes | Dataset the thread was built from |
| `subject` | text | Yes | Subject of the first email in the thread |
| `email_count` | integer | No | Number of emails in the thread; default `0` |
| `participant_count` | integer | No | Number of unique sender/recipient addresses |
| `first_date` | timestamp | Yes | Earliest email date in thread |
| `last_date` | timestamp | Yes | Latest email date in thread |
| `summary` | text | Yes | Short summary produced by `analyzeThread()` |
| `analysis_result` | jsonb | Yes | Full AI analysis: `{ summary, key_people, key_topics, investigative_leads }` |
| `created_at` | timestamp | No | Row creation time |

No explicit indexes beyond the PK. `emails.thread_id` carries its own index.

**Thread-linking algorithm:** Subject normalization strips leading `Re:`, `Fwd:`, `AW:`,
`FWD:` prefixes, trims whitespace, lowercases, and truncates to 100 characters. Emails
with the same normalized subject in the same dataset are grouped into a thread. Emails
with no subject each get a unique synthetic key `__no_subject_<email.id>`.

---

### `attachments`

One row per file attached to an email. Metadata and hash are stored during ingest; the
`file_data` bytea column exists for binary storage but is **not populated by the standard
upload pipeline** (see INGEST-PIPELINE.md §Attachments).

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | serial PK | No | |
| `email_id` | integer FK → emails | No | Parent email |
| `filename` | text | Yes | Original filename from the MIME part |
| `content_type` | text | Yes | MIME type (e.g. `"application/pdf"`, `"image/jpeg"`) |
| `size_bytes` | integer | Yes | File size in bytes |
| `content_hash` | text | Yes | SHA-256 of raw binary content; used for cross-email dedup within a thread |
| `is_duplicate` | boolean | No | `true` if this attachment is a copy of one already stored; default `false` |
| `ocr_text` | text | Yes | OCR/extraction output produced by `POST /attachments/:id/ocr` |
| `ocr_translation` | text | Yes | English translation of `ocr_text` produced by `translateOcrText()` |
| `file_data` | bytea | Yes | Raw binary file content |

**Custom `bytea` type:** Drizzle does not ship a built-in `bytea` column type. `lib/db/src/schema/attachments.ts` defines a custom type that handles the PostgreSQL hex-escape format (`\\x...`) on read.

---

### `entities`

Named entities extracted from email body text by `extractEntitiesFromText()` during ingest.
Also used to store crypto wallet addresses found by `scanForWallets()`.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | serial PK | No | |
| `email_id` | integer FK → emails | No | Email the entity was extracted from |
| `entity_type` | text | No | `PERSON`, `ORG`, `EMAIL`, `PHONE`, `LOCATION`, `DATE`, or `CRYPTO_*` |
| `value` | text | No | Raw extracted value |
| `confidence` | real | Yes | Extraction confidence score 0.0–1.0 |
| `value_translation` | text | Yes | English/romanized translation produced by `translateEntities()` |

**Indexes:**

| Index name | Column | Purpose |
|---|---|---|
| `idx_entities_email` | `email_id` | Join to email |
| `idx_entities_type` | `entity_type` | Filter by type |
| `idx_entities_value` | `value` | Search by raw value |

**Entity types used by the crypto scanner (`lib/crypto.ts`):**

| Entity type | Currency |
|---|---|
| `CRYPTO_ETH` | Ethereum / ERC-20 |
| `CRYPTO_TRX` | TRON / TRC-20 |
| `CRYPTO_XMR` | Monero |
| `CRYPTO_BTC_LEGACY` | Bitcoin (legacy P2PKH/P2SH) |
| `CRYPTO_BTC_BECH32` | Bitcoin (Bech32 / native SegWit) |
| `CRYPTO_LTC` | Litecoin |
| `CRYPTO_XRP` | XRP / Ripple |
| `CRYPTO_DOGE` | Dogecoin |
| `CRYPTO_BCH` | Bitcoin Cash |
| `CRYPTO_SOL` | Solana |
| `CRYPTO_KEYWORD` | Email contains crypto keywords but no wallet address |

---

### `identities`

Analyst-curated and AI-extracted person records. One row per distinct individual within a
dataset (or across datasets once the cross-dataset merge feature is implemented).

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | serial PK | No | |
| `dataset_id` | integer FK → datasets | Yes | Source dataset; null for cross-dataset records |
| `canonical_name` | text | Yes | Most complete/formal name (romanized if originally Farsi) |
| `aliases` | jsonb `string[]` | Yes | Other names, usernames, or titles; default `[]` |
| `email_addresses` | jsonb `string[]` | Yes | All email addresses associated with this person; default `[]` |
| `email_count` | integer | No | Number of emails sent by any address in `email_addresses`; default `0` |
| `phone_numbers` | jsonb `string[]` | Yes | Phone numbers; default `[]` |
| `physical_addresses` | jsonb `string[]` | Yes | Physical addresses; default `[]` |
| `organizations` | jsonb `OrgAffiliation[]` | Yes | `[{ name: string, role: string }]`; default `[]` |
| `risk_level` | text | Yes | One of `"critical"`, `"high"`, `"medium"`, `"low"` |
| `notes` | text | Yes | Free-text analyst notes |

**Indexes:**

| Index name | Column | Purpose |
|---|---|---|
| `idx_identities_dataset` | `dataset_id` | Filter by dataset |

**`OrgAffiliation` type:** Defined as a TypeScript interface in `lib/db/src/schema/identities.ts`
and stored as a JSONB array. Each element is `{ name: string; role: string }`.

**Why JSONB for arrays?** Aliases, email addresses, phone numbers, and organizations change
as a unit during AI extraction and analyst editing. Using JSONB avoids N+1 inserts and
simplifies the upsert logic in `POST /identities/extract`.

---

### `users`

Application user accounts. Currently supports username/password auth only.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | serial PK | No | |
| `username` | text | No | Unique login name |
| `password_hash` | text | No | bcrypt hash (10 rounds) |
| `must_change_password` | boolean | No | Forces password change on first login; default `true` |
| `created_at` | timestamp | No | Row creation time |

No indexes beyond the PK and the unique constraint on `username`. The seed script creates
the initial `wikiran_agent` user.

---

### `conversations`

Scaffold-generated table for a chat/conversation feature. Not yet used by any feature
routes or frontend pages — present in the schema from the Replit project template.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | serial PK | No | |
| `title` | text | No | Conversation title |
| `created_at` | timestamp with timezone | No | Row creation time; default `now()` |

---

### `messages`

Scaffold-generated table for chat messages associated with a `conversations` row. Cascade-deletes
when the parent conversation is deleted. Not yet used by any feature routes in this application.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | serial PK | No | |
| `conversation_id` | integer FK → conversations | No | Parent conversation; cascade delete |
| `role` | text | No | Speaker role (e.g. `"user"`, `"assistant"`) |
| `content` | text | No | Message content |
| `created_at` | timestamp with timezone | No | Row creation time |

---

## Design Decisions

**JSONB for array columns** — `to_addresses`, `cc_addresses`, `aliases`, `email_addresses`,
`phone_numbers`, `physical_addresses`, and `organizations` are all stored as JSONB rather
than normalized junction tables. The rationale is that these arrays are always read and
written as a unit, the individual elements are rarely queried by value (and when they are,
`ANY(...)` or `::text ILIKE` suffices), and JSONB avoids schema migrations as the data
model evolves. The tradeoff is that joins across values (e.g. "find all identities that
share an email address") require a table scan or a GIN index.

**Content-hash deduplication** — `emails.content_hash` is a SHA-256 digest of
`messageId + subject + bodyText + fromAddress`. At ingest time, existing hashes for the
dataset are pre-loaded into a `Set` in memory; any new email whose hash is already in the
set is marked `is_duplicate = true` but still inserted. This preserves the full record
count while allowing downstream queries to filter duplicates.

**Thread linking by subject normalization** — The threading algorithm is intentionally
simple: strip reply/forward prefixes and group by lowercase subject. This is fast enough
to run synchronously at ingest time for datasets of a few thousand emails. For large
datasets (tens of thousands), the algorithm may create many single-email "threads". A
`References:` / `In-Reply-To:` header-based algorithm would be more accurate but requires
an additional pass.

**No soft deletes** — There are no `deleted_at` columns. Data is permanent once ingested.
The only way to remove data is direct SQL against the PostgreSQL database.
