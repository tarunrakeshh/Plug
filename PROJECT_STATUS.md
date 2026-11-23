# PLUGGED project status

## Changed files

- `README.md`
- `PROJECT_STATUS.md`
- `backend/src/models/Station.ts`
- `backend/src/services/openStreetMap.ts`
- `backend/src/services/stationPersistence.ts`
- `backend/src/tests/openStreetMap.test.ts`
- `backend/src/tests/stationPersistence.test.ts`
- `frontend/src/App.tsx`
- `frontend/src/components/StationDetails/StationDrawer.tsx`
- `frontend/src/services/api.ts`
- `frontend/src/types/index.ts`
- `frontend/src/utils/externalStation.ts`

## Completed and verified

- Station details show source-attributed information only when reported; unknown values remain explicitly unavailable.
- OSM socket tags are mapped into connector type, AC/DC, power, and port count fields. Payment methods, fee status, opening hours, contact details, and source metadata are preserved when tagged.
- OSM discovery now requests cafes, restaurants, toilets, parking, rest areas, fast food, and food courts with charger results. Amenities are associated only when within 300 m of a returned charger and are not imported as charging stations.
- Source-matched repeat OSM discoveries update fields that the provider actually reports without clearing absent fields or modifying unrelated station records. Station upserts remain keyed by OSM element ID; no collection reset or delete is performed.
- The station drawer has a visible Back arrow. Selecting a station creates one same-URL history entry; browser Back and the drawer Back/close actions close the drawer without changing the map position or filter selections.
- The existing application loaded 602 station records and markers in the browser. A live `GET /api/health` returned healthy with MongoDB connected and `stationCount: 602`.
- Browser check: opened station details from a Leaflet marker; browser Back closed the drawer while preserving the Leaflet map-pane position and filter selections. The Back-arrow close path was also exercised.
- `npm run build`: passed for backend TypeScript and frontend TypeScript/Vite production build.
- `npm test`: passed, 57 tests, 0 failures (latest run).

## Current limitations and actual errors

- OpenStreetMap/Overpass is community-maintained and can time out or rate-limit. A prior live Overpass request timed out; saved MongoDB station results remain available. Coverage is not complete or guaranteed across India.
- Account registration/sign-in/sign-out, current-user retrieval, and Mongo-backed favorites are implemented in code; a real database-backed login/write flow remains unverified. Email, user reviews, check-ins, station reports, and secure photo uploads are not implemented.
- Open Charge Map is not integrated; no OCM API key or licensing configuration is present.
- Google Places remains an optional, transient nearby search. Google Place details are not persisted; no complete station-to-Place matching, nearby-amenity enrichment, or current live Places verification was completed. Google data must remain subject to its storage and attribution restrictions.
- MongoDB connectivity and existing station reads were verified at runtime. The active configured database was not inspected for Atlas-versus-local hosting, and this work did not verify Atlas credentials specifically.
- No database credentials are included in project files or this status report.

## Setup and configuration

- Required to run the backend: `MONGODB_URI` in `backend/.env`. The example file contains a placeholder URI; use the intended MongoDB deployment.
- Optional Google Places backend key: `GOOGLE_MAPS_API_KEY` in `backend/.env`, with billing and Places API (New) enabled.
- Optional browser Google Maps key for the Google Places map: `VITE_GOOGLE_MAPS_API_KEY` in `frontend/.env.local`.
- OSM Overpass discovery and OSRM/Nominatim route/place services use configured public endpoints and require no API key; observe their usage limits and policies.
- No OCM key can enable OCM features yet because that provider integration has not been implemented.

## Start commands

From the project root, install dependencies if needed:

```powershell
npm install
npm install --prefix backend
npm install --prefix frontend
```

Then start the existing backend and frontend in separate terminals:

```powershell
npm run dev:backend
npm run dev:frontend
```

## Next steps

1. Verify the new account and favorites endpoints against the configured database without deleting existing station data.
2. Add Open Charge Map integration only after implementing its documented limits, attribution, and license/storage requirements, with its API key kept backend-only.
3. If adding Google Place details or amenities, implement transient retrieval with appropriate attribution and policy-compliant caching/storage; do not treat a nearby business as proof of charger hardware.
4. Test live Overpass/Google provider requests when the services are available and credentials are configured.

## MongoDB persistence completion (2026-10-18)

