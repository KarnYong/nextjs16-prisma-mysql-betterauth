---
name: nextjs16-prisma7-mysql-betterauth1
description: Scaffold a complete Next.js 16 + Prisma 7 + MySQL/MariaDB + Better Auth (Google OAuth + email/password) stack. Use when adding Better Auth authentication to a Next.js 16 app using Prisma 7 and MySQL/MariaDB. This is the INTEGRATION layer — it orchestrates the `better-auth-best-practices` and `create-auth-skill` skills and captures the gotchas that bite when combining them on Next.js 16.
---

# Next.js 16 + Prisma 7 + MySQL/MariaDB + Better Auth

A thin integration skill. Per-library depth lives in the existing `better-auth-best-practices`
and `create-auth-skill` skills — this one handles **assembly** and the **gotchas unique to
combining them on Next.js 16**. Distilled from a verified working example (Google OAuth +
email/password → Better Auth → Prisma adapter → mariadb driver → MariaDB).

## When to apply

- New Next.js 16 app needing Better Auth (Google + email/password) backed by MySQL/MariaDB via Prisma 7.
- **Not for:** Auth.js/next-auth, MongoDB, or direct-driver MySQL — use `better-auth-best-practices` directly.

## Features

Google OAuth (social) + email/password, database sessions via the Prisma adapter (Prisma 7 +
mariadb driver), sign-in/sign-up pages, a dashboard with avatar, and server-side session checks.

## Prerequisites

- Node 20.19+, TypeScript 5.4+
- A reachable MySQL/MariaDB server. Prisma **creates the database** on `db push`.
- A Google OAuth client; redirect URI `http://localhost:3000/api/auth/callback/google` (the path Better Auth uses — same as Auth.js, handy for migrations).

## Ordered steps

1. **Install** — `npm install better-auth` (plus the Prisma 7 stack if not present; see `references/templates.md`).
2. **Server + client** — `lib/auth.ts` (server) and `lib/auth-client.ts` (React client).
3. **Route handler** — `app/api/auth/[...all]/route.ts` with `toNextJsHandler(auth)`.
4. **Generate schema** — `npx @better-auth/cli generate --output prisma/schema.prisma`.
5. **Apply** — `npx prisma generate && npx prisma db push`.
6. **Pages** — sign-in, sign-up, dashboard, home, `sign-out-button`.
7. **Env** — `.env` (`DATABASE_URL`), `.env.local` (`BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`).

## The gotchas (the real value)

1. **The Prisma adapter takes the instance** — `prismaAdapter(prisma, { provider: "mysql" })` from
   `better-auth/adapters/prisma`. It works with Prisma 7's generated client + the mariadb driver
   (`lib/prisma.ts` is unchanged from a plain Prisma 7 setup).

2. **The schema is CLI-generated, not hand-written** — `npx @better-auth/cli generate`. The CLI even
   emits MySQL-safe `@db.Text` and `@@index([col(length: 191)])` automatically (no manual
   index-length fix needed, unlike Auth.js). Re-run after adding/changing plugins.

3. **`nextCookies()` plugin is required** for `auth.api.getSession` to work inside Next.js server
   components. Add it in `plugins: [nextCookies()]`.

4. **Server/client split** — `lib/auth.ts` (server: `auth.api.getSession`, `prismaAdapter`) +
   `lib/auth-client.ts` (client: `signIn`/`signUp`/`signOut`/`useSession` from `better-auth/react`).
   Sign-in/sign-up are **client-side** calls; session checks are **server-side**.

5. **Next.js 16 `headers()` is async** — `auth.api.getSession({ headers: await headers() })`.

6. **Env var names** — `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, and `GOOGLE_CLIENT_ID` /
   `GOOGLE_CLIENT_SECRET` (Better Auth reads the `GOOGLE_*` env vars for the google provider).
   Different from Auth.js (`AUTH_*`).

7. **Route handler is `[...all]`** with `toNextJsHandler(auth)` from `better-auth/next-js` —
   NOT `[...nextauth]`.

8. **Middleware is now viable** (opposite of Auth.js) — Better Auth's `getSessionCookie` is
   edge-safe (no DB call). This example uses page-level guards for simplicity, but edge middleware
   is a valid, recommended option for protecting many routes cheaply.

9. **Google redirect URI is `/api/auth/callback/google`** — identical to Auth.js, so migrating
   requires no Google Console change (only the env var names change).

10. **Prisma 7 `env()` still doesn't auto-load `.env`** — keep `import "dotenv/config"` at the top
    of `prisma.config.ts` (this is a Prisma 7 gotcha, unchanged by the auth choice).

11. **Email/password** — `emailAndPassword: { enabled: true }`; `signUp.email({ email, password, name })`
    auto-signs-in by default. Passwords are hashed by Better Auth (stored on the `Account.password` field).

## Account picker (optional)

`prompt: "select_account"` on the google provider config forces Google's account-selection screen on every sign-in, even when the user is logged into a single account. (The google provider forwards `options.prompt` to the auth URL — inherited from `ProviderOptions`.)

## Existing skills (for depth, don't duplicate)

- `better-auth-best-practices` — config options, sessions, plugins, security, hooks.
- `create-auth-skill` — scaffolding decisions, plugin table, migration guidance, UI patterns.
