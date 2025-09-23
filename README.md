## Quick Ticket

Quick Ticket is a minimal ticketing app built with Next.js (App Router) and Prisma. Users can register, log in, create tickets, view their ticket list, and close tickets. The project focuses on a clean server-first architecture using server actions, with observability via Sentry.

Under the hood, it uses PostgreSQL (Neon-friendly), Prisma as the ORM, jose for JWT auth, bcryptjs for password hashing, and Tailwind CSS for styling. Sentry is integrated for error reporting and performance tracing, including both server and edge runtimes.

The codebase is TypeScript-first and includes strict linting and a small utilities layer for logging and auth ergonomics.

## Features

- User registration and login (JWT-based)
- Server actions for mutations and data fetching
- Ticket CRUD basics: create, list, view by id, close
- Prisma schema for User and Ticket models
- Sentry instrumentation and breadcrumb logging
- Tailwind CSS v4 with PostCSS

## Project Layout (high level)

- `app/` — App Router pages and routes (e.g., `/tickets`, `/tickets/new`, `/tickets/[id]`)
- `src/actions/` — Server actions (auth, tickets)
- `src/db/prisma.ts` — Prisma client singleton
- `src/lib/` — Auth utils (`auth.ts`), current user helper
- `src/utils/sentry.ts` — Sentry logging helper
- `prisma/` — Prisma schema and migrations
- `src/generated/prisma/` — Prisma Client (generated)

## Getting Started

### Prerequisites

- Node.js 18+ (recommended) and npm
- A PostgreSQL database URL (Neon or local Postgres)

### Environment Variables

Create a `.env` in the project root with at least:

```
DATABASE_URL="postgres://<user>:<password>@<host>:<port>/<db>?sslmode=require"
AUTH_SECRET="a-strong-random-secret"
SENTRY_AUTH_TOKEN="<optional for uploading sourcemaps in CI>"
```

Notes:
- `AUTH_SECRET` is used by `jose` to sign JWTs. It must be consistent across runs.
- If using Neon, include `sslmode=require` in the connection string.

### Install & Database Setup

1. Install dependencies
2. Generate Prisma Client
3. Apply migrations
4. Start the dev server

```bash
npm install
npx prisma generate
npx prisma migrate dev --name init
npm run dev
```

App will be available at http://localhost:3000.

## Tools and How They Fit Together

### Next.js (App Router)

- Renders UI and defines server actions used in `src/actions/*`.
- Server actions run on the server and can safely access Prisma and secrets.
- Revalidation via `revalidatePath` ensures UI stays in sync after mutations.

### Prisma (ORM)

- Schema: `prisma/schema.prisma` defines `User` and `Ticket` models and a PostgreSQL datasource.
- Client generation outputs to `src/generated/prisma` (custom output path).
- The Prisma client is instantiated once in `src/db/prisma.ts` and reused:
	- Avoids exhausting DB connections during hot reloads.
	- Always import on the server only.

Common commands:

```bash
npx prisma generate              # (re)generate client
npx prisma migrate dev --name <name>  # create/apply migration locally
npx prisma studio                # optional: browse data
```

### Neon (PostgreSQL)

- A serverless Postgres provider that works well with Prisma.
- Use its connection string as `DATABASE_URL`.
- Ensure SSL is enabled (e.g., `sslmode=require`).

### Authentication with jose (JWT)

- `src/lib/auth.ts` provides:
	- `signAuthToken(payload)`: Builds and signs a JWT using HS256 and `AUTH_SECRET`.
	- `verifyAuthToken(token)`: Verifies and decodes the JWT.
	- Cookie helpers: `setAuthCookie`, `getAuthCookie`, `removeAuthCookie` (Next.js `cookies()` API).
- `src/lib/current-user.ts` resolves the currently logged-in user by verifying the cookie and fetching from Prisma.
- JWT contains minimal payload (e.g., `{ userId }`).

### bcryptjs (password hashing)

- Used in `src/actions/auth.actions.ts` to hash passwords on registration and to compare on login.
- `bcrypt.hash(password, 10)` and `bcrypt.compare(plain, hash)`.

### Sentry (observability)

- Configured in `sentry.server.config.ts` and `sentry.edge.config.ts` and wrapped in `next.config.ts` with `withSentryConfig`.
- `src/utils/sentry.ts` offers a `logEvent(message, category, data, level, error?)` helper that:
	- Adds a breadcrumb
	- Captures a message or exception
- You’ll see breadcrumbs for auth flows and ticket mutations in server actions.

### Tailwind CSS v4 + PostCSS

- Enabled via `postcss.config.mjs` and `globals.css`.
- Use utility classes across app components.

### TypeScript, ESLint

- Strict TypeScript config with Next.js plugin.
- ESLint (flat config) with Next core-web-vitals + TypeScript rules.

Alternatively, import the default package `@prisma/client` and remove the custom `output` in `schema.prisma`.

- Server-only Prisma: Never import Prisma client in client components; keep it in server actions, route handlers, or server components only.

- Revalidation: After mutations (e.g., create/close ticket), `revalidatePath('/tickets')` keeps lists fresh.

## Troubleshooting

- “Cannot find module '@generated/prisma'”
	- Ensure `npx prisma generate` has been run and `src/generated/prisma` exists.
	- Add the path alias in `tsconfig.json` as above and restart the TypeScript server.

- “Query engine” error on macOS arm64
	- Delete `node_modules` and the `src/generated/prisma` folder, then reinstall and regenerate.

- JWT errors (AUTH_SECRET)
	- Ensure `AUTH_SECRET` is set in `.env` before running dev server.

- Sentry DSN
	- Replace the DSN with your own in `sentry.*.config.ts` or disable if not needed.

## Scripts

- `npm run dev` — Start Next.js dev server
- `npm run build` — Build for production
- `npm run start` — Start production server
- `npm run lint` — Run ESLint

---

If you plan to deploy (e.g., Vercel), set the required environment variables, including DATABASE_URL and AUTH_SECRET, and consider enabling Sentry sourcemap upload in CI using `SENTRY_AUTH_TOKEN`.