- Added account-scoped MongoDB persistence for the selected EV, saved trip snapshots (up to 20 per account), and authenticated trip listing/deletion. Trip inputs are constrained to India coordinates and up to 8 intermediate stops. Existing favorites and account credentials remain stored on the same user document; passwords stay excluded from normal queries.
- Connected the configured local MongoDB server at `127.0.0.1:27017`. The URI already selected the existing `plugged` database, so it was preserved rather than switching away from the user's station data. `/api/health` returned healthy and reported 602 saved stations.
- Verified actual authenticated API writes and reads for a selected vehicle and saved trip, persistence across a Mongo reconnect, rejection of unauthenticated reads, and isolation between two temporary users. Both temporary accounts were deleted after the check. `users` and `stations` indexes were inspected; station indexes include the unique source and 2dsphere location indexes.
- Updated `.env.example` with a local Mongo URI and an Atlas alternative. `backend/.env` remains local and untracked; a private JWT secret is configured there, and no secret value was printed or added to source control. Added `npm run verify:db` for repeatable live checks.
- Build passed. All 66 compiled backend tests passed. The regular `npm test` launcher still fails before running tests because Node 24's `tsx` startup calls `uv_os_get_passwd` and gets `ENOMEM`; running the compiled tests with `node --test backend/dist/tests/*.test.js` succeeds.
- The backend already listening on port 5000 returned healthy. Attempting another start connected to MongoDB but could not bind because port 5000 was occupied; the existing process was left untouched.
- A read-only count of the current 602 stations found 0 records with a reported operational status and 409 without a reported operator. The UI labels those fields as not reported and does not mark them operational or invent operator names. Live station operation requires a reliable source or direct operator reports.
- MongoDB was verified through the local driver/API; Compass itself was not opened. In Compass, connect to `127.0.0.1:27017` and select the existing `plugged` database to view `stations` and `users`.

## MongoDB Atlas preparation (2026-10-17)

### Work completed

- Backend startup validates that `MONGODB_URI` exists, is a MongoDB URI with a hostname, and does not still contain placeholders. Failures are explicit and do not start the HTTP API.
- Added a shared Mongoose connection routine with connection-promise reuse, bounded server selection/connect timeouts, safe classified diagnostics for credentials/permissions/database/network failures, and graceful HTTP/Mongoose shutdown on SIGINT/SIGTERM.
- Startup initializes the existing station model indexes only after connecting. It does not drop collections or reset records.
- Health now returns HTTP 503 when MongoDB is disconnected or the station count query fails; it only reports healthy when connected and able to count stations.
- Confirmed the existing station API reads persisted records and that OSM source-ID upserts remain its provider write path. No new unauthenticated generic station-write endpoint was added.
- Added `VITE_API_BASE_URL` with `/api` as the default; the frontend setting is independent from backend-only `MONGODB_URI`.
- Updated the backend environment example with placeholders and setup documentation. The existing `backend/.env` was not read or modified.

### Files changed for Atlas preparation

- `backend/.env.example`
- `backend/src/config/database.ts`
- `backend/src/config/env.ts`
- `backend/src/routes/healthRoutes.ts`
- `backend/src/server.ts`
- `backend/src/services/stationPersistence.ts`
- `backend/src/tests/databaseConnection.test.ts`
- `frontend/.env.example`
- `frontend/src/services/api.ts`
- `frontend/src/vite-env.d.ts`
- `README.md`
- `PROJECT_STATUS.md`

### Verification status

- `npm run build` passed for both backend TypeScript and frontend TypeScript/Vite.
- `npm test` passed: 50 tests, 0 failures, including auth-cookie and credential/network/permission diagnostic tests.
- A separate startup-config check with an intentionally malformed URI failed fast with the expected actionable `MONGODB_URI is invalid` error; no server or database fallback was started.
- No connection test was made against the user's intended Atlas credentials. Do not interpret an existing runtime database connection as proof the manually configured Atlas URI works.
- Checked environment-file presence without reading credential contents: `backend/.env` exists and was not modified; `frontend/.env.local` and root `.env` are absent.

### Remaining manual Atlas steps

