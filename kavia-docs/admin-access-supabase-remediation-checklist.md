<!--
MongoDB Document Metadata:
- Original File Path: kavia-docs/CodeWiki/Specs/Other/admin-access-supabase-remediation-checklist.md
- Operation: write
- Timestamp: 2026-01-07T10:45:20.498686+00:00
- Restored At: 2026-01-09T13:15:04.413005+00:00
- Task ID: cg9d1ee1eb
-->

# Admin-only Access Remediation Checklist (Supabase + frontend_admin_panel)

## Scope and goal

This document is a practical, end-to-end checklist to diagnose and fix admin-only access for the **RoadRescue – QuickAssist Admin Panel** (`roadrescue-quickassist-platform-41193-41203/frontend_admin_panel`) when running in **Supabase mode**.

The admin panel’s gating logic expects the following invariant:

1. The user is authenticated via Supabase Auth.
2. A row exists in `public.profiles` whose `id` equals the user’s `auth.uid()`.
3. That `public.profiles.role` equals `admin`.
4. RLS policies allow:
   - the signed-in user to read their own profile row (so the gate can verify the role)
   - admins to read and update the tables the admin UI uses (`profiles`, `requests`, `fees`)

This document includes:
- A precise remediation checklist.
- SQL for verifying and fixing `profiles` rows and RLS policies.
- Code touchpoints to confirm or adjust if needed.

## How admin gating works in this repo (current behavior)

### Frontend access gate

The admin panel uses:
- `src/App.js` to decide whether to route to `/dashboard` vs `/login`.
- `src/routes/RequireAuth.js` to enforce access for protected routes.

In **Supabase mode**, `RequireAuth`:
1. Resolves the current session via `dataService.getCurrentSession()` (which calls `supabase.auth.getSession()`).
2. Fetches a minimal profile via `dataService.getCurrentProfile()` which queries:

```js
supabase
  .from("profiles")
  .select("id,role,full_name")
  .eq("id", user.id)
  .maybeSingle()
```

3. Allows access only when `profile.role === "admin"`.

Additionally, in dev environments, `RequireAuth` exposes:
- an on-screen debug panel (UID/role/loading/decision)
- a console helper `window.__ADMIN_DEBUG()` to print the same state

### Login behavior

`src/pages/LoginPage.js` signs in using `dataService.login(email, password)` and immediately rejects if returned role is not `admin`.

`dataService.login()` does:
- `supabase.auth.signInWithPassword(...)`
- then `supaGetUserRole()` which runs:

```js
supabase
  .from("profiles")
  .select("role,approved")
  .eq("id", userId)
  .maybeSingle()
```

If the profile row is missing, it attempts:

```js
supabase.from("profiles").insert({ id: userId, email, role: "user", approved: true })
```

This means that if RLS prevents self-insert, login will still succeed (because the function catches errors and returns `{ role: "user", approved: true }`), but the admin portal will deny access (because the role will not be `admin`).

## Common failure modes (and what they look like)

### Failure mode A: No profiles row for the admin user

Symptoms:
- `RequireAuth` blocks with “We couldn’t load your profile…” or “Timed out waiting for your profile…”
- `window.__ADMIN_DEBUG()` shows `uid` populated but `role` is `unknown`

Root cause:
- `public.profiles` row missing for that `auth.users` account and RLS prevents self-insert, or no auto-provisioning exists.

### Failure mode B: Profiles row exists, but role is not `admin`

Symptoms:
- Login page error: “This portal is for admins only.”
- Or gate shows: “Your account role is 'user'…”

Root cause:
- `public.profiles.role` is not set to `admin` for that user.

### Failure mode C: RLS blocks reading own profile row

Symptoms:
- Auth session exists, but profile fetch returns empty or errors.
- The gate blocks even though the row exists.

Root cause:
- Missing `SELECT` policy for `profiles` allowing `id = auth.uid()`.

### Failure mode D: Admin can log in but cannot load Users/Requests/Fees

Symptoms:
- Dashboard or pages show errors such as “permission denied for table …”
- `dataService.listUsers()`, `listRequests()`, or `setFees()` fails.

Root cause:
- Missing admin-only policies on `profiles`, `requests`, and `fees`.

## Remediation checklist (do these in order)

### 1) Confirm the app is actually running in Supabase mode

This app switches between “Supabase mode” and “Mock mode” at runtime based on environment variables.

Minimum required env vars for Supabase mode in this repo:
- `REACT_APP_SUPABASE_URL`
- `REACT_APP_SUPABASE_KEY`

Checklist:
1. Confirm both env vars are present and non-empty at build/runtime.
2. Load the admin panel and run in browser console:

```js
window.__ADMIN_DEBUG?.()
```

If Supabase is configured and you navigate to a protected page, the debug panel should show a UID after login.

Important note about keys:
- In a real deployment, `REACT_APP_SUPABASE_KEY` should typically be the **anon** public key. You should not ship the Supabase service role key to a browser app.

### 2) Confirm the admin account exists in Supabase Auth

