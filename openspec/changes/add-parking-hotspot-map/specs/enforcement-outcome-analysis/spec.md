## Purpose

Determines which parking requests closed without an enforcement action being taken, surfaces that rate on each hot spot, and compares the volume of 311 parking calls against the volume of parking service requests so unconverted demand is visible.

## ADDED Requirements

### Requirement: Classify each request as actioned or closed without action

The system SHALL classify every location-bearing enforcement request as actioned, closed without action, or indeterminate.

A request SHALL be classified as closed without action when its resolution indicates it was investigated with no work required, that the requested service could not be provided, or that it was referred away to another body rather than acted on.

A request SHALL be classified as actioned when its resolution indicates the requested or an alternate service was provided, a work order was opened or linked, or the vehicle was recorded as towed.

A request whose resolution is absent or is an uninterpretable code SHALL be classified as indeterminate, and MUST NOT be silently counted as either actioned or not.

#### Scenario: No-action resolution classified

- **WHEN** a request's resolution indicates it was investigated with no work required
- **THEN** it is classified as closed without action

#### Scenario: Referral classified as no action

- **WHEN** a request's resolution indicates it was referred to another body
- **THEN** it is classified as closed without action, because no parking enforcement action resulted

#### Scenario: Tow overrides to actioned

- **WHEN** a request records that the vehicle was towed
- **THEN** it is classified as actioned

#### Scenario: Missing resolution is indeterminate

- **WHEN** a request has no resolution value, or a resolution that is a raw code with no documented meaning
- **THEN** it is classified as indeterminate and counted separately

### Requirement: Disclose the limits of the outcome signal

The overwhelming majority of requests carry a single resolution value indicating service was provided, which is consistent with that value being applied as a default at closure rather than recorded per case. Treating the resulting rate as a precise measure would overstate its reliability.

The system SHALL report, alongside any no-action figure, the share of requests sitting on the predominant resolution value, and SHALL state that the no-action rate is a lower bound rather than an exact measure.

#### Scenario: Lower-bound framing applied

- **WHEN** a no-action rate is reported
- **THEN** it is accompanied by the share of requests on the predominant resolution value and a statement that the rate is a lower bound

#### Scenario: Indeterminate share reported

- **WHEN** a no-action rate is reported
- **THEN** the share of requests classified indeterminate is reported with it

### Requirement: Expose the no-action rate per hot spot

The system SHALL compute the no-action rate for each hot spot and SHALL expose it as an attribute of that hot spot.

Requests closed without action SHALL NOT be clustered into their own independent hot spot set at the standard area size: they are a small fraction of requests, and at usable area sizes and date ranges the subset is too sparse to distinguish concentration from noise.

#### Scenario: Rate exposed on hot spots

- **WHEN** the hot spot set is computed
- **THEN** each hot spot carries its no-action rate

#### Scenario: No independent sparse clustering

- **WHEN** no-action requests are analysed
- **THEN** they are reported as a rate on existing hot spots rather than as a separate hot spot set whose areas would rest on very few requests

#### Scenario: Rate suppressed when unreliable

- **WHEN** a hot spot has too few classified requests for its no-action rate to be meaningful
- **THEN** the rate is presented as unavailable rather than as a precise percentage

### Requirement: Expose property ownership as a jurisdiction signal

A substantial share of parking requests concern private property, where the municipality's authority to act is limited. This is a plausible driver of requests closing without action and MUST be distinguishable rather than conflated with municipal-property requests.

The system SHALL expose the property ownership of each request, and SHALL report the distribution of ownership within each hot spot.

#### Scenario: Ownership distribution reported

- **WHEN** a hot spot is produced
- **THEN** the distribution of property ownership among its requests is available

#### Scenario: Private-property share distinguishable

- **WHEN** a hot spot's requests are predominantly on private property
- **THEN** that is evident from its ownership distribution, so its no-action rate can be interpreted in that light

### Requirement: Compare call volume against request volume by time bucket

The system SHALL report, over aligned time buckets, the count of 311 parking-related calls against the count of parking service requests initiated, and the resulting conversion rate.

This comparison SHALL be reported only at time-bucket granularity. The system MUST NOT assert a link between an individual call and an individual service request, and MUST NOT present any per-call outcome, because no identifier relates the two sources and no attribute exists by which a candidate pairing could be verified.

#### Scenario: Bucketed conversion reported

- **WHEN** the comparison is produced over a period
- **THEN** it reports calls, requests, and the conversion rate per time bucket

#### Scenario: No per-record linkage claimed

- **WHEN** the comparison is presented
- **THEN** no individual call is shown as having produced or failed to produce a specific service request

#### Scenario: Comparison is not mapped

- **WHEN** the comparison is presented
- **THEN** it is not rendered as a geographic layer, because the call source carries no location

### Requirement: Align call and request timestamps before comparing

The two sources do not share a timestamp convention: the call source is documented as publishing times shifted several hours from true UTC, varying with daylight saving, while the request source publishes true UTC. Comparing them uncorrected misaligns every bucket by that shift.

The system SHALL correct the call source's shift and convert both sources to Atlantic local time before bucketing, and SHALL record which correction was applied.

#### Scenario: Shift corrected before bucketing

- **WHEN** calls and requests are compared by time of day
- **THEN** both are expressed in Atlantic local time with the call source's documented shift corrected

#### Scenario: Correction recorded

- **WHEN** the comparison is produced
- **THEN** the applied correction is recorded with it, so the result can be re-evaluated if the source is fixed

### Requirement: Normalize call wrap codes and align coverage windows

The call source labels the same parking activity under several wrap codes differing only by the responsible department's name, which changed across municipal reorganizations. Counting them separately understates call volume.

The system SHALL normalize wrap codes to canonical parking categories, and SHALL restrict any comparison to the period both sources cover, reporting the window used.

#### Scenario: Departmental variants merged

- **WHEN** wrap codes differing only by department name describe the same parking activity
- **THEN** they are counted under one canonical category

#### Scenario: Coverage window aligned

- **WHEN** the two sources begin at different dates
- **THEN** the comparison covers only their overlapping period, and states that window

#### Scenario: Non-parking codes excluded

- **WHEN** a wrap code resembles a parking code by name but describes unrelated activity
- **THEN** it is excluded from parking call counts
