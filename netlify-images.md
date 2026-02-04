# Netlify Image Handling Skill

This skill covers the complete flow for handling user-uploaded images in Netlify deployments, including uploading, storing in Netlify Blobs, serving via API endpoints, and optimizing with Netlify Image CDN.

## Supported Frameworks

- **Vite + React**: Uses Netlify Functions for server-side handling
- **Astro**: Uses Astro API routes (works with SSR, SSG, or hybrid)

---

## 1. Install Dependencies

```bash
npm install @netlify/blobs
```

For local development, link your site:

```bash
netlify link
```

---

## 2. Client-Side Upload

### Vite + React

```tsx
// src/components/ImageUpload.tsx
import { useState } from 'react'

export function ImageUpload({ onSuccess }: { onSuccess?: () => void }) {
  const [uploading, setUploading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const handleSubmit = async (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault()
    setError(null)
    setUploading(true)

    const form = e.currentTarget
    const formData = new FormData(form)

    try {
      const response = await fetch('/api/upload', {
        method: 'POST',
        body: formData,
      })

      if (!response.ok) {
        const data = await response.json()
        throw new Error(data.error || 'Upload failed')
      }

      form.reset()
      onSuccess?.()
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Upload failed')
    } finally {
      setUploading(false)
    }
  }

  return (
    <form onSubmit={handleSubmit} encType="multipart/form-data">
      <input
        type="file"
        name="image"
        accept="image/jpeg,image/png,image/gif,image/webp"
        required
      />
      <button type="submit" disabled={uploading}>
        {uploading ? 'Uploading...' : 'Upload'}
      </button>
      {error && <p className="error">{error}</p>}
    </form>
  )
}
```

### Astro (with optional client-side JS)

```astro
---
// src/components/ImageUpload.astro
---
<form action="/api/upload" method="post" enctype="multipart/form-data" id="upload-form">
  <input
    type="file"
    name="image"
    accept="image/jpeg,image/png,image/gif,image/webp"
    required
  />
  <button type="submit">Upload</button>
</form>

<script>
  // Optional: Add client-side validation
  const form = document.getElementById('upload-form') as HTMLFormElement
  const input = form.querySelector('input[type="file"]') as HTMLInputElement
  const MAX_SIZE = 4 * 1024 * 1024 // 4 MB

  input.addEventListener('change', (e) => {
    const file = (e.target as HTMLInputElement).files?.[0]
    if (file && file.size > MAX_SIZE) {
      alert('File too large. Maximum size is 4 MB.')
      input.value = ''
    }
  })
</script>
```

---

## 3. Server-Side Upload Handler

### Vite + React (Netlify Function)

```typescript
// netlify/functions/upload.mts
import type { Context, Config } from '@netlify/functions'
import { getStore } from '@netlify/blobs'
import { randomUUID } from 'crypto'

const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/gif', 'image/webp']
const MAX_SIZE = 4 * 1024 * 1024 // 4 MB

export default async (request: Request, context: Context) => {
  if (request.method !== 'POST') {
    return new Response('Method not allowed', { status: 405 })
  }

  try {
    const formData = await request.formData()
    const image = formData.get('image') as File

    // Validate file exists
    if (!image || !image.size) {
      return Response.json({ error: 'No image provided' }, { status: 400 })
    }

    // Validate file size
    if (image.size > MAX_SIZE) {
      return Response.json({ error: 'File too large. Maximum size is 4 MB.' }, { status: 400 })
    }

    // Validate file type
    if (!ALLOWED_TYPES.includes(image.type)) {
      return Response.json({ error: 'Invalid file type. Allowed: JPG, PNG, GIF, WebP' }, { status: 400 })
    }

    // Generate unique key
    const extension = image.name.split('.').pop() || 'jpg'
    const key = `${randomUUID()}.${extension}`

    // Store in Netlify Blobs
    const store = getStore({ name: 'images', consistency: 'strong' })
    await store.set(key, image, {
      metadata: {
        contentType: image.type,
        originalFilename: image.name,
        uploadedAt: new Date().toISOString(),
      },
    })

    return Response.json({
      success: true,
      key,
      url: `/img/${key}`, // CDN-optimized URL
    })
  } catch (error) {
    console.error('Upload error:', error)
    return Response.json({ error: 'Upload failed' }, { status: 500 })
  }
}

export const config: Config = {
  path: '/api/upload',
}
```

