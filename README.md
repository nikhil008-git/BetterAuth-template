# How to use Prisma ORM and Prisma Postgres with Better Auth and Next.js

**Estimated time:** 25 minutes

## Introduction

Better Auth is a modern, open-source authentication solution for web applications. It is built with TypeScript and provides a simple and extensible authentication experience with support for multiple database adapters, including Prisma.

In this guide, you will wire Better Auth into a brand-new Next.js application and persist users in a Prisma Postgres database.

---

## Prerequisites

* Node.js 20+
* Basic familiarity with Next.js App Router
* Basic familiarity with Prisma

---

## 1. Set up your project

Create a new Next.js application:

```bash
npx create-next-app@latest betterauth-nextjs-prisma
```

Choose the default options:

* TypeScript: Yes
* ESLint: Yes
* Tailwind CSS: Yes
* Use `src/` directory: Yes
* App Router: Yes
* Turbopack: Yes
* Customize import alias: No

Navigate into the project directory:

```bash
cd betterauth-nextjs-prisma
```

This creates a modern Next.js project with TypeScript, ESLint, Tailwind CSS, and App Router.

---

## 2. Set up Prisma

### 2.1 Install Prisma and dependencies

```bash
npm install prisma tsx @types/pg --save-dev
npm install @prisma/client @prisma/adapter-pg dotenv pg
```

Initialize Prisma:

```bash
npx prisma init --db --output ../src/generated/prisma
```

This creates:

* `prisma/schema.prisma`
* A Prisma Postgres database
* `.env` with `DATABASE_URL`
* Generated Prisma Client output directory

---

### 2.2 Configure Prisma

Create `prisma.config.ts` at the project root:

```ts
import 'dotenv/config'
import { defineConfig, env } from 'prisma/config';

export default defineConfig({
  schema: 'prisma/schema.prisma',
  migrations: {
    path: 'prisma/migrations',
  },
  datasource: {
    url: env('DATABASE_URL'),
  },
});
```

---

### 2.3 Generate the Prisma client

```bash
npx prisma generate
```

---

### 2.4 Set up a global Prisma client

Create the Prisma client utility:

```bash
mkdir -p src/lib
touch src/lib/prisma.ts
```

```ts
import { PrismaClient } from "@/generated/prisma/client";
import { PrismaPg } from "@prisma/adapter-pg";

const adapter = new PrismaPg({
  connectionString: process.env.DATABASE_URL!,
});

const globalForPrisma = global as unknown as {
  prisma: PrismaClient;
};

const prisma =
  globalForPrisma.prisma || new PrismaClient({ adapter });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;

export default prisma;
```

---

## 3. Set up Better Auth

### 3.1 Install and configure Better Auth

```bash
npm install better-auth
```

Generate a secure secret:

```bash
npx @better-auth/cli@latest secret
```

Update `.env`:

```env
BETTER_AUTH_SECRET=your-generated-secret
BETTER_AUTH_URL=http://localhost:3000
DATABASE_URL="your-database-url"
```

Create `src/lib/auth.ts`:

```ts
import { betterAuth } from 'better-auth'
import { prismaAdapter } from 'better-auth/adapters/prisma'
import prisma from '@/lib/prisma'

export const auth = betterAuth({
  database: prismaAdapter(prisma, {
    provider: 'postgresql',
  }),
  emailAndPassword: {
    enabled: true,
  },
});
```

If using a different port, configure trusted origins:

```ts
trustedOrigins: ['http://localhost:3001']
```

---

### 3.2 Add Better Auth models to Prisma schema

```bash
npx @better-auth/cli generate
```

This adds `User`, `Session`, `Account`, and `Verification` models to `schema.prisma`.

---

### 3.3 Migrate the database

```bash
npx prisma migrate dev --name add-auth-models
npx prisma generate
```

---

## 4. Set up API routes

Create the API route:

```bash
mkdir -p src/app/api/auth/[...all]
touch src/app/api/auth/[...all]/route.ts
```

