# pnpm-prisma-postgres-setup-guide

**The annotated Prisma Postgres + pnpm quickstart — with every error fixed inline.**

Follows the [official Prisma Postgres quickstart](https://www.prisma.io/docs/prisma-orm/quickstart/prisma-postgres) step by step, but flags the exact points where the official commands break with pnpm — and how to fix them. If you searched for any of these, you're in the right place:

`pnpm dlx tsc --init` not working · `ERR_PNPM_IGNORED_BUILDS` · Prisma `P1001 Can't reach database server` · pnpm `command not found "This"` · `exactOptionalPropertyTypes` Prisma config error · TypeScript `rootDir` Prisma error · `baseUrl is deprecated` TypeScript

---

## Table of Contents

- [0. Why pnpm behaves differently from npm/bun here](#0-why-pnpm-behaves-differently-from-npmbun-here)
- [1. Create a new project](#1-create-a-new-project)
- [2. Install required dependencies](#2-install-required-dependencies)
- [3. Configure TypeScript (god-tier config)](#3-configure-typescript-god-tier-config)
- [4. Initialize Prisma ORM](#4-initialize-prisma-orm)
- [5. Define your data model](#5-define-your-data-model)
- [6. Create and apply your first migration](#6-create-and-apply-your-first-migration)
- [7. Instantiate Prisma Client](#7-instantiate-prisma-client)
- [8. Write your first query](#8-write-your-first-query)
- [9. Explore your data with Prisma Studio](#9-explore-your-data-with-prisma-studio)
- [Troubleshooting Index](#troubleshooting-index)
- [pnpm Self-Repair](#pnpm-self-repair-when-pnpm-itself-breaks)
- [The Golden Rule](#the-golden-rule)

---

## 0. Why pnpm behaves differently from npm/bun here

npm and bun run a package's `postinstall` script automatically. pnpm (since v10+) **blocks build scripts by default** — this is a deliberate supply-chain security decision, not a bug. Several errors below exist *because* of this. You'll approve trusted packages once per project with `pnpm approve-builds`, and that's it.

---

## 1. Create a new project

```bash
mkdir hello-prisma
cd hello-prisma
pnpm init
pnpm add typescript tsx @types/node --save-dev
```

> ### ⚠️ TRAP: `pnpm dlx tsc --init` — this is in the official docs and it WILL fail
> The official quickstart tells you to run `pnpm dlx tsc --init`. **Don't.** `dlx` downloads and runs a package fresh from the npm registry — and there is a **fake placeholder package literally named `tsc`** on npm that is NOT the TypeScript compiler. Running it gives you:
> ```
> This is not the tsc command you are looking for
> ```
> You already installed real TypeScript as a devDependency above. Use the local binary instead:
> ```bash
> pnpm exec tsc --init
> ```
> **Rule of thumb:** `dlx` = run a tool you don't have installed. `exec` = run a tool from your own `node_modules/.bin`. Once something is a devDependency, always use `exec`.

> ### ⚠️ TRAP: `ERR_PNPM_IGNORED_BUILDS`
> Right after install you may see:
> ```
> [ERR_PNPM_IGNORED_BUILDS] Ignored build scripts: esbuild@x.x.x
> Run "pnpm approve-builds" to pick which dependencies should be allowed to run scripts.
> ```
> This blocks `pnpm exec tsc --init` from completing. Fix:
> ```bash
> pnpm approve-builds
> ```
> Select `esbuild` with `Space`, hit `Enter`, then when asked **"Do you approve?" → choose Yes.** (Choosing "No" by accident silently skips the postinstall script and causes confusing failures later — there's no warning when you do this.)

---

## 2. Install required dependencies

```bash
pnpm add prisma @types/pg --save-dev
pnpm add @prisma/client @prisma/adapter-pg pg dotenv
```

| Package | What it does |
|---|---|
| `prisma` | CLI for `prisma init`, `prisma migrate`, `prisma generate` |
| `@prisma/client` | Query library |
| `@prisma/adapter-pg` | node-postgres driver adapter |
| `pg` | node-postgres database driver |
| `@types/pg` | TypeScript types for `pg` |
| `dotenv` | Loads `.env` variables |

> ### ⚠️ Same `ERR_PNPM_IGNORED_BUILDS` will appear again
> This time for `prisma` and `@prisma/engines`. Run `pnpm approve-builds` again, select both, approve **Yes**.

---

## 3. Configure TypeScript (god-tier config)

The official docs give you a minimal `tsconfig.json`. Here's a stricter, production-ready version with every known conflict already resolved:

```jsonc
{
  "compilerOptions": {
    /* Build output */
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "incremental": true,
    "tsBuildInfoFile": "./dist/.tsbuildinfo",

    /* Module system */
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2023",
    "lib": ["ES2023"],
    "moduleDetection": "force",
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "noUncheckedSideEffectImports": true,

    /* Strictness */
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitOverride": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noEmitOnError": true,

    /* Sanity */
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "types": ["node"],

    /* Optional path alias — note the leading "./" or it errors */
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "generated/prisma"]
}
```

Update `package.json` to enable ESM:
```json
{
  "type": "module"
}
```

> ### ⚠️ TRAP: `module: "ESNext"` + `moduleResolution: "bundler"` (the official default) breaks plain `node dist/index.js`
> The docs' default config assumes a bundler (esbuild/vite/webpack) will process your output, so it doesn't require `.js` extensions on relative imports. If you instead compile with `tsc` and run the output directly with plain Node, Node's ESM loader **requires** the extension and will crash at runtime with no compile-time warning.
> **Fix:** use `"module": "NodeNext"` and `"moduleResolution": "NodeNext"` as above. Tradeoff: you must now write extensions on relative imports yourself:
> ```ts
> import { db } from './db.js'   // not './db'
> ```
> If you're certain you'll only ever run via `tsx` (dev) and never plain `node dist/`, you can keep `"bundler"` instead — both are valid, just pick based on how you deploy.

> ### ⚠️ TRAP: `Option 'baseUrl' is deprecated`
> If you add `baseUrl` alongside `paths`, modern TypeScript warns it'll be removed in TS 7.0. Since TS 4.1+, `paths` works without `baseUrl`. **Don't add `baseUrl` at all** — just use `paths` with a leading `./` in the path value as shown above.

> ### ⚠️ TRAP: `exactOptionalPropertyTypes` breaks Prisma's own generated `prisma.config.ts`
> If you add the very strict `"exactOptionalPropertyTypes": true`, you'll get:
> ```
> Type '{ url: string | undefined; }' is not assignable to type '{ url?: string; ... }'
> with 'exactOptionalPropertyTypes: true'
> ```
> This setting distinguishes "key missing" from "key present but undefined" — and Prisma's generated config (using `process.env["DATABASE_URL"]`, typed `string | undefined`) isn't written to satisfy it. **Leave this setting off.** It's not worth fighting Prisma's own generated files over.

> ### ⚠️ TRAP: `No inputs were found in config file` / `File is not under 'rootDir'`
> If `rootDir` is set to `./src` but you haven't created a `src/` folder yet, or if `prisma.config.ts` (which lives at project root, outside `src/`) gets swept into the program by a missing `include`, you'll see rootDir conflicts. **Fix:** always keep `"include": ["src/**/*"]` set (don't comment it out), and make sure `src/` actually exists with at least one file in it.

> ### ⚠️ TRAP: `Cannot find name 'process'` in `prisma.config.ts`
> Since `prisma.config.ts` sits outside `src/` (outside your `rootDir`), it's outside this tsconfig's governance and your editor may type-check it standalone without Node globals. Fix by adding this as the very first line of `prisma.config.ts`:
> ```ts
> /// <reference types="node" />
> ```
> If the error persists after saving, restart the TS server (`Ctrl+Shift+P` → "TypeScript: Restart TS Server" in VS Code) — it's often a stale cache, not a real config problem.

---

## 4. Initialize Prisma ORM

```bash
pnpm exec prisma init --output ../generated/prisma
```

> Use `exec`, not the docs' `dlx` — same reasoning as Step 1. `prisma` is already a devDependency.

This creates:
- `prisma/schema.prisma` — your schema
- `.env` — environment variables, including `DATABASE_URL`
- `prisma.config.ts` — Prisma's config file (see Step 3's fixes above)

Create your actual Prisma Postgres database and copy the real `postgres://...` connection string into `.env`, replacing the placeholder:

```bash
pnpm exec create-db
```

---

## 5. Define your data model

Edit `prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client"
  output   = "../generated/prisma"
}

datasource db {
  provider = "postgresql"
}

model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
  posts Post[]
}

model Post {
  id        Int     @id @default(autoincrement())
  title     String
  content   String?
  published Boolean @default(false)
  author    User    @relation(fields: [authorId], references: [id])
  authorId  Int
}
```

---

## 6. Create and apply your first migration

```bash
pnpm exec prisma migrate dev --name init
pnpm exec prisma generate
```

> ### ⚠️ TRAP: `P1001 Can't reach database server`
> ```
> Error: P1001
> Can't reach database server at `localhost:XXXXX`
> ```
> Almost always a **port mismatch** between what your config says and what's actually running. Debug checklist:
> ```bash
> cat .env | grep DATABASE_URL        # what port does config claim?
> cat prisma.config.ts                # does this override .env?
> docker ps                           # what port is Postgres ACTUALLY on? (check PORTS column)
> sudo ss -tlnp | grep postgres       # if running natively, not in Docker
> ```
> Common causes: `.env` and `.env.local` both define `DATABASE_URL` with different ports; `prisma.config.ts` hardcodes a different string than `.env`; a Docker container restarted and got reassigned a new random port; a stale connection string copy-pasted from a different project. There should be exactly **one** source of truth for `DATABASE_URL`.

---

## 7. Instantiate Prisma Client

```typescript
// lib/prisma.ts
import "dotenv/config";
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "../generated/prisma/client";

const connectionString = `${process.env.DATABASE_URL}`;

const adapter = new PrismaPg({ connectionString });
const prisma = new PrismaClient({ adapter });

export { prisma };
```

> If you're querying from an edge runtime (Cloudflare Workers, Vercel Edge Functions), use the [Prisma Postgres serverless driver](https://www.prisma.io/docs/postgres/database/serverless-driver#use-with-prisma-orm) instead of `pg`.

---

## 8. Write your first query

```typescript
// script.ts
import { prisma } from "./lib/prisma";

async function main() {
  const user = await prisma.user.create({
    data: {
      name: "Alice",
      email: "alice@prisma.io",
      posts: { create: { title: "Hello World", content: "First post!", published: true } },
    },
    include: { posts: true },
  });
  console.log("Created user:", user);

  const allUsers = await prisma.user.findMany({ include: { posts: true } });
  console.log("All users:", JSON.stringify(allUsers, null, 2));
}

main()
  .then(async () => { await prisma.$disconnect(); })
  .catch(async (e) => {
    console.error(e);
    await prisma.$disconnect();
    process.exit(1);
  });
```

Run it:
```bash
pnpm exec tsx script.ts
```

---

## 9. Explore your data with Prisma Studio

```bash
pnpm exec prisma studio
```

---

## Troubleshooting Index

Quick-reference table of every error covered above, for Ctrl+F / Google search:

| Error message | Section |
|---|---|
| `This is not the tsc command you are looking for` | [Step 1](#1-create-a-new-project) |
| `ERR_PNPM_IGNORED_BUILDS` | [Step 1](#1-create-a-new-project) / [Step 2](#2-install-required-dependencies) |
| `Option 'baseUrl' is deprecated` | [Step 3](#3-configure-typescript-god-tier-config) |
| `exactOptionalPropertyTypes` ... `not assignable` (Prisma config) | [Step 3](#3-configure-typescript-god-tier-config) |
| `No inputs were found in config file` | [Step 3](#3-configure-typescript-god-tier-config) |
| `File is not under 'rootDir'` | [Step 3](#3-configure-typescript-god-tier-config) |
| `Cannot find name 'process'` in `prisma.config.ts` | [Step 3](#3-configure-typescript-god-tier-config) |
| `Error: P1001 Can't reach database server` | [Step 6](#6-create-and-apply-your-first-migration) |
| `pnpm: line 1: This: command not found` | [pnpm Self-Repair](#pnpm-self-repair-when-pnpm-itself-breaks) |

---

## pnpm Self-Repair (when pnpm itself breaks)

If you see:
```
.../node_modules/@pnpm/exe/pnpm: line 1: This: command not found
```
The pnpm binary itself is corrupted — usually from accidentally piping command output into it. Don't patch it, reinstall:

```bash
rm -rf ~/.local/share/pnpm
curl -fsSL https://get.pnpm.io/install.sh | sh -
```

Open a **fresh terminal window** (not just a new tab — shells like fish cache PATH), then verify:
```bash
which pnpm
pnpm -v
```

If `which pnpm` still shows the broken path, check for stale shell config entries (example for fish):
```bash
cat ~/.config/fish/config.fish | grep -i pnpm
```

> **Note:** pnpm itself installs to one global location (`$PNPM_HOME`, typically `~/.local/share/pnpm`) shared across all projects. Only `node_modules/` and `pnpm-lock.yaml` are per-project.

---

## The Golden Rule

> **`dlx` is for tools you don't have installed. `exec` is for tools you DO have installed.**
> Every trap in this guide came from mixing these two up — including the ones copy-pasted straight from Prisma's own official docs.

---

## Reference

- [Official Prisma Postgres Quickstart](https://www.prisma.io/docs/prisma-orm/quickstart/prisma-postgres)
- [Prisma Config Reference](https://www.prisma.io/docs/orm/reference/prisma-config-reference)
- [pnpm `approve-builds` docs](https://pnpm.io/cli/approve-builds)

## License

MIT — use it, fork it, fix your own Prisma + pnpm disasters with it.
