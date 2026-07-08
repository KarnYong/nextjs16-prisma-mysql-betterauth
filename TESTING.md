# Testing `nextjs16-prisma-mysql-betterauth`

How to verify the skill before publishing. The point of a skill is that **following it from
scratch in a fresh project yields a working app** — so the headline test is a clean-room scaffold,
not just reading the file.

Run from the project root (`D:\tutorial\nextjs16-prisma-auth`).

---

## Test 1 — Static validation (1 min)

Frontmatter + completeness + no stale references.

```bash
# 1a. Required frontmatter present and well-formed (name kebab-case, description non-empty)
grep -nE "^name:|^description:" .claude/skills/nextjs16-prisma-mysql-betterauth/SKILL.md

# 1b. Every file the skill names actually exists in references/templates.md
grep -oE "lib/auth\.ts|lib/auth-client\.ts|app/api/auth/\[\.\.\.all\]/route\.ts|components/sign-out-button" \
  .claude/skills/nextjs16-prisma-mysql-betterauth/references/templates.md

# 1c. No Auth.js code slipped in (expect 'clean'). Matches real usage, not prose mentions.
grep -rIE 'from "next-auth"|NextAuth\(|@auth/prisma-adapter|getServerSession\(|AUTH_GOOGLE_ID=' \
  .claude/skills/nextjs16-prisma-mysql-betterauth/ || echo "clean"
```

**Pass:** name = `nextjs16-prisma-mysql-betterauth`, a one-line description, all named files present, and the last grep prints `clean`.

---

## Test 2 — Discovery via the `skills` CLI (1 min)

Confirms the CLI (the thing consumers use) can find and read the skill.

```bash
npx skills add ./.claude/skills/nextjs16-prisma-mysql-betterauth --list
```

**Pass:** it lists `nextjs16-prisma-mysql-betterauth` with its description, no parse errors.

---

## Test 3 — Clean-room scaffold (the real test, ~15–20 min)

Does the skill reproduce a working stack in a brand-new project? This is what catches gaps a
hand-built example can hide.

### Prerequisites (reuse what you already have)
- MySQL/MariaDB running on `:3307` (`root`/`1234`).
- Your Google OAuth client (redirect URI `http://localhost:3000/api/auth/callback/google`).

### Steps

```bash
# 1. Install THIS skill globally so a fresh session can load it
npx skills add ./.claude/skills/nextjs16-prisma-mysql-betterauth -g -a claude-code -y

# 2. Create an empty Next.js 16 app in a sibling folder
cd D:/tutorial
npx create-next-app@latest skill-test --ts --tailwind --app --turbopack --no-eslint --import-alias "@/*"
cd skill-test
```

3. Open `D:/tutorial/skill-test` in a **fresh Claude Code session** and prompt:
   > Set up Better Auth with Google + email/password, backed by Prisma and MySQL. Use the
   > `nextjs16-prisma-mysql-betterauth` skill. DB is MariaDB on localhost:3307 (root/1234).

4. When asked, supply: `DATABASE_URL=mysql://root:1234@localhost:3307/skill_test`,
   a fresh `BETTER_AUTH_SECRET`, and your `GOOGLE_CLIENT_ID`/`GOOGLE_CLIENT_SECRET`.

5. Let it install + generate the schema + `db push`, then run the acceptance checks below.

### Acceptance checklist (all must pass)

```bash
# A. Builds + type-checks cleanly
npm run build

# B. Better Auth handler is live
curl -s http://localhost:3000/api/auth/get-session      # → null

# C. Email/password sign-up writes a row
curl -s -X POST http://localhost:3000/api/auth/sign-up/email \
  -H "Content-Type: application/json" \
  -d '{"email":"t@e.com","password":"password123","name":"T"}'
# → {"token":"…","user":{…}}   then verify in the DB:
mysql -h 127.0.0.1 -P 3307 -uroot -p1234 -e "SELECT email FROM skill_test.user;" 2>/dev/null

# D. Google flow builds the right URL (with the account picker)
curl -s -X POST http://localhost:3000/api/auth/sign-in/social \
  -H "Content-Type: application/json" \
  -d '{"provider":"google","callbackURL":"/dashboard"}' | grep -o "prompt=select_account"

# E. Dashboard guard redirects when signed out
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" http://localhost:3000/dashboard
# → 307 http://localhost:3000/sign-in
```

Also do one manual browser pass: `/sign-up` → lands on `/dashboard` → avatar shows → Sign out works; `/sign-in` → Continue with Google → account picker → `/dashboard`.

---

## Test 4 — Spot-check the gotchas are real

While the scaffold is running, confirm the skill's headline claims actually hold (so we're not
teaching something that's changed):

- `prisma.config.ts` has `import "dotenv/config"` and `db push` didn't error on `DATABASE_URL`.
- The generated `schema.prisma` kept `provider = "prisma-client"` + `output = "../generated"` (CLI didn't clobber it).
- `lib/auth.ts` has `nextCookies()` in `plugins`, and `auth.api.getSession` resolves in the server component.
- The google provider has `prompt: "select_account"` and it appears in the auth URL (Test 3-D).

If any claim is wrong, fix it in `SKILL.md` / `references/templates.md` and re-run Test 3.

---

## Cleanup after testing

```bash
# Remove the global test install
npx skills remove nextjs16-prisma-mysql-betterauth -g -a claude-code -y
# Drop the throwaway DB + project
mysql -h 127.0.0.1 -P 3307 -uroot -p1234 -e "DROP DATABASE IF EXISTS skill_test;" 2>/dev/null
rm -rf D:/tutorial/skill-test
```

---

## Pass criteria

- Tests 1 & 2: green.
- Test 3: **all** acceptance checks (A–E) pass in the fresh project, plus the manual browser pass.
- Test 4: every spot-checked gotcha matches reality.

If all four pass, the skill is safe to publish (GitHub repo → `npx skills add <you>/<repo>`).
