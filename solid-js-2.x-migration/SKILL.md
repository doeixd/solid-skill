---
name: solid-js-2.x-migration
description: Guidance for migrating an existing SolidJS 1.x codebase to SolidJS 2.x beta or next, including identifying old APIs, rewriting code to the new semantics, sequencing changes safely, and validating the migration with warnings and tests. Use this whenever the user asks to upgrade, port, convert, or migrate SolidJS code from 1.x to 2.x, especially when they mention renamed imports, removed APIs like Index or createResource, new effect behavior, Loading or Errored, store API changes, failing tests after upgrade, or ecosystem breakage during the move to Solid 2.x beta or next.
---

# SolidJS 2.x Migration

Use this skill when the user already has Solid 1.x code and wants to move it to Solid 2.x beta or `next`.

This skill is migration-oriented. Its job is not just to explain the new APIs, but to convert old code safely and preserve behavior where possible.

If the user only wants a narrow explanation of a Solid 2.x primitive or warning and is not actively upgrading existing code, prefer the Solid 2.x API skill instead.

## Goals

- Identify likely 1.x patterns quickly.
- Apply the highest-confidence migrations first.
- Avoid shallow renames that miss semantic changes.
- Use tests and dev warnings to validate the migration.
- Be honest about beta-era ecosystem gaps and package workarounds.

## Scope boundary

- Use this skill for upgrades, ports, conversions, staged migration plans, and debugging breakage that appeared during or after the upgrade.
- Keep the focus on moving existing code safely, not on giving a general tour of the 2.x API surface.
- When a task is primarily Start-specific, use the SolidStart skill alongside this one rather than assuming the framework boundary from generic Solid guidance.

## Migration mindset

Treat the migration as two kinds of work:

- mechanical changes that are mostly straightforward
- semantic changes that need careful reasoning and validation

Do not present everything as equally safe. Call out where a rewrite is mechanical versus where behavior may change.

Do not invent replacement APIs that are not part of the Solid 2 migration guidance in this skill. If a candidate rewrite depends on neighboring framework helpers or uncertain beta exports, say that explicitly and keep the migration on the highest-confidence Solid core path.

## Recommended migration workflow

1. Inspect package versions, imports, and the current usage of 1.x APIs.
2. Make import and package-boundary changes.
3. Migrate removed or renamed APIs.
4. Migrate behavior-sensitive reactivity patterns.
5. Run tests or at least read dev warnings carefully.
6. Fix remaining ecosystem or tooling issues.

## High-confidence mechanical migrations

Apply these first when present:

- `solid-js/web` -> `@solidjs/web`
- `solid-js/store` imports -> `solid-js`
- `solid-js/h` -> `@solidjs/h`
- `solid-js/html` -> `@solidjs/html`
- `solid-js/universal` -> `@solidjs/universal`
- `Suspense` -> `Loading`
- `ErrorBoundary` -> `Errored`
- `Index` -> `For keyed={false}`
- `createSelector` -> `createProjection` or function-form `createStore`
- `unwrap` -> `snapshot`
- `mergeProps` -> `merge`
- `splitProps` -> `omit`
- `classList` -> `class`
- `use:` directives -> `ref` directive factories
- `attr:` and `bool:` -> standard attribute behavior
- `oncapture:` removed
- `onMount` -> `onSettled`

Even for mechanical migrations, quickly verify surrounding semantics instead of blindly rewriting.

Also remember the non-trivial removals:

- `createResource` is not a simple rename; migrate toward async computations plus `Loading`
- `startTransition` and `useTransition` go away; express pending and transition UX with built-in transitions, `isPending`, `Loading`, and optimistic primitives
- `createComputed` is removed; replace it based on intent rather than by pattern-matching syntax

## Canonical before and after rewrites

Keep rewrites terse and behavior-aware.

### Imports

```ts
// 1.x
import { render } from "solid-js/web";
import { createStore } from "solid-js/store";

// 2.x beta
import { render } from "@solidjs/web";
import { createStore } from "solid-js";
```

### `Index` to `For`

```tsx
// 1.x
<Index each={items()}>{(item, i) => <Row item={item()} index={i} />}</Index>

// 2.x beta
<For each={items()} keyed={false}>{(item, i) => <Row item={item()} index={i()} />}</For>
```

The migration is not just renaming the component. The index is now also an accessor.

### `Suspense` to `Loading`

```tsx
// 1.x
<Suspense fallback={<Spinner />}>
  <Page />
</Suspense>

// 2.x beta
<Loading fallback={<Spinner />}>
  <Page />
</Loading>
```

### `ErrorBoundary` to `Errored`

```tsx
// 1.x
<ErrorBoundary fallback={(err, reset) => <Fallback err={err} reset={reset} />}>
  <Page />
</ErrorBoundary>

// 2.x beta
<Errored fallback={(err, reset) => <Fallback err={err} reset={reset} />}>
  <Page />
</Errored>
```

### `onMount` to `onSettled`

