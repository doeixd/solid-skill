https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/MIGRATION.md

# Solid 2.0 (beta) — quick migration guide

This is a short, practical guide for migrating from Solid 1.x to Solid 2.0’s APIs. It focuses on the changes you’ll hit most often and shows “before/after” examples.

## Quick checklist (start here)

- **Imports**: some 1.x subpath imports moved to `@solidjs/*` packages (and store helpers moved into `solid-js`).
- **Batching/reads**: setters don’t immediately change what reads return; values become visible after the microtask batch flushes (or via `flush()`).
- **Effects**: `createEffect` is split (compute → apply). Cleanup is usually “return a cleanup function”.
- **Lifecycle**: `onMount` is replaced by `onSettled` (and it can return cleanup).
- **Async UI**: use `<Loading>` for first readiness; use `isPending(() => expr)` for “refreshing…” indicators.
- **Lists**: `Index` is gone; use `<For keyed={false}>`. `For` children receive accessors (`item()`/`i()`).
- **Stores**: prefer draft-first setters; `storePath(...)` exists as an opt-in helper for the old path-style ergonomics.
- **Plain values**: `snapshot(store)` replaces `unwrap(store)` when you need a plain non-reactive value.
- **DOM**: `use:` directives are removed; use `ref` directive factories (and array refs).
- **Helpers**: `mergeProps` → `merge`, `splitProps` → `omit`.

## Core behavior changes

### Imports: where things live now

In Solid 2.0 beta, the DOM/web runtime is its own package, and some “subpath imports” from 1.x are gone.

```ts
// 1.x (DOM runtime)
import { render, hydrate } from "solid-js/web";

// 2.0 beta
import { render, hydrate } from "@solidjs/web";
```

```ts
// 1.x (stores)
import { createStore } from "solid-js/store";

// 2.0 beta (stores are exported from solid-js)
import { createStore, reconcile, snapshot, storePath } from "solid-js";
```

```ts
// 1.x (hyperscript / alternate JSX factory)
import h from "solid-js/h";

// 2.0 beta
import h from "@solidjs/h";
```

```ts
// 1.x (tagged-template HTML)
import html from "solid-js/html";

// 2.0 beta
import html from "@solidjs/html";
```

```ts
// 1.x (custom renderers)
import { createRenderer } from "solid-js/universal";

// 2.0 beta
import { createRenderer } from "@solidjs/universal";
```

### Batching & reads: values update after flush

In Solid 2.0, updates are batched by default (microtasks). A key behavioral change is that **setters don’t immediately update what reads return** — the new value becomes visible when the batch is flushed (next microtask), or immediately if you call `flush()`.

```js
const [count, setCount] = createSignal(0);

setCount(1);
count(); // still 0

flush();
count(); // now 1
```

Use `flush()` sparingly (it forces the system to “catch up now”). It’s most useful in tests, or in rare imperative code where you truly need a synchronous “settled now” point.

### Effects, lifecycle, and cleanup

Solid 2.0 splits effects into two phases:

- a **compute** function that runs in the reactive tracking phase and returns a value
- an **apply** function that receives that value and performs side effects (and can return cleanup)

```js
// 1.x (single function effect)
createEffect(() => {
  el().title = name();
});

// 2.0 (split effect: compute -> apply)
createEffect(
  () => name(),
  value => {
    el().title = value;
  }
);
```

Cleanup usually lives on the apply side now:

```js
// 1.x
createEffect(() => {
  const id = setInterval(() => console.log(name()), 1000);
  onCleanup(() => clearInterval(id));
});

// 2.0
createEffect(
  () => name(),
  value => {
    const id = setInterval(() => console.log(value), 1000);
    return () => clearInterval(id);
  }
);
```

If you used `onMount`, the closest replacement is `onSettled` (and it can also return cleanup):

```js
// 1.x
onMount(() => {
  measureLayout();
});

// 2.0
onSettled(() => {
  measureLayout();
  const onResize = () => measureLayout();
  window.addEventListener("resize", onResize);
  return () => window.removeEventListener("resize", onResize);
});
```

### Dev warnings you’ll likely see (and how to fix them)

These are **dev-only warnings** meant to catch subtle bugs earlier.

#### “Top-level reactive read” in a component

In 2.0, reading reactive values at the top level of a component body (including destructuring props) will warn. The fix is usually to move the read into a reactive scope (`createMemo`/`createEffect`) or make the intent explicit with `untrack`.

```jsx
// ❌ 2.0 warns (top-level reactive read)
function Bad(props) {
  const n = props.count;
  return <div>{n}</div>;
}

// ✅ read inside JSX/expression
function Ok(props) {
  return <div>{props.count}</div>;
}
```

```jsx
// ❌ 2.0 warns (common: destructuring in args)
function BadArgs({ title }) {
  return <h1>{title}</h1>;
}

// ✅ keep props object, or destructure inside a memo/effect
function OkArgs(props) {
  return <h1>{props.title}</h1>;
}
```

#### “Write inside reactive scope” (owned scope)

Writing to signals/stores inside a reactive scope will warn. Usually you want:

- derive values with `createMemo` (no write-back)
- write in event handlers / actions
- return cleanup from effect apply functions (instead of writing during tracking)

```js
// ❌ warns: writing from inside a memo
createMemo(() => setDoubled(count() * 2));

// ✅ derive instead of writing back
const doubled = createMemo(() => count() * 2);
```

If you truly have an **internal** signal that needs to be written from within owned scope (not app state), opt in narrowly with `pureWrite: true`.

## Async data & transitions

### `Suspense` / `ErrorBoundary` → `Loading` / `Errored`

```jsx
// 1.x
<Suspense fallback={<Spinner />}>
  <Profile />
</Suspense>

// 2.0
<Loading fallback={<Spinner />}>
  <Profile />
</Loading>
```

### `createResource` → async computations + `Loading`

```js
// 1.x
const [user] = createResource(id, fetchUser);

// 2.0
const user = createMemo(() => fetchUser(id()));
```

```jsx
<Loading fallback={<Spinner />}>
  <Profile user={user()} />
</Loading>
```

### Initial loading vs revalidation: `Loading` vs `isPending`

- **`Loading`**: initial “not ready yet” UI boundary.
- **`isPending`**: “stale while revalidating” indicator; **false during the initial `Loading` fallback**.

```jsx
const listPending = () => isPending(() => users() || posts());

<>
  <Show when={listPending()}>{/* subtle "refreshing…" indicator */}</Show>
  <Loading fallback={<Spinner />}>
    <List users={users()} posts={posts()} />
  </Loading>
</>
```

### Peeking in-flight values: `latest(fn)`

```js
const latestId = () => latest(id);
```

### “Refetch/refresh” patterns → `refresh()`

```js
// After a server write, explicitly recompute a derived read:
refresh(storeOrProjection);

// Or re-run a read tree:
refresh(() => query.user(id()));
```

### Mutations: `action(...)` + optimistic helpers

In 1.x, mutations often ended up as “call an async function, flip some flags, then manually refetch”. In 2.0, the recommended shape is:

- wrap mutations in `action(...)`
- use `createOptimistic` / `createOptimisticStore` for optimistic UI
- call `refresh(...)` at the end to recompute derived reads

```js
const [todos] = createStore(() => api.getTodos(), { list: [] });
const [optimisticTodos, setOptimisticTodos] = createOptimisticStore({ list: [] });

const addTodo = action(function* (todo) {
  // optimistic UI
  setOptimisticTodos(s => s.list.push(todo));

  // server write
  yield api.addTodo(todo);

  // recompute reads derived from the source-of-truth
  refresh(todos);
});
```

## Stores

### Draft-first setters (and `storePath` as an opt-in helper)

```js
// 2.0 preferred: produce-style draft updates
setStore(s => {
  s.user.address.city = "Paris";
});

// Optional compatibility: old “path argument” ergonomics via storePath
setStore(storePath("user", "address", "city", "Paris"));
```

### `unwrap(store)` → `snapshot(store)`

```js
const plain = snapshot(store);
JSON.stringify(plain);
```

### `mergeProps` / `splitProps` → `merge` / `omit`

```js
// 1.x
const merged = mergeProps(defaults, overrides);

// 2.0
const merged = merge(defaults, overrides);
```

One behavioral gotcha: **`undefined` is treated as a real value** (it overrides), not “skip this key”.

```js
const merged = merge({ a: 1, b: 2 }, { b: undefined });
// merged.b is undefined
```

```js
// 1.x
const [local, rest] = splitProps(props, ["class", "style"]);

// 2.0
const rest = omit(props, "class", "style");
```

### New function forms: `createSignal(fn)` and `createStore(fn)`

`createSignal(fn)` creates a **writable derived signal** (think “writable memo”):

```js
const [count, setCount] = createSignal(0);
const [doubled] = createSignal(() => count() * 2);
```

`createStore(fn, initial)` creates a **derived store** using the familiar `createStore` API:

```js
const [items] = createStore(() => api.listItems(), []);

const [cache] = createStore(
  draft => {
    draft.total = items().length;
  },
  { total: 0 }
);
```

## Control flow

### List rendering: `Index` is gone, and `For` children use accessors

If you used `Index`, it’s now `For` with `keyed={false}`.

The breaking bit: the `For` child function receives **accessors** for both the item and the index, so you’ll write `item()` / `i()` (not `item` / `i`).

```jsx
// 1.x
<Index each={items()}>
  {(item, i) => <Row item={item()} index={i} />}
</Index>

// 2.0
<For each={items()} keyed={false}>
  {(item, i) => <Row item={item()} index={i()} />}
</For>
```

### Function children often receive accessors (call them!)

This isn’t just `For`. A few control-flow APIs pass **accessors** into function children so the value is always safe to read:

```jsx
<Show when={user()} fallback={<Login />}>
  {u => <Profile user={u()} />}
</Show>

<Switch>
  <Match when={route() === "profile"}>{() => <Profile />}</Match>
</Switch>
```