```ts
import { auth } from "@/lib/auth";
import { toNextJsHandler } from "better-auth/next-js";

export const { POST, GET } = toNextJsHandler(auth);
```

Create the client helper:

```bash
touch src/lib/auth-client.ts
```

```ts
import { createAuthClient } from 'better-auth/react'

export const { signIn, signUp, signOut, useSession } = createAuthClient()
```

---

## 5. Set up pages

### Pages structure

```bash
mkdir -p src/app/{sign-up,sign-in,dashboard}
touch src/app/{sign-up,sign-in,dashboard}/page.tsx
```

---

### 5.1 Sign up page

Complete `src/app/sign-up/page.tsx`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { signUp } from "@/lib/auth-client";

export default function SignUpPage() {
  const router = useRouter();
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setError(null);

    const formData = new FormData(e.currentTarget);

    const res = await signUp.email({
      name: formData.get("name") as string,
      email: formData.get("email") as string,
      password: formData.get("password") as string,
    });

    if (res.error) setError(res.error.message || "Something went wrong.");
    else router.push("/dashboard");
  }

  return (
    <main className="max-w-md mx-auto p-6 space-y-4 text-white">
      <h1 className="text-2xl font-bold">Sign Up</h1>
      {error && <p className="text-red-500">{error}</p>}
      <form onSubmit={handleSubmit} className="space-y-4">
        <input name="name" placeholder="Full Name" required />
        <input name="email" type="email" placeholder="Email" required />
        <input name="password" type="password" minLength={8} required />
        <button type="submit">Create Account</button>
      </form>
    </main>
  );
}
```

---

### 5.2 Sign in page

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { signIn } from "@/lib/auth-client";

export default function SignInPage() {
  const router = useRouter();
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setError(null);

    const formData = new FormData(e.currentTarget);

    const res = await signIn.email({
      email: formData.get("email") as string,
      password: formData.get("password") as string,
    });

    if (res.error) setError(res.error.message || "Something went wrong.");
    else router.push("/dashboard");
  }

  return (
    <main className="max-w-md mx-auto p-6 space-y-4 text-white">
      <h1 className="text-2xl font-bold">Sign In</h1>
      {error && <p className="text-red-500">{error}</p>}
      <form onSubmit={handleSubmit} className="space-y-4">
        <input name="email" type="email" required />
        <input name="password" type="password" required />
        <button type="submit">Sign In</button>
      </form>
    </main>
  );
}
```

---

### 5.3 Dashboard page (protected)

```tsx
"use client";

import { useEffect } from "react";
import { useRouter } from "next/navigation";
import { useSession, signOut } from "@/lib/auth-client";

export default function DashboardPage() {
  const router = useRouter();
  const { data: session, isPending } = useSession();

  useEffect(() => {
    if (!isPending && !session?.user) router.push("/sign-in");
  }, [isPending, session, router]);

  if (isPending) return <p>Loading...</p>;
  if (!session?.user) return <p>Redirecting...</p>;

  const { user } = session;

  return (
    <main className="max-w-md mx-auto p-6 space-y-4 text-white">
      <h1 className="text-2xl font-bold">Dashboard</h1>
      <p>Welcome, {user.name}</p>
      <p>Email: {user.email}</p>
      <button onClick={() => signOut()}>Sign Out</button>
    </main>
  );
}
```

---

### 5.4 Home page

```tsx
"use client";

import { useRouter } from "next/navigation";

export default function Home() {
  const router = useRouter();

  return (
    <main className="flex items-center justify-center h-screen">
      <button onClick={() => router.push("/sign-up")}>Sign Up</button>
      <button onClick={() => router.push("/sign-in")}>Sign In</button>
    </main>
  );
}
```

---

## 6. Test the application

Start the development server:

```bash
npm run dev
```

Visit `http://localhost:3000`.

* Sign up with a new account
* Access the dashboard
* Sign out and sign back in

Open Prisma Studio to inspect the database:

```bash
npx prisma studio
```

---

## Conclusion

You now have a fully functional authentication system using Better Auth, Prisma, and Next.js, with email and password authentication backed by a Postgres database.
