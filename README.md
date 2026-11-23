# PLUGGED — EV charging discovery and route planning

PLUGGED is an India-focused EV charging discovery and route-planning app. It uses OpenStreetMap for charger discovery, MongoDB configured by the backend-only `MONGODB_URI` for saved station and user records, and OSRM for road routes.

## Features

- Leaflet maps with road, satellite, and hybrid styles; optional Google Maps JavaScript map for Google Places results.
- Paginated station search by current map bounds, with server-side city, operator, connector, and text filters.
- On-demand OpenStreetMap charger search around the current map view. Results are saved to MongoDB and link to their OSM records.
- EV compatibility catalogue and connector filtering for reported OSM connector data.
- Road-trip planner with location search, map-selected waypoints, and OSRM route distance and duration.
- Station details show source-reported connector type, AC/DC, power, ports, hours, access, payment, and price only when tagged. Unknown station/community details are explicitly shown as unavailable; connector status is not presented as live unless its source reports it.
- The station-details drawer has a Back control and browser-history support; closing it preserves the current map view and filters.
- Station provenance, OSM attribution, source/import timestamps, coordinates, and linked provider photos are shown in the details drawer; photos lacking explicit attribution, source URL, or license are not displayed, and missing/broken photos use a neutral “Photo unavailable” placeholder.

The backend persists OpenStreetMap station records and nearby OSM amenities when returned. It supports account registration/sign-in with hashed passwords, short-lived HttpOnly sessions, sign-out, authenticated station favorites, selected vehicles, and saved trip snapshots. Saved trips contain their locations and stops; opening one recalculates the route using current routing data. Email verification, password recovery, check-ins, and station submissions are not implemented. The app does not provide a preloaded nationwide station inventory.

## Project structure

```
project/
├── backend/       # Express/TypeScript API and MongoDB station persistence
├── frontend/      # React/Vite web app
└── package.json   # Monorepo development/build/test scripts
```

## Setup

Install dependencies:

```bash
npm install
npm install --prefix backend
npm install --prefix frontend
```

Copy `backend/.env.example` to `backend/.env` if needed; never overwrite an existing `.env`. Set `MONGODB_URI` to the Atlas driver URI for the intended database. Replace the username, URL-encoded password, cluster hostname, and database name; special characters in credentials must be percent-encoded. The backend refuses missing, malformed, or placeholder URIs and starts the API only after connecting to MongoDB and initializing the existing station indexes. It never falls back to an in-memory database or drops collections. On shutdown, it closes the HTTP server and Mongoose connection. Connection errors are reported without printing the URI or credentials.

Set `JWT_SECRET` in `backend/.env` to a locally generated private random string of at least 32 characters to enable account, favorites, vehicle, and saved-trip endpoints. Auth sessions use an HTTP-only, SameSite=Lax cookie with a one-hour expiry; production cookies are Secure. Saved trips are limited to 20 per account and store only validated India locations, not route geometry. Email verification and password reset are unavailable until an email delivery provider is implemented and configured. Development auth throttling is process-local; configure an external rate limiter before running multiple production instances.

In Atlas, choose the existing cluster, add your development machine's current public IP to **Network Access** (avoid a permanent `0.0.0.0/0` rule), and use a database user with read/write access only to the PLUGGED database. Once configured and started, verify `http://localhost:5000/api/health`: it returns HTTP 200 and `database.connected: true` only when MongoDB is connected and the station count query succeeds. A disconnected database returns HTTP 503. Test reads with `GET /api/stations?limit=3`; OSM discoveries use source-ID upserts and do not clear existing records.

By default, the backend uses the public OSRM demo server for routing and Nominatim for location search. These shared services have usage policies and no production availability guarantee. Configure `OSRM_BASE_URL` or `NOMINATIM_BASE_URL` in `backend/.env` when using alternative providers.

Google Places charging searches use the Places API (New), are limited to 20 nearby results per search, and are not persisted or cached in MongoDB. Google Places results are displayed on the Google Maps view (not Leaflet) to follow Google's map-content policy. The map search is limited to a user-selected nearby area; this is not a complete India-wide dataset or bulk import.

Google Places requires billing, Places API (New) enabled, a server-restricted API key in `backend/.env`, and a separate browser-restricted Maps JavaScript API key in `frontend/.env.local`:

```env
GOOGLE_MAPS_API_KEY=your_server_restricted_places_api_key
VITE_GOOGLE_MAPS_API_KEY=your_browser_restricted_maps_javascript_api_key
```

