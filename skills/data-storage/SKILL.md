---
name: data-storage
description: Database and data storage conventions for Netlify projects using Netlify DB (Neon) with Drizzle ORM. Use when setting up databases, writing schemas, creating migrations, or deciding between Netlify Blobs vs Netlify DB. Covers Drizzle configuration, migration strategies, and npm scripts.
---

# Data Storage

## Decision Framework

### Netlify Blobs

Use when:

- Storing files (images, documents, etc.)
- A handful of records with no growth expectation
- No relational data needs
- Want zero third-party dependencies

### Netlify DB (Neon)

Use when:

- Storing structured data
- Data will grow over time
- Need queries, filtering, relationships
- Need SQL capabilities

**Default to Netlify DB** for most non-trivial applications.

---

## Setup

### Initialize Netlify DB

```bash
# Link project first
netlify link

# Initialize database
netlify db init
```

This provisions a Neon database and sets up Drizzle ORM automatically.

### Claim the Neon Database

After `netlify db init`, follow the prompt to claim the database in your Neon account, or go to Netlify Dashboard → Site → Neon tab.

---

## Drizzle Configuration

### drizzle.config.ts

```typescript
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.NETLIFY_DATABASE_URL!,
  },
  schema: './db/schema.ts',
  out: './migrations',
  migrations: {
    prefix: 'timestamp', // Use Unix timestamps, not sequential numbers
  },
});
```

**Important:** Use `prefix: 'timestamp'` to avoid migration conflicts across parallel branches.

### Database Connection

```typescript
// db/index.ts
import { neon } from '@neondatabase/serverless';
import { drizzle } from 'drizzle-orm/neon-http';
import * as schema from './schema';

const sql = neon(process.env.NETLIFY_DATABASE_URL!);
export const db = drizzle(sql, { schema });

// Re-export schema for convenience
export * from './schema';
```

---

## Schema Patterns

### Basic Table

```typescript
// db/schema.ts
import { integer, pgTable, varchar, text, boolean, timestamp } from 'drizzle-orm/pg-core';

export const items = pgTable('items', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  title: varchar({ length: 255 }).notNull(),
  description: text(),
  isActive: boolean('is_active').notNull().default(true),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
});

// Type inference
export type Item = typeof items.$inferSelect;
export type NewItem = typeof items.$inferInsert;
```

### Table with Relations

```typescript
export const posts = pgTable('posts', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  title: varchar({ length: 255 }).notNull(),
  content: text(),
  authorId: integer('author_id')
    .notNull()
    .references(() => users.id),
  createdAt: timestamp('created_at').defaultNow(),
});

export const users = pgTable('users', {
  id: integer().primaryKey().generatedAlwaysAsIdentity(),
  email: varchar({ length: 255 }).notNull().unique(),
  name: varchar({ length: 255 }),
});
```

### Table with Unique Index

```typescript
import { uniqueIndex } from 'drizzle-orm/pg-core';

export const gameAccess = pgTable(
  'game_access',
  {
    id: integer().primaryKey().generatedAlwaysAsIdentity(),
    gameId: integer('game_id')
      .notNull()
      .references(() => games.id),
    userEmail: varchar('user_email', { length: 255 }).notNull(),
    role: varchar({ length: 20 }).notNull().default('player'),
  },
  (table) => [uniqueIndex('game_access_game_user_idx').on(table.gameId, table.userEmail)],
);
```

---

## npm Scripts

```json
{
  "scripts": {
    "db:generate": "drizzle-kit generate",
    "db:migrate": "netlify dev:exec drizzle-kit migrate",
    "db:migrate:prod": "node scripts/migrate-production.js --production",
    "db:push": "netlify dev:exec drizzle-kit push",
    "db:studio": "netlify dev:exec drizzle-kit studio"
  }
}
```

### Script Purposes

| Script            | Purpose                                             |
| ----------------- | --------------------------------------------------- |
| `db:generate`     | Generate migration files after schema changes       |
| `db:migrate`      | Run migrations on preview/dev branch                |
| `db:migrate:prod` | Run migrations on production (with confirmation)    |
| `db:push`         | Push schema directly (dev only, no migration files) |
| `db:studio`       | Open Drizzle Studio for database browsing           |

### Production Migration Script

```javascript
// scripts/migrate-production.js
import { execSync } from 'child_process';
import readline from 'readline';

const rl = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

console.log('\n⚠️  PRODUCTION DATABASE MIGRATION\n');
console.log('This will run migrations against the PRODUCTION database.');
console.log('Make sure you are on the main branch and have pulled latest.\n');

rl.question('Type the project name to confirm: ', (answer) => {
  if (answer === 'your-project-name') {
    console.log('\nRunning production migration...\n');
    execSync('netlify exec drizzle-kit migrate', { stdio: 'inherit' });
  } else {
    console.log('\nMigration cancelled.');
  }
  rl.close();
});
```

---

## Query Patterns

### Basic Queries

```typescript
import { db, items } from '../db';
import { eq, desc, and, gte } from 'drizzle-orm';

// Select all
const allItems = await db.select().from(items);

// Select with filter
const activeItems = await db.select().from(items).where(eq(items.isActive, true));

// Select single by ID
const [item] = await db.select().from(items).where(eq(items.id, itemId)).limit(1);

// Select with multiple conditions
const recentActiveItems = await db
  .select()
  .from(items)
  .where(and(eq(items.isActive, true), gte(items.createdAt, new Date('2024-01-01'))))
  .orderBy(desc(items.createdAt));
```

### Insert

```typescript
// Insert single
const [newItem] = await db
  .insert(items)
  .values({
    title: 'New Item',
    description: 'Description here',
  })
  .returning();

// Insert multiple
await db.insert(items).values([{ title: 'Item 1' }, { title: 'Item 2' }]);
```

### Update

```typescript
const [updatedItem] = await db
  .update(items)
  .set({
    title: 'Updated Title',
    updatedAt: new Date(),
  })
  .where(eq(items.id, itemId))
  .returning();
```

### Delete

```typescript
await db.delete(items).where(eq(items.id, itemId));
```

---

## Database Branches

Netlify DB supports branching:

- **Production branch:** Connected to `main` branch deploys
- **Preview branch:** Used for all other branches and deploy previews

### Workflow

1. Develop and test migrations on preview branch
2. Merge to main
3. Run `db:migrate:prod` to apply to production

**Future consideration:** Run migrations as part of Netlify build (but not for deploy previews to avoid sync issues).

---

## Anti-Patterns

- **Writing raw SQL** - Always use Drizzle ORM
- **Sequential migration numbers** - Use timestamp prefix to avoid conflicts
- **Editing migration files directly** - Generate new migrations instead
- **Using `db:push` for schema changes** - Never use `db:push` — always use `db:generate` + `db:migrate` for all schema changes, including development. `db:push` bypasses migration tracking, which leads to orphaned migration files that can't be applied and creates mismatches between the migration history and the actual database state. The only exception is initial project scaffolding before the first migration exists.
- **Storing files in database** - Use Netlify Blobs for files
- **Returning other users' data** - Scope user-owned data at the query level. If a table has a `userId` column (or equivalent foreign key to a users table), every SELECT, UPDATE, and DELETE query on that table must include a `WHERE userId = ?` condition for the authenticated user. Never return another user's data by omitting this filter. This applies to both direct queries and sub-resource lookups (e.g., if fetching sessions for a run, verify the run belongs to the user first).

---

## Related Skills

- `file-storage` - Netlify Blobs for files
- `netlify-images` - Image storage and CDN
- `auth-design` - approved_users table pattern
- `new-project` - Database setup during scaffolding
