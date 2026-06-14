# Email-Only Login — Design Spec

**Date:** 2026-06-14
**Status:** Approved
**Scope:** Login page only. Register and other OTP usages remain untouched.

---

## Security note (explicit, owner-acknowledged)

This design removes both the password and the OTP verification step from the login flow. After the change, **possession of an email address is sufficient to obtain a session as that user**, including ADMIN sessions. There is no proof-of-ownership of the email (no OTP, no magic link), and no second factor (no password). The user accepted this trade-off knowingly during brainstorming. The decision is documented here so future readers do not interpret the simplified flow as an oversight.

This applies to all environments (no env-gated override). If a stricter posture is needed later, reintroduce OTP or password by re-wiring the login page to `/api/auth/login` (password, still present) or `/api/auth/{send,verify}-otp` (still present).

---

## 1. Goal

Replace the current 2-step OTP login flow with a 1-step email-only flow.

**In scope:**
- New endpoint `POST /api/auth/login-email`
- Rewrite of `src/app/login/page.tsx` to a single email field

**Out of scope:**
- `/register` page (continues using OTP)
- `/api/auth/login` (password endpoint, left untouched)
- `/api/auth/send-otp` and `/api/auth/verify-otp` (left untouched)
- `src/lib/otp.ts` (left untouched)
- `User.password` column in the Prisma schema (left untouched, unused by the new flow)
- Other UI surfaces referencing OTP (`admin/emails`, `admin/settings`)

---

## 2. Architecture

Two files change, one is added:

| File | Change |
|---|---|
| `src/app/api/auth/login-email/route.ts` | **New.** POST handler implementing email-only login. |
| `src/app/login/page.tsx` | **Rewrite.** Single-step email form. |

Nothing else in the codebase is modified.

---

## 3. Endpoint contract — `POST /api/auth/login-email`

### Request
```json
{ "email": "user@example.com" }
```

### Behavior
1. Rate limit: 5 requests per minute per IP, scoped to `auth/login-email`. Mirrors `/api/auth/login` (`src/app/api/auth/login/route.ts:17`). Implementation: `checkRateLimit(ip, 'auth/login-email', 5, 60 * 1000)`.
2. Validate email presence and format with regex `/^[^\s@]+@[^\s@]+\.[^\s@]+$/` (same regex used by `send-otp/route.ts:29`).
3. Lookup: `prisma.user.findUnique({ where: { email: email.toLowerCase().trim() } })`.
4. If user not found → return generic 401.
5. If `user.status !== 'ACTIVE'` → return 403 with the same wording used by the current password login (`login/route.ts:58`).
6. Issue JWT identical in shape and TTL to the existing login flow (`login/route.ts:73-82`): HS256, payload `{ userId, email, role, name }`, 24h expiration, signed with `process.env.JWT_SECRET`.
7. Set cookie `auth-token`: `httpOnly: true`, `secure: NODE_ENV === 'production'`, `sameSite: 'lax'`, `maxAge: 86400`, `path: '/'`.

### Response matrix

| Condition | Status | Body |
|---|---|---|
| Rate-limited | 429 | `{ success: false, message: 'Too many login attempts. Please try again later.' }` + `Retry-After`, `X-RateLimit-*` headers |
| Missing/invalid email | 400 | `{ success: false, message: 'Invalid email' }` |
| User not found | 401 | `{ success: false, message: 'Invalid email' }` (generic — avoids account enumeration) |
| User status ≠ ACTIVE | 403 | `{ success: false, message: 'Unable to log in. Please contact support if you need assistance.' }` |
| Success | 200 | `{ success: true, message: 'Login successful', user: { ...userData without password } }` + `Set-Cookie: auth-token=...` |
| Unhandled error | 500 | `{ success: false, message: 'Login failed' }` |

The endpoint must not return distinguishable wording or status between "user not found" and "invalid email format" beyond the 400 vs 401 distinction (intentionally generic message text).

---

## 4. Frontend — `src/app/login/page.tsx`

### State (before → after)
- Remove: `step`, `otp`, `message` states. Remove `Step` type.
- Keep: `email`, `loading`, `error`.

### Handlers (before → after)
- Remove: `handleSendOTP`, `handleVerifyOTP`, `handleResendOTP`.
- Add: `handleSubmit(e)` that POSTs `{ email }` to `/api/auth/login-email` and:
  - On success: redirect to `/admin` if `user.role === 'ADMIN'`, otherwise `/affiliate` (same logic as current `page.tsx:84-89`).
  - On failure: `setError(data.message || 'Login failed')`.

### Imports to remove
`InputOTP`, `InputOTPGroup`, `InputOTPSlot`, `InputOTPSeparator`, `ShieldCheck`, `ArrowLeft`.

### Imports to keep
`Button`, `Input`, `Label`, `Card*`, `Alert`, `AlertDescription`, `Target`, `Mail`, `Loader2`.

### Visual layout
- Card title: `Welcome back`
- Card description: `Enter your email to sign in to your account`
- Single email field (same styling as current step 1)
- Single submit button: label `Sign in`, disabled when `loading || !email`
- Footer link to `/register` unchanged

Roughly half the size of the current file. No `step === 'otp'` branch remains.

---

## 5. Error handling

- All API failures render in the existing `Alert variant="destructive"` component with the server's `message`.
- Network/JS errors render: `Something went wrong. Please try again.` (matches current pattern at `page.tsx:58`).

---

## 6. Test plan (manual)

Run `npm run dev`, then exercise:

| # | Input | Expected |
|---|---|---|
| 1 | `admin@example.com` | Redirect to `/admin` |
| 2 | `sarah.johnson@example.com` | Redirect to `/affiliate` |
| 3 | `david.lee@example.com` | Redirect to `/affiliate` |
| 4 | `naoexiste@example.com` | 401, alert `Invalid email`, stays on `/login` |
| 5 | `not-an-email` | 400, alert `Invalid email` |
| 6 | Empty email | Submit button disabled |
| 7 | A user with status `PENDING` (insert via SQL) | 403, alert `Unable to log in. Please contact support if you need assistance.` |
| 8 | 6 submissions within 1 minute from same IP | 6th returns 429 |
| 9 | After successful login: refresh `/admin` | Session persists (cookie set correctly) |

No automated tests are added in this change. The repo does not currently have an auth-flow test harness; adding one is out of scope.

---

## 7. Files changed (summary)

- **Add:** `src/app/api/auth/login-email/route.ts`
- **Modify:** `src/app/login/page.tsx`
- **Untouched (intentionally):** `src/app/api/auth/login/route.ts`, `src/app/api/auth/send-otp/route.ts`, `src/app/api/auth/verify-otp/route.ts`, `src/lib/otp.ts`, `src/app/register/page.tsx`, `prisma/schema.prisma`

---

## 8. Reversibility

To revert: delete `src/app/api/auth/login-email/route.ts` and restore the previous `login/page.tsx` from git. The other auth endpoints were untouched, so the OTP flow becomes functional again immediately.