```ts
// 1.x
onMount(() => {
  measureLayout();
});

// 2.x beta
onSettled(() => {
  measureLayout();
});
```

Treat this as a lifecycle semantic review, not a blind search-and-replace, especially when old mount logic created nested primitives or assumed immediate post-setter reads.

### Single-callback effect to split effect

```ts
// 1.x
createEffect(() => {
  document.title = title();
});

// 2.x beta
createEffect(
  () => title(),
  value => {
    document.title = value;
  }
);
```

### Cleanup moves to apply side

```ts
// 1.x
createEffect(() => {
  const id = setInterval(() => console.log(name()), 1000);
  onCleanup(() => clearInterval(id));
});

// 2.x beta
createEffect(
  () => name(),
  value => {
    const id = setInterval(() => console.log(value), 1000);
    return () => clearInterval(id);
  }
);
```

### Top-level prop read fix

```tsx
// 1.x style that now warns
function Title(props) {
  const t = props.title;
  return <h1>{t}</h1>;
}

// 2.x beta
function Title(props) {
  return <h1>{props.title}</h1>;
}
```

### Store helper changes

```ts
// 1.x
const plain = unwrap(store);
const merged = mergeProps(defaults, overrides);
const [local, rest] = splitProps(props, ["class"]);

// 2.x beta
const plain = snapshot(store);
const merged = merge(defaults, overrides);
const rest = omit(props, "class");
```

Important semantic change:

- with `merge`, `undefined` is an explicit override value, not a skipped key

### `createSelector` to projection

```ts
// 1.x
const isSelected = createSelector(selectedId);

// 2.x beta
const selected = createProjection((draft) => {
  const id = selectedId();
  draft[id] = true;
  if (draft._prev != null) delete draft[draft._prev];
  draft._prev = id;
}, {});
```

If the old code used selector-like patterns, think in terms of projection-oriented derived state rather than hunting for a one-to-one selector replacement.

### Draft-first store update

```ts
// 1.x style
setStore("user", "address", "city", "Paris");

// 2.x preferred
setStore(s => {
  s.user.address.city = "Paris";
});

// 2.x compatibility bridge
setStore(storePath("user", "address", "city", "Paris"));
```

Setter return values also matter now:

```ts
setStore(s => {
  return { ...s, list: [] };
});
```

That is a top-level shallow replacement or diff, not the same thing as mutating the draft in place.

## Semantics that need careful migration

### Batching and test behavior

In 2.x, reads do not immediately observe just-written signal values. If tests or imperative code read state immediately after setters, they may now need a flush point.

Use `flush()` narrowly, especially in tests or imperative DOM code. Do not scatter it everywhere as a generic fix.

```ts
const [count, setCount] = createSignal(0);
setCount(1);
flush();
expect(count()).toBe(1);
```

The practical insight from early migrations: this tends to show up in unit tests far more than end-to-end tests.

For imperative DOM reads, `flush()` is also the replacement for many old `batch(...)` expectations.

### `createEffect`

Migrate from single-callback effects to compute/apply form when the effect depends on reactive reads and performs side effects.

When migrating:

- move tracked reads into the compute half
- move side effects into the apply half
- return cleanup from the apply half

If the old effect was really deriving state, replace it with a memo or derived primitive instead of translating it directly.

This is one of the biggest migration traps. Do not reward old effect-heavy code by preserving the shape if the real issue is that the code should be declarative.

### `onMount` to `onSettled`

Replace `onMount` with `onSettled`, but verify intent. Some code assumed old lifecycle timing or mixed mount work with nested primitive creation. Treat these cases carefully.

### `Context.Provider` to context-as-provider

```tsx
// 1.x
<ThemeContext.Provider value="dark">
  <Page />
</ThemeContext.Provider>

// 2.x beta
<ThemeContext value="dark">
  <Page />
</ThemeContext>
```

### `createComputed` replacement

Choose the replacement by intent:

- side effects -> split `createEffect`
- derived writable state -> function-form `createSignal` or `createStore`
- readonly derivation -> `createMemo`

```ts
// 1.x
createComputed(() => {
  setValue(props.input);
});

// 2.x beta, writable derived state
const [value, setValue] = createSignal(() => props.input);
```

If the old `createComputed` was really doing side effects, migrate to split `createEffect` instead.

### Ownership and root lifetime

In 2.x, roots created under an owner are owned by that parent by default. If old code depended on effectively detached lifetime, make the detachment explicit.

```ts
const singleton = runWithOwner(null, () => {
  const [value, setValue] = createSignal(0);
  return { value, setValue };
});
```

### Top-level reads and prop destructuring

2.x warns on top-level reactive reads. Migration often requires:

- moving reads into JSX, memos, or effect compute functions
- avoiding destructuring reactive props at component top level
- using `untrack` only when a one-time read is truly intentional

This also applies to control-flow callback bodies. If old code read reactive values in `Show` or `For` callback bodies before returning JSX, move those reads into JSX expressions or a proper reactive scope.

### Writes in owned scope

