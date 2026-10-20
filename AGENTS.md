# Teslatlas TypeScript client

> **Frozen on 2026-10-08.** This checkout was imported with its full history into the
> Hub repository as `hub/sdk-typescript/`, which is now the authoritative copy. Do not edit
> here; it stays only until the App re-pins Compute and the old GitHub repositories
> are archived on the owner's push request.

Node.js client for the Hub read API. Follow `../AGENTS.md` and
`../docs/development/MASTER_PLAN.md`; the plan wins on process, model, scope and commit
policy. Only the Hub client (`src/hub`) is retained; the rich client, SSE, vehicle
commands, the vendored Protocol copy, the lock and sync scripts and the two
bearer-serving browser helpers are deleted under plan group G2-2. Types are then
generated from the in-repo contract. No browser client in v7. The package is not
published in v7. The retained code moves into the `magrathean-uk/teslatlas-hub`
workspace as the `sdk-typescript/` package, keeping its Apache-2.0 licence, after an
Authorisations row.

## Boundaries

- Derive public types from the canonical contract; use `camelCase` TypeScript names and
  lowercase-hyphenated documentation names.
- Keep credential and checkpoint persistence caller-owned. Preserve typed errors,
  ETags and cursor binding.
- Keep the pairing boundary intact: the Node claim transport verifies normal TLS trust,
  hostname and the invitation leaf pin on one connection.
- Do not turn fixture, unit or package checks into a claim about a live Hub.
- Legal files (`LICENSE`, `NOTICE`, `docs/legal/`, contributor terms, copyright and
  attribution strings) are owner-controlled: change them only on the owner's explicit
  instruction.
- No vehicle commands. Short local commit per reviewed batch; push only on an owner
  request. No CI, release automation, tags or publication.

## Working locally

Use the commands in `package.json` and run the smallest relevant check: `npm run build`
then `npm run typecheck`, `npm run test:unit`, `npm run test:conformance`, or
`npm run verify` for the full local suite. `npm run build` writes `dist/` in this
checkout. Run everything through `../scripts/dev/run.sh teslatlas-sdk-typescript ...`.

<!-- clean-development-policy:v4 (clean-development 0.6.0; canonical: bolyki-dev/clean-development/docs/policy-snippet.md) -->
## Clean development (mandatory)

- Storage is routed for you: every zsh loads `~/dev/machine/env.zsh`; hooks pin the mode per session (`$CLEAN_DEV_MODE` is `external` when the 4TB is attached, else `internal`). Do not wrap commands, override or unset the routed variables.
- Temp files go to `$TMPDIR` (session scratch, deleted at session end), throwaway build output to `$CLEAN_DEV_BUILDS`, caches wherever their variables point. Never write caches, builds or temp files into `~`, `~/dev` or this repo outside its git-ignored build directories.
- Internal mode: before any download or build likely over 5 GB, stop and ask G (suggest waiting for the 4TB).
- This repo's `.clean-development.json` (schema v2) lists its git-ignored `buildDirs`/`depDirs`; keep it accurate when the build layout changes.
- No dot-folders in `~` or `~/dev`. `.env` files stay git-ignored; read them only when the task needs them; never commit them.
