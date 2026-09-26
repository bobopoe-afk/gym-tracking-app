# Accounts & sync setup (Supabase)

The app can keep each person's training log in their own account, so it's the
same on every device they sign in on, and friends can sign up for their own.
It uses [Supabase](https://supabase.com) (free tier) for sign-in and storage.
Until it's configured, the app works as before: one log, saved in the browser.

## 1. Create the project

1. Go to https://supabase.com and sign up (signing in with GitHub is fine).
2. **New project** → any name (e.g. `training-log`), set a database password
   (keep it somewhere safe; the app doesn't need it), region close to you
   (e.g. London). Under **Security**, keep **Enable Data API** ticked; the
   other two boxes can be left as they are. Wait a minute or two for it to
   finish setting up.

## 2. Create the table

**SQL Editor** → **New query** → paste this → **Run**:

```sql
create table public.training_logs (
  user_id    uuid primary key default auth.uid() references auth.users on delete cascade,
  data       jsonb not null,
  rev        bigint not null default 0,
  updated_at timestamptz not null default now()
);

alter table public.training_logs enable row level security;

-- each account can only ever see and change its own log
create policy "read own log"   on public.training_logs for select using (auth.uid() = user_id);
create policy "create own log" on public.training_logs for insert with check (auth.uid() = user_id);
create policy "update own log" on public.training_logs for update using (auth.uid() = user_id) with check (auth.uid() = user_id);

-- signed-in users may use the table (the policies above still limit them to their own row);
-- needed if "Automatically expose new tables" was unticked when creating the project
grant select, insert, update on public.training_logs to authenticated;
```

## 3. Point sign-in emails at the website

**Authentication → URL Configuration**:

- **Site URL**: `https://bobopoe-afk.github.io/gym-tracking-app/`
- **Redirect URLs** → add `https://bobopoe-afk.github.io/gym-tracking-app/`

Optional: **Authentication → Sign In / Providers → Email** → turn off
**Confirm email** if you'd rather new accounts work straight away without a
confirmation email. (Supabase's built-in email sender only allows a few emails
an hour, so this also avoids hitting that limit.)

## 4. Connect the app

**Project Settings → API** (or **API Keys**) and copy:

- the **Project URL** (`https://xxxx.supabase.co`)
- the **anon / public** key

Put them in `CLOUD` near the top of the storage code in `index.html`:

```js
const CLOUD = {
  url: 'https://xxxx.supabase.co',
  anonKey: 'eyJ...',
};
```

The anon key is designed to be public: it only lets people sign in, and the
row-level-security policies above decide what each account can touch. Never
put the **service_role** key in the app.

## How it behaves

- First sign-in on a device that already has a log offers to use that log for
  the account; otherwise a new account starts empty (with the exercise library).
- Every change saves on the device immediately and syncs to the account a
  moment later. Other devices pick changes up when they're opened or brought
  back to the front, and every minute while open.
- Offline, the app keeps working from the device's copy and syncs when the
  connection's back. If two devices both change things before syncing, the
  changes are merged (nothing logged is lost; a session deleted on one device
  while the other was offline can come back).
- **Sign out** removes that account's copy from the device.
- "Use without an account" keeps a device local-only; the footer offers
  "Sign in to sync" later.
- Free Supabase projects pause after a week with no activity; open the project
  dashboard to wake it if that ever happens.
