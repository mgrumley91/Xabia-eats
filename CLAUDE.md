# Xàbia Guide — notes for Claude

This repo is our holiday web app for Xàbia (Jávea), Spain. `index.html` is the whole app,
served at https://mgrumley91.github.io/Xabia-eats/.

## Our trip data lives in Supabase

Use the Supabase connector. Project: **mgrumley91@gmail.com** (project id `cffosbepcciwpfrzqpvy`).
Everything is in one table, `public.splitter` (columns: trip, id, kind, data jsonb, updated_at).

- There is one trip: `trip_0k3l3f1m5p511x1d0n1e471p1p515m`. Only use rows for that trip.
- `id = 'app'` row: `data->'d'` holds
  - `spots`: our apartment (`kind` = 'home') and friends' places such as Wass's (`kind` = 'friends'), with name, lat, lng
  - `places`: saved restaurants, cafés and bakeries (name, area, tags, note, menu, michelin, rating, addr, lat, lng, ig)
    plus `fav` (favourite) and `ours`, our own 0–5 rating: 5 outstanding (rare), 4 superb, will go again,
    3 good for the area, 2 nothing special but would eat again, 1 only if no other option, 0 don't go back
    (never recommend a 0). (`rating` is Google's.)
  - `plan`: the whole-trip plan, place ids in order · `dayPlans`: today's plan by date (`{"2026-11-04": {ids, done}}`)
  - `visits`: where we went (name, placeId, at, how, meal = breakfast/coffee/lunch/dinner) · `prefs`: food preferences
  - sports places (golf, tennis, padel, gym, water sports) are in `places` too, tagged Sport, Golf, Tennis, Padel, Gym or Watersports
  - `trip` dates and `holidays`
- `kind = 'meal'` rows: bills in the trip tally. `data` has place (the restaurant, once confirmed; `billName` is the
  name printed on the bill and `placeId` links to `places`), addr, at, cur and `net` in cents
  (negative = we owed Wass, positive = Wass owed us), plus `detail` with items
  (name, orig, qty, c in cents, side us/wass/shared, ws = Wass's share), payer, paid and totals.
- `data->'d'->'parties'`: the families/groups on the trip, who also split bills (id, name, `people` = their names,
  heads, `stay` = the id of the spot they're staying at, optional monzo / revolut usernames). The first two always have
  ids `us` and `wass`. `data->'d'->'diet'` lists dietary needs (e.g. Vegetarian, Nut allergy): respect them in every
  recommendation.
- "Everyone picks" meals have `group = true`, `payer` (a household id) and `shares` (household id → cents of
  the bill that household owes). Everyone except the payer owes the payer their share. Their `detail.parties`
  and `detail.t.per` show who picked what.
- Meals with `settle = true` are payments: `from` paid `to` `amt` cents, with `ref` (payment reference, e.g. XE-K7Q2X)
  and `forMeal` (the id of the bill it paid, if any). Bills carry the same `ref`.
- `kind = 'bill'` is the bill being split right now; `kind = 'claim'` rows are each household's picks for it.
  Ignore `kind = 'photo'` rows.
- `at` values are milliseconds since 1970: show dates and times in Europe/Madrid.

## Sending recommendations to the app ("Claude's picks")

When we ask for recommendations, also save them so the app shows them on the Home map as numbered
purple pins we can confirm. Write one row (it replaces the previous picks):

```sql
insert into public.splitter (trip, id, kind, data, updated_at)
values ('<current trip>', 'suggest', 'suggest', jsonb_build_object(
  'at', (extract(epoch from now()) * 1000)::bigint,
  'title', 'Lunch near the old town',
  'items', jsonb_build_array(
    jsonb_build_object('placeId', '<id from places>', 'name', 'La Cantina de Jávea', 'why', 'Rice by the fish market, open till 16:00'),
    jsonb_build_object('name', 'New place', 'lat', 38.79, 'lng', 0.17, 'why', 'Not saved yet, well reviewed')
  )), now())
on conflict (trip, id) do update set data = excluded.data, kind = excluded.kind, updated_at = now();
```

- For a day out or route (a bike ride, a walk, a tour), put the stops in the order to visit them and add
  `"mode": "bicycling" | "walking" | "driving"` and a one-line `"summary"` next to `title`. We can then start it
  as a trip or save it for later in the app (saved trips live in `data->'d'->'outings'`).
- Use 1–5 items, best first. Prefer places already in `places` (give `placeId`); for a new place give `name`,
  `lat`, `lng` and, if you know it, `gid` (its Google place id).
- `why` is one short line. Check opening hours for when we're going before recommending.
- Then tell us: "They're on your Home map."

## Draft trips ("Plan with Claude")

When we ask you to plan a day out or a trip, put it in the app as a **draft**, not a saved trip. We review it under
Plan, ask you for changes in the chat, and tap Save in the app when we're happy.

- The draft is the `suggest` row (see "Sending recommendations" above) with `"mode"` set to `bicycling`, `walking` or
  `driving`, a `"title"`, a one-line `"summary"`, and the stops in visiting order in `items` (up to 10). Each item can
  also have `"time"`, e.g. `"10:30"`. Check opening hours for the day we're going.
- When we ask for a change, write the whole updated draft again to the same `suggest` row (new `at`, all stops, in
  order). The app swaps in the new version straight away.
- Don't add it to `outings` (Saved → Trips) yourself; we save it from the app. Only edit `outings` when we ask you to
  change a trip we've already saved (see below).
- Afterwards, check the row saved and tell us: "Your draft is under Plan." If you can't write to Supabase in this
  chat, say so plainly instead of saying it's there.

## Saved trips (routes)

Our saved trips are in `data->'d'->'outings'`: each has `id`, `title`, `summary`, `mode` (bicycling / walking /
driving) and `stops` in order (`name`, `why`, `lat`, `lng`, `placeId` if it's one of our places). The trip we're on
right now is `data->'d'->'activeOuting'` (`id`, and `done` = indexes of stops already visited).

When we ask to change a trip (e.g. "add a lunch stop halfway"), work out the new list of stops (check opening hours
for when we'll be there) and save it like this. Removing `route` makes the app redraw the roads:

```sql
update public.splitter
set data = jsonb_set(
      jsonb_set(data, '{d,outings}', (
        select jsonb_agg(case when o->>'id' = '<trip id>'
          then jsonb_set(o - 'route', '{stops}', '<new stops as a JSON array>'::jsonb) else o end order by ord)
        from jsonb_array_elements(data->'d'->'outings') with ordinality as t(o, ord))),
      '{updated}', to_jsonb((extract(epoch from now()) * 1000)::bigint)),
    updated_at = now()
where trip = 'trip_0k3l3f1m5p511x1d0n1e471p1p515m' and id = 'app';
```

Then tell us what changed. The app shows it on the trip page (Saved → Trips) and on Home if the trip is running.

## Adding a place's Instagram

When we ask you to find Instagram accounts, search the web for each place's official account in Xàbia/Jávea.
Only save a handle you've actually seen on an instagram.com profile for that exact business; list the ones
you couldn't confirm instead of guessing. Save each one like this (also bumps `updated` so the phones pick it up):

```sql
update public.splitter
set data = jsonb_set(
      jsonb_set(data, '{d,places}', (
        select jsonb_agg(case when p->>'id' = '<place id>' then p || jsonb_build_object('ig', '@<handle>') else p end order by ord)
        from jsonb_array_elements(data->'d'->'places') with ordinality as t(p, ord))),
      '{updated}', to_jsonb((extract(epoch from now()) * 1000)::bigint)),
    updated_at = now()
where trip = 'trip_0k3l3f1m5p511x1d0n1e471p1p515m' and id = 'app';
```

Then tell us which places you added and which you couldn't find.

## Homes we're looking at (property search)

`data->'d'->'homes'`: apartments and villas we're considering buying. Each has `id`, `name`, `kind`
(Apartment / Villa / Townhouse / Plot), `status` (look = considering, view = viewed, short = shortlist, no = ruled out),
`price` (euros), `beds`, `url` (the listing), `note`, `lat`, `lng`, and `risk` once the app has checked the
official Generalitat Valenciana maps: `flood` (0 = not in a mapped flood zone; 4–9 = PATRICOVA hazard level 1–6,
i.e. the value minus 3; 10 = landform suggests flooding), `zif` (0, 500 or 100: within that many metres of forest,
the wildfire interface zone), `burnOn` / `burnNear` (years a wildfire burned the plot / within about 500 m),
`coast` (PATIVEL coastal-protection zone layer ids; empty = outside).
When comparing homes, use these and say they are planning maps, not a survey.

## Rules

- Read only, except the `suggest` row and Instagram handles above, or when we ask you to change something.
- Never repeat the Google API key stored in the app row, the bill-reading key in `public.secrets`, or any Supabase keys.
- Bill photos are read by the `gemini` Edge Function, which takes the key from `public.secrets`; it is never stored in the app row.
- When answering "where did we go…", check both `visits` and `meal` rows.
- Changing the app: edit `index.html`, bump `APP_VERSION`, and push to `main`;
  the app shows an "Update" bar on our phones.