In Supabase Dashboard:
- Authentication → Users → ensure your intended admin email exists.

If not, create it (Dashboard) or sign up via your auth flow.

### 3) Verify the admin user has a `public.profiles` row with `role = admin`

Run this in Supabase SQL editor to locate the profile row by email (if `email` is stored in profiles):

```sql
select id, email, role, approved
from public.profiles
where lower(email) = lower('admin@example.com');
```

If a row exists, note the `id` (UUID).

If no row exists:
- You must insert a row with `id` = the user’s `auth.users.id`.

To find the auth user id:

```sql
select id, email
from auth.users
where lower(email) = lower('admin@example.com');
```

Then insert the profile row (replace the UUID):

```sql
insert into public.profiles (id, email, role, approved)
values ('00000000-0000-0000-0000-000000000000', 'admin@example.com', 'admin', true)
on conflict (id) do update
set role = excluded.role,
    email = excluded.email,
    approved = excluded.approved;
```

### 4) Fix RLS on `public.profiles` (minimum required for gating)

The admin gate depends on the logged-in user being able to read their own profile row, by `id = auth.uid()`.

Minimum policies that must exist:

1. Allow authenticated users to read their own profile row.
2. Allow authenticated users to insert their own profile row (optional but recommended for auto-provisioning).
3. Allow admins to read and update profiles (required for Admin → Users page and mechanic approvals).

#### 4.1 Enable RLS (if not enabled)

```sql
alter table public.profiles enable row level security;
```

#### 4.2 Policy: user can read own profile

```sql
create policy "profiles_select_own"
on public.profiles
for select
to authenticated
using (id = auth.uid());
```

#### 4.3 Policy: user can insert own profile (recommended)

This supports the codepath in `dataService.login()` and `supaGetProfile()` that attempts to create a missing profile.

```sql
create policy "profiles_insert_own"
on public.profiles
for insert
to authenticated
with check (id = auth.uid());
```

#### 4.4 Policy: admins can read all profiles

This policy assumes your `profiles` table has a `role` column and that admins have `role = 'admin'` in their own row.

```sql
create policy "profiles_select_admin_all"
on public.profiles
for select
to authenticated
using (
  exists (
    select 1
    from public.profiles p
    where p.id = auth.uid()
      and p.role = 'admin'
  )
);
```

#### 4.5 Policy: admins can update profiles (needed for approvals)

The admin UI updates other users’ profiles when approving mechanics:

- `dataService.approveMechanic(userId)` does: `update profiles set approved=true, role='approved_mechanic' where id = userId`.

So admins need an UPDATE policy:

```sql
create policy "profiles_update_admin_all"
on public.profiles
for update
to authenticated
using (
  exists (
    select 1
    from public.profiles p
    where p.id = auth.uid()
      and p.role = 'admin'
  )
)
with check (
  exists (
    select 1
    from public.profiles p
    where p.id = auth.uid()
      and p.role = 'admin'
  )
);
```

### 5) Fix RLS on `public.requests` and `public.fees` for admin operations

The admin panel reads and writes these tables in Supabase mode:

- Requests:
  - `dataService.listRequests()` selects `*` from `requests`
  - `dataService.updateRequest()` updates `requests`
- Fees:
  - `dataService.getFees()` reads from `fees` where `id = 'default'`
  - `dataService.setFees()` upserts `fees` row `id = 'default'`

#### 5.1 Enable RLS

```sql
alter table public.requests enable row level security;
alter table public.fees enable row level security;
```

#### 5.2 Admin policies for requests

```sql
create policy "requests_select_admin_all"
on public.requests
for select
to authenticated
using (
  exists (
    select 1
    from public.profiles p
    where p.id = auth.uid()
      and p.role = 'admin'
  )
);
```

```sql
create policy "requests_update_admin_all"
on public.requests
for update
to authenticated
using (
  exists (
    select 1
    from public.profiles p
    where p.id = auth.uid()
      and p.role = 'admin'
  )
)
with check (
  exists (
    select 1
    from public.profiles p
    where p.id = auth.uid()
      and p.role = 'admin'
  )
);
```

If the admin UI ever inserts requests using Supabase mode (it has `createRequest()`), add:

```sql
create policy "requests_insert_admin_all"
on public.requests
for insert
to authenticated
with check (
  exists (
    select 1
    from public.profiles p
    where p.id = auth.uid()
      and p.role = 'admin'
  )
);
```

#### 5.3 Admin policies for fees

```sql
create policy "fees_select_admin"
on public.fees
for select
to authenticated
using (
  exists (
    select 1
    from public.profiles p
    where p.id = auth.uid()
      and p.role = 'admin'
  )
);
```

Upsert in Supabase is typically INSERT or UPDATE depending on existence. Add both:

```sql
create policy "fees_insert_admin"
on public.fees
for insert
to authenticated
with check (
  exists (
    select 1
    from public.profiles p
    where p.id = auth.uid()
      and p.role = 'admin'
  )
);
```

