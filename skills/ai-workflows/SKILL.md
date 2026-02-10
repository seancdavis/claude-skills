---
name: ai-workflows
description: Integrating AI capabilities via Netlify's AI Gateway and Agent Runners. Use when adding AI features to applications, calling LLM APIs, or implementing AI-powered workflows. Covers environment variable setup, SDK usage, and available models.
---

# AI Workflows

## Two Services

### AI Gateway

- Provides access to models from Anthropic, OpenAI, and Gemini
- No third-party accounts needed
- Environment variables automatically injected
- Works with official SDKs out of the box

### Agent Runners

- Built on top of AI Gateway
- For more complex agentic workflows
- Has its own API

---

## AI Gateway Setup

### Automatic Environment Variables

When running in a Netlify environment (local dev with `netlify dev` or deployed), these are automatically injected:

```bash
# OpenAI SDK compatibility
OPENAI_API_KEY
OPENAI_BASE_URL

# Anthropic SDK compatibility
ANTHROPIC_API_KEY
ANTHROPIC_BASE_URL

# Netlify-specific
NETLIFY_AI_GATEWAY_KEY
NETLIFY_AI_GATEWAY_BASE_URL
```

**No manual setup required** — just use the SDKs.

### Verify AI Gateway is Available

```typescript
if (!process.env.OPENAI_BASE_URL) {
  throw new Error('AI Gateway not available. Run with netlify dev or deploy to Netlify.');
}
```

---

## Using the OpenAI SDK

### Installation

```bash
npm install openai
```

### Basic Usage

```typescript
import OpenAI from 'openai';

const client = new OpenAI();
// Automatically uses OPENAI_API_KEY and OPENAI_BASE_URL from environment

const response = await client.chat.completions.create({
  model: 'gpt-4o-mini',
  messages: [
    { role: 'system', content: 'You are a helpful assistant.' },
    { role: 'user', content: 'Explain recursion in one sentence.' },
  ],
});

const answer = response.choices[0].message.content;
```

### With Streaming

```typescript
const stream = await client.chat.completions.create({
  model: 'gpt-4o-mini',
  messages: [{ role: 'user', content: 'Write a haiku about coding.' }],
  stream: true,
});

for await (const chunk of stream) {
  const content = chunk.choices[0]?.delta?.content || '';
  process.stdout.write(content);
}
```

---

## Using the Anthropic SDK

### Installation

```bash
npm install @anthropic-ai/sdk
```

### Basic Usage

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();
// Automatically uses ANTHROPIC_API_KEY and ANTHROPIC_BASE_URL from environment

const response = await client.messages.create({
  model: 'claude-sonnet-4-5-20250929',
  max_tokens: 1024,
  messages: [{ role: 'user', content: 'Explain the builder pattern in TypeScript.' }],
});

const answer = response.content[0].type === 'text' ? response.content[0].text : '';
```

### With System Prompt

```typescript
const response = await client.messages.create({
  model: 'claude-sonnet-4-5-20250929',
  max_tokens: 1024,
  system: 'You are a senior TypeScript developer. Be concise.',
  messages: [{ role: 'user', content: 'Review this code for issues: ...' }],
});
```

---

## In Netlify Functions

```typescript
// netlify/functions/ai-summarize.ts
import type { Context, Config } from '@netlify/functions';
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic();

export default async (request: Request, context: Context) => {
  if (request.method !== 'POST') {
    return new Response('Method not allowed', { status: 405 });
  }

  const { text } = await request.json();

  if (!text || typeof text !== 'string') {
    return Response.json({ error: 'Text is required' }, { status: 400 });
  }

  try {
    const response = await client.messages.create({
      model: 'claude-sonnet-4-5-20250929',
      max_tokens: 256,
      messages: [
        {
          role: 'user',
          content: `Summarize this text in 2-3 sentences:\n\n${text}`,
        },
      ],
    });

    const summary = response.content[0].type === 'text' ? response.content[0].text : '';

    return Response.json({ summary });
  } catch (error) {
    console.error('AI error:', error);
    return Response.json({ error: 'AI processing failed' }, { status: 500 });
  }
};

export const config: Config = {
  path: '/api/summarize',
  method: 'POST',
};
```

---

## Available Models

### Via AI Gateway

Access 30+ models including:

| Provider  | Models                                                                          |
| --------- | ------------------------------------------------------------------------------- |
| Anthropic | claude-opus-4-5-20251101, claude-sonnet-4-5-20250929, claude-haiku-4-5-20251001 |
| OpenAI    | gpt-4o, gpt-4o-mini, gpt-5-mini                                                 |
| Google    | gemini-2.5-pro, gemini-2.0-flash                                                |

### Model Selection

```typescript
// Cost-effective for simple tasks
model: 'gpt-4o-mini';
model: 'claude-haiku-4-5-20251001';

// More capable for complex reasoning
model: 'claude-sonnet-4-5-20250929';
model: 'gpt-4o';

// Most capable
model: 'claude-opus-4-5-20251101';
```

---

## In Astro API Routes

```typescript
// src/pages/api/ai/cleanup.ts
import type { APIRoute } from 'astro';
import Anthropic from '@anthropic-ai/sdk';
import { getUserWithApproval } from '../../../lib/auth';

const client = new Anthropic();

export const POST: APIRoute = async ({ request, redirect }) => {
  const auth = await getUserWithApproval(request);
  if (!auth?.isAdmin) {
    return redirect('/unauthorized', 302);
  }

  const formData = await request.formData();
  const text = formData.get('text')?.toString();

  if (!text) {
    return redirect('/?message=validation_error', 302);
  }

  try {
    const response = await client.messages.create({
      model: 'claude-sonnet-4-5-20250929',
      max_tokens: 1024,
      messages: [
        {
          role: 'user',
          content: `Clean up this text, fixing grammar and formatting:\n\n${text}`,
        },
      ],
    });

    const cleaned = response.content[0].type === 'text' ? response.content[0].text : text;

    // Store result and redirect
    await saveCleanedText(cleaned);
    return redirect('/?message=ai_cleanup_complete', 302);
  } catch (error) {
    console.error('AI cleanup error:', error);
    return redirect('/?message=ai_error', 302);
  }
};
```

---

## Error Handling

```typescript
try {
  const response = await client.messages.create({ ... });
} catch (error) {
  if (error instanceof Anthropic.APIError) {
    console.error("API Error:", error.status, error.message);

    if (error.status === 429) {
      // Rate limited - retry with backoff
    } else if (error.status === 400) {
      // Invalid request - check input
    }
  }
  throw error;
}
```

---

## Cost Considerations

- AI Gateway usage is metered on Netlify credit-based plans
- Use smaller/cheaper models for simple tasks
- Cache responses when appropriate
- Set reasonable max_tokens limits

---

## Anti-Patterns

- **Using AI for trivial tasks** - Only use when it adds value
- **No error handling** - AI APIs can fail; handle gracefully
- **Exposing raw AI output** - Validate/sanitize before displaying
- **Ignoring rate limits** - Implement retry logic
- **Not setting max_tokens** - Can lead to expensive responses

---

## Related Skills

- `netlify-functions` - API endpoint implementation
- `feedback` - Showing AI operation status to users
- `logging-and-monitoring` - Logging AI operations
- `environment-variables` - AI Gateway environment variables
