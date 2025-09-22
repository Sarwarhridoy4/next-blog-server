

## 1. Install Prisma & Client

Inside your project folder:

```bash
bun add prisma @prisma/client --dev
```

---

## 2. Initialize Prisma

Run:

```bash
npx prisma init
```

This will create:

```
📂 prisma/
   └── schema.prisma
.env
```

---

## 3. Configure `.env`

Edit `.env` and add your PostgreSQL connection string:

```env
DATABASE_URL="postgresql://postgres:your_password@localhost:5432/next_blog"
```

---

## 4. Define Your Data Model

Open `prisma/schema.prisma` and add your models.
Example for a blog project:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id       Int      @id @default(autoincrement())
  email    String   @unique
  name     String?
  posts    Post[]
  role     Role     @default(USER)
  status   UserStatus @default(ACTIVE)
}

model Post {
  id        Int      @id @default(autoincrement())
  title     String
  content   String?
  published Boolean  @default(false)
  author    User     @relation(fields: [authorId], references: [id])
  authorId  Int
}

enum Role {
  USER
  ADMIN
}

enum UserStatus {
  ACTIVE
  INACTIVE
}
```

---

## 5. Migrate Your Database

To apply the schema to your database:

```bash
npx prisma migrate dev --name init
```

If your DB already has tables and Prisma complains about **drift**, reset it:

```bash
npx prisma migrate reset
```

⚠️ This will drop all data in development DB.

---

## 6. Generate Prisma Client

Prisma Client is auto-generated after migrations, but you can regenerate anytime:

```bash
npx prisma generate
```

---

## 7. Use Prisma Client in Code

Example `db.ts`:

```ts
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();
export default prisma;
```

Example query in your code:

```ts
import prisma from "./db";

// Create a user
const user = await prisma.user.create({
  data: { email: "test@example.com", name: "Sarwar" },
});

// Fetch all users
const users = await prisma.user.findMany();
console.log(users);
```

---

## 8. Optional: Seeding

If you want test data auto-inserted after `migrate reset`, create `prisma/seed.ts`:

```ts
import { PrismaClient } from "@prisma/client";

const prisma = new PrismaClient();

async function main() {
  await prisma.user.create({
    data: {
      email: "admin@example.com",
      name: "Admin User",
      role: "ADMIN",
    },
  });
}

main()
  .then(() => console.log("Seed data created"))
  .catch((e) => console.error(e))
  .finally(async () => {
    await prisma.$disconnect();
  });
```

Then tell Prisma to use it (in `package.json`):

```json
"prisma": {
  "seed": "ts-node prisma/seed.ts"
}
```

---

✅ Now your **existing project is fully Prisma-ready**:

1. Define schema in `prisma/schema.prisma`
2. Run `npx prisma migrate dev`
3. Use `prisma` client in your app

