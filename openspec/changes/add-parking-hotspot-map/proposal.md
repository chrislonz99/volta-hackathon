## Why

Halifax Regional Municipality publishes 110,579 illegally-parked-vehicle service requests (2020-2026) through its open data portal, 98% of them carrying latitude/longitude. Nothing in the portal shows *where* parking problems concentrate: the data is a flat table, and ranking it by address string fragments a single problem corridor into several mid-ranked rows while double-counting nothing — Quinpool Rd, Agricola St, and South Park St each appear as multiple unrelated-looking entries. There is also no view of *when* enforcement activity happens at a given place, and no visibility into which requests were closed without any enforcement action taken.

A geographic clustering view answers all three questions from data HRM already publishes, and requires no new data collection.

## What Changes

- **Ingest** HRM Cityworks parking service requests and their custom fields into a local queryable snapshot, correcting known source defects (timezone handling, duplicate address spellings, out-of-bounds coordinates, unnormalized violation codes).
- **Cluster** requests into fixed geographic cells keyed on coordinates rather than address strings, so corridor activity aggregates instead of fragmenting. Classify cells as hot spots against a calibrated threshold rather than the literal "two or more actions", which would mark 67% of occupied cells.
- **Profile time within each cell** (day-of-week x time-of-day) rather than clustering on space x time jointly, which the data is too sparse to support at usable cell sizes.
- **Render a map overlay** in the style of Google Maps traffic — graduated colour over areas, not pins and not individual streets — with a legend and cell drill-down.
- **Generate a text report** over the *same* computed hot spot set as the map, ranked, with each cell labelled by human-readable place (modal street plus `COMMUNITY` and council `DISTRICT`) rather than an opaque cell id.
- **Classify enforcement outcomes**, identifying requests closed without an action taken, and surface the no-action rate as an attribute on hot spot cells.
- **Compare 311 parking call volume against service request volume** as a bucketed conversion rate over time. This is explicitly *not* a per-record join: no join key exists between the 311 call table and Cityworks in any of the four related datasets HRM publishes, and temporal matching is not identifiable at observed volumes. See `design.md`.

Non-goals for this change: density-based clustering (DBSCAN/HDBSCAN), statistical hot spot testing (Getis-Ord Gi*), live auto-refresh from the portal, and authentication. The clustering interface is designed so density-based binning can replace grid binning later without touching the renderers.

## Capabilities

### New Capabilities

- `parking-data-ingest`: Retrieve HRM Cityworks parking service requests and custom fields into a local snapshot; normalize timestamps to Atlantic time with DST handling, merge duplicate address spellings, reject out-of-bounds coordinates, normalize violation codes, and pivot the key-value custom fields onto each request.
- `hotspot-clustering`: Bin requests into geographic cells keyed on coordinates, score and classify cells as hot spots against a calibrated threshold, compute a per-cell time profile, and derive a human-readable label for each cell. Exposes one hot spot set consumed by all renderers.
- `hotspot-map`: Serve and render hot spot cells as a graduated-colour area overlay on a base map, with legend, severity tiers, date-range filtering, and per-cell detail on selection.
- `hotspot-report`: Produce a ranked text report of hot spot cells with per-cell statistics, time profile summary, and municipality-level totals, from the same hot spot set the map consumes.
- `enforcement-outcome-analysis`: Classify each request as actioned or closed-without-action, expose the no-action rate per hot spot cell, and report 311 parking call volume against service request volume as a bucketed conversion rate.

### Modified Capabilities

None. This is the project's first change; `openspec list --specs` reports no existing capabilities.

## Impact

- **New project.** The repository currently contains only OpenSpec scaffolding — no application code, no dependency manifests, no build tooling. This change establishes the initial project structure.
- **Backend**: Python. Responsibilities are ingest, normalization, clustering/scoring, report generation, and a JSON API for the frontend.
- **Frontend**: TypeScript, rendering the map overlay and cell detail.
- **Local datastore**: a file-backed snapshot of the parking subset (roughly 110k request rows plus ~774k pivoted custom-field values). At this volume no spatial database or tile server is required; the working set fits in memory.
- **External dependency**: HRM's ArcGIS FeatureServer endpoints (`services2.arcgis.com/11XBiaBYA9Ep0yNJ`) for `Cityworks_Service_Requests`, `Cityworks_Service_Requests_Custom_Fields`, and `311_Call_Details`. These are public and unauthenticated but cap responses at 1000 rows, so ingest is paged. Data is a snapshot; the portal refreshes weekly.
- **Data quality dependencies** on documented and observed source defects, recorded in `design.md`. Two are load-bearing: the 311 call table's timestamps are shifted 3-4 hours (HRM-acknowledged), while the Cityworks timestamps are correct UTC — so any comparison across the two must correct one side.
- **Interpretation constraint**: `INITIATED_BY` distinguishes the 311 Online self-serve channel from staff-entered records; it does *not* distinguish enforcement officers from call-centre agents. Reports must not claim otherwise.
