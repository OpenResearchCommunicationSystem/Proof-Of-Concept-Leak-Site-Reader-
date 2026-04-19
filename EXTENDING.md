# Leak Data Reader PoC — Extension Guide

This guide provides four self-contained how-to sections for adding new capabilities to
the Leak Data Reader PoC. Each section is written for a developer who is new to the codebase.

---

## A. Adding a New Entity Type

Entity types are strings stored in the `entities.entity_type` column. The system
recognizes them in two places: the NER AI prompt and the crypto scanner regex list. UI
color mappings are optional for custom types.

### Steps

**1. Decide on a naming convention.**

Standard NLP types: `PERSON`, `ORG`, `EMAIL`, `PHONE`, `LOCATION`, `DATE`.  
Crypto scanner types: always prefixed `CRYPTO_` (e.g. `CRYPTO_ETH`).  
Custom types: use `SCREAMING_SNAKE_CASE` (e.g. `VESSEL_IMO`, `BANK_ACCOUNT`).

**2. Add the type to the AI extraction prompt (for NER-based types).**

Open `artifacts/api-server/src/lib/ai.ts` and find `extractEntitiesFromText`. Add your
type to the `Entity types:` line in the system prompt:

```typescript
// Before:
"Entity types: PERSON, ORG, EMAIL, PHONE, LOCATION, DATE."

// After (adding VESSEL_IMO):
"Entity types: PERSON, ORG, EMAIL, PHONE, LOCATION, DATE, VESSEL_IMO."
```

Also add a description of the new type so the model knows what to extract:
```typescript
"VESSEL_IMO: International Maritime Organization vessel number (7 digits, e.g. IMO 1234567)."
```

**3. Add a regex pattern (for scanner-based types).**

If the type has a well-defined format, add it to `artifacts/api-server/src/lib/crypto.ts`
(or create a new scanner file and wire it into a new route). The pattern array format is:

```typescript
{
  type: "VESSEL_IMO",
  re: /\bIMO\s?\d{7}\b/gi,
  confidence: 0.99,
}
```

**4. Add a color mapping in the frontend (optional).**

Open `artifacts/wikiran/src/pages/entities.tsx`. Find the `TYPE_COLORS` or `ENTITY_COLORS`
constant (or the badge rendering logic) and add your type:

```typescript
const ENTITY_COLOR: Record<string, string> = {
  PERSON: "bg-blue-900/60 text-blue-300",
  ORG: "bg-green-900/60 text-green-300",
  VESSEL_IMO: "bg-cyan-900/60 text-cyan-300", // new
};
```

**5. Trigger extraction on existing data.**

New AI-based types will appear in future ingests automatically. For existing emails,
re-extract entities by calling the route (or write a one-off script that iterates emails
and calls `extractEntitiesFromText`).

**6. No database migration needed.** `entities.entity_type` is a plain `text` column with
no enum constraint. Any string value is valid.

---

## B. Adding a New AI Analysis Function

The pattern for adding a new AI capability is: write the function in `lib/ai.ts`, add a
new route in `routes/`, add the endpoint to the OpenAPI spec, and regenerate the client.

### Steps

**1. Write the AI function in `artifacts/api-server/src/lib/ai.ts`.**

Follow the existing function signatures. The function should:
- Accept typed input (not `any`).
- Call `openai.chat.completions.create(...)` with the appropriate model.
- Parse the JSON response defensively with a `try/catch`.
- Return a typed result.

Example skeleton for a new "summarize attachment" function:
```typescript
export async function summarizeAttachment(
  ocrText: string,
  filename: string
): Promise<{ summary: string; flagged_items: string[] }> {
  const response = await openai.chat.completions.create({
    model: "gpt-4o",
    max_completion_tokens: 1000,
    messages: [
      {
        role: "system",
        content: "You are an OSINT analyst. Summarize this document and flag anything suspicious.",
      },
      {
        role: "user",
        content: `Filename: ${filename}\n\n${ocrText.slice(0, 6000)}`,
      },
    ],
  });

  try {
    const raw = response.choices[0]?.message?.content ?? "{}";
    const parsed = JSON.parse(raw.replace(/```json\n?|\n?```/g, "").trim());
    return {
      summary: parsed.summary ?? "",
      flagged_items: parsed.flagged_items ?? [],
    };
  } catch {
    return { summary: response.choices[0]?.message?.content ?? "", flagged_items: [] };
  }
}
```

