# Router 2 On Solid 2

Use this reference when the app runs `@solidjs/router` 2.x (`next`) on `solid-js` 2.x, with or without `@solidjs/start`. Most router 1 patterns elsewhere in this skill do not exist there.

Sources: the `@solidjs/router@2.0.0-next.43` README and type declarations, plus field notes from a production SSR app on `solid-js@2.0.0-rc.20` (marked **In practice**). The router's own README calls this line "1.0" and the Solid 1 line "0.x", even though npm publishes it as `2.0.0-next`. Before relying on a name below, check `node_modules/@solidjs/router/README.md` and `dist/index.d.ts` for the installed version.

## Migration map from the Solid 1 router

| Solid 1 router | Router 2 on Solid 2 |
| --- | --- |
| `<Router root={App}>` with JSX `<Route>` children | `const Router = createRouter({ routes: [...] })` at module level, rendered as `<Router>{props => <App {...props} />}</Router>` |
| `<HashRouter>`, `<MemoryRouter>` | `createRouter({ routes, history: hashHistory() })`, or `memoryHistory("/initial")` |
| `<Router url>` / `<StaticRouter>` for SSR | Automatic from the request URL; `memoryHistory(url)` when there is no request (SSG, tests) |
| `root` prop, `rootPreload` | The render-prop child; the factory's `preload` option |
| `<A href activeClass end>` | Plain `<a href>`. Style `[data-active]` and `[aria-current="page"]` instead of active classes |
| `<Navigate href>` | Call `useNavigate()` during component setup, or redirect from a preload |
| `useCurrentMatches` | `useRouteMatches` |
| `createAsync`, `createAsyncStore` | Gone. Read `query()` results with `createMemo`, `createProjection`, `createOptimistic`, or `createOptimisticStore` |
| `cache` | `query` |
| router `action` | `formAction` (the old name is a deprecated alias; core `action` from `solid-js` is a different primitive) |
| `useSubmission` (in-flight state) | Gone. In-flight UI moves to optimistic primitives fed by `.onSubmit(...)`; `useSubmissions` returns settled history |
| `useAction` | Deprecated, no replacement: call the server function, or a core `action`, directly |
| `redirect`, `reload`, `json` from the router | `redirect`, `reload`, `respond` from `@solidjs/web` |
| `isRouting`, `pendingTarget` on router context | `useIsRouting()`; the destination via `isPending` and `latest` on `location.pathname` |
| Hover preloading by default | Opt-in: `preloadLinks: intentPreload()` |

## Setup

Create one router per app, at module level, from config objects. The instance is the provider component and carries a typed URL builder:

```tsx
import { lazy } from "solid-js";
import { createRouter } from "@solidjs/router";

export const Router = createRouter({
  routes: [
    { path: "/", component: lazy(() => import("./pages/Home")) },
    {
      path: "/users/:id",
      component: lazy(() => import("./pages/User")),
      children: [
        { path: "/", component: lazy(() => import("./pages/UserOverview")) },
        { path: "/settings", component: lazy(() => import("./pages/UserSettings")) },
      ],
    },
    { path: "*404", component: lazy(() => import("./pages/NotFound")) },
  ],
});

export const { paths } = Router;
```

```tsx
render(() => <Router>{props => <Layout>{props.children}</Layout>}</Router>, document.getElementById("app")!);
```

- The render-prop child is the root layout. It stays mounted, receives the matched route as `props.children`, and is the place for app-wide providers and navigation.
- The same module serves the server: the instance reads the request URL there. There is no separate server router.
- The route tree is immutable and there is one router per app. A router inside another router is not supported and warns in development. Compose large apps by spreading subtrees into `routes`.
- Wrap route arrays defined in other files in `defineRoutes([...])`. Without it (or `as const`), extracted arrays widen to `string` paths and `paths` loses its types.

### Instance or hook

The rule: if two concurrent server requests could get different answers, it is a hook; otherwise it is on the instance.

