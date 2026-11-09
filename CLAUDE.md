# CLAUDE.md

> **Git policy — never auto-commit or auto-push.** Leave work in the working tree. Run `git commit`,
> `git push`, `gh pr create`, or `push_all.sh` **only when the user explicitly asks in that turn**.
> Approval for an earlier change does not carry forward; finishing a task is not permission to commit.

## What this is

`@sudobility/raidr_processor` (BUSL-1.1) is the **pure core** of raidr: the capture bundle format,
redaction, coverage, API/route inference, code generation, and a passive security audit. Callers
supply bytes and records; this package returns values. **It never performs I/O and never touches the
DOM** (see Invariants). Public API and `RAIDR_FORMAT_VERSION` (= 1) are in `src/index.ts`.

It is published to **GitHub Packages** (not the public npm registry); `.npmrc` routes `@sudobility`
to `npm.pkg.github.com`.

Where it sits in the raidr family:

| Repo | Role | Relation |
| --- | --- | --- |
| `raidr_extension` | Chrome MV3 capture | Bundles this pkg; uses redaction, coverage, manifest, bundle assembly. |
| `raidr_cli` | Reconstruction CLI + `raidr-reconstruct` skill | Reads bundles, runs analysis/codegen; its capture harness writes bundles. |
| `raidr_crawler` | Headless crawl + `raidr-publish` skill | Writes bundles; uses redaction, `buildApiModel`, `toPathTemplate`. |
| `raidr_types`→`client`→`lib`→`app`, `raidr_api`, `raidr_web` | Catalog UI / API / landing | Do **not** depend on this package. |

Release order (in `raidr_app/scripts/push_all.sh`): raidr_types, then this repo, then the others.

## Commands

Use **Bun** for everything; never npm/yarn/pnpm. Run `bun install` first — without `node_modules`,
`tsc` is not on PATH and typecheck/build fail with `tsc: command not found`.

| Command | Does | Verified result |
| --- | --- | --- |
| `bun install` | Install deps (`fflate`; dev: `typescript`, `@types/bun`) | pass |
| `bun run typecheck` | `tsc --noEmit` | pass |
| `bun run test:unit` (= `bun test`) | all of `tests/` | 205 pass, 0 fail, 22 files |
| `bun run build` | `tsc` → `dist/` then `scripts/fix-esm-imports.mjs` | pass (4 dynamic specifiers warned, non-fatal) |

No `lint` script and no plain `test` script. `dist/` is gitignored and built by CI before publish.

## Architecture

`tests/<area>/<module>.test.ts` mirrors `src/<area>/<module>.ts`. Modules by area:

- `bundle/` — format: `types.ts` (CapturedRequest, Gap, RedactionEntry, RaidrManifest…), `paths.ts`
  (`content/<hash>.<ext>`, MIME→ext), `manifest.ts` (create/validateManifest, JSONL codec), `store.ts`
  (ContentStore, MemoryContentStore — hash fn injected), `assemble.ts` (buildBundleFiles layout,
  zipBundle, bundleFilename), `read.ts` (readBundle, unzipBundle).
- `redaction/` — `patterns.ts` (isSensitiveKey/KEY_KINDS, classifyValue), `pseudonym.ts`
  (createPseudonymizer: deterministic placeholders + redaction.json), `headers.ts`, `json.ts`,
  `index.ts` (redactRequest: strategy per MIME).
- `coverage/` — `pathTemplate.ts` (toPathTemplate, endpointKey), `coverage.ts` (computeCoverage).
- `analysis/` — `sourceMap.ts` (parse/recoverSources from sourcesContent only), `schema.ts`
  (infer/unifySchemas), `apiModel.ts` (buildApiModel), `routeModel.ts` (buildRouteModel),
  `navigations.ts` (deriveTimeline), `linkAudit.ts` (auditLinks).
- `audit/` — passive security audit (see below): `types.ts`, `text.ts`, `codeRules.ts`,
  `trafficRules.ts`, `index.ts`.
- `codegen/` — `types.ts` (schemaToType…), `client.ts` (generateTypes/generateClient), `replay.ts`
  (generateReplayServer — Hono), `project.ts` (generateProject — Vite React/Vue scaffold).

