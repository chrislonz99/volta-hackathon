## Purpose

Produces a ranked text report of parking enforcement hot spots that can be read, pasted, or circulated without a map, covering the same hot spot set the map displays so the two never disagree.

## ADDED Requirements

### Requirement: Produce a ranked hot spot report

The system SHALL produce a text report listing hot spots ranked by normalized rate, highest first.

Each entry SHALL carry its human-readable place label, council district, request count, normalized rate, and severity tier. An entry MUST be identifiable on the ground from its label alone, without reference to an internal area identifier.

#### Scenario: Report ranks hot spots

- **WHEN** a report is generated
- **THEN** hot spots are listed in descending order of normalized rate, each with its label, district, count, rate, and tier

#### Scenario: Entries are locatable from the text

- **WHEN** a reader reviews a report entry
- **THEN** the entry names a street and community, not only an internal identifier

#### Scenario: Report length is bounded

- **WHEN** the hot spot set is large enough that listing all entries would be unreadable
- **THEN** the report presents a bounded number of top entries and states how many hot spots exist in total

### Requirement: Include each hot spot's time profile

The system SHALL include, for each reported hot spot, a summary of when its activity occurs — its peak period by day of week and time of day — so the entry states both where and when.

Where the profile is low confidence, the entry MUST say so rather than asserting a peak.

#### Scenario: Peak period stated

- **WHEN** a hot spot with a clear temporal concentration is reported
- **THEN** its entry names its peak period

#### Scenario: Low-confidence profile disclosed

- **WHEN** a reported hot spot has too little activity for a meaningful profile
- **THEN** its entry states that the profile is low confidence instead of naming a peak

### Requirement: Include outcome and violation context per hot spot

The system SHALL include, for each reported hot spot, its no-action rate and its leading canonical violation types, so a reader can tell what kind of problem the area has and how often requests there close without an enforcement action.

#### Scenario: Outcome context included

- **WHEN** a hot spot is reported
- **THEN** its entry includes its no-action rate and its leading violation types

### Requirement: Report municipality and district totals

The system SHALL include summary totals: the number of requests analysed, the number of hot spots identified, and the share of all requests those hot spots account for.

The system SHALL also include a rollup of hot spot activity by council district, since districts correspond to elected representation.

#### Scenario: Summary totals present

- **WHEN** a report is generated
- **THEN** it states the requests analysed, hot spots identified, and the share of requests falling within hot spots

#### Scenario: District rollup present

- **WHEN** a report is generated
- **THEN** it totals hot spot activity per council district

### Requirement: Derive the report from the served hot spot set

The report MUST be generated from the same hot spot set the map consumes, for the same parameters, rather than from an independent aggregation.

#### Scenario: Report and map agree

- **WHEN** a report and a map are generated with identical parameters
- **THEN** every hot spot, count, rate, and tier matches between them

### Requirement: Disclose provenance, exclusions, and interpretation limits

A report circulated without its caveats invites overreading. The system SHALL include in every report:

- the snapshot's retrieval time and the date range analysed;
- the number of requests excluded for missing or implausible coordinates;
- a statement that records are service requests, not confirmed tickets or confirmed infractions, so a hot spot indicates where enforcement attention concentrates;
- a statement that the initiating channel distinguishes arrival channel only and does not identify whether an officer or a member of the public observed the infraction.

#### Scenario: Caveats included

- **WHEN** a report is generated
- **THEN** it states the snapshot retrieval time, date range, excluded request counts, the service-request caveat, and the initiating-channel caveat

#### Scenario: Caveats survive extraction

- **WHEN** the ranked listing is read on its own
- **THEN** the report's structure keeps the caveats attached to it rather than placing them where they are easily separated from the findings

### Requirement: Emit the report in a machine-readable form as well

The system SHALL be able to emit the same report content in a structured form alongside the human-readable text, so the results can be consumed by other tools without parsing prose.

#### Scenario: Structured output available

- **WHEN** a structured report is requested
- **THEN** the same hot spots, statistics, totals, and caveats are available as structured data carrying equivalent values to the text report