- Instance (facts about the app): `paths`, `match(url)`, `routes`, `config`.
- Hooks (facts about the session): `useLocation`, `useParams`, `useNavigate`, `usePreloadRoute`, `useIsRouting`, `useRouteMatches`, `useSearchParams`.

Prefer hooks. Import the instance only for typed URLs or matching outside a render; components that import it can create import cycles with the router module that references them.

## Route definitions

| key | description |
| --- | --- |
| `path` | `string` or `string[]` path segment |
| `component` | Component for the matched segment |
| `children` | Nested definitions, or a thunk `() => import("./section/routes")` for a lazy subtree |
| `preload` | Runs on navigation and on link preloading |
| `matchFilters` | Per-param constraints |
| `search` | A Standard Schema validator for search params |
| `info` | Freeform metadata, read through `useRouteMatches` |

Path syntax:

- Dynamic: `/users/:id`. Params arrive decoded (`/users/%E7%8C%AB` gives `"猫"`); `paths` encodes on the way out.
- Optional: `/stories/:id?` matches `/stories` and `/stories/123`, not `/stories/123/comments`.
- Wildcard: `foo/*` or `foo/*rest` (the remainder as `params.rest`). It must be the last segment.
- Multiple paths: `path: ["login", "register"]` keeps the component mounted while switching between them.
- Nested: only leaves become routes. A parent's `component` wraps its children and renders them through `props.children`.
- Same route, different params: the component is reused, not remounted. To force a remount, wrap it in `<Show when={params.id} keyed>`.

### Match filters

An enum array, a regex, or a predicate. A failing filter means the route does not match:

```ts
import { int } from "@solidjs/router";

{ path: "/users/:parent/:id", matchFilters: { parent: ["mom", "dad"], id: int }, component: User }
```

`int` also types the param as `number` at `paths` call sites, so `paths.users("abc")` is a type error.

### Typed params

```tsx
import { defineRoute } from "@solidjs/router";

const story = defineRoute({
  path: "/stories/:id/:tab?",
  preload: ({ params }) => getStory(params.id), // string
  component: props => <Story id={props.params.id} tab={props.params.tab} />, // tab: string | undefined
});
```

For a component defined away from its route, type props with the path node as a witness, imported as a type to avoid a runtime cycle: `RouteProps<typeof Router.paths.stories, StoryData>` or `RouteComponent<...>`. Anywhere under the route, `useParams(paths.stories)` is typed from the tree.

### Lazy subtrees

`children: () => import("./admin/routes")` keeps a section's whole route table out of the initial bundle. It loads when something needs it (a link preload, a navigation, or a server match beneath it). Types still flow through the import, so `paths.admin.users(2)` typechecks before the code loads. Keep the thunk deterministic; do not switch tables on runtime state.

### Typed search params

```tsx
const routes = defineRoutes([
  {
    path: "/search",
    component: Search,
    search: v.object({ q: v.optional(v.string(), ""), page: v.optional(v.pipe(v.unknown(), v.transform(Number)), 1) }),
  },
]);

const [search, setSearch] = useSearchParams(paths.search);
search.page; // number
setSearch({ page: search.page + 1 });
```

Without a schema, `useSearchParams()` returns raw strings. Its setter merges, removes keys set to `""`, `undefined`, or `null`, and navigates without scrolling.

## Typed paths

`paths` is a proxy built from the route tree. Properties descend static segments, calls bind params, then an optional search object and hash:

```ts
paths.users(123); // "/users/123"
paths.users(2).settings; // "/users/2/settings"
paths.users(2, { tab: "x" }, "comments"); // "/users/2?tab=x#comments"
paths(); // "/"
```

Nodes coerce to strings, so they work directly in `href`, `navigate()`, and `redirect()`. A segment not in the tree, or a param of the wrong type, is a compile error.

## Links

There is no link component. The router intercepts same-origin clicks on plain `<a>` elements through delegation:

```tsx
<a href={paths.users(user.id).settings}>Settings</a>
<a href={paths.login} replace>Log in</a>
<a href={paths.docs} noscroll>Docs</a>
```

