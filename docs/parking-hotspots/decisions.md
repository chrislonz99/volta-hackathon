# Parking hotspots: decisions

## Decided

**2026-09-12, Claude's proposal.**
Scope the first build to "Blocking Driveway (DISPATCH)", not all parking.

Reason.
It is the problem the handout names.
It is 9,729 calls, which is small enough to pull in one run and large enough to prove the pattern.
The same job runs unchanged on the other violation types by passing `--violation`.

**2026-09-12, Claude's proposal. Reversal of an earlier proposal the same day.**
Drop the per-address time window from the product.

Reason.
An adversarial review found the 17 per cent baseline was never queried and is the wrong null.
Re-tested against a matched null, the observed 58.9 per cent sits against a null mean of 48.8 per cent, and only 17 of 58 addresses beat their own 95th percentile.
Out of sample a tuned window scores 47.5 per cent against 41.7 per cent for one city-wide window, and weekday tuning loses outright.
`DATE_INITIATED` is also the staff intake clock: the `INTERNAL` channel carries 85 per cent of calls and records almost none at night.
The window was measuring office hours.

**2026-09-12, Claude's proposal.**
Point the product at physical fixes, not at enforcement shifts.

Reason.
A tow does not change the recurrence rate: 44.7 per cent against 44.6 per cent.
96 per cent of vehicles at the busiest doorways are unique, so there is no repeat offender to deter.
Enforcement has already been tried at these addresses and has not worked.

**2026-09-12, Claude's proposal.**
Rank by calls in the last 12 months, times one minus the tow rate, and drop any address with no call in 12 months.

Reason.
The earlier all-time ranking put 155 dead addresses on a 406-row list, 38 per cent of it.
One printed row had not called since 2024-01-22.
Day-one impact counts double, so a stale row is the most expensive kind of error.

**2026-09-12, Claude's proposal.**
Convert timestamps to UTC-3 before any time-of-day work.

Reason.
The service returns UTC.
Halifax is UTC-3 in summer, so the offset moves every hour-of-day figure by three hours.

## Open for Chris

**Does the demo show the effect test?**
The product cannot yet prove that a visit reduces calls.
Options: say it plainly as the next step, or stub a before and after chart and label it stubbed.
Recommendation: say it plainly. The handout rewards saying what is stubbed, and a stubbed chart invites the question the product cannot answer.

**Who is the named user?**
The product now points at whoever installs signs, bollards and curb paint, which is Public Works Traffic Management rather than parking enforcement.
The handout named enforcement.
Recommendation: say both. Enforcement gets the evidence that these calls are not theirs to win, and Traffic Management gets the list.
