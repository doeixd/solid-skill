---
name: solid-js-2.x-api-changes-and-best-practices
description: Guidance for writing, reviewing, and explaining SolidJS 2.x beta or next-era code, including new APIs, changed semantics, and idiomatic patterns. Use this whenever the user is targeting Solid 2.x beta or next, asks how a 2.x API works, wants new code written in the 2.x style, or needs help understanding warnings about top-level reactive reads, writes in owned scope, batching, Loading, Errored, new store APIs, or changed DOM and control-flow behavior. Use this even if the user does not explicitly say beta or next when the code clearly targets Solid 2.x APIs.
---

# SolidJS 2.x API Changes And Best Practices

Use this skill for Solid 2.x beta or `next` work. Treat the current beta migration guide and RFCs as the source of truth, and be explicit that some APIs or package boundaries may still shift before final stable release.

## Goals

- Help the user write correct Solid 2.x code, not just mechanically renamed 1.x code.
- Explain changed semantics, especially where old 1.x intuitions are now wrong.
- Surface likely ecosystem lag or package-version caveats instead of pretending everything is already settled.

## First step

Determine whether the user wants:

- fresh 2.x code
- explanation of a 2.x API or warning
- review of existing 2.x code
- comparison between 1.x and 2.x behavior

If the user is actually upgrading old code, prefer the dedicated migration skill unless they only want a narrow API explanation.

## How to use this skill

Read this skill in passes instead of treating every section as equally important.

1. Start with `Important framing`, `Primitive selection guide`, and `Core behavior changes to internalize`.
2. Use `Core API guide` for fresh code, reviews, and narrow API questions.
3. Use `Reference guide` when the task is concentrated in one area such as async data, ownership, stores, or DOM behavior.
4. Use `Task-focused loading` and `Common mistakes to catch` for code review, warning triage, and implementation guidance.

## Important framing

- Target is Solid 2.x beta or `next`.
- If the local project dependencies disagree with the skill's guidance, trust the installed versions and package exports in the repo.
- Prefer explaining the new mental model, because many 2.x mistakes come from carrying over 1.x habits.
- For code generation, prefer the exact primitives and package boundaries described in this skill and the official Solid 2.0 docs. Do not substitute neighboring framework helpers or unofficial aliases just because they sound plausible.
- In particular, do not introduce `createAsync`, `mount`, router helpers, or Start-specific patterns unless the user is explicitly in that ecosystem and the local repo confirms those exports.

## Scope boundary

- Use this skill for fresh 2.x code, narrow 2.x API explanations, 2.x warning triage, and review of code that already targets the new semantics.
- If the user is upgrading an existing 1.x app or library, prefer the dedicated migration skill unless they only need a narrow explanation of one changed API.
- Do not answer Start-specific route, query, action, middleware, or deployment questions from generic Solid assumptions. Use the SolidStart skill alongside this one when the app is a Start app.

## Reference guide

Keep the main body of this skill as the working guide. Read the deeper references when the task is concentrated in one area.

- for batching, split effects, top-level read warnings, `onSettled`, or ownership: read `references/reactivity-effects-and-ownership.md`
- for `Loading`, `isPending`, `latest`, `refresh`, `action`, or optimistic UI: read `references/async-data-and-mutations.md`
- for `createStore`, `createProjection`, `storePath`, `snapshot`, `merge`, or `omit`: read `references/stores-and-helpers.md`
- for `For`, `Repeat`, function-child accessors, DOM attributes, `ref` directive factories, or context providers: read `references/control-flow-dom-and-context.md`

## Primitive selection guide

Choose primitives with this bias:

1. `createSignal` for local scalar state
2. `createMemo` for derivation, including async derivation when it matches the problem
3. `createStore` for nested state
4. `Loading` and `Errored` for async and error boundaries
5. `isPending`, `latest`, and `refresh` for async coordination
6. `action` plus optimistic helpers for mutations
7. `createProjection` or function-form `createStore` for projection-oriented derived collections and keyed reconciliation
8. `createEffect` only for bridging to the outside world
9. `For`, `Show`, `Switch`, `Match`, and `Repeat` for control flow
10. `ref` factories for imperative DOM hooks

## Core API guide

Keep this dense and practical. Bias toward `use it for`, `watch out for`, and the most important 2.x-specific behavior.

### `createSignal`

- Use it for core local state.
- In 2.x, writes are batched until flush, so immediate reads do not observe the new value.
- Function form can express derived-but-writable state when that shape is truly intended:

```ts
const [value, setValue] = createSignal(() => props.something);
```

For readonly derivation, prefer `createMemo`.

