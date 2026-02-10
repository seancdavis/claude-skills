---
name: logging-and-monitoring
description: Server-side logging conventions and external monitoring patterns. Use when implementing logging utilities, debugging server code, or setting up external notifications. Covers the three-level logging approach (debug, info, error), scoped loggers, and Discord notification integration (aspirational).
---

# Logging and Monitoring

## Logging Utility

### Three Log Levels

| Level   | When to Use                          | Default State  |
| ------- | ------------------------------------ | -------------- |
| `info`  | Database mutations, important events | Always shown   |
| `error` | Failures, exceptions                 | Always shown   |
| `debug` | Verbose diagnostic info              | Off by default |

### Implementation

```typescript
// src/lib/logger.ts
type LogLevel = 'debug' | 'info' | 'warn' | 'error' | 'silent';

interface LoggerConfig {
  level: LogLevel;
  colors: boolean;
}

const colors = {
  reset: '\x1b[0m',
  bold: '\x1b[1m',
  dim: '\x1b[2m',
  red: '\x1b[31m',
  green: '\x1b[32m',
  yellow: '\x1b[33m',
  blue: '\x1b[34m',
  magenta: '\x1b[35m',
  cyan: '\x1b[36m',
};

const levelConfig = {
  debug: { priority: 0, color: colors.magenta, label: 'DEBUG' },
  info: { priority: 1, color: colors.cyan, label: ' INFO' },
  warn: { priority: 2, color: colors.yellow, label: ' WARN' },
  error: { priority: 3, color: colors.red, label: 'ERROR' },
};

function getConfig(): LoggerConfig {
  const level = (process.env.LOG_LEVEL || 'info').toLowerCase() as LogLevel;
  const colorsEnabled = process.env.LOG_COLORS !== 'false';

  return {
    level: ['debug', 'info', 'warn', 'error', 'silent'].includes(level) ? level : 'info',
    colors: colorsEnabled,
  };
}

function shouldLog(level: keyof typeof levelConfig): boolean {
  const config = getConfig();
  if (config.level === 'silent') return false;
  return levelConfig[level].priority >= levelConfig[config.level].priority;
}

function formatTimestamp(): string {
  return new Date().toLocaleTimeString('en-US', {
    hour12: false,
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit',
  });
}

function log(level: keyof typeof levelConfig, scope: string, ...args: unknown[]): void {
  if (!shouldLog(level)) return;

  const config = getConfig();
  const { color, label } = levelConfig[level];
  const timestamp = formatTimestamp();

  const message = args
    .map((arg) => (typeof arg === 'object' ? JSON.stringify(arg, null, 2) : String(arg)))
    .join(' ');

  if (config.colors) {
    const formatted = `${color}${colors.bold}[${label}]${colors.reset} ${colors.dim}[${scope}] ${timestamp}${colors.reset} ${message}`;
    console.log(formatted);
  } else {
    console.log(`[${label}] [${scope}] ${timestamp} ${message}`);
  }
}

// Scoped logger for convenience
interface ScopedLogger {
  debug: (...args: unknown[]) => void;
  info: (...args: unknown[]) => void;
  warn: (...args: unknown[]) => void;
  error: (...args: unknown[]) => void;
}

function createScopedLogger(scope: string): ScopedLogger {
  return {
    debug: (...args) => log('debug', scope, ...args),
    info: (...args) => log('info', scope, ...args),
    warn: (...args) => log('warn', scope, ...args),
    error: (...args) => log('error', scope, ...args),
  };
}

export const logger = {
  debug: (scope: string, ...args: unknown[]) => log('debug', scope, ...args),
  info: (scope: string, ...args: unknown[]) => log('info', scope, ...args),
  warn: (scope: string, ...args: unknown[]) => log('warn', scope, ...args),
  error: (scope: string, ...args: unknown[]) => log('error', scope, ...args),
  scope: createScopedLogger,
};

export type { ScopedLogger };
```

---

## Usage

### Scoped Logger (Preferred)

```typescript
import { logger } from '../lib/logger';

const log = logger.scope('ENTRIES');

log.debug('Fetching entries for user:', userId);
log.info('Entry created:', entry.id);
log.warn('User attempted to access locked resource');
log.error('Failed to save entry:', error.message);
```

### Direct Logger

```typescript
import { logger } from '../lib/logger';

logger.info('AUTH', 'User authenticated:', user.email);
logger.error('DB', 'Query failed:', error);
```

