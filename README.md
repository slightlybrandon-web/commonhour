# Commonhour

A group scheduling app: people mark their availability on a shared grid
with no account required, and the plan's organizer picks a final time and
generates a calendar invite (Google Calendar, Outlook, or `.ics`) with
attendees pre-filled.

Live at **commonhour.io**. Single-file site (`index.html`) — React +
Babel loaded from CDN, no build step — backed by Supabase (Postgres, Row
Level Security, Auth) and Resend for transactional email.

## Setting up a fresh instance

### 1. Create a Supabase project

Go to https://supabase.com/dashboard → **New project**, name it, pick a
region, and wait for it to provision.

### 2. Run this SQL (SQL Editor → paste the whole block → Run)

This is the complete, current schema in one script.

```sql
create table events (
  id text primary key,
  name text not null,
  dates jsonb not null,
  start_hour int not null,
  end_hour int not null,
  slot_min int not null,
  default_duration int,
  timezone text not null default 'UTC',
  location text,
  organizer_name text,
  organizer_email text,
  user_id uuid references auth.users(id),
  created_at timestamptz default now(),
  booked_date date,
  booked_minutes int,
  booked_duration int,
  booked_at timestamptz
);

create table availability (
  id bigserial primary key,
  event_id text references events(id) on delete cascade,
  person_name text not null,
  email text not null,
  slots jsonb not null default '[]',
  updated_at timestamptz default now(),
  unique (event_id, email)
);

create index availability_event_id_idx on availability(event_id);

alter table events enable row level security;
alter table availability enable row level security;

-- Anyone can create a plan or submit a response (no accounts required for
-- the free/no-account tier). Reads are NOT open on the raw tables — the
-- anonymous share-link flow reads through the functions below, and
-- signed-in users get their own scoped policies.
create policy "insert events" on events for insert with check (true);
create policy "insert availability" on availability for insert with check (true);

-- Signed-in users can see plans they created...
create policy "owners read own events" on events
  for select to authenticated
  using (auth.uid() = user_id);

-- ...and plans they've responded to, matched by their account's email.
create policy "responders read events they answered" on events
  for select to authenticated
  using (id in (select event_id from availability where email = auth.jwt() ->> 'email'));

-- Required for the policy above to actually find anything — a signed-in
-- user can read their own response rows (and only their own).
create policy "authenticated read own responses" on availability
  for select to authenticated
  using (email = auth.jwt() ->> 'email');

-- Anonymous plan view: reading a specific plan by its code, and its
-- responses, both go through these functions so a client can never list
-- every plan or every participant's email at once.
create function get_plan(p_id text)
returns table(
  id text, name text, dates jsonb, start_hour int, end_hour int, slot_min int,
  default_duration int, timezone text, location text, organizer_name text,
  user_id uuid, created_at timestamptz
)
language sql
security definer
set search_path = public
as $$
  select id, name, dates, start_hour, end_hour, slot_min, default_duration,
         timezone, location, organizer_name, user_id, created_at
  from events where id = p_id;
$$;
grant execute on function get_plan(text) to anon, authenticated;

create function get_plan_availability(p_id text)
returns setof availability
language sql
security definer
set search_path = public
as $$
  select * from availability where event_id = p_id;
$$;
grant execute on function get_plan_availability(text) to anon, authenticated;

-- Admin recognition for plans created without an account (email confirmation).
create function verify_admin_email(p_id text, p_email text)
returns boolean
language sql
security definer
set search_path = public
as $$
  select exists(
    select 1 from events
    where id = p_id and lower(organizer_email) = lower(trim(p_email))
  );
$$;
grant execute on function verify_admin_email(text, text) to anon, authenticated;

-- Saving availability: handles insert-or-update, and caps anonymous
-- (no-account) plans at 12 respondents. Signed-in-created plans have no cap.
create function save_availability(
  p_event_id text, p_name text, p_email text, p_slots jsonb
)
returns void
language plpgsql
security definer
set search_path = public
as $$
declare
  v_owner uuid;
  v_already_responded boolean;
  v_count int;
begin
  select user_id into v_owner from events where id = p_event_id;
  select exists(select 1 from availability where event_id = p_event_id and email = p_email)
    into v_already_responded;

  if v_owner is null and not v_already_responded then
    select count(*) into v_count from availability where event_id = p_event_id;
    if v_count >= 12 then
      raise exception 'RESPONDENT_CAP_REACHED';
    end if;
  end if;

  insert into availability (event_id, person_name, email, slots)
  values (p_event_id, p_name, p_email, p_slots)
  on conflict (event_id, email) do update
    set person_name = excluded.person_name,
        slots = excluded.slots,
        updated_at = now();
end;
$$;
grant execute on function save_availability(text, text, text, jsonb) to anon, authenticated;

-- Admin-only: persist the organizer's final chosen time. Called the
-- moment they click Google Calendar / Outlook / .ics, not just when they
-- open the plan. Requires a real signed-in session — either the account
-- owns the plan, or the signed-in user's email matches the organizer's.
create function book_plan(
  p_id text, p_date date, p_minutes int, p_duration int
)
returns void
language plpgsql
security definer
set search_path = public
as $$
declare
  v_event events%rowtype;
  v_caller_email text;
begin
  select * into v_event from events where id = p_id;
  if v_event.id is null then
    raise exception 'PLAN_NOT_FOUND';
  end if;

  v_caller_email := auth.jwt() ->> 'email';

  if auth.uid() is null then
    raise exception 'NOT_AUTHORIZED';
  end if;

  if auth.uid() = v_event.user_id
     or (v_event.organizer_email is not null and lower(v_caller_email) = lower(v_event.organizer_email)) then
    update events
      set booked_date = p_date,
          booked_minutes = p_minutes,
          booked_duration = p_duration,
          booked_at = now()
      where id = p_id;
    return;
  end if;

  raise exception 'NOT_AUTHORIZED';
end;
$$;
grant execute on function book_plan(text, date, int, int) to authenticated;

-- Delete a plan. Restricted to `authenticated` only (anon cannot call
-- this at all) — deleting is permanent, so it requires actually being
-- signed in as either the account owner or the organizer's verified
-- email, not just knowing/typing that email into a box.
create function delete_plan(p_id text)
returns void
language plpgsql
security definer
set search_path = public
as $$
declare
  v_event events%rowtype;
  v_caller_email text;
begin
  select * into v_event from events where id = p_id;
  if v_event.id is null then
    raise exception 'PLAN_NOT_FOUND';
  end if;

  v_caller_email := auth.jwt() ->> 'email';

  if auth.uid() = v_event.user_id
     or (v_event.organizer_email is not null and lower(v_caller_email) = lower(v_event.organizer_email)) then
    delete from events where id = p_id;
    return;
  end if;

  raise exception 'NOT_AUTHORIZED';
end;
$$;
grant execute on function delete_plan(text) to authenticated;

-- Daily cleanup. Booked plans expire 10 days after the booked date;
-- never-booked plans expire 30 days after the last candidate date, so
-- abandoned polls don't live forever.
create function cleanup_expired_plans()
returns void
language plpgsql
security definer
set search_path = public
as $$
begin
  delete from events
  where booked_date is not null
    and booked_date + interval '10 days' < now();

  delete from events
  where booked_date is null
    and (
      select max((d)::date) from jsonb_array_elements_text(dates) as d
    ) + interval '30 days' < now();
end;
$$;
```

