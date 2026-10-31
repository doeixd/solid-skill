# Progressive Enhancement: Working Without JavaScript

Use this reference when an app should keep working with JavaScript disabled, blocked, or not yet loaded, or when the user asks for a "zero-JS" or no-JS fallback.

Verified on `solid-js@2.0.0-rc.20`, `@solidjs/router@2.0.0-next.43`, and `@solidjs/vite-plugin` start mode, in an SSR search app whose search, filters, paging, and cart all work with JavaScript off (marked **In practice**).

## What is built in, and what is not

Built in:

- **Server rendering.** With `ssr: true`, every route arrives as complete HTML.
- **Plain links.** Router 2 has no link component; every `<a href>` navigates without JavaScript.
- **Forms bound to server functions.** A `formAction` bound to a server function, or a form whose `action` is a server function's URL, does a real POST without JavaScript. The server-function runtime runs the function and redirects back, passing the result through a one-shot flash cookie.
- **Server components** (experimental, `serverFunctions: { components: true }`): parts of a page that ship no JavaScript. Measure first: enabling them changes the transport for every server-function call.

Not built in: a switch that makes a whole app work without scripts. Each interactive control needs an HTML equivalent, and the server must render everything a visitor needs without waiting on the client.

## Rules

### 1. Put everything visible in the HTML, not in streamed content

`renderToStream` sends a `<Loading>` fallback first; the resolved content arrives later inside an inert `<template>`, and an inline `$df(...)` script swaps it in. With JavaScript off, the fallback stays. Anything a no-JS visitor should see must be resolved before the shell flushes:

- fetch it in the router's app-level `preload` (a loader) and read it synchronously, or
- hold the stream on it with `createMemo(fn, { deferStream: true })`.

**In practice:** the loader fetched the Top Results plus exactly the price ranges the list's server window would show, computed from the same filtered and sorted listing the component renders. Every rendered row was priced in the HTML, on every page and sort order. For the default first page this was the same fetch as before, so JavaScript users paid nothing.

### 2. Render the fallback controls on both sides; show them with `@media (scripting: none)`

Hydration needs the server and client markup to match, so do not render no-JS controls only on the server (`isServer && ...`) or only inside `<noscript>`. Render them always, hide them by default, and show them with CSS when scripts are off:

```css
.nojs-only { display: none; }

@media (scripting: none) {
  .nojs-only { display: flex; }
  .filters > :not(.nojs-filters) { display: none; } /* hide the scripted controls */
}
```

`display: none` also removes them from the accessibility tree when JavaScript runs.

### 3. Filters, sorting, and search as GET forms

If the server already reads state from search params (`?q=acme&sort=price`), a plain `<form method="get" action="/">` with matching `name`s works with no new server code. Carry the current query in a hidden input, use `<select>` with `selected` on options and `checked` on checkboxes for the current values, and keep it small: a text input beats a `<select>` with hundreds of options in every page's HTML.

```tsx
<form class="nojs-only nojs-filters" method="get" action="/">
  <input type="hidden" name="q" value={search.query()} />
  <label>Extension <input name="tld" value={filters.tld === "All" ? "" : filters.tld} /></label>
  <label><input type="checkbox" name="available" value="1" checked={filters.availability === "Available"} /> Available only</label>
  <select name="sort">
    <For each={sortOrders}>{order => <option value={sortSlug(order)} selected={filters.sort === order}>{labels[order]}</option>}</For>
  </select>
  <button type="submit">Apply</button>
</form>
```

Empty fields submit as empty params (`tld=&sort=`); the server should treat them as defaults.

### 4. Virtualized and infinite lists need paging links

A virtual or infinite list renders only a window on the server, and scrolling never adds rows without JavaScript. Add Previous and Next links with an offset param (`?from=33`), make the server window start at that offset, and preload that window's data (rule 1). Under `scripting: none`, switch the list from absolute positioning to normal flow so the window appears at the top of the page:

```css
@media (scripting: none) {
  .results-viewport { display: grid; height: auto; grid-template-columns: repeat(var(--cols), minmax(0, 1fr)); }
  .results-viewport .virtual-cell { position: static; transform: none; width: auto; }
}
```

