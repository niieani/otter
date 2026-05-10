# Boundaries

Where Effect meets the non-Effect world. I/O, external APIs, entry points, message sources -- these are boundaries. This doc defines the file conventions and rules for what's allowed at boundaries vs. interior code.

## File Convention

Boundary files come in two forms:

- **`*.adapter.ts`** suffix for individual adapter files (e.g., `github.adapter.ts`, `database.adapter.ts`)
- **`adapters/`** directory for apps with many adapters (each file inside is a boundary)

Entry point files are always boundaries: `index.ts`, `main.ts`, `server.ts`, `worker.ts`.

Everything else is interior code.

## What's Allowed Where

### Boundary files (`*.adapter.ts`, `adapters/`, entry points)

- `Effect.promise`, `Effect.tryPromise` -- wrapping external async APIs
- Raw `fetch`, `node:fs`, SDK clients -- if the result is wrapped as an Effect
- `ManagedRuntime.make` and `runtime.runPromise(effect)` -- entry points only, where you leave the Effect world
- `Effect.withSpan` required on all exported functions (observability at the edge)

### Interior (everything else)

- No direct I/O. All external access goes through services via `Context.Service`.
- No `Effect.runPromise` / `Effect.runSync`. Stay in Effect with `yield*`.
- No `console.*`. Use `Effect.log` / `Effect.logWarning` / `Effect.logError`.
- Errors via `Schema.TaggedErrorClass`. Services via `Context.Service`.

## Boundary Patterns by App Type

Each app type has a different shape at the boundary. The interior code is identical regardless.

### CLI

Parse args, build an Effect, provide layers, run once:

```typescript
// index.ts (boundary)
const command = yield * parseArgs();
const result = command.pipe(Effect.provide(AppContext));
Effect.runMain(result);
```

### API

Build a managed runtime once at startup. Each request enters Effect via
`runtime.runPromise(effect)`:

```typescript
// server.ts (boundary)
const runtime = ManagedRuntime.make(AppContext);

// Per-request boundary
const response = await runtime.runPromise(handleRequest(req));
```

### Worker

Message source feeds a per-message Effect handler. `runtime.runPromise` at the message boundary:

```typescript
// worker.ts (boundary)
const runtime = ManagedRuntime.make(AppContext);

// Per-message boundary
for await (const message of source) {
  await runtime.runPromise(handleMessage(message));
}
```

## Planned ast-grep Enforcement

These rules are planned but not yet implemented:

- **`require-span-at-boundary`**: Scoped to `*.adapter.ts` and `adapters/**`. Ensures all exported functions use `Effect.withSpan`.
- **`no-runpromise-in-effect` (update)**: Current rule flags `Effect.runPromise`/`runSync` inside Effect code. Boundary files should bridge with a `ManagedRuntime` instance.