---

## Environment Variables

```bash
# Set log level (default: info)
LOG_LEVEL=debug    # Show all logs including debug
LOG_LEVEL=info     # Show info, warn, error (default)
LOG_LEVEL=error    # Show only errors
LOG_LEVEL=silent   # Disable all logs

# Disable colors (for CI/log aggregators)
LOG_COLORS=false
```

---

## What to Log

### Info Level (Always Logged)

```typescript
// Database mutations with summary context
log.info('Entry created for:', user.email);
log.info('User profile updated:', { userId, fieldsChanged: ['name', 'image'] });
log.info('Session established for:', user.email);

// Don't log all data, just enough to identify what happened
```

### Error Level (Always Logged)

```typescript
// Provide helpful context
log.error('Failed to create entry:', error.message);
log.error('Database connection failed:', { host, error: error.message });

// Surface to user via feedback system
return redirect('/form?message=create_failed', 302);
```

### Debug Level (Verbose, Off by Default)

```typescript
// Intentionally verbose - once added, leave in place
log.debug('Request headers:', Object.fromEntries(request.headers));
log.debug('Session response:', response.status, response.headers);
log.debug('Query params:', { page, limit, filter });
```

---

## Auth Logging

Always log authentication events:

```typescript
const log = logger.scope('AUTH');

// Successful auth
log.info('User authenticated:', user.email);

// Failed auth
log.warn('Invalid session verifier received');

// Unapproved user
log.warn('Unapproved user attempted access:', email);

// Sign out
log.info('User signed out:', user.email);
```

---

## Discord Notifications (Aspirational)

External monitoring via Discord webhooks. This is for future implementation.

### Pattern: Notification Service

```typescript
// src/lib/notifications.ts
interface DiscordMessage {
  content?: string;
  embeds?: DiscordEmbed[];
}

interface DiscordEmbed {
  title: string;
  description?: string;
  color?: number;
  fields?: { name: string; value: string; inline?: boolean }[];
  timestamp?: string;
}

async function sendToDiscord(webhookUrl: string, message: DiscordMessage): Promise<void> {
  try {
    await fetch(webhookUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(message),
    });
  } catch (error) {
    // Don't let notification failures break the app
    console.error('Discord notification failed:', error);
  }
}

const COLORS = {
  info: 0x3498db, // Blue
  success: 0x2ecc71, // Green
  warning: 0xf1c40f, // Yellow
  error: 0xe74c3c, // Red
};

export async function notifyError(app: string, title: string, description: string): Promise<void> {
  const webhookUrl = process.env.DISCORD_ERROR_WEBHOOK;
  if (!webhookUrl) return;

  await sendToDiscord(webhookUrl, {
    embeds: [
      {
        title: `🚨 ${app}: ${title}`,
        description,
        color: COLORS.error,
        timestamp: new Date().toISOString(),
      },
    ],
  });
}

export async function notifyAuth(
  app: string,
  event: 'signin' | 'signout' | 'unauthorized',
  email: string,
): Promise<void> {
  const webhookUrl = process.env.DISCORD_AUTH_WEBHOOK;
  if (!webhookUrl) return;

  const icons = {
    signin: '🔓',
    signout: '🔒',
    unauthorized: '⚠️',
  };

  await sendToDiscord(webhookUrl, {
    embeds: [
      {
        title: `${icons[event]} ${app}: ${event}`,
        description: email,
        color: event === 'unauthorized' ? COLORS.warning : COLORS.info,
        timestamp: new Date().toISOString(),
      },
    ],
  });
}
```

### Channel Strategy

- **Errors channel:** All error-level events
- **Auth channel:** Sign in, sign out, unauthorized attempts

---

## Sentry (Future)

Not currently in use. Will be introduced for serious production applications.

When implemented, will provide:

- Error tracking and grouping
- Performance monitoring
- Release tracking

---

## Anti-Patterns

- **Logging sensitive data** - Never log passwords, tokens, full session data
- **Excessive info logging** - Keep info logs summary-level
- **Removing debug logs** - Leave them in for future debugging
- **Silent failures** - Always log errors, even if you handle them gracefully
- **Not using scopes** - Scopes make filtering logs much easier

---

## Related Skills

- `auth-design` - Auth events to log
- `feedback` - Error messages for users
- `environment-variables` - Configuring log levels