### Astro (API Route)

```typescript
// src/pages/api/upload.ts
import type { APIRoute } from 'astro'
import { getStore } from '@netlify/blobs'
import { randomUUID } from 'crypto'

const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/gif', 'image/webp']
const MAX_SIZE = 4 * 1024 * 1024 // 4 MB

export const POST: APIRoute = async ({ request, redirect }) => {
  try {
    const formData = await request.formData()
    const image = formData.get('image') as File

    // Validate file exists
    if (!image || !image.size) {
      return new Response('No image provided', { status: 400 })
    }

    // Validate file size
    if (image.size > MAX_SIZE) {
      return new Response('File too large. Maximum size is 4 MB.', { status: 400 })
    }

    // Validate file type
    if (!ALLOWED_TYPES.includes(image.type)) {
      return new Response('Invalid file type', { status: 400 })
    }

    // Generate unique key
    const extension = image.name.split('.').pop() || 'jpg'
    const key = `${randomUUID()}.${extension}`

    // Store in Netlify Blobs
    const store = getStore({ name: 'images', consistency: 'strong' })
    await store.set(key, image, {
      metadata: {
        contentType: image.type,
        originalFilename: image.name,
        uploadedAt: new Date().toISOString(),
      },
    })

    // Option 1: Redirect back (for form submissions)
    return redirect('/?upload=success', 303)

    // Option 2: Return JSON (for fetch-based uploads)
    // return Response.json({ success: true, key, url: `/img/${key}` })
  } catch (error) {
    console.error('Upload error:', error)
    return new Response('Upload failed', { status: 500 })
  }
}
```

---

## 4. Serving Images from Blobs

### Vite + React (Netlify Function)

```typescript
// netlify/functions/serve-image.mts
import type { Context, Config } from '@netlify/functions'
import { getStore } from '@netlify/blobs'

export default async (request: Request, context: Context) => {
  if (request.method !== 'GET') {
    return new Response('Method not allowed', { status: 405 })
  }

  const key = context.params.key
  if (!key) {
    return new Response('Not found', { status: 404 })
  }

  try {
    const store = getStore({ name: 'images', consistency: 'strong' })
    const blob = await store.get(key, { type: 'stream' })

    if (!blob) {
      return new Response('Image not found', { status: 404 })
    }

    // Get metadata for content type
    const metadata = await store.getMetadata(key)
    const contentType = metadata?.metadata?.contentType || 'image/jpeg'

    return new Response(blob, {
      headers: {
        'Content-Type': contentType,
        'Cache-Control': 'public, max-age=31536000, immutable',
      },
    })
  } catch (error) {
    console.error('Serve error:', error)
    return new Response('Error retrieving image', { status: 500 })
  }
}

export const config: Config = {
  path: '/uploads/:key',
}
```

### Astro (Dynamic Route)

```typescript
// src/pages/uploads/[key].ts
import type { APIRoute } from 'astro'
import { getStore } from '@netlify/blobs'

export const GET: APIRoute = async ({ params }) => {
  const key = params.key
  if (!key) {
    return new Response('Not found', { status: 404 })
  }

  try {
    const store = getStore({ name: 'images', consistency: 'strong' })
    const blob = await store.get(key, { type: 'stream' })

    if (!blob) {
      return new Response('Image not found', { status: 404 })
    }

    // Get metadata for content type
    const metadata = await store.getMetadata(key)
    const contentType = metadata?.metadata?.contentType || 'image/jpeg'

    return new Response(blob, {
      headers: {
        'Content-Type': contentType,
        'Cache-Control': 'public, max-age=31536000, immutable',
      },
    })
  } catch (error) {
    console.error('Serve error:', error)
    return new Response('Error retrieving image', { status: 500 })
  }
}
```

