# nextjs16-prisma-mysql-betterauth

A **thin integration skill** that scaffolds a complete **Next.js 16 + Prisma 7 + MySQL/MariaDB +
Better Auth** stack — Google OAuth + email/password — and captures the gotchas that bite when you
combine these on Next.js 16. Distilled from a verified, working example.

It is an **integration layer**: it handles *assembly* and the *Next.js-16-specific gotchas*, and
delegates per-library depth to two companion skills (see [Install](#install)). Don't expect this
file to duplicate Better Auth or Prisma reference material — that lives in the companions.

## What it produces

Google OAuth (social) + email/password, database sessions via the Prisma adapter (Prisma 7 +
`mariadb` driver), sign-in / sign-up pages, a dashboard with avatar, and server-side session checks.

## When to use

- A new **Next.js 16** app that needs Better Auth (Google + email/password) backed by
  **MySQL/MariaDB** via **Prisma 7**.

**Not for:** Auth.js / next-auth, MongoDB, or direct-driver MySQL — use the `better-auth-best-practices`
skill directly instead.

## Install

This skill has **two required companion skills**. Install all three:

```bash
# 1. This skill (the integration layer)
npx skills@latest add <your-github-user>/nextjs16-prisma-mysql-betterauth

# 2. Required companions — provide the Better Auth depth this skill delegates to
npx skills@latest add better-auth/skills
```

`better-auth/skills` provides both `better-auth-best-practices` and `create-auth-skill`. Without
step 2, this skill's "for depth, use these" references have nothing to point at.

### Verified companion revision

This skill was validated against these revisions of the companions (content hashes, as recorded by
the `skills` CLI). If reproducibility matters, pin them in your project's `skills-lock.json`.

| Companion | Source | Verified hash |
|-----------|--------|---------------|
| `better-auth-best-practices` | `better-auth/skills` → `better-auth/best-practices/SKILL.md` | `9ab075b5061be2a5f299c10505667345cc1ec7648de4120901cfd586643e776f` |
| `create-auth-skill` | `better-auth/skills` → `better-auth/create-auth/SKILL.md` | `393e8d2d795fa5797c9a4e0665f29183ea2e140233db59746d510340a10456e6` |

## The value: the gotchas

The real content of this skill is a distilled list of integration gotchas (full list in `SKILL.md`),
e.g.:

- The Prisma adapter takes the instance — `prismaAdapter(prisma, { provider: "mysql" })`.
- The schema is **CLI-generated** (`@better-auth/cli generate`), which auto-emits MySQL-safe
  `@db.Text` and `@@index([…(length: 191)])` — no manual index-length fix (unlike Auth.js).
- `nextCookies()` plugin is **required** for `auth.api.getSession` in server components.
- Next.js 16 `headers()` is async — `await headers()`.
- Route handler is `[...all]` with `toNextJsHandler(auth)` — **not** `[...nextauth]`.
- Env vars are `BETTER_AUTH_*` / `GOOGLE_CLIENT_*` — different from Auth.js (`AUTH_*`).

## Prerequisites (for the app this skill generates)

- Node 20.19+, TypeScript 5.4+
- A reachable MySQL/MariaDB server (Prisma creates the database on `db push`)
- A Google OAuth client; redirect URI `http://localhost:3000/api/auth/callback/google`
- The `@/*` path alias configured in `tsconfig.json` (templates use `@/lib/auth`, `@/generated/client`, …)

## Repo layout

```
nextjs16-prisma-mysql-betterauth/
├── SKILL.md              # the skill — frontmatter, ordered steps, gotchas
├── references/
│   └── templates.md      # verified file templates (copy into a fresh app)
└── README.md             # this file
```

## Validation

Validated via a clean-room scaffold — following the skill from scratch in a fresh project yields a
working app. The acceptance checks that were run are preserved in the git history.

## License

No license is specified yet. Add a `LICENSE` (e.g. MIT) before public release so consumers know their
rights; without one, default copyright reserves all rights.