### `createMemo`

- Use it for derivation, including many async derivations in 2.x.
- Tracked reads must happen before `await`.
- For Solid core 2.x examples, prefer an explicitly async computation shape instead of drifting into neighboring helpers like `createAsync`.

```ts
const profile = createMemo(async () => {
  const id = userId();
  return fetchProfile(id);
});
```

- Consumers still read the accessor normally; unresolved async values suspend through the graph until a `Loading` boundary handles them.

Track before `await`:

```ts
const filtered = createMemo(async () => {
  const q = query();
  await warmCache(q);
  return search(q);
});
```

Do not read `query()` for the first time after an `await`.

### `createEffect`

- Use it as a split-phase effect: compute first, apply second.
- Old 1.x single-callback intuition often produces the wrong shape or warning-prone code.
- Drive this distinction hard:

- compute is for reading reactive inputs and deciding what changed
- apply is for mutating the outside world
- cleanup belongs to apply
- if you are only deriving data, you probably want `createMemo`, not `createEffect`

Example:

```ts
createEffect(
  () => [title(), enabled()],
  ([nextTitle, nextEnabled]) => {
    document.title = nextTitle;
    widget.setEnabled(nextEnabled);
    return () => widget.dispose();
  }
);
```

### `onSettled`

- Use it instead of `onMount`.
- It runs once current activity settles and can return cleanup.
- Treat `onMount` migration as a semantic review, not a blind rename.

### `Loading`

- Use it as the async readiness boundary.
- It is primarily for initial readiness, not every later refresh.
- Pair it with an async computation from `createMemo(async () => ...)` or another clearly derived 2.x primitive. Do not answer with old `Suspense` code unless the user asked for comparison or migration context.

### `Errored`

- Use it as the 2.x error boundary primitive.
- It can render a fallback callback with the error and reset function.
- Pair it with the 2.x async and effect model instead of carrying forward old boundary assumptions.

### `isPending`, `latest`, and `refresh`

- Use them as async coordination helpers.
- `isPending(() => expr)` is for refresh or stale-while-revalidate states, not first load.
- `latest(fn)` lets you inspect the most recent in-flight value.
- `refresh(...)` recomputes derived reads after writes.

Example:

```ts
const users = createMemo(async () => fetchUsers(filter()));
const usersPending = () => isPending(() => users());
const latestUsers = () => latest(users);
```

### `action`, `createOptimistic`, `createOptimisticStore`

- Use `action(...)` for mutation flows.
- Actions run inside the transition model and are the intended place to sequence optimistic writes, async work, and refreshes.
- Use optimistic primitives when immediate UI feedback matters and the real source of truth still needs refresh afterward.
- Prefer this over inventing extra pending or mirror state by hand.

Example:

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

Keep the real source of truth explicit and refresh it after the mutation settles.

### `createProjection`

- Use it when the derived result is better modeled as a store-shaped projection than a plain accessor.
- It fits projection, keyed reconciliation, and selection-like patterns that 1.x users may have modeled with selectors or ad-hoc effects.
- If it returns list or map data, unchanged keyed entries can keep identity via reconciliation.

### `createStore`

- Use it for nested state, now from `solid-js`.
- Prefer draft-first setters in fresh 2.x code.
- Function-form `createStore(fn, initial)` still exists for derived-store patterns when that shape is the right fit.
- `snapshot` replaces `unwrap`, and returning a value from a setter performs a top-level shallow replacement or diff.
- Old path-style habits still exist, but they are no longer the center of the API.

Preferred update shape:

```ts
setStore(s => {
  s.user.address.city = "Paris";
});
```

### `storePath`

Compatibility helper for path-style store updates. Useful during migration, but not the default style for fresh 2.x code.

Example:

```ts
setStore(storePath("user", "address", "city", "Paris"));
```

### `merge` and `omit`

General helpers replacing `mergeProps` and `splitProps`. `merge` treats `undefined` as an explicit value, not as "missing". `omit` is the preferred way to exclude keys without split-style copying.

Example:

```ts
const merged = merge({ a: 1, b: 2 }, { b: undefined });
```

### `snapshot` and `deep`

Use `snapshot(store)` for serialization or interop. Use `deep(store)` only when deep observation is truly intended.

Example:

```ts
const plain = snapshot(store);
```

### `For`, `Show`, `Switch`, `Match`, `Repeat`

- Use them as control-flow primitives with more explicit accessor semantics.
- Function children often receive accessors.
- `For` replaces `Index` via `keyed={false}`.
- `Repeat` handles count or range rendering without list diffing.

### `createContext`

Use the context object itself as the provider component.

Example:

```tsx
const ThemeContext = createContext("light");

<ThemeContext value="dark">
  <Page />
</ThemeContext>
```

Prefer this over carrying forward `Context.Provider`.

### Ownership and `createRoot`

- A `createRoot(...)` created inside an owned scope is owned by its parent by default.
- Disposal follows the parent unless you detach explicitly.
- This reduces accidental unowned graphs and cleanup surprises.

If the user genuinely needs detached lifetime, make that explicit:

```ts
const singleton = runWithOwner(null, () => {
  const [value, setValue] = createSignal(0);
  return { value, setValue };
});
```

### `createComputed` removal

- `createComputed` is removed.
- Replace it with split `createEffect` for side effects, function-form `createSignal` or `createStore` for derived state with a setter, or `createMemo` for readonly derivation.
- 2.x wants derived state, effects, and ownership to be more explicit and easier to reason about.

### refs and directive factories

Use `ref` factories for imperative DOM hooks and directive-like behavior. Do not translate `use:` literally.

## Core behavior changes to internalize

### Batching is microtask-based

- Setters do not immediately change what reads return.
- Updated values become visible after the microtask flush.
- Use `flush()` only when a synchronous settled point is truly needed, especially in tests or imperative DOM code.

This interacts directly with async derivations and split effects. Do not reason about 2.x as though setters synchronously rewire every downstream read.

### Effects are split into compute and apply phases

Prefer the 2.x shape:

```ts
createEffect(
  () => source(),
  value => {
    sideEffect(value);
    return () => cleanup();
  }
);
```

The compute function tracks dependencies. The apply function performs effects and can return cleanup.

Use this bias:

- compute reads
- apply mutates the outside world
- cleanup belongs with apply

Be unusually strict about this. If the user asks about `createEffect`, explain the split even if the immediate bug looks small, because this is one of the main 2.x mental-model changes.

### Top-level reactive reads warn

In component bodies, avoid reading reactive values at top level unless the read is intentionally wrapped in `untrack`. This includes common patterns like destructuring reactive props.

Prefer reading inside JSX expressions, memos, or effect compute functions.

### Writes inside owned reactive scope warn

Do not use memos or tracked scopes to write application state back into signals or stores. Prefer:

- derived state with `createMemo`
- event handlers
- actions
- narrow `pureWrite: true` only for valid internal cases

Do not recommend `pureWrite: true` as a generic warning silencer.

## Async derivation patterns

Solid 2.x can derive async values directly. Show that this is possible, but do not force it as the one true answer for every async problem.

Use this guidance:

- for async derived reads, `createMemo(async () => ...)` is one viable 2.x pattern
- for keyed or collection-oriented derived state, `createProjection(...)` or function-form `createStore(...)` may fit better than a plain memo
- for initial readiness, pair async values with `Loading`
- for stale-while-refreshing UI, use `isPending` and optionally `latest`
- for user-facing mutations where immediate feedback matters, prefer `action` plus optimistic primitives over hand-rolled pending flags

Important distinctions:

- unresolved async reads suspend through the graph until a `Loading` boundary handles them
- `Loading` is for first readiness
- `isPending` is for background revalidation when usable UI already exists
- async reads and async mutations are different jobs; computations handle reads, actions handle mutations

### Async memo example

```tsx
const profile = createMemo(async () => {
  const currentId = userId();
  return fetchProfile(currentId);
});

<Loading fallback={<Spinner />}>
  <ProfileView profile={profile()} />
</Loading>
```

The accessor is still read normally. The async part is expressed by suspension and the boundary, not by adding a separate resource API.

### Combined example: stream, projection, optimistic update, pending state

```ts
const [feed] = createStore(async () => {
  const items: Message[] = [];

  for await (const chunk of getMessageStream(roomId())) {
    items.push(...chunk);
  }

  return items;
}, [], { key: "id" });

const [projectedFeed] = createStore(
  draft => {
    const visible = feed().filter(m => !m.hidden);
    draft.items = visible;
    draft.unread = visible.filter(m => !m.read).length;
  },
  { items: [], unread: 0 }
);

const [optimisticFeed, setOptimisticFeed] = createOptimisticStore(() => projectedFeed(), {
  items: [],
  unread: 0
});

const sendMessage = action(function* (text: string) {
  const optimistic = { id: crypto.randomUUID(), text, pending: true };
  setOptimisticFeed(s => {
    s.items.unshift(optimistic);
    s.unread += 1;
  });
  yield api.sendMessage(roomId(), text);
  refresh(feed);
  refresh(projectedFeed);
});

const feedRefreshing = () => isPending(() => projectedFeed().items);
```

