# Doorways enforcement cannot fix

Claude Community Impact Lab Halifax, 2026-09-12.
Problem 2, The Same Blocked Driveway.

Halifax answers a blocked driveway call in 41 minutes, closes 94 per cent of them as done, and the same doorway calls again two weeks later.

This finds the doorways where enforcement has already been tried and has not worked, and sends them to whoever owns signs, bollards and curb paint.

## Run it

```bash
python3 src/hotspots.py
```

No keys and no install.
It reads HRM open data over HTTPS and writes `out/watchlist.csv` and `out/watchlist.md`.

```bash
python3 src/hotspots.py --violation "No Parking Sign" --district 7
```

`.github/workflows/nightly.yml` runs it on a schedule and commits the result, so no person has to.

## What it found

- 9,791 blocked driveway calls, 2020 to 2026-09-04.
- 4.6 per cent ended in a tow.
- 94.1 per cent were closed as "Requested Service Provided".
- 44.6 per cent were followed by another call at the same doorway within a year.
- 363 doorways are still calling right now.

**A tow does not change anything.** Towed calls recur at 44.7 per cent. Not-towed calls recur at 44.6 per cent. The difference is 0.1 points.

**It is a different car every time.** Across the 363 doorways, 2,473 distinct vehicles produced 2,588 calls. That is 96 per cent unique. 28 Queen St has 58 calls and 58 different vehicles, with no vehicle appearing twice.

There is no repeat offender to deter. The street produces the violation, not the driver.

## What we tested and threw away

A per-address four-hour patrol window. It does not survive.

`DATE_INITIATED` is when a staff member keyed the call, not when the driveway was blocked. The `INTERNAL` channel carries 85 per cent of these calls and records 6 out of 8,344 between 21:00 and 07:00. Out of sample, a tuned per-address window beats one city-wide window by only 5.8 points, and weekday tuning loses outright.

Most teams on this problem will build that window. It is the office clock.

## Read the work

Start at [docs/index.md](docs/index.md).

- [Why this problem and not the other four](docs/problem-selection/five-problems-research.md)
- [What the product is, real against stubbed](docs/parking-hotspots/product.md)
- [The datasets and their limits](docs/parking-hotspots/data-sources.md)

## One note for the room

The handout says "Service-request data ends December 2024".
That may describe the handed-out export.
The live ArcGIS layer holds 68,857 calls from 2025 and 50,895 from 2026, through 2026-09-05.
Query the service directly and you get 119,752 more calls.
