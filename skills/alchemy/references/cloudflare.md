# Cloudflare on Alchemy v2

Every name below exists in `node_modules/@alchemy.run/cloudflare/src/`. Imports are `import * as Cloudflare from "@alchemy.run/cloudflare"`. Before you use a method not shown here, open the service's `index.ts`, then the file it re-exports. Working projects for almost every feature are in [examples/](https://github.com/alchemy-run/alchemy/tree/main/examples) as `cloudflare-*`.

## Bindings

Each binding has three parts. Declare the resource (lazy, fine at module scope). Bind it with `yield*` in the Worker's Construction phase to get a client. Provide the binding's Layer on the Construction Effect. Client calls need `RuntimeContext`, so they live in Runtime.

| Need | Declare | Bind in Construction | Provide |
| --- | --- | --- | --- |
| D1 | `Cloudflare.D1.Database("App", { migrations })` | `Cloudflare.D1.QueryDatabase(App)` | `Cloudflare.D1.QueryDatabaseBinding` |
| R2 | `Cloudflare.R2.Bucket("Files", { cors, lifecycleRules, domains, forceDestroy })` | `Cloudflare.R2.ReadBucket` / `WriteBucket` / `ReadWriteBucket(Files)` | `Cloudflare.R2.ReadBucketBinding` / `WriteBucketBinding` / `ReadWriteBucketBinding` |
| KV | `Cloudflare.KV.Namespace("Cache")` | `Cloudflare.KV.ReadNamespace` / `WriteNamespace` / `ReadWriteNamespace(Cache)` | the matching `...NamespaceBinding` |
| Queue producer | `Cloudflare.Queues.Queue("Jobs")` | `Cloudflare.Queues.WriteQueue(yield* Jobs)` | `Cloudflare.Queues.WriteQueueBinding` |
| Queue consumer | same queue | `Cloudflare.Queues.consumeQueueMessages(queue, (stream) => ...)` | `Cloudflare.Queues.EventSourceLive` |
| Rate limit | nothing to declare | `Cloudflare.RateLimit("NAME", { namespaceId, simple: { limit, period: 10 \| 60 } })` | `Cloudflare.RateLimitBinding` |
| Send email | nothing to declare | `Cloudflare.Email.Send(yield* Cloudflare.Email.SendEmail("EMAIL", { allowedSenderAddresses }))` | `Cloudflare.Email.SendBinding` |
| Cron | nothing to declare | `Cloudflare.Workers.cron("*/5 * * * *", (controller) => effect)` | `Cloudflare.Workers.CronEventSourceLive` |
| Background work | nothing to declare | `const exec = yield* Cloudflare.WorkerExecutionContext` | none |
| Another Worker (RPC) | its class | `Cloudflare.Workers.bindWorker(Greeter)` | none |
| Durable Object | its class | `const counters = yield* Counter` | none |

Client calls, all returning Effects:

- D1. `db.prepare(sql).bind(...values).all()`, `.first()`, `.run()`, plus `db.exec(sql)`, `db.batch([...])`, and `db.raw` for the native `D1Database` (handy for Drizzle).
- R2. `bucket.get(key)`, `head`, `list`, `put(key, body)`, `delete(keys)`. Presigned S3 URLs need `Cloudflare.R2.S3Credentials(Files, { access: "read-write" })`.
- KV. `kv.get(key)`, `kv.get(key, "text")`, `put`, `delete`.
- Queue. `queue.send(body)`, `queue.sendBatch(messages)`. The consumer handler gets one `Stream` of messages per batch. Success acks the batch and failure retries it. `consumeQueueMessages` also creates the `Queues.Consumer` resource.
- Rate limit. `limiter.limit({ key })` succeeds with `{ success }` and fails with tag `RateLimitError`. Namespaces are account-wide, so put the stage in the key when stages share a namespace.
- Email. `email.send({ from, to, subject, text, html })`.
- Background. `exec.waitUntil(effect)` runs the Effect after the response with the caller's services and keeps the invocation alive.

