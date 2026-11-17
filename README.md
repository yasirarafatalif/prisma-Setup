# Node.js + TypeScript + Express + Prisma 7 (PostgreSQL)

এই README-তে Node.js, TypeScript, Express, Prisma 7 এবং PostgreSQL দিয়ে একটি backend project শুরু থেকে setup করার ধাপগুলো দেওয়া হয়েছে।

> **Note:** এখানে Prisma 7 ব্যবহার করা হয়েছে। Prisma 7-এর configuration আগের version থেকে কিছুটা আলাদা। তাই `prisma.config.ts`, schema generator-এর `output` এবং PostgreSQL adapter setup-ও যোগ করা হয়েছে।

## 1. Project initialize

প্রথমে project folder-এ terminal open করে চালাও:

```bash
npm init -y
npm install -D typescript tsx @types/node
npx tsc --init
```

## 2. Prisma 7 এবং PostgreSQL packages install

```bash
npm install -D prisma@7 @types/pg
npm install @prisma/client@7 @prisma/adapter-pg pg dotenv
```

Express project-এ PostgreSQL database ব্যবহার করার জন্য `pg` এবং Prisma 7-এর PostgreSQL adapter `@prisma/adapter-pg` লাগবে।

## 3. TypeScript configuration

Project root-এর `tsconfig.json` ফাইলটি নিচের মতো update করো:

```json
{
  "compilerOptions": {
    "outDir": "./dist",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ES2023",
    "types": ["node"],
    "sourceMap": true,
    "declaration": true,
    "declarationMap": true,
    "noUncheckedIndexedAccess": true,
    "strict": true,
    "isolatedModules": true,
    "noUncheckedSideEffectImports": true,
    "moduleDetection": "force",
    "skipLibCheck": true
  },
  "exclude": ["node_modules", "dist"]
}
```

**মনে রাখবে:** ফাইলের নাম হবে `tsconfig.json`, `tesconfing.ts` নয়।

## 4. Enable ES Modules

`package.json`-এর top-level property হিসেবে `"type": "module"` যোগ করো। উদাহরণ:

```json
{
  "name": "my-backend",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js"
  }
}
```

তোমার `package.json`-এ আগে থেকে dependencies থাকলে সেগুলো মুছে ফেলবে না। শুধু `"type": "module"` এবং প্রয়োজনীয় scripts যোগ করবে।

## 5. Prisma initialize

PostgreSQL provider দিয়ে Prisma initialize করো:

```bash
npx prisma init --datasource-provider postgresql
```

এতে সাধারণত `prisma/schema.prisma`, `.env` এবং Prisma config file তৈরি হয়। Prisma 7-এর জন্য নিচের configuration নিশ্চিত করো।

### `prisma/schema.prisma`

```prisma
generator client {
  provider = "prisma-client"
  output   = "../generated/prisma"
}

datasource db {
  provider = "postgresql"
}

model User {
  id        String   @id @default(uuid())
  name      String
  email     String   @unique
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

এখানে একটি sample `User` model দেওয়া হয়েছে। নিজের project অনুযায়ী model পরিবর্তন বা নতুন model যোগ করতে পারো।

### `prisma.config.ts`

Project root-এ `prisma.config.ts` ফাইলটি রাখো:

```ts
import "dotenv/config";
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
  },
  datasource: {
    url: env("DATABASE_URL"),
  },
});
```

Prisma 7-এ database connection URL এখানে configure করা হয়।

## 6. Environment variables

Project root-এ `.env` ফাইল তৈরি করো:

```env
PORT=5000

DATABASE_URL="postgresql://USER:PASSWORD@HOST:5432/DATABASE_NAME?schema=public"

APP_URL="http://localhost:3000"

BCRYPT_SALT_ROUNDS=12

JWT_ACCESS_SECRET="replace_with_a_long_random_access_secret"
JWT_REFRESH_SECRET="replace_with_a_long_random_refresh_secret"

JWT_ACCESS_EXPIRES_IN="15m"
JWT_REFRESH_EXPIRES_IN="7d"
```

`DATABASE_URL`-এ তোমার PostgreSQL provider-এর দেওয়া connection string বসাবে। `USER`, `PASSWORD`, `HOST` এবং `DATABASE_NAME` নিজের database অনুযায়ী পরিবর্তন করবে।

**Security:** `.env` কখনো GitHub-এ push করবে না। `.gitignore`-এ অন্তত নিচের entry রাখো:

```gitignore
node_modules
dist
.env
```

## 7. Prisma migration এবং client generate

প্রথমে schema অনুযায়ী migration তৈরি করো:

```bash
npx prisma migrate dev --name init
```

তারপর Prisma Client generate করো:

```bash
npx prisma generate
```

Schema পরিবর্তন করলে নতুন migration তৈরি করতে পারো:

```bash
npx prisma migrate dev --name describe_your_change
npx prisma generate
```

## 8. Prisma client setup

`src/lib/prisma.ts` ফাইল তৈরি করো:

```ts
import "dotenv/config";
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "../../generated/prisma/client";

const connectionString = process.env.DATABASE_URL;

if (!connectionString) {
  throw new Error("DATABASE_URL is not defined in the environment variables.");
}

const adapter = new PrismaPg({ connectionString });

const prisma = new PrismaClient({ adapter });

