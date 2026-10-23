# CarbonTrace

## Purpose

CarbonTrace is a read-only Model Context Protocol (MCP) server for investigating
computing energy consumption and estimated operational carbon emissions.

Scaphandre provides CPU and available memory-domain energy telemetry. A
supported GPU exporter supplies GPU telemetry on GPU-equipped hosts.
Electricity Maps provides grid carbon intensity. CarbonTrace joins those
sources and exposes reproducible analysis to an AI agent.

The agent investigates and explains. CarbonTrace does not schedule workloads,
change infrastructure, kill processes, or make operational decisions.

## User experience

Example questions:

- Which processes accounted for the most estimated energy yesterday?
- Why did estimated emissions change between these two periods?
- Did this build use less energy, or did it run with cleaner electricity?
- What does this footprint estimate include, and what evidence is missing?

Answers should contain quantities, units, time windows, sources, measurement
boundaries, and limitations. Unsupported explanations must remain hypotheses.

## Scope

### MVP

- One supported Linux host with working Scaphandre telemetry.
- GPU energy collection on supported GPU-equipped hosts; CPU-only hosts remain
  supported. Verify one GPU exporter/hardware combination for MVP.
- One explicitly configured Electricity Maps zone for that host.
- Prometheus for retained telemetry.
- Historical energy and estimated emissions analysis.
- Host and process views, with process attribution labeled as estimated.
- Four read-only MCP tools, exposed over local `stdio`.
- An existing MCP client supplies the AI agent; no embedded LLM.

### Non-goals

- Automatic scheduling, region selection, or infrastructure changes.
- Complete lifecycle accounting, embodied emissions, or carbon offsets.
- Exact wall-plug, cooling, storage, or network energy unless separately measured.
- Universal GPU vendor support or GPU process attribution without suitable
  telemetry.
- Deployment attribution without deployment evidence.
- Multi-tenant hosting, remote MCP transport, or cloud billing integration.
- Building a dashboard before the MCP experience works.

## Architecture

```text
Supported Linux host
  Scaphandre exporter
  Supported GPU exporter (when applicable)
       |
       v
  Prometheus ------------------+
                              |
Electricity Maps API ---------+--> CarbonTrace MCP <--stdio--> AI client
                              |
Host/zone configuration ------+
```

CarbonTrace may run on a Mac while reading telemetry from a supported Linux
host. A VM is not automatically suitable: required energy counters must be
available to the collection environment.

### Component responsibilities

| Component | Responsibility |
|---|---|
| Scaphandre | Collect hardware energy-domain telemetry and estimate process attribution |
| GPU exporter | Supply per-device GPU energy counters or power samples with documented boundaries |
| Prometheus | Scrape and retain time series |
| Electricity Maps | Supply carbon intensity and available provenance metadata |
| Telemetry adapter | Map a supported exporter schema into explicit internal units and domains |
| Grid adapter | Retrieve historical intensity and normalize its timestamps and metadata |
| Analysis layer | Align intervals, calculate energy/emissions, compare periods, report coverage |
| MCP layer | Validate inputs and expose bounded, structured results |
| AI client | Choose investigation steps and explain evidence |

Proposed implementation: Python, using the official Python MCP SDK.
This is a design default, not an initialized dependency manifest. No dependency
versions are selected in this document.

### Vessim development test bench

