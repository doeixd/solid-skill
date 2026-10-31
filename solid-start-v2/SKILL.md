---
name: solid-start-v2
description: "Guidance for existing SolidStart v2 applications that depend on `@solidjs/start`: route, data, mutation, server, auth, middleware, metadata, and deployment patterns. Use this when the project has `@solidjs/start` in its dependencies or the user is maintaining or migrating a SolidStart app (`app.config.ts`, `@solidjs/start/http`, Vinxi or Nitro setups). For new Solid 2 full-stack apps, `@solidjs/vite-plugin` start mode, or `@solidjs/router` 2.x, use the `solid-2-full-stack` skill instead."
---

# SolidStart v2

Use this skill for apps that depend on `@solidjs/start`. Keep it focused on Start-specific architecture and server boundaries.

SolidStart has effectively been replaced by `@solidjs/vite-plugin` start mode, which owns entries, dev serving, SSR, middleware, server functions, and env without a framework package. For a new Solid 2 app, or a project already on start mode or `@solidjs/router` 2.x, use the `solid-2-full-stack` skill. Suggest moving an existing Start app to start mode only when the user asks about it.

If the task is mainly about Solid core reactivity, component semantics, or Solid 2.x API behavior, use the matching Solid skill alongside this one.

## Version boundary

- SolidStart v2 architecture guidance does not imply that the app is already using Solid 2.x core semantics.
- Start v2 and Solid 2 are not the same migration step. Check the installed Solid packages before applying 2.x-only core advice like `Loading`, `Errored`, split `createEffect`, or top-level reactive read warnings.
- Use this skill for Start route, server, auth, middleware, metadata, and runtime patterns. Pull in a Solid core skill only for component-level or reactivity-level questions.
- Check the installed `@solidjs/router` major version. The `createAsync`, router `action`, `useSubmission`, and `useAction` defaults below are router 1. On router 2 (`next`) with Solid 2, `createAsync` does not exist, the router's `action` is a deprecated alias of `formAction`, and `useAction` is deprecated: follow the router reference in the `solid-2-full-stack` skill.

## Goals

- Build SolidStart apps with the framework's native data and mutation model instead of generic fetch glue.
- Keep server-only code on the server and use the right boundary for each concern.
- Produce app code that is portable, secure, and easy to evolve.
- Stay token-efficient by reading deeper references only when the task needs them.

## First step

Classify the task before editing:

- new route or page
- server-backed read
- mutation or form flow
- auth, sessions, or protected routes
- external API or webhook endpoint
- metadata, SEO, or app polish
- config, migration, or runtime issue

Then choose the smallest SolidStart-native pattern that fits.

## Default architecture rules

- Use `query(..., key)` for server-backed reads that participate in router caching and invalidation.
- Use `action(..., key)` for writes and form submissions.
- Use `route.preload` to start route work early. Do not await there unless blocking is truly required.
- Use `createAsync(() => queryFn(...))` in route components to consume async route data.
- Put database, auth, session, filesystem, and secret-dependent logic behind `"use server"`.
- Prefer native HTML forms wired to actions before inventing client-side mutation plumbing.
- Use middleware for lightweight cross-cutting request concerns, not as the primary authorization layer.
- Use API routes only when the endpoint must be consumed outside the app UI or return non-HTML content.
- Keep server function inputs and outputs serialization-friendly.

## Decision guide

If the user needs route data:

- read `references/data-and-mutations.md`
- on `@solidjs/router` 2.x with Solid 2, use the `solid-2-full-stack` skill's `references/router-2.md`, which replaces its `createAsync` and `useSubmission` guidance

If the user needs auth, sessions, or protected pages:

- read `references/auth-middleware-sessions.md`

If the user needs an endpoint for webhooks, OAuth callbacks, REST, or files:

- read `references/api-routes-and-responses.md`

If the user needs route structure or end-to-end app design:

- read `references/app-architecture.md`

If the user needs metadata, social tags, or image guidance:

- read `references/seo-meta-and-images.md`

If the user needs setup, migration, env, or runtime guidance:

- read `references/config-and-runtime.md`

If the user needs Start v2 platform changes, Nitro 3 context, image support, or plugin-package guidance:

- read `references/v2-changes-and-ecosystem.md`

If the task needs a final quality pass before wrapping up:

- read `references/release-checklist.md`

## Working style

Start with the smallest correct SolidStart shape instead of a framework-agnostic one.

When implementing a feature, think through:

- route file and URL shape
- server and client boundary
- loading and error behavior
- redirect behavior during SSR and navigation
- validation and mutation flow
- metadata requirements
- serialization and runtime constraints

Prefer local, minimal changes when editing an existing app. Do not rewrite working route structure just to make it look more abstract.

## Strong defaults

- Reads should usually be keyed queries, not ad hoc `fetch` calls in component bodies.
- Writes should usually be keyed actions, not custom event handlers that manually sync state and navigation.
- Protected routes should check auth near the data access path and redirect there.
- Middleware can enrich the request with `event.locals`, but security-sensitive checks should live in server queries, actions, API routes, or server utilities.
- Route metadata should be deliberate. Add titles and relevant meta tags for user-facing routes.

## Quality checklist

Before finishing substantial SolidStart work, quickly check:

- is the read path modeled as a query when it should be?
- is the write path modeled as an action when it should be?
- does server-only logic stay behind `"use server"` and out of client bundles?
- are redirects happening at the correct server boundary?
- does the route need `Title`, `Meta`, or share tags?
- is middleware staying lightweight?
- are returned values simple enough to serialize safely?

## Common corrections

If you see these patterns, correct them unless the repo has a strong reason not to:

- route data fetched only in `onMount` for first render
- forms posting to custom client handlers when an action would fit better
- API routes created for UI-only reads and writes
- auth checks done only in middleware
- server helpers importing secrets from code that is also used by the client
- older Vinxi- or Nitro-2-specific examples copied into a v2 app without checking local packages
- `createAsync`, `useSubmission`, or router `action` written for an app on router 2 and Solid 2 (use a core `createMemo`, `useSubmissions`, and `formAction`)
- `<A>`, `activeClass`, JSX `<Route>` trees, or `<Navigate>` on router 2 (plain `<a>` with `[data-active]` / `[aria-current]`, `createRouter({ routes })`, and `useNavigate()` during setup)
- a POST server function that changes no query but returns a plain value, which on router 2 revalidates every query on screen (return `respond(value, { revalidate: "" })`)

## When the local repo disagrees

SolidStart v2 is still close to evolving ecosystem boundaries. If the installed project exports or package names differ from this skill, trust the repo's actual dependencies and current package exports.

Be especially careful with older examples that still mention Vinxi-era helpers. Use current local imports when available.