---

## 5. Netlify Image CDN Configuration

Add these redirects to `netlify.toml` to enable on-the-fly image optimization. The CDN transforms images at the edge and caches the results.

```toml
# netlify.toml

# Enable remote images if needed (for external image sources)
[images]
remote_images = ["https://api.dicebear.com/*", "https://example.com/*"]

# Basic optimized image route (auto-format optimization)
[[redirects]]
from = "/img/:key"
to = "/.netlify/images?url=/uploads/:key"
status = 200

# With width parameter
[[redirects]]
from = "/img/:key/:width"
to = "/.netlify/images?url=/uploads/:key&w=:width"
status = 200

# With width and height
[[redirects]]
from = "/img/:key/:width/:height"
to = "/.netlify/images?url=/uploads/:key&w=:width&h=:height"
status = 200

# With fit mode (cover, contain, fill)
[[redirects]]
from = "/img/:key/:width/:height/:fit"
to = "/.netlify/images?url=/uploads/:key&w=:width&h=:height&fit=:fit"
status = 200

# With quality (1-100)
[[redirects]]
from = "/img/:key/:width/:height/:fit/:quality"
to = "/.netlify/images?url=/uploads/:key&w=:width&h=:height&fit=:fit&q=:quality"
status = 200

# With format (avif, webp, jpg, png, gif)
[[redirects]]
from = "/img/:key/:width/:height/:fit/:quality/:format"
to = "/.netlify/images?url=/uploads/:key&w=:width&h=:height&fit=:fit&q=:quality&fm=:format"
status = 200
```

### CDN URL Parameters Reference

| Parameter | Description | Values |
|-----------|-------------|--------|
| `url` | Source image path (required) | Relative or absolute URL |
| `w` | Width in pixels | Any positive integer |
| `h` | Height in pixels | Any positive integer |
| `fit` | How to fit image to dimensions | `contain` (default), `cover`, `fill` |
| `position` | Crop position (with `fit=cover`) | `center`, `top`, `bottom`, `left`, `right` |
| `q` | Quality | 1-100 |
| `fm` | Output format | `avif`, `webp`, `jpg`, `png`, `gif` |

### Common Size Presets

For consistent sizing across your app, define preset routes:

```toml
# Thumbnail (150x150 square)
[[redirects]]
from = "/img/thumb/:key"
to = "/.netlify/images?url=/uploads/:key&w=150&h=150&fit=cover"
status = 200

# Small (300px wide)
[[redirects]]
from = "/img/small/:key"
to = "/.netlify/images?url=/uploads/:key&w=300"
status = 200

# Medium (600px wide)
[[redirects]]
from = "/img/medium/:key"
to = "/.netlify/images?url=/uploads/:key&w=600"
status = 200

# Large (1200px wide)
[[redirects]]
from = "/img/large/:key"
to = "/.netlify/images?url=/uploads/:key&w=1200"
status = 200

# Hero (1200x675, 16:9 aspect ratio)
[[redirects]]
from = "/img/hero/:key"
to = "/.netlify/images?url=/uploads/:key&w=1200&h=675&fit=cover"
status = 200

# Avatar (256x256 square)
[[redirects]]
from = "/img/avatar/:key"
to = "/.netlify/images?url=/uploads/:key&w=256&h=256&fit=cover"
status = 200
```

---

## 6. Displaying Images

### Using CDN URLs

```tsx
// Vite + React
<img src={`/img/${imageKey}`} alt="Description" />
<img src={`/img/thumb/${imageKey}`} alt="Thumbnail" />
<img src={`/img/${imageKey}/800`} alt="800px wide" />
<img src={`/img/${imageKey}/400/300/cover`} alt="400x300 cropped" />
```

```astro
---
// Astro
const imageKey = 'abc123.jpg'
---
<img src={`/img/${imageKey}`} alt="Description" />
<img src={`/img/thumb/${imageKey}`} alt="Thumbnail" />
```

