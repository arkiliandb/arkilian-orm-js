# Arkilian-orm

💃 Javascript ORM for Arkilian database visit arkilian.com for more info

[![npm version](https://badge.fury.io/js/arkilian-orm-js.svg)](https://www.npmjs.com/package/arkilian-orm-js)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![npm](https://img.shields.io/npm/dm/arkilian-orm-js.svg)](https://www.npmjs.com/package/arkilian-orm-js)

<img src="https://avatars.githubusercontent.com/u/261335565?s=48&v=4" width="120" />
  

## Install

```bash
bun add arkilian-orm
```

## Quick start

Define your schema in `./bruv/schema.prisma`:

```prisma
model User {
  id    String @id @default(uuid())
  email String @unique
  name  String?
  age   Int?
}

model Post {
  id      String @id @default(uuid())
  title   String
  userId  String
}
```

Initialize the database. Schema is auto-loaded from `./bruv/schema.prisma`:

```ts
import { Arkilian_orm } from "arkilian-orm";

const db = new Arkilian_orm({
  localFile: "./app.db",
});
```

Query:

```ts
await db.from("users").insert({ name: "friday", email: "f@dev.io" });

const users = await db.from("users").where("age > ?", 18).get();

const one = await db.from("users").where("id = ?", id).getOne();
```

## Platforms

```ts
// Cloudflare D1
const db = new Arkilian_orm({
  D1Config: {
    accountId: process.env.CFAccountId,
    databaseId: process.env.D1databaseId,
    apiKey: process.env.CFauthorizationToken,
  },
});

// Turso
const db = new Arkilian_orm({
  TursoConfig: {
    url: process.env.TURSO_URL,
    authToken: process.env.TURSO_AUTH_TOKEN,
  },
});

// Local SQLite (default)
const db = new Arkilian_orm({
  localFile: "./app.db",
});
```

## Query API

```ts
// Select
db.from("users").select("id", "name").get();

// Where chains
db.from("users")
  .where("age > ?", 18)
  .andWhere("country = ?", "NG")
  .orWhere("role = ?", "admin")
  .get();

// Pagination
db.from("users").orderBy("name", "ASC").limit(10).offset(20).get();

// Single record
db.from("users").where("id = ?", id).getOne();

// Count
db.from("users").where("active = ?", true).count();

// Insert (id is auto-generated)
db.from("users").insert({ name: "friday", email: "f@dev.io" });

// Update
db.from("users").where("id = ?", id).update({ name: "saturday" });

// Delete
db.from("users").where("id = ?", id).delete();

// Raw SQL
db.raw("SELECT * FROM users WHERE id = ?", [id]);
```

## Caching

```ts
const users = await db.from("users").get({ cacheAs: "all-users" });

// Later
db.invalidateCache("all-users");
```

## Migrations

Prisma-style CLI. Schema source of truth is `./bruv/schema.prisma`. Migrations live in `./bruv/migrations/`.

```bash
# Generate + apply migration (uses local dev.db)
npx bruv-cli migrate dev --name add_age_column

# Apply pending migrations to production DB (reads env vars)
npx bruv-cli migrate deploy

# Rollback last migration
npx bruv-cli migrate reset

# Check status
npx bruv-cli migrate status

# Push schema directly, no migration file
npx bruv-cli db push
```

Connection is resolved from environment variables:

| Env vars                                                | Target          |
| ------------------------------------------------------- | --------------- |
| `ARKILIAN_DATABASE_TOKEN`                        | Turso           | 
| `DB_FILE`                                               | Local file path |
| _(none)_                                                | `./main.db`     |

`migrate dev` always targets `./bruv/dev.db` for local iteration.

Migration files use `-- --> up` / `-- --> down` markers and run inside transactions:

```sql
-- --> up
CREATE TABLE IF NOT EXISTS users (
    id text PRIMARY KEY NOT NULL,
    email TEXT UNIQUE NOT NULL,
    name TEXT
);

-- --> down
DROP TABLE users;
```

## JSON query interface

For exposing queries over HTTP without writing per-route SQL:

```ts
const result = await db.executeJsonQuery({
  from: "users",
  action: "get",
  where: [{ condition: "age > ?", params: [18] }],
  orderBy: { column: "name", direction: "ASC" },
  limit: 10,
});
```

Actions: `get`, `getOne`, `insert`, `update`, `delete`, `count`.

## Security

The query builder rejects dangerous input at the condition level:

- Parameterized queries only — no string interpolation
- Blocked patterns: `; DROP`, `UNION`, `DELETE`, `INSERT`, `UPDATE`, `ALTER`, `EXEC`
- Whitelisted operators: `=`, `>`, `<`, `>=`, `<=`, `LIKE`, `IN`, `BETWEEN`, `IS NULL`, `IS NOT NULL`
- Max 100 params per query, max 1000 chars per string param

`raw()` bypasses all validation — use it for migrations and admin queries only.

### Examples

```typescript
// ✅ Safe queries
db.from("users")
  .where("email LIKE ?", "%@example.com") // ✅ Safe
  .andWhere("role = ?", "admin") // ✅ Safe
  .get();
db.from("users")
  .where("age > ?", 18)
  .andWhere("status = ?", "active")
  .orWhere("role IN (?)", ["admin", "mod"]);

// ❌ These will throw security errors:
db.where("1=1; DROP TABLE users;"); // Dangerous pattern
db.where("col = (SELECT ...)"); // Complex subqueries blocked
db.where("name = ?", "a".repeat(1001)); // String too long
```

## Contributing

1. Fork
2. Branch (`git checkout -b feature/your thing`)
3. Commit
4. PR

## License

MIT
