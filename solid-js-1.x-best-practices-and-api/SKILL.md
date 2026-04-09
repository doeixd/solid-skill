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

## How to use this skill

Read this skill in passes instead of treating every section as equally important.

1. Start with `Core mental model` and `Primitive selection guide`.
2. Use `Core best practices` when writing or refactoring ordinary component code.
3. Use `Specialized rule packs` and `Task-focused rule selection` when the task is about testing, accessibility, web components, or performance work.
4. Use `Common mistakes to catch` and `Debugging checklist` for code review or bug triage.

## Core mental model

- Components run once for setup.
- Reactive tracking happens when signals or reactive props are read inside JSX, `createMemo`, `createEffect`, and similar reactive scopes.
- If a read happens too early, outside tracking, it will not stay live.
- Props are reactive by default in Solid 1.x, so preserve their getters.

Use that model to explain bugs and justify fixes.

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

## Core primitives

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

## Core best practices

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

Default to Solid's control-flow primitives instead of ad-hoc `&&`, ternaries, and `.map(...)` inside JSX.

Use the control-flow rule pack below for the detailed bias:

- `Show` for primary conditional branches
- `For` for referentially keyed lists
- `Index` when position matters more than identity
- `Switch` and `Match` for multi-branch UI

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

For primitives and simple references, signals are often enough. For nested objects or arrays with fine-grained updates, prefer `createStore`.

Use the state-management rule pack below for the operational guidance:

- path syntax for targeted updates
- `produce` for complex mutable-style changes
- `reconcile` for incoming server or external data
- context when state is genuinely shared across a subtree

### Keep context values stable and intentional

If you provide context, prefer a stable object of capabilities or state rather than recreating provider values ad hoc. Context should clarify sharing boundaries, not replace ordinary local state.

### Use refs and effects only for real imperative work

If the job is focus management, measurement, browser APIs, or third-party integration, refs plus effects are appropriate. If the job is ordinary rendering, keep it declarative and use the refs-and-DOM rule pack below for specifics.

## Debugging checklist

When a Solid 1.x component is not updating correctly, check these first:

1. Was a signal or prop read outside a reactive scope?
2. Were props destructured in a way that broke the getter?
3. Was an accessor passed where a plain value should have been, or vice versa?
4. Is `createEffect` being used to mirror state that should be derived?
5. Should `splitProps` or `mergeProps` be used instead of destructuring?
6. Would `createStore` better fit the shape of the state?
7. Was `untrack` or `batch` used intentionally, or is it masking the real problem?

## Specialized rule packs

Use these rule packs when the task is more specialized than the general reactivity guidance above.

Treat the rule IDs as lookup labels inside this skill. They are there to make review checklists and task-focused loading more compact.

### Control flow rules

| Rule | Priority | Description |
| --- | --- | --- |
| 3-1 Use Show for Conditionals | HIGH | Use `<Show>` instead of ternary operators for primary conditional UI branches. |
| 3-2 Use For for Lists | HIGH | Use `<For>` for referentially-keyed list rendering. |
| 3-3 Use Index for Primitives | MEDIUM | Use `<Index>` when array position matters more than item identity. |
| 3-4 Use Switch/Match | MEDIUM | Use `<Switch>` and `<Match>` for multiple conditions instead of deeply nested ternaries. |
| 3-5 Provide Fallbacks | LOW | Prefer explicit `fallback` props for loading or empty states. |
| 3-6 Stable Component Mount | MEDIUM | Avoid rendering the same component in multiple `Show` or `Switch` branches; keep it in one stable position when possible. |

### State management rules

| Rule | Priority | Description |
| --- | --- | --- |
| 4-1 Signals vs Stores | HIGH | Use signals for primitives and simple references, stores for nested objects and arrays. |
| 4-2 Use Store Path Syntax | HIGH | Use store path syntax for granular, efficient updates. |
| 4-3 Use produce for Mutations | MEDIUM | Use `produce` for complex mutable-style store updates. |
| 4-4 Use reconcile for Server Data | MEDIUM | Use `reconcile` when integrating server or external data into an existing store shape. |
| 4-5 Use Context for Global State | MEDIUM | Use context for shared cross-component state or capabilities. |