| attribute | effect |
| --- | --- |
| `replace` | Replace the history entry |
| `noscroll` | Do not scroll to the top |
| `state` | JSON string pushed as history state |
| `preload` | `"false"` opts out of all preloading; `"viewport"` or `"eager"` opts into that strategy |
| `link` | Marks a router link when `explicitLinks: true` is set |
| `target` | Any value (such as `_self`) leaves the link to the browser |

Link state is attributes, styled with CSS:

```css
nav a[aria-current="page"] { font-weight: 600; } /* same path and same query */
nav a[data-active] { color: var(--accent); }    /* path is the link's or under it */
a[data-pending] { opacity: 0.6; }               /* in-flight target; needs links: pendingLinks */
```

- The root link only ever matches exactly, so it does not light up on every page.
- `data-pending` is opt-in: `createRouter({ routes, links: pendingLinks })`. Back and Forward never mark links pending.
- An `aria-current` you set yourself is left alone.
- For custom link components, `useLinkState(() => href, { end? })` returns `current()`, `active()`, and `pending()` by the same rules, with or without `pendingLinks`.

## Preloading

Nothing is preloaded unless the router is configured to. Preloading code loads the matched routes' `lazy()` components; preloading data also runs their `preload` functions with `intent: "preload"`.

```tsx
const Router = createRouter({ routes, preloadLinks: [viewportPreload({ all: true }), intentPreload()] });
```

| strategy | preloads when | default |
| --- | --- | --- |
| `intentPreload({ delay = 20, data = true })` | pointer rests on the link, focus, or touch start | code and data |
| `tapPreload({ data = true })` | pointer down, ahead of the click | code and data |
| `viewportPreload({ all, data, delay = 100, rootMargin })` | the link stays in view, then the browser is idle | code |
| `eagerPreload({ all, data })` | page loaded and the browser idle | code |

- `viewportPreload` and `eagerPreload` apply only to links with `preload="viewport"` or `preload="eager"`, unless created with `{ all: true }`. They skip Save-Data and 2g connections.
- A boolean `preloadLinks` is a type error and preloads nothing.
- Data preloads from viewport or eager strategies are only worth it when the cache outlives the moment of clicking (an HTTP-cached `GET` server function, for example); `query`'s browser preload cache lasts 5 seconds.
- `usePreloadRoute()(paths.users(2), { preloadData: true })` does the same work by hand.

### Preload functions

A route's `preload({ params, location, intent })` starts fetching in parallel with loading the route. `intent` is `"initial"` (first render), `"navigate"` (router navigation), `"native"` (Back or Forward), or `"preload"` (link preloading). Start the query and do not await it: `preload: ({ params }) => void getUser(params.id)`.

### App-level preload as a loader

The factory's `preload` runs once per mount or request with the merged params of every match, and its result reaches the root render-prop as `props.data`:

```tsx
const Router = createRouter({
  preload: async ({ location, intent }) => {
    if (intent !== "initial" || location.pathname !== "/") return undefined;
    return loadFirstPage(new URLSearchParams(location.search));
  },
  routes,
});

export default function App() {
  return <Router>{props => <Shell data={props.data}>{props.children}</Shell>}</Router>;
}
```

**In practice:** in the root, `createMemo(() => props.data, { deferStream: true })` makes the server hold the shell until the loader resolves, and the browser adopts the serialized result. Children that read the loaded data synchronously render without suspending. This beat a `query()` call per row: `query()` returns a promise even on a cache hit, so on the server every reading boundary suspended and was spliced back in later. With 37 rows that cost 4.0 ms of render per request, against 1.9 ms with the loader.

## Data

### `query`

```ts
const getUser = query(async (id: string) => (await fetch(`/api/users/${id}`)).json(), "users");

const user = createMemo(() => getUser(params.id)); // no router-specific async wrapper
const todos = createProjection(() => getTodos(), []); // deeply reactive object data

getUser.key; // "users", every entry
getUser.keyFor(5); // "users[5]", one entry
```

