# RailScope - OPEN TASKS (checklist index, 2026-10-06)

This file is a notes-style log. The checklist below is the quick index of what is still open; details live in the sections it points to. `[!]` = needs a decision or a human check first.

- [ ] Add `functions/api/vehicles.ts` (port the handler from `server.ts`), add a root `wrangler.toml` with `nodejs_compat`, then point `VEHICLES_API_BASE` back at same-origin
- [ ] Lazy-load `Map.tsx` and `Feed.tsx` with `React.lazy` (bundle is one ~2.4 MB chunk)
- [ ] Cull vehicle markers outside the padded map bounds (1000-2000 DOM markers on phones)
- [ ] Keep the map mounted across tab switches (it loses pan/zoom and restarts polling)
- [ ] Stop `stationMarkers` rebuilding all ~600 markers when a popup opens
- [ ] Add a fetch timeout and a "waking up" indicator for the Render cold start (~50 s on first map load)
- [ ] Wire the Service Details "Alerts" tab to the TfNSW `/v1/gtfs/alerts` feed (currently hardcoded "No Active Alerts")
- [ ] Fix the theme flash on first render (guard `resolvedTheme` with a mounted check)
- [ ] Route-shape following on the map (needs GTFS `shapes.txt` or a shapes endpoint), a "Show all services" toggle, and a "View in map" button on Service Details
- [ ] Fix the `vehicleId` fallback (`veh-${index}` uses different index bases in two places)
- [ ] Hygiene: remove unused deps (`@google/genai`, `react-window`, `react-virtualized-auto-sizer`), move `shadcn` out of `dependencies`, delete scratch files (`test-gtfs.ts`, `trip.json`, `find_stops.cjs`)
- [ ] Localise the splash/header logos from `i.ibb.co` into `public/`; pick one source for the header download links (hardcoded `fushigi` tag vs README `/releases`)
- [!] Decide whether Cloudflare is being retired; if so point web `API_BASE` at Render and drop `functions/`
- [!] Admin task: clear the Network Feed (tap the header 7 times, then "Clear Feed"), or ask for a one-shot Admin SDK script

---

# RailScope — TODO

## 🗺️ Live service simulation on the map (BIG — own work session)

DONE (basic): `Map.tsx` now eases each vehicle marker from its previous fix
toward the latest one via a rAF loop (straight-line lerp over ~16s, off-screen
markers skipped, big jumps snap). Trains glide instead of teleporting.

Still to do:
- Route-shape following: snap the train to the line's polyline and advance along
  it instead of straight-line lerp — needs GTFS `shapes.txt` / a shapes endpoint.
  Looks much better on curves and around stations.
- Add a **"Show all services"** toggle on the map (default to a cleaner/filtered
  view, expand to the full network on demand).
- On the **Service Details** page (`LiveFeed.tsx`) add a **"View in map"** button
  that deep-links to the map focused on that specific train (pan/zoom, highlight,
  hide the rest). Needs the selected-train id lifted to `App.tsx`, and a check
  that Trip Planner's `RealtimeTripId` / `gtfsTripId` line up with the vehicle
  feed's `trip.tripId`.

## 🧹 Clear the Network Feed

The feed is Firestore-backed; there's already a hidden admin tool for this:
sign in as the admin account, tap the **"Network Feed"** header 7 times, then hit
**"Clear Feed"**. No code change needed — just needs to be run by the admin.
(If we want a one-shot script instead, add a small Firebase Admin SDK script.)

## ☁️ Backend hosting decision

- Native (Android / Electron) builds now point at `https://railscope-l98d.onrender.com`
  (see `src/lib/config.ts`).
- The **web** app (`railscope.pages.dev`) uses its own same-origin Cloudflare Pages
  Functions (`functions/api/*`) for stations/journeys/departures/trip_details —
  no cold start, keep these.
- Decide whether Cloudflare is being retired too. If so, point web `API_BASE` at
  Render as well and drop `functions/`.

## ⚠️ Missing Cloudflare function: `/api/vehicles`