1. In MongoDB Atlas, select the existing cluster and ensure it is running.
2. Under **Network Access**, add the development machine's current public IP. Avoid leaving `0.0.0.0/0` enabled.
3. Under **Database Access**, create or select a database user with `readWrite` access only to the PLUGGED database.
4. Copy the driver's connection URI into the existing `backend/.env` as `MONGODB_URI=...`; do not paste it into chat or frontend config. Replace the username, URL-encoded password, cluster host, and database name. Percent-encode reserved password characters.
5. Restart the backend with `npm run dev:backend`. Check its startup logs for successful connection without displaying the URI.
6. Confirm `http://localhost:5000/api/health` returns HTTP 200 with `database.connected: true`; HTTP 503 means startup/database access is unhealthy.
7. Check existing records using `GET http://localhost:5000/api/stations?limit=3`. To test a real safe write, run `npm run seed:osm -- <latitude> <longitude> <radius-meters>` from the project root with an India location and a radius from 1,000 to 25,000 metres, then query the matching map bounds. OSM writes are idempotent by provider ID and do not clear existing station records; provider availability may vary.

## Nationwide station data improvement (2026-10-17)

### Files changed in this increment

- `README.md`
- `PROJECT_STATUS.md`
- `backend/src/models/Station.ts`
- `backend/src/seed/importIndiaOsm.ts`
- `backend/src/seed/indiaExtractMapping.ts`
- `backend/src/services/openStreetMap.ts`
- `backend/src/services/stationPersistence.ts`
- `backend/src/tests/indiaExtract.test.ts`
- `backend/src/tests/stationPersistence.test.ts`
- `frontend/src/components/StationDetails/StationDrawer.tsx`
- `frontend/src/components/StationDetails/StationPhotoGallery.tsx`
- `frontend/src/components/StationDetails/StationPhotoThumbnail.tsx`
- `frontend/src/components/StationDetails/StationPhotoUnavailable.tsx`
- `frontend/src/types/index.ts`
- `frontend/src/utils/stationImages.ts`

### Implemented

- Expanded the full-India Geofabrik PBF importer to write in batches of 250, retry transient MongoDB failures with bounded exponential backoff, verify saved batches, and report candidate/duplicate/existing/inserted/enriched counts plus missing-field counts.
- Enriches matching OpenStreetMap source-ID records only where fields are blank, unknown, or absent. Existing populated names, operators, locations, connectors, access, hours, payments, prices, and other known values are not replaced by OSM imports. No delete/drop operation was added.
- Added conservative within-extract duplicate collapsing only when normalized name, operator, address, connector signature, and sub-3-metre proximity all match. The document keeps every coalesced OSM source ID/link; differing or incomplete identities are kept separate. Existing records are not automatically merged or removed.
- The nationwide PBF pipeline associates mapped amenity nodes only within 300 metres of charger records. It does not load the entire amenities layer into memory.
- Station photos now render only where image attribution, source URL, and a per-image license are present; missing/unlicensed/broken images show a neutral “Photo unavailable” placeholder instead of a generic station illustration.
- Station details explicitly show source verification limitations, operational status as unavailable unless reliable live data exists, a 90-day stale-data warning based on source edit time when available (otherwise import/check time), and an OSM edit timestamp where the extract provides one.
- Inspected official data access: Geofabrik provides the downloadable India OSM PBF. During this check, the official BEE site displayed a technical-upgrade notice, the EV Yatra host was unreachable, and data.gov.in returned HTTP 403 for dataset search. No public documented station-level government download/API or operator API documentation/credentials could be verified; none was guessed, scraped, or imported.

### Verification

- `npm run build` passed for backend and frontend after the final changes; `npm test` passed: 57 tests, 0 failures.
- Read-only live checks: `GET /api/health` reported MongoDB connected and 602 stations; `GET /api/stations?limit=2` and a station-detail request returned HTTP 200 with stored OSM provenance and explicit unknown values.
- The 1,710,832,142-byte Geofabrik India PBF passed its publisher checksum and the full import completed with no error. Snapshot/import results: 557 station nodes + 45 ways; 1 relation unsupported; 0 rejected; 602 unique candidate records; 0 duplicates skipped; all 602 already existed by OSM source ID; 0 new documents inserted; 602 existing records processed/enriched; all 602 verified after writes. No collection reset or destructive database command was run.
- A post-import, read-only paginated API audit read all 602 records. Still missing: name 246, operator 409, address 565, connector details 569, power 587, port count 573, per-kWh/session price amounts 593 (9 records explicitly indicate free), opening hours 547, access 485, payment methods 602, authorized/licensed photos 602, nearby mapped amenities 183, reliable live availability 602, ratings/reviews 602, and source edit timestamp 602.
- The import process emitted a temporary MongoDB connection-lost warning on shutdown. Follow-up `/api/health` and station-list requests both returned HTTP 200; health reported MongoDB connected and 602 stations. The connection is currently healthy, but the transient log is noted rather than treated as a clean no-warning run.
- Nationwide coverage remains unverified and is not claimed. The verified extract contains 602 mapped station features, all matching existing records; this is not proof of complete national coverage. Live availability and station operations remain unverified.
- Browser verification after a full page reload: clicking a Leaflet station marker opened the in-app detail drawer. It displayed the neutral “Photo unavailable” placeholder and explicit unavailable operational/provenance fields. The page reported 100 markers in the current viewport and 602 station records overall. Several base-map tile requests were aborted during map activity; no station API 503 was observed in this check. Earlier transient 503s were not tied to a confirmed station endpoint.

