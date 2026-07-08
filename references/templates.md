# File templates — Next.js 16 + Prisma 7 + MySQL/MariaDB + Better Auth

Verified working skeleton. Copy into a fresh Next.js 16 app. Gotchas behind each non-obvious line
are in `../SKILL.md`.

## Install

```bash
npm install better-auth @prisma/client @prisma/adapter-mariadb mariadb dotenv
npm install -D prisma
```

`package.json` scripts:

```json
"db:generate": "prisma generate",
"db:push": "prisma db push"
```

Schema lifecycle: `npx @better-auth/cli generate --output prisma/schema.prisma` → `npm run db:generate && npm run db:push`.

## lib/prisma.ts (Prisma 7 + mariadb driver — unchanged by auth choice)

```ts
import { PrismaMariaDb } from "@prisma/adapter-mariadb";
import { PrismaClient } from "@/generated/client";

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient };

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({ adapter: new PrismaMariaDb(process.env.DATABASE_URL!) });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

## prisma.config.ts (Prisma 7 — env() needs explicit dotenv)

```ts
import "dotenv/config";
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  datasource: { url: env("DATABASE_URL") },
});
```

## prisma/schema.prisma

Generator + datasource are yours; the **auth models are generated** by the CLI (do not hand-write):

```prisma
datasource db {
  provider = "mysql"
}

generator client {
  provider = "prisma-client"
  output   = "../generated"
}
```

Run `npx @better-auth/cli generate --output prisma/schema.prisma` (answer `y` to overwrite). The CLI
appends `User`, `Session`, `Account`, `Verification` models with MySQL-safe `@db.Text` /
`@@index([…(length: 191)])`. For Google + email/password the `Account` model includes both OAuth
fields and a `password` field.

## lib/auth.ts (server — relative import so the CLI can load it)

```ts
import { betterAuth } from "better-auth";
import { prismaAdapter } from "better-auth/adapters/prisma";
import { nextCookies } from "better-auth/next-js";
import { prisma } from "./prisma"; // relative so @better-auth/cli can resolve it when generating

export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: "mysql" }),
  emailAndPassword: { enabled: true },
  socialProviders: {
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      prompt: "select_account", // force Google's account picker on every sign-in
    },
  },
  plugins: [nextCookies()], // required for auth.api.getSession in server components
});

export type Session = typeof auth.$Infer.Session;
```

## lib/auth-client.ts (client)

```ts
import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient();
export const { signIn, signUp, signOut, useSession } = authClient;
```

## app/api/auth/[...all]/route.ts

```ts
import { auth } from "@/lib/auth";
import { toNextJsHandler } from "better-auth/next-js";

export const { GET, POST } = toNextJsHandler(auth);
```

## components/sign-out-button.tsx

```tsx
"use client";
import { useRouter } from "next/navigation";
import { signOut } from "@/lib/auth-client";

export function SignOutButton({ className = "" }: { className?: string }) {
  const router = useRouter();
  return (
    <button
      type="button"
      onClick={async () => {
        await signOut();
        router.push("/");
      }}
      className={`rounded-md border border-border px-4 py-2 text-sm transition-colors hover:bg-foreground/5 ${className}`}
    >
      Sign out
    </button>
  );
}
```

## app/sign-in/page.tsx (client — Google button + email/password form)

```tsx
"use client";
import { useState } from "react";
import { useRouter } from "next/navigation";
import { signIn } from "@/lib/auth-client";

export default function SignInPage() {
  const router = useRouter();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);

  async function onEmail(e: React.FormEvent) {
    e.preventDefault();
    setError(null);
    const { error } = await signIn.email({ email, password });
    if (error) setError(error.message ?? "Sign-in failed");
    else router.push("/dashboard");
  }

  return (
    <div className="flex min-h-screen items-center justify-center px-6">
      <div className="w-full max-w-sm space-y-6 rounded-lg border border-border p-8 shadow-sm">
        <div>
          <h1 className="text-2xl font-semibold">Sign in</h1>
          <p className="mt-1 text-sm text-muted-foreground">Use Google or your email.</p>
        </div>
        <button
          type="button"
          onClick={() => signIn.social({ provider: "google", callbackURL: "/dashboard" })}
          className="w-full rounded-md bg-foreground px-4 py-2 text-background transition-colors hover:bg-foreground/90"
        >
          Continue with Google
        </button>
        <div className="flex items-center gap-3 text-xs text-muted-foreground">
          <span className="h-px flex-1 bg-border" /> OR <span className="h-px flex-1 bg-border" />
        </div>
        <form onSubmit={onEmail} className="space-y-3">
          <input name="email" type="email" required placeholder="Email" value={email}
            onChange={(e) => setEmail(e.target.value)}
            className="w-full rounded-md border border-border bg-background px-3 py-2 outline-none focus:ring-2 focus:ring-foreground/20" />
          <input name="password" type="password" required placeholder="Password" value={password}
            onChange={(e) => setPassword(e.target.value)}
            className="w-full rounded-md border border-border bg-background px-3 py-2 outline-none focus:ring-2 focus:ring-foreground/20" />
          {error && <p className="text-sm text-red-600 dark:text-red-400">{error}</p>}
          <button type="submit"
            className="w-full rounded-md bg-foreground px-4 py-2 text-background transition-colors hover:bg-foreground/90">
            Sign in
          </button>
        </form>
        <p className="text-center text-sm text-muted-foreground">
          No account? <a href="/sign-up" className="font-medium text-foreground hover:underline">Sign up</a>
        </p>
      </div>
    </div>
  );
}
```

## app/sign-up/page.tsx (client — auto-signs-in on success)

```tsx
"use client";
import { useState } from "react";
import { useRouter } from "next/navigation";
import { signUp } from "@/lib/auth-client";

