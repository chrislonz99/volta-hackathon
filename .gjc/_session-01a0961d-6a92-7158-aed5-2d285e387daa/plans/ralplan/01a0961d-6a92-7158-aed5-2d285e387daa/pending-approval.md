# Consensus revision 3: HRM Parking Predict, Clustering, and Alerting

## Summary

Build an incremental persistent staff product for a parking enforcement operations coordinator with **Python FastAPI, SQLite, and React/Vite TypeScript**. The first end-to-end product ingests citywide blocked-driveway Cityworks complaints and their custom fields, proposes two reviewable grouping levels (possible same-incident reports and recurring-location clusters), ranks the **top 20 clusters by risk of the same type of recorded complaint within seven days**, and presents change-triggered alerts in an in-app inbox. The coordinator can merge or split either cluster level and acknowledge, assign, or defer alerts. Every mutation is version-checked, command-idempotent, persisted, and audited; source records remain immutable.

This revision preserves all decisions from `stage-02-intent.md` and consolidates the compatible Architect `WATCH/COMMENT` and Critic `ITERATE` findings. It closes four review gaps:

1. Cluster corrections carry both `available_at` and `effective_from`; every forecast/backtest pins the cluster snapshot and correction watermark available at its cutoff. Prospective chronological metrics never use future corrections, while retrospective gold-set metrics are reported separately.
2. Every cluster or alert mutation accepts a client-generated `commandId`, persists it under a uniqueness constraint, and returns the stored result on exact replay. Reuse with a different operation or payload is rejected.
3. The v1 data-source contract is explicit: Cityworks Service Requests and Custom Fields are the only cluster/risk inputs; 311 data is aggregate/context-only without a verified join; district boundaries are display/aggregation-only after edition validation.
4. `docs/parking-hotspots/data-sources.md` and the tracked generated `out/watchlist.csv`/`out/watchlist.md` are included in the migration and claim-ledger scope. Obsolete generated watchlists will not remain shipped as authoritative outputs.

The plan is baseline-first without being analytics-only: each milestone completes a thin database/API/UI path. A transparent recent-frequency baseline and chronological evaluation precede any candidate model. Public facts, derived analysis, and synthetic scenarios remain separate in persistence, API contracts, UI labels, generated artifacts, and metrics.

## Intent Diff

| Area | Previous/current state | Consensus direction |
|---|---|---|
| Product boundary | Current script emits CSV/Markdown; earlier planning initially considered an analytics-only first product. | Persistent SQLite-backed FastAPI API and React/Vite staff UI from the first slice. Batch commands remain operational tools, not the product boundary. |
| User and action | Repository copy routes a “cannot fix” list toward physical fixes; named user had been ambiguous. | Parking enforcement operations coordinator reviews recurrence/rising risk and selects candidates for site checks or enforcement. No automatic action or guaranteed remedy. |
| Cohort | Repository defaults to a broad `Driveway` substring and documents conflicting counts. | Versioned citywide blocked-driveway predicate, with exact included labels and an as-of timestamp attached to every published count. |
| Clustering | `clean_address()` string grouping treats complaints as incidents and offers no corrections. | Separate incident candidates from recurring-location clusters; expose evidence/confidence; persist audited coordinator merge/split corrections. |
| Correction time | Revision 2 versioned corrections but did not bind future human knowledge out of historical evaluation. | Persist `available_at` and `effective_from`; forecasts pin the correction state known at cutoff. Later-reviewed gold sets are evaluated separately from prospective metrics. |
| Prediction | Current rank is `calls_12mo * (1 - tow_rate)`, not a labeled forecast. | Baseline-first seven-day same-type **recorded-complaint** risk ranking; exactly top 20 when 20 are eligible; no precise probabilities without calibration. |
| Alerting | Generated files have no inbox state. Revision 2 specified workflow state but not retry-safe client mutations. | In-app inbox with acknowledge, assign, defer, expiry, deterministic re-alerts, and required `commandId` for every cluster/alert mutation. |
| Evidence | Current docs/artifacts mix source facts, derivations, and causal interpretation. | Mechanically and visibly distinguish public, derived, proposed, and synthetic content; claim-ledger checks cover docs, UI, API, and generated outputs. |
| Sources | Repository data-source prose overstates join novelty and some field meaning. | Cityworks requests/custom fields only for v1 clusters/risk; 311 context-only absent a verified join; district polygons display/aggregation-only after edition validation. |
| Time | Fixed UTC-3 and rejected intake-time patrol windows. | Preserve UTC; use `America/Halifax` for display; never treat `DATE_INITIATED` as physical occurrence or patrol time. |
| Legacy outputs | `out/watchlist.csv` and `.md` are tracked and contain obsolete ranking/causal framing. | Stop treating them as authoritative deliverables; replace them with provenance-safe generated exports after the new pipeline exists, then choose and document tracked-snapshot versus ignored-build-output policy. |

## Decision Drivers

1. **Actionable coordinator workflow.** The coordinator must move from top-20 risk or an alert to source evidence, grouping correction, and a persisted disposition without leaving the app.
2. **Point-in-time auditability.** Immutable source history, bitemporal correction availability/effect, pinned analysis snapshots, command receipts, and append-only workflow events must reproduce what the system and staff knew at any cutoff.
3. **Measured usefulness without overclaiming.** Reviewed grouping quality, prospective chronological performance versus baseline, and staff-rated alert usefulness are separate scorecard dimensions; reported complaints are not all violations and observational associations are not causal effects.

## Principles

1. **Persist one usable vertical slice at a time.** Each milestone crosses migration, domain service, API, and UI before expanding scope.
2. **Separate evidence classes by construction.** Public facts, derived analysis, proposals, and synthetic scenarios cannot silently cross storage, API, UI, export, or metric boundaries.
3. **Make knowledge time explicit.** Source versions and human corrections used by a forecast must have been available at its cutoff; retrospective adjudication is a separate gold-set view.
4. **Make mutations safely repeatable.** Every client cluster/alert command has a unique `commandId`; retry returns its stored result and never duplicates an audit event.
5. **Baseline before complexity.** A transparent recent-frequency baseline is implemented and chronologically evaluated before any candidate model is promoted.

## Options

### Option A — Incremental persistent vertical slices (selected)