```ts
import * as Cloudflare from "@alchemy.run/cloudflare";
import * as Effect from "effect/Effect";
import * as HttpServerResponse from "effect/http/HttpServerResponse";
import * as Layer from "effect/Layer";
import { Files, Jobs } from "./resources.ts";

export default class Api extends Cloudflare.Worker<Api>()(
  "Api",
  { main: import.meta.url, compatibility: { date: "2026-09-08", flags: ["nodejs_compat"] } },
  Effect.gen(function* () {
    const files = yield* Cloudflare.R2.ReadWriteBucket(Files);
    const jobs = yield* Cloudflare.Queues.WriteQueue(yield* Jobs);
    const exec = yield* Cloudflare.WorkerExecutionContext;
    yield* Cloudflare.Workers.cron("0 * * * *", () => Effect.log("hourly"));
    return {
      fetch: Effect.gen(function* () {
        yield* files.put("hello.txt", "world");
        yield* exec.waitUntil(jobs.send({ kind: "uploaded", key: "hello.txt" }));
        return HttpServerResponse.text("ok");
      }).pipe(Effect.orDie), // R2Error and SendError are not HttpEffect errors
    };
  }).pipe(
    Effect.provide(
      Layer.mergeAll(
        Cloudflare.R2.ReadWriteBucketBinding,
        Cloudflare.Queues.WriteQueueBinding,
        Cloudflare.Workers.CronEventSourceLive,
      ),
    ),
  ),
) {}
```

A cron handler runs in Runtime. To reuse per-request services there, provide the same Layers inside the handler, or wrap both `fetch` and cron in one helper that provides them.

## Splitting the Worker identifier from its implementation

The one-call class above makes the stack import the Worker's whole implementation. When another Worker or the stack only needs the handle, declare the identifier alone and attach props and implementation with `.make()`:

```ts
// src/ApiWorker.ts: the identifier only, cheap to import
import type { RuntimeContext } from "alchemy";
import type { HttpEffect } from "alchemy/Http";
import * as Cloudflare from "@alchemy.run/cloudflare";

export class ApiWorker extends Cloudflare.Worker<ApiWorker, { fetch: HttpEffect<RuntimeContext> }>()("Api") {}
```

```ts
// src/Api.ts: a Layer that provides ApiWorker
export default ApiWorker.make({ main: import.meta.url }, Effect.gen(function* () { /* ... */ }));
```

The stack then does `yield* ApiWorker` and pipes `Effect.provide(ApiLive)` with the default export. The declared shape is the type `fetch` must meet, and the runtime provides `RuntimeContext` to `fetch`, so put it in the shape as above. With a bare `{ fetch: HttpEffect }`, client calls inside `fetch` fail to type-check until you provide `RuntimeContext` yourself.

## Worker props

All of these are on `WorkerProps` in `Workers/Worker.ts`.

- `main: import.meta.url` bundles the file whose default export is the Worker. Bundles go to `.alchemy/bundles`, and `build.output.dir` changes that.
- `compatibility: { date, flags }`. `nodejs_compat` is added for you only with a date before `2026-08-04`, so list it with a later date. `new_module_registry` is always added unless you set `legacy_module_registry`.
- `domain: "api.example.com"` or `{ name, aliases, redirects, zoneName }`. The zone must already exist on the account. Alchemy creates the custom domain, DNS and certificate. Omitting `domain` leaves custom domains unmanaged, and `domain: null` detaches all of them. `worker.url` is the first custom domain, otherwise the `workers.dev` URL.
- `routes: [{ pattern: "example.com/api/*", zoneName: "example.com" }]` for zone routes. `workersDev: false` turns off the `workers.dev` URL.
- `Cloudflare.Worker.URL` is the Worker's own URL. Use it in `env`, or `yield*` it in Construction to get an accessor, because a Worker cannot reference its own `url` Output.
- `assets: "./assets"` serves static files. `observability: { enabled: true }`, `placement: { mode: "smart" }`, `limits`, `tailConsumers` and `name` (pins the physical name) are the other common props.
- `dev: { port, host, strictPort }` for `alchemy dev`. The default Worker port is 1337, and a taken port moves to the next free one unless `strictPort` is set. `dev: { mode: "external", url }` skips the local Worker when another dev server serves it.
- `env: { ... }` is for async Workers and websites. Values route by shape. `Redacted<string>` and `Config.Redacted` become `secret_text`, `string` becomes `plain_text`, a Worker becomes a service binding, a resource becomes its native binding, and anything else becomes `json`. Type the handler's env with `Cloudflare.InferEnv<typeof MyWorker>`.

Alchemy always adds `ALCHEMY_PHASE=runtime`, `ALCHEMY_STAGE`, `ALCHEMY_STACK_NAME`, `ALCHEMY_WORKER_NAME` and `ALCHEMY_CLOUDFLARE_ACCOUNT_ID` as plain-text bindings (`Workers/WorkerProvider.ts`).

workerd has no isolate teardown, so finalizers added in Construction never run. Each request gets its own Scope, and `WorkerExecutionContext.waitUntil(effect)` keeps background work alive after the response.

## Website.Vite

