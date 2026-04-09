# Auth, Middleware, And Sessions

Use this reference for request-scoped data, login state, protected routes, and cookie-backed sessions.

## Authorization rule

Do authorization checks as close to the data access path as possible.

Good places:

- server queries
- actions
- API routes
- server utilities used by those paths

Do not rely on middleware alone for authorization. Middleware does not run on every client-side navigation path the way people often assume.

## Recommended auth shape

1. Implement a server-only helper such as `getUser()` or `requireUser()`.
2. Read session or auth state there.
3. Call that helper inside protected queries and actions.
4. Redirect or throw an auth error there.

This keeps the security boundary easy to audit.

## Middleware's real job

Use middleware for lightweight cross-cutting concerns:

- request logging
- redirects based on URL shape
- request or response headers
- feature flags
- locale selection
- attaching request-scoped data to `event.locals`

Keep middleware fast. Avoid heavy computation or direct database-heavy workflows there.

## `event.locals`

`event.locals` is request-scoped storage.

Use it for values that are helpful across multiple server-side layers during one request, such as:

- current user summary
- trace IDs
- request timing
- feature flags

Read it from server-only code with `getRequestEvent()` when needed.

## Sessions

Cookie-backed sessions are the usual baseline.

Guidelines:

- use HTTP-only cookies for auth-related state
- keep the session secret in private env vars
- use a strong secret
- keep session reads and writes in server-only code

The source material includes Vinxi-era helpers such as `useSession`, `getSession`, `updateSession`, and `clearSession`. Treat those as patterns, but confirm the exact import path and helper API in the local repo before writing code.

## Protected route streaming caveat

If a protected route may redirect during SSR data loading, use:

```ts
const data = createAsync(() => getPrivateData(), { deferStream: true });
```

Why: server-side redirects cannot occur after streaming has already started.

## What to avoid

- auth checks only in middleware
- secrets in client code
- direct cookie parsing in many different places when one helper can centralize it
- optimistic assumptions that session helper imports are identical across app versions