Deliver a migrated SQLite/FastAPI/React shell, then immutable ingestion, correctable clustering, baseline/top-20 risk, and inbox state in successive working slices.

**Pros**
- Validates persistence, API contracts, UI comprehension, analytics, and staff workflow together.
- Surfaces cluster-correction and alert-state mistakes before model complexity grows.
- Produces durable lineage, command receipts, and auditable state early.
- Directly satisfies the resolved product contract.

**Cons / bounds**
- Requires careful schema/API work earlier than a batch experiment.
- Initial UI is intentionally plain and subject to accessibility/usability iteration.
- SQLite is suitable for the bounded pilot, not an implicit high-concurrency production commitment.

### Option B — Persistent backend/analytics first, React later (fallback only)

Build the same schema, FastAPI contracts, baseline, and alert state before most screens.

**Pros**
- Backend lineage and replay tests become available sooner.
- Can accommodate temporary frontend capacity constraints.

**Cons / bounds**
- Delays validation of whether evidence, correction, and alert states make sense to the coordinator.
- Risks API design that fits calculations but not the actual workday.
- Must still deliver the resolved React staff app; it is not an analytics-only alternative.

### Invalidated option — Analytics-only batch product

CSV/Markdown cannot satisfy persistent audited corrections, command replay, alert transitions, or the in-app inbox. Batch exports remain optional products of the persistent system, not its authoritative state.

### Invalidated capability — Intake-time patrol prediction

`DATE_INITIATED` is an intake timestamp affected by entry channel and office process. Do not infer physical obstruction windows, officer arrival, or useful patrol time from it. Reconsider only with real observation/dispatch/arrival data and a separately approved target and evaluation.

## In scope / out of scope

### In scope

- Citywide blocked-driveway Cityworks complaints under a versioned cohort predicate.
- Cityworks Service Requests plus one-to-many Service Request Custom Fields as v1 cluster/risk inputs.
- Immutable source/change history and reproducible ingestion, analysis, cluster, forecast, and alert runs.
- Incident-candidate and recurring-location clustering with coordinator merge/split authority.
- Correction `available_at`/`effective_from`, cluster snapshots, prospective cutoff semantics, and a separate retrospective gold set.
- Seven-day same-type recorded-complaint target, recent-frequency baseline, chronological evaluation, and top-20 review page.
- In-app alerts with acknowledge, assign, defer, expiry, deterministic re-alert, optimistic versions, and command replay.
- Staff-facing real public addresses; synthetic/generalized locations only for public/demo presentation.
- Explicit provenance/freshness/uncertainty and a balanced pilot scorecard.
- Documentation and tracked generated-output disposition, including `data-sources.md` and `out/watchlist.*`.

### Out of scope

- Route optimization, automatic enforcement/site-check dispatch, or automatic infrastructure decisions.
- Patrol/observation/arrival/duration/labor-time prediction from Cityworks intake timestamps.
- Claims that towing is ineffective, enforcement failed, closure means durable resolution, vehicle descriptions uniquely identify vehicles/drivers, or a physical intervention will fix a site.
- Real-time vehicle detection; alerts occur after source refresh.
- Email or Teams delivery in v1.
- Public exposure of real addresses.
- Per-complaint or per-cluster use of 311 volumes/details absent a verified join; a separate aggregate workload forecast is not part of this v1.
- Using unvalidated district editions as predictive features, operational territories, or cluster boundaries.
- Causal intervention-effect claims without a designed comparison.
- Synthetic records as evidence of real accuracy, time savings, complaint reduction, or intervention effect.
- A production-scale database/platform or multi-tenant design beyond the bounded pilot.

## Findings about the idea and repository

### Confirmed product contract

- HRM is the customer; the parking enforcement operations coordinator is the primary user.
- Predict, Clustering, and Alerting are the core.
- Initial scope is citywide blocked-driveway complaints.
- Forecast contract is top-20 risk of a same-type recorded complaint within seven days.
- Coordinator corrections and inbox actions persist and are audited.
- Stack is Python FastAPI, SQLite, and React/Vite TypeScript.
- V1 delivery is an in-app inbox; email/Teams are deferred.
- Pilot success balances reviewed clustering quality, chronological forecast performance against baseline, and staff-rated alert usefulness.

### Directly observed repository facts

- `src/hotspots.py` is a standard-library CLI whose `query()` offset-pages ArcGIS, `load()` joins service requests/custom fields, `build()` groups normalized address strings and calculates aggregates, and `write_csv()`/`write_brief()` generate watchlists.
- The current rank `calls_12mo * (1 - tow_rate)` is an unevaluated heuristic, not seven-day prediction.
- `clean_address()` cannot distinguish alternate spellings, separate entrances, or multiple reports of one occurrence. `repeat_calls` counts calls rather than reviewed incidents.
- `vehicle_key()` uses make/model/colour, which is non-unique and incomplete.
- `to_local()` uses fixed UTC-3 across historical records.
- Offset pagination has no explicit snapshot/watermark/reconciliation contract; repeated custom fields overwrite in a dictionary.
- `docs/parking-hotspots/data-sources.md` is tracked and marked “Final.” It states “Nobody joins them. The join is the product,” describes vehicle values as showing whether one driver repeats, uses fixed summer UTC-3 wording, and contains live-query counts without a durable run receipt. Those claims need correction or qualification.
- The data-source document correctly records that 311 Call Details lack a service-request join key; it contains no validated basis for using 311 per address.
- `out/watchlist.csv` and `out/watchlist.md` are tracked generated files. The CSV exposes real addresses and derived tow/vehicle/repeat fields without provenance/run/cutoff columns. The Markdown states that nearly every call is a different vehicle, there is no repeat offender, and the list should go to physical-fix owners rather than an officer; those are not supportable product claims.
- The tracked Markdown reports 9,791 calls through 2026-09-04, while `data-sources.md` reports 9,729 for “Blocking Driveway (DISPATCH)” and other inspected docs use differing cohort text. The mismatch likely reflects predicate/as-of differences and must be reproduced, not harmonized by editing a number.

### Claim corrections

