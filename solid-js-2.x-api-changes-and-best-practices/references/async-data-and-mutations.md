# Async Data And Mutations

Use this reference when the task is about the Solid 2.0 async model, pending states, refresh, or optimistic updates.

The source of truth for this file is the migration guide plus the async-data and actions RFCs.

## Core model

- async reads move into computations instead of a dedicated `createResource` primitive
- `Loading` handles initial readiness
- `isPending(() => expr)` describes a value change in flight, not first load
- `latest(fn)` can inspect the in-flight value
- `action(...)` plus optimistic primitives are the intended mutation shape
- `refresh(...)` explicitly recomputes derived reads after writes
- stay on the Solid core boundary unless the task explicitly moves into SolidStart or router-specific APIs

## Async computations

`createMemo(async () => ...)` is a valid 2.x pattern for async derived reads.

```ts
const profile = createMemo(async () => {
  const currentId = userId();
  return fetchProfile(currentId);
});
```

Tracked reads need to happen before `await`.

When writing examples for Solid core 2.x, prefer this explicit async computation form instead of drifting into neighboring APIs such as `createAsync` from framework documentation.

Prefer:

```ts
const filtered = createMemo(async () => {
  const q = query();
  await warmCache(q);
  return search(q);
});
```

Do not read `query()` for the first time after `await`.

## `Loading`

Use `Loading` for initial readiness.

```tsx
<Loading fallback={<Spinner />}>
  <ProfileView profile={profile()} />
</Loading>
```

This replaces the old `Suspense` framing in the new async model.

## `isPending` and `latest`

Use `isPending(() => expr)` for background refresh UI once a usable value already exists.

```ts
const users = createMemo(async () => fetchUsers(filter()));
const usersPending = () => isPending(() => users());
const latestUsers = () => latest(users);
```

Important distinction:

- `Loading` is for not-ready-yet
- `isPending` is for updating-again while current UI stays visible

## `refresh(...)`

After a write, explicitly recompute derived reads when needed.

```ts
refresh(() => query.user(id()));
refresh(todos);
```

Use the thunk form for a read tree, or the refreshable derived primitive directly when supported.

A bare `refresh()` re-asks the same question, so it is quiet: `isPending` stays false and the new value appears without a pending state. When the reload should read as pending, such as a "Try again" button on a failed row, declare it first:

```ts
const retry = () => {
  affects(price);
  refresh(price);
};
// isPending(() => price()) is now true until the retry lands.
```

## `action(...)` and optimistic primitives

Mutations belong in `action(...)`, especially when the flow is:

- optimistic write
- async server work
- refresh source-of-truth reads

```ts
const [todos] = createStore(() => api.listTodos(), []);
const [optimisticTodos, setOptimisticTodos] = createOptimisticStore(() => todos(), []);

const addTodo = action(function* (text: string) {
  const optimistic = { id: crypto.randomUUID(), text, pending: true };
  setOptimisticTodos(list => list.push(optimistic));
  const saved = yield api.addTodo(text);
  refresh(todos);
  return saved;
});
```

Use optimistic primitives when immediate feedback helps the user experience, but keep the real source of truth explicit.

## Patterns that held up in a real app

These were verified on `solid-js@2.0.0-rc.20` in an SSR app (a domain search with an optimistic cart).

### A "saving" flag is an optimistic signal, not `isPending`

```ts
const [isSyncing, setSyncing] = createOptimistic(false);

const add = action(function* (item: Item) {
  setSyncing(true); // reverts to false on its own when the transition settles
  setView(draft => {
    draft.items.push(item);
  });
  const saved: Item[] = yield api.save(item);
  setCart(() => ({ items: saved }));
});
```

`createOptimisticStore(() => ({ items: source.items }), null, { key: "id" })` layers an optimistic copy over the source store; its writes drop away when the action settles. Pass `null` as the seed when the function returns the whole value, and a seed object when it mutates a draft.

### Share steps between actions with `yield*`

Actions are generators, so a shared save step can be its own generator and each action delegates to it with `yield* save(next)`.

### Rapid mutations need an intended value and a serialized queue

Two clicks in one frame both read the state from before either write, because reads lag writes. Responses from concurrent saves can also land out of order (and with cookies, the browser applies each `Set-Cookie` in arrival order, so an older cart can overwrite a newer one). Keep the intended value in a plain variable, update it synchronously, and send saves one at a time:

```ts
let intended = untrack(() => cart.items.map(item => item.id));
let queue: Promise<unknown> = Promise.resolve();

const save = function* (ids: string[]) {
  intended = ids;
  const request = queue.then(() => api.saveCart(intended)); // sends the latest intent
  queue = request.catch(() => {});
  const saved: Item[] = yield request;
  setCart(() => ({ items: saved }));
};
```

### `createProjection` for per-row selection

The replacement for `createSelector`: derive a keyed map so each row reads only its own key, and a toggle updates one row instead of every row.

```ts
const inCart = createProjection<Record<string, boolean>>(
  draft => {
    const ids = new Set(view.items.map(item => item.id));
    for (const id of Object.keys(draft)) if (!ids.has(id)) delete draft[id];
    for (const id of ids) draft[id] = true;
  },
  {},
);
// row: inCart[props.id] === true
```

### Held transitions: `isPending` to dim, `latest` for the control

When a change starts async work (choosing a sort that needs a bigger fetch), Solid holds that change's frame until the data lands, so the list stays put. Dim it with `isPending(() => list())`, and read the control's own value through `latest(() => filters.sort)` so the dropdown shows the new choice immediately instead of the held one.

### A value or a promise, whichever is ready

An async memo may return a plain value when the data is already at hand and a promise only when it is not. Returning the value avoids a suspension: no fallback flash in the browser, and no deferred boundary on the server (see `ssr-hydration-and-server-functions.md`). To keep a memo pending on purpose, for example while a search input is being debounced, return a promise that never settles.

```ts
const never = <T,>() => new Promise<T>(() => {});

const price = createMemo(() => {
  const cached = cache.get(key());
  if (cached) return cached;
  if (label() !== debouncedLabel()) return never<Price>();
  return fetchPrice(key());
});
```
