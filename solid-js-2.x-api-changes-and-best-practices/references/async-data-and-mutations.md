# Async Data And Mutations

Use this reference when the task is about the Solid 2.0 async model, pending states, refresh, or optimistic updates.

The source of truth for this file is the migration guide plus the async-data and actions RFCs.

## Core model

- async reads move into computations instead of a dedicated `createResource` primitive
- `Loading` handles initial readiness
- `isPending(() => expr)` describes stale-while-revalidating work, not first load
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
