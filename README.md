# Leak Data Reader — Proof of Concept

A full-stack OSINT platform for ingesting, enriching, and investigating publicly available
leaked email corpora. Built as a personal learning project exploring practical tooling for
open-source intelligence (OSINT), publicly available information (PAI), and
commercially available information (CAI) — demonstrating how analysts can work through
large email leak datasets by searching, threading, extracting entities, clustering
identities, and surfacing cryptocurrency traces.

---

## A note on data source selection

This proof of concept is built around data published on **[wikiran.org](https://wikiran.org)**
— a site that hosts leaked email archives from Iranian government ministries, state-owned
enterprises, sanctioned financial institutions, and affiliated front companies.

**The choice of this dataset has nothing to do with any animus toward Iran or its people.**
Iranian email leaks were selected for a specific and practical reason: the content does not
contain information that would be considered classified or sensitive in Western or
English-speaking countries. This made it legally unambiguous material to use for building
and demonstrating forensic tooling in a government-contractor context.

We want to be direct about this: the Iranian people have suffered enormously under the
regime whose internal communications appear in these leaks. The entities documented here —
sanctions-evasion networks, currency manipulation operations, filtering committees, and
oil-revenue front companies — represent the machinery of that regime, not its citizens.
This tool exists to make that kind of institutional behaviour easier to trace and
document, not to cast any shadow on Iran's people or culture.

The Leak Data Reader framework is fully source-agnostic. The same codebase can be pointed
at any publicly available leaked email corpus regardless of national origin.

---

## What it does

| Feature | Description |
|---|---|
| **Ingest** | Upload `.mbox` or `.eml` files; automatic deduplication, language detection, thread linking |
| **Email Explorer** | Full-text search across subjects, bodies, translations, and OCR text; filterable by dataset, date, language, sender |
| **Translation** | On-demand AI translation of email bodies and attachment OCR text (Persian/Arabic → English) |
| **Entity Extraction** | AI-powered named entity recognition (persons, orgs, phone numbers, IBANs, locations); manual translation editing |
| **Thread View** | Emails grouped by subject; AI-generated thread summaries with key people, topics, and investigative leads |
| **Crypto Wallet Scanner** | Regex-based detection of BTC, ETH, TRON, XMR, LTC, XRP, DOGE, BCH, SOL addresses across all emails |
| **Identity Clusters** | Sender-based identity records enriched by AI; editable profiles with risk levels and analyst notes |
| **Attachment OCR** | On-demand OCR and translation for image and PDF attachments via GPT-4o vision |
| **Statistics Dashboard** | Dataset-level summaries: email counts, thread counts, entity breakdowns, language distribution |

---

## Sample datasets

The repository ships with four synthetic sample datasets modelled on the types of
organisations documented in WikiRan.org leaks. These give the tool something to work
with out of the box without requiring a full data import:

| Dataset | Type | Description |
|---|---|---|
| **Sepehr Naft Trading** (سپهر نفت تجارت) | Oil & energy trading | Internal correspondence of a mid-sized petroleum trading firm; inter-office memos, shipping coordination, financial transfers |
| **Kaveh Maritime Services** (خدمات دریانوردی کاوه) | Shipping & logistics | Vessel scheduling, port agent communications, crew documentation — the kind of company used to move sanctioned cargo |
| **Aryan Currency Exchange** (صرافی آریان) | Currency exchange | Transaction confirmations, correspondent banking messages, hawala-style transfer records |
| **Alborz Industrial Supply** (تأمین صنعتی البرز) | Industrial procurement | Procurement orders, supplier invoices, customs documentation for dual-use goods |

These datasets are seeded automatically. Real WikiRan.org datasets (and any other `.mbox`
or `.eml` archive) can be added via the Upload page or the seed scripts in `scripts/`.

---

## Architecture overview

```
┌─────────────────────────────────────────┐
│  React + Vite frontend (wikiran/)       │
│  TanStack Query · shadcn/ui · Tailwind  │
└────────────────────┬────────────────────┘
                     │ HTTP
┌────────────────────▼────────────────────┐
│  Express 5 API server (api-server/)     │
│  Drizzle ORM · OpenAI proxy · JWT auth  │
└────────────────────┬────────────────────┘
                     │
         ┌───────────▼───────────┐
         │  PostgreSQL database  │
         └───────────────────────┘
```

pnpm monorepo with three runtime artifacts (`wikiran`, `api-server`, `mockup-sandbox`)
and four shared library packages (`db`, `api-spec`, `api-client-react`, `api-zod`).

AI calls use the Replit AI Integrations proxy (OpenAI-compatible, no personal API key
required). Models: `gpt-4o` for thread analysis, identity clustering, and OCR;
`gpt-5-mini` for translation and entity extraction.

---

## Getting started

```bash
# Install dependencies
pnpm install

# Push database schema
pnpm --filter @workspace/db run push

# Seed sample datasets
pnpm --filter @workspace/scripts run seed

# Start all services (or use Replit workflows)
pnpm --filter @workspace/api-server run dev
pnpm --filter @workspace/wikiran run dev
```

Default login after seeding: `wikiran_agent` / `wikiran2024!` (you will be prompted to
change the password on first login).

---

## Developer documentation

All technical documentation lives in `docs/`:

| Document | What it covers |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Monorepo layout, service ports, shared packages, data flow, auth layer, AI integration, environment variables |
| [`docs/DATA-MODEL.md`](docs/DATA-MODEL.md) | Full schema reference — all nine tables with column types, nullability, indexes, and design rationale |
| [`docs/API-REFERENCE.md`](docs/API-REFERENCE.md) | All 35 API endpoints with parameters, request/response shapes, and curl examples |
| [`docs/INGEST-PIPELINE.md`](docs/INGEST-PIPELINE.md) | Step-by-step walkthrough from file upload to indexed, entity-enriched records |
| [`docs/FEATURES.md`](docs/FEATURES.md) | Per-page feature reference: what each UI page does, which endpoints it calls, which AI functions it triggers |
| [`docs/EXTENDING.md`](docs/EXTENDING.md) | How-to guides: adding entity types, AI functions, new leak sources, and frontend pages |

Additional reference files in `docs/`:

- [`docs/WIKIRAN-ORG-DATA-SPEC.md`](docs/WIKIRAN-ORG-DATA-SPEC.md) — Wire format specification for the wikiran.org public API
- [`docs/WIKIRAN-ORG-DATASETS.json`](docs/WIKIRAN-ORG-DATASETS.json) — Machine-readable catalogue of all available wikiran.org datasets

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React 19, Vite 7, TanStack Query 5, shadcn/ui, Tailwind 4, wouter |
| Backend | Node.js 24, Express 5, TypeScript |
| Database | PostgreSQL, Drizzle ORM, drizzle-zod |
| AI | OpenAI SDK via Replit AI Integrations proxy (gpt-4o, gpt-5-mini) |
| Auth | JWT (7-day), bcrypt, localStorage token storage |
| API codegen | Orval (OpenAPI → TanStack Query hooks + Zod schemas) |
| Monorepo | pnpm workspaces, TypeScript project references |
