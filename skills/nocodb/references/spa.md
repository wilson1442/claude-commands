# React / Vue SPA — NocoDB Integration

For SPAs without a built-in server (Create React App, Vite + React, Vite + Vue, etc.),
the NocoDB API key cannot be safely stored in the browser. This template generates:

1. A lightweight Express proxy server to handle NocoDB calls
2. A browser-safe client that talks to the proxy

## Files to Generate

### 1. `server/lib/nocodb.js` — Server-side NocoDB client

Same as the Express version — see `express.md`. Generate it identically under `server/lib/`.

### 2. `server/index.js` — Minimal Express proxy

```javascript
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const nocodbRoutes = require('./routes/nocodb');

const app = express();
const PORT = process.env.PROXY_PORT || 3001;

app.use(cors({ origin: process.env.FRONTEND_URL || 'http://localhost:5173' }));
app.use(express.json());
app.use('/api/nocodb', nocodbRoutes);

app.listen(PORT, () => {
  console.log(`NocoDB proxy running on port ${PORT}`);
});
```

### 3. `server/routes/nocodb.js` — Express router

Same as the Express version — see `express.md`. Generate it identically.

### 4. `src/lib/nocodb-client.js` (or `.ts`) — Browser-safe client

```javascript
/**
 * NocoDB Client — Browser-safe
 * Talks to the Express proxy at /api/nocodb, never directly to NocoDB
 */

const API_BASE = import.meta.env.VITE_API_URL || 'http://localhost:3001';

async function api(path, options = {}) {
  const res = await fetch(`${API_BASE}${path}`, {
    headers: { 'Content-Type': 'application/json', ...options.headers },
    ...options,
  });
  if (!res.ok) {
    const err = await res.json().catch(() => ({ error: res.statusText }));
    throw new Error(err.error || `Request failed: ${res.status}`);
  }
  return res.json();
}

export const nocodbClient = {
  list(params = {}) {
    const query = new URLSearchParams();
    Object.entries(params).forEach(([k, v]) => { if (v !== undefined) query.set(k, String(v)); });
    const qs = query.toString();
    return api(`/api/nocodb${qs ? '?' + qs : ''}`);
  },
  get(id) { return api(`/api/nocodb/${id}`); },
  create(data) { return api('/api/nocodb', { method: 'POST', body: JSON.stringify(data) }); },
  update(data) { return api('/api/nocodb', { method: 'PATCH', body: JSON.stringify(data) }); },
  remove(id) { return api('/api/nocodb', { method: 'DELETE', body: JSON.stringify({ id }) }); },
  bulkRemove(ids) { return api('/api/nocodb', { method: 'DELETE', body: JSON.stringify({ ids }) }); },
};
```

### 5. `server/package.json` — Server dependencies

```json
{
  "name": "nocodb-proxy",
  "private": true,
  "scripts": {
    "start": "node index.js",
    "dev": "node --watch index.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.4.0",
    "express": "^4.21.0"
  }
}
```

## Adaptation Notes

- Place `.env` at project root with `NOCODB_URL`, `NOCODB_TABLE_ID`, `NOCODB_API_KEY`, and `FRONTEND_URL`
- For Vite projects, the client uses `import.meta.env.VITE_API_URL` — add this to the frontend `.env`
- For CRA projects, replace with `process.env.REACT_APP_API_URL`
- If the project already has a backend (e.g. in a monorepo), integrate the routes there instead of spinning up a separate server
- Add `cd server && npm install` to setup instructions
- For dev: run the proxy alongside the frontend (e.g. `concurrently "npm run dev" "cd server && npm run dev"`)