### 3. Enable pg_cron and schedule the daily cleanup

In the dashboard: **Database → Extensions**, enable `pg_cron` (and
`pg_net` if it's listed separately and not already on). Then, in the SQL
Editor, run once:

```sql
select cron.schedule(
  'cleanup-expired-plans',
  '0 3 * * *',
  $$select cleanup_expired_plans();$$
);
```

To check it's running: `select * from cron.job;`
To check run history: `select * from cron.job_run_details order by start_time desc limit 10;`

### 4. Enable sign-in (password and magic link)

**Authentication → Providers → Email** — confirm Email is enabled (it is
by default). This one toggle covers both sign-in methods the app offers.
Then **Authentication → URL Configuration**:

- **Site URL**: your deployed URL
- **Redirect URLs**: the same URL

Both magic-link and password-reset emails need this set correctly to
redirect back into the app rather than failing silently.

### 5. Set up transactional email (Resend)

Supabase's default email sender has a low rate limit, unsuitable for real
traffic. To use Resend instead: verify a sending domain in Resend,
generate SMTP credentials, and add them under **Authentication → SMTP
Settings** in the Supabase dashboard.

### 6. Configure the site

Open `index.html`, find these lines near the top, and paste in your
project's URL and anon/publishable key (Project Settings → API Keys):

```html
<script>
  window.SUPABASE_URL = "PASTE_YOUR_SUPABASE_URL_HERE";
  window.SUPABASE_ANON_KEY = "PASTE_YOUR_SUPABASE_ANON_KEY_HERE";
</script>
```

### 7. Deploy

Connect the repo to Netlify (or any static host). With Netlify connected
to GitHub, every push to `main` deploys automatically — no separate
deploy step. Once you have a live URL, go back to step 4 and set it as
the Site URL / Redirect URL in Supabase.

## Still placeholders — wire these up when ready

- `window.DONATE_URL` near the top of `index.html` — create a Stripe
  Payment Link (Stripe Dashboard → Payment Links, no code) for a flexible
  one-time amount, and paste the URL in. The donate button stays hidden
  until this is set.
- The "Sponsored" text inside the `DonateAffiliateStrip` component in
  `index.html` — reserved for a Google AdSense slot, currently in
  progress; requires a published privacy policy before Google will
  approve it.
- Both the donate button and the sponsored slot are wired to disappear
  once an `isPaid` flag is true — that's hardcoded `false` everywhere for
  now, since the paid tier doesn't exist yet.

## What's built

Anonymous quick polls (12-response cap); free accounts via password or
magic link with account-owned plans unlimited; permanent "My Plans"
history; delete a plan (account owners directly, anonymous organizers
after signing in); automatic admin recognition on your own plans;
timezone-aware scheduling with a touch-and-mouse-unified drag-to-select
grid; location field; calendar invite creation (Google Calendar, Outlook,
`.ics`) available on every tier, always; server-side booking persistence;
and automatic expiry (10 days after a booked date, 30 days after the last
candidate date if never booked).

## What's next

1. **Google AdSense** — in progress. Needs the ad slot wired in and a
   published privacy policy (drafted, not yet live) before Google will
   approve it.
2. **A paid tier** — not started. Scope and requirements to be provided
   separately before work begins.
3. Longer-term, unscheduled: calendar auto-fill via Google/Outlook OAuth,
   required/optional attendees, deadlines and auto-nudges, joint
   time-and-location polling, and real cold user testing with people
   outside the builder.

## Known limitations (by design, for now)

- Anyone with a plan's code can read and respond to it — no account
  required to participate. Intentional: requiring accounts for every
  respondent would undermine the low-friction pitch of the product.
- An anonymous organizer who never actually signs in (just types a
  matching email) can unlock booking/viewing on their own plan, but
  cannot delete it or get the precise 10-day-post-booking expiry — both
  require a real session. Signing in resolves this.
