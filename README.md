# pnpm-prisma-survival-kit

Fixes for: `pnpm dlx tsc --init` not working · `ERR_PNPM_IGNORED_BUILDS` ·
Prisma `P1001 Can't reach database server` · corrupted pnpm binary
(`line 1: This: command not found`)

A battle-tested cheatsheet for setting up **pnpm + Prisma + PostgreSQL** without falling into the same traps everyone falls into. Every "DO NOT" here is from a real error that actually broke a real project.

---

## Table of Contents

- [Project Init](#project-init-clean-no-traps)
- [Prisma Setup](#prisma-setup)
- [Schema → Database Workflow](#schema--database-workflow)
- [P1001 Debug Checklist](#p1001-cant-reach-database-server--debug-checklist)
- [Native PostgreSQL Quick Commands](#native-postgresql-cachyos--arch-based-quick-commands)
- [pnpm Self-Repair](#pnpm-self-repair-when-pnpm-itself-breaks)
- [Daily Driver Commands](#daily-driver-commands)
- [The Golden Rule](#the-golden-rule)

---

## Project Init (clean, no traps)

```bash
pnpm init
pnpm add typescript tsx @types/node --save-dev
```

> ❌ **DO NOT:** `pnpm dlx tsc --init`
> `dlx` pulls a **fake placeholder** `tsc` package from the npm registry. It is NOT the TypeScript compiler. It just prints a warning and does nothing.

```bash
# ✅ CORRECT — runs the LOCAL tsc you just installed
pnpm exec tsc --init
```

After this, `pnpm approve-builds` will almost always demand attention:

```bash
pnpm approve-builds
```
- `Space` = select package, `A` = toggle all, `Enter` = confirm
- Select: `esbuild` (and later: `prisma`, `@prisma/engines`)
- ⚠️ When asked **"Do you approve?"** → select **Yes**, not No. Saying No silently skips the postinstall script and breaks the tool later with zero explanation.

**Rule of thumb:** `dlx` = download & run from registry (use only for *one-off* CLI tools you don't have installed). `exec` = run from your own `node_modules/.bin`. For anything already in your `devDependencies`, always use `exec`.

---

## Prisma Setup

```bash
pnpm add prisma @types/pg --save-dev
pnpm add @prisma/client @prisma/adapter-pg pg dotenv

pnpm approve-builds
# Select: prisma, @prisma/engines, esbuild → Yes

pnpm exec prisma init
# generates: prisma/schema.prisma, .env (with DATABASE_URL placeholder)
```

Verify install:
```bash
pnpm exec prisma -v
```

---

## Schema → Database Workflow

```bash
# Edit prisma/schema.prisma first, then:

pnpm exec prisma migrate dev --name init   # create + apply migration, generate client
pnpm exec prisma generate                  # regenerate client only (no DB change)
pnpm exec prisma db push                   # push schema directly (prototyping only)
pnpm exec prisma studio                    # GUI to browse/edit data
pnpm exec prisma migrate reset             # ⚠️ wipes DB, use when migrations get messy
pnpm exec prisma migrate status            # see pending/applied migrations
pnpm exec prisma migrate deploy            # production — applies existing migrations only
pnpm exec prisma format                    # auto-align schema.prisma fields
pnpm exec prisma validate                  # validate schema without touching DB
```

> ❌ **DO NOT** use `pnpm dlx prisma ...` once Prisma is a project dependency. Same trap as `tsc` — always `pnpm exec prisma ...`.

---

## P1001 `Can't reach database server` — Debug Checklist

This usually means a **port mismatch** between your config and your actual running DB.

```bash
# 1. What does your .env actually say?
cat .env | grep DATABASE_URL

# 2. What does prisma.config.ts override (if it exists)?
cat prisma.config.ts

# 3. What port is Postgres ACTUALLY listening on?
docker ps                          # Docker — check the PORTS column
sudo ss -tlnp | grep postgres      # Native systemd Postgres
```

**Common causes:**
- `.env` and `.env.local` both define `DATABASE_URL` with different ports
- `prisma.config.ts` hardcodes a different connection string than `.env`
- Docker container restarted and got assigned a new random port
- Stale `DATABASE_URL` copy-pasted from a different project

**Fix:** make the port in `.env` match what `docker ps` (or your native Postgres config) actually shows. There should be exactly **one** source of truth for `DATABASE_URL`.

---

## Native PostgreSQL (CachyOS / Arch-based) Quick Commands

```bash
# Service control
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo systemctl status postgresql

# psql
sudo -iu postgres psql
```

```sql
\l              -- list databases
\c dbname       -- connect to a database
\dt             -- list tables
\d tablename    -- describe table
\du             -- list users/roles
\q              -- quit
```

```bash
sudo -iu postgres createdb mydb
sudo -iu postgres createuser --interactive
```

---

## pnpm Self-Repair (when pnpm itself breaks)

If you see:
```
.../node_modules/@pnpm/exe/pnpm: line 1: This: command not found
```
The pnpm binary itself is corrupted. Don't patch it — reinstall:

```bash
rm -rf ~/.local/share/pnpm
curl -fsSL https://get.pnpm.io/install.sh | sh -

# Open a FRESH terminal window (fish caches PATH)
which pnpm
pnpm -v
```

If `which pnpm` still shows the broken path, check for stale fish config entries:
```bash
cat ~/.config/fish/config.fish | grep -i pnpm
```

> **Note:** `pnpm` itself always installs to one global location (`$PNPM_HOME`, typically `~/.local/share/pnpm`) shared across all projects. Only `node_modules/` and `pnpm-lock.yaml` are per-project.

---

## Daily Driver Commands

| Command | What it does |
|---|---|
| `pnpm install` | install deps from lockfile |
| `pnpm add <pkg>` | add dependency |
| `pnpm add -D <pkg>` | add dev dependency |
| `pnpm remove <pkg>` | uninstall |
| `pnpm run dev` | run dev script |
| `pnpm start` | run start script |
| `pnpm run build` | run build script |
| `pnpm exec <cmd>` | run a binary from `node_modules/.bin` |
| `pnpm dlx <pkg>` | download & run a package once, not installed |
| `pnpm approve-builds` | review/allow postinstall scripts |
| `pnpm store prune` | clean unused packages from global store |
| `pnpm outdated` | check for outdated deps |
| `pnpm update` | update deps per semver range |

---

## The Golden Rule

> **`dlx` is for tools you don't have installed. `exec` is for tools you DO have installed.**
> Every error in the "DO NOT" sections above came from mixing these two up.

---

## License

MIT — use it, fork it, fix your own pnpm disasters with it.
