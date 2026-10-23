# AWS on Alchemy v2

Every name below exists in `node_modules/@alchemy.run/aws/src/`. Import the barrel with `import * as AWS from "@alchemy.run/aws"`, or one service with `import * as S3 from "@alchemy.run/aws/S3"`. Each service directory has one file per operation, so `S3/PutObject.ts` holds the binding and `S3/PutObjectHttp.ts` its Layer. Working projects are in [examples/](https://github.com/alchemy-run/alchemy/tree/main/examples) as `aws-*`.

## Stack

```ts
// alchemy.run.ts
import * as AWS from "@alchemy.run/aws";
import * as Alchemy from "alchemy";
import * as Effect from "effect/Effect";
import Api from "./src/Api.ts";

export default Alchemy.Stack(
  "MyApp",
  { providers: AWS.providers(), state: AWS.state() },
  Effect.gen(function* () {
    const api = yield* Api;
    return { url: api.functionUrl };
  }),
);
```

`AWS.state()` keeps state in an account-regional S3 bucket, `alchemy-state-<accountId>-<region>-an`, created on the first deploy. `AWS.state({ name })` is a short logical name, and `bucketName` passes a full bucket name.

## Runtimes

- `AWS.Lambda.Function` is per-invocation and is the default for event sources.
- `AWS.Lambda.WebFunction` is an HTTP server that serves many requests per environment and streams responses. Use it for APIs and web apps. `authType: "ApplicationManaged"` makes the endpoint public, and `"IamAuth"` requires SigV4.
- ECS, EKS and EC2 follow the same bundle, deploy and bind pattern. See `https://alchemy.run/aws/compute/choosing-a-runtime`.

```ts
// src/Api.ts
import * as AWS from "@alchemy.run/aws";
import * as Effect from "effect/Effect";
import * as HttpServerResponse from "effect/http/HttpServerResponse";
import * as Layer from "effect/Layer";

export default class Api extends AWS.Lambda.Function<Api>()(
  "Api",
  { main: import.meta.url, functionUrl: true },
  Effect.gen(function* () {
    // Construction: declare, bind, register consumers.
    const files = yield* AWS.S3.Bucket("Files");
    const putObject = yield* AWS.S3.PutObject(files);
    return {
      fetch: Effect.gen(function* () {
        yield* putObject({ Key: "hello.txt", Body: "world" });
        return HttpServerResponse.text("ok");
      }).pipe(Effect.orDie),
    };
  }).pipe(Effect.provide(AWS.S3.PutObjectHttp)),
) {}
```

Function props live on `FunctionProps` in `Lambda/Function.ts`: `main`, `functionUrl` (`true` or a config with `authType`), `memorySize`, `timeout` (an Effect `Duration`), `architecture`, `runtime` (`nodejs22.x` or `nodejs24.x`), `env`. Alchemy bundles `main` with Rolldown into a zip.

## Bindings

A binding is one AWS operation on one resource. `yield* AWS.DynamoDB.GetItem(table)` adds that action on that table to the Function's generated IAM execution role and returns a callable client. Its Layer is the same name with `Http`, so `GetItem` needs `AWS.DynamoDB.GetItemHttp`. The client calls the AWS API with the Function's role and takes the raw request shape: `getItem({ Key: { id: { S: id } } })`.

Common ones: `S3.GetObject`, `PutObject`, `PresignGetObject`, `ListObjectsV2`. `DynamoDB.GetItem`, `PutItem`, `Query`, `UpdateItem`. `SQS.SendMessage`, `SendMessageBatch`. `SNS.Publish`. `ls` the service directory for the rest.

Wrap bindings in your own service so handlers stay small. Declare and bind inside a `Layer.effect`, and provide the `...Http` Layers under it:

```ts
export const JobStorageLive = Layer.provideMerge(
  Layer.effect(JobStorage, Effect.gen(function* () {
    const table = yield* AWS.DynamoDB.Table("Jobs", { partitionKey: "id", attributes: { id: "S" } });
    const getItem = yield* AWS.DynamoDB.GetItem(table);
    return JobStorage.of({ getJob: (id) => getItem({ Key: { id: { S: id } } }) });
  })),
  AWS.DynamoDB.GetItemHttp,
);
```

## Event sources

A `consume...` call in Construction creates the trigger (event source mapping, notification or schedule) and registers the handler. Each needs the Function-side Layer from `AWS.Lambda`:

| Source | Consume in Construction | Provide |
| --- | --- | --- |
| SQS | `AWS.SQS.consumeQueueMessages(queue, (stream) => ...)` | `AWS.Lambda.QueueEventSource` |
| SNS | `AWS.SNS.consumeTopicNotifications(topic, (stream) => ...)` | `AWS.Lambda.TopicEventSource` |
| DynamoDB Streams | `AWS.DynamoDB.consumeTableChanges(table, { streamViewType, startingPosition, batchSize }, (stream) => ...)` | `AWS.Lambda.TableEventSource` |
| S3 | `AWS.S3.consumeBucketEvents(bucket, ...)` | `AWS.Lambda.BucketEventSource` |
| Kinesis | `AWS.Kinesis.consumeStreamRecords(stream, ...)` | `AWS.Lambda.StreamEventSource` |
| Schedule | `AWS.Scheduler.consumeSchedule(AWS.Scheduler.every("5 minutes"), (event) => ...)` or `AWS.Scheduler.cron("cron(0 3 * * ? *)")` | `AWS.Lambda.ScheduleEventSource` |

Handlers get an Effect `Stream` of records, except `consumeSchedule`, whose handler gets one `ScheduleEvent`. A sink writes a stream out: `Stream.run(sink)` with `const sink = yield* AWS.SQS.QueueSink(queue)` and `AWS.SQS.QueueSinkHttp`. Many other services expose `consume...Events` functions for their EventBridge events.

## Websites

`AWS.Website.Vite`, `StaticSite`, `Nextjs`, `Astro`, `Nuxt`, `SvelteKit`, `SolidStart`, `TanStackStart`, `ReactRouter`, `Waku` and others deploy behind CloudFront. Server-rendered sites run on a Web Function by default. Set `server: "function"` to keep a Lambda Function URL server.

## alchemy dev

`alchemy dev` runs the stack on the floci emulator in one shared Docker container (`alchemy-floci`) with no AWS account or credentials. Lambdas run in local runtime containers behind working Function URLs on `localhost:4566`, and S3, DynamoDB, SQS, SNS, EventBridge and many more are emulated. Resource IDs carry the `dev:` marker. `.pipe(Alchemy.remote())` sends one resource to real AWS.

## Credentials

`alchemy profile edit --add AWS` offers SSO, console login (AWS CLI v2.32+) or stored access keys. `alchemy profile refresh` renews an expired session. `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_REGION`, with optional `AWS_SESSION_TOKEN`, beat the profile whenever the set is complete, CI or not. Under `CI=true` they are the only source. `AWS_ACCOUNT_ID` is optional, otherwise STS resolves it.