## DOM

### Attributes & events: closer to HTML (and fewer namespaces)

Solid 2.0 aims to be more “what you write is what the platform sees”:

- built-in attributes are treated as **attributes** (not magically mapped properties), and are generally **lowercase**
- boolean attributes are presence/absence (`muted={true}` adds it, `muted={false}` removes it)
- `attr:` and `bool:` namespaces are removed (you typically don’t need them)

```jsx
<video muted={true} />
<video muted={false} />

// When the platform really wants a string:
<some-element enabled="true" />
```

Also, `oncapture:` is removed.

### Directives: `use:` → `ref` directive factories (two-phase pattern)

```jsx
// 1.x
<button use:tooltip={{ content: "Save" }} />

// 2.0
<button ref={tooltip({ content: "Save" })} />
<button ref={[autofocus, tooltip({ content: "Save" })]} />
```

Two-phase directive factories are recommended (owned setup → unowned apply):

```js
function titleDirective(source) {
  // Setup phase (owned): create primitives/subscriptions here.
  // Avoid imperative DOM mutation at top level.
  let el;
  createEffect(source, value => {
    if (el) el.title = value;
  });

  // Apply phase (unowned): DOM writes happen here.
  // No new primitives should be created in this callback.
  return nextEl => {
    el = nextEl;
  };
}
```

### `classList` → `class` (object/array forms)

```jsx
// 1.x
<div class="card" classList={{ active: isActive(), disabled: isDisabled() }} />

// 2.0
<div class={["card", { active: isActive(), disabled: isDisabled() }]} />
```

## Context

### Context providers: `Context.Provider` → “context is the provider”

```jsx
// 1.x
const Theme = createContext("light");
<Theme.Provider value="dark">{props.children}</Theme.Provider>

// 2.0
const Theme = createContext("light");
<Theme value="dark">{props.children}</Theme>
```

## Quick rename / removal map (not exhaustive)

- **`solid-js/web` → `@solidjs/web`**
- **`solid-js/store` → `solid-js`**
- **`solid-js/h` → `@solidjs/h`**
- **`solid-js/html` → `@solidjs/html`**
- **`solid-js/universal` → `@solidjs/universal`**
- **`Suspense` → `Loading`**
- **`ErrorBoundary` → `Errored`**
- **`mergeProps` → `merge`**
- **`splitProps` → `omit`**
- **`createSelector` → `createProjection` / `createStore(fn)`**
- **`unwrap` → `snapshot`**
- **`classList` → `class`**
- **`mergeProps` / `splitProps` → `merge` / `omit`**
- **`createResource` removed** → async computations + `Loading`
- **`startTransition` / `useTransition` removed** → built-in transitions + `isPending`/`Loading` + optimistic APIs
- **`use:` directives removed** → `ref` directive factories
- **`attr:` / `bool:` removed** → standard attribute behavior
- **`oncapture:` removed**
- **`onMount` → `onSettled`**



https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/README.md

# Solid 2.0 RFCs

Temporary beta documentation for Solid 2.0. Treat **`MIGRATION.md`** as the primary entrypoint for beta testers; the RFCs below are deeper, topic-focused docs that may be folded into the main documentation over time.

