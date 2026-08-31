# Architecture Addendum — Investigation Domain, Differentiation, Resolved Questions

Status: incorporates into DOMAIN_ARCHITECTURE.md, does not replace it.
Read this alongside that document — it adds one domain, one formal
principle, and closes three previously-open questions.


## 1. New: the Investigation Domain

The addendum's core product shift — from "dataset → visualization" to
"phenomenon/question → relevant datasets → context → analysis →
provenance → result" — is a real domain-model gap, not just new UI copy.
The existing six domains (Scientific Data, Dataset Catalog, Observation,
Analysis, Visualization, Interaction) can compute any individual result,
but nothing owns representing *a scientific question itself* as a
persistent, nameable, reproducible object. That's a seventh domain:

### 1.1 Investigation Domain

**Owns:** representing a scientific question or phenomenon (e.g. "how did
the mixed layer respond to this cyclone") as a first-class object that
references — not duplicates — the datasets, variables, regions, depths,
and time-ranges relevant to answering it, and composes one or more
Analysis Domain operations into a coherent, storable result.

**Does not own:** the scientific computations themselves (delegates to
Analysis Domain), rendering (delegates to Visualization Domain), or
dataset semantics (delegates to Scientific Data Domain). An Investigation
is a composition layer, not a computation layer — this keeps it thin and
stops it becoming a dumping ground for logic that belongs elsewhere.

**Invariant it protects:** an Investigation is fully reproducible from its
recorded definition. Re-running the same Investigation later — potentially
against updated model output or newly arrived observations — must produce
a result whose provenance makes clear what changed and why, never a
silently different answer to what looks like the same question.

**Relationship to the resolved workflow language:** this is what makes
*Explore → Probe → Compare → Analyze → Understand* a real capability
rather than a navigation breadcrumb. "Probe" and "Compare" produce
Analysis results; "Understand" is the point where those results, plus
their full multidimensional context, get bound together into an
Investigation worth keeping. This also directly satisfies the Analysis
Workspace requirement ("saved investigations... reproducible analysis
context") that the domain document had not yet addressed.

**Boundary update:**
```
Investigation  --composes-->  Analysis, Scientific Data, Observation
Investigation  --produces-->  Visualization (a reproducible, visualizable result)
Interaction    --creates/loads-->  Investigation (user-driven, not automatic)
```

**Sequencing note:** per the domain document's own "no premature
abstraction" stance (Section 5 there), the Investigation Domain's
interface is *defined* now because the product requirement is real and
resolved (Section 1 above), but its persistence mechanism (how an
Investigation is actually stored/shared) stays unspecified until the
first vertical slice that needs it — consistent with deferring interface
shape until a real implementation makes it evident.

## 3. New formal principle: the INCOIS differentiation test

Added alongside the existing "if branding were removed" test as a second,
complementary gate — the first asks whether the product *feels* like a
purpose-built instrument; this one asks whether a specific feature is
*worth building* at all:

> Before implementing any feature that has an existing INCOIS (or
> comparable platform) equivalent: identify what exists, how users
> currently interact with it, and what remains fragmented, disconnected,
> manual, or non-multidimensional about it. Build the fix for that gap —
> not a reproduction of the existing capability.

Practical effect on the roadmap: this doesn't change Section 7's
sequencing, but it changes what "done" means for each slice. The first
vertical slice (field → location → depth → profile) isn't done when it
matches what a Digital-Ocean-style viewer already does — it's done when
the same interaction also carries the multidimensional context (which
dataset, what quality, what's the observation coverage nearby) that those
existing tools leave the user to assemble manually.

## 4. Updated canonical workflow language

Supersedes the informal workflow lists in the earlier documents:

**Explore → Probe → Compare → Analyze → Understand**

- *Explore* — Interaction + Visualization domains, spatial/temporal/
  variable navigation
- *Probe* — Analysis Domain, point/profile extraction at a location+depth+time
- *Compare* — Analysis Domain, model-observation matching and difference
- *Analyze* — Analysis + Investigation domains, composing multiple probes/
  comparisons into a coherent question
- *Understand* — Investigation Domain, the reproducible, provenance-carrying
  result

No other document needs updating for this — it's a naming convergence,
not a structural change.