## Why

Halifax answers a blocked driveway call quickly, closes the great majority as done, and the same doorway calls again a fortnight later. 9,791 blocked-driveway calls sit in HRM's open data from 2020 to 2026, and two figures drawn from them say enforcement is not the remedy: recurrence after a tow is 44.7 per cent against 44.6 per cent without one, and across the doorways still calling, 2,473 distinct vehicles produced 2,588 calls — 96 per cent unique. One address records 58 calls and 58 different vehicles, none appearing twice.

There is no repeat offender to deter. The street produces the violation, not the driver, so these calls belong to whoever installs signs, bollards and curb paint rather than to an officer's shift.

HRM publishes the call in one layer and its violation, outcome and vehicle in a second key-value layer of 1.1 million rows, with no join between them. The join is what makes the finding available at all.

This change records the product as built and the corrections still outstanding against it. It replaces an earlier draft of this same change that described a different product — an all-parking geographic hot spot map with a time-of-day dimension, a snapshot database, and a TypeScript frontend over a JSON API. That draft was written from a data investigation carried out in parallel with, and unaware of, the work that shipped. Four of its central premises were subsequently tested and disproven; they are recorded as rejected with their evidence in `design.md` rather than discarded, because the reasoning is worth keeping.

## What Changes

Recorded as built:

- **Select and join** blocked-driveway calls by matching the alleged violation as a substring, then joining each call to its outcome and vehicle fields from the custom-fields layer.
- **Reduce each address to a doorway** and rank the doorways still calling by recent volume discounted by the enforcement already applied there, dropping any that have gone quiet.
- **Roll doorways up into census dissemination areas**, normalizing each block's load by its dwelling count, and carry onto every doorway the count of still-calling doorways sharing its block — the figure that decides whether the remedy is one bollard or a block-wide measure.
- **Publish the effectiveness evidence**: recurrence with and against a tow, and how many distinct vehicles account for the calls.
- **Ship a self-contained triage board**: one file, no server, no install, no account, carrying both lists and a zoomable map drawn from HRM's street network, recording a triage decision per doorway and per block and sharing those decisions across a team where the hosting environment allows it.
- **Write readable briefs and complete tables** for both lists, from the same run that builds the board.
- **Run without a person**, on a schedule, committing its output.

Corrections outstanding, where the earlier draft was right and the implementation is not:

- **Daylight-saving-aware time conversion.** The implementation applies a fixed three-hour offset, so every record outside daylight saving is an hour out across a six-year range.
- **Run provenance on every output.** Nothing carries the time it was generated. Because the scheduled job commits its output automatically, a stale board is indistinguishable from a fresh one.
- **An explicit recency anchor.** The twelve-month window is measured from the most recent call in the data rather than from the wall clock, so a stalled pipeline slides the window backwards silently.
- **A computed closure time.** The elapsed-time figure used to open the project's own documentation is not produced by the pipeline, while that documentation states every number in it is.

Non-goals: all-parking scope beyond the supplied violation substring (the same pipeline runs on any other violation by argument), effect measurement of an installed remedy, per-address remedy recommendation, and delivery of the brief into anyone's inbox.

## Capabilities

### New Capabilities

- `parking-data-ingest`: Select calls by alleged-violation substring, join them to their outcome and vehicle fields, reduce addresses to doorway keys, locate each doorway at a representative coordinate, place it in a census neighbourhood, convert timestamps to Atlantic local time, and stamp every run with its provenance.
- `doorway-watchlist`: Group calls into doorways, drop those that have stopped calling, carry the evidence that enforcement has not worked there, rank by calls still arriving discounted by enforcement applied, and carry the count of still-calling neighbours on the same block.
- `block-rollup`: Group listed doorways into census dissemination areas, normalize load by dwelling count, name each block by the streets its calls come from, and rank by spread before load.
- `enforcement-effectiveness`: Measure recurrence at a doorway, compare it with and without a tow, measure how often the same vehicle recurs, treat the tow flag as the only published enforcement outcome, and compute the closure time the project's framing rests on.
- `triage-board`: Ship both lists and a street-network map as one self-contained file, record a triage decision per doorway and per block, share those decisions where the environment allows and disclose when it does not, and show when the board was built.
- `text-briefs`: Write a bounded readable brief and a complete machine-readable table for each list, carrying the same figures as the board along with the limits and provenance.

### Modified Capabilities

None. `openspec/specs/` holds no capabilities yet, so all six are new. The five capabilities named in the previous draft of this change — `hotspot-clustering`, `hotspot-map`, `hotspot-report`, `enforcement-outcome-analysis`, and an all-parking reading of `parking-data-ingest` — were never archived into `openspec/specs/`, so they are replaced here rather than modified.

## Impact

- **Existing code.** `src/hotspots.py` implements the pipeline end to end. `web/template.html` and `web/map-network.json` are the board's build inputs. `out/` holds generated output. `.github/workflows/nightly.yml` runs the pipeline on a schedule and commits the result.
- **Language and runtime.** Python 3.12, standard library only — no dependency manifest, no install step, no keys. The board is plain HTML, CSS and JavaScript with no build tooling. **There is no TypeScript, no JSON API and no server**, contrary to the previous draft of this change.
- **No datastore.** Every run queries the source layers live. There is no snapshot, no database, and no resumability beyond a bounded retry — a mid-run failure repeats the run.
- **External dependencies.** Three HRM ArcGIS layers at `services2.arcgis.com/11XBiaBYA9Ep0yNJ`: `Cityworks_Service_Requests`, `Cityworks_Service_Requests_Custom_Fields`, and `Census_2021_Dissemination_Areas`. All public and unauthenticated. The tabular layers page at 1,000 rows; the census layer pages lower when geometry is requested.
- **Shared triage state** depends on shared storage offered by the board's hosting environment, with the viewer's own browser as the fallback. The board must state which is in effect.
- **Interpretation constraints** carried into the specs rather than left to prose: the initiation timestamp is a staff intake clock and not when the problem occurred; the initiating channel identifies the arrival channel and not whether an officer or a member of the public observed the infraction; the tow flag is the only published enforcement outcome and HRM publishes no ticketing field anywhere.
- **Documentation already in the repository** under `docs/parking-hotspots/` and `docs/problem-selection/` records the same decisions in narrative form and remains the project's own account. These specs are the behaviour contract; they should not contradict it.