**Overview:** [Solid 2.0 Proposed API Changes (HackMD)](https://hackmd.io/@0u1u3zEAQAO0iYWVAStEvw/SyXYy2swbg)

**Start here (beta tester guide):** [MIGRATION.md](MIGRATION.md)

The RFCs below are **deep dives** on specific topic areas. Over time, it’s expected that the most important bits will be folded into the main docs; these are intentionally detailed and “proposal-shaped”.

---

## RFC index (7)

| # | RFC | One-line summary |
|---|-----|------------------|
| 01 | [Reactivity, batching, and effects](01-reactivity-batching-effects.md) | pureWrite, strict top-level reads, flush, split effects, onSettled |
| 02 | [Signals, derived primitives, ownership, and context](02-signals-derived-ownership.md) | Derived signal/store, createRoot dispose-by-parent, Context as Provider |
| 03 | [Control flow](03-control-flow.md) | For/Repeat, Show/Switch, Loading, Errored |
| 04 | [Stores](04-stores.md) | draft-first setters, merge/omit, projections (createProjection/createStore(fn)), deep |
| 05 | [Async data](05-async-data.md) | Async in computations, isPending, latest, transitions |
| 06 | [Actions and optimistic](06-actions-optimistic.md) | action (generator), createOptimistic / createOptimisticStore |
| 07 | [DOM](07-dom.md) | HTML standards, class, booleans |

https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/01-reactivity-batching-effects.md

# RFC: Reactivity, batching, and effects

**Start here:** If you’re migrating an app, read the beta tester guide first: [MIGRATION.md](MIGRATION.md)

## Summary

Solid 2.0 tightens the reactivity model: no writes under owned scope (with narrow exceptions), stricter use of `untrack` for top-level reactive reads, and microtask batching via `flush` instead of `batch`. Effects are split into a tracking phase and an effect phase, enabling safer async and resumability. `createTrackedEffect` and `onSettled` (replacing `onMount`) complete the picture. These changes make the execution model predictable and allow Loading/Error boundaries to work correctly with async.

## Motivation

- **Writes under scope:** Writing to a signal inside a tracked context (e.g. inside an effect or component body) can cause subtle bugs and makes the graph harder to reason about. Split effects make it safe to disallow this by default.
- **Strict top-level access (new):** Top-level reactive reads in component body can accidentally capture dependencies and re-run in async situations; in 2.0 we warn in dev unless the read is inside createMemo/createEffect or explicit `untrack`.
- **Batching:** Synchronous batching with `batch()` is replaced by default microtask batching. Setters queue work; reads (and DOM/effects) reflect the update after the batch flushes (next microtask, or via `flush()`). This aligns with Vue/Svelte and simplifies the model; `batch` is no longer needed.
- **Split effects:** Running all “tracking” (compute) halves of effects before any “effect” (callback) halves gives a clear dependency picture before side effects run, which is required for async, Loading, and Errored boundaries.

## Detailed design

### No writes under owned scope

Writing to a signal inside a reactive scope (effect, memo, component body) warns in dev. Writes belong in event handlers, `onSettled`, or untracked blocks. When a signal must be written from within scope (e.g. internal state), opt in with `pureWrite: true`:

```js
// Default: warn if set in effect/component
const [count, setCount] = createSignal(0);

// Opt-in: allow writes in owned scope (e.g. internal flags)
const [ref, setRef] = createSignal(null, { pureWrite: true });
```

`pureWrite` is not a general-purpose escape hatch. A common misuse is enabling it to silence warnings for application state while still writing from reactive scope:

```js
// ❌ BAD: using pureWrite to silence a write-under-scope warning for app state
const [count, setCount] = createSignal(0, { pureWrite: true });
const [doubled, setDoubled] = createSignal(untrack(count) + 1); // force untracked read to get around other warning
createMemo(() => setDoubled(count() + 1)); // feedback loop

// ✅ GOOD: derive without writing back, or write in an event
const doubled = createMemo(() => count() * 2);
button.onclick = () => setCount((c) => c + 1);
```

### Strict top-level access (new in 2.0)

**What’s new:** In component body **top level**, reactive reads (signal, signal-backed prop, store property) **warn in dev** unless they are inside a reactive scope (e.g. `createMemo`, `createEffect`) or explicitly wrapped in `untrack`. This steers authors to avoid accidental dependencies that would re-run the component in async or lose reactivity.

```js
// New: top-level read in component body warns unless wrapped
function Title(props) {
  const t = untrack(() => props.title); // intentional one-time read — no warn
  return <h1>{t}</h1>;
}
function Bad(props) {
  const t = props.title; // warns: Untracked reactive read
  return <h1>{t}</h1>;
}

// Common pitfall: destructuring reactive props at top level (warns)
function BadArgs({ title }) {
  return <h1>{title}</h1>;
}
```

This also applies to the **bodies of control-flow function children** (e.g. Show/Match/For callbacks): those callbacks are structure-building, so reactive reads done directly in the callback body won’t update and will warn in dev. Prefer reading through JSX expressions (which compile to tracked computations), or wrap the read in a reactive scope.

```js
// ❌ BAD: reactive read in callback body (warns)
<Show when={user()}>
  {(u) => {
    const name = u().name;
    return <span>{name}</span>;
  }}
</Show>

// ✅ GOOD: read in JSX expression (tracked)
<Show when={user()}>{(u) => <span>{u().name}</span>}</Show>
```

Tests: `packages/solid/test/component.spec.ts` ("Strict Read Warning") — warns on direct signal read, props backed by signal, store destructuring; no warn inside createMemo, createEffect, or untrack.

Derived signals should not initiate other signals from reactive values except through the derived (initializer) form.

### `flush` and microtask batching

Updates are applied on the next microtask by default. After calling a setter, reads continue to return the last committed value until the batch is flushed (next microtask, or explicitly via `flush()`). DOM updates and effect callbacks also run after the flush. Use **`flush()`** when you need to read the DOM right after a state change (e.g. focus):

```js
function handleSubmit() {
  setSubmitted(true);
  flush(); // apply updates now
  inputRef.focus(); // DOM is up to date
}
```

**`batch` is removed;** `flush()` is the way to synchronously apply pending updates.

### Split tracking from effect

Effects have two phases: **compute** (reactive reads only; dependencies recorded) and **effect** (side effects; runs after all compute phases in the batch). This gives a clear dependency picture before any side effects run and is required for async and boundaries.

```js
createEffect(
  () => count(),           // compute: only reads
  (value, prev) => {       // effect: runs after flush
    console.log(value);
    return () => { /* cleanup */ };
  }
);

// With initial value
createEffect(
  () => [a(), b()],
  (deps) => { /* ... */ },
  undefined
);
```

**`createRenderEffect`** is split the same way and tears when dependencies change. **`createEffect`** may accept an options object with `effect` and `error` for handling errors from the reactive graph (e.g. async).

### createTrackedEffect and onSettled

**`createTrackedEffect`** is the single-callback form for special cases; it may re-run in async situations and is not the default. **`onSettled`** replaces `onMount`: run logic when the current activity is settled (e.g. after mount, or from an event handler to defer work until the reactive graph is idle).

```js
onSettled(() => {
  const value = count(); // reactive read allowed here
  doSomething(value);
  return () => cleanup();
});
```
Unlike other tracked scopes these primitives cannot create nested primitives which is a breaking change from Solid 1.x. They also return a cleanup function instead of their previous value.

**`onCleanup`** remains for reactive lifecycle cleanup inside computations. But is not expected to be used inside side effects.

## Migration / replacement

- **`batch`:** Remove; use `flush()` when you need synchronous application of updates (e.g. before reading DOM).
- **`onMount`:** Replace with `onSettled`.
- **Writes under scope:** Move setter calls to event handlers, `onSettled`, or untracked blocks; or create the signal with `{ pureWrite: true }` for the rare valid case (e.g. internal or intentionally in-scope writes).

## Removals

| Removed | Replacement / notes |
|--------|----------------------|
| `batch` | `flush()` when you need immediate application |
| `onError` / `catchError` | Effect `error` callback or ErrorBoundary / Errored |
| `on` helper | No longer necessary with split effects |

`@solidjs/legacy` can provide approximations for deprecated APIs where feasible.

## Alternatives considered

- Keeping `batch` as an alias for “run updates now” was considered; unifying on `flush` reduces API surface and matches the mental model (drain queue).
- Keeping a single-callback effect as the default was rejected in favor of split effects for async and boundary semantics.

## Open questions

- Whether to promote the write-under-scope warning to an error in a future release.

https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/02-signals-derived-ownership.md

# RFC: Signals, derived primitives, ownership, and context

**Start here:** If you’re migrating an app, read the beta tester guide first: [MIGRATION.md](MIGRATION.md)

## Summary

This RFC groups the “core runtime ergonomics” changes that are tightly coupled: **ownership defaults**, **context provider ergonomics**, and **derived (function-form) primitives**. The goal is to make reactive lifetime more predictable (fewer unowned graphs), reduce API surface (`Context.Provider`, `createComputed`), and make “derived state” patterns use consistent primitives with clearer semantics.

## Motivation

- **Ownership as the default:** In 1.x it’s easy to accidentally create unowned reactive graphs (especially in library code), which leads to leaks and confusing cleanup. In 2.0 we want ownership to be the default and detaching to be explicit.
- **Context ergonomics:** `Context.Provider` is boilerplate and a special-case surface. Using the context value directly as the provider component reduces ceremony and aligns with how it’s used.
- **Derived primitives:** Patterns that previously relied on `createComputed` (or ad-hoc “write-back” computations) should move to primitives that are composable with async and with split effects. “Function-form” `createSignal` / `createStore` offer a single consistent place for “derived but writable” shapes.

## Detailed design

### Ownership: `createRoot` is owned by the parent by default

In 2.0, a root created inside an existing owned scope is itself owned by that parent (and will be disposed when the parent is disposed). You still get a `dispose` callback for manual cleanup.

```js
function Widget() {
  createRoot((dispose) => {
    const [count, setCount] = createSignal(0);
    const id = setInterval(() => setCount((c) => c + 1), 1000);
    onCleanup(() => clearInterval(id));
  });
  return null;
}
// When Widget is disposed, the nested root is disposed too.
```

#### Detaching is explicit: `runWithOwner(null, ...)`

If you really want “no owner” (module singletons, external integrations), detach explicitly:

```js
// A truly detached singleton
export const singleton = runWithOwner(null, () => {
  const [value, setValue] = createSignal(0);
  return { value, setValue };
});
```

This makes “global lifetime” an explicit opt-in rather than the accidental default.

### Context: context value is the provider component

Context creation still looks the same, but usage becomes simpler: the context itself is a component that takes a `value` prop and provides it to descendants.

```js
// 2.0
const ThemeContext = createContext("light");

function App() {
  return (
    <ThemeContext value="dark">
      <Page />
    </ThemeContext>
  );
}

function Page() {
  const theme = useContext(ThemeContext);
  return <div class={theme}>...</div>;
}
```

### Derived primitives: function-form `createSignal` and `createStore`

2.0 supports function overloads for `createSignal` and `createStore` to represent “derived state” using the same primitives users already know.

#### Function-form `createSignal` (“writable memo”)

`createSignal(fn, initialValue?, options?)` creates a signal whose value is computed by `fn(prev)` and can also be written through its setter. This replaces many `createComputed` “write-back” use cases with an explicit primitive.

```js
// Example: derived signal
const [value, setValue] = createSignal(() => props.something);
// setValue(...) writes like a normal signal; the compute receives prev on recompute.ß
```

#### Function-form `createStore` (derived/projection store)

`createStore(fn, initial?, options?)` creates a derived store driven by mutation in `fn(draft)` (and may also return a value / Promise / async iterable). It’s the store analogue for derived shapes and underpins patterns like “selector-like” updates without notifying everything.

```js
// Example: derived store that only flips the active key
const [selected, setSelected] = createStore((draft) => {
  const id = selectedId();
  draft[id] = true;
  if (draft._prev != null) delete draft[draft._prev];
  draft._prev = id;
}, {});
```

## Migration / replacement

### `Context.Provider` → context-as-provider

```jsx
// 1.x
<ThemeContext.Provider value="dark">
  <Page />
</ThemeContext.Provider>

// 2.0
<ThemeContext value="dark">
  <Page />
</ThemeContext>
```

### Unowned roots → explicit detachment

- If you relied on roots living “forever,” wrap them in `runWithOwner(null, ...)`.
- Otherwise, prefer creating roots under an existing owner so disposal is automatic.

### `createComputed` removal

If you used `createComputed` to “write back”:

- Prefer split `createEffect(compute, effect)` (RFC 01) when the intent is “react to X and do side effects”.
- Prefer function-form `createSignal` / `createStore` when the intent is “derived state with a setter”.
- Prefer `createMemo` for readonly derived values.

## Removals

| Removed | Replacement |
|--------|-------------|
| `createComputed` | `createEffect` (split), function-form `createSignal`/`createStore`, or `createMemo` |
| `Context.Provider` | Use the context directly as the provider component (`<Context value={...}>`) |

## Alternatives considered

- Keeping `createRoot` detached by default: rejected because it makes leaks and “forever lifetime” accidental.
- Keeping `Context.Provider`: rejected as needless boilerplate and special casing.
- Keeping `createComputed`: rejected because “write-back computations” are harder to reason about in an async/split-effects model.

## Open questions

- Should we provide an explicit “detached root” helper (sugar over `runWithOwner(null, ...)`) for readability?

https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/03-control-flow.md

# RFC: Control flow

**Start here:** If you’re migrating an app, read the beta tester guide first: [MIGRATION.md](MIGRATION.md)

## Summary

Solid 2.0 simplifies and unifies control-flow APIs by consolidating list rendering into a single `For` signature (covering the old `For`/`Index` split), introducing `Repeat` for range/count-based rendering, and renaming/reshaping async and error boundaries as `Loading` and `Errored`. The goal is fewer “nearly-the-same” APIs, more explicit keying semantics, and control-flow callbacks that are consistent with the 2.0 reactivity model.

## Motivation

- **One list primitive:** Having both `For` and `Index` encourages bikeshedding and accidental misuse. A single `For` that can be keyed or index-based is easier to teach and document.
- **Ranges without diffing:** Rendering “count-based” lists (skeletons, ranges, windowing) shouldn’t require list diffing; `Repeat` expresses this directly.
- **Async and error UX:** Names like Suspense and ErrorBoundary are long and carry baggage. `Loading` and `Errored` are concise and align better with their actual role in the 2.0 async model.

## Detailed design

### List rendering: `For` (keyed, non-keyed, custom key)

`For` takes `each`, optional `fallback`, optional `keyed`, and a children mapping function that receives **accessors** for both the item and the index.

```jsx
// Default keyed behavior (identity)
<For each={todos()}>
  {(todo, i) => <TodoRow todo={todo()} index={i()} />}
</For>

// Index-style behavior (reuse by index)
<For each={todos()} keyed={false}>
  {(todo, i) => <TodoRow todo={todo()} index={i()} />}
</For>

// Custom key
<For each={todos()} keyed={(t) => t.id}>
  {(todo) => <TodoRow todo={todo()} />}
</For>

// Fallback
<For each={todos()} fallback={<EmptyState />}>
  {(todo) => <TodoRow todo={todo()} />}
</For>
```

Notes:

- `keyed={false}` is the direct replacement for `Index`.
- `keyed={(item) => key}` is the escape hatch for stable keys without having to pre-normalize lists.

### Range/count rendering: `Repeat`

`Repeat` renders based on `count` (and optional `from`), with no list diffing. It’s useful for skeletons, numeric ranges, and windowed UIs.

```jsx
// 10 items: 0..9
<Repeat count={10}>{(i) => <Item index={i} />}</Repeat>

// Windowing / offset
<Repeat count={visibleCount()} from={start()}>
  {(i) => <Row index={i} />}
</Repeat>

// Fallback when count is 0
<Repeat count={items.length} fallback={<EmptyState />}>
  {(i) => <div>{items[i]}</div>}
</Repeat>
```

### Conditionals: `Show`

`Show` supports element children or function children. Function children receive a narrowed accessor.

```jsx
<Show when={user()} fallback={<Login />}>
  {(u) => <Profile user={u()} />}
</Show>

// Keyed form (treats value identity as the switching condition)
<Show when={user()} keyed>
  {(u) => <Profile user={u()} />}
</Show>
```

### Branching: `Switch` / `Match`

`Switch` picks the first matching `Match`. `Match` supports element or function children.

```jsx
<Switch fallback={<NotFound />}>
  <Match when={route() === "home"}>
    <Home />
  </Match>
  <Match when={route() === "profile"}>
    <Profile />
  </Match>
</Switch>
```

### Async boundary: `Loading`

`Loading` is the boundary for async computations. It shows `fallback` while async values required by its subtree are not ready.

```jsx
<Loading fallback={<Spinner />}>
  <UserProfile id={params.id} />
</Loading>
```

In 2.0’s async model, async values are part of computations (not a separate `createResource`), so `Loading` is the user-facing “this subtree may suspend” boundary.

### Error boundary: `Errored`

`Errored` is the error boundary. It supports a static fallback or a callback form that receives the error and a reset function.

```jsx
<Errored
  fallback={(err, reset) => (
    <div>
      <p>Something went wrong.</p>
      <pre>{String(err)}</pre>
      <button onClick={reset}>Retry</button>
    </div>
  )}
>
  <Page />
</Errored>
```

## Migration / replacement

### `Index` → `For keyed={false}`

```jsx
// 1.x
<Index each={items()}>
  {(item, i) => <Row item={item()} index={i} />}
</Index>

// 2.0
<For each={items()} keyed={false}>
  {(item, i) => <Row item={item()} index={i()} />}
</For>
```

### `Suspense` → `Loading`

```jsx
// 1.x
<Suspense fallback={<Spinner />}>
  <Page />
</Suspense>

// 2.0
<Loading fallback={<Spinner />}>
  <Page />
</Loading>
```

### `ErrorBoundary` → `Errored`

```jsx
// 1.x
<ErrorBoundary fallback={(err, reset) => <Fallback err={err} reset={reset} />}>
  <Page />
</ErrorBoundary>

// 2.0
<Errored fallback={(err, reset) => <Fallback err={err} reset={reset} />}>
  <Page />
</Errored>
```

## Removals

| Removed | Replacement |
|--------|-------------|
| `Index` | `For keyed={false}` |
| `Suspense` | `Loading` |
| `ErrorBoundary` | `Errored` |

## Alternatives considered

- Keeping both `For` and `Index`: rejected in favor of one API with explicit keying.
- Adding a separate “range” mode to `For`: rejected in favor of a dedicated `Repeat` that makes “no diffing” obvious.

## Open questions

- Should `For` default to keyed-by-identity explicitly documented as the default (vs relying on undefined meaning “keyed”)? If so, should `keyed` default to `true` in docs and types?

https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/04-stores.md

# RFC: Stores

**Start here:** If you’re migrating an app, read the beta tester guide first: [MIGRATION.md](MIGRATION.md)

## Summary

Solid 2.0’s store layer leans into “mutable draft” ergonomics by default: store setters accept a draft callback (produce-style) and can optionally return a value to perform a shallow replacement/diff. Helper APIs are simplified (`mergeProps` → `merge`, `splitProps` → `omit`), and a new derived-store primitive (`createProjection`, also reachable via `createStore(fn)`) replaces selector-style patterns with a more general “mutate a projection” approach. A `deep()` helper is provided for cases where you need deep observation rather than property-level tracking.

## Motivation

- **Ergonomics without losing granularity:** Draft-mutation is the most ergonomic way to express updates; the store system can still keep fine-grained reactivity under the hood.
- **Fewer special-case helpers:** `merge` and `omit` apply broadly (props and stores) and avoid surprising `undefined` semantics.
- **Derived stores that scale:** `createSelector`-style APIs solve one pattern; `createProjection` generalizes it and can be used for selection, derived caches, and async-derived store values.

## Detailed design

### Draft-first store setters (“produce by default”)

The primary store update form is a setter that receives a mutable draft.

```js
const [store, setStore] = createStore({ greeting: "hi", list: [] });

setStore((s) => {
  s.greeting = "hello";
  s.list.push("value");
});
```

#### Returning a value performs a shallow replacement/diff

When you need to replace a top-level array/object in one go, return a value from the setter callback.

```js
const [store, setStore] = createStore({ list: ["a", "b"] });

setStore((s) => {
  // Replace the top-level list (shallow diff)
  return { ...s, list: [] };
});
```

#### `storePath` (compat helper for 1.x-style path setters)

Solid 2.0’s default store setter is draft-first (produce-style). For teams migrating from Solid 1.x’s “path argument” setter ergonomics, `storePath(...)` is provided as an **opt-in helper** that adapts the old style into a function you can pass to `setStore`.

```js
// 2.0 preferred: draft-first setter
setStore((s) => {
  s.user.address.city = "Paris";
});

// Optional compat: 1.x-style path setter via storePath
setStore(storePath("user", "address", "city", "Paris"));
```

`storePath` also supports common path patterns (indices, filters, ranges) and a delete sentinel:

```js
setStore(storePath("items", { from: 1, to: 4, by: 2 }, 99));
setStore(storePath("nickname", storePath.DELETE));
```

### `merge` (rename and semantic cleanup)

`merge` replaces `mergeProps` and is treated as a general helper for merging multiple sources. Importantly: **`undefined` is a value**, not “missing”.

```js
const defaults = { a: 1, b: 2 };
const overrides = { b: undefined };
const merged = merge(defaults, overrides);

// merged.b is undefined (explicit override)
```

### `omit` (replaces `splitProps`)

Instead of “splitting” (which creates extra objects and can de-opt proxies), use `omit` to create a view without the listed keys.

```js
const props = { a: 1, b: 2, c: 3 };
const rest = omit(props, "a", "b");

rest.c;        // 3
"a" in rest;   // false
```

### Derived stores: `createProjection` and `createStore(fn)`

`createProjection(fn, initial?, options?)` creates a mutable derived store (a projection). The derive function receives a draft that you can mutate.

If the derive function **returns a value**, that value is **reconciled** into the projection output (rather than being shallowly replaced). This makes “return new data” projections work well for lists/maps while preserving identity for unchanged entries (keyed by `options.key`, default `"id"`).

```js
// Selection without notifying every row
const [selectedId, setSelectedId] = createSignal("a");

const selected = createProjection((s) => {
  const id = selectedId();
  s[id] = true;
  if (s._prev != null) delete s[s._prev];
  s._prev = id;
}, {});
```

```js
// Reconcile returned list data into a projection (keyed reconciliation)
const users = createProjection(async () => {
  return await api.listUsers();
}, [], { key: "id" });
```

`createStore(fn, initial?, options?)` is the derived-store form; it’s effectively “projection store” creation using the familiar store API.

```js
const [cache] = createStore((draft) => {
  // mutate derived draft based on reactive inputs
  draft.value = expensive(selector());
}, { value: 0 });
```

### `snapshot(store)` (replaces `unwrap`)

`snapshot(store)` produces a **non-reactive plain value** suitable for serialization or interop with libraries that expect plain objects/arrays.

In Solid 2.0 the store implementation leans on immutable internals; simply “unwrapping” proxies is not sufficient when you need a distinct object graph. `snapshot` generates a new object/array where necessary (while preserving references when nothing has changed).

```js
const [store] = createStore({ user: { name: "A" } });

const plain = snapshot(store);
JSON.stringify(plain);
```

### `deep(store)` helper

Store tracking is normally property-level (optimal). When you truly need deep observation (e.g. for serialization, logging, or “watch everything”), use `deep(store)` inside a reactive scope.

```js
createEffect(
  () => deep(store),
  (snapshot) => {
    // runs when anything inside store changes
  }
);
```

## Migration / replacement

### `mergeProps` → `merge`

- Rename imports/usage.
- Update expectations: `undefined` overrides rather than being skipped.

### `splitProps` → `omit`

- Replace `splitProps(props, ["a", "b"])` with `omit(props, "a", "b")`.
- Prefer passing `props` through where possible rather than copying.

### `createSelector` → `createProjection`

- Replace selector patterns with a projection store that updates only the affected keys.

### `unwrap` → `snapshot`

- Replace `unwrap(store)` with `snapshot(store)` when you need a plain value for serialization/interop.

## Removals

| Removed | Replacement |
|--------|-------------|
| `mergeProps` | `merge` |
| `splitProps` | `omit` |
| `createSelector` | `createProjection` / `createStore(fn)` |
| `unwrap` | `snapshot` |

## Alternatives considered

- Keeping `splitProps`: rejected due to allocation/proxy-deopt costs and because `omit` is sufficient.
- Keeping `createSelector`: rejected as too narrow; `createProjection` is a more general tool.

https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/05-async-data.md

# RFC: Async data

**Start here:** If you’re migrating an app, read the beta tester guide first: [MIGRATION.md](MIGRATION.md)

## Summary

Solid 2.0 makes async a first-class capability of computations: `createMemo`, derived stores, and other computations can return **Promises** or **AsyncIterables**, and consumers interact with them through normal accessors. Pending async values suspend by throwing an internal “not ready” signal through the reactive graph, and `Loading` is the boundary that turns that suspension into UI. This removes the need for a separate `createResource` primitive. For “stale while revalidating” UI and coordination, 2.0 provides `isPending(fn)` and `latest(fn)`.

## Motivation

- **One model:** Async shouldn’t require a parallel set of primitives (resources vs signals). If computations can be async, the rest of the system (effects, boundaries, SSR/hydration) can treat async consistently.
- **Better types:** Async values can be represented without pervasive `T | undefined` “loading holes”. UI should be expressed via `Loading` boundaries rather than nullable types.
- **Composability:** When async is part of computations, derived values can combine sync + async naturally without bespoke resource combinators.

## Detailed design

### Async in computations (no `createResource`)

Any computation may return a Promise (or AsyncIterable) to represent pending work. Consumers read the accessor as usual; if it isn’t ready, the graph suspends until it resolves.

```js
const user = createMemo(() => fetchUser(params.id));

function Profile() {
  // user() suspends if not ready — wrap in <Loading>
  return <div>{user().name}</div>;
}

<Loading fallback={<Spinner />}>
  <Profile />
</Loading>
```

This pushes “loading state” to UI structure (boundaries) instead of leaking into every type.

### `Loading` is the UI boundary

`Loading` shows fallback while the subtree needs unresolved async values.

Importantly, `Loading` is intended to cover **initial readiness**: it handles the first time a subtree attempts to read async-derived values that are not ready yet. After the subtree has produced a value, subsequent revalidation/refresh should generally not “kick you back” into the fallback; use `isPending` for “background work is happening” UI.

```jsx
<Loading fallback={<Spinner />}>
  <UserProfile id={id()} />
</Loading>
```

Nested `Loading` boundaries can be used to avoid blocking large subtrees and to control where loading UI appears.

### `isPending(fn)` (stale-while-revalidating queries)

`isPending` answers: “Does this expression currently have pending async work while also having a usable stale value?”

This means `isPending` is **false during the initial `Loading` fallback** (there is no stale value yet, and the suspended subtree isn’t producing UI). It becomes useful once you’ve rendered at least once and want to show “refreshing…” indicators during revalidation without replacing the whole subtree with a spinner again.

```js
const users = createMemo(() => fetchUsers());
const posts = createMemo(() => fetchPosts());

const listPending = () => isPending(() => users() || posts());

return (
  <>
    <Show when={listPending()}>{/* subtle "refreshing…" indicator */}</Show>
    <Loading fallback={<Spinner />}>
      <List users={users()} posts={posts()} />
    </Loading>
  </>
);
```

The intent is to replace `.loading`-style flags that belong to a specific primitive (`createResource`) with something that works for any expression.

### `latest(fn)` (peek at in-flight values)

`latest(fn)` reads the “in flight” value of a signal/computation during transitions, and may fall back to stale if the next value isn’t available yet.

```js
const [userId, setUserId] = createSignal(1);
const user = createMemo(() => fetchUser(userId()));

// During a transition, this can reflect the in-flight userId
const latestUserId = () => latest(userId);
```

### Transitions: built-in, multiple in flight

2.0 treats transitions as a core scheduling concept rather than something you explicitly wrap in `startTransition`/`useTransition`. Multiple transitions can be in flight; “entangling” determines what should block what. The user-facing pieces are the observable pending state (`isPending`) and optimistic APIs (RFC 06).

## Migration / replacement

### `createResource` → async computations + `Loading`

```js
// 1.x
const [user] = createResource(id, fetchUser);

// 2.0
const user = createMemo(() => fetchUser(id()));
```

Wrap reads of async accessors in `Loading` to control where fallback UI appears.

### `.loading` → `isPending`

Replace resource-specific flags with expression-level pending checks.

### `startTransition` / `useTransition`

Removed in favor of built-in transition behavior. Pending UI should be expressed via `Loading` and `isPending`. Optimistic UI should use RFC 06 primitives.

## Removals

| Removed | Replacement |
|--------|-------------|
| `createResource` | Async computations (`createMemo`, `createStore(fn)`, projections) + `Loading` |
| `useTransition` / `startTransition` | Built-in transitions; use `Loading`, `isPending`, optimistic APIs |

## Alternatives considered

- Keeping `createResource`: rejected to avoid parallel async models and duplicated surface area.
- Keeping explicit transition wrappers: rejected because transitions are a scheduling concern that should be inferred and managed by the runtime.


https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/06-actions-optimistic.md

# RFC: Actions and optimistic updates

**Start here:** If you’re migrating an app, read the beta tester guide first: [MIGRATION.md](MIGRATION.md)

## Summary

Solid 2.0 introduces an `action()` wrapper for async mutations and a pair of optimistic primitives—`createOptimistic` and `createOptimisticStore`—to express “optimistic UI” without inventing a separate mutation subsystem. Actions run inside transitions and provide a structured way to interleave optimistic writes, async work, and refreshes. Optimistic primitives behave like signals/stores but reset to their source when the transition completes.

## Motivation

- **Mutations are not reads:** Async data reads can be modeled as computations (RFC 05). Mutations need a different tool: they should coordinate optimistic writes, async side effects, and follow-up refreshes.
- **Optimism should compose:** Optimistic UI should reuse the signal/store mental model, and should integrate with transitions rather than forking the reactive graph.
- **Ergonomics:** A generator-based API provides a simple “do optimistic update → await → refresh” workflow without needing ambient async context features.

## Detailed design

### `action(fn)` for async mutations

`action()` wraps a generator or async generator. It returns an async function you can call from handlers. Inside an action, you can:

- do optimistic writes
- yield/await async work
- refresh derived async computations via `refresh()`

```js
const [todos, setOptimisticTodos] = createOptimisticStore(() => api.getTodos(), []);

const saveTodo = action(function* (todo) {
  // optimistic write
  setOptimisticTodos((todos) => { todos.push(todo); });

  // perform async work
  yield api.addTodo(todo);

  // refresh reads (store/projection form)
  refresh(todos);
});
```

For better TS ergonomics, an async generator form is also viable:

```js
const saveTodo = action(async function* (todo) {
  setOptimisticTodos((todos) => { todos.push(todo); });
  const res = await api.addTodo(todo);
  yield; // resume action in the same transition context
  refresh(todos);
  return res;
});
```

### `refresh()` (explicit recomputation)

Solid 2.0 exports a `refresh()` helper to explicitly re-run derived reads when you know the underlying source-of-truth may have changed (for example, after an action completes a server write).

Conceptually, `refresh()` is “invalidate and recompute now”, without requiring you to thread bespoke `refetch()` methods through your app.

It supports two common forms:

- **Thunk form**: `refresh(() => expr)` re-runs `expr` (typically something that reads async-derived values) and returns its value.
- **Refreshable form**: `refresh(x)` requests recomputation for `x` when `x` is a derived signal/store/projection that participates in refresh (e.g. things created via function forms like `createStore(() => ...)` / projections).

```js
// Re-run a read tree explicitly
refresh(() => query.user(id()));
```

```js
// After a server write, refresh derived store reads
const [todos] = createStore(() => api.getTodos(), []);

const addTodo = action(function* (todo) {
  yield api.addTodo(todo);
  refresh(todos);
});
```

### `createOptimistic` (optimistic signal)

`createOptimistic` has the same surface as `createSignal`, but its writes are treated as optimistic—values can be overridden during a transition and revert when the transition completes.

```js
const [name, setName] = createOptimistic("Alice");

const updateName = action(function* (next) {
  setName(next);          // optimistic
  yield api.saveName(next);
});
```

### `createOptimisticStore` (optimistic store)

`createOptimisticStore(fnOrValue, initial?, options?)` is the store analogue. A common pattern is to derive from a source getter and then apply optimistic mutations in an action.

```js
const [todos, setOptimisticTodos] = createOptimisticStore(() => api.getTodos(), []);

const addTodo = action(function* (todo) {
  setOptimisticTodos((todos) => { todos.push(todo); });
  yield api.addTodo(todo);
  // refresh store/projection form (object with [$REFRESH])
  refresh(todos);
});
```

## Migration / replacement

- If you previously used ad-hoc “mutation wrappers” + manual flags, prefer consolidating the pattern into `action()` + optimistic primitives.
- If you used `startTransition` or `useTransition` for mutation UX, those go away; actions/transitions are integrated into the runtime model, and pending UX should be expressed via `isPending`/`Loading` (RFC 05).

## Removals

No direct removals; this RFC is additive. (It complements the removal of `useTransition`/`startTransition` covered in RFC 05.)

## Alternatives considered

- AsyncContext-based mutation scope: rejected for now (not widely available/portable).
- React-style `startTransition` wrappers: rejected in favor of built-in transitions and structured actions.
- Manually passing in a resume function to call after await instead of using generators.

https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/07-dom.md

# RFC: DOM — attributes, class, and standards

**Start here:** If you’re migrating an app, read the beta tester guide first: [MIGRATION.md](MIGRATION.md)

## Summary

DOM behavior in Solid 2.0 follows HTML standards by default: attributes over properties, lowercase attribute names for built-ins, and consistent boolean handling. The `class` prop is enhanced (classList merged in, support for array/object for composition). These changes improve interoperability with web components and SSR and simplify the attribute/property story.

## Motivation

- **HTML standards:** Supporting multiple ways to set attributes/properties by type makes it harder to build on Solid (e.g. custom elements, SSR). Favoring attributes and lowercase aligns with the platform and removes special cases; `attr:` and `bool:` namespaces are no longer needed.
- **class:** Merging `classList` into `class` and supporting array/object (clsx-style) reduces API surface and supports composition without extra helpers.

## Detailed design

### Follow HTML standards by default

- **Attributes over properties:** Prefer setting attributes rather than properties in almost all cases. Aligns with web components and SSR.
- **Lowercasing:** Use HTML lowercase for built-in attribute names (no camelCase for attributes). Exceptions:
  - **Event handlers** remain camelCase (e.g. `onClick`) to keep the `on` modifier clear.
  - **Default attributes** such as `value`, `selected`, `checked`, `muted` continue to be handled as props where that avoids confusion.
- **Namespaces:** `attr:` and `bool:` namespaces are removed; the single standard behavior makes the model consistent.

### Enhanced class prop

- **`classList` is removed;** its behavior is merged into `class`.
- **`class`** accepts: string, object (key = class name, value = truthy to apply), or array of strings/objects. Enables clsx-style composition. Example:

```jsx
<div class="card" />
<div class={{ active: isActive(), disabled: isDisabled() }} />
<div class={["card", props.class, { active: isActive() }]} />
```

### Consistent boolean handling

- Boolean literals add/remove the attribute (no `="true"` string). For attributes that require the string `"true"`, pass a string.
- Types are updated to reflect this.

```jsx
// Presence/absence boolean attributes
<video muted={true} />
<video muted={false} />

// When the platform requires a string value:
<some-element enabled="true" />
```

### Directives via `ref` (and removal of `use:`)

Solid 2.0 removes the `use:` directive namespace and instead treats “directives” as a first-class **ref** pattern. The `ref` prop becomes the single composition point for:

- DOM element access (`ref={el => ...}`)
- directive factories (`ref={tooltip(options)}`)
- composition (`ref={[a, b, c]}`; arrays may be nested)

```jsx
// 1.x: directive namespace
<input use:autofocus />
<button use:tooltip={{ content: "Save" }} />

// 2.0: directive/ref callbacks (factory form)
<input ref={autofocus} />
<button ref={tooltip({ content: "Save" })} />
```

Multiple refs (or directives) can be applied by passing an array:

```jsx
<button ref={[autofocus, tooltip({ content: "Save" })]} />
```

#### Two-phase directive factories (owned setup, unowned application)

The recommended directive pattern is **two-phase**, similar in spirit to “split effects” (compute phase vs apply phase):

- **Setup phase (owned):** run once to create reactive primitives and subscriptions. This phase should not perform imperative DOM mutation.
- **Apply phase (unowned):** receives the element and performs DOM writes (including reactive writes). This phase should not create reactive primitives.

```js
function titleDirective(source) {
  // Setup phase (owned): create primitives/subscriptions here
  // but avoid imperative DOM mutation at top level.
  let el;

  createEffect(source, value => {
    // Effect can run before the element is available
    if (el) el.title = value;
  });

  // Apply phase (unowned): DOM writes happen here.
  // No new primitives should be created in this callback.
  return nextEl => {
    el = nextEl;
    el.title = source();
  };
}
```

Used as:

```jsx
<button ref={titleDirective(() => props.title)} />
```

## Migration / replacement

- **classList:** Use `class` with an object or array instead.
- **Attributes:** Use lowercase attribute names; use string `"true"` only where the platform requires it.
- **Directives:** Replace `use:foo={...}` with `ref={foo(...)}` (or `ref={foo}` when no options are needed). Use an array when you need multiple directives/refs.

## Removals

| Removed | Replacement / notes |
|--------|----------------------|
| `classList` | Use `class` with object or array |
| `oncapture:` | Removed (replacement pattern TBD / use native event options where applicable) |
| `attr:` / `bool:` namespaces | Single attribute/property model above |
| `use:` directives | Use `ref` callbacks / directive factories (`ref={directive(opts)}`); arrays compose (`ref={[a, b]}`) |
| `createMutable` (from Solid primitives) | Moved to / available from `@solidjs/signals` or primitives layer |
| `scheduler` / `createDeferred` | Moved to primitives; use primitives API |

`@solidjs/legacy` can provide compatibility for deprecated DOM APIs where feasible.

## Alternatives considered

- Keeping `classList` alongside `class` was rejected to avoid two ways to do the same thing and to simplify the compiler/runtime.
- Keeping camelCase for attributes was rejected in favor of HTML alignment and web component compatibility.

## Open questions

- Exact list of "default" props that stay as props (`value`, `selected`, `checked`, `muted`, …).


https://github.com/solidjs/solid/blob/next/documentation/resources/examples.md
# Examples

## Online
- [Counter](https://codesandbox.io/s/8no2n9k94l) Simple Counter
- [Simple Todos](https://codesandbox.io/s/lrm786ojqz) Todos with LocalStorage persistence
- [Simple Routing](https://codesandbox.io/s/jjp8m8nlz5) Use 'switch' control flow for simple routing
- [Scoreboard](https://codesandbox.io/s/solid-scoreboard-sjpje) Make use of hooks to do some simple transitions
- [Tic Tac Toe](https://playground.solidjs.com/anonymous/335adbcd-289e-42f8-9a9c-152a96277747) Simple Example of the classic game
- [Form Validation](https://codesandbox.io/s/solid-form-validation-2cdti) HTML 5 validators with custom async validation
- [CSS Animations](https://codesandbox.io/s/basic-css-transition-36rln?file=/index.js) Using Solid Transition Group
- [Styled Components](https://codesandbox.io/s/solid-styled-components-yv2t1) A simple example of creating Styled Components.
- [Styled JSX](https://codesandbox.io/s/solid-styled-jsx-xgx6b) A simple example of using Styled JSX with Solid.
- [Counter Context](https://codesandbox.io/s/counter-context-gur76) Implement a global store with Context API
- [Async Resource](https://codesandbox.io/s/2o4wmxj9zy) Ajax requests to SWAPI with Promise cancellation
- [Async Resource GraphQL](https://codesandbox.io/s/async-resource-graphql-r4rcx?file=/index.js) Simple resource for handling graphql request.
- [Suspense](https://codesandbox.io/s/5v67oym224) Various Async loading with Solid's Suspend control flow
- [Suspense Tabs](https://codesandbox.io/s/solid-suspense-tabs-vkgpj) Deferred loading spinners for smooth UX.
- [SuspenseList](https://codesandbox.io/s/solid-suspenselist-eorvk) Orchestrating multiple Suspense Components.
- [Redux Undoable Todos](https://codesandbox.io/s/pkjw38r8mj) Example from Redux site done with Solid.
- [Simple Todos Template Literals](https://codesandbox.io/s/jpm68z1q33) Simple Todos using Lit DOM Expressions
- [Simple Todos HyperScript](https://codesandbox.io/s/0vmjlmq94v) Simple Todos using Hyper DOM Expressions

## Demos
- [TodoMVC](https://github.com/solidjs/solid-todomvc) Classic TodoMVC example
- [Real World Demo](https://github.com/solidjs/solid-realworld) Real World Demo for Solid
- [Hacker News](https://github.com/solidjs/solid-hackernews) Hacker News Clone for Solid
- [Storybook](https://github.com/rturnq/storybook-solid) Solid with Storybook

## Benchmarks
- [JS Framework Benchmark](https://github.com/krausest/js-framework-benchmark/tree/master/frameworks/keyed/solid) The one and only
- [Sierpinski's Triangle Demo](https://github.com/ryansolid/solid-sierpinski-triangle-demo) Solid implementation of the React Fiber demo.
- [WebComponent Todos](https://github.com/shprink/web-components-todo/tree/main/solid) Showing off Solid Element
- [UIBench Benchmark](https://github.com/ryansolid/solid-uibench) a benchmark tests a variety of UI scenarios.
- [DBMon Benchmark](https://github.com/ryansolid/solid-dbmon) A benchmark testing ability of libraries to render unoptimized data.

https://www.brenelz.com/posts/migrating-to-solid-2/

Things Learned Migrating To Solid 2.0
Posted on:
March 29, 2026
So the other day I was thinking of what my next blog post should be. I haven’t been writing much and instead have been doing my best to test out the new Solid 2.0 beta as well as upgrade certain libraries in the ecosystem. This has included things like SolidStart, Tanstack Start, as well as Solid Query. Writing about my experience migrating seems to make the most sense since it’s fresh on my mind. It might also be helpful to others looking at migrating their projects.

So what have I learned? I’m glad you asked!

AI helps a ton with migrations
To put it bluntly AI saved me a ton of time. Solid 2.0 is a pretty major departure from 1.0 in a lot of ways. I feel migrations like this used to take months and now they take weeks. In a matter of a week or two I had WIP prs up for both Tanstack Start and SolidStart. This isn’t to say this code wasn’t without its faults but it at least allowed me to forge the path and find bugs or edge cases that can only come from upgrading real projects.

What helped quite significantly was creating a local reference folder with clones of the Solid source code and the signals repo. This gives the model more context when it needs to understand the shape of the new APIs. Even without this newer models are surprisingly good at digging through node_modules, but I still like having the important repos in context.

With this in place, i’ve found that the AI can do a createEffect migration pretty well. It can figure out that createEffect now takes 2-4 arguments instead of just the 1. It can also detect which signals are being tracked and put that in the first half of the effect.

Another thing I have found super valuable to keep the AI on track is a test suite. Tanstack Router has a really good suite of unit and end-to-end tests. When doing a migration this large, feedback like this is a must. I can have the AI iterate and get the tests passing. Without this it tends to go in circles and in some cases make things even worse.

This new technology also lets me work in code I don’t fully understand (for better or worse). For example, I really have no business in the @solidjs/signals repo, but with AI I can at least narrow things down to a specific change. Now AI usually doesn’t have the correct solution to the problem, but it can at least give someone with more knowledge than me a starting point.

Solid 2.0 feels really good
The other big takeaway is that Solid 2.0 feels really good to use, especially the async story. Being able to pass a promise into createMemo and have the system handle it in a sane way feels great. It feels like the framework is meeting you closer to how you already think about async code.

The old Suspense experience could feel a little jarring when it ripped the UI away from you the moment something upstream needed to load. It worked, but it could make even relatively small async interactions feel more dramatic than they needed to. In Solid 2.0, transitions feel like much less of a thing you have to think about manually. Things entangle naturally, pending state composes more cleanly, and the UI tends to stay in place while work is happening instead of constantly snapping between fallback and content.

That shift makes the whole model feel calmer. You can spend less time managing the mechanics of loading and more time deciding what should be pending, what should stay visible, and what the user actually needs in the moment.

Migration tips
So you’ve made it this far and you want to try out the new beta. Below I list some tips that would have saved me a lot of time if I had known them.

Testing-library workaround
Because this is still early, some packages have not fully moved over yet. Sometimes that is a hard blocker, and sometimes you can work around it. One example for me was @solidjs/testing-library. In a Vite setup I needed some combination of the following:

import { defineConfig } from "vite";
import solid from "vite-plugin-solid";

export default defineConfig({
  plugins: [solid({ ssr: true, hot: !process.env.VITEST })],
  resolve: {
    alias: {
      "solid-js/web": "@solidjs/web",
      "solid-js/store": "solid-js",
    },
  },
  environments: {
    ssr: {
      resolve: {
        noExternal: ["@solidjs/web"],
      },
    },
  },
  optimizeDeps: {
    include: ["@solidjs/testing-library"],
  },
  server: {
    deps: {
      inline: ["@solidjs/testing-library"],
    },
  },
});
I would not treat that as a universal fix, but it is the sort of config surgery you may need while the ecosystem settles.

Dedupe local Solid versions
If you are linking packages locally, you may need to dedupe Solid so you do not accidentally run multiple copies. Sometimes Solid warns you about this directly. Other times you just get weird hydration errors and have to figure it out the hard way.

import { defineConfig } from "vite";

export default defineConfig({
  resolve: {
    dedupe: [
      "solid-js",
      "@solidjs/web",
      "@solidjs/router",
      "@solidjs/signals",
      "@solidjs/meta",
    ],
  },
});
Use the next tag
When upgrading the beta, using the next tag makes life a lot easier. From there you can usually run pnpm update solid-js @solidjs/web vite-plugin-solid to move to the latest published beta. If you need a newer @solidjs/signals version before a full Solid release lands, a temporary pnpm.overrides entry can help.

{
  "dependencies": {
    "@solidjs/web": "next",
    "solid-js": "next",
    "vite-plugin-solid": "next"
  },
  "pnpm": {
    "overrides": {
      "@solidjs/signals": "0.13.7"
    }
  }
}
flush() mostly shows up in tests
One gotcha that started showing up for me was flush(). If you set signals and then immediately read from the system, you may not see what you expect yet because the writes are still batched.

import { createSignal, flush } from "solid-js";

const [a, setA] = createSignal(1);
const [b, setB] = createSignal(2);

setA(10);
setB(20);

// Neither has updated yet because both writes are still batched.
flush();
This has been most noticeable in unit tests for me. E2E tests usually keep working, but lower-level tests may suddenly need a flush() to observe the state you thought you had already updated.

Track before await
This has always been true, but it becomes much more obvious now that memos can be async. If you expect a signal read to track after an await, you are going to have a bad time. Pull the tracked values above the await instead.

import { createMemo, createSignal } from "solid-js";

// Bad: `count()` is read after the await boundary.
const [count, setCount] = createSignal(0);
const double = createMemo(async () => {
  await new Promise(r => setTimeout(r, 2000));
  return count() * 2;
});

// Good: track first, then await.
const [nextCount, setNextCount] = createSignal(0);
const nextDouble = createMemo(async () => {
  const c = nextCount();
  await new Promise(r => setTimeout(r, 2000));
  return c * 2;
});
Lastly, pay attention to Solid warnings in development. They are often the first sign of a subtle bug. Warnings like reactive read or written in owned scope are usually worth stopping and understanding rather than ignoring.

Conclusion
Overall, I have come away really impressed with Solid 2.0. The async model feels better, the reactivity changes feel thoughtful, and once you get your head around the new mental model it is a very nice upgrade.

The hard part is not really whether Solid 2.0 is worth it. It is more that early migrations come with ecosystem lag, rough edges, and a few easy-to-miss reactivity gotchas. Still, the migration feels very doable, and AI genuinely made the process faster.

If you have been trying the migration yourself, I would love to hear how it went. If you get stuck, feel free to join the Solid Discord.

https://www.brenelz.com/posts/solid-js-best-practices/

https://www.brenelz.com/posts/solid-js-best-practices/

Solid.js 1.0 Best Practices

Hello hello, I am back! It’s been awhile since my last blog post. Almost 6 months to be exact. I’ve been busy contributing to open source projects like SolidStart and TanStack Start, and writing less as a result.

That’s not why you’re here though. You’ve been playing with Solid and want to know what the actual best practices are. Below contains guidance on the things I see people stumble over the most. I’ve tried to keep it Solid-specific, rather than rehashing generic frontend fundamentals though plenty of those still apply.

Solid
Call functions when passing signals to JSX props
One of the first questions people ask when using Solid’s signals is whether to pass the signal itself or the value to JSX props.

function App() {
  const [id, setId] = createSignal(0);

  return (
    <>
      {/* Call the function */}
      <User id={id()} name="Brenley" />
      {/* or not */}
      <User id={id} name="Brenley" />
    </>
  );
}
One of Solid’s principles is that components shouldn’t need to care whether a prop came from a signal or not. If you don’t call the function, the receiving component now has to treat its props differently. There is a reason there is no isSignal helper.

For example:

function App() {
  const [id, setId] = createSignal(0);

  return <User id={id} name="Brenley" />;
}

function User(props: { id: Accessor<number>; name: string }) {
  return (
    <h1>
      {props.id()} – {props.name}
    </h1>
  );
}
This starts to feel awkward — one prop is reactive and the other isn’t. A cleaner approach is:

function App() {
  const [id, setId] = createSignal(0);

  return <User id={id()} name="Brenley" />;
}

function User(props: { id: number; name: string }) {
  return (
    <h1>
      {props.id} – {props.name}
    </h1>
  );
}
The way I think about this is that any signal read inside JSX becomes a dependency. That’s exactly where Solid expects reactivity to live. In this case, you want the JSX to track the id() call and update when id changes.

Don’t destructure props
In Solid, JSX props are reactive by default. Under the hood, they’re accessed through getters so that reading a prop inside JSX or a reactive scope automatically tracks dependencies.

This works great as long as those getters are allowed to flow through your code.

When you destructure props, you extract the property value instead of preserving the getter. That breaks the reactive connection.

For example:

function User(props: { name: string }) {
  const { name } = props;

  return <h1>{name}</h1>;
}
At first glance this looks fine, but the destructured name is no longer reactive. If the parent updates name, this component won’t update.

The correct version is to access props directly:

function User(props: { name: string }) {
  return <h1>{props.name}</h1>;
}
Here, the getter stays intact. Solid can track the read, and updates work as expected.

If you really need to destructure props, Solid provides splitProps, which preserves reactivity.

This is a common source of “why isn’t this updating?” bugs in Solid, and once you know how props work, it makes sense.

Use function wrappers when something needs to be reactive
In Solid, it’s important to be aware of where reactive tracking happens. Signals only create dependencies when they’re read inside a reactive scope such as JSX, createEffect, or createMemo.

The component body itself is not reactive. It runs once as setup code.

That means if you read a signal directly in the component body, it won’t update over time.

For example:

function App() {
  const [count, setCount] = createSignal(0);
  const doubled = count() * 2;
  console.log(doubled); // logged once, never updates
}
Even if setCount is called later, this will only log once. The signal was read outside of any reactive scope.

To make this reactive, we need two changes:

Delay the signal read by wrapping it in a function
Execute that function inside a reactive scope
Here’s the same example using a function wrapper and createEffect:

function App() {
  const [count, setCount] = createSignal(0);
  const doubled = () => count() * 2;
  createEffect(() => {
    console.log(doubled());
  });
}
Now the read happens inside the effect, so Solid can track the dependency and rerun the effect when count changes.

JSX expressions also create reactive scopes, so this works as well:

function App() {
  const [count, setCount] = createSignal(0);
  const doubled = () => count() * 2;

  return (
    <>
      <div>{doubled()}</div>
      <div>{doubled()}</div>
    </>
  );
}
We can improve this with one more change. Right now every time we call doubled() it will rerun the calculation. If it doesn’t change we can reuse the previous value. This is where createMemo comes in.

function App() {
  const [count, setCount] = createSignal(0);
  const doubled = createMemo(() => count() * 2);

  return (
    <>
      <div>{doubled()}</div>
      <div>{doubled()}</div>
    </>
  );
}
Use Solid’s control-flow components (<Show>, <For>)
When coming from React, a common pattern is to use JavaScript conditionals directly in JSX for rendering:

function App() {
  return <>{open() && <SidebarMenu />}</>;
}
This works in Solid, but it’s not the most idiomatic approach. Solid provides the <Show> control-flow component for a reason, and you should generally prefer it.

Here’s the equivalent using <Show>:

import { Show } from "solid-js";

function App() {
  return (
    <Show when={open()}>
      <SidebarMenu />
    </Show>
  );
}
The key difference is that <Show> gives you explicit control-flow that Solid can optimize and reason about. The when condition is tracked, and the children are only evaluated when that condition is truthy.

As conditions get more complex <Show> scales much better:

<Show when={open()} fallback={<EmptyState />}>
  <SidebarMenu />
</Show>
Beyond the technical benefits, I also find <Show> easier to read. It communicates “conditional rendering” directly, instead of relying on inline JavaScript expressions embedded in JSX.

Similar to conditionals, it’s tempting to reach for JavaScript array methods directly in JSX:

{
  items().map(item => <Item item={item} />);
}
In this case <For> is the better default.

<For each={items()}>{item => <Item item={item} />}</For>
<For> exists for a reason. It preserves item identity and updates the DOM with minimal recalculations when the list changes. This becomes especially important for dynamic or frequently updated lists.

Use createEffect sparingly
createEffect should mostly be used as a last resort. Solid 2.0 will further reduce the need for effects by making common patterns more declarative.

The following are common scenarios where people reach for createEffect when they shouldn’t.

Fetching async data
It’s tempting to fetch data inside an effect:

const [posts, setPosts] = createSignal([]);

createEffect(async () => {
  const data = await fetch("/api/posts").then(r => r.json());
  setPosts(data);
});
This has several problems. The effect runs after the component renders, so you get a flash of empty state. There’s no built-in error handling or loading state. And if dependencies change, you can end up with race conditions.

Instead, use createResource or createAsync (in SolidStart):

const [posts] = createResource(() => fetch("/api/posts").then(r => r.json()));
This gives you proper Suspense integration, automatic loading/error states, and handles race conditions correctly.

State synchronization
Another common anti-pattern is using effects to keep two pieces of state in sync:

const [firstName, setFirstName] = createSignal("John");
const [lastName, setLastName] = createSignal("Doe");
const [fullName, setFullName] = createSignal("");

createEffect(() => {
  setFullName(`${firstName()} ${lastName()}`);
});
This works, but it introduces unnecessary indirection. The relationship between fullName and the source signals is hidden inside an effect.

The derived version is simpler:

const [firstName, setFirstName] = createSignal("John");
const [lastName, setLastName] = createSignal("Doe");
const fullName = () => `${firstName()} ${lastName()}`;
The legitimate use case for createEffect is interacting with the outside world — especially the DOM or third-party libraries that Solid doesn’t control.

Derive as much as possible
This is another best practice that is pretty standard across frameworks just like the createEffect one above.

Solid builds a fine-grained dependency graph at runtime. When you derive values declaratively, Solid knows exactly what depends on what and can update only the minimal parts of your app when something changes.

The moment you reach for createEffect to keep values in sync, you step outside that graph.

For example, this is a common anti-pattern:

const [count, setCount] = createSignal(0);
const [double, setDouble] = createSignal(0);

createEffect(() => {
  setDouble(count() * 2);
});
This works, but it introduces manual synchronization and makes the relationship between count and double implicit.

The derived version is simpler and strictly better:

const [count, setCount] = createSignal(0);
const double = createMemo(() => count() * 2);
Use stores when dealing with complex objects
When dealing with complex or nested objects, stores are usually the right tool instead of signals.

Signals are designed for primitive values or simple references. When you store an object in a signal and update it, you replace the entire object. That means everything reading the signal is notified, even if only a small part of the object changed.

Stores, on the other hand, provide fine-grained reactivity at the property level. You can update a nested property, and only the parts of your UI that depend on that specific property will re-render.

For example, prefer this:

const [board, setBoard] = createStore({
  boards: ["Board 1", "Board 2"],
  notes: ["Note 1", "Note 2"],
});
over this:

const [board, setBoard] = createSignal({
  boards: ["Board 1", "Board 2"],
  notes: ["Note 1", "Note 2"],
});
The difference becomes clear when you update. With a signal, you need to replace the whole object:

// Signal approach - replaces entire object
setBoard({
  ...board(),
  notes: [...board().notes, "Note 3"],
});
With a store, you can update just what changed using mutable reactivity.

// Store approach - updates only the notes array
setBoard(notes => notes.push("Note 3"));
This granularity matters for performance. In the signal version, any component reading board() will re-evaluate when anything changes. In the store version, a component reading board.notes won’t react to changes in board.boards.

Another benefit is that stores are deeply reactive by default. If you access a nested property in JSX, Solid tracks that specific path:

function App() {
  const [board, setBoard] = createStore({
    settings: { theme: "light" },
    notes: [],
  });

  return (
    <>
      {/* Only re-renders when theme changes */}
      <ThemeDisplay theme={board.settings.theme} />
      {/* Only re-renders when notes change */}
      <NotesList notes={board.notes} />
    </>
  );
}
SolidStart
Don’t await in preload functions (loaders)
Something different about SolidStart than other frameworks is the concept of preloading instead of loaders.

A preload function is not meant to resolve data; it’s meant to start the work as early as possible.

SolidStart encourages not awaiting in preload and letting the component handle suspending and resolving the promise for you.

For example:

export const route = {
  preload: () => getPosts(),
} satisfies RouteDefinition;

export default function Page() {
  const posts = createAsync(() => getPosts());
}
Here we are just warming up the getPosts() cache. As soon as navigation starts, getPosts() begins executing. By the time the component renders, the promise may already be resolved or at least partially complete.

Use queries/server functions to get data from the server
The preload pattern above works because of the concept of queries.

In SolidStart, the query function wraps a server function with a unique key, enabling request deduplication, caching, and cache invalidation. Here’s a simple example:

const getPosts = query(async () => {
  "use server";
  return await db.from("posts").select();
}, "posts");
Use actions to mutate data
When it comes to mutating server data, actions are the missing half of queries.

If you use the concept of actions, revalidating data becomes so easy.

const addPost = action(async (formData: FormData) => {
  "use server";

  const post = await db
    .from("posts")
    .insert({ title: formData.get("title") })
    .select()
    .single();

  throw redirect(`/posts/${post.id}`);
}, "addPost");
When an action completes, SolidStart knows that server state has changed. If you redirect after the mutation, the router automatically revalidates all queries for the next route.

Because both queries and actions are keyed, you can also opt into fine-grained control by revalidating specific keys instead of the entire route when needed.

Conclusion
These are the Solid best practices that come up for me most often — especially when helping people move from React.

If there’s one theme running through all of this, it’s to work with Solid’s reactivity model, not against it. Keep your getters intact, derive instead of sync, and let the framework do the heavy lifting.

If you have questions or want to share your own tips, find me on Twitter or drop by the Solid Discord. Happy coding!

import { createSignal, lazy, onCleanup } from "solid-js";
import { render } from "solid-js/web";
import AsyncChild from "./AsyncChild";

const LazyChild = lazy(() => import("./SimpleChild"));
const LazyAsyncChild = lazy(() => import("./AsyncChild"));

const Loader = () => {
  const [count, setCount] = createSignal(0),
    interval = setInterval(() => setCount((c) => c + 1), 1000);
  onCleanup(() => clearInterval(interval));

  return <div>Loading... {count()}</div>;
};

const App = () => {
  const startTime = Date.now();

  return (
    <Suspense fallback={<Loader />}>
      <AsyncChild start={startTime} />
      <AsyncChild start={startTime} />
      <AsyncChild start={startTime} />
      <AsyncChild start={startTime} />
      <LazyChild start={startTime} />
      <LazyAsyncChild start={startTime} />
    </Suspense>
  );
};

render(App, document.getElementById("app"));
// renderToString(App).then(console.log);
import { createSignal } from "solid-js";
import { render } from "solid-js/web";
import { Transition, TransitionGroup } from "solid-transition-group";

function shuffle(array) {
  return array.sort(() => Math.random() - 0.5);
}
let nextId = 10;

const App = () => {
  const [show, toggleShow] = createSignal(true),
    [select, setSelect] = createSignal(0),
    [numList, setNumList] = createSignal([1, 2, 3, 4, 5, 6, 7, 8, 9]),
    randomIndex = () => Math.floor(Math.random() * numList().length);

  return (
    <>
      <button onClick={() => toggleShow(!show())}>
        {show() ? "Hide" : "Show"}
      </button>
      <br />
      <b>Transition:</b>
      <Transition name="slide-fade">
        {show() && (
          <div>
            Lorem ipsum dolor sit amet, consectetur adipiscing elit. Mauris
            facilisis enim libero, at lacinia diam fermentum id. Pellentesque
            habitant morbi tristique senectus et netus.
          </div>
        )}
      </Transition>
      <br />
      <b>Animation:</b>
      <Transition name="bounce">
        {show() && (
          <div>
            Lorem ipsum dolor sit amet, consectetur adipiscing elit. Mauris
            facilisis enim libero, at lacinia diam fermentum id. Pellentesque
            habitant morbi tristique senectus et netus.
          </div>
        )}
      </Transition>
      <br />
      <b>Custom JS:</b>
      <Transition
        onBeforeEnter={(el) => (el.style.opacity = 0)}
        onEnter={(el, done) => {
          const a = el.animate([{ opacity: 0 }, { opacity: 1 }], {
            duration: 600
          });
          a.finished.then(done);
        }}
        onAfterEnter={(el) => (el.style.opacity = 1)}
        onExit={(el, done) => {
          const a = el.animate([{ opacity: 1 }, { opacity: 0 }], {
            duration: 600
          });
          a.finished.then(done);
        }}
      >
        {show() && (
          <div>
            Lorem ipsum dolor sit amet, consectetur adipiscing elit. Mauris
            facilisis enim libero, at lacinia diam fermentum id. Pellentesque
            habitant morbi tristique senectus et netus.
          </div>
        )}
      </Transition>
      <br />
      <b>Switch OutIn</b>
      <br />
      <button onClick={() => setSelect((select() + 1) % 3)}>Next</button>
      <Transition name="fade" mode="outin">
        <Switch>
          <Match when={select() === 0}>
            <p class="container">The First</p>
          </Match>
          <Match when={select() === 1}>
            <p class="container">The Second</p>
          </Match>
          <Match when={select() === 2}>
            <p class="container">The Third</p>
          </Match>
        </Switch>
      </Transition>
      <b>Group</b>
      <br />
      <button
        onClick={() => {
          const list = numList(),
            idx = randomIndex();
          setNumList([...list.slice(0, idx), nextId++, ...list.slice(idx)]);
        }}
      >
        Add
      </button>
      <button
        onClick={() => {
          const list = numList(),
            idx = randomIndex();
          setNumList([...list.slice(0, idx), ...list.slice(idx + 1)]);
        }}
      >
        Remove
      </button>
      <button
        onClick={() => {
          const randomList = shuffle(numList().slice());
          setNumList(randomList);
        }}
      >
        Shuffle
      </button>
      <br />
      <TransitionGroup name="list-item">
        <For each={numList()}>{(r) => <span class="list-item">{r}</span>}</For>
      </TransitionGroup>
    </>
  );
};

render(App, document.getElementById("app"));
import { createSignal, createResource } from "solid-js";
import { render } from "solid-js/web";

const fetchUser = async (id) =>
  (await fetch(`https://swapi.dev/api/people/${id}/`)).json();

const App = () => {
  const [userId, setUserId] = createSignal(),
    [user] = createResource(userId, fetchUser);

  return (
    <>
      <input
        type="number"
        min="1"
        placeholder="Enter Numeric Id"
        onInput={(e) => setUserId(e.target.value)}
      />
      <span>{user.loading && "Loading..."}</span>
      <div>
        <pre>{JSON.stringify(user(), null, 2)}</pre>
      </div>
    </>
  );
};

render(App, document.getElementById("app"));


