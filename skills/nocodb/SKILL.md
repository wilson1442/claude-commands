---
name: nocodb-crud
description: >
  Scaffold full NocoDB CRUD integration into any web project. Use this skill whenever
  the user wants to: connect a site to NocoDB, add a NocoDB backend to a project, build
  CRUD operations against a NocoDB table, scaffold a NocoDB API layer, integrate NocoDB
  with Next.js / Express / React / Node, or any variation of "hook up NocoDB", "connect
  to my NocoDB table", "add NocoDB to this project", "I need CRUD for NocoDB", or
  "scaffold NocoDB integration". Also trigger when the user mentions a NocoDB table by
  name or ID and wants to read/write/update/delete records from code. Trigger aggressively:
  if the user is building a web app and mentions NocoDB, this skill should fire. Do NOT
  use for NocoDB admin tasks (creating tables, managing users, etc.) — this is strictly
  for wiring up data CRUD from application code.
---

# NocoDB CRUD Skill

## Purpose

This skill scaffolds a complete, production-ready NocoDB integration into the user's
project. It auto-detects the framework, generates server-side API route handlers (to
keep the NocoDB API key hidden), and provides a full-featured client with pagination,
filtering, sorting, and bulk operations.

---

## Step 1: Gather Configuration

Before generating any code, prompt the user for these three values. Do not proceed
until all three are provided:

| Variable | What to ask | Notes |
|---|---|---|
| `NOCODB_URL` | "What's your NocoDB instance URL?" | e.g. `https://nocodb.example.com` — no trailing slash |
| `NOCODB_TABLE_ID` | "What's the NocoDB table ID?" | Alphanumeric string prefixed with `m` (e.g. `m3k7x9abc123`). Found in the URL or table context menu. |
| `NOCODB_API_KEY` | "What's your NocoDB API token?" | Created under Account Settings → API Tokens in NocoDB UI. |

Also ask: **"What are you building? Give me a quick description so I can tailor the integration."**
This context helps you name functions, add relevant comments, and wire up the right patterns.

Additionally, ask: **"Should I auto-create the table fields in NocoDB for you? If so, what fields do you need?"**

If the user wants fields auto-created, proceed to **Step 1b** below. Otherwise skip to Step 2.

---

## Step 1b: Auto-Create Table Fields

When the user wants fields auto-created in NocoDB, use the NocoDB meta API to create
columns and then make them visible in the default view.

### Creating columns

Use `POST /api/v2/meta/tables/{TABLE_ID}/columns` for each field:

```
POST {NOCODB_URL}/api/v2/meta/tables/{TABLE_ID}/columns
Headers: xc-token: {API_KEY}, Content-Type: application/json
Body: { "title": "FieldName", "uidt": "SingleLineText", "column_name": "FieldName" }
```

Common `uidt` values:
| Type | `uidt` value |
|---|---|
| Short text | `SingleLineText` |
| Long text | `LongText` |
| Email | `Email` |
| Phone | `PhoneNumber` |
| Number | `Number` |
| Checkbox | `Checkbox` |
| Date | `Date` |
| URL | `URL` |
| Single select | `SingleSelect` |

If the table already has a default `Title` column that should be repurposed (e.g. renamed
to `Name`), use `PATCH /api/v2/meta/columns/{COLUMN_ID}` to rename it instead of creating
a duplicate.

### Making columns visible in the default view

**This is critical.** Newly created columns are hidden in the default view by default.
After creating columns, you must:

1. Get the default view ID from the table metadata:
   ```
   GET {NOCODB_URL}/api/v2/meta/tables/{TABLE_ID}
   ```
   The default view is in `response.views[0].id`.

2. Get all view columns:
   ```
   GET {NOCODB_URL}/api/v2/meta/views/{VIEW_ID}/columns
   ```

3. For each column where `show` is `false`, enable it:
   ```
   PATCH {NOCODB_URL}/api/v2/meta/views/{VIEW_ID}/columns/{VIEW_COLUMN_ID}
   Body: { "show": true }
   ```

Always verify the columns are visible after creation by checking the view columns
endpoint. Do not skip this step — users will see a blank table otherwise.