Bundle layout (fixed by `assemble.ts`): `raidr.json`, `gaps.json`, `redaction.json`,
`network/{requests,websockets}.jsonl`, `runtime/{framework,routes,stores,chunks,coverage,navigations}.json`,
`content/<hash>.<ext>`, `sourcemaps/{index.json,<hash>.map}`, `snapshots/{index.json,<hash>.html}`.

Data flow (every stage pure; capture and reconstruct both run **in the consumer**): capture →
`redactRequest` + ContentStore → `buildBundleFiles`/`zipBundle`/`bundleFilename`. Reconstruct →
`validateManifest`/`parseJsonl` → `recoverSources`, `buildApiModel`→codegen, `buildRouteModel` (fed by
`deriveTimeline` when no router navigations)→`generateProject`, `auditLinks`. `endpointKey` is the join
key across API model, route model, coverage and replay.

## The security audit (`src/audit/`)

`auditCandidates(input)` runs every rule, returns candidates sorted most-severe/most-certain first,
numbered `c1…`, unique by `fingerprint`, capped at `MAX_AUDIT_CANDIDATES` (150). Caller sends
`AUDIT_INSTRUCTIONS` (fixed, cacheable) + `auditPrompt(origin, batch)` to a model, then
`applyAuditAnswer(batch, parseAuditAnswer(answer))`. The model may only drop candidates or set
severity/confidence/text; rule, category, CWE, OWASP, evidence and fingerprint never come from it.
`candidatesToIssues` is the no-model path. `AuditIssue` matches raidr_types' `SecurityIssueInput`
field-for-field (no dependency on raidr_types).

- **Every snippet is masked.** `excerpt` runs `maskSecrets` (SECRET_VALUE_RES, Google `1//0…` refresh
  and `ya29.` access tokens, quoted values under secret-named keys via `KEYED_SECRET_RE`, URL
  passwords). Never put raw script/body text into evidence or context any other way.
- **Precision over recall.** Client-code sinks count only when the written value reads URL/referrer/
  window-name/`e.data`; message handlers only when inline on `window`; PII only when the value has the
  right shape and the file is not static content. Each was a real false positive; `tests/audit/` pins them.
- Fingerprints are stable across deploys (rule+host, or hash of secret, or `stableFileName` + snippet
  minus minified identifiers). No `atob` (not in ES2022 lib) — JWTs use `base64UrlDecode`.

## Invariants that are easy to break

- **No I/O and no DOM in `src/`.** Nothing may use `fs`, `path`, `process`, `Bun`, `document`, `window`.
  The extension bundles this into an MV3 service worker where Node APIs don't exist. The compiler only
  partly enforces it: `lib: ["ES2022"]` kills DOM globals, but `@types/bun`/`@types/node` are
  auto-included (no `types` field), so `import fs` would still typecheck — **review by hand.** The
  `process.env`/`hono/bun` strings in `codegen/replay.ts` are generated output, not runtime use.
- **Hashing is injected.** `MemoryContentStore` takes a `HashFn`; all three consumers pass SHA-256 hex.
- **Redaction deliberately keeps `api_key`/`apikey`/`x-api-key` visible** — a key the browser ships is
  public and redacting it breaks the rebuilt app (comment in `patterns.ts`; asserted in tests).
  Per-user tokens (`x-auth-token`, `x-csrf-token`) **are** redacted as kind `session`.
- **No shape-only secret detection.** `classifyValue` recognises only JWT, `Bearer …`, email; never
  UUIDs. Long random strings broke hashes/trace-ids/keys once. `high-entropy` kind exists but is never produced.
- **JS/CSS/images/fonts are never modified** (`isImmutableAsset`) — changing them corrupts parsing and
  source-map offsets.
- **`RAIDR_FORMAT_VERSION` is checked exactly** by `validateManifest`; bumping it makes every existing
  bundle unreadable. `manifest.ts` imports it from `../index` (a cycle that works only because it's read
  inside functions — never read it at module top level).
