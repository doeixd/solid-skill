# Release Checklist

Use this reference when you want a final pass on implementation quality before considering SolidStart work done.

## Architecture

- route reads use `query` when router caching and invalidation matter
- writes use `action` when they mutate server state or back forms
- API routes exist only where an external HTTP boundary is actually needed
- middleware is lightweight and not the only auth layer

## Server boundary

- DB, session, auth, and secret-aware code stays behind `"use server"`
- helpers shared with client code do not accidentally pull in private env vars or server-only modules
- request-scoped values live in `event.locals` only when they are truly request-scoped

## SSR and navigation

- preload starts work early instead of over-awaiting
- redirects happen before SSR streaming commits when protection is required
- `deferStream: true` is used where protected SSR route data may redirect

## UX

- forms use actions unless the interaction is genuinely non-form-driven
- pending and validation states are exposed through `useSubmission` when appropriate
- user-facing pages have sensible title and metadata

## Runtime and portability

- responses and payloads are simple enough to serialize cleanly
- Web-standard server APIs are preferred over unnecessary runtime-specific ones
- image handling checks local support for `@solidjs/image` before assuming it exists

## Migration sensitivity

- local installed package names and exports were checked before applying older docs
- Vinxi-era examples were adapted rather than copied literally
- Nitro 3 direction informed implementation choices without dragging in historical noise