## Authentication and existing-app work (2026-10-17)

The attached large platform specification is being applied incrementally to this existing application; it is not a clean-slate rewrite. Existing station discovery, Leaflet map and OSRM trip planning were preserved.

### Completed in this increment

- Added Mongo-backed account registration/sign-in/sign-out/current-user API. Passwords are bcrypt-hashed; email addresses are normalized; server validates password length/requirements and signup consent; responses avoid account lookup detail on login.
- Added short-lived JWT sessions in HTTP-only, SameSite=Lax cookies, Secure in production, an origin check on state-changing account endpoints, token-version invalidation on logout, and process-local auth throttling.
- Added user profile basics (name, email, member-since through current-user response) and server-protected station favorites with idempotent add/remove and persisted station validation.
- Added a frontend sign-in/create-account dialog, session restoration, sign-out control, and a saved-station heart action in station details. Account emails are explicitly described as unverified because there is no email service.
- Restored the OSM controller endpoints exposed by current `stationRoutes.ts`, keeping bounds discovery rate-limited and persisted with provider-ID upserts. OSM HTTP 429 now stops endpoint retry rather than sending additional requests.
- Fixed the duplicate/incomplete `MapView` declarations and missing map prop interface so the existing map components type-check again.

### Files changed in this increment

- `backend/.env.example`
- `backend/src/config/env.ts`
- `backend/src/controllers/authController.ts`
- `backend/src/controllers/openStreetMapController.ts`
- `backend/src/middleware/authenticate.ts`
- `backend/src/middleware/authRateLimit.ts`
- `backend/src/models/User.ts`
- `backend/src/routes/authRoutes.ts`
- `backend/src/server.ts`
- `backend/src/services/openStreetMap.ts`
- `backend/src/tests/openStreetMap.test.ts`
- `frontend/src/App.tsx`
- `frontend/src/components/AuthDialog.tsx`
- `frontend/src/components/Map/MapView.tsx`
- `frontend/src/components/Navbar.tsx`
- `frontend/src/components/StationDetails/StationDrawer.tsx`
- `frontend/src/services/api.ts`
- `README.md`
- `PROJECT_STATUS.md`

### Verification and blockers

- Backend tests passed: 50 tests, 0 failures.
- Backend build passed. Frontend production build passed after restoring `MapViewProps` and removing duplicate component declarations.
- A MongoDB-backed account/favorites integration flow has not been executed. During final inspection, a backend dev server was already listening on port 5000; a read-only `GET /api/health` returned HTTP 200, `database.connected: true`, and `stationCount: 602`. This session did not start or stop that process or perform database writes; the configured cluster/URI provenance is not verified. Set a private `JWT_SECRET` before testing account persistence.
- No automated user integration tests were added yet. Authentication is not verified against an actual database.
- Still incomplete from the large attached spec: email verification/reset, password change, account deletion/settings, full profile editing, review/check-in/report/submission/photo workflows, route-based compatible charger recommendations, OCM, complete app routing/protected pages and backend/frontend test suites for those features.

## Nationwide OSM audit and final verification (2026-10-17)

### Files changed

