# Stage 1 — Data Ingestion & Spreadsheet
### Architecture, Data Model, and Design Decisions

---

## 1. What Is This?

A CLAY-inspired data enrichment platform. The core idea: bring messy data from anywhere — a CSV export, an Excel sheet, a live Google Sheet, a JSON dump — normalize it into a structured table, let users clean and map it, then view and edit it like a spreadsheet.

Stage 1 covers everything up to the point where data is live in a spreadsheet and editable. Stage 2 adds automatic enrichment (filling empty cells via external APIs).

---

## 2. Clay vs. Our Version — Key Differences

Understanding real Clay first helps explain every design decision we made.

### What real Clay does

Clay is a B2B SaaS product. It is:
- **Multi-tenant** — thousands of customers, each with their own workspace, billing, user accounts
- **Cloud-hosted** — data stored in Postgres on AWS, queued jobs on Redis/SQS
- **API-key based** — Hunter, Apollo, Clearbit, etc. are connected via user-provided keys in a settings panel
- **Real-time collaborative** — multiple users editing the same table simultaneously via websockets
- **Paid enrichment** — each provider call costs credits; Clay tracks usage per user
- **Complex UI** — column formulas, conditional logic, nested waterfalls, row filtering

### What our version does differently

| Dimension | Real Clay | Our Version |
|---|---|---|
| **Users** | Multi-tenant SaaS, auth, workspaces | Single-user, no auth (local tool) |
| **Database** | Postgres + Redis | SQLite (single file, zero ops) |
| **Enrichment providers** | 50+ real API integrations | Stubbed — structure is real, calls are placeholders |
| **Real-time** | Websocket collaboration | React Query polling (good enough for solo use) |
| **Billing** | Credit system per API call | None |
| **Formulas** | Column formulas like `=domain(website)` | Not implemented — plain data only |
| **Scale** | Millions of rows, sharded | Designed for tens of thousands of rows |
| **Deployment** | Cloud SaaS | Runs locally via `npm run dev` |

### What we kept the same conceptually

- Dynamic columns stored as JSON (no DB migrations to add a column)
- Row values keyed by column ID (not name), so renames never break data
- The import → mapping → table flow
- Enrichment waterfall pattern (Provider A → B → C until one succeeds)
- Separation of enrichment attempt logs from the actual row data

The structural decisions are identical to what Clay would do internally. We just stripped out everything that requires infrastructure (auth, queues, billing, multi-tenancy) to keep it runnable as a local tool.

---

## 3. Database Tables — Full Breakdown

There are 4 tables. Here is each one in detail.

---

### Table 1: `tables`

This is the spreadsheet container. One row here = one spreadsheet the user has created.

```
tables
├── id           TEXT     Primary key. cuid (e.g. "clx8f2j3k0000...")
├── name         TEXT     User-facing name. e.g. "ICP Companies Q2"
├── description  TEXT?    Optional notes about the table
├── entityType   TEXT     "company" | "person" | "generic"
├── columnDefs   TEXT     JSON blob — the entire schema lives here
├── createdAt    DATETIME Auto-set on insert
└── updatedAt    DATETIME Auto-updated on every change
```

**The `columnDefs` field is the most important design decision in the whole system.**

Instead of having a separate `columns` table with one row per column, the entire column schema is a single JSON array stored on the table row. Each element is a `ColumnDefinition`:

```json
[
  {
    "id":    "col_m3k9x2",
    "name":  "Company Name",
    "type":  "text",
    "order": 0,
    "width": 200
  },
  {
    "id":    "col_p7w1z4",
    "name":  "Domain",
    "type":  "url",
    "order": 1,
    "width": 180
  },
  {
    "id":    "col_r5n8q1",
    "name":  "CEO Email",
    "type":  "enrichment",
    "order": 2,
    "width": 220,
    "enrichmentConfig": {
      "providers": [
        { "name": "hunter",  "targetField": "email" },
        { "name": "apollo",  "targetField": "email" },
        { "name": "scraper", "targetField": "email" }
      ],
      "stopOnFirst": true
    }
  }
]
```

**Why JSON instead of a separate `columns` table?**

With a separate table you'd need:
- `INSERT` to add a column
- `UPDATE` to rename it
- `DELETE` to remove it
- A `JOIN` every time you load the table schema

With JSON:
- Add a column → one `UPDATE` on the table row
- Rename → one `UPDATE`
- Delete → one `UPDATE`
- Load schema → already in the table row, no join needed
- Reorder columns → update the `order` field in place
- Schema changes require zero DB migrations

