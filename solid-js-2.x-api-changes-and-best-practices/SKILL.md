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

## API surface map

Keep this dense and practical. Explain each primitive in terms of what it is, how it behaves in 2.x, and why it exists.

### `createSignal`

What it is:

- the core local state primitive

How it behaves in 2.x:

- writes are batched until flush
- function-form signals can represent derived writable shapes in some cases

Why it matters:

- many migration misunderstandings start with assuming reads update immediately after writes

Function form can express derived-but-writable state:

```ts
const [value, setValue] = createSignal(() => props.something);
```

Use that for writable derived state. For readonly derivation, prefer `createMemo`.

### `createMemo`

What it is:

- derived computation, including async computation patterns in 2.x

How it behaves in 2.x:

- central to deriving values without write-back effects
- tracked reads must happen before `await`
- can represent async derivations that would previously have pushed people toward `createResource` or framework helpers like `createAsync`
- consumers still read the accessor normally; unresolved async values suspend through the graph until a `Loading` boundary handles them

Why it matters:

- it is part of the 2.x async story and worth recognizing when reading or writing new code
- it removes the need for a separate resource-shaped read primitive in many cases

For new Solid core examples, prefer an explicitly async computation shape:

```ts
const profile = createMemo(async () => {
  const id = userId();
  return fetchProfile(id);
});
```

Do not quietly swap in `createAsync` from framework-specific docs or older examples when the task is about Solid core 2.x.

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

What it is:

- a split-phase effect: compute first, apply second

How it behaves in 2.x:

- compute tracks dependencies
- apply runs side effects and can return cleanup

Why it matters:

- old 1.x single-callback intuition often produces the wrong shape or warning-prone code

Drive this distinction hard:

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

What it is:

- the lifecycle primitive replacing `onMount`

How it behaves in 2.x:

- runs once current activity settles
- can return cleanup

Why it matters:

- mount timing and cleanup habits changed, and many examples need a semantic rewrite rather than a rename

### `Loading`

What it is:

- the async readiness boundary

How it behaves in 2.x:

- shows fallback while required async values are not ready
- is primarily for initial readiness, not for every later refresh

Why it matters:

- it replaces `Suspense` in the new async model
- it keeps loading state in UI structure instead of leaking `T | undefined` holes through code

For new 2.x examples, pair `Loading` with an async computation from `createMemo(async () => ...)` or another clearly derived 2.x primitive. Do not answer with old `Suspense` code unless the user asked for comparison or migration context.

### `Errored`

What it is:

- the error boundary primitive

How it behaves in 2.x:

- can render a fallback callback with the error and reset function

Why it matters:

- pairs with the 2.x async and effect model more directly than old boundary assumptions

### `isPending`, `latest`, and `refresh`

What they are:

- async coordination helpers

How they behave in 2.x:

- `isPending(() => expr)` signals refresh or stale-while-revalidate states
- `latest(fn)` lets you inspect the most recent in-flight value
- `refresh(...)` recomputes derived reads after writes

Why they matter:

- they replace a lot of custom "loading flag plus refetch" code
- they separate initial loading from later revalidation and mutation follow-up

Example:

```ts
const users = createMemo(async () => fetchUsers(filter()));
const usersPending = () => isPending(() => users());
const latestUsers = () => latest(users);
```

### `action`, `createOptimistic`, `createOptimisticStore`

What they are:

- mutation and optimistic UI primitives

How they behave in 2.x:

- `action(...)` expresses a mutation flow
- optimistic primitives layer temporary UI state over source-of-truth data
- actions run inside the transition model and are the intended place to sequence optimistic writes, async work, and refreshes

Why they matter:

- 2.x encourages first-class mutation flows instead of ad-hoc mutation helpers
- in many app-level flows, optimistic primitives are a better fit than inventing extra pending or mirror state by hand

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

Use optimistic primitives when the user experience benefits from showing the expected result immediately, but keep the real source of truth explicit and refresh it after the mutation settles.

### `createProjection`

What it is:

- a mutable derived store for projection, keyed reconciliation, and selection-like patterns

How it behaves in 2.x:

- derives a store shape from reactive inputs
- can mutate a draft or return reconciled list data
- pairs well with refreshable derived data and keyed collections
- if it returns list or map data, unchanged keyed entries can keep identity via reconciliation

Why it matters:

- it is the more explicit projection primitive behind several derived-store patterns that 1.x users may have modeled with selectors or ad-hoc effects

Use it when the derived result is better modeled as a store-shaped projection than a plain accessor.

### `createStore`

What it is:

- the store primitive, now exposed from `solid-js`

How it behaves in 2.x:

- draft-first setters are the preferred style
- derived-store forms exist via `createStore(fn, initial)`
- `snapshot` replaces `unwrap`
- returning a value from a setter performs a shallow replacement or diff for the top-level object or array

Why it matters:

- old path-style habits still exist, but they are no longer the center of the API
- some projection-like work can be expressed either with `createProjection` or function-form `createStore`

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

Helpers for plain-value extraction and deep observation. Use `snapshot(store)` for serialization or interop. Use `deep(store)` only when deep observation is truly intended.

Example:

```ts
const plain = snapshot(store);
```

### `For`, `Show`, `Switch`, `Match`, `Repeat`

What they are:

- control-flow primitives with more explicit accessor semantics

How they behave in 2.x:

- function children often receive accessors
- `For` replaces `Index` via `keyed={false}`
- `Repeat` handles count or range rendering without list diffing

Why they matter:

- function child semantics are a common place where 1.x instincts create stale reads or wrong assumptions

### `createContext`

Context primitive with simplified provider ergonomics. The context object itself is the provider component.

Example:

```tsx
const ThemeContext = createContext("light");

<ThemeContext value="dark">
  <Page />
</ThemeContext>
```

Prefer this over reaching for `Context.Provider`.

### Ownership and `createRoot`

What they are:

- runtime lifetime rules for reactive graphs

How they behave in 2.x:

- a `createRoot(...)` created inside an owned scope is owned by its parent by default
- disposal therefore follows the parent unless you detach explicitly

Why they matter:

- fewer accidental unowned graphs and fewer cleanup surprises

If the user genuinely needs detached lifetime, make that explicit:

```ts
const singleton = runWithOwner(null, () => {
  const [value, setValue] = createSignal(0);
  return { value, setValue };
});
```

### `createComputed` removal

What changed:

- `createComputed` is removed

How to replace it:

- use split `createEffect` when the job is side effects
- use function-form `createSignal` or `createStore` when the job is derived state with a setter
- use `createMemo` when the job is readonly derivation

Why it matters:

- 2.x wants derived state, effects, and ownership to be more explicit and easier to reason about

### refs and directive factories

Imperative DOM hooks and the replacement for `use:` directives. Express directive-like behavior through `ref` factories instead of translating `use:` literally.

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
