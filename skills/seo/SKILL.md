---
name: seo
description: SEO best practices for both Astro and Vite + React projects. Use when implementing meta tags, Open Graph data, structured data, sitemaps, or optimizing for search engines. Covers SSR advantages, client-side considerations, and performance as an SEO factor.
---

# SEO

## Framework Considerations

### Astro (SSR/SSG)

**Natural SEO advantages:**

- Server-rendered HTML is immediately crawlable
- No JavaScript required for content
- Fast initial page loads
- Easy meta tag management in frontmatter

### Vite + React (SPA)

**Challenges:**

- Content is client-rendered
- Search engines may not wait for JavaScript
- Requires additional consideration

**Solutions:**

- Netlify pre-rendering for static content
- Proper meta tag management
- sitemap.xml generation

---

## Meta Tags

### Astro Layout

```astro
---
// src/layouts/Layout.astro
interface Props {
  title: string;
  description?: string;
  image?: string;
  noindex?: boolean;
}

const {
  title,
  description = "Default site description",
  image = "/og-image.png",
  noindex = false,
} = Astro.props;

const canonicalUrl = new URL(Astro.url.pathname, Astro.site);
const imageUrl = new URL(image, Astro.site);
---
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />

    <!-- Primary Meta Tags -->
    <title>{title}</title>
    <meta name="title" content={title} />
    <meta name="description" content={description} />
    <link rel="canonical" href={canonicalUrl} />

    {noindex && <meta name="robots" content="noindex, nofollow" />}

    <!-- Open Graph / Facebook -->
    <meta property="og:type" content="website" />
    <meta property="og:url" content={canonicalUrl} />
    <meta property="og:title" content={title} />
    <meta property="og:description" content={description} />
    <meta property="og:image" content={imageUrl} />

    <!-- Twitter -->
    <meta property="twitter:card" content="summary_large_image" />
    <meta property="twitter:url" content={canonicalUrl} />
    <meta property="twitter:title" content={title} />
    <meta property="twitter:description" content={description} />
    <meta property="twitter:image" content={imageUrl} />

    <!-- Favicon -->
    <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
  </head>
  <body>
    <slot />
  </body>
</html>
```

### Vite + React

Use react-helmet-async or similar:

```bash
npm install react-helmet-async
```

```tsx
// src/components/SEO.tsx
import { Helmet } from 'react-helmet-async';

interface SEOProps {
  title: string;
  description?: string;
  image?: string;
  noindex?: boolean;
}

export function SEO({
  title,
  description = 'Default site description',
  image = '/og-image.png',
  noindex = false,
}: SEOProps) {
  const siteUrl = window.location.origin;
  const canonicalUrl = window.location.href;
  const imageUrl = `${siteUrl}${image}`;

  return (
    <Helmet>
      <title>{title}</title>
      <meta name="description" content={description} />
      <link rel="canonical" href={canonicalUrl} />

      {noindex && <meta name="robots" content="noindex, nofollow" />}

      <meta property="og:type" content="website" />
      <meta property="og:url" content={canonicalUrl} />
      <meta property="og:title" content={title} />
      <meta property="og:description" content={description} />
      <meta property="og:image" content={imageUrl} />

      <meta property="twitter:card" content="summary_large_image" />
      <meta property="twitter:title" content={title} />
      <meta property="twitter:description" content={description} />
      <meta property="twitter:image" content={imageUrl} />
    </Helmet>
  );
}
```

```tsx
// src/App.tsx
import { HelmetProvider } from 'react-helmet-async';

function App() {
  return (
    <HelmetProvider>
      <Routes>...</Routes>
    </HelmetProvider>
  );
}
```

```tsx
// src/pages/HomePage.tsx
import { SEO } from '../components/SEO';

export function HomePage() {
  return (
    <>
      <SEO title="Home | My App" description="Welcome to my application" />
      <main>...</main>
    </>
  );
}
```

---

## Structured Data (JSON-LD)

### Article Schema

```astro
---
const article = {
  "@context": "https://schema.org",
  "@type": "Article",
  headline: post.title,
  description: post.excerpt,
  image: post.image,
  datePublished: post.publishedAt,
  dateModified: post.updatedAt,
  author: {
    "@type": "Person",
    name: post.author.name,
  },
};
---
<script type="application/ld+json" set:html={JSON.stringify(article)} />
```