### Refs and DOM rules

| Rule | Priority | Description |
| --- | --- | --- |
| 5-1 Use Refs Correctly | HIGH | Use callback refs for conditional elements and imperative handles that may appear later. |
| 5-2 Access DOM in onMount | HIGH | Access DOM elements in `onMount`, not during render. |
| 5-3 Cleanup with onCleanup | HIGH | Always clean up subscriptions, timers, listeners, and imperative integrations. |
| 5-4 Use Directives | MEDIUM | Use `use:` directives for reusable element behaviors in Solid 1.x. |
| 5-5 Avoid innerHTML | HIGH | Avoid `innerHTML` with unsanitized content; prefer JSX or `textContent`. |
| 5-6 Event Handler Patterns | MEDIUM | Use `on:` and `oncapture:` namespaces and array handler syntax correctly when the platform or library expects them. |
| 5-7 Web Component Controlled State | HIGH | Use `createEffect`, refs, and imperative calls to sync signals to web component APIs when declarative attributes are not enough. |

### Performance rules

| Rule | Priority | Description |
| --- | --- | --- |
| 6-1 Avoid Unnecessary Tracking | HIGH | Do not access signals outside reactive contexts unless the read is intentionally static. |
| 6-2 Use Lazy Components | MEDIUM | Use `lazy()` for code splitting when large components do not need to be in the initial bundle. |
| 6-3 Use Suspense | MEDIUM | Use `<Suspense>` for async loading boundaries instead of ad-hoc loading booleans everywhere. |
| 6-4 Optimize Store Access | LOW | Read only the store properties you actually need. |
| 6-5 Prefer classList | LOW | Use `classList` for conditional class toggling in Solid 1.x. |
| 6-6 Web Component CSS and Bundle Strategy | MEDIUM | Import custom elements individually when possible, and keep `::part()` overrides in a global stylesheet rather than CSS modules. |

### Accessibility rules

| Rule | Priority | Description |
| --- | --- | --- |
| 7-1 Use Semantic HTML | HIGH | Use the right semantic element before reaching for ARIA. |
| 7-2 Use ARIA Attributes | MEDIUM | Add appropriate ARIA attributes for custom controls and landmark exposure. |
| 7-3 Support Keyboard Navigation | MEDIUM | Make sure interactive elements are reachable and usable from the keyboard. |

### Testing rules

| Rule | Priority | Description |
| --- | --- | --- |
| 8-1 Configure Vitest for Solid | CRITICAL | Configure Vitest with the Solid plugin and Solid-specific resolve conditions. |
| 8-2 Wrap Render in Arrow Functions | CRITICAL | Always use `render(() => <Comp />)`, not `render(<Comp />)`. |
| 8-3 Test Primitives in a Root | HIGH | Wrap signal, memo, and effect tests in `createRoot` or use `renderHook`. |
| 8-4 Handle Async in Tests | HIGH | Use `findBy` queries and correct timer configuration for async behavior. |
| 8-5 Use Accessible Queries | MEDIUM | Prefer role and label queries over test IDs. |
| 8-6 Separate Logic from UI Tests | MEDIUM | Test reactive primitives independently from component rendering when possible. |
| 8-7 Browser Mode for Web Components and PWA APIs | HIGH | Use Vitest browser mode for custom elements, shadow DOM, and browser-native APIs. |
| 8-8 Testing Headless UI Libraries with Non-Standard ARIA | MEDIUM | Inspect the real tree and portal structure before choosing queries. |
| 8-9 Browser-Native API Test Isolation | HIGH | Clear IndexedDB and localStorage between tests, and close connections before `deleteDatabase`. |
| 8-10 Router Integration Testing | HIGH | Use `MemoryRouter` root setup to provide router context to layout and provider trees. |
| 8-11 TanStack Query Test Setup | HIGH | Create a fresh `QueryClient` per test with retries and caching disabled. |