The tradeoff: you can't efficiently query "all tables that have a column named X" — but we never need to.

**Column types:**

| Type | What it means in the grid |
|---|---|
| `text` | Plain string, editable |
| `number` | Numeric, stored as number in JSON |
| `url` | String, rendered as a link |
| `email` | String, validated as email format |
| `phone` | String, phone formatting |
| `enrichment` | Read-only in grid, filled by waterfall job |

---

### Table 2: `rows`

One row here = one record in a spreadsheet (one company, one person, etc.).

```
rows
├── id        TEXT     Primary key. cuid
├── tableId   TEXT     Foreign key → tables.id (CASCADE DELETE)
├── values    TEXT     JSON blob — all cell values for this row
├── createdAt DATETIME
└── updatedAt DATETIME
```

**The `values` field is the other half of the schema-less design.**

Every cell value for the row is stored in a single JSON object, keyed by the **column ID** (not the column name):

```json
{
  "col_m3k9x2": "Acme Corp",
  "col_p7w1z4":  "acme.com",
  "col_r5n8q1":  null
}
```

**Why keyed by column ID and not column name?**

If you stored `{ "Company Name": "Acme Corp" }` and the user renamed the column to "Organization", every single row in the table would need to be updated. With cuid keys:
- Rename "Company Name" → "Organization" = one update to `columnDefs` JSON. Row data untouched.
- Delete a column = one update to `columnDefs`. The orphaned key in each row's JSON is harmless.
- Add a column = one update to `columnDefs`. Existing rows just have no key for it — treated as empty.

**CellValue types:**

```
string | number | null | string[]
```

`string[]` is used when enrichment returns multiple results — for example, Apollo finding three people at a company. The grid renders it as a comma-joined string.

**Updating a cell:**

When a user edits a cell, the server reads the existing `values`, spreads the new value in, and writes back:

```
existing = { "col_m3k9x2": "Acme Corp", "col_p7w1z4": "acme.com" }
patch    = { "col_p7w1z4": "acme.io" }
result   = { "col_m3k9x2": "Acme Corp", "col_p7w1z4": "acme.io" }
```

This means partial updates are safe — only the changed key is in the PATCH request.

---

### Table 3: `enrichment_results`

Every time the enrichment engine tries to fill a cell — whether it succeeds or fails — a record is written here.

```
enrichment_results
├── id          TEXT    Primary key. cuid
├── rowId       TEXT    FK → rows.id (CASCADE DELETE)
├── columnId    TEXT    Which column was being enriched (e.g. "col_r5n8q1")
├── provider    TEXT    "hunter" | "apollo" | "scraper"
├── status      TEXT    "success" | "not_found" | "error" | "rate_limited"
├── value       TEXT?   JSON-encoded result value (or null if not found)
├── confidence  REAL?   0.0 – 1.0 confidence score from the provider
├── rawResponse TEXT?   Full API response JSON stored for debugging
└── createdAt   DATETIME
```

**This table is a pure audit log.** It never drives the UI directly. Its purposes:

1. **Audit trail** — "Which provider filled this cell? When? With what confidence?"
2. **Retry logic** — Only retry rows where `status = 'not_found'` or `'error'`. Skip rows that already have `status = 'success'` to avoid burning API credits.
3. **Cost tracking** — Count rows per provider per table to understand spend.
4. **Debugging** — The full `rawResponse` is stored so you can inspect what an API actually returned.

The actual cell value lives in `rows.values`. This table just records the history of how it got there.

---

### Table 4: `enrichment_jobs`

A background job tracker. One row = one "run enrichment on this column for this table" operation.

```
enrichment_jobs
├── id            TEXT      Primary key. cuid
├── tableId       TEXT      FK → tables.id (CASCADE DELETE)
├── columnId      TEXT      Which enrichment column is being run
├── status        TEXT      "pending" | "running" | "completed" | "failed"
├── totalRows     INTEGER   How many rows need to be processed
├── processedRows INTEGER   Progress counter — incremented after each row
├── error         TEXT?     Error message if status = "failed"
├── startedAt     DATETIME? Set when processing begins
├── completedAt   DATETIME? Set when done or failed
└── createdAt     DATETIME
```

**Why a separate job table and not just a status field on the column?**

A column can be enriched multiple times (re-runs, partial re-runs). Storing job state on the column would only let you track one run at a time. This table gives you a complete history of every enrichment run, when it ran, how far it got, and whether it succeeded.

