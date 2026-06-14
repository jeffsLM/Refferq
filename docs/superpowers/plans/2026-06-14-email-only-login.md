# Email-Only Login Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the OTP login flow with a single email-field login that issues a JWT session when the email matches an ACTIVE user.

**Architecture:** Add a new POST endpoint `/api/auth/login-email` that mirrors the existing password-login route (`src/app/api/auth/login/route.ts`) but skips password validation. Rewrite `src/app/login/page.tsx` to a 1-step form that calls the new endpoint. Leave OTP infrastructure (`/api/auth/send-otp`, `/api/auth/verify-otp`, `src/lib/otp.ts`) and the password endpoint intact for register and possible future use.

**Tech Stack:** Next.js 16 App Router (route handlers), Prisma 6 (PostgreSQL), `jose` for JWT signing, `bcryptjs` (unused by this flow, kept for the password endpoint), TailwindCSS + shadcn/ui components, React 19.

**Spec:** `docs/superpowers/specs/2026-06-14-email-only-login-design.md`

---

## File structure

| File | Role | Status |
|---|---|---|
| `src/app/api/auth/login-email/route.ts` | New POST handler — validates email, finds ACTIVE user, signs JWT, sets cookie. | Create (Task 1) |
| `src/app/login/page.tsx` | Login UI — single email field calling the new endpoint. | Rewrite (Task 2) |
| `docs/superpowers/specs/2026-06-14-email-only-login-design.md` | Source spec. | Unchanged |

**Intentionally untouched:** `src/app/api/auth/login/route.ts`, `src/app/api/auth/send-otp/route.ts`, `src/app/api/auth/verify-otp/route.ts`, `src/lib/otp.ts`, `src/app/register/page.tsx`, `prisma/schema.prisma`.

**No tests** are added — the repo has no auth-flow test harness today, and the spec explicitly puts that out of scope (spec §6). Verification is manual via `npm run dev`.

---

## Task 1: Create the `/api/auth/login-email` endpoint

**Files:**
- Create: `src/app/api/auth/login-email/route.ts`

This task adds the backend endpoint. The handler is a near-copy of `src/app/api/auth/login/route.ts` with the password check removed and the rate-limit endpoint key changed.

- [ ] **Step 1: Create the route file**

Create `src/app/api/auth/login-email/route.ts` with the full contents below:

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { prisma } from '@/lib/prisma';
import { SignJWT } from 'jose';
import { checkRateLimit } from '@/lib/rate-limit';

const JWT_SECRET = new TextEncoder().encode(
  process.env.JWT_SECRET!
);

const EMAIL_REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

export async function POST(request: NextRequest) {
  try {
    // Rate limit: 5 login attempts per minute per IP
    const ip = request.headers.get('x-forwarded-for')?.split(',')[0]?.trim()
      || request.headers.get('x-real-ip')
      || 'unknown';
    const rateLimit = await checkRateLimit(ip, 'auth/login-email', 5, 60 * 1000);
    if (!rateLimit.allowed) {
      return NextResponse.json(
        { success: false, message: 'Too many login attempts. Please try again later.' },
        {
          status: 429,
          headers: {
            'Retry-After': Math.ceil((rateLimit.resetAt.getTime() - Date.now()) / 1000).toString(),
            'X-RateLimit-Limit': rateLimit.limit.toString(),
            'X-RateLimit-Remaining': '0',
          },
        }
      );
    }

    const body = await request.json();
    const { email } = body;

    if (!email || typeof email !== 'string' || !EMAIL_REGEX.test(email)) {
      return NextResponse.json(
        { success: false, message: 'Invalid email' },
        { status: 400 }
      );
    }

    const user = await prisma.user.findUnique({
      where: { email: email.toLowerCase().trim() },
    });

    if (!user) {
      return NextResponse.json(
        { success: false, message: 'Invalid email' },
        { status: 401 }
      );
    }

    if (user.status !== 'ACTIVE') {
      return NextResponse.json(
        { success: false, message: 'Unable to log in. Please contact support if you need assistance.' },
        { status: 403 }
      );
    }

    const token = await new SignJWT({
      userId: user.id,
      email: user.email,
      role: user.role,
      name: user.name,
    })
      .setProtectedHeader({ alg: 'HS256' })
      .setIssuedAt()
      .setExpirationTime('24h')
      .sign(JWT_SECRET);

    const { password: _password, ...userData } = user;

    const response = NextResponse.json({
      success: true,
      message: 'Login successful',
      user: userData,
    });

    response.cookies.set('auth-token', token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 86400,
      path: '/',
    });

    return response;
  } catch (error) {
    console.error('Login-email API error:', error);
    return NextResponse.json(
      { success: false, message: 'Login failed' },
      { status: 500 }
    );
  }
}
```

- [ ] **Step 2: Type-check the new file**

Run: `npx tsc --noEmit`
Expected: no errors related to `src/app/api/auth/login-email/route.ts`. (Pre-existing errors in unrelated files, if any, can be ignored — but verify the new file isn't the source.)

- [ ] **Step 3: Smoke-test the endpoint with the dev server**

In one terminal: `npm run dev`

In another, run each of these and verify the JSON status code + body match. (PowerShell users — replace `curl` with `Invoke-WebRequest`; the bodies below use `curl` syntax.)

```bash
# 1) Valid admin login — expect 200 + Set-Cookie: auth-token=...
curl -i -X POST http://localhost:3000/api/auth/login-email \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com"}'

