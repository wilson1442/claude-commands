---
name: espocrm
description: >
  Integrate a web project with a self-hosted EspoCRM instance. Use this skill whenever the
  user wants to: capture leads from web forms into EspoCRM, push contact/lead data to EspoCRM,
  connect a site to EspoCRM, build CRM integration for form submissions, or any variation of
  "hook up EspoCRM", "send leads to EspoCRM", "CRM form capture", or "integrate with EspoCRM".
  Trigger aggressively: if the user is building a web app and mentions EspoCRM or lead capture
  to a CRM, this skill should fire.
---

# EspoCRM Integration Skill

## Purpose

This skill wires up a web project to a self-hosted EspoCRM instance for capturing leads,
contacts, and other entity data from web forms. It generates server-side API routes (to
keep credentials hidden) and optional client-side helpers.

---

## Step 1: Gather Configuration

Before generating any code, prompt the user for the following. Do not proceed until all
required values are provided:

| Variable | What to ask | Required | Notes |
|---|---|---|---|
| `ESPOCRM_URL` | "What's your EspoCRM instance URL?" | Yes | e.g. `https://crm.example.com` — no trailing slash |
| `ESPOCRM_API_KEY` | "What's your EspoCRM API key?" | Yes | Created under Administration > API Users. Use API Key auth method. |
| `ESPOCRM_ENTITIES` | "Which CRM entities do you want to push data to?" | Yes | e.g. Lead, Contact, Account, or custom entities. Default: Lead |

Also ask these context questions:

- **"What forms or data are you capturing?"** — e.g. contact form, newsletter signup, quote request, early access signup. This helps tailor field mappings and naming.
- **"Do you want me to auto-detect your EspoCRM entity fields, or do you already know which fields you want to map?"** — If auto-detect, the skill will query the EspoCRM API to discover available fields.
- **"Should form submissions also trigger any EspoCRM workflow or notification?"** — Just for awareness; the skill won't configure workflows but can note it.

### Credentials file

Store credentials in `/root/.espocrm-credentials.json`:
```json
{
  "url": "https://crm.example.com",
  "apiKey": "your-api-key-here"
}
```

If this file already exists, read it and offer to use the stored values. Always ask the user
to confirm before using stored credentials.

---

## Step 2: Validate Connection & Discover Fields

### Test the connection

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -H "X-Api-Key: $ESPOCRM_API_KEY" \
  "$ESPOCRM_URL/api/v1/Lead?maxSize=1"
```

If the response is not `200`, troubleshoot:
- `401` — Invalid API key or API user not configured
- `403` — API user role lacks permission for the entity
- `404` — Wrong URL or entity type doesn't exist

### Discover entity fields (if user wants auto-detect)

For each entity the user wants to integrate, fetch its metadata:

```bash
curl -s -H "X-Api-Key: $ESPOCRM_API_KEY" \
  "$ESPOCRM_URL/api/v1/Metadata/get" \
  | jq '.entityDefs.Lead.fields'
```

Or use the simpler approach — fetch one record and inspect keys:

```bash
curl -s -H "X-Api-Key: $ESPOCRM_API_KEY" \
  "$ESPOCRM_URL/api/v1/Lead?maxSize=1&select=*"
