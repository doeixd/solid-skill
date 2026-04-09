---
name: solid-js-1.x-best-practices-and-api
description: Guidance for writing, reviewing, and refactoring SolidJS 1.x code using idiomatic APIs and reactivity patterns. Use this whenever the user is working in SolidJS 1.x and asks for help with components, signals, props, memos, effects, stores, control flow, or why a Solid component is not updating correctly. Also use it for code reviews or refactors of Solid 1.x code, especially when React habits may be causing non-idiomatic Solid patterns.
---

# SolidJS 1.x Best Practices And API

Use this skill for SolidJS 1.x core framework work. Keep it focused on idiomatic Solid usage, not SolidStart-specific routing or server patterns.

## Goals

- Preserve Solid 1.x reactivity instead of accidentally breaking it with destructuring or early reads.
- Prefer declarative derivation over effect-driven synchronization.
- Use the Solid control-flow and state primitives that match the problem.
- Explain recommendations in terms of Solid's fine-grained reactivity model.

## Working style

Start by identifying whether the user needs one of these modes:

- explanation of a Solid 1.x concept
- debugging of stale or non-updating UI
- refactor toward more idiomatic Solid 1.x code
- code review of a Solid component or state pattern

When reviewing or editing code, prefer minimal changes that preserve behavior.

Default to explaining the smallest Solid-specific reason a pattern is wrong. Avoid turning a local fix into a large rewrite unless the user asked for that.

## Core mental model

- Components run once for setup.
- Reactive tracking happens when signals or reactive props are read inside JSX, `createMemo`, `createEffect`, and similar reactive scopes.
- If a read happens too early, outside tracking, it will not stay live.
- Props are reactive by default in Solid 1.x, so preserve their getters.

Use that model to explain bugs and justify fixes.

## API surface map

Use the smallest primitive that matches the job.

### `createSignal`

What it is:

- the default primitive for local scalar state and simple references

How to use it:

- read with `count()`
- write with `setCount(next)` or `setCount(prev => next)`

Why to use it:

- explicit, cheap, easy to reason about

Do not use it as the default container for large nested objects if you want fine-grained nested updates.

### `createMemo`

What it is:

- cached derived state

How to use it:

- derive values from signals, props, or stores
- read with `memo()`

Why to use it:

- expresses dependency relationships directly
- avoids recomputing expensive or reused derivations

Do not use it to perform side effects or write state.

### `createEffect`

What it is:

- a reactive side-effect boundary

How to use it:

- read reactive values inside it
- interact with DOM, console, browser APIs, or third-party libraries
- clean up with `onCleanup`

Why to use it:

- bridges Solid's reactive graph to outside systems

Do not use it as your default way to derive or synchronize state.

If a user reaches for `createEffect` first, check whether they actually need a memo, resource, store helper, or just a JSX expression.

### `createResource`

What it is:

- Solid 1.x async data primitive for fetch-like workflows

How to use it:

- model request state declaratively
- pair it with `Suspense` for loading UI

Why to use it:

- handles loading and reactivity better than ad-hoc async effects

Prefer it over `createEffect(async () => ...)` for UI data loading.

### `batch`

What it is:

- a reactive utility for grouping synchronous updates

How to use it:

- batch multiple setter calls when you need downstream computations to see the combined update

Why to use it:

- reduces unnecessary intermediate work in 1.x synchronous reactivity

Use it intentionally, not as a vague performance ritual.

### `createStore`

What it is:

- fine-grained reactive state for nested objects and arrays

How to use it:

- read properties directly, like `store.user.name`
- update with targeted setter paths or store updates

Why to use it:

- avoids replacing entire object graphs when only one branch changed

Prefer it when data shape matters more than single-value identity.

If updates are mostly whole-object replacement and nested granularity does not matter, a signal may still be simpler.

### `produce`, `reconcile`, and `unwrap`

What they are:

- common store helpers for nested updates, reconciliation, and conversion to plain values

How to use them:

- `produce` for ergonomic nested mutations
- `reconcile` when syncing store state to incoming object graphs
- `unwrap` when a plain non-reactive value is required

Why to use them:

- they solve common store workflows without abandoning fine-grained updates

### `splitProps` and `mergeProps`

What they are:

- prop helpers that preserve reactive access patterns

How to use them:

- `splitProps` when you need local and rest groupings without breaking reactivity
- `mergeProps` for layered defaults and overrides

Why to use them:

- they are safer than ad-hoc destructuring for reactive props

### `Show`

What it is:

- conditional rendering primitive

How to use it:

- `when={condition}`
- optional `fallback={...}`

Why to use it:

- clearer and more Solid-native than scattered `&&` rendering

### `For`

What it is:

- list rendering primitive

