# Ocean Intelligence Platform — Domain Architecture & Invariants

Status: foundational layer, precedes technology mapping
Supersedes the framing (not the content) of the earlier ARCHITECTURE.md —
see "Relationship to the previous document" at the end.

## 1. Product definition

A scientific instrument for investigating ocean conditions across space,
depth, time, variable, and data source — where a model field, an
observation, and their comparison are treated as equally first-class, and
every displayed value remains traceable to its origin.

It is not a globe with ocean colors on it. The test: if you removed all
branding, would this still behave like a purpose-built instrument for
answering "what does the ocean look like here, at this depth, at this
time, and does the model agree with what was actually measured" — or
would it behave like a generic geospatial dashboard that happens to show
ocean data? Every major decision gets checked against that question.

## 2. Domains

Six domains, each with one responsibility. None of these are framework
names — they're the stable conceptual units the rest of the system is
organized around.

### 2.1 Scientific Data Domain
Owns: understanding what a dataset *means* — dimensions, coordinate
systems, units, calendars, vertical coordinate semantics, missing/fill
values, quality flags. Does not own storage format or query execution.
**Invariant it protects:** a value's unit and provenance can never be
separated from the value itself as it moves through the system.

### 2.2 Dataset Catalog Domain
Owns: what datasets exist, their coverage (spatial/temporal/vertical),
variables, resolution, provider, and how to address a specific dataset.
**Invariant it protects:** a new dataset is addressable by the rest of
the system without any other domain being modified.

### 2.3 Observation Domain
Owns: in-situ measurements (Argo, and future platforms) as a semantically
distinct category from model output — locations, tracks, profiles,
timestamps, quality flags, platform metadata.
**Invariant it protects:** an observation is never silently merged into
"just a data point" indistinguishable from a model value.

### 2.4 Analysis Domain
Owns: operations that take scientific data and produce scientific
answers — profile extraction, transects, time series, model-observation
matching, difference/anomaly calculation. Each operation has defined
inputs, outputs, and documented assumptions (especially interpolation).
**Invariant it protects:** an analytical result is independently testable
without a UI, and never silently discards quality information to produce
a cleaner-looking answer.

### 2.5 Visualization Domain
Owns: turning scientific data (scalar fields, vector fields, profiles,
tracks) into a renderer-agnostic visual representation — the "what should
be shown" layer. Does not own "how pixels get drawn."
**Invariant it protects:** visualization is always *derived from*
scientific data, never a second source of truth for it.

### 2.6 Interaction/Application Domain
Owns: navigation, selection state, panel/UI state, the workflow of
moving between field → location → depth → profile → comparison. Knows
about the other five domains' existence, not their internals.

## 3. Boundaries and contracts

Each domain exposes a contract to its neighbors; neighbors depend on the
contract, never on each other's internals.

```
Dataset Catalog  --describes-->  Scientific Data
Scientific Data  --queried by--> Analysis
Scientific Data  --queried by--> Visualization
Observation      --matched by--> Analysis (against Scientific Data)
Analysis         --produces-->   Visualization (results are visualizable)
Visualization    --renders via--> Rendering Boundary (see 3.1)
Interaction      --reads/drives--> all of the above, owns none of their internals
```

### 3.1 The Rendering Boundary specifically

This is the one boundary the brief singles out (§14) as needing explicit
protection, and it's worth stating precisely why: of everything in this
system, the rendering engine is the technology most likely to be replaced
within the platform's lifetime (WebGL → WebGPU is already underway
industry-wide; a superior engine in five years is a reasonable bet). The
contract:

```
Visualization Data Model  (scalar fields, vector fields, tracks, profiles
                            — plain data, no engine-specific types)
         ↓
Rendering Controller       (owns camera/depth/time-driven state changes,
                            engine-agnostic)
         ↓
Rendering Adapter          (the actual boundary — one implementation per
                            engine, satisfies the same interface)
         ↓
Rendering Implementation   (Three.js today; Cesium, WebGPU-native, or
                            something that doesn't exist yet, later)
```

Nothing above the Rendering Adapter line may import an engine-specific
type. This is the concrete mechanism behind "the architecture must not
fundamentally depend on a specific rendering engine" (§4) — not a
principle stated in prose and hoped for, but a testable rule: grep the
Visualization Domain and Interaction Domain for `three`/`cesium` imports;
there should be none outside the adapter implementations themselves.

### 3.2 The Dataset Adapter boundary

Symmetric reasoning for data sources: a `DatasetAdapter` contract that
`ModelDatasetAdapter`, `ArgoDatasetAdapter`, and future adapters satisfy,
so the Scientific Data and Analysis domains never branch on "if NetCDF
model vs. if Argo profile" — they operate on the adapter's normalized
output.

## 4. Architectural invariants (complete working set)

Beyond the examples the brief listed, the full set this architecture
commits to protecting:

1. A scientific value's unit is never separated from the value.
2. A scientific value's provenance (source, dataset, version, processing
   history) is never separated from the value.
3. Model output and observational data remain distinguishable at every
   layer — never merged into a generic "data point."