- Near-equal recurrence percentages among towed/non-towed records are observational and confounded, not proof of zero tow effect.
- No recorded tow is not proof of no warning, ticket, visit, or action.
- Make/model/colour cannot prove distinct vehicles or drivers.
- Closure is not durable resolution; recurrence is not staff failure.
- Signs, bollards, or curb markings are site-assessment candidates, not guaranteed fixes.
- Joining datasets that official documentation says should be joined is not defensibly “nobody has done this.”
- Every displayed/generated count requires a cohort version, source snapshot/run, and analysis cutoff.

## V1 data-source contract

| Source | V1 use | Prohibited or gated use |
|---|---|---|
| Cityworks Service Requests | Public complaint identity, address/coordinates, category/type context, intake/closure fields, channel, district attribute, and source timestamps. Core cluster/risk input. | Intake is not observed violation time, dispatch, or arrival. Closure is not durable resolution. |
| Cityworks Service Requests Custom Fields | Alleged-violation cohort selection and descriptive custom-field context, preserved one-to-many and joined by request ID. Core cluster/risk source where feature semantics are approved. | Tow is not total enforcement action; vehicle description is not identity; missing/negative values do not prove no action. |
| 311 Call Volumes | Optional aggregate context displayed independently with its own cadence/provenance. | No per-address/cluster join or risk feature. A separate aggregate workload forecast requires its own scope/evaluation. |
| 311 Call Details | Optional aggregate/context research only. | No join to service requests or clusters without a verified service-request key and documented coverage. |
| District boundaries | Optional display/aggregation overlay after edition, effective dates, CRS, and identifiers are validated and recorded. | No v1 predictive feature, enforcement-territory assumption, or clustering boundary. Existing service-request `DISTRICT` may be shown as a source attribute with its own limitations. |
| Other parking/accessible/permit layers | Deferred hypotheses/context. | No predictive contribution or causal interpretation without an explicit evaluation and provenance contract. |

## File-level changes

All entries are implementation proposals. This planning pass changes no product file.

### Runtime, persistence, and source ingestion

| File / symbols | Planned responsibility |
|---|---|
| `pyproject.toml` | Python version; FastAPI, Uvicorn, Pydantic, and selected SQLAlchemy/Alembic or minimal equivalent; test/dev commands. Avoid unrelated frameworks. |
| `src/hrm_parking/config.py` — `Settings`, `get_settings` | SQLite path, Cityworks endpoints, staff/public mode, `America/Halifax`, ingestion limits, fail-closed real-address exposure. |
| `src/hrm_parking/domain.py` — `EvidenceKind`, `ClusterLevel`, `AlertType`, `AlertStatus`, `CommandKind` | Shared domain vocabulary. |
| `src/hrm_parking/db.py` — `create_engine`, `session_scope` | SQLite foreign keys, transactions, migration integration, bounded concurrency policy. |
| `migrations/` | Version tables and constraints by milestone; never opportunistically create schema inside a request. |
| `src/hrm_parking/models.py` | Persistence mappings and invariants described below. |
| `src/hrm_parking/schemas.py` — `CommandRequest`, `CommandResult`, cluster/alert command schemas | Pydantic API contracts. Every mutating cluster/alert request requires `commandId` and `expectedVersion`; separate staff and public/demo response shapes. |
| `src/hrm_parking/ingestion/arcgis.py` — `ArcGISClient`, `iter_query_pages` | Extract `query()` with bounded retries, stable ordering, page accounting, and reconciliation evidence. |
| `src/hrm_parking/ingestion/cohort.py` — `BlockedDrivewayCohort`, `matches_blocked_driveway` | One versioned exact cohort predicate used by ingestion, metrics, docs, and exports. |
| `src/hrm_parking/ingestion/service.py` — `run_ingestion`, `reconcile_ingestion`, `pivot_custom_fields` | Persist reconciled runs/source-shaped rows; explicit changed/repeated/missing custom-field policy; failed runs never advance freshness. |
| `src/hrm_parking/time.py` — `source_datetime`, `to_halifax_display` | Preserve UTC and use IANA conversion for display only. |
| `src/hotspots.py` — `main` | Replace current monolith with a thin CLI over package services for ingest/analyze/evaluate/export. Remove parallel causal scoring/copy rather than preserve compatibility. |

### Persistent models and temporal contracts

| Table/model | Required fields/invariant |
|---|---|
| `ingestion_runs` / `IngestionRun` | Cohort/filter version, retrieval interval, newest source timestamp, counts, reconciliation, status/error. Only reconciled success is current. |
| `source_complaint_versions` / `SourceComplaintVersion` | Source key, valid/effective source interval where derivable, `available_at`/first-seen run, raw fields/hash. Never overwrite history. |
| `source_custom_field_versions` / `SourceCustomFieldVersion` | Source object identity, request ID, name/value, availability/history. Preserve one-to-many rows. |
| `analysis_runs` / `AnalysisRun` | Ingestion snapshot, cutoff, source watermark, algorithm/config version, status, evidence kind. |
| `incident_candidates`, `incident_memberships` | Versioned proposals, analysis run, evidence/confidence, membership reasons, supersession. |
| `location_clusters`, `location_memberships` | Versioned recurring-location proposals and membership evidence. |
| `cluster_corrections` / `ClusterCorrection` | `command_id` unique reference, actor, recorded timestamp, `available_at`, `effective_from`, cluster level, merge/split, expected/before/after versions and memberships, reason/result. Append-only. `available_at` is when the correction can enter system decisions; `effective_from` is the domain time the reviewer says the grouping should represent. |
| `cluster_snapshots` / `ClusterSnapshot` | Immutable materialization of proposal plus only corrections whose `available_at <= cutoff` and whose effective interval applies. Stores correction watermark and version set. Later corrections cannot rewrite it. |
| `forecast_runs` / `ForecastRun` | Cutoff, seven-day window, source watermark, exact `cluster_snapshot_id`, correction watermark, baseline/model version, population and metrics class (`prospective` or `retrospective_gold`). |
| `risk_scores` / `RiskScore` | Forecast run, pinned cluster/version, rank/tie key, tier/score, reason codes, top-20 flag. |
| `workflow_commands` / `WorkflowCommand` | Globally unique client `command_id`, actor/context, command kind, canonical request hash, received/completed timestamps, status, stored HTTP/domain result. Same ID plus identical request returns stored result; same ID with different actor/kind/payload returns conflict. |
| `alerts` / `Alert` | Stable alert ID, pinned cluster/version, logical fingerprint, trigger epoch, first/last trigger, expiry/current version/status. |
| `alert_events` / `AlertEvent` | `command_id` for client actions (or deterministic internal event ID for system expiry/re-alert), append-only transition, actor/time, prior/new versions/status, assignment/defer/reason, stored result link. |
| `synthetic_scenarios` and child records | Fixed seed/version, fictional location and explicit synthetic evidence class; schema/query constraints exclude them from real metrics. |

