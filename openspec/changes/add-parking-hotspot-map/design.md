## Context

See `proposal.md` — Why for motivation. This section records only what was measured about the source data, because nearly every decision below follows from it.

The repository is greenfield: OpenSpec scaffolding only, no application code or dependency manifests.

Three public ArcGIS FeatureServer tables at `services2.arcgis.com/11XBiaBYA9Ep0yNJ` were inspected directly. All are non-spatial tables (no ArcGIS geometry), all cap responses at 1000 rows, all refresh weekly.

**`Cityworks_Service_Requests`** — 477,343 rows. Fields: `REQUEST_ID`, `DATE_INITIATED`, `DATE_CLOSED`, `DESCRIPTION`, `INITIATED_BY`, `PRIORITY`, `ADDRESS`, `COMMUNITY`, `DISTRICT`, `REQUEST_CATEGORY`, `RESOLUTION`, `LATITUDE`, `LONGITUDE`, `STATUS`, `DEPT_RESPONSIBILITY`, `WORK_ORDER`. `PARKING` is the largest category at 121,633, of which `Illegally Parked Vehicle` is 110,579 with coordinates present on 108,404 (98%). Parking records span 2020 to 2026-09-05 and have more than doubled annually (10,146 in 2020 → 22,017 in 2025).

**`Cityworks_Service_Requests_Custom_Fields`** — 1,153,448 rows, key-value, related by request id one-to-many. HRM's metadata states this table "must be used in conjunction with" the main table. Every parking record carries seven fields, including `Alleged Violation` (71 distinct values), `Vehicle Was Towed`, and `Property Ownership`.

**`311_Call_Details`** — 4,961,240 rows, 2017 to 2026. Fields: `OBJECT_ID`, `CALL_ID`, `QUEUE_NAME`, `OUTCOME`, `WRAPUP_NAME`, `ARRIVAL_DATETIME`, `TALK_TIME_IN_SECONDS`, `DURATION_IN_SECONDS`. No address, no coordinates, no request identifier. 255,314 rows carry a parking-enforcement wrap code.

Measured distributions that drive thresholds and sizing are quoted inline under the relevant decision.

## Goals / Non-Goals

**Goals:**

- One clustering computation feeding both renderers, so the map and the report cannot disagree.
- A hot spot representation that survives replacing grid binning with density-based clustering, without changing either renderer.
- Every source defect corrected in one place during ingest, so no consumer has to know about them.
- Interpretation limits carried in the data structures, not left to whoever writes the UI copy.

**Non-Goals:**

- Live querying of the portal per request. Ingest is a batch snapshot.
- Any spatial database, tiling service, or spatial index. At ~110k rows the working set fits in memory.
- Statistical hot spot testing (Getis-Ord Gi\*) and density-based clustering. Both are anticipated by the design but out of scope here.
- Per-record linkage between calls and service requests. Ruled out below.

## Decisions

### D1. Coordinates are the canonical location key; address text is display-only

`ADDRESS` cannot be a grouping key. The same physical location appears under multiple spellings depending on postal code presence:

```
5214 GERRISH ST,  HALIFAX,  B3K 5K3   297
5214 GERRISH ST,  HALIFAX             205    same place
827 BEDFORD HWY,  BEDFORD,  B4A 0J1   286
827 BEDFORD HWY,  BEDFORD             182    same place
```

HRM's metadata also confirms the field mixes grammars: "civic address, street name, street intersections, building name and park name". Roughly 245 of the top parking locations are intersection-form (`WOODILL ST & AGRICOLA ST`) with no postal code.

Binning on coordinates makes both problems vanish — variants of one address share a coordinate and land in one area. A prototype run on 2025 data confirmed top areas contain 28–40 distinct address spellings each; string grouping would have shattered every one of them.

*Alternative considered:* normalize address strings into a canonical form. Rejected — it requires a parser per grammar, and still cannot merge an intersection with the civic addresses on the same corner. Coordinates already encode what the parser would try to recover.

### D2. Fixed-area binning, with the area size configurable and defaulted above 175 m

A prototype at 175 m over 2025 (21,744 records) produced 2,726 occupied areas, and showed that 175 m **splits corridors across adjacent areas**:

```
AGRICOLA ST, HILFORD ST     122  +  MCCULLY ST, AGRICOLA ST  111
SOUTH PARK ST, LUCKNOW ST   148  +  SOUTH PARK ST, ANNANDALE 135
QUINPOOL RD, QUINGATE PL    191  +  QUINPOOL RD, PEPPERELL    95
BENTLY DR, WASHMILL LAKE     98  +  BENTLY DR                 93
```

Each pair is one problem reported as two mid-ranked entries, which is precisely the fragmentation this change exists to remove. Corridors are the dominant hot spot shape in a city, so the default area size must be large enough to hold one.

