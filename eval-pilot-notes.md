# Eval Pilot Notes

This is a lightweight pilot on the first two evals for the two Solid 2.x skills.

Method used:

- baseline run: plain `claude -p <prompt>`
- guided run: `claude -p` with an instruction to read and follow the local `SKILL.md`

This is not the full benchmark workflow yet. It is a quick signal check to see whether the current skill content is improving answers.

## `solid-js-2.x-api-changes-and-best-practices`

### Eval 1: async profile loader

Baseline:

- decent explanation of `Loading`
- still made some questionable assumptions about exact API shape
- not obviously grounded in the local skill files

Guided run:

- improved substantially
- used `Loading`
- explained the 1.x `createResource` plus `Suspense` difference in the right direction
- mentioned tracked reads before `await`

Remaining issue:

- the generated example still needs careful source verification for exact runtime-valid 2.x beta code shape

### Eval 2: top-level reactive read warning

Baseline:

- already fairly good
- correctly identified the top-level prop read problem and fixed it

Guided run:

- slightly better framing
- more explicitly explained the Solid 2 dev warning and why top-level reads break reactivity

Takeaway:

- this skill is already helping on explanation-style prompts
- the bigger risk area is exact API correctness on new-code generation prompts

## `solid-js-2.x-migration`

### Eval 1: migrate 1.x component to 2.x

Baseline:

- clearly wrong in multiple ways
- suggested `createAsync`, `mount`, and `Suspense` instead of following the migration guidance in this repo
- failed the mechanical-versus-semantic framing badly

Guided run:

- much better than baseline
- correctly separated mechanical and semantic changes
- correctly moved `solid-js/web` to `@solidjs/web`
- correctly discussed `Loading`, `Index` to `For keyed={false}`, and `onSettled`

Remaining issue:

- the bigger remaining risk is exact beta-era API details and package-boundary correctness, not the simple act of returning `fetchUsers()` from a computation. If `fetchUsers()` already returns a promise, that part is compatible with the 2.x async model.

### Eval 2: failing tests after setters

Baseline:

- wrong fix suggestion
- used `flushSync` instead of the `flush()` guidance captured in `solid.md`
- framed the issue too generically

Guided run:

- clearly better
- used `flush()`
- correctly framed it as a migration-era batching change
- explicitly warned against scattering `flush()` everywhere

Takeaway:

- the migration skill materially improves migration-specific reasoning
- it still needs stronger pressure toward exact 2.x code patterns in generated rewrites

## Pilot summary

- guided runs were better than baselines on all four pilot checks
- the largest gain was on migration reasoning
- the largest remaining weakness is exact code-shape correctness for Solid 2 beta examples

## Recommended next step

1. tighten a few code-generation sections in the Solid 2.x API and migration skills so they more strongly prefer explicit async computation patterns and current package boundaries
2. run the rest of the 2.x eval set
3. then expand the same eval discipline to `solid-start-v2` and `solid-js-1.x-best-practices-and-api`