### Clustering, prediction, and alerts

| File / symbols | Planned responsibility |
|---|---|
| `src/hrm_parking/analytics/clustering.py` — `build_incident_candidates`, `build_location_clusters`, `explain_membership` | Conservative proposals using approved address/coordinate/type/time evidence; separate complaints, incidents, locations. |
| `src/hrm_parking/analytics/corrections.py` — `submit_merge`, `submit_split`, `materialize_cluster_snapshot`, `replay_corrections` | Validate version and `commandId`; persist correction availability/effect; replay only knowledge available at a requested cutoff; surface conflicts after re-analysis. |
| `src/hrm_parking/analytics/features.py` — `build_features_at_cutoff`, `label_seven_day_complaint`, `assert_no_future_inputs` | Leakage-safe features/labels against the pinned source and cluster snapshot. |
| `src/hrm_parking/analytics/baseline.py` — `RecentFrequencyBaseline`, `rank_top_20` | Transparent deterministic baseline and tie-breaking; no causal tow weighting. |
| `src/hrm_parking/analytics/evaluation.py` — `rolling_time_splits`, `evaluate_prospective`, `evaluate_retrospective_gold`, `build_scorecard` | Keep real-time-knowledge metrics separate from later-adjudicated grouping metrics; derive promotion gates from baseline/pilot evidence. |
| `src/hrm_parking/analytics/model.py` — `CandidateRiskModel` | Added only after baseline milestone; baseline always retained. |
| `src/hrm_parking/alerts/service.py` — `derive_alerts`, `alert_fingerprint`, `execute_alert_command`, `expire_alerts`, `should_realert` | Narrow v1 alerts, deterministic transition table, command receipt/replay, append-only events, new-evidence override. |
| `src/hrm_parking/commands.py` — `execute_once`, `canonical_request_hash`, `replay_result` | Shared transactional command idempotency boundary for cluster and alert mutations. |

### FastAPI contracts

| File / route | Planned responsibility |
|---|---|
| `src/hrm_parking/api/app.py` — `create_app` | App factory, lifespan/migrations policy, error mapping, routers; no import-time network work. |
| `api/routes/status.py` — `GET /api/status` | Last successful sync, newest complaint, analysis cutoff, versions, failed/stale warning. |
| `api/routes/clusters.py` — list/detail, `POST /api/clusters/merge`, `POST /api/clusters/split` | Staff evidence and correction commands. Mutation body includes `commandId`, `expectedVersion`, reason, and members. Exact replay returns stored original response/status; mismatched reuse is 409. |
| `api/routes/risk.py` — `GET /api/risk/top`, run detail | Enforce seven-day/top-20 v1 contract and disclose pinned source/cluster/correction versions plus metric class. |
| `api/routes/alerts.py` — inbox/detail, `POST .../acknowledge`, `/assign`, `/defer` | Each mutation requires `commandId` and `expectedVersion`; response includes command receipt and alert version. |
| `api/routes/evaluation.py` — latest/run detail | Separate prospective chronological, retrospective gold-set, and synthetic/demo result classes. |
| `api/routes/context.py` — optional aggregate context | If included, expose 311 or validated boundary context separately; never merge it invisibly into cluster/risk response evidence. |

### React/Vite TypeScript staff UI

| File / symbols | Planned responsibility |
|---|---|
| `web/package.json`, `vite.config.ts`, `tsconfig.json` | React/Vite/TypeScript setup and focused tooling. |
| `web/src/api/types.ts` — `CommandId`, `CommandRequest`, `CommandResult`, command-specific types | Mirror/generated FastAPI contracts including `commandId`, expected/result versions, availability/cutoff, correction watermark, and metric class. |
| `web/src/api/client.ts` — `apiRequest`, `newCommandId`, `executeCommand` | Generate one command ID per user intent, preserve it across network retries, replay exact request, and distinguish stored replay from version/payload conflict. Components never regenerate IDs on automatic retry. |
| `web/src/pages/RiskReviewPage.tsx` | Top 20 seven-day recorded-complaint risks with reasons, baseline/model, cutoff, pinned correction state, freshness, and evidence links. |
| `web/src/pages/ClusterDetailPage.tsx` | Complaint/incident/location separation, evidence/confidence, current version, correction history, and command-safe merge/split dialogs. |
| `web/src/pages/AlertInboxPage.tsx` | Why-now, evidence, freshness, acknowledge/assign/defer, expiry, re-alert and command history. |
| `web/src/pages/EvaluationPage.tsx` | Balanced scorecard with prospective forecast results visibly separated from retrospective corrected gold-set clustering results and staff usefulness. |
| `web/src/components/EvidenceBadge.tsx`, `FreshnessBanner.tsx`, `ClusterCorrectionDialog.tsx`, `AlertActions.tsx`, `CommandStatus.tsx` | Consistent provenance/freshness and retry-safe mutation UX. Disable duplicate submission locally but rely on server idempotency for correctness. |
| `web/src/demo/` | Synthetic-only fixtures; no real-address bundle. |

### Documentation, generated artifacts, and tests