`Cloudflare.Website.Vite("Web", { rootDir, main, env, domain, dev: { port }, assets, compatibility })` builds with the project's own Vite and deploys client assets plus the server bundle as one Worker. The build is skipped when the content hash of `rootDir` (gitignore-aware) and the lockfile is unchanged.

- `env` keys starting with `VITE_` are inlined into the client bundle as `import.meta.env.KEY` after Outputs resolve. A `VITE_` prefix also unwraps `Redacted`, so keep secrets unprefixed.
- Other `env` keys become Worker bindings for server code. `env: { API: api }` gives a service binding. Server code reads it with `const { env } = await import("cloudflare:workers")` and calls `env.API.fetch(request)`. Type it with `Cloudflare.InferEnv<typeof Web>` or `declare namespace Cloudflare { interface Env { API: Fetcher } }`.
- Alchemy injects its own Cloudflare Vite plugin and disables any `cloudflare()` from `@cloudflare/vite-plugin` left in `vite.config.ts`, so configs that need the official plugin (RedwoodSDK, Agents SDK) work unchanged. To skip the official factory's own setup too, guard it with `process.env.ALCHEMY_CLOUDFLARE_VITE_INJECTED === "1" ? null : cloudflare()`.
- `alchemy dev` routes assets like a deploy: SPA fallback is `assets: { notFoundHandling: "single-page-application" }`, and `assets: { runWorkerFirst: ["/api/*"] }` sends those paths to the Worker.
- Other frameworks have their own resource in `Website/`: Astro, Nextjs, Nuxt, SvelteKit, Vinext, Waku, Vocs, Foldkit, Octane and StaticSite.

## D1 migrations

`migrations: "./migrations"` or `{ dir, table }`. `dir` resolves against `process.cwd()`, the directory you run the CLI from. Pending files apply in order on every deploy, and on every reconcile under `alchemy dev`. Alchemy tracks them in `__alchemy_migrations`. A database first migrated by drizzle-kit or wrangler is adopted once: Alchemy copies the old history and leaves the old table frozen. `importFiles: [...]` imports extra `.sql` after migrations and re-imports only files whose content changed.

## Durable Objects

```ts
import * as Cloudflare from "@alchemy.run/cloudflare";
import * as Effect from "effect/Effect";

export default class Counter extends Cloudflare.DurableObject<Counter>()(
  "Counter",
  Effect.gen(function* () {
    const state = yield* Cloudflare.DurableObjectState; // outer: once per instance
    return Effect.gen(function* () {
      let count = (yield* state.storage.get<number>("count")) ?? 0;
      return {
        increment: () =>
          Effect.gen(function* () {
            count += 1;
            yield* state.storage.put("count", count);
            return count;
          }),
      };
    });
  }),
) {}
```

`yield* Counter` in a Worker's Construction registers the class and binding and returns a namespace. Call `counters.getByName(id).increment()` in Runtime. Methods that return Effects become typed RPC. Use `Cloudflare.SqlMigrations("./drizzle")` in the outer Effect and `migrations.apply()` in the inner one for SQLite tables. Durable Objects deploy with declarative `exports`. After the first such deploy, Alchemy versions before beta.82 cannot deploy the Worker, and that first deploy must go out at 100% traffic.

## State store

`Cloudflare.state()` keeps state in an `alchemy-state-store` Worker with a Durable Object on your account. The first run asks to bootstrap it, or run `alchemy state bootstrap --backend cloudflare` ahead. `Cloudflare.state({ name })` and `--name` pick a separate store.

## Local emulation under alchemy dev

Emulated locally: Workers (workerd), Durable Objects with SQLite, KV, R2 with an S3 endpoint for presigned URLs, D1 with migrations, Queues with retry and dead-letter, Workflows, Hyperdrive (connects straight to your database), Secrets Store, Containers in Docker, and websites on their own dev server. Simulator state sits under `.alchemy/local/`. Cron fires on schedule, and `/cdn-cgi/handler/scheduled` triggers it by hand. Email writes `.eml` files to `.alchemy/local/email`, and `POST /cdn-cgi/handler/email` drives an `email()` handler. Browser Rendering uses local Chrome. Images run through sharp. Anything else deploys live into the dev stage.

Pin a resource live with `.pipe(Alchemy.remote())`, for example D1 and R2 when browser uploads need real presigned R2 URLs.

## Credentials

`alchemy profile edit --add Cloudflare` signs in with OAuth or an API token. With `CI=true`, the CLI reads `CLOUDFLARE_ACCOUNT_ID` plus `CLOUDFLARE_API_TOKEN`, or `CLOUDFLARE_API_KEY` with `CLOUDFLARE_EMAIL`.