4. Depth/vertical-coordinate semantics are explicit and dataset-defined,
   never hardcoded as "depth is always meters below surface" (some
   datasets are pressure-coordinate).
5. Temporal semantics (calendar, timezone, resolution) are explicit,
   never assumed.
6. Missing/fill values are never silently treated as zero or valid data.
7. Quality flags are never silently discarded during subsetting,
   aggregation, or visualization.
8. Interpolation or resampling is never applied without the method and
   assumptions being recorded alongside the result.
9. A new dataset can be integrated by adding a `DatasetAdapter`
   implementation, without modifying the Analysis or Visualization domains.
10. A new rendering engine can be integrated by adding a `RenderingAdapter`
    implementation, without modifying the Visualization or Interaction
    domains.
11. Analysis operations are callable and testable independent of any UI.
12. Visualization state is derived from scientific data state, never an
    independent source of truth that can drift from it.
13. The browser never receives a full scientific dataset when a scoped
    query (variable + region + depth + time) would do.

Each of these should eventually map to an actual test (§33) — an
invariant that isn't tested is a hope, not an invariant.

## 5. What this document deliberately does not do

Per §42 (no premature abstractions) and the vertical-slice principle
(§45): this document defines boundaries, not every interface. It does not
yet specify the `RenderingAdapter` TypeScript interface, the
`DatasetAdapter` Python protocol, or the API route list — those are
implementation mapping (Section 6) and belong closer to the code they
govern, written when the first real adapter is built, not speculatively
now. Writing five adapter interfaces before a second dataset exists is
exactly the "abstraction for abstraction's sake" this brief also warns
against (§42) — the two instructions only look contradictory; the
resolution is: define the *boundary* now, defer the *interface shape*
until a second real implementation makes the right shape evident.

## 6. Relationship to the previous ARCHITECTURE.md

Reframing, not discarding: everything in the previous document (Next.js,
Three.js-over-Cesium reasoning, FastAPI, the concrete TypeScript
`FieldLayer`/`Observation` shapes) is still good, real engineering work —
it's just mislabeled. Under this framing it becomes:

- The Next.js/Three.js/FastAPI choices → **ADR-001, ADR-002, ADR-003**
  (Section 7), each with the "what would make us reconsider" clause this
  brief requires and the previous document only gestured at informally.
- The `FieldLayer`/`Observation`/`DatasetSummary` TypeScript interfaces →
  the current **implementation** of the Scientific Data Domain's and
  Observation Domain's contracts, not the domain model itself. The domain
  model is "a dataset has coverage, variables, and provenance"; the
  TypeScript interface is one legitimate way to implement that today.
- The four-layer system diagram → still accurate, now sits under Section
  3's boundary description as the current technology mapping of it.

Nothing needs to be rewritten. It needs a home under this document rather
than standing as "the architecture" on its own.

## 7. Proposed sequencing for the remaining deliverables

The brief's §52 lists 45 items. Producing all of them at reasonable depth
in one pass would be worse than not producing them — either shallow
enough to be theater, or long enough that nobody reads them before code
gets written anyway. Proposed order, each one a short, real document
written when it's about to matter rather than speculatively:

**Next (blocks starting any implementation):**
1. ADR-001/002/003 — rendering engine, frontend framework, backend
   framework, each with explicit revisit conditions (Section 6 above)
2. Scientific Data Model — the actual entity shapes (this exists already
   in the previous document; needs relabeling per Section 6, not rewriting)
3. Data pipeline (§10) — concretely, for the *first* real dataset only;
   don't design for datasets that don't exist yet

**Second (blocks the first vertical slice — field → location → depth →
profile, per §45):**
4. Rendering boundary interface (Section 3.1, now made concrete)
5. Dataset adapter interface (Section 3.2, now made concrete) for exactly
   one model source + one observation source (Argo) — not four
   hypothetical future ones
6. API architecture (§19) — scoped to the operations the first slice
   actually needs

**Deferred until there's a second real dataset or a second real rendering
requirement to design against** (per §42's own logic): GPU architecture,
advanced visualization, multi-dataset caching strategy, the intelligence
layer (§8.17). Designing these now would be designing against imagined
requirements, which this brief itself warns against.

## 8. Open questions (carried over, still blocking)

The three questions from the previous document are still unanswered and
still block sequencing:

1. Does this replace VARUNA or run alongside it?
2. Is INCOIS data access real/near-term, or is Argo + one public model
   dataset the realistic scope for a while?
3. Is the multi-route, multi-workspace product (Explorer + Dataset
   Explorer + Observation Explorer + Comparison Workspace + Analysis
   Workspace) near-term real, or the long-term vision with the first
   milestone being just Explorer?

One more, specific to this document: **do you want each of the 45 items
as a separate file under `docs/architecture/` as the brief's §36
structure suggests, or is a smaller number of consolidated documents
(like this one) actually more useful to you?** I'd default to
consolidated — five focused documents get read; forty-five files mostly
don't — but this is a genuine preference question, not one I should
assume my way through.