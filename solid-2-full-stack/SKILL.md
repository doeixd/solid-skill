---
name: solid-2-full-stack
description: "Guidance for building full-stack Solid 2 apps on `@solidjs/vite-plugin` start mode with `@solidjs/router` 2.x and `\"use server\"` functions, the setup that replaces SolidStart. Use this whenever the user configures `solid({ start, ssr, serverFunctions })`, writes entries, a Document, middleware, `setup`, or `env.ts`, or works with `createRouter`, typed `paths`, route preload, `query`, `liveQuery`, `formAction`, plain-anchor links, link preloading, server functions, `GET()`, `respond`, single-flight mutations, `virtual:solid-ssr-handler`, or deploying a Solid 2 SSR app. Use it for new Solid 2 full-stack apps even when the user says \"SolidStart\", unless the project actually depends on `@solidjs/start`."
---

# Solid 2 Full Stack

Use this skill for Solid 2 apps where `@solidjs/vite-plugin` start mode owns the serving layer, `@solidjs/router` 2.x owns routing and data, and `"use server"` functions are the backend. Start mode is the replacement for SolidStart: the plugin handles entries, dev serving, SSR, middleware, server functions, env, and the build.

For component-level reactivity, async semantics, hydration rules, and server-function calling patterns, use the Solid 2 API skill (`solid-js-2x-api-changes-and-best-practices`) alongside this one. For an existing app that depends on `@solidjs/start`, use the `solid-start-v2` skill instead.

## First step

Check `package.json` before writing code:

- `@solidjs/vite-plugin` with `start` in `vite.config.ts`: this skill.
- `@solidjs/start` as a dependency: an existing SolidStart app; use `solid-start-v2`, and only suggest moving to start mode if the user asks.
- `@solidjs/router` major version: router 2 (`2.0.0-next.*`) follows `references/router-2.md`; router 0.x is the Solid 1 router.
- Installed versions win over this skill. The plugin and router READMEs in `node_modules` are the authority for their options.

Then classify the task: app setup or config, a route or page, a server read, a mutation or form, links and navigation, request middleware or auth, environment variables, tests, or deployment.

## Default architecture

```text
vite.config.ts        solid({ start: { middleware }, ssr: true, serverFunctions: true })
env.ts                server/client Standard Schema validators (optional)
src/Document.tsx      <html> shell with <HydrationScript />
src/App.tsx           renders the router instance
src/router.tsx        createRouter({ routes, preload }), exports Router and paths
src/routes/ or pages/ route components
src/server/*.ts       "use server" functions, plus pure logic they call
src/middleware.ts     fetch-style middleware: auth, headers, REST endpoints
```

- Let the plugin generate entries unless you need to control `render()` or hydration timing.
- One router per app, created at module level from config objects. The root layout is its render-prop child and owns app-wide state through context.
- Reads are `query(fn, key)` calls read with `createMemo` under `<Loading>`. There is no `createAsync`.
- Writes are server functions called from handlers or core `action`s, or `formAction` for forms. Every POST server function counts as a mutation; return `respond(value, { revalidate: "" })` from one that changes no query.
- Data the first render needs and the server can fetch cheaply goes in the router's app-level `preload`, read synchronously by components.
- Links are plain `<a href={paths.x}>`. Active state is CSS on `[data-active]` and `[aria-current="page"]`.
- Secrets live in `virtual:env/server`, middleware, `"use server"` modules, or modules that `import "server-only"`.

## Reference guide

- for `vite.config.ts` options, entries, SSR versus client mode, middleware, `setup`, typed env, error boundaries, the server-function endpoint and its `configure` module, `server-only` markers, hosts, build tuning, or tests: read `references/start-mode.md`
- for routes, params, typed paths, links, preloading, `query`, `liveQuery`, `formAction`, submissions, single flight, search params, or navigation hooks: read `references/router-2.md`
- for hydration rules, SSR suspension cost, `deferStream`, the hydration channel, `GET()`, `respond`, and server components: read `references/ssr-hydration-and-server-functions.md` in the Solid 2 API skill

## Quality checklist

Before finishing substantial work, check:

- does every owner (memo, effect) exist on both server and client, so hydration ids line up?
- are first-render reads either preloaded or behind a `<Loading>` boundary, with no read suspending per row on the server?
- do POST server functions that change nothing return `respond(..., { revalidate: "" })`?
- are reads that should be HTTP-cacheable declared with `GET()` or called through `query()`?
- does server-only code stay out of client bundles (`server-only` marker, `virtual:env/server` only on the server)?
- do server functions validate their inputs? Anything can call the endpoint.
- are `httpStatus(404)` and redirects set during render where the route needs them?
- is the change verified in a production build (`vite build && vite preview`), not only in dev?

## Common corrections

| Pattern | Correction |
| --- | --- |
| Adding `@solidjs/start` or an `app.config.ts` to a new Solid 2 app | Use `@solidjs/vite-plugin` with `start: true` (plus `ssr: true` for SSR) |
| `ssr: { ... }` options object | `ssr: true` with the options under `start: { ... }` |
| A hand-written `index.html` and dev server script in start mode | Delete them; the plugin serves the document |
| `createAsync`, `<A>`, `<Route>` children, `useSubmission` | `createMemo` over `query()`, plain `<a>`, `createRouter({ routes })`, optimistic primitives with `.onSubmit` and `useSubmissions` |
| Reading `process.env` secrets from shared modules | `virtual:env/server` in server-only modules |
| Registering the single-flight collector or CSRF origins in app code | `serverFunctions: { configure: "./src/server-config.ts" }` |
| Running tests with the start plugin and wondering why server code acts like the client | Give server tests a Vitest project with `environment: "node"` |
| Measuring performance only in dev | Measure `vite build && vite preview` or the deployed site |
