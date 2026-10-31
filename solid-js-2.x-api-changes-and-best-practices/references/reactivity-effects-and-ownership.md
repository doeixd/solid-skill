# Reactivity, Effects, And Ownership

Use this reference when the task is about the Solid 2.0 execution model rather than just a rename.

The source of truth for this file is the Solid 2.0 migration guide plus the Reactivity and ownership RFCs.

## Core behavior to keep in mind

- writes are batched by default and become visible after the microtask flush
- `flush()` is the synchronous escape hatch when code truly needs a settled point now
- `createEffect` is split into a compute phase and an apply phase
- top-level reactive reads in component bodies warn in dev unless wrapped intentionally
- writes inside owned reactive scope throw in dev unless the signal opts in with `{ ownedWrite: true }` (formerly `pureWrite`)
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

Writing app state from memos, effect computes, or component bodies throws in dev (`REACTIVE_WRITE_IN_OWNED_SCOPE`). The guard checks the ambient owner, so wrapping the write in `untrack` does not help.

Prefer:

- deriving with `createMemo`
- writing in event handlers, `onSettled`, effect apply functions, or actions
- using `{ ownedWrite: true }` only for narrow internal cases such as refs or internal bookkeeping

Do not treat `ownedWrite: true` as a general silencer.

## Writes in one frame land together

Writes made in the same synchronous frame commit together. If one of them starts async work (a navigation, a refetch, a pending memo), the others are held with it. To land one write on its own, such as closing a menu before calling `props.onSelect`, wrap it in `onFlush(() => setOpen(false))`. Do not reach for `queueMicrotask` or `setTimeout`.

Because reads lag writes until the flush, two rapid events can each read the state from before the other. When a handler must build on its own previous write (two quick "add to cart" clicks), keep the intended value in a plain variable updated synchronously, and derive the request from that, not from a signal read.

## Seeding signals from reactive values

A signal's initial value from another accessor is a top-level read. Make it explicit:

```ts
const [debounced, setDebounced] = createSignal(untrack(label));
```

A writable derived signal receives its previous value, which suits state that follows an input but can also be set by hand:

```ts
const [layout, setLayout] = createSignal<"Landing" | "Search">(prev =>
  query() !== "" ? "Search" : (prev ?? "Landing"),
);
```

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
