# ADR-001, ADR-002, ADR-003 — Foundational Technology Decisions

Status: accepted
These sit beneath DOMAIN_ARCHITECTURE.md's boundaries (Section 3) — the
domains and invariants don't change if any of these decisions later do.

---

## ADR-001: Rendering engine

**Context.** The Visualization Domain needs a concrete initial
implementation behind the Rendering Adapter boundary (domain doc, Section
3.1): volumetric/depth-resolved scalar fields, vector/current fields,
observation markers and tracks, all in-browser, at interactive frame
rates, with GPU-accelerated particle/vector rendering explicitly in scope
per the product brief.

**Problem.** Which engine implements the first Rendering Adapter.

**Options considered.**
- **CesiumJS** — whole-Earth geospatial engine; strong built-in terrain,
  imagery basemaps, ellipsoid geodesy. Weak fit for fully custom
  volumetric/depth-slice shader work without fighting the engine's own
  scene model; materially larger bundle.
- **Three.js** — general-purpose WebGL engine; full control over custom
  scalar/vector field rendering and depth-layer compositing; no built-in
  geospatial conveniences (lat/lon projection, coastlines) — those have
  to be hand-built.
- **deck.gl** — excellent for large-scale geospatial point/line/polygon
  layers; not designed for true volumetric scalar-field rendering, which
  is the core requirement here.
- **WebGPU-native custom** — best long-term ceiling for compute-shader-
  driven particle systems and GPU interpolation; browser support and
  tooling maturity in 2026 still carry real adoption risk as a *starting*
  choice, though the trajectory is the right one to build toward.

**Evaluation.** The hard, differentiating problem is custom scientific
field rendering (depth-resolved scalar fields, current vectors), not
whole-Earth basemap navigation — Cesium is strong exactly where this
product doesn't need strength, and weak exactly where it does. deck.gl
solves an adjacent but different problem (geospatial layers, not
volumetric fields). WebGPU-native is the right eventual direction — the
product brief is correct to flag it — but committing to it as the
*initial* implementation trades a real, bounded cost (building geospatial
basics ourselves in Three.js) for an unbounded one (building on
still-maturing tooling for the first working version of anything).

**Decision.** Three.js as the initial Rendering Adapter implementation.
GPU-accelerated techniques (instancing, texture-driven interpolation) are
used within it for particle/current rendering rather than reaching for
WebGPU-native immediately.

**Consequences.** We own lat/lon-to-scene projection, coastline/land-mask
rendering, and camera conventions ourselves. Acceptable: that work is
bounded and well-understood; it is not the differentiated part of the
product.

**Replacement boundary.** The Rendering Adapter interface (domain doc,
Section 3.1). No domain above it references Three.js types.

**Revisit conditions.**
- Measured (not assumed) evidence that current-particle density/
  performance requirements exceed what WebGL2/Three.js instancing can
  sustain.
- A real, evidenced requirement for whole-Earth multi-basin navigation
  with terrain/imagery basemaps emerges.
- Three.js's own upstream WebGPU backend matures to production quality —
  at that point the benefit may be captured by a backend swap *inside*
  the existing adapter, requiring no adapter-level change at all.

---

## ADR-002: Frontend framework

**Context.** Resolved product scope (addendum, Section 1) confirms a
genuine multi-workspace product — Explorer, Dataset Explorer, Observation
Explorer, Comparison Workspace, Analysis Workspace — plus a landing page
carrying real identity/discovery weight (differentiation from INCOIS is a
stated product goal, which means the landing page needs to actually
explain that, well, to people who aren't already convinced).

**Problem.** Initial frontend framework and routing foundation.

**Options considered.**
- **Next.js (App Router)** — SSR/SEO for the landing and discovery pages;
  file-based routing matches the multi-workspace structure directly;
  doesn't force SSR on routes that don't want it (Explorer can still be
  effectively client-rendered).
- **Vite + React SPA** — lighter tooling, faster dev iteration, no SSR.
  Weaker for a landing page meant to establish identity and be
  discoverable/shareable as a link.
- **Remix** — comparable SSR benefits to Next.js; smaller ecosystem,
  fewer scientific-visualization integration examples to draw on.

**Evaluation.** If the landing/discovery surface were an afterthought,
Vite's simplicity would win — that was the right call for VARUNA, a
hackathon prototype whose only real page was the Explorer. It is not the
right call here: the resolved scope makes the landing page load-bearing
for the product's actual differentiation argument, which tips this
specifically toward SSR.

**Decision.** Next.js App Router.

**Consequences.** Slightly heavier initial build/tooling complexity than
a plain SPA — justified by the resolved scope, not assumed by default.

**Replacement boundary.** Only the Interaction/Application domain's route
structure references Next.js. The Scientific Data, Observation, Analysis,
Visualization, and Investigation domains reference nothing framework-
specific.

**Revisit conditions.** If product scope narrows back to "Explorer only"
(would contradict the current, resolved scope decision) — the SSR
justification disappears and a lighter SPA becomes correct again.

---

## ADR-003: Backend framework and scientific-computing runtime

**Context.** Serving scoped scientific queries (dataset + variable +
region + depth + time) over Argo, public ocean-model output, bathymetry,
and satellite-derived data (per resolved data-availability decision,
addendum Section 1), without INCOIS-specific schema assumptions.

**Problem.** Backend framework and language for the Scientific Data
serving layer.

**Options considered.**
- **FastAPI + Python** — direct access to xarray/NumPy/netCDF4/Dask, the
  ecosystem where correct CF-conventions handling, curvilinear-grid
  support, and calendar/unit correctness already exist as mature,
  open-source, battle-tested code.
- **Node/TypeScript backend** — unifies language with the frontend, but
  has a materially weaker NetCDF/scientific-array ecosystem; would end up
  shelling out to Python for the actual data work anyway, adding an
  integration boundary that buys nothing.
- **Go** — strong performance/concurrency; essentially no
  scientific-NetCDF ecosystem; same problem as Node, worse ecosystem gap.

**Evaluation.** This is one of the few decisions in this architecture
that isn't genuinely close. Reimplementing correct NetCDF/CF-conventions/
calendar/unit handling in another language is a multi-year undertaking
with zero product benefit — the domain expertise already lives in
xarray's ecosystem, open-source and directly reusable. The "different
language than the frontend" cost is real but small and standard for this
domain (most serious scientific-data platforms are polyglot for exactly
this reason).

**Decision.** FastAPI + Python, xarray as the core scientific-data
library.

**Consequences.** Two-language stack. Isolated behind the API contract —
never leaks into frontend code, per the domain document's boundary rules.

**Replacement boundary.** The API contract itself (scoped-query HTTP/JSON
endpoints). Nothing in the frontend or in the Analysis/Visualization
domain's conceptual model depends on Python specifically.

**Revisit conditions.** None foreseeable for the scientific-computing
layer itself. The transport mechanism (REST vs. streaming/binary) remains
open per the domain document's deferred items, to be revisited if large
field payloads become a measured bottleneck — not assumed now.