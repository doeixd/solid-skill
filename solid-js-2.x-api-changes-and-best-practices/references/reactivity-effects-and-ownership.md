# Reactivity, Effects, And Ownership

Use this reference when the task is about the Solid 2.0 execution model rather than just a rename.

The source of truth for this file is the Solid 2.0 migration guide plus the Reactivity and ownership RFCs.

## Core behavior to keep in mind

- writes are batched by default and become visible after the microtask flush
- `flush()` is the synchronous escape hatch when code truly needs a settled point now
- `createEffect` is split into a compute phase and an apply phase
- top-level reactive reads in component bodies warn in dev unless wrapped intentionally
- writes inside owned reactive scope warn in dev unless the signal is a narrow internal `pureWrite` case
- `onMount` is replaced by `onSettled`
- nested `createRoot(...)` calls are owned by the parent by default

## Batching and `flush()`

In Solid 2.0, setters queue work for the next microtask. Reads keep returning the last committed value until the queue flushes.

```ts
const [count, setCount] = createSignal(0);

setCount(1);
count(); // still 0

flush();
count(); // now 1
```

Use `flush()` sparingly. It is most appropriate in tests or imperative DOM workflows that need an immediately updated DOM or state snapshot.

## Split effects

The compute function tracks dependencies. The apply function performs side effects and may return cleanup.

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

Bias hard toward this model:

- compute reads reactive inputs
- apply touches the outside world
- cleanup belongs on the apply side
- if the job is derivation, prefer `createMemo`

## Top-level reactive reads

In component bodies, top-level reads warn in dev. This includes common patterns like destructuring reactive props or storing `props.title` in a local variable before JSX.

Prefer:

```tsx
function Title(props) {
  return <h1>{props.title}</h1>;
}
```

If non-tracking behavior is truly intended, make it explicit with `untrack`.

## Writes inside owned scope

Writing app state from memos, effects, or other owned reactive scopes is now a warning-shaped smell.

Prefer:

- deriving with `createMemo`
- writing in event handlers or actions
- using `pureWrite: true` only for narrow internal cases such as refs or internal bookkeeping

Do not treat `pureWrite: true` as a general warning silencer.

## `onSettled`

`onSettled` replaces `onMount` and can return cleanup.

```ts
onSettled(() => {
  measureLayout();
  const onResize = () => measureLayout();
  window.addEventListener("resize", onResize);
  return () => window.removeEventListener("resize", onResize);
});
```

Treat `onMount` migrations as semantic review, not a blind rename.

## Ownership and `createRoot`

In Solid 2.0, a root created inside an existing owner is owned by its parent and is disposed with it.

Detach only when global lifetime is actually intended:

```ts
const singleton = runWithOwner(null, () => {
  const [value, setValue] = createSignal(0);
  return { value, setValue };
});
```

That explicit detachment is preferable to accidentally creating unowned graphs.
