# Stores And Helpers

Use this reference when the task is about store updates, projections, helper renames, or plain-value extraction.

The source of truth for this file is the migration guide plus the Solid 2.0 stores RFC.

## Main changes

- store APIs are exported from `solid-js`
- draft-first store setters are the preferred update style
- function-form `createStore(fn, initial)` supports derived-store patterns
- `createProjection` replaces selector-like patterns with a more general projection model
- `snapshot(store)` replaces `unwrap(store)`
- `mergeProps` becomes `merge`
- `splitProps` becomes `omit`
- `storePath(...)` exists as a compatibility bridge, not the default fresh-code style

## Draft-first setters

Prefer:

```ts
setStore(s => {
  s.user.address.city = "Paris";
});
```

This is the center of the 2.x store API.

`storePath(...)` is still useful when migrating heavily path-based code:

```ts
setStore(storePath("user", "address", "city", "Paris"));
```

## Setter return values

Returning a value from a setter performs a top-level shallow replacement or diff.

```ts
setStore(s => {
  return { ...s, list: [] };
});
```

That is different from mutating the draft in place.

## `createProjection`

Use `createProjection` when the derived result is better modeled as a store-shaped projection than a plain accessor.

```ts
const selected = createProjection((draft) => {
  const id = selectedId();
  draft[id] = true;
  if (draft._prev != null) delete draft[draft._prev];
  draft._prev = id;
}, {});
```

It is the more explicit replacement for several old selector-like patterns.

## Helper renames

### `snapshot(store)`

Use `snapshot(store)` when you need a non-reactive plain value for serialization or interop.

### `merge`

`merge` replaces `mergeProps`.

Important semantic change: `undefined` is treated as a real override value.

```ts
const merged = merge({ a: 1, b: 2 }, { b: undefined });
```

### `omit`

`omit` replaces `splitProps` for excluding keys.

```ts
const rest = omit(props, "class", "style");
```

### `deep(store)`

Use `deep(store)` only when deep observation is truly intended. Do not treat it as a default access pattern.
