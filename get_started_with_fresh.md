# Express + Prisma + Prisma Studio — Fresh Start (Step‑by‑step)

A compact, practical README to get a fresh Express app working with **Prisma**, **migrations** (including a "remigrate" workflow without full DB reset), **Prisma Studio**, and optional seeding. Designed for local development (Postgres example) and includes TypeScript examples.

---

## Table of contents

1. Overview
2. Prerequisites
3. Quickstart (commands)
4. Project layout (recommended)
5. Install & init (Bun / npm)
6. Configure `.env` and `schema.prisma`
7. Initial migration & Prisma Client
8. Remigrate workflow (apply changes without reset)
9. Seed data (optional)
10. Example Express (TypeScript) code
11. Useful package.json scripts
12. Prisma Studio
13. Troubleshooting & tips
14. Commands cheat sheet

---

## 1) Overview

This repo will give you:

* A minimal Express server (TypeScript) using Prisma Client
* A `prisma/schema.prisma` ready to evolve
* A repeatable migration workflow for development that preserves data when possible
* Prisma Studio for browsing/editing rows during development

All examples use **PostgreSQL** (localhost:5432) but adapt easily to other databases by changing the datasource provider.

---

## 2) Prerequisites

* Node.js (v18+ recommended) or Bun
* PostgreSQL running and accessible (example: `postgres://postgres:password@localhost:5432/express_prisma`)
* git

Optional (if using TypeScript):

* TypeScript (`tsc`), `ts-node`, `nodemon`

---

## 3) Quickstart (copy + paste)

Using **npm / npx** (TypeScript):

```bash
mkdir express-prisma-app && cd express-prisma-app
npm init -y
npm install express @prisma/client
npm install -D prisma typescript ts-node nodemon @types/express
npx prisma init
# edit .env and prisma/schema.prisma as shown below
npx prisma migrate dev --name init
npx prisma generate
npm run dev
# open prisma studio in new terminal
npx prisma studio
```

Using **bun**:

```bash
bun init -y
bun add express @prisma/client
bun add -d prisma typescript ts-node nodemon @types/express
npx prisma init
# edit .env and prisma/schema.prisma
bunx prisma migrate dev --name init
bunx prisma generate
bun run dev
bunx prisma studio
```

---

## 4) Recommended project layout

```
express-prisma-app/
├─ prisma/
│  ├─ schema.prisma
│  └─ migrations/
├─ src/
│  ├─ index.ts      # server
│  └─ db.ts         # prisma client
├─ prisma/seed.ts   # optional seeding
├─ .env
├─ package.json
└─ tsconfig.json
```

---

## 5) Install & init (detailed)

1. Install deps (see Quickstart above).
2. `npx prisma init` will create `.env` and `prisma/schema.prisma`.
3. Edit `.env` to point to your database.

`.env` example:

```env
DATABASE_URL="postgresql://postgres:your_password@localhost:5432/express_prisma"
```

> **Tip:** Use `postgresql://` (Prisma recommends it), and ensure the user/password/DB exist.

---

## 6) Example `schema.prisma`

Put this into `prisma/schema.prisma` (basic blog example with enums):

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum Role {
  USER
  ADMIN
}

enum UserStatus {
  ACTIVE
  INACTIVE
}

model User {
  id       Int        @id @default(autoincrement())
  email    String     @unique
  name     String?
  role     Role       @default(USER)
  status   UserStatus @default(ACTIVE)
  posts    Post[]
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
}
```

---

## 7) Initial migration & Prisma Client

1. Create & apply first migration (dev):

```bash
npx prisma migrate dev --name init
```

This creates `prisma/migrations/YYYYMMDDHHmmss_init/` with SQL, applies it to your DB and generates `@prisma/client`.

2. Generate client (if needed):

```bash
npx prisma generate
```

---

## 8) Remigrate workflow — make schema changes without resetting (dev)

When you change `prisma/schema.prisma` (add model, field, enum, etc.), use one of these safe workflows depending on your situation.

### A — Normal dev workflow (recommended when migrations history is healthy)

1. Edit `prisma/schema.prisma` (e.g. add `Post` or new column).
2. Run:

```bash
npx prisma migrate dev --name add-post-model
```

This **generates** a migration file and **applies** it to your DB (no reset). Good for normal, linear development.

### B — Edit the SQL before applying (avoid destructive generated SQL)

If a generated migration will cause data loss (Prisma warns about destructive changes), you can:

1. Create migration files **without applying**:

```bash
npx prisma migrate dev --create-only --name add-post-model
```

2. Edit the SQL in the newly created folder under `prisma/migrations/<timestamp>_add-post-model/migration.sql` to customize the migration (move or preserve data, use temporary tables, etc.).

3. Apply the edited migration (locally):

```bash
npx prisma migrate dev --name apply-edited
# or simply run npx prisma migrate dev which detects unapplied migrations
```

> Note: `--create-only` lets you customize the SQL before applying. Use this when you need to preserve data across structural changes. citeturn1search5

### C — Prototyping: fast changes without generating migrations

If you are still experimenting (no migration history or disposable DB), you can use:

```bash
npx prisma db push
```

* `db push` syncs the Prisma schema to the DB **without creating migration files**. It’s great for prototyping but **not** recommended for long-term or production use. citeturn1search2

### D — If Prisma detects schema *drift*

If `prisma migrate dev` reports **drift detected** (your DB schema doesn't match migration history), it will usually prompt you to **reset** the DB. You have options:

* If you can accept wiping development data: accept reset (`prisma migrate reset`) and re-run migrations.
* If you cannot wipe data: create a safe migration with `--create-only`, edit SQL to match DB, or use `prisma migrate resolve` to mark migrations as applied after confirming DB state. citeturn1search2turn0search2

> Short guidance: For dev, prefer the regular `migrate dev` flow. If Prisma wants to reset because of drift, use `--create-only` to craft a custom migration or `db push` for quick prototyping. Don’t use `db push` in production.

---

## 9) Seeding (optional)

Create `prisma/seed.ts` (TypeScript) or `prisma/seed.js`.

`prisma/seed.ts` example:

```ts
import { PrismaClient } from '@prisma/client'
const prisma = new PrismaClient()