The frontend polls `GET /api/enrichment/[jobId]` every second while `status = 'running'` and shows a progress bar: `247 / 1247 rows`. When it flips to `completed`, the query cache is invalidated and the grid refreshes with newly filled cells.

---

## 4. How the Tables Connect

```
tables (1)
  │
  ├──< rows (many)
  │       │
  │       └──< enrichment_results (many)
  │                (one per provider attempt per column per row)
  │
  └──< enrichment_jobs (many)
          (one per "run enrichment on column X" click)
```

- A `Table` owns many `Rows`
- A `Row` owns many `EnrichmentResults` (one per provider attempt)
- A `Table` owns many `EnrichmentJobs` (one per enrichment run)
- `EnrichmentResults` and `EnrichmentJobs` both reference `columnId` as a plain string — not a foreign key — because columns are not a DB table. They live in `tables.columnDefs` JSON.
- All child records CASCADE DELETE when their parent `Table` or `Row` is deleted

---

## 5. Ingestion Flow — Step by Step

### Step 1: Source → ParsedSource (server-side)

```
User picks:  CSV file  /  Excel file  /  Google Sheets URL  /  JSON paste
                 │
                 ▼
         POST /api/ingest/parse
         (multipart form for files, JSON body for URLs/text)
                 │
         ┌───────┴──────────────────────────────────┐
         │  CSV:    PapaParse → row 0 = headers      │
         │  Excel:  xlsx.read → Sheet 1 → rows       │
         │  Sheets: server fetches ?format=csv → CSV │
         │  JSON:   flatten objects → keys=headers   │
         └───────────────────────────────────────────┘
                 │
                 ▼
         Returns: ParsedSource
         {
           headers:   ["Company", "Website", "Email Addr", "Region"]
           sampleRows: [ first 5 rows for preview ]
           totalRows:  1247
           allRows:    [ all 1247 rows ]
         }
```

All four sources arrive at the same `ParsedSource` shape. Everything downstream is source-agnostic.

**Why server-side parsing?**

- Google Sheets URL would fail from the browser (CORS). The server fetches it.
- Large Excel files (50MB+) shouldn't be parsed in the browser — it freezes the tab.
- Keeps parser libraries (PapaParse, xlsx) out of the client bundle.

---

### Step 2: Column Mapping

The user sees a table — one row per source column:

```
Source Column      Maps To                    Sample Data
─────────────      ─────────────────────      ──────────────
"Company"      →   [Company Name ▼]           Acme Corp · Stripe
"Website"      →   [Domain ▼]                 acme.com · stripe.com
"Email Addr"   →   [Email ▼]                  ceo@acme.com
"Region"       →   [Custom: "Region" ▼]       West · East
"Internal ID"  →   [— Skip — ▼]              10042 · 10043
```

The dropdown has three zones:

```
— Skip —
──── Standard Fields ────
  Company Name
  Domain
  LinkedIn (Company)
  Industry
  Employee Count
  HQ Location
  Description
──── Custom ────
  Custom field…  (shows a name input when selected)
```

**Auto-suggest:** On load, each source column name is normalized (lowercased, trimmed) and matched against alias lists. "Website" matches the alias list for `domain`. "Email Addr" matches `email`. This pre-fills the dropdowns so users usually only need to fix a few mappings rather than set all of them from scratch.

**Custom field:** If a source column has no standard equivalent, the user can name it anything. It becomes a `type: 'text'` column with whatever name they give it.

**Skip:** Excluded columns are dropped. Their data is never written to the DB.

---

### Step 3: Import (one request, one transaction)

```
PUT /api/tables
{
  tableName: "ICP Companies Q2",
  entityType: "company",
  columnMappings: [
    { sourceColumn: "Company",     targetField: "companyName", isCustom: false },
    { sourceColumn: "Website",     targetField: "domain",      isCustom: false },
    { sourceColumn: "Email Addr",  targetField: "email",       isCustom: false },
    { sourceColumn: "Region",      targetField: "region",      isCustom: true, customName: "Region" },
    { sourceColumn: "Internal ID", targetField: null }
  ],
  allRows: [ ...1247 string arrays... ]
}
```

Server does:

```
1. For each active mapping (targetField !== null):
      assign a new cuid → this is the permanent column ID
      build a ColumnDefinition object

2. INSERT into tables:
      columnDefs = JSON.stringify(the array of ColumnDefinitions)

3. For each source row:
      build values = {}
      for each mapping at source index i:
           values[colId] = row[i]
      INSERT into rows: { tableId, values: JSON.stringify(values) }

4. prisma.row.createMany() — bulk insert, one DB round-trip
```

The result is a fully populated table. The user is redirected to `/tables/[id]`.

---

## 6. Spreadsheet View

The spreadsheet is AG Grid Community Edition. The column definitions from `tables.columnDefs` drive the grid columns. The rows from `rows.values` drive the grid rows.

**Data transformation for the grid:**

AG Grid needs flat objects. A row from the DB looks like:

```json
{ "id": "row_abc", "tableId": "tbl_xyz", "values": { "col_m3k9x2": "Acme", "col_p7w1z4": "acme.com" } }
```

It gets flattened to:

```json
{ "_rowId": "row_abc", "col_m3k9x2": "Acme", "col_p7w1z4": "acme.com" }
```

The `_rowId` is stored on each grid row so we know which DB row to PATCH on edit.

**Cell edit flow:**

```
User double-clicks cell  →  types new value  →  clicks away
         │
         ▼
AG Grid fires onCellValueChanged
         │
         ▼
Optimistic update: React Query cache updated immediately
(grid feels instant — no waiting for server)
         │
         ▼
PATCH /api/tables/[id]/rows/[rowId]
{ values: { "col_p7w1z4": "acme.io" } }
         │
         ▼
Server: spread-merges into existing values, saves
         │
  success → nothing to do, cache already updated
  failure → cache invalidated, grid reverts
```

**Adding a column:**

```
+ Column button  →  modal: name + type  →  confirm
         │
         ▼
POST /api/tables/[id]/columns
{ name: "Revenue", type: "text" }
         │
         ▼
Server: appends new ColumnDefinition to columnDefs JSON
(existing rows have no key for this column — grid shows blank)
         │
         ▼
React Query invalidates ['table', id]  →  grid re-renders with new column
```

No rows are touched. No migration. The new column just shows as blank for existing rows.

---

## 7. API Surface

| Endpoint | Method | What it does |
|---|---|---|
| `/api/ingest/parse` | POST | Parse any source → `ParsedSource` |
| `/api/tables` | GET | List all tables with row counts |
| `/api/tables` | POST | Create an empty table |
| `/api/tables` | PUT | Create table + import rows in one shot |
| `/api/tables/[id]` | GET | Get table with `columnDefs` |
| `/api/tables/[id]` | PATCH | Rename table / update description |
| `/api/tables/[id]` | DELETE | Delete table, all rows, all jobs |
| `/api/tables/[id]/columns` | POST | Add a new column |
| `/api/tables/[id]/columns` | PATCH | Rename or resize a column |
| `/api/tables/[id]/columns` | DELETE | Delete a column (data orphaned in rows, harmless) |
| `/api/tables/[id]/rows` | GET | Paginated rows (page + limit params) |
| `/api/tables/[id]/rows` | POST | Add a blank row |
| `/api/tables/[id]/rows/[rowId]` | PATCH | Update one or more cell values |
| `/api/tables/[id]/rows/[rowId]` | DELETE | Delete a row |
| `/api/enrichment` | POST | Start an enrichment job (Stage 2) |
| `/api/enrichment/[jobId]` | GET | Poll job progress (Stage 2) |

All responses follow a consistent envelope:

```json
{ "success": true,  "data": { ... } }
{ "success": false, "error": "Table not found" }
{ "success": true,  "data": [...], "meta": { "total": 1247, "page": 1, "limit": 200 } }
```

---

## 8. Tech Stack and Why

| Layer | Choice | Why |
|---|---|---|
| **Framework** | Next.js 14 App Router | API routes and React UI in one project, no separate backend |
| **Database** | SQLite via Prisma | Zero ops — single file on disk, perfect for a local tool |
| **ORM** | Prisma | Type-safe queries, readable schema, easy migrations |
| **Spreadsheet** | AG Grid Community | Free, handles thousands of rows without virtualisation issues, inline editing built-in |
| **Data fetching** | TanStack Query | Optimistic updates, cache management, polling for job status |
| **Validation** | Zod | Schema defined once, TypeScript types inferred from it |
| **CSV parsing** | PapaParse | Handles encoding, quoted fields, malformed CSV gracefully |
| **Excel parsing** | xlsx | Reads .xlsx and .xls, no native dependencies |
| **Styling** | Tailwind CSS | No component library needed — keeps bundle small |
| **IDs** | cuid2 | Collision-resistant, URL-safe, sortable — better than UUID for this use case |