```

**Present the discovered fields to the user** in a clean table format, highlighting the
most commonly mapped ones:

| Field | Type | Common Use |
|---|---|---|
| `firstName` | varchar | First name |
| `lastName` | varchar | Last name |
| `emailAddress` | email | Email |
| `phoneNumber` | phone | Phone |
| `description` | text | Message / notes |
| `source` | enum | Lead source (e.g. "Web Form") |
| `status` | enum | Lead status |
| `assignedUserId` | link | Assign to a user |

Ask the user to confirm which fields they want to map from their forms.

---

## Step 3: Detect the Project Framework

Inspect the project root to determine the stack:

```
1. next.config.js / next.config.mjs / next.config.ts  →  Next.js (check for app/ vs pages/)
2. nuxt.config.ts / nuxt.config.js                     →  Nuxt
3. package.json with "express" in dependencies          →  Express
4. package.json with "fastify" in dependencies          →  Fastify
5. package.json with "react" but no meta-framework      →  React SPA (needs proxy backend)
6. app/main.py or manage.py with fastapi/flask          →  Python (FastAPI/Flask)
7. No framework detected                                →  Generate standalone module
```

Tell the user what you detected and confirm before generating.

---

## Step 4: Generate the Integration

### What gets generated (all frameworks)

**A) Environment config**
- Add `ESPOCRM_URL` and `ESPOCRM_API_KEY` to `.env` (create if missing)
- Add the same keys (without values) to `.env.example`
- Add `.env` to `.gitignore` if not already present

**B) EspoCRM API client** — a server-side-only module

The client must support:

- **Create entity:** `POST /api/v1/{EntityType}` with JSON body
- **List entities:** `GET /api/v1/{EntityType}` with search params
- **Get entity:** `GET /api/v1/{EntityType}/{id}`
- **Update entity:** `PUT /api/v1/{EntityType}/{id}` with JSON body
- **Delete entity:** `DELETE /api/v1/{EntityType}/{id}`

All requests include the header: `X-Api-Key: {ESPOCRM_API_KEY}`

And for JSON payloads: `Content-Type: application/json`

**C) Server-side API route handlers**

For each configured entity (e.g., Lead):

- `POST /api/crm/leads` → Create a lead in EspoCRM
- `GET  /api/crm/leads` → List leads (with pagination, filtering)
- `GET  /api/crm/leads/:id` → Get a single lead
- `PUT  /api/crm/leads/:id` → Update a lead
- `DELETE /api/crm/leads/:id` → Delete a lead

**D) Form integration helper (client-side)**

A lightweight function that form components can call:

```typescript
// Example usage in a React form
const submitLead = async (formData: LeadFormData) => {
  const response = await fetch('/api/crm/leads', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(formData),
  });
  if (!response.ok) throw new Error('Failed to submit');
  return response.json();
};
```

### Framework-specific patterns

**FastAPI + React SPA (this project's stack):**

For the backend (FastAPI):
- Create `app/api/crm.py` — router with endpoints
- Create `app/services/espocrm.py` — EspoCRM API client class
- Create `app/schemas/crm.py` — Pydantic models for request/response validation
- Register the router in `app/main.py`

For the frontend (React + TypeScript):
- Create `src/lib/crm-client.ts` — typed fetch wrapper
- Create `src/hooks/useCrmSubmit.ts` — React hook for form submissions (using TanStack Query if available)

**Next.js (App Router):**
- `lib/espocrm.ts` — API client
- `app/api/crm/[entity]/route.ts` — dynamic route handler
- `lib/crm-client.ts` — client-side helper

**Express:**
- `lib/espocrm.js` — API client
- `routes/crm.js` — Express router

---

## Step 5: Field Mapping & Validation

Generate a field mapping configuration that translates form field names to EspoCRM field names:

```typescript
// Example field mapping
const LEAD_FIELD_MAP: Record<string, string> = {
  name: 'lastName',        // or split into firstName/lastName
  email: 'emailAddress',
  phone: 'phoneNumber',
  message: 'description',
  company: 'accountName',
};

