# App Architecture

Use this reference when the task is about structuring a feature or whole app.

## Core model

SolidStart works best when you separate concerns clearly:

- route components describe UI and call async accessors
- queries load server-backed data
- actions mutate server-backed state
- server utilities hold DB, auth, and secret-aware logic
- middleware handles lightweight request-wide concerns
- API routes exist only for endpoints outside the UI contract

This keeps route files thin and makes caching, redirects, and revalidation line up with the router.

## Recommended feature shape

For a typical data-backed route, prefer this shape:

1. Put reusable server logic in a local helper or server utility.
2. Wrap reads in a keyed `query`.
3. Start the read from `route.preload`.
4. Consume the read with `createAsync` in the route component.
5. Use an `action` for writes coming from that route.

This pattern gives you early fetch start, deduplication, router-aware invalidation, and simpler UI code.

## Route composition guidance

- Keep route files focused on the specific page or endpoint.
- Extract shared server logic when two routes need the same auth or database steps.
- Avoid pushing everything into middleware or giant root-level abstractions.
- Avoid hiding route behavior behind too many helper layers when the feature is local.

## Protected route pattern

For a protected route:

1. Use a server-side helper such as `getUser()`.
2. Call it inside the query or action that needs protection.
3. Redirect there if the user is not allowed.
4. If the redirect can happen during SSR data loading, use `createAsync(..., { deferStream: true })`.

Why: redirects cannot happen after streaming has already started.

## What to avoid

- Do not fetch critical server data directly in `onMount` for route pages.
- Do not build your own parallel cache and invalidation layer when queries already model it.
- Do not use middleware as the only auth gate.
- Do not create API routes for internal UI-only data flows unless there is a real external consumer.
