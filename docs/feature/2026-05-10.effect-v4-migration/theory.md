# Effect v4 Scaffold Migration

## Problem Thesis

The scaffold teaches Effect v3 idioms: `Context.Tag` services, `Schema.decodeUnknown`,
`Schema.parseJson`, `Effect.catchAll`, `Runtime.runPromise(runtime)`, and platform
packages such as `@effect/platform`. Those examples become wrong defaults for new
apps generated from this template once the project targets Effect v4.

## Operating Theory

The migration is documentation- and rule-driven. This repo is a scaffold, not an app;
its most important runtime behavior is the guidance encoded in `docs/`, `AGENTS.md`,
`README.md`, and ast-grep examples. The correct leverage is to make every example and
lint note point to v4 APIs so future agents copy v4 code.

Effect v4 keeps Effect, Layer, Schema, and ManagedRuntime, but changes the shape of
common scaffold patterns:

- services use `Context.Service`, usually with a static `layer`;
- default custom errors should use `Schema.TaggedErrorClass`;
- Schema decoding APIs use `decodeUnknownEffect` / `decodeEffect`;
- JSON string parsing uses `Schema.fromJsonString`;
- multi-literal and union schemas take arrays via `Schema.Literals([...])` and
  `Schema.Union([...])`;
- `Effect.catchAll` is now `Effect.catch`;
- detached background work uses `Effect.forkDetach`;
- external framework bridges call `runtime.runPromise(effect)`.

## Strategy

Update tests first where the rules encode old API names, then update the rules to make
those tests pass. After the enforcement layer speaks v4, rewrite docs/templates to align
with those rules.

Verification is mostly rule/doc oriented because the scaffold currently has no app or
package TypeScript inputs. `sg test`, `bun run lint`, `bun run lint:ast`, and
`bun run format:check` are the meaningful gates. `bun run check` is partially blocked
by a missing host `drift` binary; `bun run typecheck` reports no TS inputs; `bun run
test` reports no workspace packages matched.

## Open Questions

`Data.TaggedError` still exists in v4 source, but the v4 LLMS guidance and Schema
migration docs prefer `Schema.TaggedErrorClass` for custom application errors. The
scaffold should prefer the documented v4 pattern while allowing legacy `Data.TaggedError`
only where existing rules need to recognize older code during migration.

## Research Artifacts & References

[^migration]:
    `references/effect-smol/MIGRATION.md` describes v4 package consolidation,
    unstable modules, and links migration details.

[^services]:
    `references/effect-smol/migration/services.md` maps `Context.Tag` to
    `Context.Service`.

[^schema]: `references/effect-smol/migration/schema.md` maps Schema API renames.

[^runtime]: `references/effect-smol/migration/runtime.md` explains `Runtime<R>` removal.

[^llms]: `references/effect-smol/LLMS.md` gives current v4 authoring guidance.
