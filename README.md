# Barbet Route Planner

A single-file web tool for planning field sales/delivery routes: pulls Barbet's
live retailer feed, lets you upload your own prospect lists, finds accounts
near a route or destination, and builds a Google Maps–ready "Road Day."

Everything lives in **one file**: `barbet-route-planner.html`. No build step,
no dependencies to install — it's plain HTML/CSS/JS that runs directly in a
browser.

## How it's hosted

This repo is meant to be connected to Netlify for auto-deploy:

1. Push this repo to GitHub (ideally under a shared company account/org, not
   a personal one, so it survives staff turnover).
2. In Netlify: **Add new site → Import an existing project → GitHub** → pick
   this repo. Build command: none. Publish directory: `/` (repo root).
3. Every push to the main branch auto-deploys. That's the whole pipeline.

## How data storage works

There are three kinds of data, and they're **not** all treated the same way:

| Data | Where it lives | Shared across devices? |
|---|---|---|
| Retailer feed | Fetched live from Barbet's Storemapper feed on every page load | N/A — always current, not stored |
| Uploaded accounts | Browser localStorage, **optionally synced to Supabase** | Only if Supabase is configured (see below) |
| Road Day (today's planned stops) | Browser localStorage only | No — intentionally personal/per-device |
| Geocode cache | Browser localStorage only | No — a nice-to-have speed optimization, not shared |

**Uploaded accounts are the one thing worth making shared**, since the whole
point of "upload once" is that everyone using the tool should see the same
list. That requires a small free Supabase project.

### Enabling shared uploads (Supabase)

1. Create a free project at [supabase.com](https://supabase.com) (use the
   company email/account, not a personal one).
2. In the Supabase SQL editor, run:

   ```sql
   create table shared_uploads (
     id text primary key default 'main',
     data jsonb not null default '[]'::jsonb,
     updated_at timestamptz default now()
   );
   insert into shared_uploads (id, data) values ('main', '[]');

   alter table shared_uploads enable row level security;
   create policy "Allow anon read" on shared_uploads for select using (true);
   create policy "Allow anon write" on shared_uploads for update using (true);
   ```

3. In **Project Settings → API**, copy the **Project URL** and the
   **anon public key**.
4. In `barbet-route-planner.html`, find this block near the top of the
   `<script>` section and fill in the two values:

   ```js
   const SUPABASE_URL = '';       // e.g. 'https://xxxxxxxx.supabase.co'
   const SUPABASE_ANON_KEY = '';  // the "anon public" key
   ```

5. Push the change. That's it — the tool now pulls the shared upload list on
   every load, and pushes any change (new upload, delete, toggle) back to
   Supabase automatically. If it can't reach Supabase for any reason, it
   silently falls back to that device's local copy rather than breaking.

**Security note, worth understanding before turning this on:** the tool is
deliberately open with no login, so the anon key above allows *anyone* with
the site link to read and write the shared upload list — not just view it.
That's consistent with how the rest of the tool works (no login anywhere),
but it does mean there's no access control on who can add/delete shared
uploaded accounts. If that ever becomes a problem, the fix is adding some
form of auth in front of the writes — a bigger change, not needed today.

## Key things a future maintainer should know

- **Everything is in one `<script>` tag**, organized top-to-bottom roughly
  as: state/config → map setup → uploads (drop zone, parsing, geocoding) →
  Road Day (list, export, optimize) → main search logic. Search for the
  `// ----` comment banners to jump between sections.
- **External services used** (all free, no API keys except optionally
  Supabase):
  - Barbet's Storemapper feed (retailer data)
  - OSRM's public routing server (`router.project-osrm.org`) — driving
    distance/time/route geometry
  - Nominatim (`nominatim.openstreetmap.org`) — geocoding, rate-limited to
    ~1 request/second by its usage policy. This is the main speed
    bottleneck on large uploads; see in-code comments near `geocode()`.
  - Leaflet + OpenStreetMap tiles — the map itself
- **MAX_CANDIDATES**, **MAX_PER_LEG**, and the detour-radius input are the
  main "tuning knobs" if search behavior ever needs adjusting.
- If Barbet's retailer feed URL or account ID ever changes, update
  `STORE_FEED_URL` near the top of the script.

## Local development

There isn't really a "dev mode" — just open the HTML file directly in a
browser, or serve the folder with any static file server, and edit away.