Decision: make area size a parameter, default it coarser than 175 m (in the 300–400 m range), and tune it against the corridor cases above during implementation. A hierarchical indexing scheme whose areas nest is preferred over a naive latitude/longitude grid, for two reasons: nesting supports rolling areas up when zoomed out, matching how traffic layers change detail with zoom; and an equirectangular grid's areas are visibly distorted at Halifax's latitude.

*Alternatives considered:* (a) 175 m as specified — rejected on the evidence above; (b) merging contiguous above-threshold areas into one hot spot — deferred, it is most of the way to density-based clustering and belongs with that upgrade; (c) administrative boundaries such as postal areas or districts — rejected as far too coarse, since `COMMUNITY` alone puts 68% of records in "HALIFAX".

### D3. Hot spot threshold is a normalized rate, calibrated against the measured distribution

The literal reading of "multiple actions" is unusable. Measured over 2025 at 175 m:

```
areas with >=   2 requests:  1829  (67% of occupied areas, 95.9% of requests)
areas with >=  25 requests:   212  (49.7% of requests)
areas with >=  50 requests:    75  (28.4% of requests)
areas with >= 100 requests:    17  (10.8% of requests)
areas with >= 200 requests:     0
```

A threshold of 2 marks two thirds of occupied areas — a uniform wash conveying nothing. Around 25 requests/year (roughly one per fortnight) is where "hot" starts to mean something, and the 212 areas at that level account for half of all activity.

The threshold is therefore expressed as **requests per year**, not raw count, so that changing the analysis date range does not change what qualifies. Severity tiers derive from the same normalized rate, with boundaries published alongside the hot spot set so the legend and report read them rather than hardcoding them.

*Alternative considered:* percentile-based ("top 5% of areas"). Rejected as the default because it guarantees a populated map even when there is genuinely little activity, which misleads; an absolute normalized rate can honestly return few hot spots. Getis-Ord Gi\* would give statistically grounded tiers and is the natural upgrade, but adds a dependency and explanatory burden disproportionate to this change.

### D4. Cluster on space only; attach a time profile per area

Time is a clustering dimension in intent, but keying areas by `(area, time bucket)` collapses under the measured density:

```
spatial only                      8.0 requests per area (2025, 175 m)
x weekday/weekend x 4 dayparts    1.0 per area-bucket
x 24 hours                        0.33 per area-bucket
```

At one request per area-bucket, a "hot spot at 9am" is noise. Using the full 2020–2026 history multiplies counts roughly fivefold and a coarser area size adds more, but the joint key still divides the signal by the number of buckets — and every additional bucket also multiplies the areas a renderer and a report must carry.

Decision: areas are spatial. Each hot spot carries a day-of-week by time-of-day profile computed over its own requests, and reports its peak period. This answers "where, and when there" with the counts concentrated in one place instead of spread across buckets. A profile computed from too few requests is flagged low-confidence rather than presented as a finding.

*Alternative considered:* joint space-and-time areas with coarse buckets over full history. Not rejected on principle — it is the more ambitious reading of the requirement and remains a viable later addition, since the per-area profile already computes the underlying cross-tabulation. It is deferred because it also requires a time-scrubbing control in the map and roughly multiplies the served payload.

### D5. Batch snapshot into a local file-backed store

110,579 parking requests at 1000 rows per response is ~111 paged requests; the custom fields add ~774,000 values for those requests. Both are a few minutes once, and the portal refreshes weekly, so per-view live querying buys nothing and makes the app fail whenever the portal does.

A single-file embedded relational store is sufficient. There is no spatial extension requirement: binning is arithmetic on coordinates followed by a grouped count, and the whole working set is small enough to hold in memory. Ingest must be resumable, because a partial failure partway through a hundred-odd paged requests should not discard the pages already fetched.

Custom fields are pivoted from key-value rows onto their request at ingest, so that every consumer reads `alleged_violation` as an attribute rather than re-implementing the one-to-many join.

### D6. Correct each source's timestamp convention separately — they differ

This is the subtlest trap in the data and it is load-bearing for D4.

**`Cityworks_Service_Requests` publishes true UTC.** Verified empirically: staff-entered parking requests show a hard floor of essentially zero records from 00:00–10:00 UTC and a cliff-edge onset at 11:00 UTC, which is 08:00 Atlantic Daylight Time. The onset also smears between 11:00 and 12:00 UTC across the year (77 records at 11:00 against 432 at 12:00 in a 2025 sample), which is exactly the signature of a real daylight-saving boundary in correctly-stored UTC.

