## 1. Project scaffolding

- [ ] 1.1 Create the Python backend project structure with a dependency manifest and verify a clean install succeeds in a fresh virtual environment
- [ ] 1.2 Create the TypeScript frontend project structure with its dependency manifest and verify the dev server starts and serves a placeholder page
- [ ] 1.3 Add a test runner to the backend and verify an intentionally trivial test passes via the documented test command
- [ ] 1.4 Add linting/formatting config to both projects and verify both pass on the empty scaffold
- [ ] 1.5 Write a top-level README documenting the ingest, build, and run commands, and verify a reader can follow it end to end once later tasks land

## 2. Snapshot store and ingest

- [ ] 2.1 Define the local snapshot schema for requests, pivoted custom-field attributes, and 311 calls, and verify the store is created from scratch by a documented command
- [ ] 2.2 Implement paged retrieval against the portal that pages until exhaustion rather than assuming a page size, and verify with a test asserting more than 1000 rows are retrieved for a known multi-page query
- [ ] 2.3 Implement the parking-subset filter covering the enforcement types named in `parking-data-ingest` (illegally parked vehicle, vehicles obstructing snow operations, vehicle immobilization, winter parking ban) plus the administrative types, and verify counts match the portal's own count-only query for each type
- [ ] 2.4 Tag each ingested request as location-bearing enforcement or non-enforcement, and verify a test asserts a `Parking Inquiries` record is retained but tagged non-enforcement while a `Winter Parking Ban` record is tagged enforcement despite its differing source category
- [ ] 2.5 Implement resumable ingest so an interrupted run continues without duplicating retrieved pages, and verify by interrupting mid-run and re-running to reach the same final row count
- [ ] 2.6 Implement ingest failure handling that names the failing source and leaves a prior snapshot intact, and verify with a test that points ingest at an unreachable endpoint and asserts the previous snapshot still loads
- [ ] 2.7 Retrieve and pivot the custom-fields table onto requests so alleged violation, tow flag, and property ownership are direct attributes, and verify a test asserts a known request exposes all three and that a request with no custom-field rows is retained with them absent
- [ ] 2.8 Record snapshot provenance (retrieval time, covered date range) and verify it is readable from the store by a test

## 3. Normalization and data quality

- [ ] 3.1 Implement daylight-saving-aware conversion of request timestamps to Atlantic local time, and verify a test asserts two requests at the same local clock time on either side of a DST transition report the same local hour
- [ ] 3.2 Implement the documented 3-4 hour correction for the 311 call source applied separately from the request source, and verify a test asserts corrected parking-call activity begins near 08:00 local and ends near 19:00 local
- [ ] 3.3 Record which timestamp correction was applied to which source in the snapshot, and verify it is present in provenance output
- [ ] 3.4 Implement canonical violation normalization stripping the `(DISPATCH)` suffix while preserving a separate dispatch indicator, and verify a test asserts the suffixed and unsuffixed hydrant variants merge to one type with the summed count and correct dispatch flags
- [ ] 3.5 Implement coordinate bounds validation excluding out-of-municipality and missing coordinates, counting each reason separately, and verify a test asserts a transposed-coordinate record is excluded and counted as out-of-bounds
- [ ] 3.6 Surface retrieved, missing-coordinate, and out-of-bounds counts at the end of ingest, and verify the reported total reconciles against the portal's count for the same filter
- [ ] 3.7 Implement 311 wrap-code normalization merging the departmental-rename variants of parking enforcement and parking tickets while excluding the name-similar parks code, and verify a test asserts the merged parking-enforcement total and that the parks code is absent

## 4. Clustering engine

- [ ] 4.1 Implement coordinate-keyed binning into configurable fixed geographic areas, and verify a test asserts two records with differing address spellings at one location fall in the same area
- [ ] 4.2 Emit an explicit boundary per area rather than an identifier requiring grid arithmetic downstream, and verify a test asserts each served hot spot carries a usable boundary
- [ ] 4.3 Implement normalized requests-per-year rate computation, and verify a test asserts the rate for one area is comparable across a one-year and a multi-year range with proportional activity
- [ ] 4.4 Implement hot spot classification against a configurable rate threshold, and verify a test asserts raising the threshold reduces the classified set
- [ ] 4.5 Tune the default area size against the corridor cases in `design.md` D2 (Agricola, South Park, Quinpool, Bently) and verify each resolves into a single area without merging unrelated streets; record the chosen size
- [ ] 4.6 Calibrate default threshold and severity tier boundaries over full 2020-2026 history and verify hot spots are a bounded minority of occupied areas, recording the chosen values
- [ ] 4.7 Publish tier boundaries alongside the hot spot set so consumers read rather than hardcode them, and verify a test asserts a consumer can derive each tier's rate range from the served payload
- [ ] 4.8 Implement day-of-week by time-of-day profiling per hot spot using local time, including peak period identification, and verify a test asserts a synthetic area with concentrated activity reports the expected peak
- [ ] 4.9 Implement low-confidence flagging for profiles computed from too few requests, and verify a test asserts a sparse area's profile is flagged rather than reporting a peak
- [ ] 4.10 Implement human-readable labelling from modal street plus community, exposing council district, and verify a test asserts a known hot spot yields a street-and-community label and that an area with no usable address text falls back to community alone
- [ ] 4.11 Implement date-range, initiating-channel, and canonical-violation filters over the clustering computation, and verify a test asserts a violation-type filter changes the resulting hot spot set
- [ ] 4.12 Ensure a single computation per parameter set is shared by all consumers, and verify a test asserts report and API outputs for identical parameters carry identical counts, rates, and tiers

## 5. Enforcement outcome analysis

