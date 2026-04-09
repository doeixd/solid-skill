# API Routes And Responses

Use this reference when the task needs an HTTP endpoint beyond the normal UI data flow.

## When to use API routes

Use an API route when the endpoint must be consumed outside the app's route component flow, such as:

- webhooks
- OAuth callbacks
- public or partner REST endpoints
- GraphQL or tRPC endpoints
- file, image, PDF, or other non-HTML responses

If the logic mainly serves the app's own route UI, prefer queries and actions instead.

## Shape

API routes follow file-based routing, but instead of exporting a page component they export HTTP method handlers such as:

- `GET`
- `POST`
- `PATCH`
- `DELETE`

Handlers receive an API event with request data, route params, and an internal fetch helper.

## Response guidance

Return either:

- a JSON-like value when the framework can serialize it cleanly
- a `Response` when you need full control over status, headers, or body shape

Use a `Response` for explicit content types, downloads, or low-level HTTP behavior.

## Overlap with UI routes

API routes take priority over overlapping UI route paths. If a path must support both API-like and UI behavior, be deliberate about content negotiation and fallback behavior.

## Auth and validation

Treat API routes as hard server boundaries.

- validate input there or in a server helper they call
- perform authorization there or in the called helper
- return clear HTTP status codes for external clients

Do not assume browser-only behavior when writing an API route.
