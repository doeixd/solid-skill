# Data And Mutations

Use this reference for route data, forms, revalidation, and write flows.

## Reads: prefer `query`

Use `query(async (...args) => { ... }, key)` for server-backed reads.

Why:

- request deduplication
- router-aware caching
- invalidation support
- a stable pattern shared across routes

When the read depends on secrets, auth, sessions, or direct database access, put `"use server"` inside the query body.

## Preload: start early, do not over-await

`route.preload` is for starting work as early as possible, not for fully resolving it.

Preferred pattern:

```ts
export const route = {
  preload: ({ params }) => getPost(params.id),
};

const getPost = query(async (id: string) => {
  "use server";
  return db.post.findUnique({ where: { id } });
}, "post");

export default function Page(props) {
  const post = createAsync(() => getPost(props.params.id));
}
```

Why: navigation starts the work early, and the component consumes the result through the router's async flow.

## Writes: prefer `action`

Use `action(async (...args) => { ... }, key)` for mutations.

Use actions when:

- handling a form submission
- creating, updating, or deleting server data
- redirecting after a successful write
- depending on router revalidation behavior

If the mutation changes server state and then navigates, redirect from the action so the next route's queries are revalidated naturally.

## Native forms first

Prefer:

```tsx
<form action={savePost} method="post">
```

over custom client fetch handlers when the interaction is a normal submission flow.

Why:

- less client code
- built-in router integration
- simpler pending and error handling

## Validation pattern

Validate in the action.

- Parse `FormData` using native APIs.
- Return a structured error object when you want the form to stay on the page.
- Redirect when the happy path should land on another route.

Avoid client-only validation as the source of truth.

## Pending and optimistic UI

Use `useSubmission(action)` when the route should react to a submission.

Common uses:

- disable the submit button while pending
- show server validation errors
- show optimistic content using `submission.input`

Use `useSubmissions` only when the UI truly needs to reason about multiple concurrent submissions.

## Programmatic mutations

Use `useAction(action)` when the write should be triggered imperatively instead of by a form.

Do not default to this for normal forms. Use it when the UX is genuinely non-form-driven.

## Good defaults

- read with queries
- write with actions
- preload with queries
- consume with `createAsync`
- redirect after successful writes when navigation is expected
- keep mutation logic server-side when it touches app state or secrets