| File | Disposition |
|---|---|
| `README.md` | Replace causal watchlist pitch with resolved staff workflow/stack, local run path, evidence classes, seven-day top-20 contract, and refresh limitations. |
| `docs/parking-hotspots/product.md` | Supersede infrastructure-only “Final” framing; document coordinator flow, corrections, inbox, scorecard, staff/public boundary, and non-goals. |
| `docs/parking-hotspots/decisions.md` | Record all resolved intent plus correction-time semantics, command ID contract, v1 source contract, and legacy-output disposition. |
| `docs/parking-hotspots/data-sources.md` | Remove join-novelty and driver-identity claims; distinguish field meaning from interpretation; replace fixed UTC-3 advice with IANA display conversion; document cohort/run receipts, source cadence/licence verification, and v1 source contract including 311/boundary restrictions. |
| `out/watchlist.md` | Tracked but obsolete. Stop shipping current causal/no-fix copy. At Milestone 0 mark it legacy/non-authoritative or remove it from current product references; once new export exists, replace/regenerate with evidence-safe wording and run/cutoff/provenance. Do not retain invented backward-compatible semantics. |
| `out/watchlist.csv` | Tracked, contains real addresses and undocumented derived fields. Stop treating it as product state. Remove from public/demo distribution and current links; replace with a staff-only generated export carrying run/cutoff/schema/provenance when needed. Decide in Milestone 0 whether such exports are ignored build outputs or deliberately tracked snapshots; record the policy and access implications. |
| `src/hrm_parking/exports.py` — `export_top_risk`, `export_claim_manifest` | Generate optional staff exports only from a successful pinned run and emit a sidecar/embedded claim manifest. No causal prose or weak vehicle identity claim. |
| `tests/unit/`, `tests/integration/`, `tests/api/` | Domain, bitemporal correction, persistence, command replay, source contract, provenance, and API tests. |
| `web/src/**/*.test.tsx`, `web/e2e/` | Mutation-retry ID reuse, replay UI, version conflicts, top-20, correction/inbox/freshness/provenance flows. |

## Sequencing and dependencies

### Milestone 0 — Contract, claim cleanup, and persistent shell

1. Record resolved product decisions, review resolutions, v1 data-source table, and non-goals in product/decision/data-source docs.
2. Run a claim-ledger review across `README.md`, product/decision/data-source docs, `src/hotspots.py` emitted copy, and tracked `out/watchlist.*`. Map each assertion to public source, reproducible derivation, proposal, or synthetic scenario; remove unsupported causal/identity claims.
3. Choose and document the replacement policy for tracked `out/watchlist.*`: recommended default is remove legacy files from current product references/public distribution, make future staff exports generated/ignored, and preserve reproducibility through run metadata rather than committed real-address snapshots. If snapshots must remain tracked, make access intent explicit and regenerate only from a pinned successful run with provenance.
4. Create Python packaging, migrations, FastAPI app factory, SQLite connection, React/Vite shell, shared evidence/version/command vocabulary, `GET /api/status`, and a freshness empty state.

**Exit:** Migrated app/API/UI starts against SQLite; no current product surface presents obsolete watchlists as authoritative; artifact disposition is recorded; no fabricated results appear.

### Milestone 1 — Immutable Cityworks ingestion and source contract

1. Extract ArcGIS access from `src/hotspots.py`; implement stable paging, retries, reconciliation, and run/source watermarks.
2. Centralize/version the citywide blocked-driveway predicate and reconcile existing 9,729/9,791 counts by filter and as-of rather than editing toward one value.
3. Persist source complaint/custom-field versions without flattening one-to-many history; define repeated/missing/changed-field access rules.
4. Preserve UTC and convert to `America/Halifax` only for display.
5. Add staff source view and enforce staff-only real-address responses. Do not ingest 311 or boundaries into cluster/risk feature tables.

**Exit:** A successful source run is reproducible and freshness-safe; failed/partial runs cannot advance current state; source restrictions are enforceable in code boundaries.

### Milestone 2 — Correctable clustering plus command idempotency

1. Propose incident candidates then recurring-location clusters with separate IDs/counts, membership evidence, confidence, and configuration version.
2. Implement shared `workflow_commands` transaction semantics before exposing mutations.
3. Add cluster list/detail UI and merge/split APIs/dialogs. Require `commandId`, expected version, reason, and members.
4. Persist correction `available_at` and `effective_from`; materialize immutable point-in-time cluster snapshots. Reapply corrections on later proposals only through explicit replay/conflict rules.
5. Review ambiguous fixtures including duplicate reports, distinct nearby entrances/opposite sides, alternate spellings, missing coordinates, and cross-day recurrence.

**Exit:** Coordinator corrections survive restart, are reconstructable, and do not mutate source. Exact command retry returns the stored result with one correction event; changed reuse conflicts. Snapshot queries reproduce correction knowledge at any cutoff.

### Milestone 3 — Baseline-first top-20 seven-day risk

1. Freeze label: same type of recorded complaint for the pinned location cluster in `(cutoff, cutoff + 7 days]`.
2. Each forecast/backtest chooses a source watermark and `cluster_snapshot_id` composed only from source/corrections available at cutoff. Later `available_at` corrections never enter prospective features, labels, ranks, or metrics.
3. If later staff corrections create a better grouping gold set, calculate separately labeled `retrospective_gold` cluster/forecast diagnostics; never substitute them for prospective chronological performance.
4. Implement cutoff-safe features, `RecentFrequencyBaseline`, deterministic ties, rolling chronological splits, persisted eligible ranks, and top-20 output.
5. Build risk API/UI showing reasons, target/cutoff/freshness, baseline/model, cluster snapshot, correction watermark, and “recorded complaints, not all violations.”
6. Establish minimum lift/stability promotion gates from observed baseline behavior and staff capacity. Only then test a candidate model; show probability only after calibration.

**Exit:** Historical runs reproduce the top 20 from knowledge available then; prospective and retrospective-gold metrics cannot be confused; no candidate is promoted without the recorded gate.

### Milestone 4 — Command-safe in-app alert inbox

1. Approve deterministic triggers and state table for create, acknowledge, assign, defer, defer expiry, alert expiry, trigger persistence, re-alert, materially new evidence, and invalid/repeated commands.
2. Generate only v1 new-hotspot, meaningful risk/top-20 increase, and recurrence-after-closure alerts from pinned successful analysis state.
3. Implement inbox/detail and actions. Each client mutation carries a stable `commandId` and expected version through schema, FastAPI route, transaction, event, response, TS type, and UI retry.
4. Store exact command results. Exact replays return the stored original result even if the entity has since advanced; mismatched ID reuse is a conflict. System expiry/re-alert uses deterministic internal IDs.
5. Display why-now, evidence, uncertainty, freshness/cutoff, expiry, history, and a coordinator-owned next action. Materially new safety/access evidence bypasses ordinary suppression.

**Exit:** Triage persists across restart; retries produce one event; version conflicts are visible; unchanged analyses produce no duplicate logical alert; refresh behavior is explicitly non-real-time.