How to use it:

- `each={items()}`
- render each item through the child callback

Why to use it:

- preserves identity and updates lists efficiently

Prefer it over `.map(...)` in dynamic UI code.

### `Index`

What it is:

- index-oriented list rendering when item identity is stable by position

How to use it:

- use it when the list shape is stable and per-index access matters more than keyed identity

Why to use it:

- avoids the wrong mental model when `For` is not the best fit

Do not blindly replace every list with `Index`; choose based on update behavior.

### `Suspense` and `ErrorBoundary`

What they are:

- async readiness and error boundaries for UI subtrees

How to use them:

- pair `Suspense` with async reads such as `createResource`
- use `ErrorBoundary` where async or rendering failures should be isolated

Why to use them:

- they provide clearer async and failure handling than custom boolean flags scattered through the tree

Do not invent parallel `loading`, `error`, and `ready` booleans if the framework boundary already models the behavior better.

### `lazy`

What it is:

- code-splitting for components

How to use it:

- lazy-load subtree components and usually render them under `Suspense`

Why to use it:

- keeps initial bundles smaller without changing the component model

### `onCleanup`

What it is:

- cleanup registration for reactive scopes

How to use it:

- tear down timers, listeners, subscriptions, and external resources created inside effects or component-owned scopes

Why to use it:

- prevents leaks and stale subscriptions

### `untrack`

What it is:

- escape hatch for intentionally non-reactive reads

How to use it:

- wrap reads that should happen once without subscribing the current scope

Why to use it:

- makes non-tracking intent explicit instead of accidental

Use it narrowly. If a value should stay live, `untrack` is the wrong tool.

### `createContext` and `useContext`

What they are:

- dependency injection for shared subtree state

How to use them:

- provide stable shared values near the tree root that needs them
- consume with `useContext`

Why to use them:

- avoids prop drilling for shared cross-cutting state

Prefer context for stable shared capabilities, not as a blanket replacement for all local state.

### refs

What they are:

- direct access to DOM nodes or child handles

How to use them:

- use `ref={el}` patterns when imperative DOM access is actually needed

Why to use them:

- integrates DOM libraries and focus or measurement workflows

Prefer normal declarative rendering unless imperative DOM work is necessary.

## Best practices

### Pass values to JSX props unless you intentionally want an accessor API

Default to calling signals when passing them as props:

```tsx
<User id={id()} name="Brenley" />
```

Prefer this over passing `id` directly, because most components should not need to care whether a prop originated from a signal. If a component intentionally accepts an accessor, say so explicitly and keep that choice narrow.

The main idea is that JSX is where Solid expects dependency tracking to happen. If the parent reads `id()` in JSX, Solid tracks that read naturally and the child can accept a normal value-shaped prop.

Prefer:

```tsx
function App() {
  const [id] = createSignal(0);
  return <User id={id()} name="Brenley" />;
}

function User(props: { id: number; name: string }) {
  return <h1>{props.id} - {props.name}</h1>;
}
```

Avoid making ordinary component props accessor-shaped without a reason.

### Do not destructure reactive props by default

This often breaks reactivity:

```tsx
function User(props: { name: string }) {
  const { name } = props;
  return <h1>{name}</h1>;
}
```

Prefer direct property access:

```tsx
function User(props: { name: string }) {
  return <h1>{props.name}</h1>;
}
```

If destructuring is truly needed, prefer helpers that preserve reactivity such as `splitProps`.

The underlying reason is that Solid props are getter-based. Direct property access preserves the getter. Plain destructuring usually extracts the current value and loses the reactive connection.

### Keep reactive reads inside reactive scopes

If a value should update over time, do not compute it once in component setup and expect it to stay current.

Prefer:

```tsx
const doubled = createMemo(() => count() * 2);
```

or:

```tsx
const doubled = () => count() * 2;
```

Then read it inside JSX or another reactive scope.

Remember that the component body is setup code, not a tracked render loop.

If the user wants a reactive value, two things need to be true:

1. delay the read with a function or memo
2. execute that read inside JSX or another reactive scope

### Preserve prop reactivity when reshaping props

If you need to separate props into local and remaining groups, prefer `splitProps` over object destructuring. If you need defaults, prefer `mergeProps` over eager destructuring plus fallback values.

### Prefer `createMemo` for reusable derived values

Use plain wrappers like `() => count() * 2` for trivial one-off derivations. Use `createMemo` when:

- the computation is reused
- the computation is non-trivial
- stable caching improves clarity or performance

Do not introduce `createMemo` for every tiny expression without a reason.

Good rule of thumb:

- wrapper function for cheap one-off derivation
- `createMemo` when the value is reused or meaningfully benefits from caching

### Prefer derivation over synchronization

