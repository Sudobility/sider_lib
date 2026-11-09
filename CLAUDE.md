# CLAUDE.md

Guidance for working in this repo. Keep it accurate; see `README.md` for fuller
prose.

## What this is

`@sudobility/sider_lib` — pure Sider business logic (recipe compiler, egress
gates, tokenizer, composite execution, chat-history rules). **No Chrome / DOM /
server dependencies**; imported by `sider_api`, `sider_extension`, and
`sider_app` alike. Keep it that way — do not add runtime deps on browser or
Node-only APIs.

## Stack

- TypeScript, ESM-only (`"type": "module"`), `strict`, `isolatedModules`,
  `verbatimModuleSyntax`, `moduleResolution: "bundler"`.
- Bun for install/test; `tsc` for build/typecheck.
- Sole dependency: `@sudobility/sider_types` (peer + dev). Needs GitHub Packages
  auth (`read:packages`) — `.npmrc` scopes `@sudobility` to `npm.pkg.github.com`.

## Commands (verified)

```bash
bun install          # needs GitHub Packages auth for @sudobility scope
bun test             # 43 tests; alias: bun run test:unit
bun run typecheck    # tsc --noEmit
bun run build        # tsc -> dist, then scripts/fix-esm-imports.mjs
```

There is no lint script and no bundler config in this repo.

## Architecture (the security spine)

The pipeline is **tokenize → compile → gate → detokenize**, and the ordering is
the security guarantee — do not reorder or short-circuit it.

- `tokenize.ts` — scrubs real secret values out of captured observations,
  replacing them with `{{secret:<slotId>}}` placeholders, and proposes
  `SecretSlot`s describing each secret's injection location. A slot is proposed
  only for a value actually *found* in the request (never defaulted).
- `recipe.ts` — `compileRecipe` turns `ToolSpec` + args into a `CompiledRequest`
  with secrets as PUA **sentinels** (`secretSentinel`), never values.
- `gates.ts` — `evaluateGates` runs on the sentinel form: Gate A (same-origin),
  Gate B (sentinel appears only in its declared location, cookie secrets never
  injected), Gate C (mutation policy; `financial` needs `confirmed`).
- `execute.ts` — `detokenizeRequest` substitutes real values **only after** gates
  pass. `prepareRequest` is the one public entry point (compile→gate→detokenize).
- `composite.ts` — `executeComposite` orchestrates multi-step/fan-out tools;
  caller supplies the `StepRunner` that applies gates + issues each request.
- `correlation.ts` / `chat-history.ts` — see conventions below.

`index.ts` re-exports clustering, recipe hashing, and personal-data redaction
**from `@sudobility/sider_types`** (shared with the backend). Add such shared
rules there, not here, so both sides stay identical; re-export to keep the API
stable.

## Conventions & gotchas

- **Secrets are sentinels until the last mile.** Gates must run on sentinel
  form; detokenization is the only place real values appear, and only after
  `ok`. Never gate on, log, or compile values.
- **Correlation headers are not credentials.** `isCorrelationHeader` must be
  consulted before minting a slot from a header name — a per-page trace/
  correlation/request id has no local value to resolve and would otherwise block
  the call at egress. `CREDENTIAL_RE` wins over `CORRELATION_RE`.
- **Cookie-borne secrets (`role: auto_cookie` / `at: "cookie"`)** are attached by
  the browser and must never be injected or substituted.
- **Tokenizer bias is conservative-for-privacy:** over-mask rather than leak.
- **`chat-history.ts` is pure and storage-free** — storage/UI live in the
  extension. Chat identity is the registrable domain, not host/URL; keep the
  `MULTI_PART_SUFFIXES` approach (no full public-suffix-list dependency).
- **ESM import fixup:** `moduleResolution: "bundler"` lets `tsc` emit
  extensionless relative imports that Node's ESM loader rejects;
  `scripts/fix-esm-imports.mjs` rewrites them after build. Keep `build` running
  it; it is idempotent and dependency-free.
- `dist/` is gitignored and generated — do not edit or commit it.

## Release

Bump `version` in `package.json`; push to `main`/`develop`. CI
(`.github/workflows/ci-cd.yml`) uses the shared
`johnqh/workflows/.../unified-cicd.yml` and publishes to GitHub Packages.
