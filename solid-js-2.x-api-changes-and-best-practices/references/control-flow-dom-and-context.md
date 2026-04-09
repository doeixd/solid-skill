# Control Flow, DOM, And Context

Use this reference when the task is about list rendering, function-child accessors, DOM semantics, directive migration, or context providers.

The source of truth for this file is the migration guide plus the control-flow and DOM RFCs.

## Control flow

### `For`

`Index` is gone. The replacement is `For` with `keyed={false}`.

Important behavior change: `For` children receive accessors for both item and index.

```tsx
<For each={items()} keyed={false}>
  {(item, i) => <Row item={item()} index={i()} />}
</For>
```

### Function children often receive accessors

This also shows up in `Show`, `Match`, and related APIs.

Prefer:

```tsx
<Show when={user()}>{u => <Profile user={u()} />}</Show>
```

Avoid doing reactive reads at the top of those callback bodies.

### `Repeat`

Use `Repeat` for count or range rendering when the task is about repeated indexes rather than list diffing.

## DOM behavior

### Attributes over properties

Solid 2.0 moves closer to platform HTML semantics.

- built-in attributes are generally lowercase
- boolean attributes are presence or absence
- `attr:` and `bool:` namespaces are removed
- `oncapture:` is removed

When a platform API truly wants the string `"true"`, pass a string.

## `class`

`classList` is removed. Use `class` with string, object, or array forms.

```tsx
<div class={["card", { active: isActive(), disabled: isDisabled() }]} />
```

## Directives via `ref`

`use:` directives are removed. Use `ref` callbacks or directive factories.

```tsx
<input ref={autofocus} />
<button ref={tooltip({ content: "Save" })} />
<button ref={[autofocus, tooltip({ content: "Save" })]} />
```

Recommended pattern:

- setup phase creates subscriptions or primitives
- apply phase receives the element and performs DOM writes

## Context provider ergonomics

The context object itself is the provider component.

```tsx
const ThemeContext = createContext("light");

<ThemeContext value="dark">
  <Page />
</ThemeContext>
```

Prefer this over carrying forward `Context.Provider`.
