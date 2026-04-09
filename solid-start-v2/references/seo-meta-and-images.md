# SEO, Meta, And Images

Use this reference for route metadata, social sharing tags, and image guidance.

## Metadata baseline

SolidStart does not bundle metadata helpers by default. Use `@solidjs/meta` for page title and meta tags.

Useful primitives:

- `Title`
- `Meta`

## Route-level guidance

For user-facing routes:

- set a meaningful title
- add a description when the page is indexable or shareable
- add OG tags when the route is likely to be shared

Put site-wide defaults in the root document and override or extend them in route components where needed.

## Dynamic metadata

When the page title or description depends on route data, derive metadata from the same route data source the page uses. Keep metadata consistent with the rendered content.

## Image guidance

Image optimization is moving into `@solidjs/image` rather than the core Start package.

If the task involves production image handling, consider:

- local and remote image support
- lazy loading
- aspect ratio preservation
- build-time optimization

Confirm the installed image package and build setup in the repo before assuming the image component is available.

## Keep it practical

Do not add SEO tags mechanically to every route. Prioritize user-facing pages where search or sharing value exists.