async function main(){
  await prisma.user.create({ data: { email: 'admin@example.com', name: 'Admin', role: 'ADMIN' } })
}

main()
  .catch(e => { console.error(e); process.exit(1) })
  .finally(async () => { await prisma.$disconnect() })
```

Tell Prisma about the seed script in `package.json`:

```json
"prisma": {
  "seed": "ts-node prisma/seed.ts"
}
```

Running `npx prisma migrate reset` will now run the seed after re-applying migrations. (Be careful: `reset` drops data.)

---

## 10) Example Express server (TypeScript)

`src/db.ts`:

```ts
import { PrismaClient } from '@prisma/client'
const prisma = new PrismaClient()
export default prisma
```

`src/index.ts`:

```ts
import express from 'express'
import prisma from './db'

const app = express()
app.use(express.json())

app.post('/users', async (req, res) => {
  const { name, email } = req.body
  try {
    const user = await prisma.user.create({ data: { name, email } })
    res.json(user)
  } catch (err) {
    res.status(400).json({ error: 'Invalid or duplicate data' })
  }
})

app.get('/users', async (req, res) => {
  const users = await prisma.user.findMany()
  res.json(users)
})

app.listen(4000, () => console.log('Server ready: http://localhost:4000'))
```

---

## 11) Useful `package.json` scripts

```json
"scripts": {
  "dev": "nodemon --exec ts-node src/index.ts",
  "build": "tsc",
  "start": "node dist/index.js",
  "studio": "prisma studio",
  "migrate:dev": "prisma migrate dev",
  "migrate:create": "prisma migrate dev --create-only",
  "migrate:deploy": "prisma migrate deploy",
  "db:push": "prisma db push",
  "prisma:generate": "prisma generate",
  "seed": "ts-node prisma/seed.ts"
}
```

> When you need to pass the migration name from the CLI, do:

```bash
npm run migrate:create -- --name add-post-model
# then edit SQL and apply with
npm run migrate:dev -- --name apply-edited
```

---

## 12) Prisma Studio

Run:

```bash
npx prisma studio
# or
npm run studio
# or bunx prisma studio
```

By default Studio opens at `http://localhost:5555` and lets you inspect and edit rows.

---

## 13) Troubleshooting & tips

* `Error: P1000 Authentication failed` → check `.env` DATABASE\_URL credentials and that Postgres accepts connections. Ensure correct protocol: `postgresql://`.
* `bash: prisma: command not found` → either run via `npx prisma` or install the CLI globally (`npm i -g prisma`) or use `bunx prisma` if using Bun.
* `Drift detected` → read section **Remigrate workflow** above. Prefer `--create-only` + manual SQL if you can’t reset dev DB. citeturn1search2turn1search5
* If you accidentally used `db push` on a DB you intend to migrate later, you may end up in a drift state — fix by creating a migration with `--create-only` and reconciling the SQL.

---

## 14) Commands cheat sheet

```
npx prisma init                      # create prisma folder + .env
npx prisma migrate dev --name init   # create+apply migration (dev)
npx prisma migrate dev --create-only --name X  # create migration files only
npx prisma migrate deploy            # apply migrations (CI / production)
npx prisma db push                   # push schema to DB (no migration files)
npx prisma studio                    # open Prisma Studio
npx prisma generate                  # generate Prisma Client
npm run dev                          # start dev server
```

---

<!-- End of README -->
