---
name: environment-variables
description: Environment variable management for Netlify projects. Use when configuring API keys, managing secrets, or setting up local development environments. Covers pulling from Netlify, CLI management, and the rare cases when local env files are needed.
---

# Environment Variables

## Core Principle

**Always pull from Netlify** — work within the Netlify environment context.

Never store secrets locally when avoidable.

---

## How Variables Are Injected

### Astro Projects

The `netlify dev` wrapper injects Netlify environment variables:

```json
{
  "scripts": {
    "dev": "netlify dev --no-open"
  }
}
```

### Vite + React Projects

The Netlify Vite plugin handles injection:

```typescript
// vite.config.ts
import netlify from '@netlify/vite-plugin';

export default defineConfig({
  plugins: [netlify()],
});
```

No additional configuration needed.

---

## Managing Variables

### View Current Variables

```bash
# List all variables
netlify env:list

# Get specific variable
netlify env:get VARIABLE_NAME
```

### Set Variables

```bash
# Set a non-secret variable (visible in logs)
netlify env:set VARIABLE_NAME "value"

# Set a secret variable (hidden from logs)
netlify env:set --secret API_KEY "sk_..."
```

### Unset Variables

```bash
netlify env:unset VARIABLE_NAME
```

### Import from File

```bash
# Import from .env file (one-time)
netlify env:import .env
```

---

## Common Variables

### Database

```bash
# Automatically set by netlify db init
NETLIFY_DATABASE_URL
```

### AI Gateway

```bash
# Automatically injected by Netlify AI Gateway
OPENAI_API_KEY
OPENAI_BASE_URL
ANTHROPIC_API_KEY
ANTHROPIC_BASE_URL
NETLIFY_AI_GATEWAY_KEY
NETLIFY_AI_GATEWAY_BASE_URL
```

### Authentication

```bash
# Neon Auth URL (manually set after enabling auth)
VITE_NEON_AUTH_URL  # For Vite + React client-side
# Server-side uses origin-relative /neon-auth proxy
```

### Application

```bash
# Log level control
LOG_LEVEL=debug|info|error|silent

# Admin/manager emails (comma-separated)
ADMIN_EMAILS="admin@example.com,another@example.com"
MANAGER_EMAILS="manager@example.com"
```

---

## Accessing Variables

### Server-Side (Node.js)

```typescript
const dbUrl = process.env.NETLIFY_DATABASE_URL;
const logLevel = process.env.LOG_LEVEL || 'info';
```

### Client-Side (Vite)

Only variables prefixed with `VITE_` are exposed:

```typescript
const authUrl = import.meta.env.VITE_NEON_AUTH_URL;
```

```bash
# This is exposed to client
netlify env:set VITE_PUBLIC_API_URL "https://api.example.com"

# This is NOT exposed to client (no VITE_ prefix)
netlify env:set SECRET_API_KEY "sk_..."
```

### Client-Side (Astro)

For SSR, all variables are available server-side. For client-side:

```typescript
// Only PUBLIC_ prefix variables are exposed
const publicKey = import.meta.env.PUBLIC_STRIPE_KEY;
```

---

## Local Development

### Automatic Injection

When you run `netlify dev` or use the Netlify Vite plugin with a linked site, environment variables are automatically pulled from Netlify.

### Rare Exception: .envrc

For truly local-only variables (not stored in Netlify):

```bash
# .envrc (must be in .gitignore)
export LOCAL_DEBUG=true
export SOME_LOCAL_OVERRIDE=value
```

Auto-loads with direnv. **Use sparingly.**

---

## Deploy Contexts

Variables can be scoped to deploy contexts:

```bash
# Production only
netlify env:set --context production API_URL "https://api.prod.com"

# Deploy previews only
netlify env:set --context deploy-preview API_URL "https://api.staging.com"

# Branch-specific
netlify env:set --context branch:feature-x DEBUG=true
```

---

## Security Best Practices

1. **Use --secret for sensitive values**

   ```bash
   netlify env:set --secret API_KEY "sk_live_..."
   ```

2. **Never commit secrets**
   - No `.env` files with secrets in git
   - No hardcoded API keys

3. **Use VITE\_ prefix carefully**
   - Only for values safe to expose publicly
   - API keys should never have VITE\_ prefix

4. **Rotate compromised keys immediately**
   ```bash
   netlify env:set --secret API_KEY "new_key_here"
   ```

---

## Debugging

### Check What's Available

```typescript
// Temporary debug - remove before committing
console.log('Available env vars:', Object.keys(process.env));
```

### Verify Injection

```bash
# In project directory
netlify dev

# Variables should be available to your app
```

### Force Refresh

```bash
# Unlink and relink if variables seem stale
netlify unlink
netlify link
```

---

## Anti-Patterns

- **Storing secrets in .env files** - Use Netlify's env management
- **Using VITE\_ for secrets** - Exposes to client-side code
- **Hardcoding values** - Use environment variables for anything that varies
- **Not using --secret flag** - Secrets should be hidden from logs
- **Checking .env files into git** - Add to .gitignore

---

## Related Skills

- `new-project` - Initial environment setup
- `auth-design` - Auth-related environment variables
- `ai-workflows` - AI Gateway environment variables
- `logging-and-monitoring` - LOG_LEVEL configuration
