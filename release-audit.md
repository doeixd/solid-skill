# Solid Skill Release Audit

## Scope reviewed

- `solid.md` in full
- `solidstart.md`
- `solid-js-1.x-best-practices-and-api/SKILL.md`
- `solid-js-2.x-api-changes-and-best-practices/SKILL.md`
- `solid-js-2.x-migration/SKILL.md`
- `solid-start-v2/SKILL.md`
- all existing `evals/evals.json`
- `solid-start-v2/references/*`

## Current repo shape

- All four skills have a `SKILL.md` and a prompt-only `evals/evals.json`.
- Only `solid-start-v2` has bundled references.
- `solid-js-2.x-api-changes-and-best-practices` is very large and is carrying both workflow guidance and deep reference material in one file.
- There are no assertions yet in any eval set, so there is no quantitative release gate.

## Release blockers

### 1. Missing benchmarkable evals across all skills

Severity: high

All four skills currently have only 3 prompt-only evals. They are useful as smoke tests, but not enough for release.

What is missing:

- no assertions
- no overlap tests between neighboring skills
- no should-trigger versus should-not-trigger cases
- no file-backed evals for realistic edit or review work
- no release viewer workflow artifacts yet

Release action:

- expand each eval set to include objective checks where possible
- add overlap prompts that distinguish:
  - `solid-js-2.x-api-changes-and-best-practices` vs `solid-js-2.x-migration`
  - `solid-start-v2` vs both Solid core skills
- run the full human-review plus benchmark loop before calling any skill release-ready

### 2. Skill-boundary overlap is still too risky

Severity: high

The current descriptions are decent, but the release risk is wrong-skill triggering, especially for the 2.x pair.

Highest-risk overlaps:

- `solid-js-2.x-api-changes-and-best-practices` may over-trigger on migration requests because it mentions changed semantics, warnings, and new APIs.
- `solid-js-2.x-migration` may over-trigger on narrow API explanation requests because it mentions removed APIs and upgrade breakage.
- `solid-start-v2` correctly says to use a matching Solid skill for core behavior, but it does not yet strongly warn that Start v2 is not the same thing as Solid 2.x.

Release action:

- sharpen descriptions after content stabilizes
- add boundary language inside the body, not just frontmatter
- create dedicated trigger evals before running description optimization

### 3. `solid-js-2.x-api-changes-and-best-practices/SKILL.md` is too monolithic

Severity: high

The file is about 640 lines and mixes three jobs:

- skill instructions
- API reference
- migration/reference examples

That makes it harder to maintain and increases the chance of noisy or overly broad outputs.

Why this matters for release:

- the skill-writing guidance recommends keeping `SKILL.md` lean and moving deeper material into references
- this file is the highest-churn source because Solid 2.x is still beta or `next`
- a monolithic file makes future source-alignment reviews slower and riskier

Release action:

- keep the top-level workflow and primitive-selection guidance in `SKILL.md`
- move deeper material into references such as:
  - `references/reactivity-and-effects.md`
  - `references/async-and-boundaries.md`
  - `references/stores-and-projections.md`
  - `references/dom-and-control-flow.md`
  - `references/migration-hotspots.md`

### 4. Start v2 guidance needs a stronger boundary against Solid 2 assumptions

Severity: high

`solidstart.md` is explicit that Start v2 is not lockstep with Solid v2, and that Start v2 does not drop Solid v1 support. The current Start skill is directionally correct, but this needs to be made much more explicit in release form.

Risk:

- the skill may encourage the model to bring in Solid 2 terms or primitives when the local Start app still targets Solid 1.x-compatible behavior
- the current line about using a matching Solid skill alongside this one is not enough to prevent version bleed

Release action:

- add an explicit rule near the top of `solid-start-v2/SKILL.md`:
  - Start v2 architecture guidance does not imply Solid 2 core semantics
  - check the installed Solid version before applying 2.x-specific core advice
- add evals that test Start v2 plus Solid 1.x-compatible app situations

## Per-skill audit

### `solid-js-2.x-api-changes-and-best-practices`

Status: strongest content coverage, highest maintainability risk

Strengths:

- good coverage of the main 2.x mental-model shifts from `solid.md`
- correctly emphasizes batching, split effects, top-level reads, writes under scope, async boundaries, stores, and DOM changes
- good caveat that repo-installed versions beat generic guidance

Risks to fix before release:

- file is too large and should be split into references
- it contains material that may be better handled by the migration skill when the user is upgrading existing code
- current evals only cover async loading, top-level reads, and a small batching/effect example

Coverage gaps in evals:

- directives via `ref`
- `For keyed={false}` and accessor children
- `action`, `refresh`, optimistic primitives
- `createProjection` and function-form stores
- `merge`/`omit`/`snapshot`
- ownership and `runWithOwner(null, ...)`

