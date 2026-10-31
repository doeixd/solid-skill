# `@solidjs/vite-plugin` Start Mode

Use this reference for the serving layer of a Solid 2 app: config, entries, SSR versus client rendering, middleware, per-request setup, typed env, server-function wiring, build output, deployment hosts, and tests.

Sources: the `@solidjs/vite-plugin@3.0.0-next.35` README, plus field notes from a production app on Cloudflare Workers (marked **In practice**). The plugin supports Vite 8 and 9 and needs Node `^20.19.0 || >=22.12.0`. Check `node_modules/@solidjs/vite-plugin/README.md` for the installed version's options.

## What start mode is

Start is a mode of the plugin and the replacement for SolidStart. The plugin owns entries, dev serving, and the build: no `index.html`, no entry files, and no server script are required.

```ts
// vite.config.ts
import { defineConfig } from "vite";
import solid from "@solidjs/vite-plugin";

export default defineConfig({
  plugins: [solid({ start: true, ssr: true, serverFunctions: true })],
});
```

- `start: true` is the same as `start: {}`. The object form takes `app`, `document`, `entryServer`, `entryClient`, `middleware`, `setup`, `env`, `devtools`, `errorBoundary`, `css`, and `external`.
- `ssr: true` selects SSR start mode: streaming server rendering with hydration. Without it, the same app is client-rendered onto a prerendered static shell (client mode, experimental).
- Switching an app between SPA and SSR is that one boolean: same `App`, same `Document`, same routes, same server functions.
- `ssr` must be a boolean. The old `ssr: { ... }` object form is a config error; its options moved to `start: { ... }`.
- Without `start`, `ssr: true` only turns on the SSR transforms and you provide entries and a server yourself.
- Use plain `vite`, `vite build`, and `vite preview`.

## Entries

The plugin resolves entries in this order (paths relative to the Vite root):

1. `start.entryServer` and `start.entryClient`.
2. Conventional files `src/entry-server.*` and `src/entry-client.*`. They come in pairs; providing only one is an error. The server entry exports `render(request?, context?)` returning a `renderToStream` result, an HTML string, or a `Response`. In production, a literal `"/src/entry-client.tsx"` in the rendered HTML is rewritten to the hashed asset.
3. Generated entries (zero config): the root component is `start.app`, defaulting to `src/App.tsx`, wrapped in `start.document`, defaulting to `src/Document.tsx`, else a built-in minimal shell. A custom document receives the app as `props.children` and must render the full `<html>` including `<HydrationScript />`; the client entry script is injected into `<head>` for you.

Prefer generated entries. Author entries only when you need to control `render()` or the hydration call.

**In practice:** this app authors both entries so the client can hydrate after first paint:

```tsx
// src/entry-server.tsx
import { renderToStream } from "@solidjs/web";
import manifest from "virtual:solid-manifest";

export function render() {
  return renderToStream(() => <Document><App /></Document>, { manifest });
}

// src/entry-client.tsx
import { hydrate } from "@solidjs/web";
const start = () => hydrate(() => <Document><App /></Document>, document);
if (document.visibilityState === "visible") requestAnimationFrame(() => setTimeout(start, 0));
else start();
```

`virtual:solid-manifest` is the client asset manifest, available in dev and SSR builds. Add `/// <reference types="@solidjs/vite-plugin/virtual-solid-manifest" />` for its types.

## SSR start mode: dev, build, prod

- **Dev:** `vite` streams the rendered app for HTML GET requests, with HMR and the error overlay.
- **Build:** `vite build` writes client assets and the manifest to `dist/client`, and the server bundle to `dist/server/server.js`. Server builds read the client manifest, so the plugin orders environment builds client-first, including under hosts like `@cloudflare/vite-plugin`.
- **Prod:** the server bundle's entry is `virtual:solid-ssr-handler`. It exports `handleRequest(request, options?)`, mapping a web `Request` to a streamed `Response`, and a default `{ fetch(request) }` for Workers, Nitro, Netlify Functions, Bun, and `deno serve`. Serve `dist/client` statically and send everything else to the handler.
- **Preview:** `vite build && vite preview` runs the production artifact with no server file.

Response status and headers set during render with `httpStatus()` and `httpHeader()` from `@solidjs/web` land on the wire at shell flush. A `Location` header set before the flush becomes a real 3xx redirect; one set after it falls back to a script redirect in the stream.

