# SSR, Hydration, And Server Functions

Use this reference when the task involves server rendering, streaming, hydration mismatches, `"use server"` functions, cacheable reads, or server components in a Solid 2.x app that renders with `@solidjs/web`.

The rules below were verified on `solid-js` / `@solidjs/web` `2.0.0-rc.20` with `@solidjs/vite-plugin` `3.0.0-next.35`. Where the installed package's `CHEATSHEET.md` or `skills/` disagree, trust the installed package.

## Hydration ids: same owners on both sides

Hydration matches server HTML to components by ids handed out in creation order. Every owner (memo, effect, projection, component) takes the next id. An owner created on only one side shifts every id after it, so the browser claims none of the server's HTML: the page looks right but nothing responds.

Wrong:

```ts
if (!isServer) {
  createEffect(source, value => seedCache(value));
}
```

Right: create it on both sides. Effects never apply on the server, so the apply function is already client-only.

```ts
createEffect(source, value => seedCache(value));
```

Put `isServer` checks inside functions, not around primitives. A memo that genuinely must not take part in hydration can opt out with `{ transparent: true }` (it consumes no id and computes live).

Symptom to recognize: SSR output is correct, dev reports hydration key misses (two ids differing by one), and typing or clicking does nothing.

## What hydration adopts

- The browser adopts each async read the server answered (value or error) and does not ask again.
- Adopted computes still run once to track reads, with a fake `Promise` and `fetch` that never settle. Do not cache a promise or `import()` created inside a compute in anything that outlives it; cache resolved values.
- `ssrSource` on memos, signals, and projections chooses the initial value: `"server"` (default, adopt the serialized value), `"hybrid"` (adopt, then keep consuming a stream), or `"client"` (skip serialization, compute after hydration).

## Server rendering cost: avoid suspensions you do not need

Each read that suspends during SSR leaves a hole that the stream renders later and splices back in. That bookkeeping is real: in one app, about 37 list rows each suspending on an already-cached promise moved server render from 1.3 ms to 4.0 ms per request.

When the data can be fetched before render, fetch it once at the top (a loader, or one async memo) and let rows read it synchronously:

- an async memo may return a plain value when the data is in hand, and a promise only when it is not
- rows that read synchronously render inline, with no boundary work on the server and no fallback flash in the browser

That change brought the same render to 1.9 ms.

## `deferStream`

`createMemo(fn, { deferStream: true })` makes the server hold the stream until that value resolves, instead of streaming the fallback first. Use it when the data is cheap to get on the server (the backend runs in-process) and the first HTML should contain it. Leave it off for slow or optional data, where streaming the shell first is the point.

## Passing your own data to the client

The hydration channel carries values the server computed so the client does not recompute them:

```ts
import { getHydrationWriter, takeHydrationValue, isServer } from "@solidjs/web";

const readInitialCart = () => {
  if (isServer) {
    const names = namesFromCookie(getRequestEvent()?.request.headers.get("cookie"));
    getHydrationWriter()?.write("cart", names);
    return names;
  }
  const fromServer = takeHydrationValue<string[]>("cart");
  if (fromServer?.status === "resolved") return fromServer.value;
  return namesFromCookie(document.cookie); // page was not server-rendered
};
```

This also avoids reading `document.cookie` during hydration, which is a synchronous round trip to the browser's network process (measured at about 24 ms in one app).

## Response status and headers from components

`httpStatus(404)` from `@solidjs/web` inside a not-found component sets the server response's status during SSR, and is a no-op in the browser.

## Painting before hydrating

`<HydrationScript />` in the document head captures clicks and typing before hydration and replays them afterwards. When the server HTML is complete and styled, hydrating on the frame after first paint keeps hydration from holding that paint:

```ts
const start = () => hydrate(() => <App />, document);
if (document.visibilityState === "visible") requestAnimationFrame(() => setTimeout(start, 0));
else start(); // hidden tabs get no animation frames
```

Measure before adopting: it helps when the bundle arrives before first paint, and does nothing when it does not.

## Server functions

A function whose body starts with `"use server"` runs in-process on the server (including during SSR) and becomes a typed RPC stub in the browser, which POSTs to `/_server`. Validate inputs inside it: anything can call the endpoint.

### Cacheable reads with `GET()`

```ts
import { getRequestEvent } from "@solidjs/web";
import { GET } from "@solidjs/web/server-functions";

export const search = GET(async (input: SearchInput) => {
  "use server";
  const event = getRequestEvent();
  const result = await runSearch(input);
  // During SSR this runs inside the page's own request: only set cache
  // headers for calls that arrived over HTTP, or the page gets cached too.
  if (event && new URL(event.request.url).pathname.startsWith("/_server")) {
    event.response?.headers.set("cache-control", "public, max-age=60");
  }
  return result;
});
```

`GET()` makes the browser call a GET with the arguments in the URL, so the HTTP cache can answer repeats.

### `respond(value, { revalidate })`

Under Solid Router, any POST server function counts as a mutation, and a mutation that declares nothing revalidates every query on screen. A write that changes no query should say so:

```ts
import { respond } from "@solidjs/web";

export async function saveCart(names: string[]) {
  "use server";
  const items = priceCart(names);
  setCartCookie(items);
  return respond(items, { revalidate: "" }); // "" or [] = nothing; "*" = everything
}
```

`redirect()` and `reload()` from `@solidjs/web` take the same `revalidate` option.

### Unit tests

Modules that use `"use server"` or `GET()` need the Solid compiler, so a Vitest config without the Solid plugin fails to load them. Either keep pure logic (validation, search, pricing) in modules with no server-function wrappers and test those directly, or run tests with the plugin and give server tests a Vitest project with `environment: "node"` (see the `solid-2-full-stack` skill). When a test run reports its totals, check the "Test Files" line too: a file that fails to load does not show up in the passed-test count.

## Server components

`@solidjs/web/frames` provides server components: a `"use server"` function that returns a template, whose code never reaches the browser. Read `node_modules/@solidjs/web/skills/server-components/SKILL.md` before writing one.

Enabling them (`serverFunctions: { components: true }` in `@solidjs/vite-plugin`) switches every server-function call to a frames-capable transport. In one app that added about 14 KB gzip to a page that used no server components, more than the 5 KB code-split page it was meant to save. Measure the whole app's bundle before keeping them.

## App setup

Configuring `@solidjs/vite-plugin` start mode (entries, middleware, env, the server-function endpoint, hosts, and tests) and `@solidjs/router` 2 is covered by the `solid-2-full-stack` skill. The dev toolbar (`@solidjs/start-devtools`) is development-only and is not in production builds.
