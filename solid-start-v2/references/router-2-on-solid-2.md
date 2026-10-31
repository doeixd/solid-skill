# Router 2 On Solid 2

Use this reference when the app runs `@solidjs/router` 2.x (`next`) on `solid-js` 2.x, with or without `@solidjs/start`. Several router 1 patterns elsewhere in this skill do not exist there.

Verified on `@solidjs/router@2.0.0-next.43` with `solid-js@2.0.0-rc.20`. Check `node_modules/@solidjs/router/dist/index.d.ts` for the installed export list before relying on any name below.

## What changed from router 1

| Router 1 on Solid 1 | Router 2 on Solid 2 |
| --- | --- |
| `createAsync(() => getPost(id))` | Not exported. Read the query in a core async memo: `createMemo(() => getPost(props.params.id))` under `<Loading>` |
| `action(fn, key)` from the router | `formAction(fn, name)`. `action` is a deprecated alias; core `action` from `solid-js` is a different primitive (transitions and optimistic writes) |
| `useSubmission(action)` | `useSubmissions(formAction, filter?)` |
| `useAction(action)` | Deprecated: call the server function, or a core `action`, directly. The router applies a server function's revalidation and redirect on any call |
| `<Router>` with `<Route>` children | `createRouter({ routes, preload })` returns the `Router` component; `defineRoute` and `defineRoutes` type route definitions |

Also new: `liveQuery`, link preload strategies (`intentPreload`, `viewportPreload`, `eagerPreload`, `tapPreload`), `pendingLinks`, and single-flight mutations (a mutation's response seeds fresh route data into the `query` cache).

## App-level preload as a loader

`createRouter` accepts a top-level `preload` beside the routes. Its result reaches the root component as `props.data`:

```tsx
const Router = createRouter({
  preload: async ({ location, intent }) => {
    if (intent !== "initial" || location.pathname !== "/") return undefined;
    return loadFirstPage(new URLSearchParams(location.search));
  },
  routes: [
    { path: "/", component: SearchPage },
    { path: "/compare", component: lazy(() => import("./Compare")) },
    { path: "*404", component: NotFound },
  ],
});

export default function App() {
  return <Router>{props => <Shell data={props.data}>{props.children}</Shell>}</Router>;
}
```

In the root, `createMemo(() => props.data, { deferStream: true })` makes the server hold the shell until the loader resolves, and the browser adopts the serialized result instead of re-running it. Children that read the loaded data synchronously render without suspending.

Prefer this over a `query()` call in every row when many components need data the server can fetch up front. `query()` returns a promise even on a cache hit, so on the server every reading boundary suspends and is spliced back in later. In one app, 37 rows doing that cost 4.0 ms of render per request, against 1.9 ms with a loader and synchronous reads.

## `query()` in practice

```ts
export const fetchRange = query(async (label: string, offset: number, limit: number) => {
  return searchDomains({ label, offset, limit });
}, "domains");

const key = fetchRange.keyFor(label, offset, limit); // one entry's key
fetchRange.key; // the prefix shared by every entry
```

- It needs router context: inside a memo it calls `useRouter()`, which throws outside the router. A `query()` read makes the router a dependency of that component.
- It dedupes calls in flight and caches per request on the server.
- It never aborts a request. If keystrokes drive reads, debounce the input the reads depend on; otherwise each keystroke's request runs to completion.
- `query.set(key, value)` seeds an entry with a resolved value (never a promise). Seed from a loader's data in an effect's apply function, which is where writes belong, so later reads are cache hits.
- `revalidate(keys, true)` re-runs the readers of those keys; `query.clear()` drops everything.
- Cached transitions such as re-sorting or scrolling back settle inside the transition: in measurement they showed no fallback flash even though each hit is a promise.

## Mutations and revalidation

A POST server function counts as a mutation, and a mutation that declares no keys revalidates every query on screen. For a write that changes no query (saving a cart cookie, logging), return `respond(value, { revalidate: "" })` from `@solidjs/web`. Name keys to revalidate only those.

## Search params as state

`useSearchParams()` returns a reactive params object and a setter. When an input drives the URL, keep a local signal as the source of truth while typing and write the URL from the handler with `setParams(next, { replace: true, scroll: false })`. Reading the value back from the URL makes the input wait for each navigation. Listen for `popstate` (in `onSettled`, with cleanup) so Back and Forward restore earlier values.