### Milestone 5 — Synthetic scenarios and balanced scorecard

1. Add deterministic fictional cases covering duplicate reports, recurrence, ambiguity, missing outcomes, false alerts, non-recurring controls, late/unknown response, and recurrence after documented action.
2. Enforce synthetic separation in database, API, UI, exports, and evaluation queries.
3. Present scorecard sections separately: reviewed clustering quality (including retrospective gold-set views), prospective chronological baseline/candidate performance, and coordinator-rated alert usefulness.
4. Conduct coordinator walkthroughs of corrections, top-20 review, alert actions, retry behavior, and evidence/freshness understanding.

**Exit:** Demo contains success and failure modes; real versus synthetic and prospective versus retrospective metrics are unmistakable.

### Milestone 6 — Pilot hardening

1. Inspect any scheduled workflow before changing it; run transactional ingest/analyze and preserve failed/stale state.
2. Validate source cadence, licence/redistribution, and any district boundary edition before use.
3. Establish staff authentication/authorization before real-address endpoints are network-accessible beyond controlled local use.
4. Define SQLite migration, backup/recovery, retention, WAL/busy-timeout, and supported concurrency.
5. Expand violation types only via a versioned cohort and fresh clustering/forecast evaluation.

**Exit:** Failures are visible, stale results cannot look fresh, real addresses are staff-restricted, and operating/recovery procedures are proven for the bounded pilot.

## Acceptance criteria

### Persistent product and source contract

- SQLite-backed FastAPI and React/Vite state survives process restart.
- Staff landing provides top-20 seven-day risk and inbox with cluster evidence links.
- Real addresses are absent from public/demo endpoints and static bundles.
- Cityworks requests/custom fields are the only v1 cluster/risk inputs. Tests fail if 311 or boundary attributes enter a v1 feature vector; optional context responses identify their independent source and do not imply a join.
- District geometry is not displayed/aggregated until edition/effective dates/identifiers are recorded; it never defines v1 clusters or risk.
- Sync completion, newest source timestamp, analysis cutoff, source watermark, and seven-day target window are distinct.

### Evidence, artifacts, and claims

- Every record/response/export has enforceable public/derived/synthetic provenance and run/version lineage.
- No current doc, UI, API description, or generated output asserts tow ineffectiveness, no enforcement action, unique drivers/vehicles, durable resolution, staff failure, guaranteed physical fix, join novelty, or real-time detection.
- `data-sources.md` matches the v1 source contract and field limitations.
- `out/watchlist.*` is no longer shipped or linked as authoritative legacy output. Any replacement staff export is generated from a pinned successful run, includes schema/cohort/source/cutoff/provenance metadata, and is excluded from public/demo distribution.
- A claim-ledger check scans/inspects generated Markdown/CSV headers/manifests as well as hand-written docs and UI copy.
- Conflicting counts are explained by predicate/as-of or remain explicitly unresolved; none is silently selected.

### Clustering, corrections, and cutoff semantics

- Complaints, incident candidates, and location clusters have distinct identities/counts and explainable memberships.
- Merge/split requires coordinator, reason, `commandId`, expected version, `available_at`, and `effective_from`; server owns trusted receipt/availability timestamp while domain-effective time is validated.
- Each exact retried command produces one correction event and returns the stored response. Reusing its ID for a different payload/actor/kind conflicts.
- Immutable cluster snapshots identify proposal versions and correction watermark. A forecast run references exactly one snapshot available at cutoff.
- A correction recorded after cutoff cannot change that run's features, label membership, rank, or prospective metrics even when `effective_from` predates cutoff.
- Retrospective later-corrected gold-set results have a different metric class and are never presented as prospective forecast performance.
- Stale/concurrent correction requests cannot overwrite newer state; replay conflicts are surfaced for human review.

### Prediction

- Target is exactly same-type recorded complaint within seven days for the pinned cluster snapshot.
- API/UI return exactly top 20 when at least 20 are eligible, else all with eligible count; ties are deterministic.
- Feature generation uses only public/source versions and corrections available at cutoff. Post-cutoff source, response, or human-correction changes cannot alter a saved prospective feature/rank.
- Baseline and candidate use identical cutoff-scoped eligible cases; chronological top-20 metrics, coverage, missingness, and stability are reported.
- Promotion thresholds are recorded from baseline/pilot evaluation before candidate promotion. Failure leaves baseline active and visible.
- Numeric probabilities require calibration evidence. Intake time is never physical event/patrol/arrival time.

### Alert inbox and commands

- Merge, split, acknowledge, assign, and defer request schemas and TS types require `commandId` and expected version.
- One client user intent creates one ID; automatic retries reuse it. Server persists globally unique ID, canonical request hash, status, and exact result transactionally with the domain event.
- Exact replay returns stored result without a second mutation; mismatched reuse returns a defined conflict. Crash/recovery tests cannot leave a committed mutation without its replayable receipt.
- Alert actions persist across restarts and append actor/time/prior/new version state.
- Trigger, defer/expiry, unchanged rerun, re-alert, and materially new evidence follow one documented deterministic table.
- Each alert explains why now, evidence and cluster version, uncertainty, freshness/cutoff, expiry, and next action.
- Email/Teams remain absent except as deferred roadmap notes.

### Synthetic data and balanced scorecard

- Synthetic records use fictional locations and explicit synthetic IDs/badges; they cannot enter real source, prospective forecast, or clustering metrics.
- Demo includes ambiguity, false alerts, missing data, controls, and recurrence after response.
- Scorecard reports reviewed cluster quality, prospective baseline/candidate performance, retrospective gold-set diagnostics, and staff-rated usefulness as separate sections—not one accuracy number.

## Verification