- Root: `.gitignore`, `package.json`, `README.md`, `PROJECT_STATUS.md`
- Backend: `backend/package.json`; `backend/src/config/database.ts`; `backend/src/controllers/openStreetMapController.ts`, `stationController.ts`; `backend/src/models/Station.ts`; `backend/src/seed/importIndiaOsm.ts`, `indiaExtractMapping.ts`, `stationDataTools.ts`; `backend/src/services/geocoding.ts`, `openStreetMap.ts`, `stationBackup.ts`, `stationPersistence.ts`, `stationQuality.ts`; `backend/src/tests/indiaExtract.test.ts`, `stationPersistence.test.ts`, `stationQuality.test.ts`, `stationRoutes.test.ts`
- Frontend: `frontend/src/App.tsx`; `frontend/src/components/FilterBar.tsx`, `frontend/src/components/Map/MapView.tsx`; `frontend/src/components/StationDetails/StationDrawer.tsx`, `StationPhotoGallery.tsx`, `StationPhotoThumbnail.tsx`, `StationPhotoUnavailable.tsx`; `frontend/src/services/api.ts`; `frontend/src/types/index.ts`; `frontend/src/utils/externalStation.ts`, `stationImages.ts`
- Generated reports: `backend/reports/india_ev_charging_master.csv`, `station_audit.csv`, `station_audit.json`, `station_quality_report.json`

### Work completed and observed data

- The checksum-verified Geofabrik India PBF contained 602 mapped charging features. All matched existing OSM source IDs; no records were inserted, removed, or merged. The station collection was snapshotted before the importer wrote.
- Compared with that pre-write snapshot, 3 existing records gained mapped amenity details; 599 records changed only the legacy `lastVerifiedAt` import/check timestamp. The report now separates those counts, and future importer runs no longer write `lastVerifiedAt` for ordinary imports. Read-back persistence checks are called persistence checks, not station verification.
- The current collection has 602 unique station IDs and valid in-India coordinates; location quality is 45 Approximate and 557 Unverified, with 0 independently Verified. Operational status is Unknown for all 602. Eight nearby identity pairs are flagged for human review only; none are merged. No source-backed per-kWh/session price amounts are present.
- Remaining unavailable-field counts in the generated report: name 246, operator 409, address 565, connectors 569, power 587, price amounts 602, access type 485, opening hours 547, operational status 602.
- The map viewport API returned 187 unique Delhi-area records over two pages. The India-wide view displays up to 500 of 602 records and clusters markers; this is not proof of nationwide coverage. Browser checks previously verified marker-to-drawer selection, neutral photo fallback, and browser-Back closing the drawer while retaining map/filter state.
- A fresh browser reload and filter interaction did not reproduce the earlier React maximum-update-depth warning. The only reproducible HTTP 503 was `GET /api/auth/me`: authentication returns “Set JWT_SECRET in backend/.env.” MongoDB health remained HTTP 200/connected with 602 stations. No station API failure or 504 was confirmed in the final check.
- React Leaflet viewport/zoom event handlers now use stable callbacks to avoid needless re-subscription during map renders. The latest integrated-browser navigation timed out before a post-change UI check could complete; direct station API/health requests still returned HTTP 200. Treat the new browser-level clustering/drawer regression check as pending, not passed.
- No documented public EV Yatra/BEE station-level feed, authorized operator bulk feed, or OCM credential/license was available. The 22,753 figure remains an unverified target. Do not claim complete India coverage, station operation, or live availability.

### Reports, backup, and verification

- Read-only report outputs: `backend/reports/india_ev_charging_master.csv` (602 rows and unique station IDs), `station_audit.csv`, `station_audit.json`, and `station_quality_report.json`. The CSV now uses explicit “Not available” values and distinguishes import time from OSM source-edit time.
- Pre-write station snapshot: `backend/data/backups/stations-2026-10-17T16-07-53-756Z.jsonl` (602 records). It is local/ignored and is not a whole-database backup.
- `npm run build`: passed for backend TypeScript and frontend TypeScript/Vite.
- `npm test`: passed, 63 tests, 0 failures.
- `npm run report:stations`: passed against the currently connected MongoDB; generated 602 rows, 0 new IDs, 3 detail-enriched records, 599 metadata-only updates, and 0 removed records.
- Current remaining auth setup: add a private random `JWT_SECRET` of at least 32 characters to `backend/.env`, then re-test sign-in/favorites. The actual MongoDB cluster/URI provenance and user/community write flows have not been verified.

## Final production-quality improvement pass (2026-10-18)

### Files changed in this pass

- `backend/src/controllers/tripController.ts`
- `frontend/src/App.tsx`
- `frontend/src/components/AuthDialog.tsx`
- `frontend/src/components/EVCompatibility/VehicleModal.tsx`
- `frontend/src/components/Map/MapView.tsx`
- `frontend/src/components/Navbar.tsx`
- `frontend/src/components/RoutePlanner/RoutePlannerModal.tsx`
- `frontend/src/components/StationDetails/StationDrawer.tsx`
- `frontend/src/types/index.ts`
- `PROJECT_STATUS.md`