Recommended edit pass:

1. split references out of `SKILL.md`
2. tighten the opening decision rule so narrow migration asks route to the migration skill
3. add 3-5 more evals covering DOM, stores, actions, and control flow

### `solid-js-2.x-migration`

Status: good structure, needs stronger release tests and overlap hardening

Strengths:

- separates mechanical and semantic migration work well
- correctly treats warnings and tests as migration feedback loops
- includes real migration caveats from the source material, especially ecosystem lag and test behavior

Risks to fix before release:

- current trigger boundary with the 2.x API skill is still too soft
- no bundled references yet, even though this skill has a natural split between rename-map, semantic rewrites, and ecosystem caveats
- current evals do not test config-level migration issues or ecosystem breakage handling

Coverage gaps in evals:

- `solid-js/web` and store import migration in config or tests
- test-library or alias workarounds
- linked-package dedupe issues
- `track before await` and async migration pitfalls
- `Context.Provider` and `createComputed` migration examples

Recommended edit pass:

1. add a short "use this for upgrades, not narrow API questions" section near the top
2. split deeper examples into references
3. expand evals to include tooling and ecosystem prompts, not just component snippets

### `solid-js-1.x-best-practices-and-api`

Status: stable and coherent, but under-tested for release

Strengths:

- aligns well with the 1.x best-practices material in `solid.md`
- good emphasis on props, getters, derivation, `createEffect`, stores, and control-flow components
- strong debugging checklist

Risks to fix before release:

- the skill is mostly blog-derived guidance, so it needs stronger eval coverage to prove it produces the right practical corrections
- it currently has no bundled references, which is acceptable, but a small reference split could still help maintenance if the file continues to grow

Coverage gaps in evals:

- `createResource` versus `createEffect(async ...)`
- `splitProps` / `mergeProps`
- `For` versus `Index`
- store-helper guidance with `produce`, `reconcile`, and `unwrap`
- context and accessor-shaped prop edge cases

Recommended edit pass:

1. keep the skill mostly as-is unless source-alignment issues appear during test runs
2. expand evals to cover async and stores
3. add one near-miss test so it does not try to answer 2.x questions in a 1.x frame

### `solid-start-v2`

Status: best packaging, strongest structure, needs version-boundary hardening

Strengths:

- best organized skill in the repo
- good use of bundled references
- clear route, data, auth, middleware, API, metadata, and runtime guidance
- has a useful release checklist already

Risks to fix before release:

- Start v2 versus Solid 2 version-boundary needs to be much more explicit
- some source material in `solidstart.md` is migration-era or Vinxi-era, so the skill should label version-sensitive guidance even more strongly than it currently does
- current evals cover route reads, auth, and form writes, but miss API routes, middleware usage, image support, and runtime portability concerns

Coverage gaps in evals:

- API route for webhook or callback
- middleware that enriches `event.locals` without becoming the auth boundary
- metadata-heavy route work
- image handling with `@solidjs/image`
- migration-sensitive config help

Recommended edit pass:

1. add a stronger version-boundary section near the top
2. add 2-4 evals covering API routes, middleware, and ecosystem-sensitive work
3. keep the bundled reference pattern and use it as the model for the other skills

## Release order

Recommended order:

1. `solid-js-2.x-migration`
2. `solid-js-2.x-api-changes-and-best-practices`
3. `solid-start-v2`
4. `solid-js-1.x-best-practices-and-api`

Why this order:

- the two 2.x skills have the most overlap and the highest correctness risk
- `solid-start-v2` is already structured well, so it benefits from being tuned after the 2.x boundaries are clearer
- the 1.x skill looks comparatively stable and lower-risk

## Release workflow to execute next

### Phase 1: content hardening

- tighten scope boundaries in all four skills
- split large 2.x files into references
- add Start v2 versus Solid version boundary language

### Phase 2: eval hardening

- expand each `evals/evals.json`
- add assertions
- add overlap and near-miss tests

### Phase 3: review loop

- run with-skill and baseline evals
- generate the review viewer
- collect human feedback
- revise skills based on both benchmark results and human review

### Phase 4: trigger optimization

- only after content stabilizes, generate trigger evals
- optimize descriptions
- rerun overlap checks

### Phase 5: packaging

- package each skill
- attach short release notes with version assumptions and intended trigger surface

## Immediate next edits I would make

1. Refactor `solid-js-2.x-api-changes-and-best-practices` into a leaner `SKILL.md` plus references.
2. Add stronger boundary language to `solid-js-2.x-migration` and `solid-start-v2`.
3. Expand all four eval sets before any release claim.
4. Run the first full review plus benchmark loop on the 2.x pair first.