# 2) Unknown email — expect 401, message "Invalid email"
curl -i -X POST http://localhost:3000/api/auth/login-email \
  -H "Content-Type: application/json" \
  -d '{"email":"naoexiste@example.com"}'

# 3) Malformed email — expect 400, message "Invalid email"
curl -i -X POST http://localhost:3000/api/auth/login-email \
  -H "Content-Type: application/json" \
  -d '{"email":"not-an-email"}'

# 4) Missing body field — expect 400
curl -i -X POST http://localhost:3000/api/auth/login-email \
  -H "Content-Type: application/json" \
  -d '{}'
```

Stop the dev server (Ctrl+C) when done. If any response doesn't match, fix the route before continuing.

- [ ] **Step 4: Commit**

```bash
git add src/app/api/auth/login-email/route.ts
git commit -m "feat(auth): add email-only login endpoint"
```

---

## Task 2: Rewrite the login page to use the new endpoint

**Files:**
- Modify: `src/app/login/page.tsx` (full rewrite — replace file contents)

This task swaps the 2-step OTP UI for a 1-step email-only form.

- [ ] **Step 1: Replace `src/app/login/page.tsx` with the new contents**

Overwrite the file with exactly this:

```tsx
'use client';

import React, { useState } from 'react';
import { useRouter } from 'next/navigation';
import Link from 'next/link';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import {
  Card,
  CardContent,
  CardDescription,
  CardFooter,
  CardHeader,
  CardTitle,
} from '@/components/ui/card';
import { Alert, AlertDescription } from '@/components/ui/alert';
import { Target, Mail, Loader2 } from 'lucide-react';