## Task-focused rule selection

### Writing new components

Load these rules first when creating new Solid 1.x components:

- 1-1 Ensure signals are called as functions
- 2-1 Prevent reactivity breakage from destructured props or early reads
- 2-2 Handle default props correctly
- 2-3 Separate local and forwarded props
- 2-6 No early returns; prefer JSX control flow
- 2-9 Never call components as plain functions
- 3-1 Proper conditional rendering with `Show`
- 3-2 Efficient list rendering with `For`
- 5-3 Cleanup long-lived work with `onCleanup`

### Code review

Bias code review toward these priorities:

- CRITICAL: 1-1, 2-1, 2-6, 2-9
- HIGH: 1-2, 1-3, 1-7, 2-7, 5-2, 5-3, 5-5

### Performance optimization

Load these rules when optimizing performance:

- 1-2 Prevent unnecessary recomputation
- 1-6 Reduce update cycles
- 4-2 Prefer granular store updates
- 6-1 Prevent unwanted subscriptions
- 6-2 Use code splitting when it meaningfully shrinks the initial bundle
- 6-4 Read only the store properties that matter

### State management

Load these rules when the task is mostly about application state:

- 4-1 Choose the right primitive
- 4-2 Use efficient store updates
- 4-3 Use `produce` for complex mutations
- 4-4 Use `reconcile` for external data integration
- 4-5 Use context for cross-component state

### Accessibility audit

Load these rules when auditing accessibility:

- 7-1 Semantic structure
- 7-2 Screen reader support
- 7-3 Keyboard support

### Writing tests

Load these rules when writing or reviewing tests:

- 8-1 Correct Vitest configuration
- 8-2 Reactive render scope
- 8-3 Reactive ownership for primitives
- 8-4 Async queries and timers
- 8-5 Accessible query selection
- 8-6 Test architecture separation
- 8-7 Browser mode versus jsdom
- 8-8 Portals and non-standard ARIA structures
- 8-9 IndexedDB and localStorage cleanup patterns
- 8-10 MemoryRouter setup for integration tests
- 8-11 QueryClient configuration for tests

### Integrating web components and custom elements

Load these rules when the user is using Shoelace, FAST, Lion, Material Web Components, native `<dialog>`, the Popover API, or similar browser-native imperative APIs:

- 2-10 Declare custom element tags in the JSX namespace and type newer HTML attributes when TypeScript needs help
- 5-6 Use `on:` for custom element events and type `CustomEvent` payloads correctly
- 5-7 Sync Solid signals to imperative web component APIs with refs and effects
- 6-6 Prefer per-component imports and keep `::part()` overrides in global CSS

## Common mistakes to catch

