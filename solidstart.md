https://github.com/solidjs/solid-start/discussions/2119

Hey, folks.

Long time no talk, our last announcement was done when beginning the DeVinxi work. We have successfully completed that and I'm happy to let you know that we have SolidStart v2 being used in production already in real-world apps, like OpenCode Console

As per the last announcement, our plan was to maintain Start and Solid in a version lockstep, it seemed possible with the works in Nitro v3, Solid v2, and Start v2 converging into similar release dates. We are now deciding against this plan in order to provide a better Developer Experience and migration plan for existing Start v1 and Solid v1 apps.

For us to reach ideal support in Solid v2 we would also need breaking changes in Router and Meta packages. However since the team has managed to keep the DeVinxi effort with minimal breaking changes, we believe it's worth the effort to create a stable release in the new architecture in order to lay down an incremental migration step towards the future of Solid.

Migrating from Start v1 to v2 requires small changes to vite.config.ts, tsconfig.json, and optionally to your createMiddleware signature. Although the internals are much different and improved, there is no change impacting your Server Functions, Actions, or component code.

I recommend you to give it a try, follow migration docs

Updated Timeline
roadmap-announcement
Remaining blockers in SolidStart alpha
new directives plugin
built-in server-only environment variables support
At this point, we don't anticipate any breaking changes to user-facing APIs already. But we need to land the above changes before moving to Beta.

Beta
Once those pull-requests are merged, we will release Start 2.0.0-beta.0.
Collect feedback, security audits, and address bugs. And consolidate our recommended server runtime from Nitro v2 to Nitro v3.

We also plan to deprecate the @solidjs/vite-plugin-nitro-2 in favor of Nitro 3 Beta.

As said above, we don't plan any breaking changes here. Unless something critical surface from audits or bugfixes, APIs are stable.

Release Candidate
By the time we're confident enough that the APIs are stable, we will ship 2.0.0-rc.0 , effectively deprecating Start v1, the create-solid CLI will stop offering Start v1 as an option and we will update the official docs with deprecation warnings on v1-only APIs.

Solid v2
As the ecosystem adapts to the new APIs and updates land in Solid-Router and Solid-Meta, we will start working on support for Solid v2.

Though likely, it's not certain if supporting Solid 2 means v3 for Start. Start v2 will not drop support on Solid v1.

If there's a chance to support an incremental migration from Solid v1 to v2 within Start v2, we will try it. But at this point we're not ready to make any promises.

We are also working on developing a plugin-based ecosystem for your apps.
The first package is the long awaited <Image> component with built-in image optimization.

As always, we're open and eager for feedback.
Follow the migration plan to upgrade an existing app.
For new ones, you can use the CLI

pnpm create solid --start --v2
If you've tried Start v2 alpha, then we'd love to hear your feedback!
Join us on Solid discord

Cheers from the Start team 👋

Replies:1 comment

agoldstein03
2 days ago
Now that #2070 merged (though I'm guessing #2127 may also be a blocker), and #2110 merged, anything else left blocking beta?

0 replies





https://nitro.build/blog/v3-beta

Nitro v3 Beta is here!
Nitro v3 is now available as a public beta — a ground-up evolution of the server framework, built around web standards, Rolldown, Vite v8, and the same deploy-anywhere promise.
Pooya Parsa

A Brief History
Nitro started as the server engine for Nuxt 3, designed to solve a specific problem: deployment-agnostic servers. Over time, Nitro grew beyond Nuxt. It became the foundation for many meta-frameworks and a toolkit for building standalone servers.

With Nitro v3, we took the opportunity to rethink the fundamentals. leaner APIs, Web standards, first-class Rolldown and Vite v8 integration, and a better experience for both humans and agents (more on that later!)

Since we quietly announced v3 alpha.0 (11 Oct 2025) at the first Vite Conf, Nitro v3 has been adopted by many users (~280k weekly downloads!) and refined through amazing contributions and feedback. including Tanstack Start, Vercel Workflows, and production apps like T3Chat.

A huge thanks to the VoidZero (Vite and Rolldown), Nuxt (v5 is coming!) and TanStack Start teams and every contributor who helped bring Nitro v3 to this milestone. ❤️

Why Build Servers?
We don't ship raw source files to the browser. We use build tools because they solve real problems: HMR for instant feedback, code splitting to load only what a route needs, tree shaking to eliminate dead code, and minification for smaller payloads. Tools like Webpack and then Vite transformed frontend development from painful to productive.

But frontend apps don't exist in isolation, they need APIs, databases, authentication, real-time data. They need a server.

With the rise of serverless and edge computing, the server side now faces the same constraints the frontend solved years ago. Cold starts mean every millisecond of startup matters. Memory limits are strict — bloated dependencies can push you over. Bundle size directly impacts deploy speed and boot time. And your code needs to run everywhere: Node.js, Deno, Bun, Cloudflare Workers, Vercel, etc. Yet most server frameworks still ship unoptimized, unbundled code, assuming a long-running process where none of this matters.

Nitro brings the build-tool philosophy to the backend. The same great DX you expect from frontend tooling: HMR for fast iteration and optimized builds powered by Rolldown with tree-shaken production output that performs as close to bare-metal as possible. One codebase, any runtime, any platform.

⚡ First-Class Vite Integration
Nitro now has a native Vite plugin to build full stack apps.

vite.config.ts

import { defineConfig } from "vite";
import { nitro } from "nitro/vite";

export default defineConfig({
  plugins: [nitro()],
});
Adding nitro() to your Vite apps gives you:

API routes via filesystem routing
Server-side rendering integrated with your frontend build
A production server — a single vite build produces an optimized .output/ folder with both frontend and backend, ready to deploy
This means you can add a full backend to any Vite project — See examples with React, Vue and Solid.js.

