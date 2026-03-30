---
RFC: 0003
Title: Kotlin Data Mapper Library for RADAR-base Export Conversion
Author(s): <Yatharth Ranjan (@yatharthranjan)>
Status: Draft
Created: 2026-03-30
Updated: 2026-03-30
Discussion: N/A
---

Summary
-------
This RFC proposes a Kotlin data-mapper library that converts RADAR-base flattened CSV output (with supporting AVRO schemas) into destination data formats with associated schemas. The first implementation target is REDCap (data dictionary driven), delivered with a CLI to run conversions and write outputs to disk. The design uses abstraction-oriented, object-oriented architecture so additional destination adapters (FHIR, HL7, CDISC) and a later REST API layer can be added without redesigning the core. The mapper is explicitly designed for adoption by `radar-output-restructure`, with planned reuse and extraction of common components.

Motivation
----------
RADAR-base export data is rich and structured, but downstream systems require format-specific schemas and field conventions. Teams currently need custom ad hoc scripts for each destination and project context. This leads to duplicated logic, low traceability, and brittle mappings when schemas evolve.

The existing `radar-output-restructure` codebase already provides mature abstractions for plugin loading, format conversion, storage access, and CLI/configuration. Reusing and extracting those common capabilities reduces duplication and lowers operational risk while introducing mapper-specific functionality.

We need a reusable conversion engine that:
- Enforces schema-aware mapping.
- Supports composition of multiple RADAR data types into one destination representation.
- Supports external enrichment data for fields not present in source exports (for example REDCap event names).
- Preserves provenance and validation evidence for auditable conversion.

Non-Goals
---------
- Delivering a REST API in the first release (library + CLI only).
- Fully implementing FHIR, HL7, and CDISC adapters in MVP.
- Building a GUI or no-code editor for mappings.
- Designing a broad expression language in v1 beyond essential transforms needed for MVP.

Guide-level explanation
-----------------------
The mapper is a Kotlin library with clear interfaces and pluggable adapters:

1. Read RADAR flattened CSV plus AVRO schema metadata.
2. Normalise rows into a canonical intermediate model.
3. Merge multiple source streams/data types when required.
4. Enrich missing fields from external sources (Excel sheets, REDCap dictionary exports, etc.).
5. Transform canonical data to destination schema fields.
6. Validate destination records.
7. Write converted output to disk and emit a conversion report.

The CLI (`radar-mapper`) exposes this flow through commands such as:
- `validate`: checks source/destination schema compatibility, mapping config integrity, and enrichment availability.
- `convert`: executes end-to-end conversion and writes destination output.
- `dry-run`: performs conversion logic without final write, producing diagnostics and sample output.

MVP behaviour:
- Destination: REDCap only.
- Support combining multiple RADAR data types into one REDCap-ready output when mapping rules require it.
- Support external enrichment for event names and related metadata.
- Emit structured diagnostics for missing mappings, unresolved enrichment keys, type conversion errors, and record-level validation failures.

Reference-level design
----------------------
### Architecture

The design follows dependency inversion with interface-driven components:

- `SourceReader`
  - Reads source data and associated source schema metadata.
  - MVP implementation: `CsvAvroSourceReader`.
- `SchemaProvider`
  - Loads and resolves source and destination schema descriptors.
  - MVP implementations: `RadarSchemaProvider`, `RedcapSchemaProvider`.
- `CanonicalNormalizer`
  - Converts raw records into canonical model objects.
- `MergeStrategy`
  - Combines records from multiple source data types into one logical destination context.
- `EnrichmentProvider`
  - Resolves missing values from external inputs with precedence rules.
  - MVP implementations: `ExcelEnrichmentProvider`, `RedcapDictionaryEnrichmentProvider`.
- `TransformationEngine`
  - Applies declarative mapping rules and typed transforms.
- `DestinationValidator`
  - Validates transformed records against destination schema constraints.
- `DestinationWriter`
  - Writes destination data and metadata to disk.
  - MVP implementation: `RedcapWriter`.
- `ConversionOrchestrator`
  - Coordinates pipeline lifecycle and error handling.

Later REST API support is enabled by reusing the same orchestration and service abstractions behind transport-specific controllers.

### Reuse and extraction strategy with `radar-output-restructure`

The mapper programme should prioritise reuse of existing components where possible, then extraction of neutral components into shared libraries where direct reuse is not feasible.

#### Reuse as-is in the first mapper release

- Plugin SPI pattern from `org.radarbase.output.Plugin`.
- Dynamic plugin/factory instantiation pattern from `PluginConfig` and extension utilities.
- CLI parsing pattern from `CommandLineArgs` and startup orchestration approach from `Application`.
- YAML configuration loading and classpath fallback approach from `YAMLConfigLoader`.
- `FormatProvider` pattern for format/compression/adapter discovery.
- Storage abstractions from `SourceStorage` and `TargetStorage` for file/object store integration.

#### Extract as common libraries for long-term reuse

To support bidirectional reuse between mapper and `radar-output-restructure`, define shared modules:

1. `radar-output-common-plugin`
   - Shared plugin SPI, reflective loading utilities, and plugin configuration contracts.
2. `radar-output-common-io`
   - Shared storage abstractions (`SourceStorage`, `TargetStorage`) and portable resource wrappers.
3. `radar-output-common-format`
   - Shared format-provider contracts and base conversion interfaces.
4. `radar-output-common-cli-config`
   - Shared command-line/configuration loading primitives and environment override utilities.

Mapper-specific modules (`radar-data-mapper-core`, destination adapters, enrichment providers) depend on these common modules. `radar-output-restructure` then migrates to consume the same common modules incrementally.

#### Integration plan for `radar-output-restructure`

- Phase A: integrate mapper as a dependency and add a mapper execution mode in `radar-output-restructure`.
- Phase B: switch selected conversion paths to mapper-backed transformation while preserving existing output restructuring behaviour.
- Phase C: migrate duplicated plugin/config/IO primitives in `radar-output-restructure` to extracted common libraries.
- Phase D: retire superseded local implementations after compatibility validation.

#### Implementation work packages (cross-repository)

The following work packages are intended to be tracked as epics/issues across both repositories.

**WP1 - Mapper core and contracts (new mapper repository)**
- Define core interfaces: `SourceReader`, `SchemaProvider`, `CanonicalNormaliser`, `MergeStrategy`, `EnrichmentProvider`, `TransformationEngine`, `DestinationWriter`.
- Implement `ConversionOrchestrator` and shared error/result model.
- Add mapper configuration schema and validation for mapping, merge, and enrichment rules.
- Deliver unit tests for contract behaviour and error pathways.

**WP2 - REDCap destination adapter MVP (new mapper repository)**
- Implement REDCap schema adapter and dictionary-driven field validation.
- Implement REDCap writer and output serialisation format.
- Implement external enrichment providers for Excel and REDCap dictionary exports.
- Deliver end-to-end golden tests from RADAR input to REDCap output.

**WP3 - Common component extraction (new `radar-output-common-*` modules)**
- Extract plugin/configuration primitives into `radar-output-common-plugin` and `radar-output-common-cli-config`.
- Extract storage abstractions into `radar-output-common-io`.
- Extract format/provider abstractions into `radar-output-common-format`.
- Publish versioned artifacts and migration notes for consumers.

**WP4 - Mapper CLI and operability (new mapper repository)**
- Implement `validate`, `convert`, and `dry-run` commands.
- Add structured reporting (JSON + human-readable summary) and deterministic output options.
- Add observability hooks (run metrics, structured warnings/errors) and secure logging defaults.
- Package CLI for CI and containerised runtime usage.

**WP5 - `radar-output-restructure` integration (radar-output-restructure repository)**
- Add mapper dependency and feature flag/config toggle for mapper execution mode.
- Integrate mapper invocation into processing pipeline without breaking existing path/bucket organisation.
- Add side-by-side comparison mode for legacy conversion output versus mapper output.
- Add fallback logic to legacy conversion path on mapper hard failure, with explicit diagnostics.

**WP6 - Convergence and de-duplication (both repositories)**
- Replace duplicated local abstractions in `radar-output-restructure` with `radar-output-common-*` modules.
- Deprecate superseded local implementations after parity and stability criteria are met.
- Finalise compatibility matrix (mapper version, common libraries version, restructure version).
- Execute staged rollout and rollback drills before defaulting mapper mode on.

**Suggested ownership and sequencing**
- Sequence: WP1 -> WP2 -> WP4 -> WP5 -> WP3 -> WP6 (WP3 may begin earlier if extraction is low risk).
- Primary ownership: mapper team for WP1/WP2/WP4; restructure maintainers for WP5; shared ownership for WP3/WP6.
- Each work package should define entry/exit criteria, test evidence, and rollback notes before closure.

### Canonical intermediate model

The library introduces a destination-neutral canonical model with explicit provenance:
- `CanonicalRecord`
- `CanonicalField`
- `CanonicalValue`
- `SourceProvenance` (source file, column, schema version, timestamp where available)

This keeps source parsing and destination writing decoupled and supports adding new destinations with minimal changes.

### Mapping and transformation configuration

Mappings are defined in a declarative configuration (YAML or JSON), including:
- source field references (single or composite),
- destination field target,
- transformation pipeline,
- null/default policy,
- validation expectations,
- enrichment key bindings.

MVP transform set includes type coercion, date/time formatting, controlled vocabulary mapping, concatenation/splitting, and constant/default injection.

### Multi-source merge behaviour

When required by mapping configuration, multiple source streams are merged using configured join keys and conflict resolution policy:
- deterministic precedence order,
- null-aware merge rules,
- optional timestamp-driven selection.

Merge decisions are logged in diagnostics for auditability.

### External enrichment behaviour

External enrichment is modelled as a provider chain:
1. Inline mapping override
2. External files (Excel, dictionary export)
3. Default fallback

If required enrichment cannot be resolved, behaviour is configurable:
- fail conversion,
- skip record with error report,
- emit warning and continue (only for non-required fields).