### Responsive Images

```tsx
<img
  src={`/img/medium/${imageKey}`}
  srcSet={`
    /img/small/${imageKey} 300w,
    /img/medium/${imageKey} 600w,
    /img/large/${imageKey} 1200w
  `}
  sizes="(max-width: 600px) 300px, (max-width: 1200px) 600px, 1200px"
  alt="Responsive image"
/>
```

---

## 7. Complete Example: netlify.toml

```toml
[build]
command = "npm run build"
publish = "dist"

[functions]
directory = "netlify/functions"

# Image CDN: Allow remote images if needed
[images]
remote_images = ["https://api.dicebear.com/*"]

# Serve raw uploads from blob storage
# (This is handled by the Netlify Function, not a redirect)

# CDN-optimized image routes
[[redirects]]
from = "/img/:key"
to = "/.netlify/images?url=/uploads/:key"
status = 200

[[redirects]]
from = "/img/thumb/:key"
to = "/.netlify/images?url=/uploads/:key&w=150&h=150&fit=cover"
status = 200

[[redirects]]
from = "/img/small/:key"
to = "/.netlify/images?url=/uploads/:key&w=300"
status = 200

[[redirects]]
from = "/img/medium/:key"
to = "/.netlify/images?url=/uploads/:key&w=600"
status = 200

[[redirects]]
from = "/img/large/:key"
to = "/.netlify/images?url=/uploads/:key&w=1200"
status = 200

[[redirects]]
from = "/img/hero/:key"
to = "/.netlify/images?url=/uploads/:key&w=1200&h=675&fit=cover"
status = 200

[[redirects]]
from = "/img/avatar/:key"
to = "/.netlify/images?url=/uploads/:key&w=256&h=256&fit=cover"
status = 200

[[redirects]]
from = "/img/:key/:width"
to = "/.netlify/images?url=/uploads/:key&w=:width"
status = 200

[[redirects]]
from = "/img/:key/:width/:height"
to = "/.netlify/images?url=/uploads/:key&w=:width&h=:height"
status = 200

[[redirects]]
from = "/img/:key/:width/:height/:fit"
to = "/.netlify/images?url=/uploads/:key&w=:width&h=:height&fit=:fit"
status = 200

# SPA fallback (for Vite + React)
[[redirects]]
from = "/*"
to = "/index.html"
status = 200
```

---

## 8. Key Considerations

### Blob Storage

- **Store name**: Use descriptive names like `images`, `avatars`, `uploads`
- **Key format**: Use UUIDs or content hashes to avoid collisions
- **Consistency**: Use `strong` for immediate reads after writes; `eventual` for better performance when immediate consistency isn't required
- **Metadata**: Store content type, original filename, and upload timestamp

### Image CDN

- **Automatic format optimization**: If no `fm` parameter is specified, Netlify automatically serves AVIF or WebP based on browser support
- **Caching**: Transformed images are cached at the edge; use immutable cache headers on source images
- **Remote images**: Must be explicitly allowed in `netlify.toml` under `[images].remote_images`

### File Validation

- **Always validate on server**: Client-side validation can be bypassed
- **Check both size and type**: Limit to reasonable sizes (4 MB is common) and allowed MIME types
- **Validate MIME type, not just extension**: Use `file.type` to check actual content type

### Error Handling

- **Return appropriate status codes**: 400 for validation errors, 404 for missing images, 500 for server errors
- **Log errors server-side**: Use `console.error` for debugging
- **Provide user-friendly messages**: Don't expose internal error details to clients

---

## References

- [Netlify Blobs Documentation](https://docs.netlify.com/build/data-and-storage/netlify-blobs/)
- [Netlify Image CDN Documentation](https://docs.netlify.com/build/image-cdn/overview/)
- [@netlify/blobs on npm](https://www.npmjs.com/package/@netlify/blobs)
- [Serving User-Generated Uploads Guide](https://developers.netlify.com/guides/user-generated-uploads-with-netlify-blobs/)
