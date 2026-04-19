# Leak Data Reader PoC — Feature Reference

This document describes each major UI page: what it does, which API endpoints it calls,
and any AI operations it triggers. Page routes are defined in `artifacts/wikiran/src/App.tsx`.
API hooks come from `@workspace/api-client-react` (generated from the OpenAPI spec).

---

## Auth Gate (`AuthGate` component)

**Route:** Wraps all other routes.

Before rendering any page, `AuthGate` checks the `useAuth` hook:

- If no user → renders `<LoginPage />` (see below).
- If `user.mustChangePassword` → renders `<ChangePasswordModal />`.
- Otherwise → renders the main `<Router>` with all page routes.

---

## Login Page

**Route:** `/` (when unauthenticated)
**File:** `src/pages/login.tsx`

A simple form that collects username and password. On submit it calls
`POST /api/auth/login`. On success, the returned JWT and user object are stored in the
`AuthContext`. If `mustChangePassword` is `true`, the `ChangePasswordModal` intercepts
navigation.

**API calls:** `POST /api/auth/login`, `POST /api/auth/change-password`

---

## Overview Dashboard

**Route:** `/`
**File:** `src/pages/dashboard.tsx`

Displays aggregate statistics and charts for the selected dataset (or all datasets if none
is selected). The active dataset is held in `useAppState()` — a React context that
persists the `datasetId` selection across pages.

**Sections:**

| Section | Hook | API endpoint |
|---|---|---|
| Metric cards (emails, threads, entities, identities, duplicates, attachments) | `useGetStats` | `GET /api/stats` |
| Email volume timeline (area chart) | `useGetEmailTimeline` | `GET /api/stats/timeline?granularity=day` |
| Top senders (bar chart) | `useGetTopSenders` | `GET /api/stats/top-senders?limit=10` |
| Language breakdown (pie chart) | `useGetLanguageBreakdown` | `GET /api/stats/languages` |
| Entity type breakdown (bar chart) | `useGetEntityTypeBreakdown` | `GET /api/stats/entity-types` |