---

## Step 2: Detect the Project Framework

Inspect the project root to determine the stack. Use this detection order:

```
1. next.config.js / next.config.mjs / next.config.ts  →  Next.js (check for app/ vs pages/)
2. nuxt.config.ts / nuxt.config.js                     →  Nuxt
3. remix.config.js / app/root.tsx with @remix-run       →  Remix
4. package.json with "express" in dependencies          →  Express
5. package.json with "fastify" in dependencies          →  Fastify
6. package.json with "react" but no meta-framework      →  React SPA (needs proxy backend)
7. package.json with "vue" but no meta-framework        →  Vue SPA (needs proxy backend)
8. manage.py / requirements.txt with flask/fastapi      →  Python (Flask/FastAPI)
9. No framework detected                                →  Generate standalone Node.js module
```

Tell the user what you detected and confirm before generating.

---

## Step 3: Generate the Integration

### What gets generated (all frameworks)

Every scaffold produces these layers:

**A) Environment config**
- Add `NOCODB_URL`, `NOCODB_TABLE_ID`, `NOCODB_API_KEY` to `.env` (create if missing)
- Add the same keys (without values) to `.env.example` (create if missing)
- Add `.env` to `.gitignore` if not already present

**B) NocoDB API client** — a server-side-only module
- Full CRUD: `listRecords`, `getRecord`, `createRecord`, `updateRecord`, `deleteRecord`
- Bulk operations: `bulkCreate`, `bulkUpdate`, `bulkDelete`
- Pagination: `offset` + `limit` params, returns `{ list, pageInfo }`
- Filtering: accepts NocoDB `where` clause strings
- Sorting: accepts `sort` param (e.g. `"-CreatedAt"` for descending)
- Field selection: accepts `fields` param
- Error handling: typed errors with HTTP status, message, and raw response

**C) Server-side API route handlers**
- `GET    /api/nocodb`         → list records (query params: limit, offset, where, sort, fields)
- `GET    /api/nocodb/:id`     → get single record
- `POST   /api/nocodb`         → create record(s) — single object or array for bulk
- `PATCH  /api/nocodb`         → update record(s) — must include `Id` field
- `DELETE /api/nocodb`         → delete record(s) — body: `{ ids: [1, 2, 3] }` or `{ id: 1 }`

**D) Client-side helper (optional, for SPAs)**
- Typed fetch wrapper that calls the server-side routes (never the NocoDB API directly)
- Provides the same CRUD interface but safe for browser use

### Framework-specific generation

Read the appropriate template reference before generating:

| Framework | Template Reference |
|---|---|
| Next.js (App Router) | `references/nextjs-app.md` |
| Next.js (Pages Router) | `references/nextjs-pages.md` |
| Express / Fastify | `references/express.md` |
| React / Vue SPA | `references/spa.md` |
| Python (FastAPI/Flask) | `references/python.md` |
| Standalone Node.js | `references/standalone.md` |

---

## Step 4: Wire It Up

After generating files, provide the user with:

1. **What was created** — list every file with a one-line description
2. **Environment setup** — remind them to fill in `.env` values
3. **Usage examples** — show 3-4 practical examples using their specific use case context:
   - List records with pagination
   - Create a record
   - Update a record
   - Delete a record
   - Filter/sort example
4. **NocoDB `where` clause cheat sheet** — quick reference for common filters

---

## NocoDB v2 API Reference (for code generation)

This is the canonical API surface to target. All generated code must use these endpoints.

### Authentication

Two equivalent methods (use `xc-token` as primary):
```
xc-token: <api_token>
Authorization: Bearer <api_token>
```

### Endpoints

Base: `{NOCODB_URL}/api/v2/tables/{TABLE_ID}`

| Method | Path | Body | Description |
|---|---|---|---|
| `GET` | `/records` | — | List records. Query params: `limit`, `offset`, `where`, `sort`, `fields`, `viewId` |
| `GET` | `/records/:rowId` | — | Get single record by row ID |
| `POST` | `/records` | `{ field: value }` or `[{ field: value }, ...]` | Create one or many records |
| `PATCH` | `/records` | `{ Id: N, field: value }` or `[{...}, ...]` | Update one or many records (Id required) |
| `DELETE` | `/records` | `{ Id: N }` or `[{ Id: N }, ...]` | Delete one or many records |