---

## 9. Why JSON — and Where It Stops

JSON appears in two places in this system. Understanding exactly what it is doing (and what it is not doing) is important before assuming it will cause problems later.

### What JSON is doing

**Job 1 — Store a flexible schema (`tables.columnDefs`)**

Every table has different columns. A companies table might have 5, a contacts table might have 12, all different names and types. The alternative is a separate `columns` DB table:

```
columns
├── id
├── tableId  (FK)
├── name
├── type
├── order
└── width
```

But then every column add, rename, reorder, or delete is a DB operation on that table. And every time you load a table you need a JOIN just to get the schema.

JSON makes the schema travel with the table — one read gets you everything:

```json
{ "id": "tbl_x", "name": "ICP Companies", "columnDefs": [...] }
```

This is not a workaround. It is the right tool for a variable-length schema that changes frequently. Real Clay uses the same approach.

**Job 2 — Store flexible cell values (`rows.values`)**

Each row has values for columns that do not exist as actual DB columns. You cannot do:

```sql
ALTER TABLE rows ADD COLUMN "CEO Email" TEXT;  -- every time a user adds a column
```

So instead, the entire row is one JSON blob keyed by column ID:

```json
{ "col_abc": "Acme Corp", "col_def": "acme.com", "col_ghi": null }
```

Read the whole row in one DB read. Update one cell by merging one key. No schema migration ever needed.

### What JSON is NOT doing

JSON is not storing anything that needs to be queried across rows. That is why `enrichment_results` is a fully relational table with real columns — because you need to ask questions across thousands of records:

```sql
-- Which provider fills cells most? Works perfectly.
SELECT provider, status, COUNT(*)
FROM enrichment_results
GROUP BY provider, status

-- Success rate per provider across all runs?
SELECT provider,
  SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END) * 100.0 / COUNT(*) AS success_rate
FROM enrichment_results
GROUP BY provider
```

These queries work because `provider`, `status`, `columnId` are actual DB columns — not buried inside JSON. The enrichment audit log was designed relational specifically so waterfall analytics are always queryable without any workaround.

### The rule

> JSON stores **structure that changes** — schema and cell values, read and written as a whole.
> Relational rows store **data that gets analyzed** — enrichment attempts, job status, progress.

The moment something needs to be queried, filtered, counted, or aggregated across many records — it gets its own column. The moment something is just loaded and saved as a unit — JSON is simpler and faster.

### Known limitation and the fix

Server-side filtering of row values — "show me all rows where `industry = SaaS`" — requires `json_extract(values, '$.col_abc')` in SQLite, which is slow without an index on large tables. For Stage 1, filtering and sorting happen client-side in AG Grid, which handles up to ~10k rows without issue.

If server-side filtering becomes necessary later, the fix is a one-line Prisma migration to add a generated/virtual column on `rows`, or a switch to Postgres with native `jsonb` indexing. The application code does not change.

---

## 10. Key Design Principles (Summary)

**Schema lives in JSON, not in DB columns.**
Adding or removing a column never requires a DB migration. The `columnDefs` JSON is the schema. This is the same approach Clay uses internally and is why their product feels so fluid.

**Data lives keyed by ID, not by name.**
User-visible names are cosmetic. The actual storage keys are immutable cuids. Rename freely — nothing breaks.

**Server-side parsing for all sources.**
Files, Google Sheets, JSON — all parsed on the server. The client just shows results and captures user decisions.

**Optimistic updates everywhere in the grid.**
Cell edits feel instant. The server call happens in the background. On failure, the cache is invalidated and the grid reverts. This matches how Clay's grid behaves.

**Audit log separate from live data.**
`enrichment_results` records every provider attempt. The live value in `rows.values` is just the latest winner. This separation means you can always inspect the history, retry selectively, and track provider costs — without polluting the main data model.

---

## 10. What's Next — Stage 2

Stage 2 adds the enrichment waterfall:

- Enrichment column type is already modelled in `columnDefs`
- `enrichment_jobs` and `enrichment_results` tables are already created
- Provider stubs (`hunter`, `apollo`, `scraper`) are in place
- Stage 2 replaces the stubs with real API calls
- Frontend progress bar and polling are already implemented

The data model does not change between Stage 1 and Stage 2. Stage 2 is purely filling in the provider implementations.