Use this kind of shape when:

- an async source feeds a projection-backed collection
- another projection derives the UI-facing slice
- optimistic updates should appear immediately
- `isPending` should describe refresh state without replacing the current UI

## API changes and replacements

### Quick rename and removal map

- `solid-js/web` -> `@solidjs/web`
- `solid-js/store` -> `solid-js`
- `solid-js/h` -> `@solidjs/h`
- `solid-js/html` -> `@solidjs/html`
- `solid-js/universal` -> `@solidjs/universal`
- `Suspense` -> `Loading`
- `ErrorBoundary` -> `Errored`
- `mergeProps` -> `merge`
- `splitProps` -> `omit`
- `createSelector` -> `createProjection` or function-form `createStore`
- `unwrap` -> `snapshot`
- `classList` -> `class`
- `createResource` removed -> async computations plus `Loading`
- `startTransition` and `useTransition` removed -> built-in transitions, `isPending`, `Loading`, and optimistic primitives
- `batch` removed -> `flush()` when immediate application is truly needed
- `Context.Provider` removed -> use the context directly as provider component
- `createComputed` removed -> split `createEffect`, function-form `createSignal` or `createStore`, or `createMemo`
- `use:` directives removed -> `ref` directive factories
- `attr:` and `bool:` removed -> standard attribute behavior
- `oncapture:` removed
- `onMount` -> `onSettled`

### Imports

Use the quick map above for package moves. The main practical change is that DOM or renderer subpaths often move to `@solidjs/*`, while store APIs move into `solid-js`.

### Async and boundaries

- use `isPending(() => expr)` for stale-while-refreshing indicators
- use `latest(fn)` to inspect in-flight values when needed
- use `refresh(...)` to recompute reads after mutations

The rename and removal map above covers the high-level swaps. This section is about how to use the surviving async pieces together.

### Control flow

- `Index` is removed; use `<For keyed={false}>`
- `For` child functions receive accessors, so call `item()` and `i()`
- function children in `Show` and related APIs may also receive accessors
- `Repeat` exists for count or range-based rendering

### Stores and helpers

- prefer draft-first `setStore(draft => { ... })`
- `createSelector` patterns generally move to `createProjection` or function-form `createStore`
- `storePath(...)` exists as a compatibility helper when path-style ergonomics are still useful
- `unwrap(store)` -> `snapshot(store)`
- `mergeProps` -> `merge`
- `splitProps` -> `omit`
- `deep(store)` is available when deep observation is truly needed
- `createSignal(fn)` and `createStore(fn, initial)` support derived forms

### DOM and directives

- `use:` directives are removed; use `ref` directive factories
- `classList` moves to `class` object or array forms
- `attr:` and `bool:` namespaces are removed
- `oncapture:` is removed
- built-in attributes are closer to raw HTML semantics

### Context

- `Context.Provider` becomes using the context directly as the provider component

## Best-practice guidance

When writing or reviewing 2.x code:

- prefer derivation over write-back synchronization
- preserve reactive reads inside valid tracking sites
- keep side effects in effect apply functions, not the tracking half
- treat dev warnings as useful signals, not noise
- use `flush()` sparingly and usually only after you can explain why it is needed
- prefer `Loading` for initial readiness and `isPending` for refresh state instead of inventing extra flags
- prefer draft-first store updates in fresh 2.x code
- prefer `ref` factories over trying to recreate `use:` mentally
- for mutations with immediate UI feedback, reach for optimistic primitives before inventing custom mirror state
- keep read concerns in computations and write concerns in actions
- keep ownership explicit; detached lifetime should be rare and deliberate
- prefer the context object as provider instead of carrying forward `Context.Provider` boilerplate
- avoid top-level reactive reads unless the non-tracking behavior is intentional and explicit
- avoid writing application state from owned reactive scopes
- keep examples on the Solid core boundary unless the user asked for SolidStart, router, or another framework layer
- if an exact beta import or export is uncertain, say so explicitly instead of inventing a helper
- do not replace `render` with unrelated mount APIs unless the local repo or installed exports show that is the right move

## Task-focused loading

### Writing fresh 2.x code

Load these ideas first:

- pick primitives from the `Primitive selection guide`
- keep derivation in `createMemo`
- use `Loading` and `Errored` for boundaries
- use `action` plus optimistic helpers for mutations
- stay on the Solid core boundary unless the task explicitly moves into Start or router APIs

### Explaining warnings or behavior changes

Load these ideas first:

- microtask batching and `flush()`
- split `createEffect`
- top-level reactive read warnings
- writes inside owned reactive scope warnings
- ownership and `createRoot`

