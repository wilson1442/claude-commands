# Next.js Pages Router — NocoDB Integration

Generate these files for Next.js projects using the Pages Router (`pages/` directory, no `app/`).

## Files to Generate

### 1. `lib/nocodb.ts`

Same as the App Router version — see `nextjs-app.md` for the full `nocodb` client module.
Generate it identically.

### 2. `pages/api/nocodb/index.ts` — List + Create + Update + Delete

```typescript
import type { NextApiRequest, NextApiResponse } from 'next';
import { nocodb } from '@/lib/nocodb';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  try {
    switch (req.method) {
      case 'GET': {
        const { limit, offset, where, sort, fields } = req.query;
        const data = await nocodb.list({
          limit: limit ? Number(limit) : 25,
          offset: offset ? Number(offset) : 0,
          where: where as string || undefined,
          sort: sort as string || undefined,
          fields: fields as string || undefined,
        });
        return res.json(data);
      }
      case 'POST': {
        const data = Array.isArray(req.body)
          ? await nocodb.bulkCreate(req.body)
          : await nocodb.create(req.body);
        return res.status(201).json(data);
      }
      case 'PATCH': {
        const data = Array.isArray(req.body)
          ? await nocodb.bulkUpdate(req.body)
          : await nocodb.update(req.body);
        return res.json(data);
      }
      case 'DELETE': {
        if (req.body.ids) {
          const data = await nocodb.bulkDelete(req.body.ids);
          return res.json(data);
        }
        const data = await nocodb.delete(req.body.id || req.body.Id);
        return res.json(data);
      }
      default:
        res.setHeader('Allow', 'GET, POST, PATCH, DELETE');
        return res.status(405).json({ error: `Method ${req.method} not allowed` });
    }
  } catch (e: any) {
    return res.status(e.status || 500).json({ error: e.message });
  }
}
```

### 3. `pages/api/nocodb/[id].ts` — Get single record

```typescript
import type { NextApiRequest, NextApiResponse } from 'next';
import { nocodb } from '@/lib/nocodb';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'GET') {
    res.setHeader('Allow', 'GET');
    return res.status(405).json({ error: `Method ${req.method} not allowed` });
  }
  try {
    const { id } = req.query;
    const data = await nocodb.get(id as string);
    return res.json(data);
  } catch (e: any) {
    return res.status(e.status || 500).json({ error: e.message });
  }
}
```

### 4. `lib/nocodb-client.ts`

Same browser-safe client as the App Router version — see `nextjs-app.md`.
Generate it identically.

## Adaptation Notes

- If using `src/` layout, place under `src/pages/api/` and `src/lib/`
- The `@/lib/nocodb` import alias depends on `tsconfig.json` paths — check for `baseUrl` or `paths` config
- For JS projects, drop type annotations and rename to `.js`