🚀 Performance by Default, Zero Bloat
Nitro compiles your routes at build time. There is no runtime router — each route loads on demand. Only the code needed to handle a specific request is loaded and executed.

Minimal server bundle built with the standard preset is less than 10kB, can be served with srvx at close to native speeds, and includes all the good features from H3.

We have also significantly reduced the number of dependencies, down to less than 20 from 321 dependencies.

🖌️ New Identity: nitro
Nitro v3 ships under a new NPM package: nitro, replacing the legacy nitropack.

All imports now use clean nitro/* subpaths:


import { defineNitroConfig } from "nitro/config";
import { defineHandler } from "nitro";
import { useStorage } from "nitro/storage";
import { useDatabase } from "nitro/database";
No more deep nitropack/runtime/* paths, plus, you can import nitro subpaths outside of builder useful for unit testing.

🔧 Bring Your Own Framework
Nitro v3 is not opinionated about your HTTP layer. You can use the built-in filesystem routing, or take full control with a server.ts entry file and bring any framework you prefer:

server.ts

import { Hono } from "hono";

const app = new Hono();
app.get("/", (c) => c.text("Hello from Hono!"));

export default app;
🌐 H3 (v2) with Web Standards
Nitro v3 upgrades to H3 v2, which has been fully rewritten around web standard primitives — Request, Response, Headers, and URL.

The result is cleaner, more portable server code:

routes/hello.ts

import { defineHandler } from "nitro";

export default defineHandler((event) => {
  const ua = event.req.headers.get("user-agent");
  return { message: "Hello Nitro v3!", ua };
});
Reading request bodies uses native APIs:

routes/submit.ts

import { defineHandler } from "nitro";

export default defineHandler(async (event) => {
  const body = await event.req.json();
  return { received: body };
});
No wrappers, no abstractions for things the platform already provides. If you know the Web API, you know H3 v2.

Elysia, h3, Hono — anything that speaks web standards works with Nitro.

🗄️ Built-in Primitives
Nitro ships with powerful but small and fully opt-in agnostic server primitives that work across every runtime.

When not used, nothing extra will be added to the server bundle. You can still use native platform primitives alongside Nitro's built-in ones. We are also bringing first class emulation for platform-specific primitives for dev See env-runner and nitrojs/nitro#4088 for more details.
Storage
A runtime-agnostic key-value layer with 20+ drivers — FS, Redis, S3, Cloudflare KV, Vercel Blob and more. Attach drivers to namespaces and swap them without changing your application code.


import { useStorage } from "nitro/storage";

const storage = useStorage();
await storage.setItem("user:1", { name: "Nitro" });
 Read more in Docs > Storage.
Caching
Cache server routes and functions, backed by the storage layer. Supports stale-while-revalidate, TTL, and custom cache keys out of the box.


import { defineCachedHandler } from "nitro/cache";

export default defineCachedHandler((event) => {
  return "I am cached for an hour";
}, { maxAge: 60 * 60 });
 Read more in Docs > Cache.
Database
A built-in SQL database that defaults to SQLite for development and can connect to Postgres, MySQL, and more using the same API.


import { useDatabase } from "nitro/database";

const db = useDatabase();
const users = await db.sql`SELECT * FROM users`;
 Read more in Docs > Database.
🌍 Deploy Anywhere
Build your server into an optimized .output/ folder compatible with:

Runtimes: Node.js, Bun, Deno
Platforms: Cloudflare Workers, Netlify, Vercel, AWS Lambda, Azure, Firebase, Deno Deploy, and more
No configuration needed — Nitro auto-detects your deployment target. Take advantage of platform features like ISR, SWR, and edge rendering without changing a single line of code.

🎨 Server-Side Rendering
Render HTML with your favorite templating engine, or use component libraries like React, Vue, or Svelte directly on the server. Go full universal rendering with client-side hydration.

Nitro provides the foundation and a progressive approach — start with API routes, add rendering when you need it, and scale to full SSR at your own pace.

 Read more in Docs > Renderer.
🟢 Nuxt v5
Nitro v3 will power the next major version of Nuxt.

Nuxt v5 will ship with Nitro v3 and H3 v2 at its core, bringing web-standard request handling, Rolldown-powered builds, and the Vite Environment API to the Nuxt ecosystem.

If you're a Nuxt user, you can already start preparing by familiarizing yourself with Nitro v3's new APIs, which will carry directly into Nuxt 5, and you can follow progress on adopting Nitro v3 in Nuxt

🏁 Getting Started
Create a New Project

npm

yarn

pnpm

bun

deno

npx create-nitro-app
See the quick start guide for a full step-by-step walkthrough.

🔄 Migrating from v2
Nitro v3 introduces intentional breaking changes to set a cleaner foundation. Here are the key ones:

nitropack → nitro (package rename)
nitropack/runtime/* → nitro/* (clean subpath imports)
eventHandler → defineHandler (H3 v2)
createError → HTTPError (H3 v2)
Web standard event.req headers and body APIs
Node.js minimum version: 20
Preset renames and consolidation (e.g., cloudflare → cloudflare_module)
For a complete list, see the migration guide.

Thank you to everyone who has contributed to Nitro over the years. We can't wait to see what you build with the new Nitro! ❤️

GitHub — Issues and discussions
Discord — Chat with the community
Edit this page



https://github.com/solidjs/solid-start/pull/2111

This PR introduces @solidjs/image, a new standalone package for image optimization in SolidStart applications. The image component is extracted from the core @solidjs/start package to allow for better modularity and independent versioning.

Changes
features moved to a standalone package @solidjs/image

Vite plugin for build-time image optimization using sharp
<Image> component with support for:
Local and remote images
Lazy loading with fallback
Aspect ratio preservation
CSS-based styling
Transformers for processing images at build time
Hash-based caching for optimized assets
sharp (^0.34.5) is a dependency of this package
@solidjs/start changes in relation to feat-image branch

Removed image re-export from the main package (no longer bundles image functionality)
Test & Fixture Apps
Added apps/fixtures/image - demo application
Added image tests in apps/tests for local and remote scenarios

Migrating from v1
This is a migration guide of how to upgrade your v1 SolidStart app to our new v2 version.

Please note that some third-party packages may not be compatible with v2 yet.

Migration steps
Update dependencies
npm
pnpm
yarn
bun
deno
npm i @solidjs/start@2.0.0-alpha.2 @solidjs/vite-plugin-nitro-2 vite@7
npm
pnpm
yarn
bun
deno
npm remove vinxi
Configuration files
Remove app.config.ts
Create vite.config.ts
import { solidStart } from "@solidjs/start/config";
import { defineConfig } from "vite";
import { nitroV2Plugin } from "@solidjs/vite-plugin-nitro-2";

export default defineConfig(() => {
  return {
    plugins: [
      solidStart({
        middleware: "./src/middleware/index.ts",
      }),
      nitroV2Plugin(),
    ],
  };
});
Compile-time environment variables are now handled by Vite's environment API.

// ...
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), "");

  return {
    // ...
    environments: {
      ssr: {
        define: {
          "process.env.DATABASE_URL": JSON.stringify(env.DATABASE_URL),
        },
      },
    },
  };
});
Update the build/dev commands to use native Vite instead of vinxi.

"scripts": {
  "dev": "vite dev",
  "build": "vite build",
  "start": "vite preview"
}
Environment types
Only the types entry is new in v2. Everything else can remain unchanged.

"compilerOptions": {
  "types": ["@solidjs/start/env"]
}
Server runtime helpers
Replace all imports from vinxi/http with @solidjs/start/http
Optional: update the middleware syntax to the newer H3 syntax
Previous
← Getting started
Next
Routing →

SolidStart
Don’t await in preload functions (loaders)
Something different about SolidStart than other frameworks is the concept of preloading instead of loaders.

A preload function is not meant to resolve data; it’s meant to start the work as early as possible.

SolidStart encourages not awaiting in preload and letting the component handle suspending and resolving the promise for you.

For example:

export const route = {
  preload: () => getPosts(),
} satisfies RouteDefinition;

export default function Page() {
  const posts = createAsync(() => getPosts());
}
Here we are just warming up the getPosts() cache. As soon as navigation starts, getPosts() begins executing. By the time the component renders, the promise may already be resolved or at least partially complete.

Use queries/server functions to get data from the server
The preload pattern above works because of the concept of queries.

In SolidStart, the query function wraps a server function with a unique key, enabling request deduplication, caching, and cache invalidation. Here’s a simple example:

const getPosts = query(async () => {
  "use server";
  return await db.from("posts").select();
}, "posts");
Use actions to mutate data
When it comes to mutating server data, actions are the missing half of queries.

If you use the concept of actions, revalidating data becomes so easy.

const addPost = action(async (formData: FormData) => {
  "use server";

  const post = await db
    .from("posts")
    .insert({ title: formData.get("title") })
    .select()
    .single();

  throw redirect(`/posts/${post.id}`);
}, "addPost");
When an action completes, SolidStart knows that server state has changed. If you redirect after the mutation, the router automatically revalidates all queries for the next route.

Because both queries and actions are keyed, you can also opt into fine-grained control by revalidating specific keys instead of the entire route when needed.


To preload data before a route renders:

Export a route object with a preload function.
Run your query inside the preload function.
Use the query as usual in your component.
TypeScript
JavaScript
// src/routes/index.tsx
import { ErrorBoundary } from "solid-js";
import { query, createAsync, type RouteDefinition } from "@solidjs/router";

const getPosts = query(async () => {
  const posts = await fetch("https://my-api.com/posts");
  return await posts.json();
}, "posts");

export const route = {
  preload: () => getPosts(),
} satisfies RouteDefinition;

export default function Page() {
  const post = createAsync(() => getPosts());
  return (
    <div>
      <ErrorBoundary fallback={<div>Something went wrong!</div>}>
        <h1>{post().title}</h1>
      </ErrorBoundary>
    </div>
  );
}
Passing parameters to queries
When creating a query that accepts parameters, define your query function to take any number of parameters:

TypeScript
JavaScript
// src/routes/posts/[id]/index.tsx
import { ErrorBoundary } from "solid-js";
import { query, createAsync, type RouteDefinition } from "@solidjs/router";

const getPost = query(async (id: string) => {
  const post = await fetch(`https://my-api.com/posts/${id}`);
  return await post.json();
}, "post");

export const route = {
  preload: ({ params }) => getPost(params.id),
} satisfies RouteDefinition;

export default function Page() {
  const postId = 1;
  const post = createAsync(() => getPost(postId));
  return (
    <div>
      <ErrorBoundary fallback={<div>Something went wrong!</div>}>
        <h1>{post().title}</h1>
      </ErrorBoundary>
    </div>
  );
}

Guides
Data mutation
This guide provides practical examples of using actions to mutate data in SolidStart.

Handling form submission
To handle <form> submissions with an action:

Ensure the action has a unique name. See the Action API reference for more information.
Pass the action to the <form> element using the action prop.
Ensure the <form> element uses the post method for submission.
Use the FormData object in the action to extract field data using the native FormData methods.
TypeScript
JavaScript
// src/routes/index.tsx
import { action } from "@solidjs/router";

const addPost = action(async (formData: FormData) => {
  const title = formData.get("title") as string;
  await fetch("https://my-api.com/posts", {
    method: "POST",
    body: JSON.stringify({ title }),
  });
}, "addPost");

export default function Page() {
  return (
    <form action={addPost} method="post">
      <input name="title" />
      <button>Add Post</button>
    </form>
  );
}
Passing additional arguments
To pass additional arguments to your action, use the with method:

TypeScript
JavaScript
// src/routes/index.tsx
import { action } from "@solidjs/router";

const addPost = action(async (userId: number, formData: FormData) => {
  const title = formData.get("title") as string;
  await fetch("https://my-api.com/posts", {
    method: "POST",
    body: JSON.stringify({ userId, title }),
  });
}, "addPost");

export default function Page() {
  const userId = 1;
  return (
    <form action={addPost.with(userId)} method="post">
      <input name="title" />
      <button>Add Post</button>
    </form>
  );
}
Showing pending UI
To display a pending UI during action execution:

Import useSubmission from @solidjs/router.
Call useSubmission with your action, and use the returned pending property to display pending UI.
TypeScript
JavaScript
// src/routes/index.tsx
import { action, useSubmission } from "@solidjs/router";

const addPost = action(async (formData: FormData) => {
  const title = formData.get("title") as string;
  await fetch("https://my-api.com/posts", {
    method: "POST",
    body: JSON.stringify({ title }),
  });
}, "addPost");

export default function Page() {
  const submission = useSubmission(addPost);
  return (
    <form action={addPost} method="post">
      <input name="title" />
      <button disabled={submission.pending}>
        {submission.pending ? "Adding..." : "Add Post"}
      </button>
    </form>
  );
}
Handling errors
To handle errors that occur within an action:

Import useSubmission from @solidjs/router.
Call useSubmission with your action, and use the returned error property to handle the error.
TypeScript
JavaScript
// src/routes/index.tsx
import { Show } from "solid-js";
import { action, useSubmission } from "@solidjs/router";

const addPost = action(async (formData: FormData) => {
  const title = formData.get("title") as string;
  await fetch("https://my-api.com/posts", {
    method: "POST",
    body: JSON.stringify({ title }),
  });
}, "addPost");

export default function Page() {
  const submission = useSubmission(addPost);
  return (
    <form action={addPost} method="post">
      <Show when={submission.error}>
        <p>{submission.error.message}</p>
      </Show>
      <input name="title" />
      <button>Add Post</button>
    </form>
  );
}
Validating form fields
To validate form fields in an action:

Add validation logic in your action and return validation errors if the data is invalid.
Import useSubmission from @solidjs/router.
Call useSubmission with your action, and use the returned result property to handle the errors.
TypeScript
JavaScript
// src/routes/index.tsx
import { Show } from "solid-js";
import { action, useSubmission } from "@solidjs/router";

const addPost = action(async (formData: FormData) => {
  const title = formData.get("title") as string;
  if (!title || title.length < 2) {
    return {
      error: "Title must be at least 2 characters",
    };
  }
  await fetch("https://my-api.com/posts", {
    method: "POST",
    body: JSON.stringify({ title }),
  });
}, "addPost");

export default function Page() {
  const submission = useSubmission(addPost);
  return (
    <form action={addPost} method="post">
      <input name="title" />
      <Show when={submission.result?.error}>
        <p>{submission.result?.error}</p>
      </Show>
      <button>Add Post</button>
    </form>
  );
}
Showing optimistic UI
To update the UI before the server responds:

Import useSubmission from @solidjs/router.
Call useSubmission with your action, and use the returned pending and input properties to display optimistic UI.
TypeScript
JavaScript
// src/routes/index.tsx
import { For, Show } from "solid-js";
import { action, useSubmission, query, createAsync } from "@solidjs/router";

const getPosts = query(async () => {
  const posts = await fetch("https://my-api.com/blog");
  return await posts.json();
}, "posts");

const addPost = action(async (formData: FormData) => {
  const title = formData.get("title");
  await fetch("https://my-api.com/posts", {
    method: "POST",
    body: JSON.stringify({ title }),
  });
}, "addPost");

export default function Page() {
  const posts = createAsync(() => getPosts());
  const submission = useSubmission(addPost);
  return (
    <main>
      <form action={addPost} method="post">
        <input name="title" />
        <button>Add Post</button>
      </form>
      <ul>
        <For each={posts()}>{(post) => <li>{post.title}</li>}</For>
        <Show when={submission.pending}>
          {submission.input?.[0]?.get("title")?.toString()}
        </Show>
      </ul>
    </main>
  );
}
Multiple Submissions
If you want to display optimistic UI for multiple concurrent submissions, you can use the useSubmissions primitive.

Redirecting
To redirect users to a different route within an action:

Import redirect from @solidjs/router.
Call redirect with the route you want to navigate to, and throw its response.
TypeScript
JavaScript
// src/routes/index.tsx
import { action, redirect } from "@solidjs/router";

const addPost = action(async (formData: FormData) => {
  const title = formData.get("title") as string;
  const response = await fetch("https://my-api.com/posts", {
    method: "POST",
    body: JSON.stringify({ title }),
  });
  const post = await response.json();
  throw redirect(`/posts/${post.id}`);
}, "addPost");

export default function Page() {
  return (
    <form action={addPost} method="post">
      <input name="title" />
      <button>Add Post</button>
    </form>
  );
}
Using a database or an ORM
To safely interact with your database or ORM in an action, ensure it's server-only by adding "use server" as the first line of your action:

TypeScript
JavaScript
// src/routes/index.tsx
import { action } from "@solidjs/router";
import { db } from "~/lib/db";

const addPost = action(async (formData: FormData) => {
  "use server";
  const title = formData.get("title") as string;
  await db.insert("posts").values({ title });
}, "addPost");

export default function Page() {
  return (
    <form action={addPost} method="post">
      <input name="title" />
      <button>Add Post</button>
    </form>
  );
}
Triggering an action programmatically
To programmatically trigger an action:

Import useAction from @solidjs/router.
Call useAction with your action, and use the returned function to trigger the action.
TypeScript
JavaScript
// src/routes/index.tsx
import { createSignal } from "solid-js";
import { action, useAction } from "@solidjs/router";

const addPost = action(async (title: string) => {
  await fetch("https://my-api.com/posts", {
    method: "POST",
    body: JSON.stringify({ title }),
  });
}, "addPost");

export default function Page() {
  const [title, setTitle] = createSignal("");
  const addPostAction = useAction(addPost);
  return (
    <div>
      <input value={title()} onInput={(e) => setTitle(e.target.value)} />
      <button onClick={() => addPostAction(title())}>Add Post</button>
    </div>
  );
}

API routes
While Server Functions can be a good way to write server-side code for data needed by your UI, sometimes you need to expose API routes. Some reasons for wanting API Routes include:

There are additional clients that want to share this logic.
Exposing a GraphQL or tRPC endpoint.
Exposing a public-facing REST API.
Writing webhooks or auth callback handlers for OAuth.
Having URLs not serving HTML, but other kinds of documents like PDFs or images.
For these use cases, SolidStart provides a way to write these routes in a way that is easy to understand and maintain. API routes are just similar to other routes and follow the same filename conventions as UI Routes.

The difference between API routes and UI routes is in what you should export from the file. UI routes export a default Solid component, while API Routes do not. Rather, they export functions that are named after the HTTP method that they handle.

note
API routes are prioritized over UI route alternatives. If you want to have them overlap at the same path remember to use Accept headers. Returning without a response in a GET route will fallback to UI route handling.

Writing an API route
To write an API route, you can create a file in a directory. While you can name this directory anything, it is common to name it api to indicate that the routes in this directory are for handling API requests:

export function GET() {
  // ...
}

export function POST() {
  // ...
}

export function PATCH() {
  // ...
}

export function DELETE() {
  // ...
}
API routes get passed an APIEvent object as their first argument. This object contains:

request: Request object representing the request sent by the client.
params: Object that contains the dynamic route parameters. For example, if the route is /api/users/:id, and the request is made to /api/users/123, then params will be { id: 123 }.
fetch: An internal fetch function that can be used to make requests to other API routes without worrying about the origin of the URL.
An API route is expected to return JSON or a Response object. In order to handle all methods, you can define a handler function that binds multiple methods to it:

async function handler() {
  // ...
}

export const GET = handler;
export const POST = handler;
// ...
An example of an API route that returns products from a certain category and brand is shown below:

import type { APIEvent } from "@solidjs/start/server";
import store from "./store";

export async function GET({ params }: APIEvent) {
  console.log(`Category: ${params.category}, Brand: ${params.brand}`);
  const products = await store.getProducts(params.category, params.brand);
  return products;
}
Session management
Since HTTP is a stateless protocol, you need to manage the state of the session on the server. For example, if you want to know who the user is, the most secure way of doing this is through the use of HTTP-only cookies. Cookies are a way to store data in the user's browser that persist in the browser between requests.

The user's request is exposed through the Request object. Through parsing the Cookie header, the cookies can be accessed and any helpers from vinxi/http can be used to make that a bit easier.

import type { APIEvent } from "@solidjs/start/server";
import { getCookie } from "vinxi/http";
import store from "./store";

export async function GET(event: APIEvent) {
  const userId = getCookie("userId");
  if (!userId) {
    return new Response("Not logged in", { status: 401 });
  }
  const user = await store.getUser(event.params.userId);
  if (user.id !== userId) {
    return new Response("Not authorized", { status: 403 });
  }
  return user;
}
In this example, you can see that the userId is read from the cookie and then used to look up the user in the store. For more information on how to use cookies for secure session management, read the session documentation.


Head and metadata
SolidStart does not come with a metadata library. In cases where you want to customize the content in the head of your document, you can use the @solidjs/meta library.

bash frame="none" npm i @solidjs/meta
The common elements used in the head are:

title: Specifies the title of the page, used by the browser tab and headings of search results.
meta: Specifies a variety of metadata about the page specified by name, ranging from favicon, character set to OG tags for SEO.
link: Adds assets like stylesheets or scripts for the browser to load for the page.
style: Adds inline styles to the page.
Inside a Route component
When applying metadata to a specific route, you can use the Title:

import { Title } from "@solidjs/meta";

export default function About() {
  return (
    <>
      <Title>About</Title>
      <h1>About</h1>
    </>
  );
}
These tags will be applied for that specific route only and will be removed from document.head once a user navigates away from the page. routeData can also be used here to create titles and SEO metadata that is specific to the dynamic parts of the route.

Adding a site suffix in Title
Custom components can be created to wrap the Title component to add a site-specific prefix to all the titles:

export default function MySiteTitle(props) {
  return <Title>{props.children} | My Site</Title>;
}
import MySiteTitle from "~/components/MySiteTitle";

export default function About() {
  return (
    <>
      <MySiteTitle>About</MySiteTitle>
      <h1>About</h1>
    </>
  );
}
Using async data in Title
Resources can be used to create titles specific to the dynamic parts of the route:

import { Title } from "@solidjs/meta";
import { RouteSectionProps } from "@solidjs/router";
import { createResource, Show } from "solid-js";

export default function User(props: RouteSectionProps) {
  const [user] = createResource(() => fetchUser(props.params.id));

  return (
    <Show when={user()}>
      <Title>{user()?.name}</Title>
      <h1>{user()?.name}</h1>
    </Show>
  );
}
For this example, routeData can be used to retrieve the user's name from the id in /users/:id and use it in the Title component. Similarly, other information can be used to build up other tags for SEO.

Adding SEO tags
SEO tags like og:title, og:description, og:image, use the Meta component. Since these tags may want to be used across multiple routes, they can be added inside the Head of the root.tsx file.

export default function Root() {
  return (
    <Html lang="en">
      <Head>
        <Meta property="og:image" content="https://example.com/image.jpg" />
        <Meta property="og:image:alt" content="Welcome to my site" />
        <Meta property="og:image:width" content="1200" />
        <Meta property="og:image:height" content="600" />
        <Meta property="og:site_name" content="GitHub" />
      </Head>
    </Html>
  );
}
If you need to add route-specific information inside your route, much like the Title component, you can use the Meta component within the desired route. This overrides the Meta tags used within the Head component.

import MySiteTitle from "~/components/MySiteTitle";

export default function About() {
  return (
    <>
      <MySiteTitle>About</MySiteTitle>
      <Meta name="description" content="This is my content tag." />
      <Meta property="og:title" content="Welcome to my site!" />
      <Meta property="og:description" content="A website" />
      <h1>About</h1>
    </>
  );
}


Middleware
Middleware intercepts HTTP requests and responses to perform tasks like authentication, redirection, logging, and more. It also enables sharing request-scoped data across the application using the event.locals object.

Common use cases
Here are some common use cases for middleware:

Request and response header management: Middleware allows modifying headers to control caching (e.g., Cache-Control), improve security (e.g., Content-Security-Policy), or implement custom behaviour based on request characteristics.
Global data sharing: The event.locals object allows storing and sharing request-scoped data between middleware and any server-side context (e.g., API routes, server-only queries/actions). This is useful for passing information like user authentication status, feature flags, or other request-related data.
Server-side redirects: Middleware can redirect users based on various request properties, such as locale, authentication state, or custom query parameters.
Request preprocessing: Middleware can perform lightweight preprocessing tasks, such as validating tokens or normalizing paths.
Limitations
While middleware is powerful, certain tasks are better handled in other parts of your application for performance, maintainability, or security reasons:

Authorization: Middleware does not run on every request, especially during client-side navigations. Relying on it for authorization would create a significant security vulnerability. As a result, authorization checks should be performed as close to the data source as possible. This means it within API routes, server-only queries/actions, or other server-side utilities.
Heavy computation or long-running processes: Middleware should be lightweight and execute quickly to avoid impacting performance. CPU-intensive tasks, long-running processes, or blocking operations (e.g., complex calculations, external API calls) are best handled by dedicated route handlers, server-side utilities, or background jobs.
Database operations: Performing direct database queries within middleware can lead to performance bottlenecks and make your application harder to maintain. Database interactions should be handled by server-side utilities or route handlers, which will create better management of database connections and handling of potential errors.
Basic usage
Middleware is configured by exporting a configuration object from a dedicated file (e.g., src/middleware/index.ts). This object, created using the createMiddleware function, defines when middleware functions execute throughout the request lifecycle.

import { createMiddleware } from "@solidjs/start/middleware";

export default createMiddleware({
  onRequest: (event) => {
    console.log("Request received:", event.request.url);

    event.locals.startTime = Date.now();
  },
  onBeforeResponse: (event) => {
    const endTime = Date.now();
    const duration = endTime - event.locals.startTime;
    console.log(`Request took ${duration}ms`);
  },
});
For SolidStart to recognize the configuration object, the file path is declared in app.config.ts:

import { defineConfig } from "@solidjs/start/config";

export default defineConfig({
  middleware: "src/middleware/index.ts",
});
Lifecycle events
A middleware function executes at specific points in the request lifecycle, using two key events: onRequest and onBeforeResponse.

onRequest
The onRequest event is triggered at the beginning of the request lifecycle, before the request is handled by the route handler. This is the ideal place to:

Store request-scoped data in event.locals for use in later middleware functions or route handlers.
Set or modify request headers.
Perform early redirects.
onBeforeResponse
The onBeforeResponse event is triggered after a request has been processed by the route handler but before the response is sent to the client. This is the ideal place to:

Set or modify response headers.
Log response metrics or perform other post-processing tasks.
Modify the response body.
Locals
In web applications, there's often a need to share request-specific data across different parts of the server-side code. This data might include user authentication status, trace IDs for debugging, or client metadata (e.g., user agent, geolocation).

The event.locals is a plain JavaScript object that can hold any JavaScript value. This object provides a temporary, request-scoped storage layer to address this need. Any data stored within it is only available during the processing of a single HTTP request and is automatically cleared afterward.

import { createMiddleware } from "@solidjs/start/middleware";

export default createMiddleware({
  onRequest: (event) => {
    event.locals.user = {
      name: "John Wick",
    };
    event.locals.sayHello = () => {
      return "Hello, " + event.locals.user.name;
    };
  },
});
Within middleware, event.locals can be accessed and modified directly. Other server-side contexts must use the getRequestEvent function to access the event.locals object.

import { getRequestEvent } from "solid-js/web";
import { query, createAsync } from "@solidjs/router";

const getUser = query(async () => {
  "use server";
  const event = getRequestEvent();
  return {
    name: event?.locals?.user?.name,
    greeting: event?.locals?.sayHello(),
  };
}, "user");

export default function Page() {
  const user = createAsync(() => getUser());

  return (
    <div>
      <p>Name: {user()?.name}</p>
      <button onClick={() => alert(user()?.greeting)}>Say Hello</button>
    </div>
  );
}
Headers
Request and response headers can be accessed and modified using the event.request.headers and event.response.headers objects. These follow the standard Web API Headers interface, exposing built-in methods for reading/updating headers.

import { createMiddleware } from "@solidjs/start/middleware";

export default createMiddleware({
  onRequest: (event) => {
    // Reading client metadata for later use
    const userAgent = event.request.headers.get("user-agent");
    // Adding custom headers to request/response
    event.request.headers.set("x-custom-request-header", "hello");
    event.response.headers.set("x-custom-response-header1", "hello");
  },
  onBeforeResponse: (event) => {
    // Finalizing response headers before sending to client
    event.response.headers.set("x-custom-response-header2", "hello");
  },
});
Headers set in onRequest are applied before the route handler processes the request, allowing downstream middleware or route handlers to override them. Headers set in onBeforeResponse are applied after the route handler and are finalized for the client.

Cookies
HTTP cookies are accessible through the Cookie request header and Set-Cookie response header. While these headers can be manipulated directly, Vinxi, the underlying server toolkit powering SolidStart, provides helpers to simplify cookie management. See the Vinxi Cookies documentation for more information.

import { createMiddleware } from "@solidjs/start/middleware";
import { getCookie, setCookie } from "vinxi/http";

export default createMiddleware({
  onRequest: (event) => {
    // Reading a cookie
    const theme = getCookie(event.nativeEvent, "theme");

    // Setting a secure session cookie with expiration
    setCookie(event.nativeEvent, "session", "abc123", {
      httpOnly: true,
      secure: true,
      maxAge: 60 * 60 * 24, // 1 day
    });
  },
});
Custom responses
Returning a value from a middleware function immediately terminates the request processing pipeline and sends the returned value as the response to the client. This means no further middleware functions or route handlers will be executed.

import { createMiddleware } from "@solidjs/start/middleware";

export default createMiddleware({
  onRequest: () => {
    return new Response("Unauthorized", { status: 401 });
  },
});
Only Response objects can be returned from middleware functions. Returning any other value will result in an error.

Redirects
Solid Router provides the redirect helper function which simplifies creating redirect responses.

import { createMiddleware } from "@solidjs/start/middleware";
import { redirect } from "@solidjs/router";

const REDIRECT_MAP: Record<string, string> = {
  "/signup": "/auth/signup",
  "/login": "/auth/login",
};

export default createMiddleware({
  onRequest: (event) => {
    const { pathname } = new URL(event.request.url);

    // Redirecting legacy routes permanently to new paths
    if (pathname in REDIRECT_MAP) {
      return redirect(REDIRECT_MAP[pathname], 301);
    }
  },
});
This example checks the requested path and returns a redirect response if it matches a predefined path. The 301 status code indicates a permanent redirect. Other redirect status codes (e.g., 302, 307) are available as needed.

JSON responses
Solid Router provides the json helper function which simplifies sending custom JSON responses.

import { createMiddleware } from "@solidjs/start/middleware";
import { json } from "@solidjs/router";

export default createMiddleware({
  onRequest: (event) => {
    // Rejecting unauthorized API requests with a JSON error
    const authHeader = event.request.headers.get("Authorization");
    if (!authHeader) {
      return json({ error: "Unauthorized" }, { status: 401 });
    }
  },
});
Chaining middleware functions
onRequest and onBeforeResponse options in createMiddleware can accept either a single function or an array of middleware functions. When an array is provided, these functions execute sequentially within the same lifecycle event. This enables composing smaller, more-focused middleware functions, rather than handling all logic in a single, large middleware function.

import { createMiddleware } from "@solidjs/start/middleware";
import { type FetchEvent } from "@solidjs/start/server";

function middleware1(event: FetchEvent) {
  event.request.headers.set("x-custom-header1", "hello-from-middleware1");
}

function middleware2(event: FetchEvent) {
  event.request.headers.set("x-custom-header2", "hello-from-middleware2");
}

export default createMiddleware({
  onRequest: [middleware1, middleware2],
});
The order of middleware functions in the array determines their execution order. Dependent middleware functions should be placed after the middleware functions they rely on. For example, authentication middleware should typically run before logging middleware.


Sessions
Sessions allow web applications to maintain state between user requests. Since HTTP is stateless, each request is treated independently. Sessions address this by allowing the server to recognize multiple requests from the same user, which is helpful for tracking authentication and preferences.

How sessions work
A session typically involves:

Session creation: When tracking is needed (e.g., upon login or first visit), the server creates a session. This involves generating a unique session ID and storing the session data, encrypted and signed, within a cookie.
Session cookie transmission: The server sends a Set-Cookie HTTP header. This instructs the browser to store the session cookie.
Subsequent requests: The browser automatically includes the session cookie in the Cookie HTTP header for requests to the server.
Session retrieval and data access: For each request, the server checks for the session cookie, retrieves the session data if a cookie is present, then decrypts and verifies the signature of the session data for the application to access and use this data.
Session expiration and destruction: Sessions typically expire after a period of time or upon user sign-out and the data is removed. This is done by setting a Max-Age attribute on the cookie or by sending a Set-Cookie HTTP header with an expired date.
Most of these steps are automatically managed by the session helpers.

Database sessions
For larger applications or when more advanced session management is needed, session data can be stored in a database. This approach is similar to the cookie-based approach, but with some key differences:

The session data is stored in the database, associated with the session ID.
Only the session ID is stored in the cookie, not the session data.
The session data is retrieved from the database using the session ID, instead of being retrieved directly from the cookie.
Upon expiration, in addition to the session cookie, the database record containing the session data is also removed.
SolidStart does not automatically handle interactions with a database; you need to implement this yourself.

Session helpers
Vinxi, the underlying server toolkit powering SolidStart, provides helpers to simplify working with sessions. It provides a few key session helpers:

useSession: Initializes a session or retrieves the existing session and returns a session object.
getSession: Retrieves the current session or initializes a new session.
updateSession: Updates data within the current session.
clearSession: Clears the current session.
These helpers work only in server-side contexts, such as within server functions and API routes. This is because session management requires access to server-side resources as well as the ability to get and set HTTP headers.

For more information, see the Cookies documentation in the Vinxi docs.

Creating a session
The useSession helper is the primary way to create and manage sessions. It provides a comprehensive interface for all session operations.

import { useSession } from "vinxi/http";

type SessionData = {
  theme: "light" | "dark";
};

export async function useThemeSession() {
  "use server";
  const session = await useSession<SessionData>({
    password: process.env.SESSION_SECRET as string,
    name: "theme",
  });

  if (!session.data.theme) {
    await session.update({
      theme: "light",
    });
  }

  return session;
}
In this example, the useThemeSession server function creates a session that stores a user's theme preference.

useSession requires a strong password for encrypting and signing the session cookie. This password must be at least 32 characters long and should be kept highly secure. It is strongly recommended to store this password in a private environment variable, as shown in the example above, rather than hardcoding it in your source code.

A password can be generated using the following command:

openssl rand -base64 32
useSession adds a Set-Cookie HTTP header to the current server response. By default, the cookie is named h3, but can be customized with the name option, as shown in the example above.

Getting the session data
The useSession helper provides access to the session data from the current request with the data property.

export async function getThemeSession() {
  "use server";
  const session = await useThemeSession();

  return session.data.theme;
}
Updating the session data
The useSession helper provides the update method to update the session data from the current request.

export async function updateThemeSession(data: SessionData) {
  "use server";
  const session = await useThemeSession();
  await session.update(data);
}
Clearing the session data
The useSession helper provides the clear method to clear the session data from the current request.

export async function clearThemeSession() {
  "use server";
  const session = await useThemeSession();
  await session.clear();
}

Returning responses
In SolidStart, it is possible to return a Response object from a server function. solid-router knows how to handle certain responses with its query and action APIs. For TypeScript, when returning a response using solid-router's redirect, reload, or json helpers, they will not impact the return value of the server function.

While we suggest depending on the type of the function to handle errors differently, you can always return or throw a response.

Examples
In the following example, the hello function will return a value of type Promise<{ hello: string }>:

import { json } from "@solidjs/router";
import { GET } from "@solidjs/start";

const hello = GET(async (name: string) => {
  "use server";
  return json(
    { hello: new Promise<string>((r) => setTimeout(() => r(name), 1000)) },
    { headers: { "cache-control": "max-age=60" } }
  );
});
However, in this example, since redirect and reload return never as their type, getUser can only return a value of type Promise<User>:

export async function getUser() {
  "use server";

  const session = await getSession();
  const userId = session.data.userId;
  if (userId === undefined) return redirect("/login");

  try {
    const user: User = await db.user.findUnique({ where: { id: userId } });
    // throwing can be awkward.
    if (!user) return redirect("/login");
    return user;
  } catch {
    // do stuff
    throw redirect("/login");
  }
}

Serialization
SolidStart serializes server function arguments and return values so they can travel between server and client. It uses Seroval under the hood and streams payloads to keep responses responsive.

Configuration
Configure serialization in your app.config.ts with defineConfig:

v1
v2
import { defineConfig } from "@solidjs/start/config";

export default defineConfig({
  serialization: {
    mode: "js",
  },
});
See the full config reference in defineConfig.

Modes
json: Uses JSON.parse on the client. Best for strict CSP because it avoids eval. Payloads can be slightly larger.
js: Uses Seroval's JS serializer for smaller payloads and better performance, but it requires unsafe-eval in CSP.
v2 Breaking Change: Defaults
SolidStart v1 defaults to js for backwards compatibility. SolidStart v2 defaults to json for CSP compatibility.

Supported types (default)
SolidStart enables Seroval plus a default set of web platform plugins. These plugins add support for:

AbortSignal, CustomEvent, DOMException, Event
FormData, Headers, ReadableStream
Request, Response
URL, URLSearchParams
Seroval supports additional value types. The compatibility list is broader than what SolidStart enables by default, so treat it as a superset. See the Seroval compatibility docs.

Limits and exclusions
RegExp is disabled by default.
JSON mode enforces a maximum serialization depth of 64. If you exceed this, flatten the structure or return a simpler payload.
Related guidance
Configure modes and defaults in defineConfig.
CSP implications and nonce examples live in the Security guide.

Auth
Server functions can be used to protect sensitive resources like user data.

"use server";

async function getPrivatePosts() {
  const user = await getUser();
  if (!user) {
    return null; // or throw an error
  }

  return db.getPosts({ userId: user.id, private: true });
}
The getUser function can be implemented using sessions.

Protected Routes
Routes can be protected by checking the user or session object during data fetching. This example uses Solid Router.

const getPrivatePosts = query(async function () {
  "use server";
  const user = await getUser();
  if (!user) {
    throw redirect("/login");
  }

  return db.getPosts({ userId: user.id, private: true });
});

export default function Page() {
  const posts = createAsync(() => getPrivatePosts(), { deferStream: true });
}
Once the user hits this route, the router will attempt to fetch getPrivatePosts data. If the user is not signed in, getPrivatePosts will throw and the router will redirect to the login page.

To prevent errors when opening the page directly, set deferStream: true. This would ensure getPrivatePosts resolves before the page loads since server-side redirects cannot occur after streaming has started.

WebSocket endpoint
WebSocket endpoint may be included by passing the ws handler file you specify in your start config. Note that this feature is experimental on the Nitro server and its config may change in future releases of SolidStart. Use it with caution.

import { defineConfig } from "@solidjs/start/config";

export default defineConfig({
  server: {
    experimental: {
      websocket: true,
    },
  },
}).addRouter({
  name: "ws",
  type: "http",
  handler: "./src/ws.ts",
  target: "server",
  base: "/ws",
});
Inside the ws file, you can export an eventHandler function to manage WebSocket connections and events:

import { eventHandler } from "vinxi/http";

export default eventHandler({
  handler() {},
  websocket: {
    async open(peer) {
      console.log("open", peer.id, peer.url);
    },
    async message(peer, msg) {
      const message = msg.text();
      console.log("msg", peer.id, peer.url, message);
    },
    async close(peer, details) {
      console.log("close", peer.id, peer.url);
    },
    async error(peer, error) {
      console.log("error", peer.id, peer.url, error);
    },
  },
});