- **Domain/unit:** cohort predicate; UTC/DST; one-to-many fields; provenance; membership invariants; correction `available_at`/`effective_from`; point-in-time snapshot materialization; top-20 ties; leakage; fingerprints/state table; command request hashing/replay; synthetic exclusion; source-use allowlist.
- **Persistence/migrations:** clean upgrade; uniqueness/foreign keys; immutable source history; rollback on partial ingest/analysis/command; global `commandId` constraint; atomic command/event/result; restart; cluster correction replay/conflict; SQLite concurrency settings.
- **FastAPI contracts:** staff/public address boundary; fixed v1 risk semantics; correction and alert `commandId`; exact replay and mismatched conflict; expected-version conflict; prospective versus retrospective result types; 311/context isolation.
- **React:** ID generated once per user intent and reused on retry; replay/result/conflict UI; evidence/freshness; complaint/incident counts; correction history; top 20; alert transitions; metric-class labels.
- **Browser:** pinned ingest → cluster review → merge/split → retry same command → top 20 → alert acknowledge/assign/defer with retry → restart → confirm one event and retained result/history.
- **Chronological evaluation:** reconstruct source and correction knowledge at every cutoff; compare baseline/candidate on same cases; assert later corrections have no effect; separately compute retrospective gold diagnostics.
- **Claim/artifact review:** inspect docs, UI text, API descriptions, `data-sources.md`, and every generated Markdown/CSV/manifest; fail publication on prohibited claims or missing run/provenance.
- **Human review:** coordinator assesses grouping, reasons, authority of actions, false-alert burden, retry behavior, and re-alert/suppression usefulness.

No tests, linters, formatters, live API calls, dependency changes, or product mutations were performed in this planning pass.

## Escalation/Risk Gate

### Resolved decisions

There are no material-open product choices. User, cohort, seven-day/top-20 contract, in-app workflow, stack, correction authority, and balanced scorecard remain fixed.

### Constrained discovery

Record these before their corresponding gate without reopening intent:

- Detailed match parameters and reviewed false-merge/false-split tradeoff.
- Baseline-derived lift/stability promotion thresholds.
- Alert trigger, expiry, defer, and re-alert durations consistent with review capacity.
- Source cadence/change handling and licence/redistribution terms.
- Boundary edition/effective dates if context mapping is enabled.
- Final tracked-versus-ignored policy for replacement staff exports; default is generated/ignored and staff-restricted.
- UI accessibility/design, staff access mechanism, retention, and SQLite recovery/concurrency bounds.

Escalate only if evidence contradicts the fixed contract—for example, the source cannot support the label, top 20 is unusable at coordinator capacity, or staff-only address access cannot be enforced. Do not substitute analytics-only artifacts, patrol timing, public real addresses, or email delivery.

### Handoff guidance

- **Executor:** implement milestones in order. Establish commands and immutable/persistent foundations before mutation endpoints; establish baseline before candidate modeling.
- **Architect:** gate bitemporal correction/snapshot design, atomic command receipt semantics, staff/public isolation, and SQLite concurrency before Milestones 2–4.
- **Critic:** verify claim/artifact cleanup, source contract, prospective/retrospective separation, and falsifiable scorecard gates.
- **Autoresearch:** only after the chronological baseline harness and cutoff-scoped snapshots exist, for bounded clustering/model experiments that cannot auto-promote.
- **Ultragoal:** only for a separately approved coordinated end-to-end build/pilot, not an individual milestone.

## Verification Plan

Likely executor command surface after project setup:

```bash
python -m pytest tests/unit -q
python -m pytest tests/integration -q
python -m pytest tests/api -q
npm --prefix web test -- --run
npm --prefix web run typecheck
npm --prefix web run build
npm --prefix web run e2e
python -m pytest
```

Required scenario replays:

1. **Source:** successful and failed/partial ingestion, repeated custom fields, changed source row, stable rerun, count reconciliation, and forbidden 311/boundary feature injection.
2. **Correction knowledge:** correction before cutoff; correction recorded after cutoff but effective before it; later correction conflict; snapshot reproduction; prospective result unchanged; separately labeled retrospective gold set changed.
3. **Command retry:** merge/split/acknowledge/assign/defer succeeds but client loses response; exact retry returns stored response with one event; changed-payload and changed-actor reuse conflicts; crash at transaction boundaries leaves either neither mutation nor both event and receipt.
4. **Risk:** fewer/more than 20 eligible, deterministic ties, later source/correction mutation, rolling cutoffs, drift slices, candidate below/above recorded gate.
5. **Alerts:** unchanged analysis, meaningful crossing, recurrence after closure, repeated command, defer/expiry, qualifying re-alert, materially new safety/access evidence, stale source, restart.
6. **Artifact:** legacy watchlist references removed; replacement export staff-restricted; all rows/headers/manifests carry required run/cutoff/provenance; prohibited copy absent.
7. **Evidence/access:** public + derived + fictional records in one database remain labeled and metric-separated; public/demo endpoints and bundles cannot reveal real addresses.

Required pre-pilot evidence:

- Ingestion reconciliation/source-freshness report and v1 source-use audit.
- Reviewed clustering set and correction audit/snapshot replay.
- Prospective baseline/candidate report plus separately labeled retrospective gold diagnostics and promotion decision.
- Command-idempotency and alert transition replay reports.
- Synthetic-separation report.
- Claim ledger covering handwritten and generated outputs.
- Coordinator walkthrough results for clustering, top-20 risk, inbox usefulness, and retry/re-alert behavior.

## Risks and mitigations