Keep the server key private in `backend/.env`; do not put it in frontend code or commit either local environment file. Copy placeholders from `backend/.env.example` and `frontend/.env.example`, then restart both dev servers after adding the keys. Without the browser key, the app stays on Leaflet and Google Places searches are not sent.

Google Places content is shown only during the current search session and is not written to MongoDB. Connector availability, pricing, and operational status are not inferred from place results. Before public deployment, publish Terms of Use and a Privacy Policy that satisfy Google Maps Platform requirements.

## Run locally

In separate terminals:

```bash
npm run dev:backend
npm run dev:frontend
```

The frontend runs at `http://localhost:5173` and the backend at `http://localhost:5000`.

MongoDB can run locally at `mongodb://127.0.0.1:27017/plugged_india` or through Atlas. Keep the URI and `JWT_SECRET` only in `backend/.env`; `backend/.env.example` shows the local default and an Atlas template. A database name explicitly present in `MONGODB_URI` takes precedence over `MONGODB_DATABASE` so an existing Compass database is not silently switched. The configured local URI currently selects the existing `plugged` database. Run `npm run verify:db` to build the backend and exercise authenticated vehicle/trip persistence, ownership checks, reconnect persistence, and index inspection against that configured database; it creates and removes two temporary accounts.

The frontend API base URL is independently configured with `VITE_API_BASE_URL` in `frontend/.env.local` (defaults to `/api`, proxied to the backend by Vite). Do not put `MONGODB_URI` or other server credentials in frontend environment files.

To find chargers, zoom to a city or neighborhood and select **Find more**. Searches use OpenStreetMap's public Overpass service, are limited to 25 km and 50 nearby (100 viewport) charging stations, and depend on community mapping coverage. Cafe, restaurant, toilet, parking, rest-area, fast-food, and food-court amenities are included only when OSM maps them within 300 m of a returned station. Successful search results are upserted by OSM element ID; repeated results refresh only fields explicitly reported by that same OSM element, leaving unavailable fields and unrelated records untouched. The API verifies records are present before returning success.

Retrieve saved stations with `GET /api/stations`. Results are paginated (`limit` defaults to 100 and is capped at 100); `skip`, `bbox=west,south,east,north`, `search`, `city`, `state`, `pincode`, `operator`, `connector`, `accessType`, `amenity`, and `minPowerKW` filters are supported. `GET /api/health` reports MongoDB connection state, database name, and station count.

To import real OpenStreetMap results for a chosen India location without opening the frontend, run:

```bash
npm run seed:osm -- <latitude> <longitude> <radius-meters>
```

The radius must be between 1,000 and 25,000 meters. The importer fetches provider data, safely upserts by OSM element ID, and verifies the saved count. It does not generate station records or delete existing data.