A query:

1. dedupes on the server for the lifetime of the request
2. fills a 5-second preload cache in the browser, so a link preload and the route entry share one fetch
3. refetches by key on revalidation
4. serves Back and Forward from cache for up to 5 minutes (user-initiated navigation bypasses it)

- A `"use server"` function called through `query()` is switched to GET automatically, so it is not treated as a mutation.
- A query may redirect: throwing or returning `redirect()` from `@solidjs/web` navigates instead of resolving.
- `revalidate(key)` or `revalidate(keys, true)` re-runs readers. `query.set(key, value)` seeds an entry with a resolved value (never a promise); `query.clear()` drops everything.

**In practice:**

- `query()` needs router context: inside a memo it calls `useRouter()`, which throws outside the router.
- It never aborts a request. If keystrokes drive reads, debounce the input the reads depend on.
- Seed the cache from loader data with `query.set` in an effect's apply function, where writes belong, so later reads are cache hits.
- Cached transitions such as re-sorting or scrolling back settled inside the transition, with no fallback flash, even though each hit is a promise.

### `liveQuery` (experimental)

A keyed query over a stream whose every yield is the current state, not an event:

```ts
const roomMessages = liveQuery(async function* (room: string) {
  "use server";
  yield await db.messages.list(room);
  for await (const _ of db.messages.watch(room)) yield await db.messages.list(room);
}, "messages");

const messages = createMemo(() => roomMessages(params.room));
```

One connection per key is shared by every consumer and closes when the last one leaves. It reconnects with backoff on transient failures. SSR renders the first value and hydration reconnects. `revalidate(key)` reconnects; mutations do not sweep it. `roomMessages.status(room)` reads `"idle" | "connecting" | "connected" | "reconnecting" | "closed"`.

### `formAction`

A form action is a core action with a URL: submission tracking, `aria-busy`, and posts that work without JavaScript.

```tsx
import { formAction } from "@solidjs/router";
import { redirect } from "@solidjs/web";

const updateUser = formAction(async (form: FormData) => {
  "use server";
  await db.users.update(form.get("id"), form);
  throw redirect(paths.users(form.get("id")));
});

<form action={updateUser} method="post">
  <button>Save</button>
</form>;
<button type="submit" formaction={updateUser}>Save</button>;
<form action={deleteTodo.with(todo.id)} method="post">...</form>; // typed args instead of hidden fields
```

- Forms need `method="post"`.
- A submitting form gets `aria-busy="true"` until the result is on screen, including any revalidation or redirect. Style it with `form[aria-busy] button { ... }`.
- Without JavaScript the form does a real POST and redirects back; the result reaches submission state through a one-shot cookie.
- A form action that is not a server function needs a stable name for SSR: `formAction(fn, "add-todo")`.
- Return values are the result channel; thrown errors land on `Submission.error`.

Optimistic UI uses core primitives fed by owner-scoped hooks:

```tsx
const [todos, setTodos] = createOptimisticStore(() => getTodos(), []);

const addTodo = formAction(saveTodo, "add-todo").onSubmit(todo => {
  setTodos(items => {
    items.push({ ...todo, pending: true });
  });
});
```

`.onSubmit(...)` and `.onSettled(...)` register in the current owner and are removed with it. `useSubmissions(action, filter?)` returns settled records `{ input, result, error, url, clear(), retry() }`; take the latest with `.at(-1)`.

### Mutations, single flight, and revalidation

- Single flight is on by default (`singleFlight: false` opts out). A mutation's response carries the refreshed data for the page, or for the page it redirects to, in the same round trip. Custom server setups get it from `createFlightDataCollector(Router)` in `@solidjs/router/server`.
- The router applies any server function's revalidation keys, `redirect()`, and `reload()`, whether the call came from a form or straight from an event handler. No wrapper is needed.
- Every POST server function call is a mutation. One that declares no `revalidate` keys makes the client revalidate every query on screen. Reads are exempt through `query()` (switched to GET) or `GET(fn)` from `@solidjs/web/server-functions`.
- A POST that changes no query (logging, saving a preference cookie) should return `respond(value, { revalidate: "" })` from `@solidjs/web`. Name keys to revalidate only those: `respond(value, { revalidate: [getUser.keyFor(id)] })`.

