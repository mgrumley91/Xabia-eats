# Xàbia Eats — notes for Claude

This repo is our holiday web app for Xàbia (Jávea), Spain. `index.html` is the whole app,
served at https://mgrumley91.github.io/Xabia-eats/.

## Our trip data lives in Supabase

Use the Supabase connector. Project: **mgrumley91@gmail.com** (project id `cffosbepcciwpfrzqpvy`).
Everything is in one table, `public.splitter` (columns: trip, id, kind, data jsonb, updated_at).

- Several `trip` values exist from setup; **our current trip is the one whose `id = 'app'` row
  has the latest `updated_at`.** Only use rows for that trip.
- `id = 'app'` row: `data->'d'` holds
  - `spots`: our apartment (`kind` = 'home') and friends' places such as Wass's (`kind` = 'friends'), with name, lat, lng
  - `places`: saved restaurants, cafés and bakeries (name, area, tags, note, menu, michelin, rating, addr, lat, lng, ig)
  - `plan`: place ids in order · `visits`: where we went (name, placeId, at, how) · `prefs`: food preferences
  - `trip` dates and `holidays`
- `kind = 'meal'` rows: bills in the trip tally. `data` has place, at, cur and `net` in cents
  (negative = we owed Wass, positive = Wass owed us), plus `detail` with items
  (name, orig, qty, c in cents, side us/wass/shared, ws = Wass's share), payer, paid and totals.
- `data->'d'->'parties'`: households splitting bills (id, name, heads, optional monzo / revolut usernames).
  The first two always have ids `us` and `wass`.
- "Everyone picks" meals have `group = true`, `payer` (a household id) and `shares` (household id → cents of
  the bill that household owes). Everyone except the payer owes the payer their share. Their `detail.parties`
  and `detail.t.per` show who picked what.
- Meals with `settle = true` are payments: `from` paid `to` `amt` cents.
- `kind = 'bill'` is the bill being split right now; `kind = 'claim'` rows are each household's picks for it.
  Ignore `kind = 'photo'` rows.
- `at` values are milliseconds since 1970: show dates and times in Europe/Madrid.

## Rules

- Read only, unless we ask you to change something.
- Never repeat the Google API key stored in the app row, or any Supabase keys.
- When answering "where did we go…", check both `visits` and `meal` rows.
- Changing the app: edit `index.html`, bump `APP_VERSION`, and push to `main`;
  the app shows an "Update" bar on our phones.
