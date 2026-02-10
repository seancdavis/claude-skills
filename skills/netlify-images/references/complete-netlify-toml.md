# Complete netlify.toml Example

Full example of a netlify.toml file configured for image handling with CDN optimization.

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
