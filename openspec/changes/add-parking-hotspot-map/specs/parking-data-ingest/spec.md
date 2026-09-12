## Purpose

Retrieves Halifax Regional Municipality parking enforcement service requests from the municipal open data portal into a local, queryable snapshot, and repairs the known defects in that source data so every downstream consumer works from one clean, consistently-interpreted dataset.

## ADDED Requirements

### Requirement: Retrieve the parking request subset

The system SHALL retrieve parking-related service requests and their associated custom field values from the municipal open data portal into a local snapshot.

The portal caps any single response at 1000 rows, so retrieval MUST page until exhausted and MUST NOT assume a single response contains the full result set.

Each retrieved request SHALL retain at minimum: its unique request identifier, initiation timestamp, closure timestamp, request type, initiating channel, priority, address text, community, district, resolution, status, and coordinates.

#### Scenario: Paged retrieval completes

- **WHEN** ingest runs against the portal
- **THEN** the snapshot contains every parking request the portal reports for the configured date range, not only the first 1000

#### Scenario: Retrieval is resumable

- **WHEN** ingest is interrupted partway through paging
- **THEN** re-running it completes the snapshot without duplicating already-retrieved requests

#### Scenario: Portal unavailable

- **WHEN** the portal cannot be reached or returns an error
- **THEN** ingest SHALL fail with a message naming the failing source, and SHALL leave any previously completed snapshot intact and usable

### Requirement: Classify request types by enforcement relevance

Not every parking request describes an infraction at a location. The system SHALL tag each retrieved request as either location-bearing enforcement or non-enforcement.

Location-bearing enforcement types SHALL include illegally parked vehicle, vehicles obstructing snow operations, vehicle immobilization, and winter parking ban requests. Administrative types — parking inquiries, paystation faults, and non-payment ticket issues — SHALL be tagged non-enforcement.

Only requests tagged as location-bearing enforcement SHALL be eligible for hot spot computation.

#### Scenario: Administrative request excluded from clustering

- **WHEN** a request of type "Parking Inquiries" is ingested
- **THEN** it is retained in the snapshot but tagged non-enforcement, and is excluded from hot spot computation

#### Scenario: Enforcement types beyond the parking category

- **WHEN** a request of type "Winter Parking Ban" is ingested, which the source files under a category other than parking
- **THEN** it is tagged as location-bearing enforcement and is eligible for hot spot computation

### Requirement: Attach custom field values to each request

The source publishes per-request attributes in a separate key-value table related by request identifier, with a one-to-many relationship. The system SHALL attach these values to their request so each request exposes them as named attributes.

For parking enforcement requests these attributes SHALL include the alleged violation, whether the vehicle was towed, and the property ownership of the location.

#### Scenario: Custom fields attached

- **WHEN** a parking enforcement request with associated custom field rows is ingested
- **THEN** the resulting request exposes its alleged violation, tow flag, and property ownership as directly readable attributes

#### Scenario: Request with no custom field rows

- **WHEN** a request has no associated custom field rows
- **THEN** the request is retained with those attributes absent, and is not discarded

### Requirement: Normalize alleged violation codes

The source records the same violation under two spellings, one bearing a `(DISPATCH)` suffix — for example `Within 5M of Hydrant` and `Within 5M of Hydrant (DISPATCH)`. Treating these as distinct values splits counts for a single violation type.

The system SHALL normalize each alleged violation to a canonical violation type with the suffix removed, and SHALL separately expose whether the original value carried the dispatch marker.

#### Scenario: Suffixed and unsuffixed values merge

- **WHEN** requests carrying `Within 5M of Hydrant` and `Within 5M of Hydrant (DISPATCH)` are ingested
- **THEN** both report the same canonical violation type, and their combined count is the sum of the two source spellings

#### Scenario: Dispatch marker preserved

- **WHEN** a request carrying a `(DISPATCH)` suffixed violation is ingested
- **THEN** its dispatch indicator is true, and for an unsuffixed value it is false

### Requirement: Interpret timestamps in Atlantic local time

Source request timestamps are UTC. Halifax observes Atlantic time with daylight saving, so a fixed offset misplaces every timestamp for part of the year and smears any time-of-day analysis across an hour boundary.

The system SHALL convert request timestamps to Atlantic local time using daylight-saving-aware conversion, and SHALL expose the local time for time-of-day and day-of-week analysis.

#### Scenario: Daylight saving boundary handled

- **WHEN** two requests are initiated at the same local clock time, one during daylight saving and one during standard time
- **THEN** both report the same local hour

#### Scenario: Source timestamps with a different convention

- **WHEN** a source is documented as publishing timestamps shifted from true UTC
- **THEN** the system SHALL correct that shift before local conversion, and SHALL record which correction was applied to which source

### Requirement: Treat coordinates as the canonical location

The source address text is unreliable as a grouping key: the same physical location appears under multiple spellings depending on whether a postal code is present, and the field mixes several address grammars including civic addresses and street intersections.

The system SHALL use coordinates as the canonical location of a request. Address text SHALL be used only for display and labelling, never as the key for grouping requests by location.

#### Scenario: Duplicate address spellings group together

- **WHEN** two requests carry the same civic address, one with a postal code and one without
- **THEN** they resolve to the same location for grouping purposes

#### Scenario: Intersection-style address retained for display

- **WHEN** a request's address is an intersection rather than a civic address
- **THEN** it is located by its coordinates and its address text remains available for labelling

### Requirement: Reject implausible coordinates

A small number of source records carry coordinates outside the municipality, including transposed latitude and longitude.

The system SHALL exclude from hot spot computation any request whose coordinates fall outside the municipality's plausible bounds, and SHALL report how many were excluded so the exclusion is visible rather than silent.

#### Scenario: Transposed coordinates excluded

- **WHEN** a request carries a latitude value that is actually a longitude, placing it outside the municipality
- **THEN** it is excluded from hot spot computation and counted in the reported exclusions

#### Scenario: Missing coordinates excluded

- **WHEN** a request has no coordinates
- **THEN** it is excluded from hot spot computation and counted separately from out-of-bounds exclusions

#### Scenario: Exclusion count surfaced

- **WHEN** ingest completes
- **THEN** it reports the total requests retrieved and the number excluded for missing and for out-of-bounds coordinates

### Requirement: Report snapshot provenance

Consumers need to know how current the data is, because the source refreshes on its own schedule.

The system SHALL record, and make available with any derived output, the time the snapshot was retrieved and the date range of requests it covers.

#### Scenario: Provenance available to consumers

- **WHEN** a map or report is produced from a snapshot
- **THEN** the snapshot's retrieval time and covered date range are available for display alongside it