The client must read the same offset when it hydrates, so the window matches; its first scroll measurement then takes over.

### 5. Mutations as forms posting to a server function, enhanced in the page

Wrap each mutation control in a form whose `action` is the server function's URL with its arguments bound, and handle the submit in the page when JavaScript runs:

```tsx
import { type ServerFunction, serverFunctionActionUrl } from "@solidjs/web/server-functions";
import { changeCart } from "../server/cart";

// "use server" functions are compiled into server references, but their
// declared type stays the plain async function.
const changeCartRef = changeCart as unknown as ServerFunction;

const submitWith = (onSubmit: () => void) => (form: HTMLFormElement) =>
  form.addEventListener("submit", event => {
    event.preventDefault();
    onSubmit();
  });

<form
  method="post"
  action={serverFunctionActionUrl(changeCartRef, cart.has(name) ? "remove" : "add", name)}
  ref={submitWith(() => cart.toggle(domain))}
>
  <button type="submit" aria-label={cart.has(name) ? `Remove ${name}` : `Add ${name}`}>…</button>
</form>
```

```ts
// src/server/cart.ts
export async function changeCart(op: "add" | "remove", name: string) {
  "use server";
  if ((op !== "add" && op !== "remove") || typeof name !== "string") throw new RangeError("Invalid change");
  const next = applyChange(readCartCookie(), op, name);
  writeCartCookie(next);
  return respond(next, { revalidate: "" }); // changes no query
}
```

- **Without JavaScript**, the browser posts the form; the runtime calls `changeCart(op, name)` and redirects back, and the next render reads the updated cookie.
- **With JavaScript**, the listener runs the optimistic `action` instead. Attach it on the form element (a `ref` factory), not as a delegated `onSubmit`: the router listens for submits on `document`, skips events that are already default-prevented, and otherwise handles server-function forms itself (loading its form-action code and treating the post as a mutation). An element listener always runs first.
- Bind state into the URL (`add` versus `remove`) so the post does what the button shows, and validate every argument: anything can post to the endpoint.
- Use `formAction` from the router instead when you want its submission tracking, `aria-busy`, and `useSubmissions`, and accept loading its action code.

**In practice:** this added about 1 KB of JavaScript, and the router's form-action chunk never loaded with JavaScript on.

### 6. State that only scripts can show

Some UI has no sensible HTML form (a cart drawer opened by a button, a modal editor). Either give it a route of its own (`/cart`) or say plainly what needs JavaScript. Make sure the essentials are visible regardless: a count in the header, an "in cart" state on each row.

## Testing

Run a Playwright spec with JavaScript disabled, and keep it in the suite so it does not regress:

```ts
import { expect, test } from "@playwright/test";

test.use({ javaScriptEnabled: false });

test("search, paging, and the cart work without JavaScript", async ({ page }) => {
  await page.goto("/?q=demo");
  // every rendered row is priced, nothing waits on streamed content
  expect(await page.locator(".row .price").count()).toBe(await page.locator(".row").count());
  await page.getByRole("navigation", { name: "Result pages" }).getByRole("link", { name: "Next" }).click();
  await expect(page).toHaveURL(/from=/);
  await page.getByRole("button", { name: "Add demo.com to cart" }).click();
  await expect(page.locator("#cart-button")).toHaveAttribute("aria-label", "Cart, 1 domains");
});
```

Also check the JavaScript path did not change: the mutation goes through the optimistic action (no form POST in the network log), no new chunks load on submit, the scripted controls are visible and the no-JS ones hidden, and there are no hydration warnings.

## Measuring the cost

Compare initial-load JavaScript and HTML against the previous build at the same compression level: build the previous commit in a separate `git worktree`, then compress both with the same settings. A CDN's on-the-fly compression varies between builds, so its as-served numbers can move more than the code did. Report the as-served number where you publish sizes, with the build-level difference beside it.

If the worktree shares `node_modules` through a junction or symlink, delete the link itself before removing the worktree; some tools follow links when deleting.
