# @sudobility/sider_lib

Sider business logic: the recipe compiler, egress gates, tokenizer, and
composite execution that turn a user's own browser session into safe,
agent-driven API calls. **Pure** — no Chrome, DOM, or server dependencies — so
the same code runs in `sider_api`, `sider_extension`, and `sider_app`.

Licensed under BUSL-1.1.

## What it is

Sider lets an agent act on a logged-in website by replaying the kinds of
requests the site's own front end makes, without ever letting a real secret
leave for the wrong place. This package holds the parts of that worth testing
in isolation from any browser or backend. The end-to-end flow is:

1. **Observe + tokenize** (`tokenize.ts`) — a raw request/response captured in
   the browser is scrubbed of secret values, which are replaced with readable
   placeholders (`{{secret:<slotId>}}`). Nothing with a real credential leaves
   the browser; the tokenizer also emits `SecretSlot` proposals describing where
   each secret lives (header / query / body / cookie).
2. **Compile** (`recipe.ts`) — a `ToolSpec` + arguments compile into a
   `CompiledRequest`. Secrets are compiled to an internal Private-Use-Area
   **sentinel** (`secretSentinel`), never to their values.
3. **Gate** (`gates.ts`) — `evaluateGates` runs the three egress gates on the
   compiled request (secrets still sentinels):
   - **Gate A — same-origin:** the destination must equal the site origin, and
     each slot's `allowedDestination` must match it.
   - **Gate B — injection location:** a secret's sentinel may appear *only* in
     its declared slot (header name / query param / body JSON path), and nowhere
     else. Cookie-borne secrets are never injected (the browser attaches them).
   - **Gate C — safety:** mutations are blocked where a site disallows them;
     `financial` actions require explicit user confirmation.
4. **Detokenize + prepare** (`execute.ts`) — only *after* the gates pass does
   `detokenizeRequest` substitute real secret values (read from a caller-supplied
   resolver backed by the live session) into the already-confined sentinel
   positions. `prepareRequest` is the single entry point: **compile → gate →
   detokenize** in one call, returning a request to issue only when `ok`.

Supporting modules:

- **`composite.ts`** — `executeComposite` threads multi-step / fan-out tools
  (e.g. "list sections → for each, fetch seats"). Pure orchestration: the caller
  injects a `StepRunner` that applies the gates and issues each real request.
- **`correlation.ts`** — `isCorrelationHeader` distinguishes request-tracing
  headers (traceparent, correlation ids, request ids) from real credentials, so
  they are dropped rather than mistaken for secrets and blocked at egress.
- **`chat-history.ts`** — pure, storage-free rules for client-side chat history:
  which conversation belongs to a site (`registrableDomain`, `latestChatFor`),
  and retention/pruning (`sweepChats`, `isExpired`, `orderedHistory`). Storage
  and UI live in the extension.

Clustering/templating (`clusterObservations`, `templatePath`, `endpointKey`,
`graphqlOperationOf`), recipe hashing (`hashRecipe`, `canonicalizeRecipe`), and
personal-data redaction (`redactPersonalText`, `isSensitiveValue`, …) live in
`@sudobility/sider_types` — shared with the backend so the rules are identical
on both sides — and are **re-exported** from this package's `index.ts` so its
public API stays stable. See `src/index.ts` for the full export surface.

## Requirements

- [Bun](https://bun.sh) (used for installing and testing)
- Access to GitHub Packages for the `@sudobility` scope. `.npmrc` already points
  that scope at `https://npm.pkg.github.com`; authenticate with a token that has
  `read:packages` before installing (see the org's package-access docs).

## Install

```bash
bun install
```

## Build

Compiles `src` to `dist` with `tsc`, then rewrites emitted relative imports to
Node-ESM-resolvable paths (`scripts/fix-esm-imports.mjs`):

```bash
bun run build
```

Note: the published package points `main`/`types` at `src/index.ts` (TypeScript
source), and ships both `src` and `dist`. Consumers typically compile it through
their own bundler.

## Test

```bash
bun test            # or: bun run test:unit
```

43 unit tests cover the gates, tokenizer, and chat-history rules
(`src/*.test.ts`).

## Type-check

```bash
bun run typecheck   # tsc --noEmit
```

## Project layout

```
src/
  index.ts          Public API (local exports + re-exports from sider_types)
  tokenize.ts       Browser-side secret tokenizer + SecretSlot proposals
  recipe.ts         compileRecipe: ToolSpec + args -> CompiledRequest (sentinels)
  gates.ts          evaluateGates: the three egress gates
  execute.ts        detokenizeRequest + prepareRequest (compile->gate->detokenize)
  composite.ts      executeComposite: multi-step / fan-out orchestration
  correlation.ts    isCorrelationHeader: tracing headers vs. credentials
  chat-history.ts   Pure chat-history rules (domain identity, retention)
  *.test.ts         Bun unit tests
scripts/
  fix-esm-imports.mjs  Post-build ESM import fixup
```

## CI / publishing

`.github/workflows/ci-cd.yml` delegates to the shared
`johnqh/workflows/.github/workflows/unified-cicd.yml` reusable workflow on pushes
and PRs to `main`/`develop`, publishing to **GitHub Packages**
(`publish-target: github`). Bump `version` in `package.json` to release.