### Organization Schema

```astro
---
const org = {
  "@context": "https://schema.org",
  "@type": "Organization",
  name: "Company Name",
  url: "https://example.com",
  logo: "https://example.com/logo.png",
  sameAs: [
    "https://twitter.com/company",
    "https://linkedin.com/company/company",
  ],
};
---
<script type="application/ld+json" set:html={JSON.stringify(org)} />
```

### Product Schema

```typescript
const product = {
  '@context': 'https://schema.org',
  '@type': 'Product',
  name: 'Product Name',
  description: 'Product description',
  image: 'https://example.com/product.jpg',
  offers: {
    '@type': 'Offer',
    price: '19.99',
    priceCurrency: 'USD',
    availability: 'https://schema.org/InStock',
  },
};
```

---

## Sitemap Generation

### Astro

Use the official integration:

```bash
npx astro add sitemap
```

```javascript
// astro.config.mjs
import sitemap from '@astrojs/sitemap';

export default defineConfig({
  site: 'https://yoursite.com',
  integrations: [sitemap()],
});
```

### Vite + React

Generate during build:

```typescript
// scripts/generate-sitemap.js
import { writeFileSync } from 'fs';

const pages = [
  '/',
  '/about',
  '/products',
  // Add dynamic pages from your data source
];

const sitemap = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  ${pages
    .map(
      (page) => `
    <url>
      <loc>https://yoursite.com${page}</loc>
      <lastmod>${new Date().toISOString().split('T')[0]}</lastmod>
    </url>
  `,
    )
    .join('')}
</urlset>`;

writeFileSync('dist/sitemap.xml', sitemap);
```

Add to build:

```json
{
  "scripts": {
    "build": "vite build && node scripts/generate-sitemap.js"
  }
}
```

---

## robots.txt

```
# public/robots.txt
User-agent: *
Allow: /

Sitemap: https://yoursite.com/sitemap.xml

# Disallow admin and API routes
Disallow: /admin/
Disallow: /api/
```

---

## Performance as SEO

### Core Web Vitals

Performance directly impacts SEO ranking:

- **LCP (Largest Contentful Paint):** < 2.5s
- **INP (Interaction to Next Paint):** < 200ms
- **CLS (Cumulative Layout Shift):** < 0.1

### Optimization Strategies

1. **Image optimization** (use Netlify Image CDN)
2. **Minimal JavaScript** (Astro's default)
3. **Proper caching headers**
4. **Font loading optimization**

```html
<!-- Preload critical fonts -->
<link rel="preload" href="/fonts/main.woff2" as="font" type="font/woff2" crossorigin />

<!-- Preconnect to external domains -->
<link rel="preconnect" href="https://fonts.googleapis.com" />
```

---

## Canonical URLs

Always set canonical URLs to prevent duplicate content:

```astro
<link rel="canonical" href={new URL(Astro.url.pathname, Astro.site)} />
```

For paginated content:

```astro
<link rel="canonical" href={`https://site.com/blog`} />
<link rel="prev" href={prevPage ? `https://site.com/blog?page=${prevPage}` : undefined} />
<link rel="next" href={nextPage ? `https://site.com/blog?page=${nextPage}` : undefined} />
```

---

## Open Graph Images

Create dynamic OG images for sharing:

1. **Static default:** `/public/og-image.png` (1200x630)
2. **Per-page images:** Generate or design for key pages
3. **Dynamic generation:** Consider Netlify Edge Functions or external services

---

## Anti-Patterns

- **Missing meta descriptions** - Every page needs a unique description
- **Duplicate titles** - Each page should have a unique title
- **No canonical URLs** - Can cause duplicate content issues
- **Blocking JavaScript crawling** - Search engines need to render React apps
- **Ignoring Core Web Vitals** - Performance directly impacts ranking
- **No sitemap** - Makes discovery harder for search engines

---

## Related Skills

- `astro-best-practices` - SSR/SSG advantages
- `vite-best-practices` - SPA considerations
- `netlify-images` - Image optimization for performance
- `component-design` - Progressive rendering for performance