```sql
create policy "fees_update_admin"
on public.fees
for update
to authenticated
using (
  exists (
    select 1
    from public.profiles p
    where p.id = auth.uid()
      and p.role = 'admin'
  )
)
with check (
  exists (
    select 1
    from public.profiles p
    where p.id = auth.uid()
      and p.role = 'admin'
  )
);
```

### 6) Validate end-to-end in the browser (what “good” looks like)

After applying the SQL fixes:

1. Hard refresh the admin panel.
2. Log in as the admin email/password (Supabase Auth credentials).
3. Navigate to a protected route (e.g., `/dashboard`).
4. In console:

```js
window.__ADMIN_DEBUG()
```

Expected:
- `uid` is not null.
- `role` is `admin` (or at minimum it reflects admin after the profile fetch).
- `decision` becomes `allow`.
- The Users page loads, and you can approve mechanics (updates profiles).
- Requests and Fees pages load without RLS errors.

## Code touchpoints to update if needed (and why)

The code is already structured to support the intended model (`profiles.id = auth.uid()`), but these are the key places to inspect if behavior still diverges.

### 1) `src/services/dataService.js` (Supabase profile provisioning and reads)

Relevant functions:
- `dataService.getCurrentProfile()` reads `profiles` by `id = user.id`. This requires `profiles_select_own`.
- `supaGetUserRole()` reads role/approved and attempts to insert a default profile if missing. This requires `profiles_insert_own` to succeed.
- `dataService.listUsers()` selects all profiles. This requires `profiles_select_admin_all`.
- `dataService.approveMechanic()` updates a profile row. This requires `profiles_update_admin_all`.

If admin role checks fail because profile insert is blocked, the fix is normally in RLS (policies above), not in React code. The current code already prefers `id`-based matching, which is correct.

### 2) `src/routes/RequireAuth.js` (gate + diagnostics)

This is the enforcement point for all protected routes. If users claim they are admin but get blocked, the debug panel and `window.__ADMIN_DEBUG()` can narrow down which layer is failing:
- UID null means auth session is missing.
- UID set but role unknown means profile fetch failed/blocked.
- UID set and role != admin means the profile row is present but not configured for admin.

If you ever want to disable the debug panel in non-production builds, the visibility check is:

```js
process.env.NODE_ENV !== "production" && process.env.REACT_APP_NODE_ENV !== "production"
```

### 3) `src/pages/LoginPage.js` (admin-only constraint)

Login explicitly rejects non-admin users immediately after `dataService.login()`.

If you want to rely purely on `RequireAuth` for the decision (instead of the login page rejecting), you could remove the login page role check, but that would be a product decision. As written, it is consistent: the login page rejects non-admins and the routes are also protected.

### 4) `src/App.js` (top-level routing uses `getCurrentUser()`)

At boot, `App.js` uses `dataService.getCurrentUser()` and then only considers you “authed” if the returned object has `role === "admin"`.

If role resolution is wrong due to missing profile row or blocked RLS, `App.js` will keep sending the user to `/login` on `/`. That is expected and correct for admin-only access.

Therefore, if you see repeated redirects to `/login` after a successful Supabase sign-in, it strongly indicates the profile role read (`profiles.role`) is not returning `admin`.

## Verification SQL snippets (quick checks)

### Check current user’s profile row by UID

If you know the UUID:

```sql
select id, email, role, approved
from public.profiles
where id = '00000000-0000-0000-0000-000000000000';
```

### Check which policies exist (per table)

```sql
select schemaname, tablename, policyname, permissive, roles, cmd
from pg_policies
where schemaname = 'public'
  and tablename in ('profiles', 'requests', 'fees')
order by tablename, policyname;
```

### Confirm RLS status

```sql
select relname as table, relrowsecurity as rls_enabled
from pg_class
where relname in ('profiles', 'requests', 'fees');
```

## Safety notes

### Do not use service role key in the browser

This project uses `REACT_APP_SUPABASE_KEY` to create the Supabase client in the browser (`createClient(url, key)`). This key should be the **anon** key. Admin authorization must be enforced by RLS/policies, not by shipping privileged credentials to the client.

### Keep `profiles.id` aligned with `auth.users.id`

All policies and code assume `profiles.id = auth.uid()` for the signed-in user. If your schema stores profiles under a different identifier, you must adjust either:
- schema and backfill to match `auth.users.id`, or
- code and policies to match the alternative model

This repo currently uses the `auth.uid()` alignment model.

## If admin access still fails after these steps

If the gate still denies access, capture:
1. Output of `window.__ADMIN_DEBUG()` from the browser.
2. Supabase SQL results for:
   - `select id, email from auth.users where lower(email)=lower('<admin email>');`
   - `select id, email, role from public.profiles where id='<auth user id>';`
3. The error string shown in the UI (RequireAuth “Access restricted” detail).

From those three, the remaining cause is almost always one of:
- mismatch between `auth.users.id` and `profiles.id`
- missing `profiles_select_own` policy
- profile row exists but role not set to `admin`
