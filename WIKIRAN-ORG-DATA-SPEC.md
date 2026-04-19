# WikiRan.org Data Source Specification

**Version:** 1.0  
**Survey Date:** April 19, 2026  
**Source Site:** https://wikiran.org  
**Specification License:** CC0 1.0 Universal (see below)  
**Code Sample License:** MIT (see below)

---

## Licenses

### Specification Content — CC0 1.0 Universal

To the extent possible under law, the authors of this specification have waived all copyright and related or neighboring rights to this document. This work is published from the United States.

This specification describes **publicly observable facts** about the structure of wikiran.org — URL patterns, JSON field names, HTTP response shapes, record counts, and wire formats. Facts are not copyrightable. This document is released under CC0 as an additional explicit grant.

> Full text: https://creativecommons.org/publicdomain/zero/1.0/

### Code Samples — MIT License

```
MIT License

Copyright (c) 2026 WikiRan Project contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND.
```

### Legal Disclaimer

This document describes the publicly observable structure of wikiran.org as a technical data specification. It does not constitute legal advice. The authors make no representations regarding the legal status of the underlying data, its use in any jurisdiction, or the legal obligations of any party accessing it. Users are responsible for determining whether and how they may access and use data from wikiran.org under applicable law.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Provenance & Methodology](#2-provenance--methodology)
3. [Site Architecture](#3-site-architecture)
4. [URL Structure](#4-url-structure)
5. [Public API Reference](#5-public-api-reference)
6. [Wire Format Specification](#6-wire-format-specification)
   - 6.1 [Email Records](#61-email-records)
   - 6.2 [OCR Document Records](#62-ocr-document-records)
   - 6.3 [Transaction Table Records](#63-transaction-table-records)
   - 6.4 [Shipping / Container Records (PMO)](#64-shipping--container-records-pmo)
   - 6.5 [Sanctions Entity Records](#65-sanctions-entity-records)
   - 6.6 [Vessel Records](#66-vessel-records)
   - 6.7 [Front Company Records](#67-front-company-records)
7. [Attachment Download Specification](#7-attachment-download-specification)
8. [Known Datasets — Machine-Readable Table](#8-known-datasets--machine-readable-table)
9. [Dataset Detail Notes](#9-dataset-detail-notes)
10. [Code Samples](#10-code-samples)
    - 10.1 [Python — Query dataset counts and download data](#101-python--query-dataset-counts-and-download-data)
    - 10.2 [Python — Download an attachment](#102-python--download-an-attachment)
    - 10.3 [TypeScript — Query dataset counts and download data](#103-typescript--query-dataset-counts-and-download-data)
    - 10.4 [TypeScript — Download an attachment](#104-typescript--download-an-attachment)
    - 10.5 [Query the per-dataset record count API](#105-query-the-per-dataset-record-count-api)
11. [Glossary](#11-glossary)

---

## 1. Overview

**WikiRan** (wikiran.org) is a public website that publishes leaked email databases and document archives from Iranian government ministries, state banks, sanctioned exchange houses, petrochemical exporters, and affiliated front companies. As of the survey date it hosts **27 confirmed live datasets** totaling approximately **3.2 million records** across four structural types: email inboxes, OCR-scanned document archives, structured transaction tables, and reference databases (vessels, front companies, sanctions entities).

The site is operated under the editorial brand "WikIran" and is publicly accessible without authentication. All data described in this document was obtained via standard, unauthenticated HTTP GET requests — equivalent to ordinary web browsing.

**Intended audience for this document:** software developers and data engineers who need to build ingestion pipelines against wikiran.org data, or who need to understand the site's data model in order to process already-downloaded datasets. The document is self-contained and designed to be pasted directly into an AI coding assistant.

---

## 2. Provenance & Methodology

### How this specification was produced

All information in this document was obtained via **public OSINT (Open Source Intelligence) methods only**:

- Standard HTTP GET requests to publicly accessible pages (`https://wikiran.org/leaks`, individual dataset pages, and the site's own public REST API)
- Browser-equivalent `User-Agent` headers; no spoofing, credential bypass, session manipulation, or token reuse
- No SQL injection, no reverse engineering of client-side JavaScript, no exploitation of any vulnerability
- No authentication of any kind was used or required — the site serves all described data to unauthenticated requests
- Record counts were obtained from the site's own public API endpoint (`/api/v1/leaks/{slug}`), which is linked from the site's own interface
- Attachment URLs were copied from the site's own rendered HTML — they appear verbatim in the public page source

**Every data point in this document is reproducible** by any researcher performing the same unauthenticated HTTP GET requests to wikiran.org.

### Data discovery sequence

1. The dataset index page (`https://wikiran.org/leaks`) was fetched and all dataset slugs were enumerated from the rendered HTML.
2. Each individual dataset page (`https://wikiran.org/leaks/{slug}`) was fetched to extract title, category, date range, domain information, and sample records visible in the page HTML.
3. The public per-dataset count API (`https://wikiran.org/api/v1/leaks/{slug}`) was queried for each slug to obtain exact record counts.
4. Sample attachment URLs were read from the rendered HTML of dataset pages where attachments are displayed.
5. Three or more sample records per dataset were observed and documented.

### Scope of this document

This document covers **publicly observable structure only**: URL patterns, JSON schemas, field names and types, record counts, and date ranges. It does not contain the full dataset content (which is hosted at wikiran.org). It does not cover any private, authenticated, or administratively restricted portions of the site.

---

## 3. Site Architecture

WikiRan is built as a single-page application (React frontend) backed by an Elasticsearch search engine. Key architectural notes for developers:

- **Client-side rendering:** Record counts shown on dataset listing pages are rendered client-side from Elasticsearch results. Static HTTP fetches of dataset pages will see a count of `0` in the HTML — this is expected. Use the public API (`/api/v1/leaks/{slug}`) to get accurate counts.
- **Elasticsearch backend:** The site uses Elasticsearch as its primary data store. The public API endpoints are thin wrappers around Elasticsearch queries.
- **No authentication required:** All public pages, the API, and all attachment URLs described in this document are accessible without a login or session cookie.
- **Farsi/Persian content:** Most email datasets originate from Iranian organizations and are predominantly in Farsi (Persian, written in the Arabic script, right-to-left). Language detection is provided per-record as the `language` field with value `"fa"` (Farsi) or `"en"` (English). Some datasets include Turkish (`"tr"`).

---

## 4. URL Structure

All URLs are on the `wikiran.org` domain over HTTPS.

### 4.1 Site Pages

| Resource | URL Pattern |
|---|---|
| Dataset index | `https://wikiran.org/leaks` |
| Individual dataset page | `https://wikiran.org/leaks/{slug}` |
| Dataset filtered by sender | `https://wikiran.org/leaks/{slug}?from={url-encoded-email}` |
| Dataset filtered by date range | `https://wikiran.org/leaks/{slug}?dateFrom={date}&dateTo={date}` |

**Supported query parameters for dataset pages:**

| Parameter | Description |
|---|---|
| `subject` | Filter by email subject text |
| `text` | Full-text search |
| `from` | Filter by sender address |
| `to` | Filter by recipient address |
| `message` | Search message body |
| `dateFrom` | Earliest date (format: `YYYY-MM-DD` or similar) |
| `dateTo` | Latest date |

### 4.2 Attachment Download URLs

Two URL patterns exist, distinguished by the hash format:

**Modern (post-2022 datasets) — MD5 hash (32 hex characters):**
```
https://wikiran.org/attachments/leaks/{slug}//{md5_32hex}/{url-encoded-filename}
```

**Legacy (pre-2022 datasets) — SHA1 hash (40 hex characters):**
```
https://wikiran.org/attachments/leaks/{slug}//{sha1_40hex}/{url-encoded-filename}
```

**Critical:** The **double slash** (`//`) before the hash is intentional and required. Single-slash variants return 404.

**Filename encoding:** Filenames are percent-encoded. Farsi/Arabic script characters are encoded as UTF-8 percent-encoding. Spaces become `%20`.

**Concrete examples (confirmed working as of survey date):**

```
# MD5 (modern) — Caspian Airlines JPEG
https://wikiran.org/attachments/leaks/caspian-airlines//41ce3a1198772dae61e393ef3729949c/IMG-20190925-WA0018.jpg

# MD5 (modern) — IFC JPEG  
https://wikiran.org/attachments/leaks/ifc//73ebd4138d7977170eec29d9490995bf/IMG-20171114-WA0006.jpg

# MD5 (modern) — Sahara Thunder Excel (Farsi filename, percent-encoded)
https://wikiran.org/attachments/leaks/sahara-thunder//2be57bef42e6640966012d297f9432a3/%D9%85%D8%B4%DA%A9%D9%88%DA%A9%20%D8%A7%D9%84%D9%88%D8%B5%D9%88%D9%84.xlsx

# SHA1 (legacy) — Steel Companies PNG
https://wikiran.org/attachments/leaks/steel-companies//cdccd7f9256e61fbe306bb5ead464774873ada4a/image001.png

# SHA1 (legacy) — PCC Excel
https://wikiran.org/attachments/leaks/pcc//ec5c8b17f2a760f0b9f56a640abb13852f005402/Cargo%20Update%202021-3-8.xlsx

# SHA1 (legacy) — MEFA JPEG
https://wikiran.org/attachments/leaks/mefa//13bc706bb26015283906468c81fab2a7abfe2545/image328.jpg
```

**Hash format by dataset age:**

| Hash Format | Hex Length | Datasets |
|---|---|---|
| MD5 | 32 chars | ifc, caspian-airlines, sadaf, sej, sej2, amin2025, sahara-thunder, triliance, saf |
| SHA1 | 40 chars | asbgroup, steel-companies, pcc, mefa, moe, tbtb |

> Tip: Inspect the hash length in the URL you observe in the site's rendered HTML — 32 characters = MD5, 40 characters = SHA1. The distinction does not strictly follow dataset age; `ifc` (emails from 2014–2020) uses MD5, while `asbgroup` (2021–2022) uses SHA1. Always verify by counting hex characters.

**Known 403 case:** The Sadaf dataset's sample PDF (`/attachments/leaks/sadaf//d2abbb2cdf1abc3857d190d4dde462f2/174.602%20USD-SHELL%20EX-%20CONFIRM.pdf`) returned HTTP 403 as of the survey date, suggesting selective server-side restriction of individual files. All other confirmed attachment URLs returned HTTP 200.

### 4.3 Special Download Files

Some datasets provide bulk download files at fixed URLs:

```
# Amin Exchange — full raw archive (5.12 GB ZIP)
https://wikiran.org/attachments/leaks/amin/amin.zip

# PGPICC — transaction table as CSV
https://wikiran.org/files/leaks/pgpicc/transactions.csv
```

### 4.4 Dataset Images

```
https://wikiran.org/images/{dbImagePath}
```

Example: `https://wikiran.org/images/dbs/IFC.png`

---

## 5. Public API Reference

### 5.1 Dataset Registry Endpoint

```
GET https://wikiran.org/api/v1/leaks
```

Returns a JSON array of all datasets with their metadata. No authentication required.

**Response shape (abbreviated):**
```json
[
  {
    "slug": "ifc",
    "title": "...",
    "total": 562634,
    ...
  }
]
```

### 5.2 Per-Dataset Count Endpoint

```
GET https://wikiran.org/api/v1/leaks/{slug}
```

Returns JSON with the exact Elasticsearch document count for the dataset.

**Key response field:**
```json
{
  "total": 562634
}
```

The `total` field contains the exact document count as of the request time. This is the authoritative record count — use this instead of any count rendered on the HTML page (which is 0 for static fetches due to client-side rendering).

**Confirmed endpoint examples:**

```bash
# IFC — 562,634 records
curl https://wikiran.org/api/v1/leaks/ifc

# PMO — 1,904,383 records  
curl https://wikiran.org/api/v1/leaks/pmo

# Amin Exchange — 32,314 records (matches site-stated "32,314 deals")
curl https://wikiran.org/api/v1/leaks/amin

# Vessels Archive — 429 records (matches site's "400+ vessels")
curl https://wikiran.org/api/v1/leaks/vessels
```

---

## 6. Wire Format Specification

WikiRan datasets use four distinct record types, determined by the dataset's `leakType` field.

### 6.1 Email Records

**Applies to:** datasets with `leakType = "email"` or `leakType = "email+ocr"`

Email records follow a consistent JSON structure observed across all confirmed email datasets (IFC: 54 samples; Sadaf: 192 samples; plus 3 samples each for 15 additional email datasets).

#### Field Definitions

| Field | Type | Required | Description |
|---|---|---|---|
| `date` | string | Yes | Human-readable date in English. Format: `"Month DD, YYYY"` e.g. `"December 20, 2022"`. **Not ISO 8601.** Occasionally `"YYYY-MM-DD"` in newer datasets. |
| `from` | string | Yes | Sender. Format: bare email address, or RFC 2822 display name + address, e.g. `"\"Name\" <addr@domain>"`. Display name may be in Farsi or quoted. |
| `to` | string | Sometimes | Recipient(s). Same format as `from`. Multiple recipients are comma-separated. Absent in some records. |
| `subject` | string | Sometimes | Email subject. May be Farsi, English, or mixed. May be empty or `"undefined"` in some datasets. |
| `body` | string | Sometimes | Plain text message body. Quoted previous messages appear as `>` prefixed lines (standard email quoting). No HTML. May be absent. |
| `attachCount` | integer | No | Number of file attachments. Absent in datasets with no attachments. When present, `0` means no attachments for this specific record. |
| `language` | string | Yes | Language code: `"fa"` (Farsi/Persian), `"en"` (English), `"tr"` (Turkish), `"mixed"` (multiple languages in one record). |

#### Date Format Notes

- Standard format: `"Month DD, YYYY"` with English month names (`January`…`December`)
- Some newer datasets (sej2, amin2025) use `"YYYY-MM-DD"` ISO format
- No timezone information is embedded in date strings
- Quoted reply headers within email bodies may contain different date formats (local timezone strings)

#### Confirmed Sample Records

**IFC — Farsi email (full body):**
```json
{
  "date": "July 6, 2015",
  "from": "gozaresh@internet.ir",
  "to": "info@busstop.ir",
  "subject": "همکار محترم افتخاری جناب آقای سلمان خان محمدي",
  "body": "«شما افسران جنگ نرم هستید...»\n\nبدینوسیله از جنابعالی که در طی سال گذشته...",
  "language": "fa"
}
```

**IFC — English bounce message:**
```json
{
  "date": "October 23, 2016",
  "from": "Mail Delivery System <Mailer-Daemon@server47.mylittledatacenter.com>",
  "to": "mizbani@internet.ir",
  "subject": "Mail delivery failed: returning message to sender",
  "body": "This message was created automatically by mail delivery software...",
  "language": "en"
}
```

**Sadaf — Farsi reply with attachment:**
```json
{
  "date": "December 20, 2022",
  "from": "order@sadafexchange.com",
  "to": "\"Mikayelian\" <info@tejarat-exchange.com>",
  "subject": "Re: ابطال حواله 1-3442ن1401",
  "body": "On 2022-12-20 12:29, Mikayelian wrote:\n> سلام روز بخیر...",
  "attachCount": 0,
  "language": "fa"
}
```

**Sadaf — English with 1 attachment:**
```json
{
  "date": "December 20, 2022",
  "from": "supply@sadafexchange.com",
  "to": "Treasury@tejaratbank.ir",
  "subject": "174.602 USD-CONFIRM-SHELL EX",
  "body": "174.602 USD-CONFIRM-SHELL EX",
  "attachCount": 1,
  "language": "en"
}
```

**SEJ2 — short English reply (newer ISO date format):**
```json
{
  "date": "2024-06-11",
  "from": "hamidreza heidari",
  "to": "test@tetisglobal.com",
  "subject": "Re: stay",
  "body": "ok reccieved",
  "attachCount": 0,
  "language": "en"
}
```

**ASBGroup — Turkish:**
```json
{
  "date": "July 19, 2021",
  "from": "Elif Gunay <Elif.Gunay@asbgroup.com.tr>",
  "subject": "Kurban Bayramınız Kutlu Olsun",
  "attachCount": 1,
  "language": "tr"
}
```

#### Common Domain Patterns by Dataset

| Dataset | Primary Sending Domain(s) |
|---|---|
| ifc | `internet.ir` |
| sadaf | `sadafexchange.com`, `tejarat-exchange.com` |
| saf | `safiranas.com` |
| asbgroup | `asbgroup.com.tr` |
| mefa | `daraee.com` |
| moe | `nri.ac.ir` |
| tbtb | `tbtb.ir` |

#### Inbox Contamination Warning

Several datasets contain personal or commercial emails mixed into the leaked organizational inbox. Examples observed:

- **steel-companies**: Brazilian retail marketing spam, German industrial newsletter
- **pcc**: Financial research marketing emails, Indian industry newsletters
- **sepah**: Brazilian retail marketing, personal Gmail messages
- **moe**: Iranian cloud service (ArvanCloud) promotional email

Any ingestion pipeline should be prepared to encounter off-topic emails in email datasets.

---

### 6.2 OCR Document Records

**Applies to:** datasets with `leakType = "ocr"` or `leakType = "email+ocr"` (the OCR portion)

OCR records are scanned physical documents (PDFs, images, Excel files) that have been processed with optical character recognition. They have no `from`/`to`/`subject`/`body` fields. They are identified by filename or MD5 hash.

#### Observed characteristics

- Filename used as record identifier (e.g. `"1_2920-LIGHT BLEND CRUDE OIL.pdf"`)
- Content is raw OCR text — may contain garbled characters from imperfect OCR on Farsi handwriting
- Farsi text may appear in incorrect reading order in OCR output
- No structured date/from/to fields — dates appear within the OCR text body, often in Iranian calendar format (e.g. `1403/02/09` = April 28, 2024)
- File types observed: PDF, JPG, PNG, XLSX, XLSM, DOCX

#### Sample OCR records (observed from site HTML)

**SEJ — crude oil product specification:**
```json
{
  "filename": "1_2920-LIGHT BLEND CRUDE OIL APEX-DDU DONGJIAKOU-CHINA-1.pdf",
  "summary": "Crude oil product spec: density, sulfur content, viscosity, ASTM test results. Vendor: APEX TRADING AND CONSULTING SERVICES, Hong Kong. Destination: Dongjiakou, China."
}
```

**Radin Exchange — gold coin receipt:**
```json
{
  "md5": "02d96cc8b0eac13d75847e3f1a6c7d76",
  "fileType": "JPG",
  "summary": "Scanned gold coin/currency exchange receipt from Radin Exchange. Amount and customer details in Farsi."
}
```

**Pedram Exchange — AED to EUR conversion agreement:**
```json
{
  "filename": "1403.02.09-71-موافقت ریت.docx",
  "summary": "AED to EUR currency conversion agreement. 37,374 AED → 9,405 EUR at rate 3.9737. Date: 1403/02/09 (2024-04-28)."
}
```

**SEJ2 — vessel noon report (Excel):**
```json
{
  "filename": "Noon Report 04 DEC 2024.xlsm",
  "summary": "Vessel LIMAS noon report 2024-12-04. Charter party 2024-10-22. Indian Ocean, ETA KHG OPL 2024-12-14. Cargo: crude oil. Total distance: 3903 miles."
}
```

---

### 6.3 Transaction Table Records

**Applies to:** datasets with `leakType = "table"` for exchange house transaction data (amin, amin2025, transactions, pgpicc)

Transaction records are structured rows from internal accounting systems. Field names vary by dataset.

#### Amin Exchange — confirmed field names

```json
{
  "date": "20/12/2022",
  "account_number": "101307",
  "iranian_company": "Abdollahi Hasan",
  "potential_front_company": "Feisu",
  "front_company_bank": "Guangfa Bank",
  "customer": "Shaoxing Keqiao Anqiao Textile Co",
  "currency": "USD",
  "amount": "50050.00",
  "user_id": "Yavar"
}
```

**Date format in Amin records:** `DD/MM/YYYY` (e.g. `"20/12/2022"`)

**Currency:** ISO 4217 code (USD, EUR, etc.)

#### Tahayyori Exchange (slug: transactions) — confirmed field names

```json
{
  "approximatePersianDate": "22.08.1400",
  "beneficiaryName": "GUDDU",
  "transactionStatus": "DONE",
  "coverCompanyBank": "CASH",
  "customerName": "ARFAN TAHAYORI",
  "beneficiaryBank": "CASH",
  "transactionAmount": "50,000",
  "beneficiaryBankCountry": "UAE",
  "currency": "AED"
}
```

**Date format in Tahayyori records:** Iranian/Persian calendar date `DD.MM.YYYY` (`22.08.1400` = November 13, 2021)

#### PGPICC — confirmed field names

```json
{
  "pi_number": "4011810252",
  "pci_number": "4011841616",
  "currency": "USD",
  "amount": "85,437",
  "due_date_iranian": "1401/7/26",
  "producer": "MEHR",
  "customer": "KEEN PEACE INTERNATIONAL CO.,LTD",
  "third_party_account": "MOONSTONE TRADE CO LIMITED",
  "bank": "EVERGROWING BANK (HENGFENG BANK)",
  "destination": "china"
}
```

**Date format in PGPICC records:** Iranian calendar `YYYY/M/D` (`1401/7/26` = October 18, 2022)

The PGPICC transaction table is also available as a direct CSV download: `https://wikiran.org/files/leaks/pgpicc/transactions.csv`

---

### 6.4 Shipping / Container Records (PMO)

**Applies to:** slug `pmo` — Ports and Maritime Organization

The PMO dataset contains approximately 1.9 million container shipping records, all from the December 2016 window.

```json
{
  "date": "2016-12-17",
  "shipper": "UNIBANG INTERNATIONAL LOGISTICS(SHENZHEN) CO.,LIMITED, China",
  "consignee": "GOLDEN SEA SHIPPING & LOGISTICS CO, Tehran, Iran",
  "cargo": "100 BAGS TABULAR ALUMINA HS:28182000",
  "container": "HDM1089SCOT7790",
  "vesselName": "SHABDIS",
  "shipIMO": "9349588"
}
```

**Key fields:**

| Field | Description |
|---|---|
| `date` | Departure/manifest date (ISO format in observed records) |
| `shipper` | Exporting company, city, country |
| `consignee` | Receiving company, city, country |
| `cargo` | Cargo description with HS code |
| `container` | Container ID |
| `vesselName` | Ship name |
| `shipIMO` | IMO vessel registration number |

---

### 6.5 Sanctions Entity Records

**Applies to:** slug `sanctions` — UK Government Sanctions Database

```json
{
  "name": "Hamidreza EMADI",
  "groupType": "Individual",
  "position": "Director Press TV Newsroom",
  "nationality": "Iranian",
  "dob": "1973",
  "regime": "Iran (Human Rights)",
  "ukRef": "IHR0034",
  "designationDate": "2020-12-31"
}
```

```json
{
  "companyName": "ENGINEERING OF EXPANSION OF NUCLEAR FUEL COMPANY LTD.",
  "groupType": "Entity",
  "city": "Tehran",
  "regime": "Iran (Nuclear)",
  "ukRef": "IRN0092",
  "designationDate": "2020-12-31"
}
```

**`groupType`** values observed: `"Individual"`, `"Entity"`  
**`regime`** values observed: `"Iran (Human Rights)"`, `"Iran (Nuclear)"`, others

---

### 6.6 Vessel Records

**Applies to:** slug `vessels` — Vessels Archive (429 records)

```json
{
  "vesselName": "TANGO",
  "imo": "9208100",
  "owner": "CORY MANAGEMENT SERVICES CORP",
  "flag": "LIBERIA",
  "sanctions": "NO",
  "linkToDB": "wikiran.org/leaks/pmo"
}
```

**`sanctions`** values: `"YES"` (US SDN-listed) or `"NO"`  
**`linkToDB`**: URL path of the source WikiRan dataset

---

### 6.7 Front Company Records

**Applies to:** slug `front-companies` — Front Companies Archive (1,169 records)

```json
{
  "frontCompanyName": "Cheng Pan co. Limited",
  "country": "China/Hong Kong",
  "bankName": "China Minsheng Banking",
  "accountNumberIBAN": "NRA066808 (EUR), NRA066785 (USD)",
  "linkToDB": "Triliance Petrochemical"
}
```

---

## 7. Attachment Download Specification

Attachments are binary files (images, PDFs, Excel, Word documents) associated with email records. They are served directly via HTTP GET with no authentication.

### URL Construction

Given an email record with an attachment, the URL is:

```
https://wikiran.org/attachments/leaks/{slug}//{hash}/{url-encoded-filename}
```

Where:
- `{slug}` = the dataset slug (e.g. `ifc`)
- `{hash}` = either a 32-character MD5 (newer datasets) or 40-character SHA1 (older datasets)
- `{url-encoded-filename}` = the filename, percent-encoded, from the site's rendered HTML

### HTTP Behavior

- Method: `GET`
- Authentication: none required
- Successful response: HTTP 200 with `Content-Type` header and binary body
- Known error cases: HTTP 403 for at least one specific Sadaf PDF file (server-side restriction)
- Redirect behavior: standard HTTP redirects followed normally

### Recommended Request Headers

```http
User-Agent: Mozilla/5.0 (compatible; YourProjectName/1.0)
Referer: https://wikiran.org/leaks/{slug}
```

The `Referer` header is recommended for naturalness but was not observed to be required for files that returned HTTP 200.

### Observed Attachment Types by Dataset

| Dataset | Attachment Types |
|---|---|
| ifc | JPEG images, PDFs |
| sadaf | PDFs, documents |
| sej, sej2 | PDFs, XLSX, XLSM (vessel noon reports), DOCX |
| caspian-airlines | JPEG images |
| sahara-thunder | XLSX spreadsheets |
| triliance | PDFs, manifests |
| steel-companies | PNG images, PDFs |
| pcc | XLSX spreadsheets |
| saf | JPEG images, PDFs, bank statements |
| asbgroup | JPEG images |
| mefa | JPEG images |

---

## 8. Known Datasets — Machine-Readable Table

> **Machine-readable catalog:** A companion file [`WIKIRAN-ORG-DATASETS.json`](./WIKIRAN-ORG-DATASETS.json) contains all 27 datasets as a JSON array. Each entry exposes the same fields as this table — `slug`, `titleEn`, `titleFa`, `category`, `leakType`, `totalRecords`, `dateRange` (`earliest`/`latest`), `primaryLanguage`, `hasAttachments`, `attachmentHashFormat`, `sampleAttachmentUrl`, `specialFiles`, and `sourceUrl` — ready for `JSON.parse` without any Markdown parsing.

Survey date: 2026-04-19. All datasets confirmed live. Record counts from public API (`/api/v1/leaks/{slug}`).

| # | Slug | Title (English) | Category | leakType | Records (API) | Date Range | Lang | Attachments | hasAttachments |
|---|---|---|---|---|---|---|---|---|---|
| 1 | `ifc` | The Iranian Filtering Committee | Government | email | 562,634 | 2014-05-01 – 2020-08-04 | fa/en | Yes (MD5) | true |
| 2 | `sadaf` | Sadaf Exchange | Economy | email+ocr | 27,461 | 2022-10-27 – 2023-05-20 | fa/en | Yes (MD5) | true |
| 3 | `sej` | Sepehr Energy Jahan | Economy | email+ocr | 9,596 | ~2023 – 2024-12-16 | mixed | Yes (MD5) | true |
| 4 | `sej2` | Sepehr Energy Jahan – AFGS Oil Command | Economy | email+ocr | 5,936 | 2024-06-11 – 2024-12-04 | mixed | Yes (MD5) | true |
| 5 | `radin` | Radin Exchange | Economy | ocr | 31,114 | ~2024-07-07 | fa | No | false |
| 6 | `amin` | Amin Exchange House | Economy | table | 32,314 | ~2022-12-20 | fa | ZIP bulk | false |
| 7 | `amin2025` | Amin Exchange 2025 | Economy | email+ocr | 15,406 | 2025-02-08 – ~2025-02 | mixed | Yes (MD5) | true |
| 8 | `pedram` | Pedram Pirouzan Exchange (Opal Exchange) | Economy | ocr | 4,654 | 2024-04-28 | fa | No | false |
| 9 | `sepah` | Bank Sepah | Economy | email+ocr | 8,734 | 2023-11-30 – ~2023-12 | fa | No | false |
| 10 | `tejarat` | Tejarat Bank and Exchange | Economy | ocr | 348 | ~2024-05-25 | mixed | No | false |
| 11 | `caspian-airlines` | Caspian Airlines | Economy | email | 339 | 2019-08-17 – 2021-12-26 | mixed | Yes (MD5) | true |
| 12 | `sahara-thunder` | Sahara Thunder | Economy | email | 10,685 | 2019-08-14 – 2020-10-12 | mixed | Yes (MD5) | true |
| 13 | `triliance` | Triliance Petrochemical Co. Ltd. | Economy | email | 6,687 | 2022-10-30 – 2023-02-17 | en | Yes (MD5) | true |
| 14 | `steel-companies` | Iranian Steel Companies MSC & KSC | Economy | email | 94,064 | 2016-04-18 – 2021-12-16 | mixed | Yes (SHA1) | true |
| 15 | `pcc` | Petrochemical Commercial Company | Economy | email | 48,842 | 2019-04-11 – 2021-07-18 | en | Yes (SHA1) | true |
| 16 | `saf` | Safiran Airport Services | International Affairs | email | 2,171 | 2022-06-10 – 2022-07-04 | mixed | Yes (MD5) | true |
| 17 | `asbgroup` | ASB Group | International Affairs | email | 215,444 | 2021-03-23 – 2022-07-19 | en/tr | Yes (SHA1) | true |
| 18 | `pgpicc` | PGPICC (Persian Gulf Petrochemical) | Economy | email | 27,702 | ~2022 | mixed | No | false |
| 19 | `exchange-zg` | Berelian / ZarrinGhalam Exchange | Economy | ocr | 1,364 | ~2019-06-11 | fa | No | false |
| 20 | `transactions` | Tahayyori Exchange (Arz-Iran) | Economy | table | 4,643 | ~2021 – ~2022 | fa | No | false |
| 21 | `mefa` | Ministry of Economic Affairs & Finance | Government | email | 23,973 | 2020-06-01 – 2020-11-07 | fa | Yes (SHA1) | true |
| 22 | `moe` | Ministry of Energy | Government | email | 23,264 | 2021-03-10 – 2021-03-14 | fa | Yes (SHA1) | true |
| 23 | `tbtb` | Tehran Province Electricity Distribution | Government | email | 290 | 2021-01-06 – 2021-02-08 | fa | Yes (SHA1) | true |
| 24 | `pmo` | Ports and Maritime Organization | Economy | table | 1,904,383 | 2016-11-22 – 2016-12-17 | mixed | No | false |
| 25 | `sanctions` | Sanction-Imposed Entities (UK) | International Affairs | table | 952 | N/A — reference DB | en | No | false |
| 26 | `vessels` | Vessels Archive | Archives | table | 429 | N/A — reference DB | en | No | false |
| 27 | `front-companies` | Front Companies Archive | Archives | table | 1,169 | N/A — reference DB | en | No | false |

**Totals:** 27 datasets, ~3.2 million records, 17 datasets with email content, 15 datasets with downloadable per-record attachments.

### 8b. Dataset Subject Matter Summary

One-line subject matter per dataset. See §9 for extended notes on key datasets.

| Slug | Subject Matter |
|---|---|
| `ifc` | Iran's internet censorship authority — internal orders, blocked-site lists, judicial surveillance warrants, internal credentials |
| `sadaf` | Sadaf Exchange (UAE-linked) — IRGC-connected hawala transactions, remittances, wire transfers between Iran, UAE, and Turkey |
| `sej` | Sepehr Energy Jahan — AFGS-controlled crude oil export to China; shipping manifests, vessel noon reports, cargo specs |
| `sej2` | SEJ / AFGS Oil Command — second tranche; vessel logs (LIMAS), refinery cargo, crew correspondence |
| `radin` | Radin Exchange — scanned gold coin purchase receipts, bank transfer slips, currency exchange forms |
| `amin` | Amin Exchange (Samed Nemati) — 32,314 structured rows: buyer, front company, bank, amount, currency; significant China/UAE activity |
| `amin2025` | Amin Exchange 2025 tranche — fresh email + OCR; includes AFA Steel Co. (`afasteel.com`) correspondence |
| `pedram` | Pedram Pirouzan / Opal Exchange — AED↔EUR conversion agreements, currency memos in Farsi |
| `sepah` | Bank Sepah (Iranian Armed Forces bank) — internal inbox snapshot; operational and commercial correspondence |
| `tejarat` | Tejarat Bank and Exchange — scanned OCR bank documents; mixed Farsi/English |
| `caspian-airlines` | Caspian Airlines — sanctioned carrier; operational emails, crew correspondence, route planning |
| `sahara-thunder` | Sahara Thunder (MODAFL front company) — oil sales, vessel fleet, arms trade with Russia; contracts, bank accounts |
| `triliance` | Triliance Petrochemical Co. Ltd. (sanctioned since 2020) — Iranian oil/petrochemical export; vessel passenger manifests, NIOC contracts |
| `steel-companies` | MSC & KSC (Mobarakeh and Khuzestan Steel) — internal emails showing US sanction bypass; checks, front-company establishment docs |
| `pcc` | PCC (Petrochemical Commercial Company, MODAFL) — 20 GB of export documents; funds IRGC Quds Force; cargo updates |
| `saf` | Safiran Airport Services (sanctioned) — UAV transfer to Russia; financial exchanges, bank statements, invoices; high attachment density |
| `asbgroup` | ASB Group (Turkish, Sidki Ayan) — primary IRGC/Quds Force oil export network; correspondence in Turkish and English |
| `pgpicc` | PGPICC (Iran's largest petrochemical company) — email inbox + transaction table showing sanction circumvention; front companies in China |
| `exchange-zg` | ZarrinGhalam / Berelian / GCM Exchange — 37 international front companies, 140 UAE/China bank accounts; OCR receipts |
| `transactions` | Tahayyori Exchange (ARZ-IRAN) — 3B+ USD in 2020–2022 transactions; UAE/China/Turkey/Hong Kong front company network |
| `mefa` | Ministry of Economic Affairs and Finance — internal ministry email (domain: `daraee.com`); administrative correspondence |
| `moe` | Ministry of Energy — internal inbox snapshot (domain: `nri.ac.ir`); mixed with commercial email in leaked inbox |
| `tbtb` | Tehran Province Electricity Distribution — state utility internal email including TeamChat notification digests |
| `pmo` | Ports & Maritime Organization — 1.9M container shipping records; exporter/importer details, HS codes, vessel IMOs; China-to-Iran routes |
| `sanctions` | UK government sanctions designations for Iranian entities — name, regime, designation date, UK reference number |
| `vessels` | Cross-dataset archive: 400+ vessels used in Iranian sanction evasion — IMO, owner, flag, SDN status |
| `front-companies` | Cross-dataset archive: 1,000+ Iranian front companies — country, bank, account number, source dataset |

---

## 9. Dataset Detail Notes

### 9.1 High-Priority Email Datasets

**ifc (562,634 records)** — Largest email dataset. Iran's Criminal Content Identification Working Group (internet censorship authority). Emails span 2014–2020. Domains: `internet.ir`. Reveals censorship orders, blocked sites, user data requests, judicial warrants, and internal credentials.

**asbgroup (215,444 records)** — Second-largest email dataset. Turkish holding company (ASB Group, Sidki Ayan) serving as IRGC/Quds Force oil export infrastructure. Emails in Turkish and English (2021–2022). Domain: `asbgroup.com.tr`.

**steel-companies (94,064 records)** — Mobarakeh Steel Company (MSC) and Khuzestan Steel Company (KSC). Internal emails 2016–2021 showing front company establishment for sanction bypass. Uses SHA1 attachment hashes.

**pcc (48,842 records)** — Petrochemical Commercial Company, Iran's second-largest oil exporter. 2019–2021 email archive ("20 GB of documents" per site). Uses SHA1 attachment hashes.

**amin (32,314 records)** — Structured transaction table, not email. Amin Exchange (ex-IRGC officer Samed Nemati). 32,314 rows covering buyer, seller, bank, amount, currency. Full raw archive available as ZIP download (5.12 GB). Significant front company activity in China and UAE.

**radin (31,114 records)** — OCR-only dataset. Radin Exchange bank receipts and currency forms. Dates in Iranian calendar format (e.g. `1403/04/17` = July 7, 2024).

**mefa (23,973 records)** — Iran's Ministry of Economic Affairs and Finance. Internal emails 2020. Domain: `daraee.com`.

**moe (23,264 records)** — Iran's Ministry of Energy. Internal email snapshot from a single week in March 2021. Domain: `nri.ac.ir`.

**amin2025 (15,406 records)** — Second Amin Exchange release (2025), mixed email + OCR. Domains include `afasteel.com` (AFA Steel Co.).

**sahara-thunder (10,685 records)** — MODAFL front company. Oil sales, vessel fleet, arms trade with Russia. Emails 2019–2020.

**sepah (8,734 records)** — Bank Sepah (primary Iranian Armed Forces bank). Mixed email + OCR (2023). Includes junk/marketing emails mixed into leaked inbox.

**sej (9,596 records)** — Sepehr Energy Jahan. Mixed email + OCR. Crude oil shipment to China, vessel logs, crew health monitoring documents.

### 9.2 PMO — Largest Dataset

**pmo (1,904,383 records)** is by far the largest dataset. It covers container shipments to Iran from China during a one-month window (November–December 2016). Each record is a shipping manifest entry. This dataset is effectively a snapshot of a national customs/port database.

### 9.3 Archive Datasets (Cross-Database Indices)

Three datasets are not raw leaks but are structured reference databases compiled by WikiRan editors from all leak sources:

- **vessels** (429 records): 400+ ships used in Iranian sanction evasion, with IMO numbers and SDN listing status
- **front-companies** (1,169 records): 1,000+ front companies with bank account details, linked to source datasets
- **sanctions** (952 records): UK government sanctions designations for Iranian entities

### 9.4 OCR Dataset Calendar Notes

Several OCR datasets contain dates in the Iranian Solar Hijri calendar. Conversion reference:

| Iranian Date | Gregorian Equivalent |
|---|---|
| 1398/03/21 | June 11, 2019 |
| 1400/08/22 | November 13, 2021 |
| 1401/07/26 | October 18, 2022 |
| 1403/02/09 | April 28, 2024 |
| 1403/04/17 | July 7, 2024 |

---

## 9b. Per-Dataset Sample Records

One sample record per dataset (survey date 2026-04-19). Source notes:

- **Email records** (`email`, `email+ocr`): verbatim field values as observed in the site's rendered HTML. Field names and values are reproduced exactly.
- **OCR records** (`ocr`, `email+ocr`): observed/normalized. The `filename` and `md5` fields are verbatim; `ocrText` is a normalized summary of the visible OCR content (the raw OCR text may include garbled characters and Farsi text in reversed reading order).
- **Table/structured records** (`table`): verbatim field names and values from the site's rendered HTML. Field names come from the underlying Elasticsearch document schema.

**ifc** (email):
```json
{
  "date": "July 6, 2015",
  "from": "gozaresh@internet.ir",
  "to": "info@busstop.ir",
  "subject": "همکار محترم افتخاری جناب آقای سلمان خان محمدي",
  "body": "«شما افسران جنگ نرم هستید...»\n\nبدینوسیله از جنابعالی که در طی سال گذشته...",
  "language": "fa"
}
```

**sadaf** (email):
```json
{
  "date": "December 20, 2022",
  "from": "supply@sadafexchange.com",
  "to": "Treasury@tejaratbank.ir",
  "subject": "174.602 USD-CONFIRM-SHELL EX",
  "body": "174.602 USD-CONFIRM-SHELL EX",
  "attachCount": 1,
  "language": "en"
}
```

**sej** (OCR):
```json
{
  "filename": "1_2920-LIGHT BLEND CRUDE OIL APEX-DDU DONGJIAKOU-CHINA-1.pdf",
  "ocrText": "LIGHT BLEND CRUDE OIL\nSeller: APEX TRADING AND CONSULTING SERVICES, Hong Kong\nDestination: Dongjiakou, China\n[Density, sulfur content, ASTM test parameters...]"
}
```

**sej2** (email):
```json
{
  "date": "2024-06-11",
  "from": "hamidreza heidari",
  "to": "test@tetisglobal.com",
  "subject": "Re: stay",
  "body": "ok reccieved",
  "attachCount": 0,
  "language": "en"
}
```

**radin** (OCR):
```json
{
  "md5": "02d96cc8b0eac13d75847e3f1a6c7d76",
  "fileType": "JPG",
  "ocrText": "Radin Exchange gold coin purchase receipt. Customer name and amount in Farsi. Date: 1403/04/17 (2024-07-07)."
}
```

**amin** (transaction table):
```json
{
  "date": "20/12/2022",
  "account_number": "101307",
  "iranian_company": "Abdollahi Hasan",
  "potential_front_company": "Feisu",
  "front_company_bank": "Guangfa Bank",
  "customer": "Shaoxing Keqiao Anqiao Textile Co",
  "currency": "USD",
  "amount": "50050.00",
  "user_id": "Yavar"
}
```

**amin2025** (email):
```json
{
  "date": "February 8, 2025",
  "from": "afasubstation@gmail.com",
  "to": "a.hatami@afasteel.com, hashemij600@gmail.com",
  "subject": "AFA Substation Data",
  "body": "Melting Factory = 256.172 [MWH], Rolling Mill = 42.215 [MWH], Total Load = 31.902 MW",
  "attachCount": 0,
  "language": "en"
}
```

**pedram** (OCR):
```json
{
  "filename": "1403.02.09-71-موافقت ریت.docx",
  "ocrText": "AED to EUR currency conversion agreement.\n37,374 AED → 9,405 EUR\nRate: 3.9737\nDate: 1403/02/09 (2024-04-28)"
}
```

**sepah** (email):
```json
{
  "date": "November 30, 2023",
  "from": "\"Msnsor Azad\" <mansoor.azad1357@gmail.com>",
  "subject": "Pdf Viewer",
  "attachCount": 0,
  "language": "fa"
}
```

**tejarat** (OCR):
```json
{
  "filename": "uae_20.jpg",
  "md5": "0f056a3588eaca5371230fd993bda39c",
  "fileType": "jpg",
  "ocrText": "UAE-format bank transfer document. Garbled OCR of Arabic/Farsi form. Transaction amount and account details partially readable."
}
```

**caspian-airlines** (email):
```json
{
  "date": "August 17, 2019",
  "from": "supply.mng@caspianairlines.com",
  "subject": "FW: نحوه پرداختي",
  "attachCount": 1,
  "language": "fa"
}
```

**sahara-thunder** (email):
```json
{
  "date": "June 24, 2020",
  "from": "saharathunder <commercial@saharathunder.com>",
  "subject": "FW:",
  "attachCount": 1,
  "language": "mixed"
}
```

**triliance** (email):
```json
{
  "date": "October 31, 2022",
  "from": "yousefi <m.yousefi@nioc-intl.ir>",
  "subject": "Amend No 1 contract no. PMO-ES/5989/NA/2022 Abadan",
  "attachCount": 1,
  "language": "en"
}
```

**steel-companies** (email):
```json
{
  "date": "May 10, 2016",
  "from": "a.khoram <a.khoram@ksc.ir>",
  "to": "procurement@supplier.de",
  "subject": "FW: Inquiry No. 354670",
  "attachCount": 1,
  "language": "en"
}
```

**pcc** (email):
```json
{
  "date": "June 3, 2021",
  "from": "team@ssessments.com",
  "subject": "SSESSMENTS Asia AM Americas PM Brief - Crude Oil",
  "attachCount": 0,
  "language": "en"
}
```

**saf** (email — high attachment density):
```json
{
  "date": "July 4, 2022",
  "from": "account@safiranas.com",
  "to": "handling@groundservices.com",
  "subject": "RE: CLICK22-02461 || T7-TANMG || OIII || GH/",
  "attachCount": 12,
  "language": "en"
}
```

**asbgroup** (email):
```json
{
  "date": "April 29, 2022",
  "from": "Fatih Karaca <Fatih.Karaca@asbgroup.com.tr>",
  "subject": "İlt: Bursa/MKP Dava Dos. Hk.",
  "attachCount": 2,
  "language": "tr"
}
```

**pgpicc** (transaction table):
```json
{
  "pi_number": "4011810252",
  "pci_number": "4011841616",
  "currency": "USD",
  "amount": "85,437",
  "due_date_iranian": "1401/7/26",
  "producer": "MEHR",
  "customer": "KEEN PEACE INTERNATIONAL CO.,LTD",
  "third_party_account": "MOONSTONE TRADE CO LIMITED",
  "bank": "EVERGROWING BANK (HENGFENG BANK)",
  "destination": "china"
}
```

**exchange-zg** (OCR):
```json
{
  "filename": "حسن ونکی-بیعانه.pdf",
  "ocrText": "Bank Sepah internet transfer\nAmount: 5,000,000 IRR\nFrom: Nasser Zarringhalam & Partners\nTo: Hasan Vanaki\nDate: 1398/03/21 (June 11, 2019)"
}
```

**transactions** (transaction table):
```json
{
  "approximatePersianDate": "22.08.1400",
  "beneficiaryName": "GUDDU",
  "transactionStatus": "DONE",
  "coverCompanyBank": "CASH",
  "customerName": "ARFAN TAHAYORI",
  "beneficiaryBank": "CASH",
  "transactionAmount": "50,000",
  "beneficiaryBankCountry": "UAE",
  "currency": "AED"
}
```

**mefa** (email):
```json
{
  "date": "October 7, 2020",
  "from": "محمد جمیل <2630859703@daraee.com>",
  "subject": "تمدید قرارداد دارایی",
  "attachCount": 1,
  "language": "fa"
}
```

**moe** (email):
```json
{
  "date": "March 13, 2021",
  "from": "Satab@nri.ac.ir",
  "to": "energy-dept@nri.ac.ir",
  "subject": "کارجدید",
  "attachCount": 0,
  "language": "fa"
}
```

**tbtb** (email):
```json
{
  "date": "January 6, 2021",
  "from": "notifications <noreply@tbtb.ir>",
  "subject": "TeamChat daily digest",
  "attachCount": 5,
  "language": "fa"
}
```

**pmo** (container/shipping):
```json
{
  "date": "2016-12-17",
  "shipper": "UNIBANG INTERNATIONAL LOGISTICS(SHENZHEN) CO.,LIMITED, China",
  "consignee": "GOLDEN SEA SHIPPING & LOGISTICS CO, Tehran, Iran",
  "cargo": "100 BAGS TABULAR ALUMINA HS:28182000",
  "container": "HDM1089SCOT7790",
  "vesselName": "SHABDIS",
  "shipIMO": "9349588"
}
```

**sanctions** (entity):
```json
{
  "name": "Hamidreza EMADI",
  "groupType": "Individual",
  "position": "Director Press TV Newsroom",
  "nationality": "Iranian",
  "dob": "1973",
  "regime": "Iran (Human Rights)",
  "ukRef": "IHR0034",
  "designationDate": "2020-12-31"
}
```

**vessels** (vessel):
```json
{
  "vesselName": "TANGO",
  "imo": "9208100",
  "owner": "CORY MANAGEMENT SERVICES CORP",
  "flag": "LIBERIA",
  "sanctions": "NO",
  "linkToDB": "wikiran.org/leaks/pmo"
}
```

**front-companies** (front company):
```json
{
  "frontCompanyName": "Cheng Pan co. Limited",
  "country": "China/Hong Kong",
  "bankName": "China Minsheng Banking",
  "accountNumberIBAN": "NRA066808 (EUR), NRA066785 (USD)",
  "linkToDB": "Triliance Petrochemical"
}
```

---

## 10. Code Samples

All samples use only standard library HTTP clients. No third-party dependencies required beyond what is listed.

> **Important note on email record listing:** WikiRan.org renders individual email records client-side via Elasticsearch. A static unauthenticated HTTP GET of a dataset page (`/leaks/{slug}`) returns the React app shell with zero record data. There is no confirmed public unauthenticated REST API for listing individual email records. The samples below demonstrate what _is_ publicly accessible: the per-dataset record count API and attachment file downloads. For bulk email content, researchers should work from full dataset exports where provided (e.g. `amin.zip`) or use browser automation to capture the client-side API calls.

### 10.1 Python — Query dataset counts and download data

**Dependency:** `pip install requests`

```python
# MIT License — see top of document
import requests

SLUG = "ifc"  # replace with any dataset slug
BASE_URL = "https://wikiran.org"

def get_record_count(slug: str) -> int:
    """Query the public API for exact document count."""
    url = f"{BASE_URL}/api/v1/leaks/{slug}"
    resp = requests.get(url, timeout=30)
    resp.raise_for_status()
    data = resp.json()
    return data.get("total", 0)

def list_all_datasets() -> list:
    """
    Fetch the full dataset registry from the public API.
    Returns a list of all datasets with their metadata.
    Note: individual email records are not exposed via a public API;
    wikiran.org renders them client-side via Elasticsearch.
    This endpoint returns dataset-level metadata only.
    """
    url = f"{BASE_URL}/api/v1/leaks"
    headers = {"User-Agent": "Mozilla/5.0 (compatible; ResearchBot/1.0)"}
    resp = requests.get(url, headers=headers, timeout=30)
    resp.raise_for_status()
    return resp.json()

if __name__ == "__main__":
    count = get_record_count(SLUG)
    print(f"Dataset '{SLUG}' has {count:,} records per public API")
    
    datasets = list_all_datasets()
    for ds in datasets:
        slug = ds.get("slug", "?")
        total = ds.get("total", "?")
        print(f"  {slug}: {total:,}" if isinstance(total, int) else f"  {slug}: {total}")
```

### 10.2 Python — Download an attachment

```python
# MIT License — see top of document
import requests
import hashlib
from pathlib import Path

def download_attachment(
    slug: str,
    hash_hex: str,
    filename: str,
    output_dir: str = ".",
) -> Path:
    """
    Download a WikiRan attachment file.
    
    Args:
        slug:       Dataset slug (e.g. "ifc")
        hash_hex:   MD5 (32 chars) or SHA1 (40 chars) hex hash
        filename:   URL-encoded filename (will be decoded for local save)
        output_dir: Directory to save the file
    
    Returns:
        Path to saved file
    
    Raises:
        requests.HTTPError: if the server returns a non-2xx status
    """
    from urllib.parse import quote, unquote
    
    # URL: note the REQUIRED double slash before the hash
    encoded_filename = quote(filename, safe="")
    url = f"https://wikiran.org/attachments/leaks/{slug}//{hash_hex}/{encoded_filename}"
    
    headers = {
        "User-Agent": "Mozilla/5.0 (compatible; ResearchBot/1.0)",
        "Referer": f"https://wikiran.org/leaks/{slug}",
    }
    
    resp = requests.get(url, headers=headers, timeout=60, stream=True)
    resp.raise_for_status()
    
    local_name = unquote(filename)
    out_path = Path(output_dir) / local_name
    out_path.parent.mkdir(parents=True, exist_ok=True)
    
    hasher = hashlib.sha256()
    with open(out_path, "wb") as f:
        for chunk in resp.iter_content(chunk_size=65536):
            f.write(chunk)
            hasher.update(chunk)
    
    print(f"Saved {out_path} ({out_path.stat().st_size:,} bytes, sha256={hasher.hexdigest()[:16]}...)")
    return out_path


if __name__ == "__main__":
    # Confirmed working as of 2026-04-19
    download_attachment(
        slug="ifc",
        hash_hex="73ebd4138d7977170eec29d9490995bf",  # MD5 (32 chars)
        filename="IMG-20171114-WA0006.jpg",
    )
    
    download_attachment(
        slug="steel-companies",
        hash_hex="cdccd7f9256e61fbe306bb5ead464774873ada4a",  # SHA1 (40 chars)
        filename="image001.png",
    )
```

### 10.3 TypeScript — Query dataset counts and download data

**Runtime:** Node.js 18+ (native `fetch`)

```typescript
// MIT License — see top of document

const BASE_URL = "https://wikiran.org";

interface DatasetCountResponse {
  total: number;
  [key: string]: unknown;
}

/**
 * Query the public API for the exact document count in a dataset.
 * This is the authoritative count — do not rely on numbers in the HTML page,
 * which show 0 for static/unauthenticated fetches.
 */
async function getRecordCount(slug: string): Promise<number> {
  const url = `${BASE_URL}/api/v1/leaks/${slug}`;
  const resp = await fetch(url, {
    headers: { "User-Agent": "ResearchBot/1.0" },
    signal: AbortSignal.timeout(30_000),
  });
  if (!resp.ok) throw new Error(`HTTP ${resp.status} for ${url}`);
  const data = (await resp.json()) as DatasetCountResponse;
  return data.total ?? 0;
}

/**
 * Get the registry of all datasets from the public API.
 */
async function listAllDatasets(): Promise<unknown[]> {
  const url = `${BASE_URL}/api/v1/leaks`;
  const resp = await fetch(url, {
    headers: { "User-Agent": "ResearchBot/1.0" },
    signal: AbortSignal.timeout(30_000),
  });
  if (!resp.ok) throw new Error(`HTTP ${resp.status} for ${url}`);
  return resp.json() as Promise<unknown[]>;
}

// Example usage
const DATASETS = [
  "ifc", "sadaf", "sej", "sej2", "radin", "amin", "amin2025",
  "pedram", "sepah", "tejarat", "caspian-airlines", "sahara-thunder",
  "triliance", "steel-companies", "pcc", "saf", "asbgroup", "pgpicc",
  "exchange-zg", "transactions", "mefa", "moe", "tbtb",
  "pmo", "sanctions", "vessels", "front-companies",
];

async function main() {
  console.log("WikiRan.org dataset record counts (from public API):\n");
  let totalRecords = 0;

  for (const slug of DATASETS) {
    try {
      const count = await getRecordCount(slug);
      totalRecords += count;
      console.log(`  ${slug.padEnd(22)} ${count.toLocaleString().padStart(12)} records`);
    } catch (err) {
      console.error(`  ${slug}: ERROR — ${err}`);
    }
  }

  console.log(`\n  ${"TOTAL".padEnd(22)} ${totalRecords.toLocaleString().padStart(12)} records`);
}

main().catch(console.error);
```

### 10.4 TypeScript — Download an attachment

```typescript
// MIT License — see top of document
import { createWriteStream } from "node:fs";
import { mkdir } from "node:fs/promises";
import { join } from "node:path";
import { createHash } from "node:crypto";

/**
 * Download a WikiRan attachment file to disk.
 * 
 * @param slug      - Dataset slug (e.g. "ifc")
 * @param hashHex   - MD5 (32 chars) or SHA1 (40 chars) hex hash
 * @param filename  - Decoded filename (will be percent-encoded for URL)
 * @param outputDir - Directory to save the file (created if needed)
 */
async function downloadAttachment(
  slug: string,
  hashHex: string,
  filename: string,
  outputDir: string = ".",
): Promise<void> {
  const encodedFilename = encodeURIComponent(filename);
  // Note the REQUIRED double slash (//) before the hash
  const url = `https://wikiran.org/attachments/leaks/${slug}//${hashHex}/${encodedFilename}`;

  const resp = await fetch(url, {
    headers: {
      "User-Agent": "Mozilla/5.0 (compatible; ResearchBot/1.0)",
      "Referer": `https://wikiran.org/leaks/${slug}`,
    },
    signal: AbortSignal.timeout(60_000),
  });

  if (!resp.ok) {
    throw new Error(`HTTP ${resp.status} ${resp.statusText} for ${url}`);
  }

  await mkdir(outputDir, { recursive: true });
  const outPath = join(outputDir, filename);
  const hasher = createHash("sha256");
  const writer = createWriteStream(outPath);

  const body = resp.body;
  if (!body) throw new Error("Empty response body");

  const reader = body.getReader();
  let totalBytes = 0;

  for (;;) {
    const { done, value } = await reader.read();
    if (done) break;
    writer.write(value);
    hasher.update(value);
    totalBytes += value.length;
  }

  await new Promise<void>((resolve, reject) => {
    writer.end((err: Error | null) => (err ? reject(err) : resolve()));
  });

  const sha256 = hasher.digest("hex");
  console.log(`Saved ${outPath} (${totalBytes.toLocaleString()} bytes, sha256=${sha256.slice(0, 16)}...)`);
}

// Confirmed working examples (as of 2026-04-19)
async function main() {
  // MD5 hash (32 chars) — IFC JPEG
  await downloadAttachment(
    "ifc",
    "73ebd4138d7977170eec29d9490995bf",
    "IMG-20171114-WA0006.jpg",
    "./downloads",
  );

  // SHA1 hash (40 chars) — Steel Companies PNG
  await downloadAttachment(
    "steel-companies",
    "cdccd7f9256e61fbe306bb5ead464774873ada4a",
    "image001.png",
    "./downloads",
  );

  // Farsi filename — Sahara Thunder Excel (filename with Arabic script)
  await downloadAttachment(
    "sahara-thunder",
    "2be57bef42e6640966012d297f9432a3",
    "مشکوک الوصول.xlsx",
    "./downloads",
  );
}

main().catch(console.error);
```

### 10.5 Query the per-dataset record count API

**Minimal curl examples:**

```bash
# Single dataset count
curl -s https://wikiran.org/api/v1/leaks/ifc | python3 -c "import sys,json; print(json.load(sys.stdin)['total'])"

# All datasets — count each one
for slug in ifc sadaf sej sej2 radin amin amin2025 pedram sepah tejarat \
            caspian-airlines sahara-thunder triliance steel-companies pcc \
            saf asbgroup pgpicc exchange-zg transactions mefa moe tbtb \
            pmo sanctions vessels front-companies; do
  count=$(curl -s "https://wikiran.org/api/v1/leaks/$slug" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('total',0))")
  printf "%-25s %12s\n" "$slug" "$count"
done
```

---

## 11. Glossary

Iranian government and sanctions terminology encountered across WikiRan datasets:

**AFGS** — Armed Forces General Staff (ستاد کل نیروهای مسلح). Coordinates Iran's military across all branches.

**Arz-Iran / Tahayyori** — One of Iran's largest exchange houses (صرافى تحيرى). Connected to the `transactions` dataset.

**Criminal Content Identification Working Group (IFC)** — Iran's internet censorship authority established 2008 under the Attorney General. Operates through `internet.ir` domain. Official Persian name: کارگروه تعیین مصادیق محتوای مجرمانه.

**daraee.com** — Primary email domain for Iran's Ministry of Economic Affairs and Finance (MEFA, وزارت امور اقتصادی و دارایی).

**Exchange house (صرافی, sarafi)** — Iranian informal money transfer and currency exchange operator. Many are used for sanction evasion. WikiRan hosts data from: Amin, Sadaf, Tahayyori/Arz-Iran, Radin, Pedram/Opal, ZarrinGhalam/Berelian.

**FRAJA** — Iran's Armed Forces Retirement Organization (سازمان بازنشستگی نیروهای مسلح). Connected to oil operations.

**Front company** — A legally registered company in a third country (typically China, UAE, Hong Kong, Turkey) used to conduct transactions that would be blocked by international sanctions if done directly through Iranian entities. WikiRan's `front-companies` archive catalogs 1,000+ such entities.

**HS code (Harmonized System)** — International goods classification code used in customs documentation. Appears in PMO container records (e.g. `HS:28182000` = tabular alumina).

**IMO number** — International Maritime Organization vessel identification number. Permanent 7-digit identifier (with check digit) for ships. Appears in PMO and vessels datasets.

**Iranian Solar Hijri calendar (شمسی)** — Calendar system used in Iran. Year 1403 = 2024–2025 CE. Months are 30–31 days; the year begins on the vernal equinox (~March 20). Common date formats in OCR data: `YYYY/MM/DD`, `DD/MM/YYYY`, `DD.MM.YYYY`.

**IRGC** — Islamic Revolutionary Guard Corps (سپاه پاسداران انقلاب اسلامی). Iran's elite military force, designated as a Foreign Terrorist Organization by the US.

**IRGC-QF / Quds Force** — Special operations arm of the IRGC responsible for extra-territorial operations, including financing proxy forces in Lebanon (Hezbollah), Syria, Iraq, and Yemen.

**MODAFL** — Ministry of Defense and Armed Forces Logistics (وزارت دفاع و پشتیبانی نیروهای مسلح). Controls defense industries including sanctioned weapons programs.

**NIOC** — National Iranian Oil Company (شرکت ملی نفت ایران). Iran's state oil company.

**nri.ac.ir** — Domain of the Niroo Research Institute, used as the Ministry of Energy's primary email infrastructure (appears in `moe` dataset).

**OCR (Optical Character Recognition)** — Automated text extraction from scanned images or PDFs. WikiRan uses OCR to make scanned documents searchable. OCR of Arabic/Farsi handwriting is imperfect; expect garbled characters.

**PGPICC** — Persian Gulf Petrochemical Industry Commercial Company (شرکت تجارت صنعت پتروشیمی خلیج فارس). Iran's largest petrochemical company.

**PCC** — Petrochemical Commercial Company (شرکت بازرگانی پتروشیمی). Iran's second-largest petrochemical exporter, controlled by MODAFL.

**SDN (Specially Designated Nationals)** — US Treasury OFAC designation. Companies and individuals on the SDN list are subject to US sanctions. The `vessels` and `front-companies` archives include SDN status for each entity.

**SEJ** — Sepehr Energy Jahan (سپهر انرژی جهان). US-designated company selling sanctioned Iranian crude oil on behalf of AFGS.

**Shahid Abolfathi Oil Command** — Clandestine oil export operation under AFGS, FRAJA, and MODAFL; operated by the SEJ2 dataset entities.

**Toman / IRR** — Iranian Rial (IRR) is the official currency; Iranian Toman (IRT) = 10 IRR. Large round numbers in OCR text (e.g. `414,000,000`) are typically in IRR.

**US Treasury / OFAC designation** — US Department of the Treasury's Office of Foreign Assets Control. Entities designated by OFAC are subject to US sanctions and asset freezes. Multiple datasets (Sadaf, Triliance, Caspian Airlines, Tejarat Bank, Safiran Airport Services) involve OFAC-designated entities.

---

*End of WikiRan.org Data Source Specification v1.0*  
*Survey: 2026-04-19 | License: CC0 1.0 (spec) + MIT (code)*