### Query Parameters for GET /records

| Param | Type | Example | Description |
|---|---|---|---|
| `limit` | number | `25` | Records per page (default 25, max 1000) |
| `offset` | number | `0` | Skip N records |
| `where` | string | `(Status,eq,Active)` | Filter expression |
| `sort` | string | `-CreatedAt` | Sort field (prefix `-` for descending) |
| `fields` | string | `Name,Email,Status` | Comma-separated field names to return |
| `viewId` | string | `vw3abc123` | Optional view filter |

### Where Clause Operators

| Operator | Meaning | Example |
|---|---|---|
| `eq` | equals | `(Status,eq,Active)` |
| `neq` | not equals | `(Status,neq,Archived)` |
| `gt` / `lt` | greater/less than | `(Amount,gt,100)` |
| `gte` / `lte` | greater/less or equal | `(Score,gte,80)` |
| `like` | contains | `(Name,like,%john%)` |
| `nlike` | not contains | `(Name,nlike,%test%)` |
| `is` | is null/empty | `(Email,is,null)` |
| `isnot` | is not null/empty | `(Email,isnot,null)` |
| `in` | in list | `(Status,in,Active,Pending)` |
| `btw` | between | `(Amount,btw,10,100)` |
| `~and` | AND combiner | `(Status,eq,Active)~and(Amount,gt,50)` |
| `~or` | OR combiner | `(Status,eq,Active)~or(Status,eq,Pending)` |

### Response Format

```json
{
  "list": [
    { "Id": 1, "Name": "Example", "Status": "Active", ... }
  ],
  "pageInfo": {
    "totalRows": 150,
    "page": 1,
    "pageSize": 25,
    "isFirstPage": true,
    "isLastPage": false
  }
}
```

---

## Error Handling Patterns

All generated code should handle these error cases:

| Status | Meaning | Action |
|---|---|---|
| `401` | Invalid or missing API token | Throw auth error, log hint to check NOCODB_API_KEY |
| `404` | Table or record not found | Throw not-found error with the ID that failed |
| `422` | Validation error (bad field, missing required) | Pass through NocoDB's error message |
| `429` | Rate limited | Retry with exponential backoff (3 attempts) |
| `5xx` | NocoDB server error | Throw with raw response for debugging |

---

## Important Notes

- **API key must NEVER be exposed to the browser.** All NocoDB calls go through server-side routes.
- **The PATCH endpoint takes the record body (with Id) directly**, not as a URL parameter. This is a common gotcha — NocoDB v2 does NOT use `PATCH /records/:id`.
- **Bulk operations** send arrays in the request body to the same endpoints.
- **The `Id` field** is NocoDB's internal row ID (numeric, starting from 1). It must be included in update/delete payloads.
- **TypeScript**: If the project uses TypeScript (detected via `tsconfig.json`), generate `.ts` files with proper types. Otherwise generate `.js`.
- **Field names**: NocoDB field names are case-sensitive and may contain spaces. The generated code should handle this gracefully.

---

## File Placement Guide

Place generated files following the project's existing conventions:

| Framework | API Client | Routes | Client Helper |
|---|---|---|---|
| Next.js (App Router) | `lib/nocodb.ts` | `app/api/nocodb/route.ts` + `app/api/nocodb/[id]/route.ts` | `lib/nocodb-client.ts` |
| Next.js (Pages Router) | `lib/nocodb.ts` | `pages/api/nocodb/index.ts` + `pages/api/nocodb/[id].ts` | `lib/nocodb-client.ts` |
| Express | `lib/nocodb.js` | `routes/nocodb.js` (mounted at `/api/nocodb`) | — |
| React SPA + Express | `server/lib/nocodb.js` | `server/routes/nocodb.js` | `src/lib/nocodb-client.js` |
| Standalone | `lib/nocodb.js` | `server.js` (Express mini-server) | — |

If the project uses a `src/` directory, nest files under `src/` accordingly.