For the full India OpenStreetMap extract, download the current `.osm.pbf` file and its matching `.md5` checksum from [Geofabrik's India download page](https://download.geofabrik.de/asia/india.html) into `backend/data/india-latest.osm.pbf` and `backend/data/india-latest.osm.pbf.md5`. The importer verifies the publisher checksum before parsing. Geofabrik's current India file is a large download (about 1.6 GB at the time this was written). You can resume an interrupted download with `curl.exe --continue-at -`. Then inspect candidates without writing to MongoDB:

```bash
npm run import:india -- --dry-run
```

If the candidate summary looks appropriate, import the genuine OSM charging-station nodes and ways:

```bash
npm run import:india
```

You can pass another extract with `--input <path>` (and place its checksum beside it as `<path>.md5`). The importer streams the extract, derives way locations from their actual OSM node coordinates, collects mapped amenity nodes within 300 m, preserves OSM feature edit timestamps when present, includes source IDs and ODbL attribution, rejects invalid coordinates, and writes batches of 250 with bounded retry/backoff. Re-running after an interruption is safe. Existing records are enriched only where fields are missing; already reported values are retained. Matching source IDs are deduplicated, and exceptionally strong within-extract matches (same normalized name/operator/address and connector signature within 3 m) are stored as one station with all source links. Ambiguous or incomplete identities remain separate to avoid merging distinct chargers. The JSON summary reports candidate records, duplicates skipped, existing and enriched records, database writes confirmed by read-back, and fields missing from source data. Read-back confirms persistence only; it does not verify a station's location, details, or operating status. OSM station relations are counted but not imported because their geometry is not resolved by this importer. Add and update records only: the importer never deletes existing station data. See the Geofabrik page for the latest extract date and size and review the OpenStreetMap [ODbL attribution requirements](https://www.openstreetmap.org/copyright) before redistributing OSM-derived data.

## Data coverage and provenance

The map loads saved stations for its current viewport using 100-record pages (up to 100,000 records per request cycle); dense areas show a “zoom in or refine” message if that safety bound is reached. Low-zoom Leaflet markers are clustered locally and expand as the map zooms in. Queries use the existing 2dsphere index and stable pagination. Station searches support text, city, state, postcode, operator, connector, AC/DC type, reported operational status, location quality, access, amenity, and minimum reported power filters. English OSM address/name tags are preferred when present. On station selection, missing or non-English address fields are looked up with Nominatim using English response language; if English details are unavailable, the UI says so instead of displaying a misleading translation. New records store their import time; OSM source edit time is stored separately when supplied. Routine imports do not update independent verification timestamps. Existing populated fields are not overwritten by lower-quality or absent OSM values.

OSM connector power, AC/DC classification, port counts, payment methods, fee status, and nearby amenities are displayed only when corresponding OSM data is returned. User-authored reviews, check-ins, and station reports are not currently backed by a community API and appear as unavailable; the app does not manufacture these details.

OpenStreetMap/Overpass is the only persistent station-data source currently integrated. Its coverage depends on community mapping and is not a complete national inventory. Nearby Google Places results are transient and are never saved. On 2026-10-17, the official [Bureau of Energy Efficiency site](https://beeindia.gov.in/) displayed a technical-upgrade notice, the official [EV Yatra host](https://evyatra.beeindia.gov.in/) could not be reached, and [data.gov.in dataset search](https://www.data.gov.in/search?title=electric%20vehicle%20charging%20stations) returned HTTP 403; no public, documented station-level government download/API or reuse terms could therefore be confirmed. No EV Yatra, government, Open Charge Map, or operator data is imported. No operator API credentials or documented contracts are configured, so there are no operator adapters enabled. Do not bypass access restrictions; add another source only after its official access, attribution, and storage terms are verified. The app does not claim nationwide completeness or fabricate fallback stations.

The Geofabrik India extract checksum-verified and imported on 2026-10-17 contained 602 charging-station features. All 602 source IDs already existed in the MongoDB collection, so the import added zero new station IDs while updating provenance/check time and filling missing reported fields where possible. This is a count of OSM-mapped features in that extract, not evidence that all Indian charging locations are mapped.

### Validate, enrich, import, and export station data

These commands use the existing `MONGODB_URI` database and never drop or reset collections:

```powershell
npm run validate:stations
npm run report:stations
npm run export:stations
npm run enrich:stations
npm run import:stations
```

Validation is read-only. `report:stations` and `export:stations` generate `backend/reports/india_ev_charging_master.csv`, `station_audit.csv`, `station_audit.json`, and `station_quality_report.json` from the current database records. The master export is deterministically ordered with records in the Erode review area first (by recorded place text or the queried Erode-area bounding box), then by recorded state/city/operator/location. This ordering does not assign a city to a station. The audit lists unresolved fields and validation issues per station. Records sharing coordinates remain separate; potential duplicates are flagged for review, not merged. Generated reports contain only database/source record data; no rows are synthesized to reach a target count.

`enrich:stations` reads the checksum-verified India PBF and updates only existing records matched by exact OSM source ID; `import:stations` reads the same extract and can add source-identified OSM records. Both create a pre-write JSONL snapshot of the complete station collection under ignored `backend/data/backups/`, verify its record count before writing, and stop if the snapshot cannot be completed. The snapshot is a station-collection backup, not a whole-database backup. Pass `--dry-run` to either importer to parse and report without connecting or writing, and `--input <path>` to use another PBF with a matching `.md5` file. Back up other application collections separately before any broader database migration.

Location accuracy is distinct from operational status: exact OSM node coordinates are marked Unverified without independent confirmation; way centroids are Approximate; only explicit, documented verification metadata can qualify as Verified. Operational status remains Unknown unless a source reports it. The import/report tools do not geocode a city centre as a station coordinate and do not infer operational status from a mapped charger tag. Import and read-back timestamps are not evidence of independent station verification.

The cited 22,753 number is currently only a user-provided target. Its publication date, reporting period, station/location/charging-point counting method, and operational/public eligibility have not been verified from a dated official station-level dataset. Official BEE/EV Yatra and operator directories were not confirmed to expose an authorized bulk station feed with usable coordinates and storage terms, so they have not been scraped or imported. The report therefore counts actual records and reports the target shortfall without claiming the target is validated or achieved.

## Build and test

```bash
npm run build
npm test
```
