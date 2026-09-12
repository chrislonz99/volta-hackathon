## Purpose

Presents the computed hot spot set as a graduated-colour overlay on a map of Halifax, in the visual idiom of live traffic layers, so a viewer can see at a glance which areas concentrate parking enforcement activity and drill into any one of them.

## ADDED Requirements

### Requirement: Render hot spots as graduated-colour areas

The system SHALL render each hot spot as a filled area on a base map of the municipality, coloured by its severity tier.

Hot spots MUST be rendered as areas, not as point markers and not as clustered count bubbles. Areas MUST NOT be constrained to the outline of a single street, since activity spans corridors and adjacent blocks.

#### Scenario: Areas rendered by severity

- **WHEN** the map loads a hot spot set spanning several severity tiers
- **THEN** each hot spot appears as a filled area whose colour corresponds to its tier

#### Scenario: Not point markers

- **WHEN** several requests occupy the same hot spot
- **THEN** the map shows one coloured area, not one marker per request nor a bubble bearing a count

#### Scenario: Base map provides orientation

- **WHEN** the map is displayed
- **THEN** streets and place names of the underlying municipality remain legible enough to locate the coloured areas

### Requirement: Do not imply that uncoloured areas are clear

In a traffic layer, green means measured and flowing. Absence of parking enforcement activity is not equivalent: it may mean nobody parks there, or that the area is not patrolled, or that no data exists. Colouring such areas as "clear" would assert something the data does not support.

The system SHALL leave areas below the hot spot threshold uncoloured, and SHALL use a single-direction colour scale from lower to higher severity rather than one running from "good" to "bad".

#### Scenario: Sub-threshold area uncoloured

- **WHEN** an area contains activity below the hot spot threshold
- **THEN** it is not filled with a colour signifying low or acceptable activity

#### Scenario: Legend states the meaning of absence

- **WHEN** the legend is displayed
- **THEN** it states that uncoloured areas indicate no recorded hot spot rather than an absence of parking problems

### Requirement: Display a legend describing the scale

The system SHALL display a legend naming each severity tier and the rate range it represents, taking those ranges from the served hot spot set rather than from values fixed in the interface.

#### Scenario: Legend matches the data

- **WHEN** the threshold or tier boundaries change and the map reloads
- **THEN** the legend reflects the new ranges without requiring an interface change

### Requirement: Show hot spot detail on selection

The system SHALL allow a viewer to select a hot spot and SHALL then present its detail: its human-readable label, council district, request count, normalized rate, severity tier, time profile including its peak period, leading violation types, and its no-action rate.

Where a hot spot's time profile is low confidence, the detail MUST present it as such rather than stating a peak as established.

#### Scenario: Detail presented on selection

- **WHEN** a viewer selects a hot spot area
- **THEN** its label, district, count, rate, tier, time profile, leading violation types, and no-action rate are presented

#### Scenario: Low-confidence profile marked

- **WHEN** a selected hot spot's time profile is low confidence
- **THEN** the detail indicates that rather than presenting its peak period as established

### Requirement: Filter the map by date range and attributes

The system SHALL let a viewer set the analysis date range and SHALL re-render the overlay against the recomputed hot spot set.

The system SHALL also let a viewer restrict the map by initiating channel and by canonical violation type.

Where a filter by initiating channel is offered, the interface MUST describe it as the channel the request arrived through, and MUST NOT label it as identifying whether an officer or a member of the public observed the infraction.

#### Scenario: Date range re-renders the map

- **WHEN** a viewer changes the date range
- **THEN** the overlay updates to the hot spot set for that range, and the legend updates to match

#### Scenario: Channel filter honestly labelled

- **WHEN** the initiating channel filter is displayed
- **THEN** its options are described as arrival channels, without claiming to distinguish enforcement officers from call-centre staff

#### Scenario: Empty result after filtering

- **WHEN** a filter combination yields no hot spots
- **THEN** the map indicates that no hot spots meet the criteria, rather than appearing to have failed to load

### Requirement: Show data provenance

The system SHALL display the snapshot's retrieval time and covered date range, so a viewer can judge how current the view is.

#### Scenario: Provenance visible

- **WHEN** the map is displayed
- **THEN** the snapshot retrieval time and covered date range are visible

### Requirement: Remain legible without relying on colour alone

Severity MUST be distinguishable by a means other than hue alone, so that viewers who cannot distinguish the palette's colours can still read the map.

#### Scenario: Severity readable without hue

- **WHEN** the map is viewed without colour discrimination
- **THEN** severity remains determinable, whether through selection detail, ordering, labelling, or an additional visual channel