export { prisma };
```

এই উদাহরণে `prisma.ts` ফাইলটি `src/lib/`-এর ভেতরে এবং generated client project root-এর `generated/prisma/`-এ আছে। তাই import path `../../generated/prisma/client`।

তোমার folder structure আলাদা হলে import path-ও সেই অনুযায়ী ঠিক করবে।

## 9. Install Express and middleware packages

```bash
npm install express http-status-codes cors bcrypt cookie-parser jsonwebtoken
npm install -D @types/express @types/cors @types/bcrypt @types/cookie-parser @types/jsonwebtoken
```

Package-গুলোর কাজ:

| Package | কাজ |
|---|---|
| `express` | Backend server ও API routes |
| `http-status-codes` | HTTP status code constants |
| `cors` | Frontend থেকে cross-origin request allow করা |
| `bcrypt` | Password hash করা |
| `cookie-parser` | Request cookie read করা |
| `jsonwebtoken` | JWT access/refresh token তৈরি ও verify করা |

## 10. Config setup

`src/config/index.ts` ফাইল তৈরি করো:

```ts
import dotenv from "dotenv";
import path from "path";

dotenv.config({
  path: path.join(process.cwd(), ".env"),
});

const config = {
  port: Number(process.env.PORT) || 5000,
  database_url: process.env.DATABASE_URL,
  app_url: process.env.APP_URL,
  bcrypt_salt_rounds: Number(process.env.BCRYPT_SALT_ROUNDS) || 12,
  jwt_access_secret: process.env.JWT_ACCESS_SECRET,
  jwt_refresh_secret: process.env.JWT_REFRESH_SECRET,
  jwt_access_expires_in: process.env.JWT_ACCESS_EXPIRES_IN,
  jwt_refresh_expires_in: process.env.JWT_REFRESH_EXPIRES_IN,
};

export default config;
```

## 11. Express app setup

`src/app.ts` ফাইল তৈরি করো:

```ts
import cookieParser from "cookie-parser";
import cors from "cors";
import express, { type Application, type Request, type Response } from "express";
import config from "./config/index.js";

const app: Application = express();

app.use(
  cors({
    origin: config.app_url,
    credentials: true,
  }),
);

app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser());

app.get("/", (req: Request, res: Response) => {
  res.send("Hello, World!");
});

export default app;
```

`origin` হিসেবে `.env`-এ দেওয়া frontend URL ব্যবহার হবে। Cookie-based authentication ব্যবহার করলে frontend request-এ credentials পাঠানোর configuration-ও লাগবে।

## 12. Server setup

`src/server.ts` ফাইল তৈরি করো:

```ts
import "dotenv/config";
import app from "./app.js";
import config from "./config/index.js";
import { prisma } from "./lib/prisma.js";

const PORT = config.port;

async function main() {
  try {
    await prisma.$connect();
    console.log("Connected to the database successfully.");

    app.listen(PORT, () => {
      console.log(`Server is running on port ${PORT}`);
    });
  } catch (error) {
    console.error("Error starting the server:", error);
    await prisma.$disconnect();
    process.exit(1);
  }
}

main();
```

এই project-এ ES Modules এবং TypeScript ব্যবহার করা হয়েছে। তাই NodeNext-style runtime import resolution-এর জন্য relative import-এ `.js` extension ব্যবহার করা হয়েছে। `tsx` TypeScript source চালানোর সময় এগুলো resolve করতে পারে।

## 13. Recommended folder structure

```text
project-root/
│
├── generated/
│   └── prisma/
│
├── prisma/
│   ├── migrations/
│   └── schema.prisma
│
├── src/
│   ├── config/
│   │   └── index.ts
│   ├── lib/
│   │   └── prisma.ts
│   ├── app.ts
│   └── server.ts
│
├── .env
├── .gitignore
├── package.json
├── prisma.config.ts
└── tsconfig.json
```

## 14. Run the project

Development mode-এ server চালাও:

```bash
npm run dev
```

সব ঠিক থাকলে terminal-এ এমন output দেখতে পাবে:

```text
Connected to the database successfully.
Server is running on port 5000
```

Browser বা Postman-এ নিচের URL request করো:

```text
http://localhost:5000/
```

Response:

```text
Hello, World!
```

Production build করতে:

```bash
npm run build
npm start
```

## 15. Common commands

| Command | কাজ |
|---|---|
| `npm run dev` | Development server চালু |
| `npm run build` | TypeScript compile করে `dist/` তৈরি |
| `npm start` | Compiled server চালু |
| `npx prisma` | Prisma CLI help |
| `npx prisma migrate dev --name init` | Migration তৈরি ও apply |
| `npx prisma generate` | Prisma Client generate |
| `npx prisma studio` | Database-এর data GUI-তে দেখা |
| `npx prisma migrate status` | Migration status দেখা |

## Troubleshooting

### 1. `DATABASE_URL is not defined`

- Project root-এ `.env` আছে কি না দেখো।
- `DATABASE_URL` variable-এর নাম ঠিক আছে কি না দেখো।
- Connection string সঠিক কি না যাচাই করো।

### 2. Prisma Client import error

- `schema.prisma`-এ generator-এর `output` path ঠিক আছে কি না দেখো।
- `npx prisma generate` চালাও।
- `src/lib/prisma.ts`-এর import path folder structure অনুযায়ী ঠিক করো।

### 3. Database connection error

- PostgreSQL server চালু আছে কি না দেখো।
- Database, username, password, host ও port যাচাই করো।
- Hosted database হলে প্রয়োজনীয় network access ও SSL settings provider-এর নির্দেশনা অনুযায়ী configure করো।

### 4. `npm run dev` command not found

`package.json`-এর `scripts` section-এ এটি আছে কি না নিশ্চিত করো:

```json
{
  "scripts": {
    "dev": "tsx watch src/server.ts"
  }
}
```

---

**Setup complete!** এখন Express routes, controllers, services, authentication এবং অন্য backend features তৈরি করা শুরু করতে পারো।