| Mistake | Rule | Solution |
| --- | --- | --- |
| Forgetting `()` on signal access | 1-1 | Always call signals like `count()`. |
| Destructuring props | 2-1 | Access them via `props.name` or `splitProps`. |
| Using ternaries for primary conditionals | 3-1 | Prefer `<Show>`. |
| Using `.map()` for dynamic lists | 3-2 | Prefer `<For>`. |
| Deriving values in effects | 1-2 | Prefer `createMemo` or a derived function. |
| Setting signals in effects to mirror other state | 1-4 | Derive the value or trigger updates from the real source. |
| Accessing DOM during render | 5-2 | Use `onMount`. |
| Forgetting cleanup | 5-3 | Use `onCleanup`. |
| Early returns in components | 2-6 | Use `<Show>` or `<Switch>` in JSX. |
| Using `className` or `htmlFor` | 2-7 | Use `class` and `for`. |
| Using `style="color: red"` or camelCase inline styles | 2-8 | Use `style={{ color: "red" }}` and keep CSS property names platform-correct. |
| Using `innerHTML` with user data | 5-5 | Use JSX or sanitize with a library such as DOMPurify first. |
| Spreading a whole store when only one field is needed | 6-4 | Read specific properties. |
| String concatenation for conditional classes | 6-5 | Prefer `classList={{ active: isActive() }}`. |
| `render(<Comp />)` without an arrow | 8-2 | Use `render(() => <Comp />)`. |
| Effects or memos in tests without an owner | 8-3 | Wrap them in `createRoot` or use `renderHook`. |
| `getBy` for async content | 8-4 | Use `findBy`. |
| Calling `MyComp(props)` instead of `<MyComp />` | 2-9 | Use JSX or `createComponent()`. |
| Calling router hooks like `useMatch()` inside `createEffect` | 1-7 | Call hooks once at component init, not inside reactive computations. |
| Rendering the same component in multiple `Switch` branches | 3-6 | Keep it mounted in one stable position. |
| Custom elements not upgrading in tests | 8-7 | Use browser mode instead of jsdom. |
| IndexedDB state leaking between tests | 8-9 | Close the connection before `deleteDatabase` and reset state between tests. |
| Router primitives throwing missing-route errors | 8-10 | Provide router context with `MemoryRouter`. |
| Query retries masking failures in tests | 8-11 | Use a fresh QueryClient with retries disabled. |
| `waitFor(length === 0)` passing before data loads | 8-4 | Use a settled anchor like `findBy` before asserting absence. |
| `getByRole('form')` failing even though the form exists | 7-2 | Add `aria-label` or `aria-labelledby` so the form has an accessible name. |
| `<my-element onMyChange={...}>` missing custom events | 5-6 | Use `on:my-change`. |
| `::part()` rules inside CSS modules not applying | 6-6 | Move them to a global stylesheet. |
| Barrel imports for whole web component libraries | 6-6 | Import only the components you use. |
| `value={signal()}` on a custom element not syncing state | 5-7 | Listen for events and push values imperatively through the element ref. |
| `<div popover>` or `<button popoverTarget="x">` TypeScript error | 2-10 | Augment the relevant HTMLElement types in a `.d.ts` file for newer HTML attributes. |
| Object props on custom elements becoming `"[object Object]"` | 5-7 | Use `prop:myProp={value}` when the library expects a JS property. |
| Experimental CSS properties such as `anchor-name` causing a TypeScript error | 2-8 | Cast through `unknown` to `JSX.CSSProperties` instead of forcing `never`. |

## React comparison

When the user is thinking in React terms, keep these comparisons handy:

| React | Solid.js 1.x |
| --- | --- |
| Components re-render on state change | Components run once; signals update DOM directly |
| `useState()` returns `[value, setter]` | `createSignal()` returns `[getter, setter]` |
| `useMemo(fn, deps)` | `createMemo(fn)` with automatic tracking |
| `useEffect(fn, deps)` | `createEffect(fn)` with automatic tracking |
| Destructure props freely | Preserve prop getters; do not destructure by default |
| Early returns like `if (!x) return null` | Prefer `<Show>` or `<Switch>` in JSX |
| `{condition && <Comp />}` | `<Show when={condition}><Comp /></Show>` |
| `{items.map(item => ...)}` | `<For each={items}>{item => ...}</For>` |
| `className` | `class` |
| `htmlFor` | `for` |
| `style={{ fontSize: 14 }}` | `style={{ "font-size": "14px" }}` when needed for CSS-value correctness |
| React 19 custom element events can look like `onMyEvent` | Solid uses `on:my-event` for custom element events |

## Priority levels

- CRITICAL: fix immediately; likely runtime bugs, broken reactivity, or broken tests
- HIGH: address in code review; correctness or maintainability issues
- MEDIUM: apply when relevant; improves quality and ergonomics
- LOW: refactor-level improvements and secondary optimizations

## Tooling note

For automated linting alongside these rules, prefer `eslint-plugin-solid`. It catches many of the same issues around destructured props, early returns, React-style props, style prop format, and unsafe patterns such as `innerHTML` misuse.

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