### Control and workflow audit

- Main navigation: station search and clear, mobile search toggle, city picker, EV selector, trip planner, sign in/out, and station-count status are wired to app state or API actions. Keyboard Escape now closes the city menu/mobile search.
- Filters: connector, AC/DC, operational status, location quality, state, operator, map style, and reset are wired to the station query/display pipeline. The map clusters nearby markers at lower zoom and keeps individual markers selectable when zoomed or selected.
- Map actions: marker popup/details, map pan/zoom/layers, location, OSM discovery, optional Google Places, and map point selection have handlers and provider error states. Browser location failures now appear as an in-app message instead of a blocking alert. A live browser control-by-control pass remains unavailable in this environment.
- Station details: close/Back, browser Back history, share/navigation, add-to-trip, favorite, review, retry details, image attribution/fallback, connectors, addresses, and amenities are wired. Reviews and favorites require an authenticated account and database. Escape closes the drawer without also closing an underlying view when a modal is open.
- Dialogs: account and EV selection dialogs have visible close controls and Escape behavior; vehicle cards are keyboard-operable buttons. Account submits and station reviews expose pending/error/success states.
- Trip planner: typed origin/destination/stop queries can resolve from the first actual geocoder result on Plan Trip; map/GPS picking, waypoint add/remove/reorder, reverse-trip, route drawing, distance/time, route-adjacent saved charger suggestions, and Open in Google Maps are implemented. If OSRM is unavailable, the in-app line and distance/time are explicitly labelled rough estimates. Saved route chargers can be added as intermediate stops.

### Changes in this pass

- Added a read-only MongoDB lookup for saved charging stations within 25 km of the returned route geometry. Results are capped at 30, connector and status fields are reported without claiming live availability, and lookup failure does not fail route planning.
- Added route charging-station results and Add stop controls, plus waypoint reorder/reverse actions.
- Added the Open in Google Maps action with origin, destination, and entered waypoints.
- Added rough straight-line route fallback with a road-distance factor and 45 km/h estimate when the OSRM provider is unavailable. UI labels and warning text identify it as an estimate and direct users to Google Maps for turn-by-turn directions.
- Added Escape handling for the planner, station drawer, auth dialog, vehicle dialog, city menu, and mobile search; converted EV selection rows to keyboard-operable buttons.
- Replaced the map location permission alert with an in-app message.
- Traced the dark band under the station-count strip to the shared glass-panel header shadow and reduced the navbar shadow to a lighter, narrower elevation.
- Kept station records read-only during route lookup; no database write, migration, delete, or collection reset was performed.

### Verification

- Root `npm run build`: passed (backend TypeScript and frontend TypeScript/Vite production build).
- The normal `npm test` wrapper could not start under the installed Node 24 runtime because `tsx` calls `os.userInfo()` and receives `uv_os_get_passwd returned ENOMEM` before test discovery. Compiling the same tests with the backend build and running `node --test backend/dist/tests/*.test.js` passed all 63 tests (12 suites, 0 failures).
- A direct controller smoke check with the routing fetch forced offline returned a successful `routeType: approximate` result with non-zero rough distance and no charger suggestions when MongoDB was disconnected.
- Live OSRM routing and route-corridor database suggestions were not integration-tested in this pass. Earlier live OSRM access from this environment failed at the external network request. Browser automation tools are not available in this environment, so responsive visuals, click interactions, console output, and external Google Maps launch were not browser-verified.
- No performance baseline or before/after measurement was run; no speed improvement is claimed.

### Remaining limitations

- Road routes and geocoding depend on public OSRM/Nominatim endpoints and their current availability and usage limits. The straight-line fallback is only a rough geographic estimate and is not a drivable route.
- In development, the API can start without `MONGODB_URI` for geocoding and route planning; station/account endpoints return 503 without the database. Production still requires MongoDB configuration.
- Route charger suggestions use saved station records in the database and a 25 km line-distance corridor. They do not prove access, compatibility, or current operation. Unsaved Overpass/Google results are not searched along the route.
- Google Places discovery requires its optional key and browser Maps configuration; no external Google launch test was possible here.
- Sign-in, favorites, and reviews still need an authenticated database-backed integration test. Saved trips, email verification, account settings, check-ins, and station reports are not implemented.
- The verified station dataset and coverage limitations in the earlier audit remain unchanged. No nationwide coverage, live availability, or completeness claim is made.