Old patterns that wrote state from memos or tracked scopes should usually be redesigned rather than patched. Prefer derived state or event-driven writes.

`pureWrite: true` is for narrow internal cases, not for silencing application-state warnings during migration.

### Async data

Do not just rename `Suspense` to `Loading` and stop there. If the app relied on `createResource`, migrate toward async computations plus `Loading`, pending indicators, and explicit refresh patterns.

Also distinguish initial loading from refresh state:

- `Loading` for not-ready-yet
- `isPending` for updating-again
- `latest` when stale-while-refreshing output should remain visible

Reasonable 2.x migration targets include:

- `createMemo(async () => ...)` for async derived values
- `createStore(fn, initial)` or `createProjection(...)` for async derived collections or keyed reconciled results

For selector-heavy or keyed-list code, `createProjection(...)` is often the more faithful migration target.

```ts
// 1.x
const [user] = createResource(id, fetchUser);

// 2.x beta
const user = createMemo(async () => {
  const currentId = id();
  return fetchUser(currentId);
});
```

Then move loading UX to structure:

```tsx
<Loading fallback={<Spinner />}>
  <Profile user={user()} />
</Loading>
```

Do not recreate `resource.loading` flags mechanically. If the old UI wanted stale-while-refreshing behavior, model that with `isPending(() => expr)` and optionally `latest(...)`.

```ts
const listPending = () => isPending(() => users() || posts());
```

If the old app used `startTransition` or `useTransition` for mutation UX, move that logic toward `action(...)`, optimistic primitives, and `refresh(...)`.

```ts
const [todos] = createStore(() => api.getTodos(), []);
const [optimisticTodos, setOptimisticTodos] = createOptimisticStore(() => todos(), []);

const addTodo = action(function* (todo) {
  setOptimisticTodos(list => {
    list.push(todo);
  });
  yield api.addTodo(todo);
  refresh(todos);
});
```

### Stores

Prefer draft-first store updates in 2.x. If path-style setters are deeply woven into the codebase, `storePath(...)` can help stage the migration without rewriting every update immediately.

Use `storePath(...)` as a bridge, not as the main style to promote once the migration settles.

## Ecosystem and tooling caveats

Solid 2.x beta or `next` migrations may require temporary compatibility work, especially around:

- testing libraries
- package aliases
- SSR bundling
- deduping local Solid versions
- package versions pinned to `next`

If a dependency has not caught up, say so explicitly and separate framework migration work from ecosystem breakage.

Useful migration-era insights from real projects:

- some packages may still require aliasing or temporary config surgery
- test tooling may need `solid-js/web` aliased to `@solidjs/web`
- linked local packages may need dedupe settings to avoid multiple Solid copies
- `next` tags and temporary overrides may be the practical path during beta adoption
- development warnings are often your first accurate signal that a migration is semantically wrong, not just syntactically incomplete
- deep observation is no longer something to stumble into casually; if the old code expected "watch everything" behavior, it may need an explicit `deep(store)` read
- loading and mutation state are less primitive-specific now, so many old `.loading` and transition-wrapper patterns should collapse into `Loading`, `isPending`, actions, and optimistic primitives
- old code that accidentally relied on unowned lifetime may start behaving more sanely in 2.x, but singleton-style integrations should now detach explicitly with `runWithOwner(null, ...)`
- writes-under-scope and strict-read warnings are usually design feedback, not noise to suppress

## When to pause and ask the user

Ask for confirmation when:

- a change appears behavior-affecting rather than mechanical
- a dependency is not Solid 2.x compatible yet
- there are multiple plausible migration strategies with real tradeoffs

Otherwise, keep moving through the migration.

## Output expectations

When migrating code, prefer this structure:

1. List the 1.x patterns found.
2. Separate mechanical changes from semantic rewrites.
3. Apply minimal safe code changes.
4. Note any remaining warnings, test failures, or ecosystem blockers.

When possible, include one-line reasoning beside each migration hotspot so the user understands why the rewrite happened.

For code rewrites, keep these guardrails:

- do not switch to unrelated helpers like `createAsync` or `mount` unless the local repo proves they are the right target
- prefer the highest-confidence Solid core migration path from this skill over neighboring framework patterns
- if the exact beta export is uncertain, say so explicitly instead of fabricating a confident rewrite

## Example prompts this skill should handle well

- "Upgrade this Solid 1 app to Solid 2 beta."
- "Can you migrate these components away from createResource, Suspense, and Index?"
- "My tests started failing after moving to Solid 2 next."
- "What is the safest order to migrate this Solid codebase to 2.x?"

## Additional resources

- Solid 2.0 migration guide: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/MIGRATION.md
- Solid 2.0 docs index: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/README.md
- Reactivity, batching, and effects RFC: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/01-reactivity-batching-effects.md
- Stores RFC: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/04-stores.md
- Async data RFC: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/05-async-data.md
- Actions and optimistic RFC: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/06-actions-optimistic.md
- DOM RFC: https://github.com/solidjs/solid/blob/next/documentation/solid-2.0/07-dom.md