// Default values applied to every submission
const LEAD_DEFAULTS: Record<string, string> = {
  source: 'Web Form',
  status: 'New',
};
```

The generated code should:
- Map form fields to EspoCRM fields using the configured mapping
- Apply default values (like `source: "Web Form"`)
- Validate required fields before sending to EspoCRM
- Handle email specially — EspoCRM expects `emailAddress` as a string for the primary email
- Handle phone specially — EspoCRM expects `phoneNumber` as a string for the primary phone
- Return the created entity ID on success

---

## Step 6: Wire It Up

After generating files, provide the user with:

1. **What was created** — list every file with a one-line description
2. **Environment setup** — remind them to fill in `.env` values if not already set
3. **EspoCRM setup checklist:**
   - Create an API User under Administration > API Users
   - Choose "API Key" authentication method
   - Create a Role with appropriate permissions (at minimum: create on Lead/Contact)
   - Assign the role to the API user
   - Copy the generated API key
4. **Usage examples** — show how to call the new endpoints:
   - Submit a lead from a form
   - List leads with filtering
   - Example curl commands for testing
5. **Form integration example** — show how to wire up an existing form component

---

## EspoCRM API Reference (for code generation)

### Authentication

Use API Key authentication (simplest for server-to-server):
```
X-Api-Key: <api_key>
```

### Base URL

All API calls go to: `{ESPOCRM_URL}/api/v1/`

### CRUD Endpoints

| Method | Path | Body | Description |
|---|---|---|---|
| `GET` | `/api/v1/{Entity}` | — | List records. Query params: `maxSize`, `offset`, `where`, `orderBy`, `order`, `select` |
| `GET` | `/api/v1/{Entity}/{id}` | — | Get single record |
| `POST` | `/api/v1/{Entity}` | `{ field: value }` | Create a record |
| `PUT` | `/api/v1/{Entity}/{id}` | `{ field: value }` | Update a record |
| `DELETE` | `/api/v1/{Entity}/{id}` | — | Delete a record |

### Search/List Parameters

| Param | Type | Example | Description |
|---|---|---|---|
| `maxSize` | number | `20` | Records per page (default 20, max 200) |
| `offset` | number | `0` | Skip N records |
| `orderBy` | string | `createdAt` | Sort field |
| `order` | string | `desc` | Sort direction: `asc` or `desc` |
| `select` | string | `id,firstName,lastName,emailAddress` | Comma-separated fields to return |
| `where` | array | See below | Filter conditions |

### Where Clause Format

The `where` parameter is a JSON-encoded array of filter objects:

```json
[
  {
    "type": "equals",
    "attribute": "status",
    "value": "New"
  },
  {
    "type": "contains",
    "attribute": "emailAddress",
    "value": "@example.com"
  }
]
```

Filter types: `equals`, `notEquals`, `greaterThan`, `lessThan`, `greaterThanOrEquals`,
`lessThanOrEquals`, `like`, `contains`, `startsWith`, `endsWith`, `isNull`, `isNotNull`,
`in`, `notIn`, `between`.

### Email & Phone Handling

EspoCRM handles emails and phones as linked records. When creating via API, you can use
the shorthand:

```json
{
  "firstName": "John",
  "lastName": "Doe",
  "emailAddress": "john@example.com",
  "phoneNumber": "+1234567890"
}
```

EspoCRM will automatically create the linked email/phone records.

### Response Format

**Single record:**
```json
{
  "id": "abc123def456",
  "firstName": "John",
  "lastName": "Doe",
  "emailAddress": "john@example.com",
  "status": "New",
  "createdAt": "2025-01-15 10:30:00"
}
```

**List response:**
```json
{
  "total": 150,
  "list": [
    { "id": "abc123", "firstName": "John", ... },
    { "id": "def456", "firstName": "Jane", ... }
  ]
}
```

---

## Error Handling

All generated code should handle these cases:

| Status | Meaning | Action |
|---|---|---|
| `400` | Bad request (invalid field values) | Return EspoCRM's error message to help debug |
| `401` | Invalid or missing API key | Throw auth error, hint to check ESPOCRM_API_KEY |
| `403` | Forbidden — role lacks permission | Throw permission error, hint to check API user role |
| `404` | Entity or record not found | Throw not-found error |
| `409` | Conflict (duplicate) | Return conflict details |
| `429` | Rate limited | Retry with exponential backoff (3 attempts) |
| `5xx` | EspoCRM server error | Throw with raw response for debugging |

---

## Important Notes

- **API key must NEVER be exposed to the browser.** All EspoCRM calls go through server-side routes.
- **EspoCRM uses PUT for updates**, not PATCH. The full entity path includes the ID: `PUT /api/v1/Lead/{id}`.
- **IDs are strings** in EspoCRM (not integers). They look like `abc123def456`.
- **TypeScript**: If the project uses TypeScript (detected via `tsconfig.json`), generate `.ts` files with proper types. Otherwise generate `.js`.
- **Duplicate detection**: EspoCRM can detect duplicates on create. The API may return a `409` with duplicate info. Generated code should handle this gracefully and surface it to the caller.
- **The `source` field on Lead** is commonly set to `"Web Form"` or `"Web Site"` for form captures — always set a sensible default.

---

## Argument Handling

If the user passes arguments (e.g., `/espocrm Lead,Contact`), treat them as the entity
types to integrate. If no arguments, prompt for entity types.
