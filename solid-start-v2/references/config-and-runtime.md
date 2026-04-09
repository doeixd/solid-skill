# Config And Runtime

Use this reference for project setup, environment wiring, migration-sensitive details, and runtime constraints.

## v2 setup direction

SolidStart v2 moves toward Vite-native setup.

Important practical changes from the source material:

- use `vite.config.ts`
- use Vite commands such as `vite dev`, `vite build`, and `vite preview`
- add `@solidjs/start/env` to TS compiler types when needed
- replace older `vinxi/http` imports with `@solidjs/start/http` where the local version supports it

Treat these as defaults, but verify the exact installed package versions in the repo before editing config.

## Runtime mindset

Write server code against Web-standard primitives where possible:

- `Request`
- `Response`
- `Headers`
- `URL`

This keeps the app more portable across runtimes and avoids accidental lock-in to one environment.

## Serialization

SolidStart serializes server function arguments and return values between server and client.

High-value guidance:

- keep payloads plain and simple
- avoid deeply nested or exotic values when a simpler shape works
- remember that v2 defaults to `json` serialization for CSP compatibility
- `js` mode can be smaller or faster but requires `unsafe-eval`

If serialization gets awkward, simplify the return shape rather than fighting the serializer.

## Environment variables

Keep secrets server-only. Prefer the repo's established Vite environment pattern for compile-time values and private server configuration.

Do not leak private env vars into client code just because a helper is imported into a shared file.

## Compatibility note

Some docs and examples around Start v2 still reflect transition-era package boundaries. If local imports, config exports, or helper names differ, trust the installed project and adapt the pattern rather than copying old examples literally.
