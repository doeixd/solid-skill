# v2 Changes And Ecosystem

Use this reference when the task touches SolidStart v2 migration-sensitive behavior, Nitro 3 runtime expectations, the standalone image package, or authoring packages that extend the Start ecosystem.

## SolidStart v2 in practice

The important framing is that Start v2 is a large internal architecture change with relatively small app-level surface changes for many projects.

High-value practical changes:

- Vite-native config and commands are the default direction.
- `app.config.ts`-era patterns are being replaced by `vite.config.ts`-based setup.
- older `vinxi/http` imports are being replaced by `@solidjs/start/http` where the installed version supports it.
- middleware signatures and surrounding server APIs may look more H3- and Web-standard-oriented than older examples.
- component code, server functions, and actions are intentionally closer to v1 than the internals are.

When helping with existing apps, prefer small migration edits over broad rewrites.

## Nitro 3 context

SolidStart v2 is converging around Nitro 3 rather than older Nitro 2-era compatibility layers.

What matters for implementation:

- expect a stronger bias toward Web-standard server APIs like `Request`, `Response`, `Headers`, and `URL`
- prefer portable server code over Node-only assumptions when there is no concrete need for Node-specific APIs
- expect runtime portability to matter for deployment targets and future ecosystem packages
- treat older deep Nitro or Vinxi-specific examples carefully and verify current local exports before copying them

Do not include Nitro marketing or release-history details in normal outputs. Use the runtime shift only to inform implementation choices.

## `@solidjs/image`

Image optimization is being split out of the core Start package into the standalone `@solidjs/image` package.

What the AI should assume:

- image support may be present as a separate package, not as a core Start feature
- image optimization may rely on a Vite plugin and build-time processing
- local and remote image flows, lazy loading, aspect-ratio preservation, and optimized asset caching are part of the intended image story
- `sharp` may be involved in the build pipeline, so installation and deployment constraints can matter

When asked to add high-quality image handling, first confirm whether `@solidjs/image` is installed and how the local repo wires it into Vite.

## Plugin authorship

Start v2 is moving toward a more plugin-oriented ecosystem. When the user is creating reusable Start add-ons or app-level extensions, treat that as package and runtime design work, not just route code.

Good plugin authorship guidance:

- keep the package narrowly scoped to one capability
- separate framework-facing integration from the app's business logic
- prefer Web-standard and Vite-native hooks where possible
- document required config, env vars, and runtime assumptions clearly
- avoid coupling the plugin to one app's route tree or internal folder layout unless that is explicitly the product
- keep server-only behavior explicit so consumers do not accidentally import it into client code

If the user is authoring a reusable image, auth, or middleware extension, check the local build and package boundaries before writing code. Start ecosystem packages are still close enough to transition-era changes that copied examples may be stale.

## When to read this reference

Read this file when the task is about:

- upgrading or stabilizing a Start v2 app
- understanding current runtime direction
- adding image optimization support
- creating a reusable SolidStart package or plugin
- deciding whether older Vinxi- or Nitro-2-style examples still apply
