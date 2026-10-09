# 01 – Skip login for local use

## Goal

When running the app locally (`pnpm dev` → http://localhost:3000/), open straight into the app (`/dashboard`) with no Google/Apple sign-in. Production/Vercel behaviour must stay unchanged.

## How login works today (findings)

- Auth is **Supabase** (`@supabase/ssr`) with **Google/Apple OAuth** only (`app/auth/sign-in/route.ts`, `app/auth/callback/route.ts`, `components/landing-content.tsx`). `app/auth/login/page.tsx` just redirects to `/`.
- `middleware.ts` → `lib/supabase/proxy.ts` (`updateSession`) calls `supabase.auth.getUser()`. If there is no user and the path starts with `/dashboard`, `/profile` or `/invoices`, it redirects to `/auth/login` (→ landing page with the sign-in buttons). A logged-in user visiting `/auth*` or `/` is sent to `/dashboard`.
- `app/page.tsx` also calls `getUser()` and shows `LandingContent` (login buttons) when there is no user.
- **~20 other places read the user** and depend on a real Supabase session: all `app/api/*` routes (server client), plus client components (`dashboard`, `invoices/new`, `suppliers`, `buyers`, `step-buyer`, `supplier-form`, `supplier-context`, `navbar`, `onboarding`).
- **The database enforces the user.** Every table in `scripts/001_master_schema.sql` has `user_id uuid references auth.users(id)` and Row Level Security policies `auth.uid() = user_id`. Browser-side queries use the anon key, so they only work with a genuine authenticated session.

Consequence: simply removing the redirect would leave the app unable to read or write any data. We need a real session, just one nobody has to log into by hand.

## Options considered

| | Approach | Verdict |
|---|---|---|
| **A** | **Dev auto-login**: in dev only, the middleware silently signs in a fixed Supabase email+password user and stores the session cookie. | **Recommended.** ~1 file changed, RLS and all existing code untouched. |
| B | Fake a user in code (hard-coded `user.id`) and replace every `getUser()` plus switch to the service-role key to bypass RLS. | Rejected. 20+ files touched, browser-side queries would need to be moved server-side, weakens security model, high regression risk. |
| C | Run a fully local Supabase (Docker) with a seeded user. | Heavier setup; still needs A's auto-login to skip the sign-in screen. Possible later if you want to stop using the hosted project. |

## Plan (Option A)

### Step 0 – Prepare a password-based dev user in Supabase (one-time, manual)

Pick one:

1. **Keep your existing data (recommended):** set a password on the user you currently log in with via Google, using the Admin API (needs `SUPABASE_SERVICE_ROLE_KEY`, already in `.env.local`):
   ```ts
   // one-off script, run once locally
   await admin.auth.admin.updateUserById('<your-user-uuid>', { password: '<dev-password>' })
   ```
   User UUID: Supabase Dashboard → Authentication → Users. The email stays the same; the user can still use Google too.
2. **Fresh empty account:** Dashboard → Authentication → Users → *Add user* (tick *Auto Confirm User*). The app will start with no supplier/invoices and go through `/onboarding`.

Also confirm *Authentication → Providers → Email* is enabled (password sign-in is allowed even if "Confirm email" is on, as long as the user is confirmed).

### Step 1 – Environment variables

Add to `.env.local` (git-ignored) and document in `.env.local.example`:

```
# Local only: auto sign-in, skips the login screen. Never set on Vercel.
DEV_AUTO_LOGIN=true
DEV_USER_EMAIL=you@example.com
DEV_USER_PASSWORD=...
```

Deliberately **not** `NEXT_PUBLIC_*` so the credentials never reach the browser.

### Step 2 – Auto sign-in in the middleware (`lib/supabase/proxy.ts`)

After the existing `supabase.auth.getUser()`:

```ts
const devAutoLogin =
  process.env.NODE_ENV === 'development' &&
  process.env.DEV_AUTO_LOGIN === 'true' &&
  !process.env.VERCEL

let user = (await supabase.auth.getUser()).data.user

if (!user && devAutoLogin) {
  const { data, error } = await supabase.auth.signInWithPassword({
    email: process.env.DEV_USER_EMAIL!,
    password: process.env.DEV_USER_PASSWORD!,
  })
  if (error) console.error('[dev-auto-login] failed:', error.message)
  else user = data.user
}
```

Why this works with no other changes:
- The existing `cookies.setAll` callback already writes the session cookies to both the request (so server components/API routes in the *same* request see the user) and the response (so the browser keeps them).
- `@supabase/ssr` cookies are readable by the browser client, so every client-side `supabase.auth.getUser()` and RLS-protected query works.
- With a user present, the existing logic redirects `/` and `/auth/*` to `/dashboard` and stops the `/auth/login` redirect. The landing/login page is never shown.
- The sign-in happens once; afterwards the cookie is valid and the normal `getUser()` path is used (token refresh is handled as before).
- If sign-in fails (wrong creds, offline) it logs and falls through to today's behaviour, so nothing breaks worse than it does now.

### Step 3 – Safety guards

- Triple gate: `NODE_ENV === 'development'` **and** `DEV_AUTO_LOGIN === 'true'` **and** not on Vercel. `next build`/`next start` and production deployments can never auto-login.
- Credentials only in `.env.local`; confirm it is covered by `.gitignore` (and `.env.local.example` holds placeholders only).
- Do not delete the Google/Apple flow – it keeps working for production.

### Step 4 – Small UX clean-ups (optional)

- `components/navbar.tsx`: hide the "Sign out" button when running in dev auto-login (signing out would just log straight back in). Needs a server-provided flag, e.g. via a tiny layout prop – skip if you don't care.
- `app/(app)/dashboard/page.tsx` and `invoices/new/page.tsx` push to `/auth/login` when `user` is null – harmless, no change required.

## Files touched

| File | Change |
|---|---|
| `lib/supabase/proxy.ts` | Add dev auto sign-in block (≈15 lines) |
| `.env.local.example` | Document the three `DEV_*` vars |
| `.env.local` (local, untracked) | Set the vars |
| *(optional)* `components/navbar.tsx` | Hide Sign out in dev |

## Verification

1. `pnpm dev`, open http://localhost:3000/ → lands on `/dashboard` with no login screen.
2. Dashboard shows your suppliers/invoices (Step 0 option 1) or goes to `/onboarding` (option 2).
3. Create a supplier and a draft invoice → saved (confirms RLS writes work with the auto session).
4. Hit `/api/peppol/status` etc. from the UI → no 401s (server routes see the user).
5. Delete cookies and reload → silently logs in again.
6. Set `DEV_AUTO_LOGIN=false` (or run `pnpm build && pnpm start`) → original login page appears again (guard check).

## Risks / notes

- Needs network access to the hosted Supabase project (same as today).
- Supabase may rate-limit password sign-ins if the cookie can't persist (e.g. blocked cookies); the log line from Step 2 makes this visible.
- Invoice data is per-`user_id`; if you choose a *new* dev user (Step 0 option 2) existing invoices will not show up.
- Rollback: set `DEV_AUTO_LOGIN=false` or `git revert` the single commit.