**2. Create a new route file or add to an existing one.**

If it fits logically (e.g. this is an attachment action), add to
`artifacts/api-server/src/routes/attachments.ts`. Otherwise create a new file:

```typescript
// artifacts/api-server/src/routes/my-feature.ts
import { Router } from "express";
import { myNewAiFunction } from "../lib/ai.js";

const router = Router();

router.post("/my-feature/:id/action", async (req, res): Promise<void> => {
  // validate, call AI function, persist result, return JSON
  res.json(result);
});

export default router;
```

**3. Register the router in `artifacts/api-server/src/routes/index.ts`.**

```typescript
import myFeatureRouter from "./my-feature";
// ...
router.use(myFeatureRouter);
```

**4. Add the endpoint to the OpenAPI spec.**

Open `lib/api-spec/openapi.yaml`. Add a path entry following the existing pattern:

```yaml
/my-feature/{id}/action:
  post:
    operationId: myFeatureAction
    summary: Run the new AI analysis
    parameters:
      - name: id
        in: path
        required: true
        schema:
          type: integer
    responses:
      '200':
        description: Analysis result
        content:
          application/json:
            schema:
              type: object
              properties:
                summary:
                  type: string
                flagged_items:
                  type: array
                  items:
                    type: string
```

**5. Regenerate the API client.**

```bash
pnpm --filter @workspace/api-client-react run generate
```

This produces a `useMyFeatureAction` hook in `lib/api-client-react/src/generated/api.ts`.

**6. Use the hook in the frontend.**

```tsx
import { useMyFeatureAction } from "@workspace/api-client-react";

const { mutate, isPending } = useMyFeatureAction();

<Button onClick={() => mutate({ id: attachmentId })} disabled={isPending}>
  Run Analysis
</Button>
```

---

## C. Adding a New Leak Source (wikiran.org as example)

The ingest pipeline is source-agnostic — any collection of email data can be normalized
to the `ParsedEmail` interface and passed to `ingestParsedEmails`. The steps below use
**wikiran.org as a worked example**; substitute your own source's API, file format, or
database export at each step.

> **Note on example data source:** wikiran.org was chosen as the example because its
> archives are publicly available and do not contain information considered classified in
> Western or English-speaking countries. This tool is not intended to target Iran or
> Iranian individuals; it is a generic framework applicable to any publicly available
> leaked email corpus.

New datasets are imported using the seed workflow. The relevant script is in `scripts/`.
The process is: identify the dataset slug on wikiran.org (or the equivalent identifier
for your source), fetch records via the public API, normalize to `ParsedEmail` format,
and call the ingest pipeline.

### Steps

**1. Find the dataset slug.**

Visit `https://wikiran.org/leaks`. Each dataset card links to
`https://wikiran.org/leaks/<slug>`. The slug is the last path segment (e.g. `sadaf`,
`ifc`, `amin`). Confirm the record count via the public API:

```bash
curl https://wikiran.org/api/v1/leaks/<slug>
```

**2. Check the dataset record type.**

See `WIKIRAN-ORG-DATA-SPEC.md` section 6 for wire format details. Email datasets
(`leakType = "email"`) are supported by the current ingest pipeline. Transaction tables
(`leakType = "table"`) and OCR-only datasets require a different importer.

**3. Fetch and normalize records.**

The wikiran.org API paginates with `?page=<n>&size=<n>`. Fetch records in batches and
map each record to the `ParsedEmail` interface:

```typescript
function wikIranRecordToEmail(record: WikIranEmailRecord): ParsedEmail {
  return {
    messageId: undefined,              // wikiran.org does not expose Message-ID
    subject: record.subject ?? undefined,
    fromAddress: extractAddress(record.from),
    fromName: extractName(record.from),
    toAddresses: record.to ? parseAddressList(record.to) : [],
    ccAddresses: [],
    date: parseWikIranDate(record.date), // handles both "Month DD, YYYY" and ISO 8601
    bodyText: record.body ?? undefined,
    bodyHtml: undefined,
    attachments: [],
  };
}
```

**4. Create the dataset and call `ingestParsedEmails`.**

```typescript
import { db, datasetsTable } from "@workspace/db";
import { ingestParsedEmails } from "../lib/ingest";

const [dataset] = await db
  .insert(datasetsTable)
  .values({ name: "Sadaf Exchange 2022", description: "Imported from wikiran.org/leaks/sadaf" })
  .returning();

const result = await ingestParsedEmails(dataset.id, normalizedEmails);
console.log(`Ingested: ${result.ingested}, duplicates: ${result.duplicates}`);
```

**5. Run crypto scan and AI identity extraction.**

After ingestion, trigger the crypto scanner and identity extractor via the API:

```bash
TOKEN=<your-jwt>
DATASET_ID=<new-id>

curl -X POST https://<host>/api/crypto/scan \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"dataset_id\": $DATASET_ID}"

curl -X POST https://<host>/api/identities/extract \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"dataset_id\": $DATASET_ID}"
```

**6. Verify in the UI.**

Select the new dataset from the sidebar dropdown. The dashboard should show email counts,
timelines, and entities.

---

## D. Adding a New Frontend Page

Adding a page means: create the page component, add a route in `App.tsx`, add a nav item
in the sidebar layout, and (if needed) add an API hook.

### Steps

**1. Create the page component.**

Create a new file in `artifacts/wikiran/src/pages/`. Follow the naming convention
`kebab-case.tsx` (e.g. `sanctions-map.tsx`). The component must be a default export.

Minimal template:
```tsx
import { useAppState } from "@/hooks/use-app-state";

export default function SanctionsMap() {
  const { datasetId } = useAppState();

  return (
    <div className="p-6">
      <h1 className="text-2xl font-bold text-foreground mb-6">Sanctions Map</h1>
      {/* page content */}
    </div>
  );
}
```

**2. Add the route in `App.tsx`.**

Open `artifacts/wikiran/src/App.tsx`. Import your new component and add a `<Route>` inside
the `<Switch>`:

```tsx
import SanctionsMap from "@/pages/sanctions-map";

// Inside <Switch>:
<Route path="/sanctions" component={SanctionsMap} />
```

Route order matters in wouter: put more specific paths before catch-all routes. The
`<Route component={NotFound} />` must remain last.

**3. Add a navigation entry in the sidebar.**

Open the layout component (`artifacts/wikiran/src/components/layout.tsx` or similar).
Add a nav item with an icon and label. Import a Lucide icon:

```tsx
import { Shield } from "lucide-react";

// In the nav items array:
{ href: "/sanctions", label: "Sanctions Map", icon: Shield },
```

**4. Add API hooks if needed.**

If the page requires a new backend endpoint, follow the steps in section B to add the
route, update the OpenAPI spec, and regenerate the client. Then use the generated hook:

```tsx
import { useListSanctionsEntities } from "@workspace/api-client-react";

const { data, isLoading } = useListSanctionsEntities({ dataset_id: datasetId });
```

**5. Test navigation.**

Use wouter's `useLocation` for programmatic navigation (not `window.location.href`):

```tsx
import { useLocation } from "wouter";

const [, navigate] = useLocation();
navigate(`/emails?dataset_id=${datasetId}`);
```

Do not use `react-router-dom` — it is not installed. The router library is exclusively
wouter.

**6. Update `replit.md`.**

Add a note about the new page to `replit.md` under the features section so future
sessions have an accurate map of the application.