Common warning-to-cause mapping:

- stale immediate reads after setters usually mean the user is still assuming 1.x-style synchronous visibility
- top-level read warnings usually mean props or signals were read in setup instead of JSX or another tracked scope
- owned-scope write warnings usually mean derivation was modeled as write-back synchronization instead of derived state or an action

### Reviewing existing 2.x code

Look for these first:

- derivation done through effects instead of `createMemo`
- top-level reactive reads or destructured reactive props
- writes to signals or stores from owned tracked scopes
- fresh-code examples using migration-only or compatibility helpers by default
- drift into Start or router helpers when the task is supposed to stay on Solid core
- nested `createRoot(...)` calls that assume detached lifetime without making that explicit
- control-flow callbacks that forget accessor semantics such as `item()` or `i()`

### Async data and mutation design

Load these ideas first:

- `createMemo(async () => ...)` for reads
- `Loading` for first readiness
- `isPending` for revalidation state
- `latest` when inspecting current in-flight value matters
- `action` and optimistic primitives for writes
- `refresh(...)` after mutations

Keep the job split explicit:

- computations handle async reads
- actions handle writes and mutation sequencing
- `refresh(...)` recomputes reads after writes; it is not just a renamed `refetch` habit

### Store and projection work

Load these ideas first:

- draft-first `createStore` setters
- `createProjection` for store-shaped derived results
- function-form `createStore` when derived store state fits better there
- `storePath` only as a compatibility helper, not the fresh-code default
- `snapshot`, `merge`, and `omit` instead of old 1.x helpers

### DOM and directive work

Load these ideas first:

- `ref` directive factories instead of `use:`
- standard HTML-like attribute behavior
- `class` object or array forms instead of `classList`
- accessor-aware control-flow callbacks

## Common mistakes to catch

| Mistake | Better direction |
| --- | --- |
| Assuming a setter read updates immediately | Explain microtask batching and use `flush()` only when a settled point is truly needed. |
| Using `createEffect` for derivation | Move derivation to `createMemo` or another derived form. |
| Reading reactive props at the top level of a component | Read them inside JSX, memos, or effect compute functions. |
| Writing app state from memos or tracked scopes | Move writes to actions, event handlers, or explicit writable derived-state patterns. |
| Using `pureWrite: true` as a generic warning silencer | Reserve it for narrow internal cases such as refs or internal bookkeeping; fix the state flow instead. |
| Using `Suspense` in fresh 2.x core examples | Use `Loading` unless the user asked for migration comparison. |
| Treating async reads and async mutations as the same kind of problem | Use computations for reads and `action(...)` for writes. |
| Reaching for `createAsync`, router helpers, or Start helpers in core Solid prompts | Stay on Solid core primitives unless the task explicitly moves into that ecosystem. |
| Treating `storePath` as the fresh-code default | Prefer draft-first store setters in new 2.x code. |
| Forgetting accessor semantics in control-flow callbacks | In `For keyed={false}` and similar callback sites, read values with `item()` and `i()` when accessors are provided. |
| Translating `use:` directives literally | Use `ref` directive factories. |
| Treating `isPending` like first-load state | Use `Loading` for first readiness and `isPending` for refresh or stale-while-revalidate state. |
| Using `Context.Provider` out of habit | Use the context object directly as the provider component. |
| Assuming nested `createRoot(...)` is detached by default | Explain that nested roots are owned by the parent unless detached explicitly with `runWithOwner(null, ...)`. |
| Replacing `render` with a plausible-sounding mount helper | Only move imports or APIs when the local repo or official 2.x docs confirm it. |

## Output expectations

When answering, prefer this structure:

1. Name the 2.x concept or behavior change.
2. Explain how it differs from 1.x.
3. Show the 2.x-idiomatic code.
4. Call out beta or ecosystem caveats if relevant.

## Example prompts this skill should handle well

- "Write this component using Solid 2.0 beta APIs."
- "Why am I getting a top-level reactive read warning in Solid 2?"
- "What replaced createResource and Suspense in Solid 2.x?"
- "Review this Solid 2 code and point out places where I am still thinking in 1.x terms."

## Additional resources

- Solid 2.0 docs index: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/README.md
- Reactivity, batching, and effects RFC: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/01-reactivity-batching-effects.md
- Signals and ownership RFC: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/02-signals-derived-ownership.md
- Control flow RFC: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/03-control-flow.md
- Stores RFC: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/04-stores.md
- Async data RFC: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/05-async-data.md
- Actions and optimistic RFC: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/06-actions-optimistic.md
- DOM RFC: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/07-dom.md