export default function SignUpPage() {
  const router = useRouter();
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);

  async function onSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);
    const { error } = await signUp.email({ email, password, name }); // auto-signs-in
    if (error) setError(error.message ?? "Sign-up failed");
    else router.push("/dashboard");
  }

  // …render a form with name / email / password inputs bound to state, onSubmit={onSubmit},
  // and a link back to /sign-in.
}
```

## app/dashboard/page.tsx (server — session guard + avatar)

```tsx
import Image from "next/image";
import { headers } from "next/headers";
import { redirect } from "next/navigation";
import { auth } from "@/lib/auth";
import { SignOutButton } from "@/components/sign-out-button";

export default async function DashboardPage() {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) redirect("/sign-in");

  const user = session.user;
  const initials = (user.name ?? user.email).split(" ").map((p) => p[0]).join("").slice(0, 2).toUpperCase();

  return (
    <div className="flex min-h-screen flex-col items-center justify-center gap-5 px-6 text-center">
      {user.image ? (
        <Image src={user.image} alt={user.name} width={72} height={72} className="rounded-full object-cover" />
      ) : (
        <div className="flex h-[72px] w-[72px] items-center justify-center rounded-full bg-foreground text-lg font-medium text-background">
          {initials}
        </div>
      )}
      <div className="space-y-1">
        <h1 className="text-2xl font-semibold">{user.name}</h1>
        <p className="text-muted-foreground">{user.email}</p>
      </div>
      <SignOutButton />
    </div>
  );
}
```

## app/page.tsx (server — session-aware home)

```tsx
import Link from "next/link";
import { headers } from "next/headers";
import { auth } from "@/lib/auth";
import { SignOutButton } from "@/components/sign-out-button";

export default async function Home() {
  const session = await auth.api.getSession({ headers: await headers() });
  // session?.user ? show email + Dashboard + SignOutButton : show Sign in / Sign up links.
}
```

## next.config.ts (avatars via next/image)

```ts
import type { NextConfig } from "next";
const nextConfig: NextConfig = {
  images: { remotePatterns: [{ protocol: "https", hostname: "**.googleusercontent.com" }] },
};
export default nextConfig;
```

## app/globals.css (adaptive light/dark — unchanged)

Define `--background`, `--foreground`, `--muted-foreground`, `--border` in `:root` + a
`prefers-color-scheme: dark` block, and expose them via `@theme inline { --color-*: var(--*) }`
so utilities like `bg-foreground`, `border-border`, `text-muted-foreground` resolve and flip.

## .env.example

```env
# Prisma + MySQL — read by the Prisma CLI AND Next.js. Put in .env.
DATABASE_URL="mysql://USER:PASSWORD@localhost:3307/DATABASE"

# Better Auth — Next.js only. Put in .env.local.
BETTER_AUTH_SECRET=""
BETTER_AUTH_URL="http://localhost:3000"
GOOGLE_CLIENT_ID=""
GOOGLE_CLIENT_SECRET=""
```

`DATABASE_URL` in `.env` (Prisma CLI reads `.env`); `BETTER_AUTH_*` / `GOOGLE_*` in `.env.local`.
Ensure `.gitignore` has `.env*` plus `!.env.example`.

## What is deliberately NOT here

- **No middleware** in the example (page-level guards via `auth.api.getSession`). Better Auth's
  `getSessionCookie` IS edge-safe, so add middleware if you want cheap route protection without a DB hit.
- **No SessionProvider wrapper** — Better Auth's `useSession` from `better-auth/react` needs no provider.