### Error model and reporting

The library returns structured results:
- converted record outputs,
- warnings,
- errors with category and provenance,
- aggregate metrics (processed, converted, skipped, failed).

CLI writes a machine-readable report (JSON) and optional human-readable summary.

Compatibility and migration
---------------------------
Backward compatibility is handled via versioned mapping configs and explicit schema compatibility checks:
- source schema version support matrix,
- destination schema version support matrix,
- mapping config versioning and migration notes.

As new destination adapters are introduced, existing REDCap mappings remain unaffected unless users opt into new mapping versions.

For `radar-output-restructure` adoption:
- mapper integration is introduced behind a feature flag/config toggle;
- existing restructure flows remain default until mapper parity is proven;
- migration includes side-by-side output comparison tests for representative topics;
- rollback is achieved by disabling mapper mode and returning to current conversion path.

Alternatives considered
-----------------------
1. Direct source-to-destination conversion without canonical model
   - Pros: lower initial complexity.
   - Cons: duplicates transformation logic, harder to extend to FHIR/HL7/CDISC, weaker testability.

2. Separate standalone converter per destination
   - Pros: straightforward destination-specific optimisation.
   - Cons: code duplication and inconsistent behaviour for merge/enrichment/validation concerns.

3. Canonical model + adapter architecture (chosen)
   - Pros: extensibility, reusable pipeline logic, better isolation and testability, easiest migration to REST exposure.
   - Cons: additional upfront design effort.

Operational considerations
--------------------------
- Deliver as a versioned Kotlin library artifact plus CLI distribution.
- Include deterministic output options for reproducible runs.
- Provide configuration examples for REDCap mapping and enrichment files.
- Add telemetry hooks for run metrics (for later API/ops integration).
- Rollback strategy: pin to previous mapper version and previous mapping config version.
- Publish compatibility notes for `radar-output-restructure` integration, including minimum supported version and feature-flag defaults.

Security and privacy
--------------------
- Treat all data as sensitive by default.
- Prevent raw sensitive values from being emitted to logs by default.
- Support configurable field redaction in diagnostics.
- Avoid retaining temporary files beyond run scope unless explicitly configured.
- Document expected handling for protected health information in deployment and operations guides.

Testing strategy
----------------
Testing includes:
- Unit tests for each interface contract (`SourceReader`, `MergeStrategy`, `TransformationEngine`, `EnrichmentProvider`, `DestinationWriter`).
- Contract tests for schema validation behaviour.
- Integration tests for end-to-end REDCap conversion with representative RADAR datasets.
- Golden-file tests for deterministic output and diagnostics.
- Performance and load tests on multi-file conversion workloads.
- Negative-path tests for invalid schemas, missing enrichment keys, and transform failures.
- Compatibility tests executed from `radar-output-restructure` consuming the mapper library.
- Golden-dataset parity tests comparing legacy restructure conversion outputs against mapper-backed outputs where semantics should match.

Success criteria for MVP:
- Converts RADAR flattened CSV + AVRO input to REDCap destination output with validated mappings.
- Supports configured multi-source merging.
- Supports external enrichment for required REDCap metadata.
- Produces actionable and auditable conversion reports.

Open questions
--------------
- What exact REDCap import format variant should be the canonical writer target in MVP?
- What minimum join keys are mandatory for multi-source merge in initial supported datasets?
- Should mapping configuration allow custom Kotlin plugins in MVP, or defer to post-MVP?
- What is the authoritative schema versioning policy when source metadata and external dictionaries diverge?
- Which initial RADAR data types from the commons schemas should be mandatory for MVP support?
- Which `radar-output-restructure` components should be migrated first to shared common modules to minimise release risk?
- Should common module extraction happen in this RFC scope, or as a follow-up RFC once mapper MVP is validated?

References
----------
- RADAR-base Schemas (commons): https://github.com/RADAR-base/RADAR-Schemas/tree/master/commons
- RADAR-base output restructure repository: https://github.com/RADAR-base/radar-output-restructure
- Plugin SPI: https://github.com/RADAR-base/radar-output-restructure/blob/main/src/main/java/org/radarbase/output/Plugin.kt
- Format provider/factory pattern: https://github.com/RADAR-base/radar-output-restructure/blob/main/src/main/java/org/radarbase/output/format/FormatProvider.kt
- CSV/AVRO conversion implementation: https://github.com/RADAR-base/radar-output-restructure/blob/main/src/main/java/org/radarbase/output/format/CsvAvroConverterFactory.kt
- Storage abstractions: https://github.com/RADAR-base/radar-output-restructure/blob/main/src/main/java/org/radarbase/output/source/SourceStorage.kt
- Target storage abstractions: https://github.com/RADAR-base/radar-output-restructure/blob/main/src/main/java/org/radarbase/output/target/TargetStorage.kt
- YAML config loading: https://github.com/RADAR-base/radar-output-restructure/blob/main/src/main/java/org/radarbase/output/config/YAMLConfigLoader.kt
- RADAR-base RFC Template: `rfcs/0000-template.md`