- [ ] 5.1 Implement actioned / closed-without-action / indeterminate classification per `enforcement-outcome-analysis`, and verify a test covers a no-work-required record, a referral, a tow overriding to actioned, and a null and `SRR09` resolution both landing as indeterminate
- [ ] 5.2 Compute per-hot-spot no-action rate as an attribute of the hot spot, and verify a test asserts the attribute is present on each hot spot
- [ ] 5.3 Suppress the per-hot-spot no-action rate where classified counts are too low to be meaningful, and verify a test asserts a sparse hot spot reports the rate as unavailable rather than a percentage
- [ ] 5.4 Attach the predominant-resolution share and indeterminate share to any reported no-action figure, and verify a test asserts both accompany the rate
- [ ] 5.5 Compute property-ownership distribution per hot spot, and verify a test asserts a predominantly private-property hot spot is distinguishable by that distribution
- [ ] 5.6 Confirm no-action requests are never clustered as an independent hot spot set, and verify a test asserts the clustering entry points expose no such set

## 6. Call-versus-request comparison

- [ ] 6.1 Implement bucketed comparison of normalized parking call volume against parking request volume with a conversion rate per bucket, and verify a test asserts counts and rate for a known bucket
- [ ] 6.2 Restrict the comparison to the overlapping coverage window of the two sources and report that window, and verify a test asserts the window excludes years present in only one source
- [ ] 6.3 Ensure the comparison output carries no per-call outcome or per-record pairing and is not exposed as a geographic layer, and verify a test asserts the output contains no request identifier per call and no geometry

## 7. Text report

- [ ] 7.1 Generate the ranked hot spot report with label, district, count, rate, and tier per entry, and verify a test asserts descending rate order and presence of all five fields
- [ ] 7.2 Bound the report's entry count while stating the total hot spots identified, and verify a test asserts the stated total exceeds the listed entries when the set is large
- [ ] 7.3 Include each entry's peak period, with low-confidence profiles disclosed as such, and verify a test covers both a confident and a sparse entry
- [ ] 7.4 Include each entry's no-action rate and leading violation types, and verify a test asserts both appear per entry
- [ ] 7.5 Include summary totals (requests analysed, hot spots identified, share of requests within hot spots) and verify the share reconciles against the analysed total
- [ ] 7.6 Include the council-district rollup and verify per-district totals sum to the hot spot total
- [ ] 7.7 Include provenance, exclusion counts, the service-request-not-ticket caveat, and the initiating-channel caveat, and verify a test asserts all four are present in generated output
- [ ] 7.8 Emit an equivalent structured form of the report and verify a test asserts the structured and text outputs carry matching values for every hot spot and total

## 8. API

- [ ] 8.1 Expose the hot spot set, including boundaries, tier boundaries, labels, profiles, and outcome attributes, over a JSON endpoint accepting the date-range, channel, and violation filters, and verify a test asserts a filtered request returns the expected hot spot count
- [ ] 8.2 Expose snapshot provenance through the API and verify a test asserts retrieval time and covered range are returned
- [ ] 8.3 Expose caveat text from the backend so the frontend renders rather than authors it, and verify a test asserts the service-request and initiating-channel caveats are present in the payload
- [ ] 8.4 Expose the call-versus-request comparison over its own endpoint, and verify a test asserts it returns buckets with no geometry
- [ ] 8.5 Return an explicit empty-result response distinguishable from an error when filters match no hot spots, and verify a test asserts the distinction

## 9. Map frontend

- [ ] 9.1 Render a base map of the municipality and verify streets and place names are legible at the default view
- [ ] 9.2 Render hot spots as filled areas coloured by severity tier, drawn from served boundaries, and verify in a browser that areas appear as regions rather than markers or count bubbles
- [ ] 9.3 Leave sub-threshold areas uncoloured on a single-direction scale with no good-to-bad baseline, and verify in a browser that no area is coloured to signify acceptable activity
- [ ] 9.4 Render the legend from served tier boundaries including the statement that uncoloured means no recorded hot spot, and verify changing the backend threshold updates the legend without a frontend change
- [ ] 9.5 Implement hot spot selection detail showing label, district, count, rate, tier, profile with peak, leading violation types, and no-action rate, and verify in a browser that selecting a known hot spot shows all fields
- [ ] 9.6 Mark low-confidence profiles and unavailable no-action rates in selection detail, and verify in a browser that a sparse hot spot presents neither as established
- [ ] 9.7 Implement date-range, channel, and violation filter controls that re-render overlay and legend, and verify in a browser that changing the range updates both
- [ ] 9.8 Label the channel filter as arrival channel with no officer-versus-public framing anywhere in the interface, and verify by reviewing all rendered strings
- [ ] 9.9 Display snapshot provenance in the interface and verify it is visible in a browser
- [ ] 9.10 Render the empty-result state distinctly from a load failure, and verify in a browser using a filter combination yielding no hot spots
- [ ] 9.11 Ensure severity is distinguishable without relying on hue alone, and verify by viewing the map under a colour-blindness simulation

## 10. Integration verification

- [ ] 10.1 Run ingest end to end against the live portal and verify the snapshot's request total reconciles against the portal's count-only query for the same filter
- [ ] 10.2 Generate a map and a report with identical parameters and verify every hot spot, count, rate, and tier matches between them
- [ ] 10.3 Verify the top reported hot spots are recognizable Halifax parking locations and that the design D2 corridor cases each appear as one entry rather than several
- [ ] 10.4 Verify corrected call and request time-of-day distributions both place the 08:00 local onset correctly, confirming the two corrections were applied independently
- [ ] 10.5 Re-run ingest over an existing snapshot and verify the refresh completes and the prior snapshot remains recoverable
- [ ] 10.6 Review all user-facing output for the interpretation constraints in design D8 and verify no text claims to distinguish enforcement officers from call-centre staff
