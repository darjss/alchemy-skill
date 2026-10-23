---
name: alchemy
description: Alchemy v2 (alchemy.run), Infrastructure-as-Effects on Effect 4. Use when a project has `alchemy.run.ts` or imports `alchemy` or `@alchemy.run/*`, or when you write Stacks, Resources, Workers, Lambdas and their bindings, Config secrets or stages, or run `alchemy dev`, `deploy` or `destroy`. For v2 only, not v1's `await alchemy(...)`.
license: MIT
compatibility: Alchemy 2.0.0-beta.84+ with Effect 4. Notes for beta.83 and earlier inside.
metadata:
  author: darjss
  version: "0.1.0"
  alchemy-version: 2.0.0-beta.84
---

# Alchemy v2

## Check the version first

This skill matches `alchemy@2.0.0-beta.84`, checked against the alchemy source on main at commit `cb4ec94` (2026-10-22), which carries the beta.84 release notes ahead of the tag. Read the project's `package.json` before you write code:

- `@alchemy.run/<provider>` dependencies (`@alchemy.run/cloudflare`, `@alchemy.run/aws`) mean beta.84 or later. Everything here applies as written.
- `alchemy/Cloudflare` or `alchemy/AWS` imports mean beta.83 or earlier. The model and the APIs are the same, with these differences:
  - Providers ship inside `alchemy`. Import `alchemy/Cloudflare`, `alchemy/AWS`, `alchemy/Neon`, and read source under `node_modules/alchemy/src/Cloudflare/`.
  - The state store bootstraps with `alchemy provider cloudflare bootstrap`.
  - The CLI needs Node 24.11.1+ or Bun.
  - A `vite.config.ts` that keeps `cloudflare()` from `@cloudflare/vite-plugin` must drop it while `process.env.ALCHEMY_CLOUDFLARE_VITE_INJECTED === "1"`, or dev starts two workerd runtimes and only one has bindings.
  - Upgrading means installing each `@alchemy.run/<provider>`, rewriting imports and bumping `effect` to `^4.0.5`. The full list is in the [beta.84 notes](https://github.com/alchemy-run/alchemy/blob/main/website/src/content/docs/blog/2026-10-22-beta-84.md).
- `await alchemy("app")` and `async` resources mean v1, a different API documented at [v1.alchemy.run](https://v1.alchemy.run). Use this skill there only to port the project, with [/migrating-from-v1](https://alchemy.run/migrating-from-v1).

The installed source wins over this skill and over the website. Every package ships `src/`, so check an API by searching one file with `rg -n`: core in `node_modules/alchemy/src/`, a provider in `node_modules/@alchemy.run/<provider>/src/<Service>/`. The [where to look](#where-to-look) table names the file.

Provider depth lives in references. Read the one you need before you add or change a resource or binding:

- [references/cloudflare.md](references/cloudflare.md): Workers, every common binding, Website.Vite, D1 migrations, Durable Objects, local emulation.
- [references/aws.md](references/aws.md): Lambda Functions and Web Functions, `...Http` binding layers, event sources, credentials, local emulation.
- Any other provider (GCP, Fly, Neon, PlanetScale, Stripe, GitHub, Docker, Kubernetes and the rest): find the closest example in [alchemy-run/alchemy/examples](https://github.com/alchemy-run/alchemy/tree/main/examples) (`<provider>-<thing>`, such as `neon-website-vite` or `gcp-pubsub-worker`), then read the package's `src/index.ts` and the docs at `https://alchemy.run/<provider>`.

## Mental model

- A **Stack** is the program in `alchemy.run.ts`. `Alchemy.Stack(name, { providers, state, secrets? }, Effect.gen(...))` yields resources and returns outputs. `providers` and `state` are Layers, one per provider package (`Cloudflare.providers()`, `AWS.state()`).
- A **Resource** is a value you `yield*`. Declaring one is lazy, so `export const Files = Cloudflare.R2.Bucket("Files")` at module scope is fine. The first argument is the logical ID, which keys state.
- An **Output** is a resource attribute that resolves during deploy, like `worker.url` or `fn.functionName`. Pass it into other props to create a dependency edge.
- A **Platform** is a resource that carries code: a Cloudflare Worker, an AWS Lambda Function, a Durable Object. Its Effect has two **phases**. **Construction** is the outer `Effect.gen`. It runs at plan time to discover bindings and again at every cold start. **Runtime** is the handler it returns (`fetch`, a queue consumer). It runs per invocation and never at plan time.
- A **Binding** is a `yield*` inside Construction, like `yield* Cloudflare.D1.QueryDatabase(Db)` or `yield* AWS.S3.PutObject(bucket)`. It wires the permission or native binding into the Platform and returns a typed client. You also provide an implementation Layer, picked by where the code runs: `...Binding` is a native Cloudflare binding and works only on a Worker, `...Http` calls the provider's API with scoped credentials and works from any host (an AWS `...Http` on a Worker creates an IAM user and role), and `...Local` uses your own credentials, for Actions. A missing Layer shows up as an unsatisfied requirement type error.
- Client methods need `RuntimeContext`, which only Runtime has. The type system rejects calling them in Construction. Capture the client in Construction, call it in Runtime.

## Stack file

```ts
// alchemy.run.ts
import * as Cloudflare from "@alchemy.run/cloudflare";
import * as Alchemy from "alchemy";
import * as Effect from "effect/Effect";
import Api from "./src/Api.ts";

export default Alchemy.Stack(
  "MyApp",
  { providers: Cloudflare.providers(), state: Cloudflare.state() },
  Effect.gen(function* () {
    const api = yield* Api;
    return { url: api.url.as<string>() };
  }),
);
```

```ts
// src/Api.ts: a Platform class. Props, then the Construction Effect.
import * as Cloudflare from "@alchemy.run/cloudflare";
import * as Config from "effect/Config";
import * as Effect from "effect/Effect";
import * as HttpServerResponse from "effect/http/HttpServerResponse";
import * as Redacted from "effect/Redacted";

export const Files = Cloudflare.R2.Bucket("Files");

export default class Api extends Cloudflare.Worker<Api>()(
  "Api",
  { main: import.meta.url, compatibility: { date: "2026-09-08", flags: ["nodejs_compat"] } },
  Effect.gen(function* () {
    // Construction: plan time and cold start. Bind here.
    const files = yield* Cloudflare.R2.ReadWriteBucket(Files);
    const apiKey = yield* Config.Redacted("API_KEY");
    return {
      // Runtime: per request.
      fetch: Effect.gen(function* () {
        const object = yield* files.get("hello.txt");
        return HttpServerResponse.text(object ? "found" : "missing");
      }).pipe(Effect.orDie),
    };
  }).pipe(Effect.provide(Cloudflare.R2.ReadWriteBucketBinding)),
) {}
```

The same shape on AWS is `AWS.Lambda.Function<Api>()("Api", { main: import.meta.url, functionUrl: true }, ...)` with `AWS.providers()` and `AWS.state()`. See [references/aws.md](references/aws.md).

- Props are a plain object or an Effect of props, so they can depend on the stage: `Stack.useSync((stack) => ({ main: import.meta.url, memorySize: stack.stage === "prod" ? 1024 : 512 }))`, with `Stack` from `alchemy/Stack`.
- Several providers merge with `Layer.mergeAll(Cloudflare.providers(), AWS.providers())`.
- A `fetch` handler is an `HttpEffect` (`alchemy/Http`). Its error channel allows only `HttpServerError | HttpBodyError`, so catch domain errors (`Effect.catchTag`, `Effect.orDie`) before you return it.
- `const Announce = Alchemy.Action("Announce", (input) => effect)` defines a deploy-time step with no cloud object behind it. Call it in the Stack, `yield* Announce({ url: api.url })`. Its body runs when the input changes or under `--force`.

## Config, secrets and Outputs in props

- Read env values with `effect/Config` in Construction: `Config.Redacted("KEY")`, `Config.String`, `Config.schema(Schema.URLFromString, "APP_URL")`. Alchemy reads the value at deploy time and binds it to the Platform, and the same `Config` reads the binding at runtime. A `Config` yielded only inside a handler is never discovered and never bound.
- A `Config` yielded in Construction always binds as a secret (`secret_text` on Cloudflare), whichever constructor you used. The raw source is bound. Combinators such as `Config.withDefault` run again at runtime, and a default is never bound.
- Unwrap with `Redacted.value(secret)` at the call site. `` `Bearer ${secret}` `` sends the literal `<redacted>`.
- Without a `secrets` option, the CLI reads `.env` from the working directory into Effect `Config` only, not into `process.env`. Shell values beat `.env`. With `--env-file path` the file beats the shell. Code in `alchemy.run.ts` that reads `process.env` needs `process.loadEnvFile(...)` under Node. Under Bun, which the CLI switches to inside `bun run`, Bun itself also loads `.env` into `process.env`.
- The Stack's `secrets` option picks sources in order, later ones winning, with the shell implicitly last: `secrets: ({ stage }) => stage === "prod" ? Doppler.Secrets({ project: "app", config: "prod" }) : Secrets.DotEnv()`, with `Secrets` from `alchemy`. Doppler and Infisical ship as `@alchemy.run/doppler` and `@alchemy.run/infisical`. A stack with `secrets` rejects `--env-file`.
- A prop value can be a literal, an `Output`, a `Config`, or `Output.fromEffect(effect)`. A bare `Effect` nested in props passes through unresolved and ships as a serialized Effect object. Wrap it: `allowedOrigins: [Output.fromEffect(appOrigin.pipe(Effect.orDie))]`. The Effect must not fail, so `orDie` it.
- Combine Outputs with `Output.map`, `Output.all(a, b)` and `` Output.interpolate`https://${host}` `` from `alchemy/Output`.
- `Random("Id")` from `alchemy` mints a value once and keeps it in state. `.text` is a `Redacted<string>`.

## Stages, names and removal

- The stage comes from `--stage`, then `$ALCHEMY_STAGE` (shell, `--env-file` or `.env`), then `live_$USER` for deploy, plan, destroy, logs and drift, and `dev_$USER` for `alchemy dev`. Names match `[a-z0-9][-_a-z0-9]*`, case-insensitive.
- During plan, read it with `(yield* Stack).stage` (`Stack` from `alchemy/Stack`) or `yield* Stage` from `alchemy/Stage`. A deployed Worker has no `Stage` service. Cloudflare binds `ALCHEMY_STAGE` as plain text, so code that runs in both phases uses `Effect.serviceOption(Stage)` and falls back to `Config.String("ALCHEMY_STAGE")`.
- Physical names default to `${stack}-${logicalId}-${stage}-${16-char suffix}`. The suffix comes from the instance ID in state, so names stay stable across deploys. Providers sanitize and lowercase where the cloud requires it. Pass `name` to pin one.
- Renaming a logical ID plans a delete and a create. `.pipe(Alchemy.renamedFrom("OldId"))` moves the state row instead, and the cloud object stays.
- `.pipe(RemovalPolicy.retain())` (`alchemy/RemovalPolicy`) keeps the cloud object on destroy or when you delete the declaration. Alchemy drops the state row and skips the provider's delete. `retain(stack.stage === "prod")` or `retain(effectOfBoolean)` makes it per stage. A policy change plans as a no-op and still takes effect on the next deploy. A few types default to retain, such as `Cloudflare.Zone` and `GitHub.Repository`. Pipe `RemovalPolicy.destroy()` to let Alchemy delete them.

## State and credentials

- `state` is required. Each provider ships a remote store: `Cloudflare.state()` is a Worker with a Durable Object on your account, and `AWS.state()` is an account-regional S3 bucket. `Alchemy.localState()` writes `.alchemy/state/<stack>/<stage>/` on disk. `Cloudflare.state({ name: "team-a" })` picks a named store.
- Create a store ahead with `alchemy state bootstrap --backend cloudflare [--name team-a]`. In CI, a missing or outdated store fails unless you pass `--yes`.
- Pick local state under dev only: `Layer.unwrap(AlchemyContext.pipe(Effect.map(({ dev }) => dev ? Alchemy.localState() : Cloudflare.state())))`, with `AlchemyContext` from `alchemy/AlchemyContext`.
- Inspect with `alchemy state list` and `alchemy state read <path>`. `alchemy state delete` drops records and leaves the cloud object alive.
- Credentials come from profiles: `alchemy profile edit --add Cloudflare` (or `AWS`) saves to `~/.alchemy/profiles.json`, and `--profile <name>` picks one. A provider's full set of env vars (`CLOUDFLARE_API_TOKEN` with `CLOUDFLARE_ACCOUNT_ID`, or the AWS keys with `AWS_REGION`) beats any profile, CI or not, and that includes values in `.env`. With `CI=true`, env vars are the only source.

## alchemy dev

- Platforms run locally with hot reload, and each provider brings its own emulation: workerd and local simulators for Cloudflare, the floci emulator in Docker for AWS. Local resource IDs start with `dev:`. A resource with no local provider deploys for real into the dev stage.
- `.pipe(Alchemy.remote())` runs one resource, or a whole piped scope, against the real cloud in dev. It is a no-op on deploy. Moving a resource between local and remote plans a replacement. Yield each resource in one mode only: the same resource yielded both inside and outside `remote()` fails with `ConflictingProviderModeError`, so declare it once and share the value.
- `yield* ALCHEMY_DEV` (from `alchemy`) or `(yield* AlchemyContext).dev` tells you the run is `alchemy dev`.

## References across stages and stacks

`yield* Cloudflare.R2.Bucket.ref("Files", { stage: "staging" })` reads an already deployed resource from state and types it like the real one. Every resource type has `.ref`. Add `stack: "other"` for another stack. The upstream stage must be deployed first, or the plan fails with `InvalidReferenceError`. Destroying the borrower never touches the referenced resource, so `pr-*` preview stages can borrow staging's database. For a whole stack's outputs use `yield* Output.stackRef<{ url: string }>("Backend", { stage: "prod" })`, which returns an Output.

## CLI

Run it from the directory that holds `alchemy.run.ts`, or pass `-c <file>`. Commands: `deploy`, `plan` (same as `deploy --dry-run`), `destroy`, `dev`, `logs [--tail] [--since 1h]`, `state`, `drift`, `profile`, `provider`. Common flags are `--stage`, `--env-file`, `--profile`, `--include`/`--exclude` with FQN globs, and `--no-input` for plain output. Deploy also takes `--yes` to skip the prompt, `--adopt` for cloud resources that already exist, and `--force` to update unchanged resources and rerun Actions. Flags differ per subcommand, so check `alchemy <command> --help`.

The CLI needs Node 22.22.3+, Node 24.11.1+ or Bun. Its launcher hands off to Bun whenever `npm_execpath` points at bun, as it does under `bun run`. To force Node inside a Bun project, run `env -u npm_execpath -u npm_config_user_agent node ./node_modules/alchemy/bin/cli.js deploy`.

## Custom resources

Write a provider when no package covers a service: a `Resource` type plus a `Provider`. `reconcile`, `delete` and `list` are required, and `list` may return `[]` when the service has no way to enumerate. `read` and `diff` are optional. A provider reads its instance with `const { instanceId } = yield* ResourceContext`. The guide is [/infrastructure-as-code/custom-provider](https://alchemy.run/infrastructure-as-code/custom-provider), and upstream's own `.agents/skills/alchemy-provider` skill covers the conventions for contributing one.

## Where to look

Core paths are under `node_modules/alchemy/src/`, provider paths under `node_modules/@alchemy.run/<provider>/src/`. Docs are on https://alchemy.run, indexed at https://alchemy.run/llms.txt.

| Question | Source | Docs |
| --- | --- | --- |
| Stack options, `secrets` | `Stack.ts`, `Secrets/` | `/infrastructure-as-code/stack`, `/environments/secret-providers` |
| Output operators, `fromEffect`, `stackRef` | `Output.ts` | `/infrastructure-as-code/outputs`, `/infrastructure-as-code/references` |
| Which prop values resolve | `Plan.ts` (`resolveInput`) | `/infrastructure-as-code/outputs` |
| Phases, `ALCHEMY_DEV` | `Phase.ts`, `RuntimeContext.ts` | `/infrastructure-as-effects/phases` |
| Bindings, event sources, sinks | `Binding.ts`, provider `<Service>/index.ts` | `/infrastructure-as-effects/binding`, `/infrastructure-as-effects/event-sources` |
| Physical names, renames | `PhysicalName.ts`, `Rename.ts` | `/infrastructure-as-code/resource`, `/infrastructure-as-code/renaming` |
| retain, remote | `RemovalPolicy.ts`, `ProviderMode.ts` | `/infrastructure-as-code/resource-lifecycle`, `/environments/local-development` |
| State stores | `State/LocalState.ts`, provider `StateStore/State.ts` | `/state-store` |
| `.env` loading, stage default | `Util/ConfigProvider.ts`, `Cli/commands/flags.ts` | `/environments/secrets`, `/environments/stages` |
| Profiles, CI credentials | `Auth/Resolve.ts`, provider `AuthProvider.ts` or `Auth/AuthProvider.ts` | `/environments/profiles`, `/environments/ci` |
| CLI commands and flags | `Cli/commands/` | `/cli` |
| Custom resource and provider | `Resource.ts`, `Provider.ts` | `/infrastructure-as-code/custom-provider` |
