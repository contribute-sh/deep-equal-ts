# Project Constitution

Version: 1.0.0
Date: 2026-04-21

## Purpose

Build a tiny, well-tested TypeScript utility library exposing a single pure function: `deepEqual(a: unknown, b: unknown): boolean` that performs recursive structural equality.

Targets Node.js (>=22) and modern browsers. Zero runtime dependencies. Deterministic: pure data input → pure boolean output.

## Principles

- **One job, done correctly** — the library has exactly one public function. Don't grow the surface.
- **Zero runtime dependencies** — pure TypeScript, standard library only.
- **Explicit over clever** — every branch of the comparison is named and testable; no magic.
- **Test every accept/reject path** — Vitest covers primitives, arrays, objects, nested structures, null/undefined, NaN, Date, and RegExp.

## Stack

- language: typescript
- package_manager: pnpm
- install: pnpm install --frozen-lockfile
- test: pnpm vitest run
- lint: pnpm eslint .
- typecheck: pnpm tsc --noEmit
- build: pnpm build

## Boundaries

- Will NOT add runtime dependencies.
- Will NOT handle cyclic references in the MVP (returns `false` or throws — document the choice; do not infinite-loop).
- Will NOT compare class instances by class identity — only by structural equality of their own enumerable properties.
- Will NOT support Set or Map in the MVP (explicit out-of-scope; handle Hardening later).

## Quality Standards

- `pnpm tsc --noEmit` passes with `strict: true`.
- `pnpm vitest run` passes with >= 12 test cases.
- `pnpm eslint .` passes with zero warnings.
- `pnpm build` emits `dist/index.js` + `dist/index.d.ts`.

## Roadmap

### MVP

- Implement `deepEqual(a, b)` in `src/index.ts` handling: primitives (including NaN-NaN → true, ±0 → true), arrays, plain objects, Date (same time), RegExp (same source+flags), null, undefined, mixed types → false.
- Wire up package.json, tsconfig.json (strict), eslint.config.js (flat), vitest.config.ts.
- Provide >= 12 Vitest cases in `src/index.test.ts` covering valid + invalid paths.

### Hardening

- Add Set and Map support (size-match + element-wise equality).
- Add JSDoc to the exported symbol.

### Polish

- Add a README with the full accepted/rejected grammar.
- Add examples/basic.ts showing typical usage.

## Verification

- type: library
