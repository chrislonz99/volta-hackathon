## Purpose

Groups parking enforcement requests into rough geographic areas, decides which of those areas qualify as hot spots, and describes each one with a severity, a human-readable place label, and a profile of when activity occurs there. This produces the single hot spot set that every map and report consumes.

## ADDED Requirements

### Requirement: Group requests into geographic areas

The system SHALL group enforcement requests into contiguous geographic areas covering roughly a city block or two, keyed on request coordinates.

Grouping MUST NOT be keyed on street name or address text. Activity along a single corridor MUST aggregate into an area rather than being split per street name, and multiple spellings of one address MUST NOT produce separate areas.

The area size SHALL be configurable, because the appropriate size is a trade-off: areas too small split a single problem corridor across neighbours, and areas too large blur distinct problems together.

#### Scenario: Corridor activity aggregates

- **WHEN** requests are spread along one street across several adjacent intersections
- **THEN** they contribute to the area covering that stretch rather than producing one area per intersection name

#### Scenario: Address spelling variants do not split an area

- **WHEN** requests at one physical location carry differing address spellings
- **THEN** all of them fall in the same area

#### Scenario: Area size is configurable

- **WHEN** the configured area size is changed and clustering is re-run
- **THEN** the resulting areas reflect the new size, and every consumer receives the recomputed set without change to how it reads the set

### Requirement: Classify areas as hot spots by normalized rate

An area qualifies as a hot spot based on how concentrated activity is, not on the literal presence of more than one request — over two thirds of occupied areas contain at least two requests, so that test would classify almost everything as hot.

The system SHALL classify an area as a hot spot when its request rate meets or exceeds a configurable threshold. The rate SHALL be normalized to requests per unit time so that areas remain comparable when the analysis date range changes.

The default threshold SHALL be calibrated so that hot spots are a bounded, reviewable minority of occupied areas rather than the majority.

#### Scenario: Rate normalization across date ranges

- **WHEN** the same area is evaluated over a one-year range and over a six-year range with proportionally similar activity
- **THEN** its reported rate is comparable between the two, rather than scaling with the length of the range

#### Scenario: Low-activity area is not a hot spot

- **WHEN** an area contains a small number of requests below the configured threshold
- **THEN** it is not classified as a hot spot, and is excluded from the ranked report

#### Scenario: Threshold is configurable

- **WHEN** the threshold is raised
- **THEN** fewer areas are classified as hot spots, and the map, report, and any other consumer reflect the same narrowed set

### Requirement: Assign a severity tier to each hot spot

A single count conveys no urgency at a glance. The system SHALL assign each hot spot a severity tier derived from its normalized rate, to support graduated visual encoding and report grouping.

Tiers SHALL be ordered and named, and the boundaries between them SHALL be reported alongside the hot spot set so any legend or report can state what each tier means.

#### Scenario: Tier assigned

- **WHEN** a hot spot's rate falls within a tier's bounds
- **THEN** it reports that tier

#### Scenario: Tier boundaries are discoverable

- **WHEN** a consumer receives the hot spot set
- **THEN** it can determine the rate range each tier covers without hardcoding those values

#### Scenario: Areas below threshold carry no tier

- **WHEN** an area is not classified as a hot spot
- **THEN** it is not assigned a severity tier, and consumers MUST NOT present it as a checked-and-clear area, because an absence of requests does not establish that parking there is permitted or unenforced

### Requirement: Compute a time profile for each hot spot

Knowing where activity concentrates without knowing when is not actionable. The system SHALL compute, for each hot spot, a profile of its request activity across day of week and time of day, using Atlantic local time.

The profile SHALL identify the area's peak period. Time SHALL NOT be used as a grouping key for the areas themselves, because the data is too sparse to support joint space-and-time areas at usable area sizes.

#### Scenario: Peak period identified

- **WHEN** a hot spot's requests concentrate in a particular part of the week
- **THEN** its profile reports that period as its peak

#### Scenario: Profile uses local time

- **WHEN** a hot spot's profile is computed across a daylight-saving transition
- **THEN** requests at the same local clock time fall in the same profile bucket regardless of which side of the transition they occurred on

#### Scenario: Areas are not split by time

- **WHEN** a hot spot has activity in several distinct periods
- **THEN** it remains one hot spot carrying one profile, rather than becoming several hot spots at the same location

#### Scenario: Sparse profile disclosed

- **WHEN** a hot spot has too few requests for its profile to be meaningful
- **THEN** the profile indicates low confidence rather than presenting a peak as established

### Requirement: Label each hot spot with a human-readable place

An area's internal identifier is meaningless to a reader. The system SHALL derive for each hot spot a human-readable label sufficient to locate it on the ground without a map.

The label SHALL be built from the most common street name among the area's requests together with the community, and SHALL expose the council district, so hot spots can be grouped by district.

#### Scenario: Label derived from contained requests

- **WHEN** a hot spot's requests are predominantly on one street within one community
- **THEN** its label names that street and community

#### Scenario: District exposed

- **WHEN** a hot spot is produced
- **THEN** its council district is available, so hot spots can be totalled per district

#### Scenario: No usable address text

- **WHEN** none of a hot spot's requests carry usable address text
- **THEN** it is labelled by community alone rather than being left unlabelled

### Requirement: Serve one hot spot set to all consumers

The map and the report MUST describe the same reality. Computing them from separate aggregations permits them to disagree.

The system SHALL compute the hot spot set once per set of parameters and SHALL serve that same set to every consumer.

Each hot spot SHALL carry its own area boundary rather than relying on consumers to reconstruct it from a uniform grid, so that a future change to how areas are formed does not require changing consumers.

#### Scenario: Map and report agree

- **WHEN** a map and a report are produced with identical parameters
- **THEN** they present the same hot spots with the same counts, rates, and tiers

#### Scenario: Consumers do not assume uniform areas

- **WHEN** a hot spot is served
- **THEN** its boundary is described explicitly, such that an irregularly-shaped area would render and report without consumers changing

### Requirement: Filter the analysis by date range and attributes

The system SHALL accept a date range and SHALL compute the hot spot set over only the requests within it.

The system SHALL also support restricting the analysis by initiating channel and by canonical violation type, so activity of one kind can be examined on its own.

#### Scenario: Date range applied

- **WHEN** a date range is supplied
- **THEN** only requests initiated within it contribute to the hot spot set, and reported rates reflect that range

#### Scenario: Filter by violation type

- **WHEN** the analysis is restricted to one canonical violation type
- **THEN** only requests of that type contribute, and the resulting hot spots may differ from the unrestricted set

#### Scenario: Filter by initiating channel

- **WHEN** the analysis is restricted to a single initiating channel
- **THEN** only requests from that channel contribute, and any presentation of the result MUST NOT describe the channel as identifying who observed the infraction