**`311_Call_Details` does not.** HRM's own item description states: "There is currently an issue with the timestamp for this datasets where the time is offset by 3-4 hours (depending on daylight savings)." Verified: parking-enforcement calls in summer 2025 show onset at 15:00 UTC and taper ending 02:00 UTC. Subtracting the Atlantic offset places onset at 08:00 local and close at 19:00 local, matching call-centre hours. The offset was applied in the wrong direction at publication.

Consequences: conversion must be daylight-saving-aware, not a fixed offset, or the 8am shift boundary smears across an hour and blurs the very signal the time profile exists to show. And the two sources must be corrected *differently* — applying one correction to both misaligns any call-versus-request comparison by 3–4 hours. Which correction was applied to which source is recorded with the output, so results can be re-derived if HRM fixes the call table.

### D7. No per-record join between calls and service requests — rejected with evidence

The original intent was to cross-reference the two sources to find calls that produced no enforcement action. This was investigated and is not possible.

Every table HRM publishes in this family was checked for a shared identifier:

```
Cityworks Service Requests       REQUEST_ID, WORK_ORDER ('Y'/'N' flag)  -- no call reference
Cityworks SR Custom Fields       REQUESTID + 88 distinct field names    -- no call reference
Cityworks Work Orders            separate dataset                       -- no call reference
311 Call Details                 CALL_ID (telephony identifier only)    -- no request reference
```

HRM's metadata describes `REQUEST_ID` as a "foreign key to the Cityworks Service Request Outcomes dataset" — **that dataset is not published**. Only Service Requests, SR Custom Fields, Work Orders, and WO Custom Fields exist under the `opendata_HRM` account.

Temporal matching is also not identifiable. During business hours the sources run at roughly 15 parking calls/hour against ~7 parking requests/hour, so any candidate window holds multiple plausible partners on both sides, with no address, plate, or shared attribute to disambiguate and no ground truth to validate a pairing against. `OUTCOME` cannot substitute: it is 99.5% `Handled`, an agent disposition, not a municipal outcome.

Decision: report the comparison at time-bucket granularity only — calls, requests, and conversion rate per bucket — and never assert a link between an individual call and an individual request. The finding survives the restriction and is substantial: 255,314 parking-enforcement calls against roughly 116,660 parking service requests over overlapping years, so on the order of half of parking calls never became a request.

Wrap codes must be normalized first. The same activity appears under codes differing only by the responsible department's name across reorganizations (`5- PW - Parking Enforcement` 152,071; `5- TPW - Parking Enforcement` 54,526; `5- P&D - Parking Enforcement` 48,717), and a name-similar code (`P&R - Parks`, 58,954) is unrelated and must be excluded. Coverage windows also differ — calls from 2017, parking requests from 2020 — so comparisons are restricted to the overlap.

### D8. `INITIATED_BY` is a channel flag, not an observer flag

HRM's metadata defines it as "an indication of how the request was initiated, either by 311 Online or Internal". `INTERNAL` therefore means entered by HRM staff, which covers both enforcement officers and call-centre agents, with no field distinguishing them.

An earlier reading of `INTERNAL` as officer-initiated suggested an appealing "enforcement gap" framing — areas with high citizen reporting and low officer presence. That framing is **not supported** and must not appear in any output. The attempted discriminator failed: calls and `INTERNAL` requests have near-identical weekday shapes, both about 50% weekend volume with an 08:00 local onset.

The geographic contrast is nonetheless real and worth surfacing, correctly labelled as self-serve web versus staff-entered channel:

```
PLEASANT ST, DARTMOUTH      113 requests   113 staff-entered    0 311 Online
MALIK CRT, LOWER SACKVILLE  150 requests    10 staff-entered  140 311 Online
```

The channel filter and any per-area channel breakdown must be described in those terms. This constraint is written into the specs rather than left to interface copy, because it is the kind of claim that is easy to reintroduce accidentally.

### D9. Outcome classification uses `RESOLUTION` plus the tow flag, reported as a lower bound

Measured `RESOLUTION` distribution for illegally-parked-vehicle requests:

```
105,152  95.1%  Requested Service Provided
  2,466   2.2%  (null)
  1,174   1.1%  Investigated, No Work Required
  1,033   0.9%  Requested Service Could Not be Provided
    536   0.5%  Alternate Service Provided
    110   0.1%  Investigated, Work Order Opened or Linked
     65   0.1%  SRR09                              <- raw code, undocumented
     43         referrals to other bodies
```

No-action is therefore ~2,250 records, about 2.0%. Two dead ends were eliminated: `WORK_ORDER` is 110,508 `N` against 71 `Y`, and `DEPT_RESPONSIBILITY` is 96% the single value `PW` — neither discriminates anything for parking.

`Vehicle Was Towed` (1,868 `Y`, 1.7%) is the only concrete positive action signal in the data and overrides to actioned.