export default function LoginPage() {
  const router = useRouter();
  const [email, setEmail] = useState('');
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');
    setLoading(true);

    try {
      const res = await fetch('/api/auth/login-email', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        credentials: 'include',
        body: JSON.stringify({ email }),
      });

      const data = await res.json();

      if (res.ok && data.success) {
        const user = data.user;
        if (user.role === 'ADMIN') {
          router.push('/admin');
        } else {
          router.push('/affiliate');
        }
      } else {
        setError(data.message || 'Login failed');
      }
    } catch (_e) {
      setError('Something went wrong. Please try again.');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-gradient-to-br from-background via-muted/30 to-background p-4">
      <div className="w-full max-w-md space-y-6">
        <div className="text-center space-y-2">
          <div className="mx-auto flex h-14 w-14 items-center justify-center rounded-2xl bg-primary shadow-lg shadow-primary/25">
            <Target className="h-7 w-7 text-primary-foreground" />
          </div>
          <h1 className="text-2xl font-bold tracking-tight">Refferq</h1>
          <p className="text-sm text-muted-foreground">
            Affiliate Marketing Platform
          </p>
        </div>

        <Card className="border-0 shadow-xl shadow-black/5">
          <CardHeader className="text-center pb-4">
            <CardTitle className="text-xl">Welcome back</CardTitle>
            <CardDescription>
              Enter your email to sign in to your account
            </CardDescription>
          </CardHeader>
          <form onSubmit={handleSubmit}>
            <CardContent className="space-y-4">
              {error && (
                <Alert variant="destructive">
                  <AlertDescription>{error}</AlertDescription>
                </Alert>
              )}
              <div className="space-y-2">
                <Label htmlFor="email">Email address</Label>
                <div className="relative">
                  <Mail className="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-muted-foreground" />
                  <Input
                    id="email"
                    type="email"
                    placeholder="you@example.com"
                    value={email}
                    onChange={(e) => setEmail(e.target.value)}
                    className="pl-10"
                    required
                    autoFocus
                    autoComplete="email"
                  />
                </div>
              </div>
            </CardContent>
            <CardFooter className="flex-col gap-4">
              <Button type="submit" className="w-full" size="lg" disabled={loading || !email}>
                {loading ? (
                  <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                ) : (
                  <Mail className="mr-2 h-4 w-4" />
                )}
                {loading ? 'Signing in...' : 'Sign in'}
              </Button>
            </CardFooter>
          </form>
        </Card>

        <p className="text-center text-sm text-muted-foreground">
          Don&apos;t have an account?{' '}
          <Link href="/register" className="font-medium text-primary hover:underline">
            Sign up
          </Link>
        </p>
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Type-check**

Run: `npx tsc --noEmit`
Expected: no new errors in `src/app/login/page.tsx`. The removed imports (`InputOTP*`, `ShieldCheck`, `ArrowLeft`) should not cause unused-import warnings because they are gone from the file entirely.

- [ ] **Step 3: Manual UI smoke test**

Start the dev server: `npm run dev`

Walk through the checklist below in a browser at `http://localhost:3000/login`. After each scenario, refresh the page to reset state.

| # | Action | Expected |
|---|---|---|
| 1 | Type `admin@example.com`, click `Sign in` | Redirected to `/admin`; refreshing `/admin` stays logged in (cookie set). |
| 2 | Log out (DELETE `/api/auth/logout` or clear `auth-token` cookie), type `sarah.johnson@example.com`, click `Sign in` | Redirected to `/affiliate`. |
| 3 | Log out, type `david.lee@example.com`, click `Sign in` | Redirected to `/affiliate`. |
| 4 | Log out, type `naoexiste@example.com`, click `Sign in` | Red alert: `Invalid email`. Stays on `/login`. |
| 5 | Log out, type `not-an-email` | Browser blocks submission (`type="email"` + `required`), OR if HTML validation is bypassed, alert says `Invalid email`. |
| 6 | Log out, leave field empty | `Sign in` button disabled. |
| 7 | Open SQL client, run `UPDATE users SET status='PENDING' WHERE email='david.lee@example.com';` then try to log in as `david.lee@example.com` | Red alert: `Unable to log in. Please contact support if you need assistance.` After test, restore: `UPDATE users SET status='ACTIVE' WHERE email='david.lee@example.com';` |
| 8 | Submit `admin@example.com` 6 times in under a minute from the same browser | 6th attempt shows: `Too many login attempts. Please try again later.` |

If any scenario fails, stop and fix the issue before committing.

Stop the dev server (Ctrl+C).

- [ ] **Step 4: Commit**

```bash
git add src/app/login/page.tsx
git commit -m "feat(auth): switch login page to email-only flow"
```

---

## Self-review pass (already done during plan authoring)

Spec coverage check against `docs/superpowers/specs/2026-06-14-email-only-login-design.md`:

| Spec section | Implemented by |
|---|---|
| §2 Architecture (new endpoint + page rewrite) | Tasks 1 and 2 |
| §3 Endpoint contract (rate limit, regex, lookup, ACTIVE check, JWT, cookie, all status codes) | Task 1 Step 1 (full handler code) |
| §4 Frontend (state, handler, imports, layout, button label) | Task 2 Step 1 (full file) |
| §5 Error handling (Alert variant + network fallback) | Task 2 Step 1 — `setError` + try/catch fallback message |
| §6 Test plan (manual scenarios 1–9) | Task 2 Step 3 (UI smoke) + Task 1 Step 3 (API smoke) |
| §7 Files changed | File structure table at top |
| §8 Reversibility | Each task is its own commit — revert deletes the route file and reverts the page edit |

No placeholders, no "TBD", every step contains the full code or the exact command. Function/identifier names (`handleSubmit`, `EMAIL_REGEX`, `auth-token`, `/api/auth/login-email`) match between Tasks 1 and 2.

---

## Out of scope (reminder)

- Register page (`/register`) — continues using OTP unchanged.
- `/api/auth/login` (password) — left intact; not used by the UI.
- `/api/auth/send-otp`, `/api/auth/verify-otp`, `src/lib/otp.ts` — left intact.
- `User.password` column — left in schema, unused by this flow.
- Automated tests — none added; manual verification only.
- The uncommitted `package.json` fix for `db:seed` from prior work — unrelated to this plan; commit separately if desired.
