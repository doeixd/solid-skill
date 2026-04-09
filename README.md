# Solid Skills

This repo contains a small set of agent skills for SolidJS and SolidStart work, along with source notes and eval definitions used to harden them for release.

## Skills

- `solid-js-1.x-best-practices-and-api`
  - SolidJS 1.x idioms, reactivity, props, effects, stores, and control flow.
- `solid-js-2.x-api-changes-and-best-practices`
  - Solid 2.x beta or `next` APIs, changed semantics, warnings, stores, async patterns, DOM behavior, and best practices.
- `solid-js-2.x-migration`
  - Migration guidance for upgrading existing Solid 1.x codebases to Solid 2.x beta or `next`.
- `solid-start-v2`
  - SolidStart v2 architecture, route data, actions, auth, middleware, API routes, metadata, and runtime guidance.

## Source material

- `solid.md`
  - Main Solid source bundle used for these skills.
  - Includes Solid 2.0 migration and RFC material, plus supplemental best-practice and migration notes.
- `solidstart.md`
  - Main SolidStart v2 source bundle used for Start-specific guidance.

## Repo layout

Each skill directory follows the same basic shape:

```text
<skill-name>/
├── SKILL.md
├── evals/
│   └── evals.json
└── references/        # present where deeper material is split out
```

Top-level files:

- `release-audit.md`
  - Current release-readiness audit and recommended execution order.
- `README.md`
  - This file.

## Current status

This repo is in release-hardening mode.

Completed recently:

- reviewed `solid.md` in full
- wrote `release-audit.md`
- added stronger scope boundaries between the Solid 2.x API, migration, and Start skills
- added Solid 2.x reference files under `solid-js-2.x-api-changes-and-best-practices/references/`
- expanded the 2.x eval sets with broader coverage and lightweight assertions

Still to do before release:

- expand and harden evals for `solid-js-1.x-best-practices-and-api` and `solid-start-v2`
- run the full human-review and benchmark loop for each skill
- add trigger-eval coverage and optimize descriptions after content stabilizes
- package the final skills for release

## Release workflow

The working release flow for this repo is:

1. review source docs and align skill content
2. tighten skill boundaries so neighboring skills do not over-trigger
3. expand `evals/evals.json` with realistic prompts and assertions
4. run with-skill and baseline evaluations
5. review outputs and benchmark data
6. revise skills and repeat
7. optimize descriptions for triggering
8. package for release

## Recommended release order

1. `solid-js-2.x-migration`
2. `solid-js-2.x-api-changes-and-best-practices`
3. `solid-start-v2`
4. `solid-js-1.x-best-practices-and-api`

This order prioritizes the highest-risk overlap and correctness areas first.

## Notes

- `solid-start-v2` guidance is intentionally not the same thing as Solid 2 core guidance. Start v2 and Solid 2 are separate version and migration concerns.
- The Solid 2.x API skill keeps most material in `SKILL.md`, with references added to stay closer to the official 2.0 source docs without stripping out useful working guidance.