| Risk | Consequence | Mitigation |
|---|---|---|
| Future coordinator corrections leak into historical evaluation | Inflated backtest quality | `available_at`/`effective_from`, immutable cutoff snapshots, correction watermark, and separate retrospective gold metrics. |
| Client retry duplicates a successful mutation | Duplicate corrections/events or misleading conflicts | Required global `commandId`, atomic stored result, canonical request hash, exact replay, mismatched conflict, and TS retry reuse. |
| Unjoinable 311 or unvalidated boundaries become predictive inputs | Misleading lineage/features | Enforced source allowlist; context-only endpoints; boundary validation gate; feature-injection tests. |
| Legacy tracked watchlists continue to ship unsupported claims/real addresses | Reputational/access harm | Remove current references/distribution, choose recorded export policy, regenerate only provenance-safe staff artifacts, and include generated files in claim checks. |
| Persistent vertical slice expands into platform work | Evidence/workflow delayed | One SQLite/FastAPI/React app, strict milestone exits, defer production-scale architecture. |
| Corrections conflict with re-analysis | Staff work lost or misapplied | Stable source IDs, versioned snapshots, append-only corrections, deterministic replay and surfaced conflict queue. |
| Address grouping is inaccurate | Inflated or fragmented recurrence/risk | Two-level explainable grouping, hard reviewed cases, confidence, audited merge/split. |
| Mutable paginated source is irreproducible | Counts/ranks shift silently | Versioned source history, watermarks, reconciliation, pinned runs, failed-state visibility. |
| Weak vehicle/tow/closure fields drive causal claims | Misleading operational conclusions | Preserve missingness, exclude identity/causal scoring, correct docs/artifacts, claim ledger. |
| Intake time is mistaken for event time | Bad patrol recommendations | Preserve UTC/IANA display, observed-target wording, feature review, explicit non-goal. |
| Baseline is weak or drifted | Top 20 marketed without evidence | Chronological measures, stability slices, derived gate, baseline visibility, no unsupported probability. |
| Alert fatigue/suppression | Alerts ignored or material evidence hidden | Capacity-aware triggers, deterministic transition table, logical fingerprints, new-evidence override, staff review. |
| SQLite contention/migration failure | Lost workflow state | Short transactions, WAL/busy-timeout evaluation, migration/backup tests, documented single-writer bound. |
| Real addresses leak to public demo | Privacy/operational exposure | Separate schemas/routes/build fixtures, fail-closed configuration, staff auth gate, access tests. |
| Synthetic records contaminate real metrics | Invalid performance claims | Evidence constraints, fictional locations, query guards, UI badges, metric-class tests. |
| Stale source looks current | Staff acts on old rankings | Distinct sync/source/cutoff timestamps, failed-run status, no promotion on partial runs. |

## Claimed review resolutions

| Review finding | Resolution pointers in this plan |
|---|---|
| `ARCH-HRM-PARKING-S02-001` — correction availability and backtest leakage | Principles 3; persistent `ClusterCorrection`, `ClusterSnapshot`, `ForecastRun`; Milestone 3 steps 1–3; clustering/prediction acceptance; correction-knowledge replay. |
| `ARCH-HRM-PARKING-S02-002` — mutating workflow command idempotency | Principle 4; `WorkflowCommand`, correction/alert models; `commands.py`; FastAPI and TS contracts; Milestones 2/4; alert-command acceptance and retry replay. |
| `ARCH-HRM-PARKING-S02-003` — auxiliary source disposition | V1 data-source contract; source ingestion files; Milestone 1 step 5; persistent source acceptance; source-use verification. |
| `critic-stage02-F001` — data-source docs/generated artifacts | Repository findings; Documentation/generated artifacts table; Milestone 0 steps 1–3; evidence/artifact acceptance; claim/artifact verification and replay. |

## Final recommendation

Proceed with **Option A, incremental persistent vertical slices**, and treat the four review resolutions as implementation contracts rather than optional notes. Establish the migrated app shell and claim/artifact cleanup, then immutable Cityworks ingestion, cutoff-correct audited clustering, baseline-first seven-day top-20 risk, and a command-safe in-app inbox. Candidate modeling begins only after the prospective baseline harness and correction-aware cutoff snapshots are proven.


## Intent Reconciliation

- Primary user: parking enforcement operations coordinator who reviews recurring complaints and risk increases, then selects candidates for site checks or enforcement.
- Initial cohort: citywide blocked-driveway complaints. Real public addresses are restricted to staff-facing views.
- Forecast contract: rank risk of the same type of recorded complaint within seven days and present the top 20 for review; minimum lift/stability gates are established honestly during baseline evaluation.
- Alert workflow: persistent in-app inbox with acknowledge, assign, defer, expiry, and deterministic re-alert; email and Teams are deferred.
- Architecture: Python FastAPI and analytics, SQLite persistence, React + Vite TypeScript frontend.
- Pilot success: balanced scorecard across reviewed cluster quality, forecast performance against a chronological baseline, and staff-rated alert usefulness.
- Open confirmations: none.

## ADR

### Decision
Build an incremental persistent vertical slice for HRM parking enforcement: immutable Cityworks ingestion, correctable two-level clustering, cutoff-safe seven-day top-20 recorded-complaint risk ranking, and an idempotent in-app alert inbox using Python FastAPI, SQLite, and React/Vite TypeScript.

### Drivers
1. The product must make Predict, Clustering, and Alerting operational for a named HRM user without overstating what complaint records measure.
2. Every grouping, forecast, correction, and alert must be reproducible, explainable, temporally valid, and traceable to source evidence.
3. The first version must validate staff usefulness end to end while remaining small enough to evaluate and revise.

### Alternatives considered
- Analytics-only batch artifacts: rejected as the product boundary because they cannot support durable merge/split corrections or alert workflow state.
- Backend-first with UI deferred: retained only as a delivery fallback; it delays validation of coordinator review and inbox semantics.
- Intake-time patrol prediction: rejected because Cityworks `DATE_INITIATED` reflects intake/recording time rather than observed street-event time.
- Complex ML before a baseline: rejected until chronological evaluation demonstrates meaningful, stable improvement.

### Why chosen
The selected architecture is the smallest end-to-end system that satisfies the confirmed core capabilities and resolved workflow. It preserves the useful ArcGIS ingestion groundwork while replacing unsupported causal language, address-only recurrence assumptions, fixed-offset time handling, and non-persistent watchlists with explicit domain, lineage, evaluation, and workflow contracts.

### Consequences
- The repository gains Python API/analytics modules, SQLite schema/migrations, and a TypeScript UI rather than remaining a single dependency-free script.
- Corrections and workflow commands require append-only audit history, point-in-time availability, optimistic versioning, and command-idempotency keys.
- Forecast claims remain limited to recorded complaints; ordinal/ranked risk remains the default until calibration supports probabilities.
- Real addresses require staff-only presentation; synthetic scenarios remain clearly separated and cannot enter real metrics.
- Cityworks requests/custom fields are the only v1 cluster/risk inputs; 311 and district layers stay context-only or gated until validated.

### Follow-ups
- Validate HRM Hub publication/change cadence, Open Data Licence terms, and the correct district-boundary edition.
- Establish clustering thresholds and minimum forecast lift/stability gates from reviewed fixtures and chronological evaluation, not demo convenience.
- Conduct a coordinator walkthrough to score alert actionability and confirm merge/split and inbox semantics before broader rollout.
- Consider email/Teams delivery, production identity/hosting, and intervention-effect study only after the v1 balanced scorecard is met.
