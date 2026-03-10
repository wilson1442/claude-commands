# Standalone Node.js — NocoDB Integration

For projects with no web framework detected. Generates a self-contained Express
mini-server with the NocoDB client baked in.

## Files to Generate

### 1. `lib/nocodb.js` — NocoDB client

Same as the Express version — see `express.md`. Generate it identically.

### 2. `server.js` — Standalone Express server with NocoDB routes

```javascript
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const { nocodb } = require('./lib/nocodb');

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(express.json());

// ── NocoDB CRUD Routes ────────────────────────────────────────

// List records
app.get('/api/nocodb', async (req, res) => {
  try {
    const { limit, offset, where, sort, fields } = req.query;
    const data = await nocodb.list({
      limit: limit ? Number(limit) : 25,
      offset: offset ? Number(offset) : 0,
      where: where || undefined,
      sort: sort || undefined,
      fields: fields || undefined,
    });
    res.json(data);
  } catch (e) {
    res.status(e.status || 500).json({ error: e.message });
  }
});

// Get single record
app.get('/api/nocodb/:id', async (req, res) => {
  try {
    const data = await nocodb.get(req.params.id);
    res.json(data);
  } catch (e) {
    res.status(e.status || 500).json({ error: e.message });
  }
});

// Create record(s)
app.post('/api/nocodb', async (req, res) => {
  try {
    const data = Array.isArray(req.body)
      ? await nocodb.bulkCreate(req.body)
      : await nocodb.create(req.body);
    res.status(201).json(data);
  } catch (e) {
    res.status(e.status || 500).json({ error: e.message });
  }
});

// Update record(s)
app.patch('/api/nocodb', async (req, res) => {
  try {
    const data = Array.isArray(req.body)
      ? await nocodb.bulkUpdate(req.body)
      : await nocodb.update(req.body);
    res.json(data);
  } catch (e) {
    res.status(e.status || 500).json({ error: e.message });
  }
});

// Delete record(s)
app.delete('/api/nocodb', async (req, res) => {
  try {
    if (req.body.ids) {
      const data = await nocodb.bulkDelete(req.body.ids);
      return res.json(data);
    }
    const data = await nocodb.delete(req.body.id || req.body.Id);
    res.json(data);
  } catch (e) {
    res.status(e.status || 500).json({ error: e.message });
  }
});

// ── Start ─────────────────────────────────────────────────────

app.listen(PORT, () => {
  console.log(`NocoDB API server running on http://localhost:${PORT}`);
  console.log(`Routes: GET/POST/PATCH/DELETE /api/nocodb`);
});
```

### 3. `package.json`

```json
{
  "name": "nocodb-api",
  "private": true,
  "scripts": {
    "start": "node server.js",
    "dev": "node --watch server.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.4.0",
    "express": "^4.21.0"
  }
}
```

## Setup Instructions to Show User

```bash
npm install
# Fill in .env with your NocoDB credentials
npm run dev
```

Then test:
```bash
# List records
curl http://localhost:3000/api/nocodb

# Create a record
curl -X POST http://localhost:3000/api/nocodb \
  -H "Content-Type: application/json" \
  -d '{"Name": "Test", "Status": "Active"}'

# Update a record
curl -X PATCH http://localhost:3000/api/nocodb \
  -H "Content-Type: application/json" \
  -d '{"Id": 1, "Status": "Completed"}'

# Delete a record
curl -X DELETE http://localhost:3000/api/nocodb \
  -H "Content-Type: application/json" \
  -d '{"id": 1}'
```