If one value can be computed from another, derive it.

Prefer:

```tsx
const fullName = () => `${firstName()} ${lastName()}`;
```

or:

```tsx
const fullName = createMemo(() => `${firstName()} ${lastName()}`);
```

Avoid creating a second writable signal only to mirror the first.

### Use `createEffect` sparingly

`createEffect` is mainly for synchronizing with the outside world, such as:

- DOM APIs
- browser APIs
- logging
- third-party libraries

Avoid effects for:

- data fetching that should use `createResource`
- syncing one piece of state into another when a memo or derived function would do

When you see an effect that just copies state, try to replace it with derived state.

In practice, many Solid bugs come from treating `createEffect` like a default orchestration primitive instead of a bridge to non-Solid systems.

Common anti-pattern:

```tsx
const [firstName, setFirstName] = createSignal("John");
const [lastName, setLastName] = createSignal("Doe");
const [fullName, setFullName] = createSignal("");

createEffect(() => {
  setFullName(`${firstName()} ${lastName()}`);
});
```

Prefer:

```tsx
const fullName = createMemo(() => `${firstName()} ${lastName()}`);
```

For async reads, prefer `createResource` over `createEffect(async () => ...)` so loading, errors, and cancellation behavior stay aligned with Solid's async model.

### Prefer Solid control-flow components

Default to Solid's control-flow primitives when they clarify intent:

- `<Show>` for conditional rendering
- `<For>` for list rendering

These are generally a better default than ad-hoc `&&` rendering or `.map(...)` inside JSX, especially for frequently changing lists.

Use `Index` when list positions are stable and index-based retention is the real goal.

Prefer:

```tsx
<Show when={open()} fallback={<EmptyState />}>
  <SidebarMenu />
</Show>
```

and:

```tsx
<For each={items()}>{item => <Item item={item} />}</For>
```

These primitives communicate intent directly and let Solid optimize the structure it controls.

### Use stores for complex nested state

For primitives and simple references, signals are often enough.

For nested objects or arrays that change in fine-grained ways, prefer `createStore` so updates can stay targeted instead of replacing entire objects.

If incoming server or parent data should be merged into an existing store shape, consider `reconcile` rather than ad-hoc nested replacement logic.

Signals are still fine for primitives or simple references. Reach for stores when property-level reactivity is the real requirement.

Prefer store-style updates over whole-object replacement when only one branch changed.

### Keep context values stable and intentional

If you provide context, prefer passing a stable object of capabilities or state rather than recreating ad-hoc provider values every render. Context should clarify ownership and sharing boundaries.

### Use refs and effects only for real imperative work

If the job is focus management, measuring layout, integrating a library, or attaching listeners, refs plus effects are appropriate. If the job is ordinary rendering, keep it declarative.

## Primitive selection guide

Choose primitives with this bias:

1. `createSignal` for local scalar state
2. derived function or `createMemo` for computed values
3. `createStore` for nested structured state
4. `createResource` plus `Suspense` for async data loading
5. `createEffect` only when touching the outside world
6. `Show`, `For`, and `Index` for UI control flow
7. `splitProps` and `mergeProps` when reshaping props
8. context only for genuinely shared subtree concerns

## Core heuristics

Keep these ideas in the foreground:

1. Keep getters intact.
2. Derive instead of synchronizing.
3. Let JSX and reactive scopes own dependency tracking.
4. Use effects for the outside world, not for routine data flow.
5. Use stores when the shape of nested data matters.

## Debugging checklist

When a Solid 1.x component is not updating correctly, check these first:

1. Was a signal or prop read outside a reactive scope?
2. Were props destructured in a way that broke the getter?
3. Was an accessor passed where a plain value should have been, or vice versa?
4. Is `createEffect` being used to mirror state that should be derived?
5. Should `splitProps` or `mergeProps` be used instead of destructuring?
6. Would `createStore` better fit the shape of the state?
7. Was `untrack` or `batch` used intentionally, or is it masking the real problem?

## Output expectations

When helping the user, prefer this structure:

1. State the Solid-specific issue.
2. Explain why it happens in Solid 1.x.
3. Show the minimal corrected code.
4. Mention any tradeoff only if it materially affects the choice.

## Example prompts this skill should handle well

- "Why does this Solid component stop updating when I destructure props?"
- "Can you refactor this SolidJS component to be more idiomatic?"
- "I think I am overusing createEffect in this Solid app."
- "Review this Solid 1.x component and point out reactivity mistakes."

## Additional resources

- Solid 1.x best practices: https://www.brenelz.com/posts/solid-js-best-practices/
- Solid docs: https://docs.solidjs.com/
- Reactivity concepts: https://docs.solidjs.com/concepts/intro-to-reactivity