- **`generateReplayServer`'s text is a contract with raidr_cli**, which rewrites it with exact-string
  `.replace()` in `raidr_cli/src/commands/reconstruct.ts`: `const app = new Hono();`,
  `serveStatic({ root: './dist' })`, the `app.get('*', …)` fallback, and the two-line `pick` body of
  `respond`. Change any and the rewrite silently no-ops — no test in either repo catches it.
- **Replay route registration order matters**: literal routes → static files → param routes → 501 gap
  guards → SPA fallback (comments in generated source explain each).
- **Gaps fail loudly** — uncaptured data becomes a 501 `RAIDR-GAP` / page comment / `RAIDR-GAPS.md`,
  never invented. **No silent loss in the route model**: unclaimed endpoints go to `unattributed`.

## Testing & CI

- All pure `bun:test` units — no browser/network. `tests/fixtures/minimal/` is used only by
  `tests/bundle/manifest.test.ts`; `tests/smoke.test.ts` checks the format version.
- CI `.github/workflows/ci-cd.yml` calls `johnqh/workflows` `unified-cicd.yml` with
  `publish-target: github` (GitHub Packages). It runs install, typecheck, lint (skipped — no script),
  test:unit, build.
- CI **cannot** catch: a Node/Bun/DOM API creeping into `src/`; breaking changes to consumers (nothing
  here builds them against a local copy); whether generated projects install/compile (raidr_cli's
  round-trip tests do that).

## Versioning & publishing

- Dependents (raidr_extension, raidr_cli, raidr_crawler) pin `^0.1.1`. On 0.x a caret admits only patch
  releases, so a `0.2.0` needs each dependent's range bumped.
- Publishing is **CI-only**: on push to `main`, the workflow publishes to GitHub Packages if the
  `package.json` version isn't already on the registry. Pushes to `develop` only run tests. Never
  `npm publish` by hand and never bump the version unless asked — `push_all.sh` does bumps/pushes in
  dependency order. `files` ships `dist/`, `LICENSE.md`, `README.md`; no `prepublish` script.

## Common-change recipes

- **Bundle field/file** → `bundle/types.ts`; for a new file also `assemble.ts` (+`paths.ts` if
  content-addressed); reject old bundles via `validateManifest` (bump `RAIDR_FORMAT_VERSION` only for a
  breaking layout change); tests in `tests/bundle/`; update writers (consumers) and reader `read.ts`.
- **Redaction rule** → key-based in `KEY_KINDS` (`patterns.ts`, first match wins — place deliberately);
  value-based in `classifyValue`. New kind → add to `RedactionKind` (`bundle/types.ts`) and `LABELS`
  (`pseudonym.ts`, typecheck fails until you do). Add a negative test for anything that must stay visible.
- **Export** → change module, re-export from `src/index.ts`, update README exports table, add a test.
- **Codegen** → change `codegen/*.ts`, update `tests/codegen/`, check the `.replace()` strings in raidr_cli.

## Gotchas

- `redactRequest` does not touch the URL (query-string tokens pass through) and treats every body as
  JSON (form-encoded bodies left unredacted).
- `redactJsonText` re-serializes compactly, so a redacted body's bytes/hash differ from the original
  even when nothing changed; a non-JSON body comes back unchanged.
- Pseudonym digests are 16-bit salted FNV-1a; collisions share a placeholder. Masked emails
  (`ab***@x.com`) become `masked<N>@example.com`; real ones `user<N>@example.com` (AUDIT_INSTRUCTIONS
  tells the reviewer `user<N>@…` is a real leaked address). `unauthenticated-pii` ignores masked addresses.
- `buildApiModel` detects auth only from lowercase `authorization`/`cookie` header keys.
- `toPathTemplate` reuses placeholder names, so `/a/1/b/2` → `/a/{id}/b/{id}` — `generateClient` then
  emits two params named `id` (won't compile), and two 2xx statuses on one endpoint yield duplicate
  type declarations.
- `RouteModel.lazy` is true for every non-root route, and `generateProject` uses it as-is.
- `zipBundle` uses `zipSync` on purpose — fflate's async `zip` breaks under Bun workers.
- `docs/superpowers/` specs/plans are dated 2026-08-24 (first milestone); code has moved on — verify
  against `src/` before relying on them.