The 95.1% concentration on one value is consistent with a default applied at closure rather than a per-case judgement, so the no-action rate is reported as a **lower bound**, accompanied by that share and by the indeterminate share. `SRR09` and nulls are classified indeterminate rather than folded into either side.

Because no-action is only ~2% — roughly 375 records per year across thousands of areas — it is exposed as a **rate attribute on spatial hot spots computed over full history**, never as its own hot spot set. Clustering 375 annual records at 300 m would produce areas resting on one or two requests.

`Property Ownership` gives a testable explanation: 15,969 requests (14.5%) are on `PRIVATE` property and `Private Property` is the second most common alleged violation at 17,029. Municipal authority on private lots is limited, which would predict no-action concentrating there. Ownership distribution is reported per hot spot so the rate can be read in that light.

### D10. Hot spots carry their own boundary geometry

Each hot spot is served with an explicit boundary rather than an area identifier that consumers resolve against a known grid. This costs a little payload and buys the D2 upgrade path: replacing fixed-area binning with density-based clustering changes area shapes from regular to irregular, and with explicit boundaries neither renderer needs to change. It also lets the report and the map be driven from one served structure with no shared grid arithmetic.

### D11. Backend computes, frontend renders

Python owns ingest, normalization, clustering, scoring, profiling, labelling, and report generation, and serves the hot spot set over a JSON API. TypeScript owns the map overlay, legend, filters, and selection detail.

All classification, thresholds, tier boundaries, labels, and caveat text originate in the backend and travel with the data. The frontend must not recompute or hardcode them — that is what keeps the map and report in agreement per the `hotspot-clustering` spec, and what keeps the D8 interpretation constraint from being quietly dropped in interface copy.

### D12. Sequential colour scale with no baseline layer

Traffic layers colour free-flowing roads green because a clear road is still a measured road. An area with no parking requests is ambiguous — nobody parks there, or it is unpatrolled, or there is no data — and colouring it green asserts "safe to park", which the data does not support and which a member of the public might act on.

Only areas at or above threshold are coloured, on a single-direction low-to-high scale rather than a good-to-bad diverging one. The legend states explicitly that uncoloured means no recorded hot spot. Severity is also distinguishable by a channel other than hue.

## Risks / Trade-offs

- **Coarsening areas to hold corridors blurs adjacent distinct problems** → Area size is a parameter, tuned against the known corridor cases in D2; the report's per-area street breakdown reveals when one area is mixing unrelated streets.
- **No-action rate rests on a field that looks defaulted** → Reported as a lower bound with the predominant-value share and indeterminate share always attached; suppressed per-area when counts are too low.
- **Reintroducing the officer-versus-citizen framing** → The prohibition is written into the `hotspot-map` and `hotspot-report` specs as scenarios, not left to reviewer memory.
- **Coordinates are address-derived, not captured at the scene** → Observed latitudes share a near-constant fractional tail, indicating derivation from a civic address point rather than GPS. Positions are parcel-accurate, not vehicle-accurate, which is immaterial at 300 m areas but means repeated requests at one address stack on an identical coordinate. Density-based clustering later must weight duplicate coordinates rather than treat them as independent points.
- **Portal schema or availability changes** → Ingest is a separable stage writing a local snapshot; renderers keep working from the last good snapshot, and ingest failure names the failing source.
- **HRM fixes the 311 timestamp offset, silently double-correcting** → The applied correction is recorded with every output, and the corrected distribution has a checkable signature: onset at 08:00 local and close near 19:00 local.
- **Growing volume** → Parking requests more than doubled from 2020 to 2025 and 2026 is already at 16,141 by September. Rate normalization keeps areas comparable across ranges, but a fixed absolute threshold will admit more hot spots over time; the report states hot spot count against total so drift is visible.
- **Portal paging limit is undocumented behaviour** → 1000 rows per response was observed, not promised. Ingest pages until exhaustion rather than assuming a page size.

## Migration Plan

Greenfield; there is no existing system, data, or deployment to migrate, and no rollback target. Bootstrapping order is: project scaffolding, then ingest producing a verified snapshot, then clustering over that snapshot, then the report renderer, then the API and map.

Snapshot refresh is a re-run of ingest against the weekly-refreshed portal. Because the snapshot is a single local artifact, reverting to a prior one is a file operation; retaining the previous snapshot until a new one is verified is sufficient.

## Open Questions

- Exact default area size within the 300–400 m range, to be settled by checking the D2 corridor cases resolve into single areas without merging unrelated streets.
- Exact severity tier boundaries, once the threshold is applied over full history rather than the single 2025 year that was prototyped.
- Whether to include `Parking Inquiries` in any report as context. They are excluded from hot spots by the `parking-data-ingest` spec, and this does not affect clustering, the API shape, or the task breakdown.