To pass platform objects into the request, use the `event` option: `handleRequest(request, { event: { nativeEvent: req } })`, then read `getRequestEvent().nativeEvent` anywhere in the request. The plugin's own dev and preview servers pass the Node request this way. Behind a proxy, read forwarding headers from `event.request` only when you trust the proxy.

## Middleware

`start.middleware` points at a server-only module that default-exports a fetch-style middleware `(request, next) => Response`, or an array composed in order:

```ts
// src/middleware.ts
import { getRequestEvent } from "@solidjs/web";

export default async function auth(request: Request, next: (request?: Request) => Promise<Response>) {
  getRequestEvent()!.locals.user = await userFromCookie(request);
  const response = await next();
  response.headers.set("server-timing", "app"); // headers stay mutable after next()
  return response;
}
```

- The chain fronts every request the plugin dispatches: page SSR and the server-function endpoint, in dev, preview, and production.
- It runs inside the request event, so `locals` set here are visible to pages and server functions.
- Nothing reaches the wire until the outermost middleware returns.

**In practice:** a small REST API lives here next to the RPC endpoint, matching on `new URL(request.url).pathname` and returning `Response` objects, with `next()` for everything else.

## Per-request setup

`start.setup` points at a server-only module default-exporting `(event, App) => Component | void | Promise<...>`. The generated server entry awaits it after middleware and right before `renderToStream`. Return a component to render in place of `<App />`, or nothing for a side-effect-only setup.

- It runs for page renders only, not for middleware or the server-function endpoint.
- It exists only with generated entries; configuring it alongside an authored `entry-server` is an error.
- Whatever it renders must match on the client for hydration.

With `@solidjs/router` 2 this is rarely needed: the router reads the request URL itself and its app-level `preload` replaces a per-request load step.

## Typed environment variables

A schema file at the project root, `env.ts` (or `start: { env: "./path" }`; `env: false` disables it), default-exports `server` and `client` maps of Standard Schema validators:

```ts
// env.ts
import { z } from "zod";

export default {
  server: { DATABASE_URL: z.url(), SESSION_SECRET: z.string().min(32) },
  client: { VITE_APP_NAME: z.string().min(1) },
};
```

```ts
import { env } from "virtual:env/server"; // server-only modules: every var
import { env } from "virtual:env/client"; // anywhere: VITE_-prefixed vars
```

- `.env*` files load automatically (with `process.env` winning) and are validated before anything builds. In dev, failures show in the error overlay.
- Client values are baked into the build as plain JSON; client failures fail the build. Server values are read and validated at boot, so secrets rotate without a rebuild and never appear in `dist`. Server failures only warn at build time.
- Server validators must be synchronous.
- Importing `virtual:env/server` from client code is a build error naming the importer, and a client chunk containing a server var's literal value fails the build.
- Types are generated into `solid-env.d.ts` next to the schema; keep both in the tsconfig `include`.
- Env is a start-mode feature: without `start` there is no env layer.

## Errors, CSS, devtools

- `errorBoundary` (default `true`): in production, generated entries wrap the app in an error boundary that streams a generic 500 fallback with no stack, logs the error, and commits a real 500 status when the error happens before the shell flushes. Set `false` when middleware owns error handling.
- `css.filter`: include and exclude patterns for the module graphs the dev server crawls to inline CSS (no flash of unstyled content). The default skips `node_modules`; `include` opts a dependency's CSS in. Dev only.
- `devtools`: start mode detects `@solidjs/start-devtools` (install with `-D`) and adds a development toolbar with runtime errors and server-function calls. `devtools: true` requires it and `false` disables it. With authored entries, wrap the app in `<DevToolbar>` from that package. It is never in production builds.

## Server functions

`serverFunctions: true` compiles `"use server"` functions, with the runtime from `@solidjs/web/server-functions` and the endpoint at `/_server`. The object form takes `runtime`, `endpoint`, `filter`, `directive`, `manifest`, `devMiddleware`, `configure`, and `components`.

- **Dev:** a Vite middleware serves the endpoint end to end.
- **Production:** SSR start mode's handler serves it. Without start mode, import `virtual:solid-server-function-handler` in your server entry and mount `handleServerFunctionRequest(request)` on the endpoint.
- **Client mode:** pages stay static, and `dist/server` is kept only to serve the endpoint.

`configure: "./src/server-config.ts"` pins a server-only module into the handler graph, loaded before the first dispatch everywhere. Register server-side runtime config there, such as the router's single-flight collector or extra trusted origins:

```ts
// src/server-config.ts
import { configureServerFunctionsServer } from "@solidjs/web/server-functions/server";
import { createFlightDataCollector } from "@solidjs/router/server";
import { Router } from "./router";

configureServerFunctionsServer({
  collectFlightData: createFlightDataCollector(Router),
  csrf: { origin: ["https://app.example.com"] },
});
```

Registration placed in app code only loads with the first page render, so after a dev restart the first mutation can race it. Server-function requests are same-origin protected by default; set `csrf: false` only when another trusted layer protects the endpoint.

For calling patterns (`GET()`, `respond({ revalidate })`, hydration, and server components) read `references/ssr-hydration-and-server-functions.md` in the Solid 2 API skill.

## Keeping code on the right side

The plugin claims the bare specifiers `server-only` and `client-only` as markers:

```ts
import "server-only"; // this module must never reach a client bundle

export const db = createDbClient(process.env.DATABASE_URL);
```

Importing `server-only` from a module that ends up in a client bundle fails the build, naming the importer (and the reverse for `client-only`). Add `/// <reference types="@solidjs/vite-plugin/boundary-modules" />` to an `env.d.ts` for types.

The app root and everything it imports hydrate, so they are client code. Keep secrets, database clients, and `virtual:env/server` in middleware, `"use server"` modules, or modules marked `server-only`.

## Hosts and deployment

- Any Fetchable host: serve `dist/client` statically and call the handler's `fetch(request)`.
- Hosts with their own Vite environment (`@cloudflare/vite-plugin` and similar) can adopt the plugin's `ssr` environment directly. When a provider replaces the dev environment with a non-runnable one, the plugin stands its dev middlewares down automatically.
- `serverFunctions.devMiddleware: false` hands only dev endpoint dispatch to the host, so functions see platform bindings in dev, for example workerd `env` and `ctx`. The host then loads `virtual:solid-server-function-handler` itself and should side-effect import `virtual:solid-server-function-manifest` so functions referenced only by client code still register.
- `start.external: true` hands the whole server side to a host that does not adopt the `ssr` environment, while keeping the generated entries, manifest, and `virtual:solid-ssr-handler`.
- Meta-frameworks that dispatch through their own server use the standalone `serverFunctions()` export, which never installs the dev middleware.

**In practice:** Alchemy's `Cloudflare.Website.Vite` resource runs the app's own `vite build` with Cloudflare's runtime plugin and uploads the assets and handler as one Worker, with no extra Solid configuration. It reads `public/_headers` for asset cache rules, so `/assets/*` can be `immutable`. Responses from a `workers.dev` address are not cached at Cloudflare's edge, so `Cache-Control` on `GET()` server functions only helps the browser cache there.

## Build tuning

**In practice:** with Vite 8 (Rolldown), code-split panels that share modules with the entry produced several tiny chunks, each its own preload request. Grouping shared app code into one chunk fixed it:

```ts
build: {
  rolldownOptions: {
    output: {
      advancedChunks: { groups: [{ name: "app", test: /[\/]src[\/](lib|data|server)[\/]/ }] },
    },
  },
},
```

## Compiler

`compiler` defaults to `"native"` (`@solidjs/compiler`). `"babel"` switches only the JSX transform to `@solidjs/babel-plugin` and is the escape hatch when native output looks wrong. Platforms without a native binary fall back to a WASM build automatically. `solid: { ... }` passes compiler options to either backend.

## Testing

With the plugin in the Vitest config, tests default to the client posture regardless of `ssr`: DOM codegen, non-hydratable, browser conditions, and a `jsdom` environment. Server-runtime tests (server functions, sessions, `renderToString`, anything needing `isServer`) opt out per Vitest project with `environment: "node"`:

```ts
test: {
  projects: [
    { extends: true, test: { name: "client", environment: "jsdom", include: ["src/**/*.test.tsx"] } },
    { extends: true, test: { name: "server", environment: "node", include: ["src/server/**/*.test.ts"] } },
  ],
},
```

**In practice:** a separate `vitest.config.ts` without the plugin is simpler for plain-TypeScript unit tests, but then modules with `"use server"` or `GET()` fail to load because nothing compiles them. Keep pure logic in modules without server-function wrappers, or use the plugin's server project above. Check the "Test Files" line as well as the passed count: a file that fails to load does not show up in the passed tests.

For browser tests, run Playwright against `vite preview` on `127.0.0.1` (not `localhost`, which can resolve to IPv6 while preview listens on IPv4).
