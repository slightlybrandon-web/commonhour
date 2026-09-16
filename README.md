# Commonhour — setup and deployment (fresh project)

Single-file site (`index.html`), backed by its own dedicated Supabase
project — separate from TimeBlocked or any other app you've built.

## 1. Create a new Supabase project

1. Go to https://supabase.com/dashboard and click **New project**.
2. Give it its own name (e.g. "commonhour"), pick a region, wait ~1 minute
   for it to provision.
3. This is a completely separate database from any other project — nothing
   here can affect TimeBlocked or vice versa.

## 2. Run this SQL (SQL Editor → paste the whole block → Run)

This is the complete, current schema in one script — no need to run
anything else afterward.

```sql
create table events (
  id text primary key,
  name text not null,
  dates jsonb not null,
  start_hour int not null,
  end_hour int not null,
  slot_min int not null,
  timezone text not null default 'UTC',
  location text,
  organizer_name text,
  organizer_email text,
  user_id uuid references auth.users(id),
  created_at timestamptz default now()
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
-- anonymous share-link flow reads through the two functions below, and
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
  timezone text, location text, organizer_name text, user_id uuid, created_at timestamptz
)
language sql
security definer
set search_path = public
as $$
  select id, name, dates, start_hour, end_hour, slot_min, timezone, location,
         organizer_name, user_id, created_at
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
```

## 3. Enable magic-link sign-in

In your new project: **Authentication → Providers → Email** — confirm Email
is enabled (it is by default). Then **Authentication → URL Configuration**:

- **Site URL**: your Commonhour Netlify URL (once you have one — see below)
- **Redirect URLs**: the same URL

You can come back and set these once you know the deployed URL — the app
works before this is set, but magic-link emails won't redirect correctly
until it is.

## 4. Configure the site

Open `index.html`, find these two lines near the top, and paste in your new
project's URL and anon/publishable key (Project Settings → API Keys):

```html
<script>
  window.SUPABASE_URL = "PASTE_YOUR_SUPABASE_URL_HERE";
  window.SUPABASE_ANON_KEY = "PASTE_YOUR_SUPABASE_ANON_KEY_HERE";
</script>
```

## 5. Deploy

Create a **new** Netlify site for this (don't reuse your TimeBlocked site) —
drag this folder onto https://app.netlify.com/drop, or connect a fresh
GitHub repo. Once you have the resulting URL, go back to step 3 and set it
as the Site URL / Redirect URL in Supabase.

## Still placeholders — wire these up when ready

- `window.DONATE_URL` near the top of `index.html` — create a Stripe
  Payment Link (Stripe Dashboard → Payment Links, no code) for a flexible
  one-time amount, and paste the URL in. The donate button is hidden until
  this is set.
- The "Sponsored" text inside the `DonateAffiliateStrip` component in
  `index.html` — replace with real affiliate content once a partner is in
  place.
- Both the donate button and the affiliate slot are wired to disappear once
  a `isPaid` flag is true — that's hardcoded `false` everywhere for now,
  since the actual paid tier (Stripe subscriptions) isn't built yet.

## What's built vs. what's next

**Built:** anonymous quick polls (12-response cap), free accounts via magic
link (unlimited responses, permanent "My polls" history, automatic admin
recognition on your own plans), timezone-aware scheduling, location field,
calendar invite creation (Google Calendar, Outlook, `.ics`) — available to
every tier, always.

**Not yet built:** paid subscriptions (Stripe checkout + webhooks + feature
gating), calendar auto-fill (Google/Outlook OAuth), required/optional
attendees, deadlines + auto-nudges, time+location joint polling. Each of
these is a real, separate project.

## Known limitations (by design, for now)

- No password reset flow needed — magic link is the entire auth model.
- Anyone with a plan's code can still read and respond to it (no accounts
  required) — same trust model as When2Meet or Doodle.
- Email-based admin recognition for anonymous plans is weaker than the
  account-based version — anyone who knows or guesses the organizer's email
  can claim admin on a plan that was never linked to an account.
