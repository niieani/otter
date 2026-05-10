# How to Build an Effect CLI

This is a reference template for CLI tools. For conventions that apply to all app
types, see the patterns docs (`docs/patterns/`). This template uses Effect v4,
Bun, and the native `effect/unstable/cli` module.

## Project Structure

Create the app in `apps/<name>/`:

```
apps/<name>/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts          # Entry point (Command.run, BunRuntime.runMain)
│   ├── services.ts       # Context.Service definitions
│   ├── layers.ts         # Layer composition
│   ├── errors.ts         # Schema.TaggedErrorClass definitions
│   └── commands/
│       ├── hello.ts      # One file per command or command group
│       └── ...
```

## package.json

```json
{
  "name": "@otter/<name>",
  "version": "0.0.1",
  "private": true,
  "type": "module",
  "bin": {
    "<name>": "src/index.ts"
  },
  "scripts": {
    "dev": "bun run src/index.ts",
    "build": "bun build src/index.ts --outdir dist --target bun",
    "lint": "oxlint src/",
    "format": "oxfmt --write src/",
    "typecheck": "tsgo --noEmit",
    "test": "bun test"
  },
  "dependencies": {
    "effect": "^4.0.0-beta.65",
    "@effect/platform-bun": "^4.0.0-beta.65"
  }
}
```

**Note:** Check the local `references/effect-smol/packages/*/package.json` versions
before creating package files. Effect v4 ecosystem packages share one version.

## tsconfig.json (app-level)

Extends the root config, adds build settings:

```json
{
  "extends": "../../tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "noEmit": false,
    "declaration": true,
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"]
}
```

## Services (services.ts)

Define services as `Context.Service` -- one per external dependency or configuration
source:

```typescript
import { Context, Effect, Layer, Schema } from "effect";

export class AppConfig extends Context.Service<
  AppConfig,
  {
    readonly configDir: string;
    readonly projectRoot: string;
    readonly verbose: boolean;
  }
>()("app/AppConfig") {
  static readonly layer = Layer.effect(
    AppConfig,
    Effect.gen(function* () {
      return AppConfig.of({
        configDir: ".",
        projectRoot: process.cwd(),
        verbose: false,
      });
    }),
  );
}

export class CommandError extends Schema.TaggedErrorClass<CommandError>()("CommandError", {
  command: Schema.String,
  reason: Schema.String,
}) {}
```

Keep services minimal. The shape and primary layer can live together when the service
is small; split implementation into adapters when it wraps external I/O.

## Layers (layers.ts)

Compose live implementations once:

```typescript
import { BunServices } from "@effect/platform-bun";
import { Layer } from "effect";
import { AppConfig } from "./services";

export const AppContext = Layer.mergeAll(AppConfig.layer, BunServices.layer);
```

## Commands (commands/\*.ts)

Commands return Effects. Prefer `Effect.fn("name")` for named spans and better
traces:

```typescript
import { Console, Effect } from "effect";
import { AppConfig, CommandError } from "../services";

export const hello = Effect.fn("hello")(function* (
  name: string,
): Effect.fn.Return<void, CommandError, AppConfig> {
  const config = yield* AppConfig;
  yield* Console.log(`Hello, ${name} from ${config.configDir}`);
});
```

## Entry Point (index.ts)

Use Effect's native CLI builder instead of yargs:

```typescript
#!/usr/bin/env bun
import { BunRuntime } from "@effect/platform-bun";
import { Effect } from "effect";
import { Argument, Command, Flag } from "effect/unstable/cli";
import { hello } from "./commands/hello";
import { AppContext } from "./layers";

const root = Command.make("<name>").pipe(
  Command.withSharedFlags({
    verbose: Flag.boolean("verbose").pipe(
      Flag.withAlias("v"),
      Flag.withDescription("Print diagnostic output"),
    ),
  }),
);

const helloCommand = Command.make(
  "hello",
  {
    name: Argument.string("name"),
  },
  Effect.fn("helloCommand")(function* ({ name }) {
    yield* hello(name);
  }),
).pipe(Command.withDescription("Say hello"));

root.pipe(
  Command.withSubcommands([helloCommand]),
  Command.run({ version: "0.0.1" }),
  Effect.provide(AppContext),
  Effect.catchCause((cause) =>
    Effect.gen(function* () {
      yield* Effect.logError("Command failed", cause);
      process.exitCode = 1;
    }),
  ),
  BunRuntime.runMain,
);
```

## Adding Observability

To add tracing to your CLI, see `docs/patterns/observability.md`. The key steps:

1. Add `@effect/opentelemetry` and matching OpenTelemetry dependencies
2. Create an env-var-gated `TracingLive` layer
3. Add `TracingLive` to `AppContext`
4. Use `Effect.fn("name")` or `Effect.withSpan` in commands
5. Run with `EFFECT_TRACE=1 bun run dev` to see traces

## Checklist

When building a new CLI app:

- [ ] Create `apps/<name>/` with the structure above
- [ ] Define services with `Context.Service`
- [ ] Define tagged errors with `Schema.TaggedErrorClass`
- [ ] Compose app layers once in `layers.ts`
- [ ] One file per command in `commands/`
- [ ] Wire commands with `effect/unstable/cli`
- [ ] Boundary convention: I/O in `*.adapter.ts` files, pure Effect everywhere else
- [ ] Add `Effect.fn("name")` or `Effect.withSpan` for observability
- [ ] Run `bun run check` from root