**In practice:** a cart saved through a plain POST server function would have refetched every visible price on each add; `respond(items, { revalidate: "" })` stopped it.

## Navigation and session hooks

- `useNavigate()(to, { resolve = true, replace = false, scroll = true, state })` accepts a string or a typed path node.
- `useLocation()` returns a reactive location: `pathname`, `search`, `hash`, `query`, `state`, `key`.
- `useParams()` returns a reactive, store-like object; `useParams(paths.users)` types it.
- `useIsRouting()` is true while a navigation and its data settle. To show where it is going, read the location with `isPending` and `latest`:

```ts
const location = useLocation();
const target = () => (isPending(() => location.pathname) ? latest(() => location.pathname) : undefined);
```

- `useMatch(() => "/admin/*rest")` tests a pattern you supply against the current location, independent of the route tree. `useRouteMatches()` returns the resolved route chain, outermost first, for reading `info` (breadcrumbs, titles). Type `info` app-wide by augmenting `RouteInfo` in `declare module "@solidjs/router"`.
- `useBeforeLeave(e => { ... })` can `e.preventDefault()` a navigation and later `e.retry(true)` it, for unsaved-changes prompts.
- `useInRouter()` is true under a router. The other hooks throw outside one, so a component that also renders outside it (an error fallback above the router) can check first. Call it during setup, not in an event handler.
- `useHref` and `useResolvedPath` remain for manual resolution of relative paths.

**In practice:** when an input drives the URL, keep a local signal as the source of truth while typing and write the URL from the handler with `setParams(next, { replace: true, scroll: false })`. Reading the value back from the URL makes the input wait for each navigation. Listen for `popstate` in `onSettled`, with cleanup, so Back and Forward restore earlier values.

## Other environments and integrations

- History adapters are imports: `browserHistory()` (client default), `hashHistory()`, `memoryHistory("/path")`. Unused ones stay out of the bundle.
- `Router.match("/users/2/settings?tab=x")` matches any URL with no rendering, for middleware, sitemaps, and tests.
- Other config: `base`, `actionBase` (default `/_server`), `explicitLinks`, `transformUrl`.
- File routes: `fileRoutes(pageRoutes)` from `@solidjs/router/fs` maps a `virtual:file-routes` manifest; route files export the component as `default` and config as `route = defineFileRoute("/blog/:id", { ... })`.
- Server component routes (experimental): `serverRouteComponent(query(fn, key))` uses a `"use server"` function that returns a component as a route's `component`. It is called with `{ params, search }` derived from the route. `Router.keysFor(url)` returns the query keys of the server routes a URL shows, for naming what a mutation changed. Requires `serverFunctions: { components: true }` in `@solidjs/vite-plugin`; see the Solid 2 skill's SSR reference for its bundle cost.

## Common mistakes on router 2

| Mistake | Fix |
| --- | --- |
| `import { A } from "@solidjs/router"` or `activeClass` | Plain `<a>`; style `[data-active]` and `[aria-current="page"]` |
| JSX `<Route>` children or `<Router root>` | `createRouter({ routes })` and the render-prop root |
| `createAsync(() => getUser(id))` | `createMemo(() => getUser(id))` under `<Loading>` |
| `useSubmission(action).pending` | `formAction(...).onSubmit(...)` with optimistic primitives; `form[aria-busy]` for styling |
| A second router mounted inside the app | One tree; spread subtrees or use lazy `children` |
| Expecting hover preloading | Add `preloadLinks: intentPreload()` |
| A POST server function that changes nothing | `respond(value, { revalidate: "" })` |
| Extracted route arrays losing `paths` types | Wrap them in `defineRoutes` |