All queries include `dataset_id` when a dataset is selected. Charts use
[Recharts](https://recharts.org/) (`AreaChart`, `BarChart`, `PieChart`).

**AI calls:** None directly. Stats are computed from already-ingested data.

---

## Email Explorer

**Route:** `/emails`
**File:** `src/pages/emails.tsx`

The primary email browsing interface. A left sidebar contains filters; the main area shows
a paginated list of emails; clicking an email opens a detail panel (Sheet) on the right.

**Sidebar filters:**
- Dataset selector (from `useListDatasets` → `GET /api/datasets`)
- Text search (debounced, searches subject + body + translation + OCR)
- Sender filter
- Language filter
- Date range (from/to)
- Has attachments toggle
- Is junk toggle

**Email list:** `useListEmails` → `GET /api/emails` with all active filters. Each card
shows subject, sender, date, language badge, attachment indicator, and a body preview.

**Email detail panel (Sheet):**
- Full body text + `body_translation` tab (translated by `POST /api/emails/:id/translate`)
- Entities section: list of `{ entity_type, value, value_translation }` from
  `GET /api/emails/:id` response. "Translate all" button calls
  `POST /api/emails/:id/entities/translate`. Per-entity inline editing of
  `value_translation` calls `PATCH /api/entities/:id`. "Suggest" (sparkle icon)
  calls `POST /api/entities/:id/suggest` (gpt-5-mini).
- Attachments section: attachment records are embedded in the `useGetEmail` response
  (`email.attachments`). The detail view does not call a separate attachment list endpoint.
  Each attachment card offers: OCR (`POST /api/attachments/:id/ocr`, raw `fetch`),
  OCR translate (`POST /api/attachments/:id/ocr/translate`, raw `fetch`), and a
  download link (`GET /api/attachments/:id/file`).

**API calls:** `GET /api/datasets`, `GET /api/emails`, `GET /api/emails/:id`,
`POST /api/emails/:id/translate`, `POST /api/emails/:id/entities/translate`,
`PATCH /api/entities/:id`, `POST /api/entities/:id/suggest`,
`POST /api/attachments/:id/ocr`, `POST /api/attachments/:id/ocr/translate`,
`GET /api/attachments/:id/file`

**AI calls:** Email translation (gpt-5-mini), entity translation bulk (gpt-5-mini),
entity translation single suggest (gpt-5-mini), OCR cleanup + image OCR (gpt-4o)

---

## Threads

**Route:** `/threads`
**File:** `src/pages/threads.tsx`

Browse email threads. A paginated list shows thread subject, email count, participant
count, and date range. Clicking a thread opens a detail panel.

**Thread list:** `useListThreads` → `GET /api/threads?dataset_id=...`

**Thread detail panel:**
- Subject, date range, participant list
- Each email in the thread (chronological order) with body text and translation
- Collapsed attachment panel per email showing OCR text
- Deduplicated entity list for all emails in the thread
- "Analyze with AI" button → `POST /api/threads/:id/summarize` (gpt-4o). Shows the
  returned `summary`, `key_people`, `key_topics`, and `investigative_leads`.

**API calls:** `GET /api/threads`, `GET /api/threads/:id`, `POST /api/threads/:id/summarize`

**AI calls:** Thread analysis (gpt-4o) — triggered by the analyst clicking "Analyze"

---

## Entity Map

**Route:** `/entities`
**File:** `src/pages/entities.tsx`

Browse all extracted named entities with filtering by type and free-text search.
The page is read-only — no inline editing of translations is available here.
Per-entity translation editing and AI suggestion are provided in the **Email Explorer**
entity panel within the email detail sheet (see above).

**Filters:** Dataset selector, entity type (PERSON / ORG / EMAIL / PHONE / LOCATION /
DATE / IBAN / CRYPTO_* …), search field. Filters submit on form submission; pagination
resets to page 1 on new searches.

**Entity list:** `useListEntities` → `GET /api/entities` with active filters.
Paginated, 50 records per page. Displays entity value (with `value_translation` if
present), entity type badge, email ID reference, and confidence score.
Copy-to-clipboard button appears on hover.

**API calls:** `GET /api/datasets`, `GET /api/entities`

**AI calls:** None (this page makes no AI calls; entity translation happens in Email Explorer)

---

## Identity Clusters

**Route:** `/identities`
**File:** `src/pages/identities.tsx`

Manage person-level identity records built from sender analysis and AI extraction.

**Left sidebar filter:** Risk level (All / Critical / High / Medium / Low) wired to the
`risk_level` query parameter on `GET /api/identities`.

**Identity cards:** Grid of cards showing canonical name, risk level badge, email count,
organization affiliations (first two + count), and aliases. Sorted by `email_count`
descending.

**Identity list:** `useListIdentities` → `GET /api/identities?dataset_id=...&risk_level=...`

**Header buttons:**
- **"Extract with AI"** → `POST /api/identities/extract` with the current `dataset_id`.
  Sends up to 500 emails to GPT-4o, upserts the returned clusters. Shows created/updated
  counts in a toast notification.
- **"New Identity"** → Opens `IdentityFormDialog` in create mode.

**Identity detail panel (Sheet, opened on card click):**
- Full profile: name, aliases, email addresses, organizations, phone numbers, physical
  addresses, risk level, notes.
- Associated emails tab: fetched from `GET /api/identities/:id/emails`. Each row has a
  "View in Emails" link that navigates to `/emails` filtered by that email.
- Edit button (pencil icon): opens `IdentityFormDialog` in edit mode.

**IdentityFormDialog:**
- Create: `POST /api/identities` (201 response, invalidates identity list query)
- Edit: `PATCH /api/identities/:id` (invalidates identity list and detail panel)
- Form has 9 editable fields: canonical name, aliases (comma-separated), email addresses,
  phone numbers, physical addresses, organizations (JSON), risk level (select),
  dataset ID, notes.
- `useEffect` resets the form whenever the `identity` prop or `open` state changes,
  preventing stale values from a previous edit session.

**API calls:** `GET /api/datasets`, `GET /api/identities`, `POST /api/identities`,
`PATCH /api/identities/:id`, `POST /api/identities/extract`,
`GET /api/identities/:id/emails`

**AI calls:** Bulk identity extraction (gpt-4o) — triggered by "Extract with AI" button

---

## Crypto Wallets

**Route:** `/crypto`
**File:** `src/pages/crypto.tsx`

Detect and review cryptocurrency wallet addresses mentioned in emails.

**Summary cards:** `GET /api/crypto/summary` — counts by wallet type and unique addresses.

**Per-dataset breakdown table:** `GET /api/crypto/by-dataset` — dataset name, total
emails, unique addresses, keyword hits, top wallet addresses.

**Wallet list:** `GET /api/crypto/wallets` — all detected addresses grouped by address,
with occurrence metadata (email subject, sender, date, dataset).

**"Scan Dataset" button:** Calls `POST /api/crypto/scan` with the selected `dataset_id`.
Uses regex patterns in `lib/crypto.ts` (no AI). Results are stored in the `entities` table
with `CRYPTO_*` entity types and immediately reflected in the wallet list.

**Wallet types shown:** ETH, TRON, XMR, BTC (legacy + Bech32), LTC, XRP, DOGE, BCH, SOL.
Each has a color-coded badge defined in `TYPE_COLORS`.

**API calls:** `GET /api/crypto/summary`, `GET /api/crypto/by-dataset`,
`GET /api/crypto/wallets`, `POST /api/crypto/scan`

All four calls in this page are made via raw `fetch` wrapped in TanStack Query's
`useQuery`/`useMutation`, not via generated hooks from `@workspace/api-client-react`.

**AI calls:** None. Wallet detection is regex-based.

---

## Upload

**Route:** `/upload`
**File:** `src/pages/upload.tsx`

A drag-and-drop / file picker interface for uploading email archives. Accepts `.mbox`
and `.eml` files up to 100 MB.

The user optionally provides a dataset name before uploading. On submit, the file is sent
via `POST /api/upload` as `multipart/form-data`. The response shows ingestion counts
(emails, duplicates, threads).

**After upload:** The `datasets` query cache is invalidated via `useQueryClient().invalidateQueries`,
so the dataset selector in the sidebar and dashboard update automatically.

**API calls:** `GET /api/datasets`, `POST /api/upload`

**AI calls:** `extractEntitiesFromText` is called per non-duplicate email during ingest
(gpt-5-mini). This is automatic — the user does not trigger it explicitly.

---

## Shared Context and Navigation

### `AppStateProvider` / `useAppState`

Provides the `datasetId` state (currently selected dataset) to all pages. Changing the
dataset in any page's selector updates this context and causes all active queries to
re-fetch with the new `dataset_id`.

### `AppLayout` / `Sidebar`

The sidebar lists all page routes with icons and labels. It also shows the dataset selector
(populated from `GET /api/datasets`) at the top. The current user and a logout button are
shown at the bottom.

### Router

Routes are managed by [wouter](https://github.com/molefrog/wouter). The router base is
set from `import.meta.env.BASE_URL` (Vite's base path). Internal navigation uses
`useLocation` from wouter — not `window.location` or `react-router-dom`.
