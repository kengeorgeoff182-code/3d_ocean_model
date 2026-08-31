# Ocean Explorer — Architecture & Data Model

Status: design proposal, no implementation yet
Repo: kengeorgeoff182-code/3d_ocean_model (currently empty — one README line, one commit)

## 0. Scope and assumptions

This repo is a blank slate. Nothing here conflicts with or duplicates existing
code, so this document makes foundational choices rather than adapting to
prior art.

**Working assumption:** this is a distinct, longer-horizon product effort —
not a replacement for the VARUNA SIH hackathon prototype built separately
(Vite/React/Three.js frontend, FastAPI backend, synthetic + NOAA ERSSTv5
data). That prototype validated two things worth carrying forward
deliberately rather than re-discovering: (1) a stacked-depth-layer approach
to volumetric rendering in Three.js works and reads well, and (2) a thin
`DataProvider`-style abstraction (synthetic data and real NOAA data behind
the same interface) is the right shape for swapping in real INCOIS/Argo
data later without touching the UI. Both ideas resurface below, generalized.

If this is actually meant to supersede or absorb VARUNA, say so — it
changes whether step one is "start fresh" or "port and restructure."

## 1. Technology decisions and rationale

The brief lists Next.js, TypeScript, Cesium/Three.js, Plotly, FastAPI, and
xarray as the *expected* eventual stack, with an explicit instruction not
to adopt any of it blindly. Here's the actual reasoning per decision:

| Layer | Choice | Why |
|---|---|---|
| Frontend framework | **Next.js (App Router) + TypeScript** | The product surface is genuinely multi-page (landing, /explore, /datasets, /datasets/[id], /observations, /compare, /about) with a marketing-weight landing page that benefits from SSR/SEO. That's a real reason to prefer Next.js over a plain Vite SPA — this isn't true for every project, but it's true for this one. |
| 3D engine | **Three.js**, not Cesium, for now | Cesium is a whole-Earth globe engine — ellipsoid terrain, imagery layers, geodesy. Our domain is a bounded regional ocean volume (India's EEZ) needing custom volumetric/depth-slice rendering, which is closer to a Three.js problem than a globe problem, and is what the VARUNA prototype already validated. Cesium adds real bundle weight (~2MB+) and a different mental model. **Recommendation, not a ban:** keep Cesium behind the Rendering Adapter interface (Section 3) as a future option if true multi-ocean/global-scale navigation becomes a real requirement — don't pay its cost speculatively. |
| Charting | **Hand-rolled SVG initially; add Plotly only for Compare** | Simple profile charts (depth vs. temperature/salinity) don't need a charting library. The Compare view's time-series and statistical comparisons genuinely benefit from Plotly's zoom/pan/multi-series handling. Introduce it scoped to that feature, not globally. |
| Backend | **FastAPI + Python + xarray** | Directly reuses the validated pattern from VARUNA's `real_data.py`: open NetCDF via xarray, subset, regrid, return a stable JSON contract. No reason to reinvent this. |
| State management | **React Context + hooks per domain, no global store library** | Five distinct, small state domains (Section 4) don't justify Redux/Zustand overhead. Revisit only if a concrete cross-cutting problem appears. |
| Server-state/caching | **TanStack Query** | The one addition worth its weight — request dedup, caching, and loading/error states for dataset/observation fetches are exactly its job, and hand-rolling this well is a time sink for no benefit. |
| Styling | **Tailwind, mapped to CSS custom properties for design tokens** | Tailwind's utility classes for layout/spacing; the actual color/token values live in `:root` CSS variables (Section 6) so theming isn't scattered across `className` strings. |

Explicitly **not** adopting: a GraphQL layer (REST is sufficient for this
resource shape), a heavyweight global store, or Cesium — until a concrete
requirement forces the question.

## 2. System architecture

The four-layer shape, top to bottom:

- **Ocean Explorer UI** (Next.js/TypeScript) — the viewport, control panel,
  and scientific inspector. This is the primary product surface.
- **Rendering adapter** — an interface the UI codes against, not against
  Three.js directly. Swappable by design (Section 3).
- **Backend API** (FastAPI) — dataset-agnostic REST endpoints (Section 5).
  Doesn't know or care whether data is synthetic, NOAA, or eventually
  INCOIS/Argo.
- **Data providers** — one implementation per data source, each producing
  the same response shape. This is the generalized version of VARUNA's
  `data.py` / `real_data.py` split: instead of two hardcoded modules, a
  registered set of providers behind one interface.

*(See the diagram above this document for the visual layout.)*

### Why the adapter boundary matters here specifically

The brief's principle #8 ("keep the 3D rendering layer replaceable") isn't
just good hygiene in general — it's concretely likely to matter for this
project: the Cesium-vs-Three.js decision above is a *should revisit later*,
not a *closed* decision. If `VisualizationViewport` calls Three.js APIs
directly from React components, that revisit becomes a rewrite. If it calls
a `RenderingAdapter` interface, it becomes a new adapter implementation.

```ts
// rendering/adapter.ts
export interface RenderingAdapter {
  mount(container: HTMLElement): void;
  unmount(): void;
  setDepthStack(layers: FieldLayer[], selectedDepth: number): void;
  setObservations(observations: Observation[]): void;
  onLocationHover(cb: (loc: GeoCoordinate | null) => void): void;
  onLocationSelect(cb: (loc: GeoCoordinate) => void): void;
  setCameraTarget(depth: number): void;
}

// rendering/three-adapter.ts
export class ThreeRenderingAdapter implements RenderingAdapter { /* ... */ }

// A future rendering/cesium-adapter.ts would implement the same interface.
```

`VisualizationViewport` holds a `RenderingAdapter` reference and never
imports `three` or `cesium` directly. This is the one piece of indirection
in this design that's justified by a specific, named, plausible future
change — not indirection for its own sake (see principle #3: prefer simple
architecture over unnecessary abstraction).

## 3. Component structure

```
OceanExplorer                     (page-level composite, /explore route)
  |-- ControlLayer
  |     |-- DatasetSelector
  |     |-- VariableSelector
  |     |-- LayerControl
  |     `-- DepthControl
  |-- VisualizationViewport
  |     `-- (RenderingAdapter -> Three.js, isolated per Section 2)
  |-- ScientificInspector
  |     |-- CoordinateDisplay
  |     |-- DataInspector
  |     `-- ScientificValue  (reused inside DataInspector)
  |-- Timeline
  |     `-- TimeControl
  `-- ScientificLegend
```

Generic UI primitives (buttons, sliders, panels, drawers) live in
`components/ui/` and know nothing about oceanography. Domain components
(`DatasetSelector`, `ScientificValue`, `ObservationMarker`, `ProfileChart`,
etc.) live in `components/ocean/` and compose the primitives. This is the
brief's principle #6 (generic vs. domain-specific separation) — worth
stating explicitly because it's the difference between a `ScientificValue`
component that's reusable across Explorer/Compare/Observations, and three
slightly-different copies.

None of these components contain fabricated scientific logic — every one
takes real data as props and renders it. Where there's no data yet, they
render an explicit empty state, not placeholder numbers (see Section 5 on
why this matters for a science product specifically).

## 4. State model

Five domains, deliberately not merged into one store:

```ts
// --- UI state (local component state / URL search params) ---
interface UIState {
  activePanel: 'control' | 'inspector' | null;  // for drawer collapse on smaller screens
  isDrawerOpen: boolean;
  activeNavRoute: string;
}

// --- Dataset state (React Context, populated via TanStack Query) ---
interface DatasetState {
  selectedDatasetId: string | null;
  availableDatasets: DatasetSummary[];
  selectedVariable: VariableId | null;
}

// --- Visualization state (React Context; this is what the RenderingAdapter reads) ---
interface VisualizationState {
  selectedDepth: number;            // metres
  selectedTime: string;             // ISO date
  activeLayers: LayerId[];          // e.g. ['temperature-field', 'argo-floats']
  cameraPreset: 'default' | 'top-down' | 'profile-view';
}

// --- Interaction state (React Context, high-frequency updates -- kept separate
//     from VisualizationState so hover doesn't re-render the whole viewport tree) ---
interface InteractionState {
  hoveredLocation: GeoCoordinate | null;
  selectedLocation: GeoCoordinate | null;
  selectedObservationId: string | null;
}

// --- Server state (TanStack Query cache, not hand-rolled) ---
// datasets, observations, field data, comparison results all live here,
// keyed by query params (datasetId, variable, depth, time).
```

The separation that actually earns its keep: `InteractionState` updates on
every pointer-move during hover; `VisualizationState` updates only on
deliberate control changes. Merging them means every hover event
re-renders anything subscribed to depth/time/layers — exactly the
unnecessary-re-render problem principle #20 (performance) calls out.

## 5. Data model

### 5.1 Core domain entities (TypeScript, frontend + API contract)

```ts
type VariableId = 'temperature' | 'salinity' | 'current_speed' | 'chlorophyll';

interface DatasetSummary {
  id: string;
  name: string;
  provider: 'synthetic' | 'noaa-ersst' | 'incois-model' | 'argo';
  description: string;
  variables: VariableId[];
  depthLevels: number[];            // metres
  timeCoverage: { start: string; end: string; resolution: string };
  spatialBounds: GeoBounds;
  resolution: { gridSize: number; units: string };
  isReal: boolean;                  // real observation/model output vs. synthetic placeholder
  provenance: DatasetProvenance;
}

interface DatasetProvenance {
  source: string;                   // e.g. "NOAA ERSSTv5"
  citation?: string;
  license?: string;
  lastUpdated: string;
}

interface GeoBounds {
  latMin: number; latMax: number;
  lonMin: number; lonMax: number;
}

interface GeoCoordinate {
  lat: number;
  lon: number;
  depth?: number;
}

// A single depth layer's gridded values for one variable/time -- this is
// the direct generalization of VARUNA's field response shape.
interface FieldLayer {
  variable: VariableId;
  depth: number;
  time: string;
  bounds: GeoBounds;
  gridSize: number;
  values: number[][];
  min: number;
  max: number;
  isReal: boolean;
  source: string;
}

interface FieldStack {
  datasetId: string;
  variable: VariableId;
  time: string;
  depths: number[];
  layers: FieldLayer[];
}

// Observations (Argo, Glider, CTD, BGC -- unified shape, `kind` discriminates)
interface Observation {
  id: string;
  kind: 'argo' | 'glider' | 'ctd' | 'bgc';
  location: GeoCoordinate;
  lastReportTime: string;
  profile: ObservationProfilePoint[];
}

interface ObservationProfilePoint {
  depth: number;
  temperature?: number;
  salinity?: number;
  chlorophyll?: number;
}

// Model-vs-observation comparison (Compare feature)
interface ComparisonResult {
  datasetId: string;
  observationId: string;
  variable: VariableId;
  depth: number;
  time: string;
  modelValue: number;
  observedValue: number;
  difference: number;               // modelValue - observedValue
  withinTolerance: boolean;
}
```

### 5.2 Backend API surface (FastAPI, REST)

```
GET  /api/datasets                         -> DatasetSummary[]
GET  /api/datasets/{id}                     -> DatasetSummary
GET  /api/datasets/{id}/field_stack         -> FieldStack
       ?variable=temperature&time=2026-05-01
GET  /api/datasets/{id}/field               -> FieldLayer
       ?variable=temperature&depth=0&time=2026-05-01

GET  /api/observations                      -> Observation[]
       ?kind=argo&bounds=...
GET  /api/observations/{id}                 -> Observation

GET  /api/compare                           -> ComparisonResult[]
       ?datasetId=...&variable=temperature&time=...
```

Every dataset-scoped endpoint takes a `datasetId` — this is the key
structural change from VARUNA's backend, which hardcoded a single implicit
dataset. Internally, a small provider registry maps `datasetId` to a
`DataProvider` implementation:

```python
# app/providers/base.py
class DataProvider(Protocol):
    def get_field(self, variable: str, depth: float, time: str) -> FieldLayer: ...
    def get_field_stack(self, variable: str, time: str) -> FieldStack: ...
    def summary(self) -> DatasetSummary: ...

# app/providers/synthetic.py   -- generalizes VARUNA's data.py
# app/providers/noaa_ersst.py  -- generalizes VARUNA's real_data.py
# app/providers/incois.py      -- future, once access exists
```

This is the same shape-compatibility trick that made VARUNA's real-data
toggle a one-file addition instead of a rewrite — formalized as an actual
interface instead of two modules that happened to agree on a dict shape.

### 5.3 Why `isReal` / provenance are first-class, not incidental

This is a scientific tool. A user (or judge, or actual forecaster) will
ask "is this real data" — VARUNA's demo script had to plan for that
question explicitly. Baking `isReal` and `provenance` into the data model
itself, rather than as a UI-only afterthought, means every component that
renders a value can honestly label it, and it's structurally impossible to
silently present synthetic data as observed data. This directly serves the
"scientific correctness" and "do not fabricate scientific data" principles
in the brief — as a data-shape decision, not just a UI convention.

## 6. Design tokens

Adopting the palette from the brief as CSS custom properties in
`app/globals.css`, mapped into Tailwind's theme rather than used as raw
hex in components:

```css
:root {
  --background: #05080D;
  --surface: #0A1018;
  --surface-elevated: #101923;
  --border: #1B2A38;
  --text-primary: #E8F0F5;
  --text-secondary: #8FA3B3;
  --text-muted: #5A6B78;      /* added: brief didn't specify a third text tier */
  --accent: #38BDF8;
  --accent-secondary: #2563EB;
  --deep-ocean: #0F3D5E;
  --success: #34D399;          /* added: brief listed the token but not a value */
  --warning: #FBBF24;
  --error: #F87171;
  --focus: #38BDF8;
}
```

**Scientific color scales are a separate module** (`lib/color-scales.ts`),
not the UI accent palette — the brief is explicit about this and it's the
right call: `--accent` is a UI decision that might change with a rebrand;
a temperature colormap is a scientific-communication decision that must
stay consistent regardless. Each variable gets its own scale:

```ts
export const colorScales: Record<VariableId, ColorScale> = {
  temperature: { stops: [...], label: 'degC' },   // warm-to-cool, perceptually uniform
  salinity:    { stops: [...], label: 'PSU' },
  current_speed: { stops: [...], label: 'm/s' },
  chlorophyll: { stops: [...], label: 'mg/m3' },
};
```

Recommend a perceptually-uniform scale (e.g. viridis/cividis-derived) for
each rather than inventing one per variable from scratch — scientific
visualization has existing, well-reasoned conventions here worth reusing
rather than reinventing to match the brand palette.

## 7. Routing (Next.js App Router)

```
app/
  page.tsx                      /            landing
  explore/page.tsx              /explore     Ocean Explorer
  datasets/page.tsx             /datasets
  datasets/[id]/page.tsx        /datasets/[id]
  observations/page.tsx         /observations
  compare/page.tsx              /compare
  about/page.tsx                /about
```

Matches the brief's route list directly — no deviation needed here.

## 8. What this explicitly does not decide yet

Per the brief's own working mode ("work incrementally... do not implement
ten major features simultaneously"), this document stops short of:

- Exact Three.js scene implementation details (the VARUNA prototype's
  stacked-layer approach is the starting reference, not a final spec)
- Auth/user accounts (nothing above implies a user model is needed yet)
- The Compare view's statistical methodology (needs a domain-expert
  decision on tolerance thresholds, not an engineering one)
- Whether `datasetId` routing means one Next.js app instance serves many
  datasets, or each dataset gets dedicated infra — deferred until a second
  real dataset actually exists