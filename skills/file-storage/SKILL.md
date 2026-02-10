---
name: file-storage
description: File upload and storage patterns using Netlify Blobs. Use when implementing file uploads for documents, PDFs, or other non-image files, storing files without CDN transformation needs, or retrieving raw files from blob storage. For image-specific handling with CDN optimization, see the netlify-images skill instead.
---

# File Storage

For images with resizing/optimization, see `netlify-images` skill instead.

---

## Setup

### Install Dependencies

```bash
npm install @netlify/blobs
```

### Link Project (for local development)

```bash
netlify link
```

---

## Basic Storage Operations

### Store a File

```typescript
import { getStore } from '@netlify/blobs';
import { randomUUID } from 'crypto';

async function storeFile(file: File): Promise<string> {
  const extension = file.name.split('.').pop() || 'bin';
  const key = `${randomUUID()}.${extension}`;

  const store = getStore({ name: 'files', consistency: 'strong' });

  await store.set(key, file, {
    metadata: {
      contentType: file.type,
      originalFilename: file.name,
      uploadedAt: new Date().toISOString(),
    },
  });

  return key;
}
```

### Retrieve a File

```typescript
import { getStore } from '@netlify/blobs';

async function getFile(key: string): Promise<ReadableStream | null> {
  const store = getStore({ name: 'files', consistency: 'strong' });
  return await store.get(key, { type: 'stream' });
}

async function getFileMetadata(key: string) {
  const store = getStore({ name: 'files', consistency: 'strong' });
  return await store.getMetadata(key);
}
```

### Delete a File

```typescript
import { getStore } from '@netlify/blobs';

async function deleteFile(key: string): Promise<void> {
  const store = getStore({ name: 'files', consistency: 'strong' });
  await store.delete(key);
}
```

### List Files

```typescript
import { getStore } from '@netlify/blobs';

async function listFiles(prefix?: string) {
  const store = getStore({ name: 'files', consistency: 'strong' });
  const { blobs } = await store.list({ prefix });
  return blobs;
}
```

---

## Vite + React: Upload Handler (Netlify Function)

```typescript
// netlify/functions/upload.ts
import type { Context, Config } from '@netlify/functions';
import { getStore } from '@netlify/blobs';
import { randomUUID } from 'crypto';

const MAX_SIZE = 10 * 1024 * 1024; // 10 MB

export default async (request: Request, context: Context) => {
  if (request.method !== 'POST') {
    return new Response('Method not allowed', { status: 405 });
  }

  try {
    const formData = await request.formData();
    const file = formData.get('file') as File;

    if (!file || !file.size) {
      return Response.json({ error: 'No file provided' }, { status: 400 });
    }

    if (file.size > MAX_SIZE) {
      return Response.json({ error: 'File too large. Maximum size is 10 MB.' }, { status: 400 });
    }

    const extension = file.name.split('.').pop() || 'bin';
    const key = `${randomUUID()}.${extension}`;

    const store = getStore({ name: 'files', consistency: 'strong' });
    await store.set(key, file, {
      metadata: {
        contentType: file.type,
        originalFilename: file.name,
        uploadedAt: new Date().toISOString(),
      },
    });

    return Response.json({ success: true, key });
  } catch (error) {
    console.error('Upload error:', error);
    return Response.json({ error: 'Upload failed' }, { status: 500 });
  }
};

export const config: Config = {
  path: '/api/upload',
};
```

---

## Vite + React: Download Handler (Netlify Function)

```typescript
// netlify/functions/download.ts
import type { Context, Config } from '@netlify/functions';
import { getStore } from '@netlify/blobs';

export default async (request: Request, context: Context) => {
  if (request.method !== 'GET') {
    return new Response('Method not allowed', { status: 405 });
  }

  const key = context.params.key;
  if (!key) {
    return new Response('Not found', { status: 404 });
  }

  try {
    const store = getStore({ name: 'files', consistency: 'strong' });
    const blob = await store.get(key, { type: 'stream' });

    if (!blob) {
      return new Response('File not found', { status: 404 });
    }

    const metadata = await store.getMetadata(key);
    const contentType = metadata?.metadata?.contentType || 'application/octet-stream';
    const originalFilename = metadata?.metadata?.originalFilename || key;

    return new Response(blob, {
      headers: {
        'Content-Type': contentType,
        'Content-Disposition': `attachment; filename="${originalFilename}"`,
        'Cache-Control': 'public, max-age=31536000, immutable',
      },
    });
  } catch (error) {
    console.error('Download error:', error);
    return new Response('Error retrieving file', { status: 500 });
  }
};

export const config: Config = {
  path: '/api/files/:key',
};
```