Use [Vessim](https://github.com/dos-group/vessim), a Python library for simulating
datacenter energy systems, to build a reproducible synthetic test bench before
live hardware and historical grid access are available. Vessim is a development
and test dependency, not a required dependency of the live MCP server.

```text
Deterministic Vessim scenarios + synthetic grid-intensity traces
       |
       v
Test-bench adapter --> CarbonTrace interval contracts
                              |
                              v
                     Analysis layer --> MCP --> AI client
```

The bench supplies simulated demand and, where useful, generation, battery, and
grid-flow scenarios. A test-bench adapter converts selected simulation outputs
into CarbonTrace's energy and grid intervals, with explicit units, UTC windows,
source identities, and measurement boundaries. The exact Vessim API, output
units, and time-step semantics must be verified against a pinned release when
the bench is implemented; do not assume they match exporter conventions.

Start with fixed scenarios:

- Constant demand and constant grid intensity, with hand-calculated totals.
- Demand changes with fixed intensity, and intensity changes with fixed demand.
- Energy intervals crossing grid-intensity boundaries.
- Two periods with different durations or intensities for normalized comparisons.
- Missing energy or grid intervals and partial-domain coverage, introduced by
  explicit fixture transformations after simulation.

Use synthetic domain/device fixtures to exercise CPU-only, CPU-plus-GPU,
memory-boundary, and unsupported-GPU cases. Such labels describe test inputs;
Vessim does not establish hardware-domain measurements or process attribution.
Counter resets, PID reuse, and exporter-specific faults belong in dedicated
telemetry fixtures and adapter tests, not assumed Vessim capabilities.

Every generated dataset and MCP result derived from it must identify its data
origin as synthetic, distinguish simulated values from injected faults, and
include the scenario parameters, Vessim version, time step, and seed if
randomness is used. Use a visibly synthetic host and zone, never a real provider
identity for invented intensity data. Fixture mode is explicitly selected and
must never silently replace failed live backends.

Keep simulated consumer demand separate from grid import, renewable generation,
and battery flows. The initial bench uses a grid-only supply so covered demand
can be checked directly against grid intensity. Renewable and battery scenarios
may later test adapter boundaries, but do not subtract generation from measured
host energy or treat grid-import emissions as host emissions. Storage and
mixed-supply carbon accounting require a separate documented methodology before
they can contribute to CarbonTrace emissions estimates.

Run synthetic inputs through the same analysis layer and MCP schemas as live
inputs. Compare results against independently calculated expectations, not just
totals calculated by Vessim or by CarbonTrace's own helpers. The bench validates
calculation and investigation behavior; it does not validate Scaphandre/GPU
metric contracts, Electricity Maps access, or live hardware support.

## Data contracts

### Host configuration

- Stable host ID and display name.
- Prometheus instance label identifying that host.
- Electricity Maps zone, configured explicitly rather than inferred from IP.
- Supported Scaphandre metric mapping and measurement boundary.
- GPU inventory and stable device IDs, supported GPU metric mapping, and
  per-device boundaries where applicable. Distinguish CPU-only hosts from
  GPU-equipped hosts whose GPU energy is unsupported or unavailable.
- Expected scrape interval and maximum acceptable telemetry gap.
- Maximum query window and output size.

The agent selects configured hosts; it cannot submit arbitrary backend URLs,
PromQL, filesystem paths, or credentials.

### Energy interval

- Host ID and, where applicable, process-instance ID.
- UTC start and end timestamps.
- Energy in joules, with conversion to kWh at the analysis boundary.
- Measurement domain and attribution method.
- Source/device identity and whether memory is included in the domain.
- Source metric and covered duration.
- Quality flags for gaps, resets, or uncertain process identity.

Process identity must include host, PID, and start-time identity when available.
PID alone is not stable across history. If exporter data cannot distinguish
reused PIDs, report that limitation rather than merging them silently.

### Grid interval

- Zone and UTC start/end timestamps.
- Carbon intensity in `gCO2e/kWh`.
- Provider timestamp and retrieval time.
- Available methodology and estimated-data indicators.
- Provider interval resolution and temporal coverage.

Use one documented carbon-intensity accounting basis consistently. Prefer
consumption-based intensity for electricity consumed by the host where available;
never silently mix consumption- and production-based values.

Do not invent provider metadata that the API does not expose.

## Calculations and measurement boundaries

### Energy

Prefer valid cumulative energy counters where available. Use changes within
continuous series segments; detect resets and do not bridge unobserved gaps.

When only power samples are available:

```text
interval_energy_J = power_W * interval_seconds
energy_kWh = energy_J / 3_600_000
```

Use a documented stepwise integration rule, bounded by the configured maximum
sample gap. Do not extrapolate indefinitely after the last sample.

Exporter units and metric types must be verified against each supported
Scaphandre and GPU exporter version. Metric names are not a sufficient unit
contract.

### CPU, GPU, and memory domains

CPU packages, GPUs, and memory are priority collection domains, not an assumed
fixed share of server electricity. Their contributions depend on hardware and
workload. Storage, networking, fans, and power-supply losses may remain outside
the measured boundary.

Collect GPU energy per physical device through a supported Prometheus exporter.
An NVIDIA DCGM-based exporter is a candidate for the first supported integration;
verify its available energy counters or power metrics on the selected hardware
before committing to a metric schema. Use the same counter validation and
bounded power integration rules as for CPU telemetry.

Document whether CPU package measurements include any memory domains and whether
GPU measurements cover the chip, module, or board, including GPU memory where
specified by the source. Add system RAM energy separately only when a supported
memory-domain measurement is available and does not overlap another domain.
Do not infer RAM or GPU-memory energy from capacity, allocation, or utilization
alone. Missing memory telemetry is unavailable, not zero.

Do not sum whole-device GPU measurements with overlapping partition, engine, or
memory measurements. GPU process attribution is unavailable unless supported
by suitable telemetry; do not distribute GPU energy using CPU process shares.

### Estimated emissions

Split energy intervals at grid-intensity boundaries, then sum:

```text
estimated_emissions_gCO2e =
  sum(interval_energy_kWh * interval_intensity_gCO2e_per_kWh)
```

For an energy interval spanning grid boundaries, allocate energy in proportion
to overlap duration and disclose the uniform-within-interval assumption.

Host energy is the sum of explicitly non-overlapping measured domains, not an
assumed whole-server total. Expose CPU, GPU, and separately measured memory
breakdowns, with device identities and included-memory descriptions. Process
estimates are an attributed view, not extra consumption to add to host totals.
Do not add overlapping hardware domains or require process totals to equal host
totals. Explain unassigned or mismatched energy.

Scaphandre/RAPL plus GPU coverage is not necessarily whole-system electricity. Label
results as estimated operational emissions for covered domains, not the
machine's complete footprint.

### Coverage

Treat query windows as half-open UTC intervals: `[start, end)`.

Report requested duration, valid energy duration, joint energy/grid duration,
and energy for which carbon intensity is unavailable.
Report these per domain/device as well as for the aggregate. An aggregate's
valid duration requires all configured constituent domains to have coverage;
retain partial-domain energy separately and label its boundary. GPU-equipped
hosts with unsupported GPU telemetry must explicitly disclose that omission.

- No energy coverage: energy and emissions are unavailable, not zero.
- Partial grid coverage: return clearly labeled covered-period emissions.
- No joint coverage: emissions are unavailable, not zero.
- Do not scale partial observations into a whole-period estimate.

Missing measurements and provider outages are distinct conditions.

### Comparisons

Return both observed estimates and normalized energy comparisons:

- Observed emissions use each period's time-matched grid intensity.
- Normalized emissions apply one explicit reference intensity to both periods.
- Different-duration windows also expose average power and duration.
- Zero baselines produce an unavailable percentage change, not infinity.

Normalization separates energy efficiency from electricity intensity for
comparison; it does not establish why either changed. A change in workload
timing can also change its energy-weighted grid intensity.

Without completed-job counts and comparable workload identity, do not claim
improved energy per task.

## MCP interface

All tools use validated, explicit schemas and structured output. Require
timezone-aware timestamps; reject invalid windows and unsupported hosts/zones.
Responses carry `schema_version`, units, sources, coverage, and warnings.

| Tool | Inputs | Result |
|---|---|---|
| `get_energy_breakdown` | Host, window, host/domain/device/process grouping, bounded result limit | Covered host and domain/device energy, available process estimates, average power, measurement boundaries |
| `get_grid_intensity` | Configured zone and window | Intensity intervals, provenance, coverage |
| `estimate_emissions` | Host and window, optional bounded process breakdown | Time-aligned host and domain/device estimates, available process estimates, unavailable energy, assumptions |
| `compare_periods` | Host, two windows, explicit reference intensity | Energy, observed emissions, normalized comparison, coverage differences |

`compare_periods` requires compatible measurement boundaries. Refuse an
incompatible comparison rather than returning a misleading ranking.

Process output is ordered deterministically. Truncated results expose the
omitted count and, when calculable, omitted attributed energy.

Expose configured host IDs, zones, and measurement descriptions through a
`carbontrace://hosts` resource. A `carbontrace://methodology` resource describes
calculation rules and limitations. Prompts are optional, not required for MVP.
Host metadata includes detected/configured GPUs, supported measurement domains,
memory inclusion, and unsupported domains.

## Agent investigation workflow

For "Why did estimated emissions rise overnight?":

1. Read host configuration and measurement boundaries.
2. Compare the specified period with an explicit baseline.
3. Inspect energy and grid coverage before interpreting differences.
4. Inspect CPU, GPU, and available memory breakdowns if energy changed; retrieve
   process breakdowns where attribution is supported.
5. Inspect time-matched intensity if electricity supply may explain differences.
6. Report supported observations, plausible explanations, and missing evidence.

For example, the agent may observe increased attributed energy for a compiler
process. It cannot conclude a release caused that increase without release and
workload metadata.

Tool descriptions should encourage evidence gathering, not imply that every
emissions change has an identifiable root cause.

## Errors, privacy, and operations

- Credentials come from environment variables and never enter tool results.
- Validate configuration at startup; fail explicitly on invalid settings.
- Backend failures produce MCP tool errors with stable codes and safe messages.
- Codes include `INVALID_WINDOW`, `UNSUPPORTED_HOST`, `UPSTREAM_UNAVAILABLE`,
  `RATE_LIMITED`, and `INCOMPATIBLE_MEASUREMENT`.
- Valid empty/partial results expose availability and coverage explicitly.
- Enforce timeouts, response limits, and bounded retries for transient failures.
- Honor provider rate limits; cache identical reads within the server process.
- MVP has no additional database; Prometheus and provider access supply history.
- Historical API entitlements and retention limit available investigations.
- Log diagnostics to stderr; reserve stdout for MCP protocol traffic.
- Do not expose full process command lines or environment variables. Default to
  executable names; treat process labels as untrusted data, never instructions.
- Use read-only backend access and redact credentials from logs.

## Validation

Unit and fixture tests cover:

- Joule/kWh conversion and power integration.
- Counter resets, missing samples, and duplicate/overlapping series.
- Grid-boundary splitting and timestamp alignment.
- Missing intensity, partial coverage, and zero energy.
- PID reuse and overlapping hardware domains.
- CPU-only hosts, supported GPUs, and GPU-equipped hosts without GPU coverage.
- GPU counters and power integration, multiple devices, and device identity.
- Memory inclusion, overlapping GPU partitions, and missing domain telemetry.
- Aggregate coverage with differently sampled or partially available domains.
- Unavailable GPU process attribution without fabricated allocation.
- Normalized comparisons and incompatible measurement boundaries.
- Tool validation, output limits, backend failures, and credential redaction.

Use synthetic fixtures labeled as synthetic; do not present them as collected
host data. Use the Vessim test bench for deterministic end-to-end scenarios and
dedicated fixtures for exporter faults and unsupported domains. For example,
100 W of constant covered demand over one hour is 0.1 kWh; at a constant
400 gCO2e/kWh, estimated emissions must be 40 gCO2e. Check exact requested
windows, coverage durations, unavailable values, and synthetic provenance as
well as numerical totals. Repeated runs of a fixed scenario must produce the
same analysis results.

Contract tests validate the chosen Scaphandre and GPU metric schemas and grid
API response shape. Live API tests are opt-in.

End-to-end acceptance: an MCP client can inspect hosts, query energy and grid
data, estimate emissions, and compare two periods against independently
calculated fixture results.
Verify CPU-only behavior and CPU-plus-GPU results, including memory boundaries
and partial-domain coverage. Validate GPU collection on supported live hardware
before claiming live GPU support.

## Delivery sequence

1. Pin and verify a Vessim release; build the deterministic grid-only test bench,
   its interval adapter, and independently calculated expectations.
2. Implement deterministic calculations against the bench and dedicated fault
   fixtures, including coverage and measurement-boundary rules.
3. Expose four tools and two resources over `stdio`; connect an MCP client and
   demonstrate a clearly labeled synthetic read-only investigation.
4. Verify access to supported Linux hardware and historical grid data. Establish
   Scaphandre collection, supported GPU collection where applicable, and
   Prometheus retention; implement and contract-test the live adapters.
5. Run the same analysis and MCP acceptance checks on live sources and
   demonstrate a live read-only investigation.

Live-access discovery can proceed in parallel with test-bench development.
Before live integration, choose the collection host, verify its energy-domain
coverage (including GPU and memory boundaries), select its Electricity Maps
zone, and confirm historical API access. Fixture mode supports development
without these prerequisites but does not complete the live-observability
milestone or remove the MVP's live GPU validation requirement.