There is no `functions/api/vehicles.ts`, so on `railscope.pages.dev` the live
vehicle map used to get the SPA HTML back and silently render nothing. Stopgap:
`VEHICLES_API_BASE` in `config.ts` now routes the web map's vehicle polling to
the Render backend (which has the endpoint). Downside: first load of the Map tab
can cold-start Render (~50s).

Proper fix: add `functions/api/vehicles.ts` porting the `/api/vehicles` handler
from `server.ts`. It decodes GTFS-Realtime protobuf via `gtfs-realtime-bindings`,
so the Pages project needs `nodejs_compat` — add a root `wrangler.toml` with
`compatibility_flags = ["nodejs_compat"]` and a recent `compatibility_date`, then
switch `VEHICLES_API_BASE` back to same-origin.

## 🧰 Code-health backlog (from the reviews)

### Performance / scale
- **Bundle is one ~2.4 MB chunk** (~690 KB gzip) — MapLibre GL added ~800 KB.
  Top priority: lazy-load `Map.tsx` (Leaflet + MapLibre) and `Feed.tsx`
  (Firebase) behind `React.lazy`. Map + Feed are separate tabs — ideal splits.
- **Vehicle markers are not viewport-culled.** Daytime can be 1000-2000 bus
  markers, each a glowing DOM DivIcon — heavy on phones. Render only markers
  within the (padded) map bounds; track bounds on `moveend`.
- **Map tab remounts on every tab switch** — loses pan/zoom, restarts the poll
  + rAF, re-flies to origin. Keep `<TrainMap>` mounted and toggle with `hidden`.
- **`stationMarkers` memo rebuilds all ~600 markers when any popup opens**
  (`openStationId` is a dep) and re-`setLatLng`s them (new `position` array each
  time). Precompute stable `[lat,lng]` tuples; lift `active` out of the memo.
- `renderToStaticMarkup` runs per unique icon on first load — ~100 calls for
  distinct bus routes = a small hitch. Could build the SVG strings directly.

### Correctness / robustness
- **`vehicleId` fallback `veh-${index}`** uses different index bases in
  `applyPositions` (entities) vs render (filteredTrains). Vehicles with no
  `id`/`tripId` (rare) won't animate. Use a content hash or drop them.
- **Map tab first load can hang ~50s** on a Render cold start (free tier) with
  no fetch timeout / "waking up" indicator.
- **`LiveFeed` "Alerts" tab is hardcoded** "No Active Alerts" — never wired to
  the TfNSW `/v1/gtfs/alerts` feed.
- **Theme FOUC**: `resolvedTheme` is undefined on first render → map flashes the
  day raster before the dark vector loads. Guard with a mounted check.
- **`functions/api/vehicles.ts` still missing** (see section above) — web map
  depends on the cold-starting Render backend.

### Hygiene
- **Unused / misplaced deps**: `@google/genai`, `react-window`,
  `react-virtualized-auto-sizer` unused; `shadcn` (a CLI) is in `dependencies`.
- **Scratch files committed**: `test-gtfs.ts`, `trip.json`, `find_stops.cjs`.
- **Header download links** hardcode the `fushigi` release tag (`App.tsx`);
  README points at `/releases`. Pick one.
- **Logos load from `i.ibb.co`** (splash + header) — third-party dependency for
  core branding. Localise into `public/`.
- Note: `firebase-applet-config.json` in git is **fine** — Firebase web config
  is public by design; security is the Firestore rules + authorized domains.

### Fixed in this review pass
- localStorage crash guard (corrupt JSON / disabled storage no longer white-screens)
- Feed listener `limit(100)`; `handleClearFeed` batch chunking
- 8s cache on `/api/vehicles` + 8s timeouts on every TfNSW call (server + CF)
- `startServer().catch()`; `String(routeId)` guard
- Origin station no longer double-rendered on the map
- Search race guard; virtualizer `estimateSize`
- iOS: search input attributes, `viewport-fit=cover`, `h-screen` fallback
- WebGL fallback to raster; local favicon