---

## Astro: Upload Handler (API Route)

```typescript
// src/pages/api/upload.ts
import type { APIRoute } from 'astro';
import { getStore } from '@netlify/blobs';
import { randomUUID } from 'crypto';

const MAX_SIZE = 10 * 1024 * 1024; // 10 MB

export const POST: APIRoute = async ({ request, redirect }) => {
  try {
    const formData = await request.formData();
    const file = formData.get('file') as File;

    if (!file || !file.size) {
      return redirect('/?message=no_file', 302);
    }

    if (file.size > MAX_SIZE) {
      return redirect('/?message=file_too_large', 302);
    }

    const extension = file.name.split('.').pop() || 'bin';
    const key = `${randomUUID()}.${extension}`;

    const store = getStore({ name: 'files', consistency: 'strong' });
    await store.set(key, file, {
      metadata: {
        contentType: file.type,
        originalFilename: file.name,
        uploadedAt: new Date().toISOString(),
      },
    });

    return redirect(`/?message=upload_success&key=${key}`, 302);
  } catch (error) {
    console.error('Upload error:', error);
    return redirect('/?message=upload_failed', 302);
  }
};
```

---

## Astro: Download Handler (API Route)

```typescript
// src/pages/api/files/[key].ts
import type { APIRoute } from 'astro';
import { getStore } from '@netlify/blobs';

export const GET: APIRoute = async ({ params }) => {
  const key = params.key;
  if (!key) {
    return new Response('Not found', { status: 404 });
  }

  try {
    const store = getStore({ name: 'files', consistency: 'strong' });
    const blob = await store.get(key, { type: 'stream' });

    if (!blob) {
      return new Response('File not found', { status: 404 });
    }

    const metadata = await store.getMetadata(key);
    const contentType = metadata?.metadata?.contentType || 'application/octet-stream';
    const originalFilename = metadata?.metadata?.originalFilename || key;

    return new Response(blob, {
      headers: {
        'Content-Type': contentType,
        'Content-Disposition': `attachment; filename="${originalFilename}"`,
        'Cache-Control': 'public, max-age=31536000, immutable',
      },
    });
  } catch (error) {
    console.error('Download error:', error);
    return new Response('Error retrieving file', { status: 500 });
  }
};
```

---

## Store Configuration

### Store Names

Use descriptive store names for different types of content:

```typescript
getStore({ name: 'files' }); // General files
getStore({ name: 'documents' }); // PDFs, docs
getStore({ name: 'exports' }); // Generated exports
getStore({ name: 'images' }); // Images (see netlify-images skill)
```

### Consistency Options

```typescript
// Strong consistency: Immediate reads after writes
getStore({ name: 'files', consistency: 'strong' });

// Eventual consistency: Better performance, eventual sync
getStore({ name: 'files', consistency: 'eventual' });
```

Use `strong` when you need to read immediately after writing.

---

## Key Patterns

Use UUIDs to avoid collisions:

```typescript
const key = `${randomUUID()}.${extension}`;
```

Or content-based keys for deduplication:

```typescript
import { createHash } from 'crypto';

async function getContentHash(file: File): Promise<string> {
  const buffer = await file.arrayBuffer();
  return createHash('sha256').update(Buffer.from(buffer)).digest('hex');
}

const key = `${await getContentHash(file)}.${extension}`;
```

---

## Anti-Patterns

- **Storing large files without size limits** - Netlify Functions have size limits
- **Using predictable keys** - Use UUIDs or hashes to prevent guessing
- **Not storing metadata** - Always store content type and original filename
- **Mixing file types in one store** - Use separate stores for organization

---

## Related Skills

- `netlify-images` - Image storage with CDN optimization
- `netlify-functions` - API endpoint patterns
- `forms` - File upload forms