## Map popup, basemaps, and emergency contacts (2026-10-18)

### Files changed

- `frontend/src/App.tsx`
- `frontend/src/components/EmergencyContactsModal.tsx` (new)
- `frontend/src/components/FilterBar.tsx`
- `frontend/src/components/Map/GoogleMapView.tsx`
- `frontend/src/components/Map/MapView.tsx`
- `frontend/src/components/Navbar.tsx`
- `frontend/src/components/StationDetails/StationPhotoThumbnail.tsx`
- `frontend/src/utils/stationImages.ts`
- `PROJECT_STATUS.md`

### Work completed

- Reworked station map popups to show an image, name, operator, English address/location fallback, connector details, power, Details action, and an explicit close control. Missing names and addresses use useful fallbacks rather than repeating “Not available”.
- Corrected image labels: verified provider photos display their source attribution; generic or user-supplied fallback art is identified as illustrative and explicitly says it is not a photo of that station. Unverified raw photo URLs are no longer presented as sourced provider photos.
- Added Terrain to the existing Road, Satellite, and Hybrid map styles. Switching styles replaces the tile layer while retaining the Leaflet map, station markers, and current filters. Hybrid retains its place/boundary label overlay. A visible in-map notice appears when tiles fail.
- Added a searchable Emergency Contacts dialog to the main navigation. It includes verified national helplines, clickable phone links, official source links, and guidance that the app does not place calls or share location automatically. The modal does not invent EV operator assistance numbers.
- Official references used for contact information: [Government of India helpline directory](https://www.india.gov.in/directory/helpline), [112 Emergency Response Support System](https://112.gov.in/), [National Cyber Crime Reporting Portal](https://cybercrime.gov.in/), [PMUY LPG safety information](https://pmuy.gov.in/Safety_Pledge.aspx), and [RailMadad](https://railmadad.indianrailways.gov.in/madad/feedback.jsp?id=hzhQZACB).

### Verification and limitations

- `npm.cmd run build`: passed (backend TypeScript and frontend TypeScript/Vite production build).
- `node --test backend/dist/tests/*.test.js`: passed, 66 tests across 13 suites, 0 failures.
- Direct read-only requests returned HTTP 200 and image content for Esri Street, Satellite, Topographic, and Reference/World_Boundaries_and_Places tiles. This verifies the configured endpoints from this environment, not that they will be reachable from every user network.
- Browser automation is unavailable in this environment, so popup appearance, responsive layout, emergency-dialog interactions, and live layer switching were not click-tested. The production build and layer endpoint checks passed; perform a browser review in the running app for final visual confirmation.
- This work does not change the audited station data, assert that unknown stations are operational, or claim complete nationwide station coverage.

## Authentication success notice (2026-10-18)

- Removed the registration response's “Email delivery is not configured; this account is not email-verified” message, which had been displayed as a toast after account creation.
- Successful account creation now shows “Thanks for signing up!” and successful login shows “Thanks for signing in!”. Authentication failures continue to display their actual error.
- Updated the registration API response type to no longer expect the removed email-delivery field.
- Verification: `npm.cmd run build` passed; `node --test backend/dist/tests/*.test.js` passed all 66 tests. A source search found no remaining references to the removed notice or `emailVerification` field.

## Connector filter marker matching (2026-10-18)

- The map now loads all station records in the current viewport before applying the connector selection in the UI. This allows legacy/raw provider connector labels to be normalized consistently before the markers are filtered.
- Connector normalization now prefers a recognized reported external connector value, then falls back to the saved connector type. The map's result count reflects markers that match the active filters.
- “All Plugs” clears connector-type filtering and restores all stations in the current viewport. Specific plug choices show matching station markers; records without a reported matching plug are omitted when a specific type is selected.
- Verification: `npm.cmd run build` passed. Browser interaction was not available in this environment.

## Restore all station markers (2026-10-18)

- Replaced the generic filter-clear label with a clear **Show all stations** action. It clears connector, network, location, status, search, and city filters, clears the selected EV compatibility filter and selected station, and returns the map to the India-wide view.
- The action now appears when an EV is selected, even if no other filter is active.
- Verification: `npm.cmd run build` passed. Browser interaction was not available in this environment.
