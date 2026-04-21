**Overview**
This MVP builds a tiny TypeScript utility library that exports a single pure function, `deepEqual(a, b)`, for recursive structural equality across primitives, arrays, structural objects, `Date`, `RegExp`, `null`, and `undefined`. Structural objects include plain objects and class instances, compared by their own enumerable properties rather than constructor identity. The implementation stays within the constitution boundaries: no runtime dependencies, no extra public API, strict TypeScript, explicit rejection of `Map` and `Set` as unsupported MVP containers, cyclic references returning `false` instead of throwing, and a small but complete toolchain using `pnpm`, flat ESLint, Vitest, and a TypeScript ESM build via `tsc -p tsconfig.build.json` that emits `dist/index.js` and `dist/index.d.ts`.

**Architecture**
The library consists of one public entry point in `src/index.ts` plus a small set of internal comparison branches inside the same module or private helpers local to that module. Data flow is straightforward: callers pass two `unknown` values into `deepEqual`, the function first resolves fast-path primitive and identity checks, then rejects unsupported `Map` and `Set` inputs, detects repeated object-pair traversal and returns `false` for cycles, dispatches by runtime shape (`null`/`undefined`, array, `Date`, `RegExp`, structural object, mixed types), and finally recurses into nested array elements or object properties until it can return a single boolean result. Tooling files (`package.json`, `tsconfig.json`, `tsconfig.build.json`, `eslint.config.js`, `vitest.config.ts`) support typechecking, linting, testing, package metadata, and build output, while `src/index.test.ts` exercises both accepted and rejected comparisons so each explicit branch in the comparator is covered.

**User experience**
Public API:
- `export function deepEqual(a: unknown, b: unknown): boolean`

Public vs internal:
- Public: `deepEqual` is the only exported symbol and the only semver-stable API promised by the package.
- Internal: helper functions, type guards, and normalization utilities used to implement `deepEqual` remain unexported and can change without a major version bump.

Behavior and result shape:
- The function returns `true` when two supported values are structurally equal under the MVP rules and `false` otherwise.
- The function performs no I/O, no network access, no filesystem writes, and no global mutation; it is pure and deterministic.
- The function should not throw for ordinary caller input because the API accepts `unknown` and communicates comparison outcome through the boolean return value.
- Cyclic references are unsupported in the MVP and must return `false` once a repeated object pair is detected, so the implementation never infinite-loops and does not throw for cycles.
- `Map` and `Set` are unsupported in the MVP and must always return `false`, including `Map`-vs-`Map` and `Set`-vs-`Set`; they never fall through to structural object comparison.
- Plain objects and class instances compare structurally by own enumerable string-keyed properties without requiring matching constructors or prototypes, except for dedicated built-in branches such as `Date`, `RegExp`, `Map`, and `Set`.

Usage snippets:

```ts
import { deepEqual } from "<package-name>";

deepEqual(
  { id: 42, tags: ["core", "mvp"] },
  { id: 42, tags: ["core", "mvp"] },
); // true
```

```ts
import { deepEqual } from "<package-name>";

deepEqual(new Date("2024-01-01T00:00:00.000Z"), new Date("2024-01-01T00:00:00.000Z")); // true
deepEqual(/abc/gi, /abc/g); // false
```

```ts
import { deepEqual } from "<package-name>";

deepEqual(NaN, NaN); // true
deepEqual(+0, -0); // true
deepEqual([1, { ok: true }], [1, { ok: false }]); // false
```

Error handling:
- No validation errors are expected for normal caller input because both parameters are `unknown`.
- No result wrapper type is needed; the full result surface is a boolean.
- MVP out-of-scope behavior for cycles, `Map`, and `Set` must be documented in a short module-level comment in `src/index.ts` and reinforced by explicit tests in `src/index.test.ts`.

**File tree**
Files to create or modify for the MVP implementation:

```text
.
|-- package.json
|-- pnpm-lock.yaml
|-- tsconfig.json
|-- tsconfig.build.json
|-- eslint.config.js
|-- vitest.config.ts
`-- src
    |-- index.ts
    `-- index.test.ts
```

Files involved in planning and review:

```text
.
|-- CONSTITUTION.md
`-- PLAN.md
```

**Dependencies**
Runtime dependencies:
- None

Development dependencies:
- `typescript` for strict typechecking and declaration/build output
- `eslint` for linting
- `@eslint/js` if needed to compose the flat ESLint baseline cleanly
- `typescript-eslint` for TypeScript-aware flat ESLint configuration
- `vitest` for tests

Package manager and commands:
- `pnpm install --frozen-lockfile`
- `pnpm eslint .` with zero warnings in the reported output
- `pnpm tsc --noEmit`
- `pnpm vitest run`
- `pnpm build`, implemented as `tsc -p tsconfig.build.json`

Packaging and runtime contract:
- `package.json` should declare an ESM package surface with `type: "module"`, `exports`, and `types` entries aligned to `dist/index.js` and `dist/index.d.ts`.
- `package.json` should declare `engines.node: ">=22"` to match the constitution's Node.js target.
- `tsconfig.build.json` should emit an ESM build targeted at Node.js >=22 and modern browsers without introducing extra runtime transforms or bundling.

**Data model**
Primary function types:
- `deepEqual(a: unknown, b: unknown): boolean`

Runtime value categories the implementation must distinguish:
- Primitive values: `string`, `number`, `boolean`, `bigint`, `symbol`, `undefined`, `null`
- Special numeric cases: `NaN`, `+0`, `-0`
- Indexed collections: arrays of nested `unknown` values
- Structural objects: plain objects and class instances compared by own enumerable string-keyed properties, excluding values handled by dedicated built-in or unsupported-container branches
- Built-ins with custom comparison rules: `Date`, `RegExp`
- Explicitly unsupported containers: `Map`, `Set`

Comparison rules by shape:
- Primitives compare by value, except `NaN` equals `NaN` and signed zero values are treated as equal.
- Arrays compare by kind, then length, then ordered element-wise recursive equality.
- Structural objects compare by own enumerable string-key count and key set, then recursive equality for each matching property value; constructor and prototype identity are ignored, so a class instance and a plain object with the same own enumerable properties compare equal unless a dedicated built-in or unsupported-container rule applies first.
- `Date` compares by timestamp value; two invalid dates whose timestamps are both `NaN` compare as equal, while an invalid date and a valid date compare as `false`.
- `RegExp` compares by `source` and `flags`.
- `Map` and `Set` always compare as `false` in the MVP, even against the same container type, and are rejected before structural-object fallback.
- Cyclic references always compare as `false` in the MVP once a repeated object pair is encountered during recursion.
- Mixed remaining categories compare as `false`.

Non-goals reflected in the model:
- No element-wise or entry-wise equality semantics for `Map` or `Set` in the MVP; the only defined behavior is explicit rejection as `false`.
- No class-instance identity checks; object comparison is purely structural and based on own enumerable properties.
- No cycle-aware graph equivalence in the MVP; cycle detection exists only to short-circuit to `false` and avoid infinite recursion.

**Implementation phases**
1. Create project scaffolding with `package.json`, `pnpm` scripts, `tsconfig.json` in strict authoring mode, `tsconfig.build.json` for declaration-emitting ESM builds to `dist`, flat `eslint.config.js`, and `vitest.config.ts` aligned to the constitution commands and the Node.js >=22 target.
2. Implement the single exported `deepEqual` function in `src/index.ts` with explicit branches for primitive equality, `NaN`, signed zero normalization, `null`/`undefined`, arrays, `Date` (including invalid-date equality), `RegExp`, `Map`/`Set` rejection, structural object comparison for plain objects and class instances, cycle detection that returns `false`, and mixed-type rejection.
3. Keep comparison internals private and minimal, extracting only the smallest helper functions needed to make each branch named and testable.
4. Add Vitest coverage in `src/index.test.ts` with at least 12 cases spanning positive and negative paths, including nested arrays/objects, class-instance structural equality, `bigint`, `symbol`, `NaN`, signed zero, valid and invalid `Date`, `RegExp`, `Map`/`Set` rejection, cycles returning `false`, and mixed types.
5. Run `pnpm eslint .`, `pnpm tsc --noEmit`, `pnpm vitest run`, and `pnpm build`, then fix remaining configuration or branch gaps until all quality standards pass with zero ESLint warnings.
6. Confirm the built package emits `dist/index.js` and `dist/index.d.ts`, ensure the final implementation still exposes only `deepEqual`, and verify that `src/index.ts` contains the short module-level note documenting the cycle and `Map`/`Set` rejection policy.

**Acceptance criteria**
- `src/index.ts` exports exactly one public function: `deepEqual(a: unknown, b: unknown): boolean`.
- The implementation returns correct boolean results for all MVP-supported categories: primitives (including `bigint` and `symbol` cases), `NaN`, signed zero, arrays, structural objects/class instances by own enumerable properties, `Date`, `RegExp`, `null`, and `undefined`.
- `Date` comparison treats two invalid dates as equal and a valid/invalid pair as unequal.
- Class instances are compared without constructor or prototype identity checks; matching own enumerable properties determine equality unless a dedicated special-case branch applies.
- `Map` and `Set` comparisons always return `false` and never fall through to structural object comparison.
- Cyclic references return `false` rather than throw and never cause infinite recursion.
- No runtime dependencies are introduced.
- Tooling exists and uses the constitution-defined stack: `pnpm`, strict `tsconfig.json`, `tsconfig.build.json` for emit, flat ESLint, and Vitest.
- `src/index.test.ts` contains at least 12 Vitest cases covering both accepted and rejected paths, including `bigint`, `symbol`, invalid `Date`, class instances, `Map`/`Set` rejection, and cycles returning `false`.
- `pnpm eslint .` reports zero warnings; `pnpm tsc --noEmit`, `pnpm vitest run`, and `pnpm build` all pass.
- `pnpm build` runs `tsc -p tsconfig.build.json` and emits ESM `dist/index.js` plus `dist/index.d.ts`.
- `package.json` declares `type: "module"`, `exports`, `types`, and `engines.node: ">=22"` so the emitted package matches the Node.js >=22 and modern-browser target.
- `src/index.ts` contains a short module-level note documenting that cycles, `Map`, and `Set` are out of MVP scope and therefore return `false`, with tests asserting the same behavior.

**Open questions**
- What package name should be used in `package.json` and examples? The constitution defines behavior and tooling, but not the publish/install name.

## Revision notes for round 1
- Addressed the cycle-policy comments by fixing the MVP behavior to return `false` for cyclic references, adding explicit cycle detection to the architecture/data-model/implementation sections, and removing the old open question about throwing vs returning `false`.
- Addressed the `Map`/`Set` comments by making rejection unconditional: all comparisons involving `Map` or `Set` now return `false` and never fall through to structural object comparison.
- Addressed the class-instance comments by defining structural-object comparison as own-enumerable-property comparison that ignores constructor and prototype identity, so class instances and plain objects share the same branch unless a dedicated built-in rule applies.
- Addressed the build and runtime-contract comments by adding `tsconfig.build.json`, specifying `pnpm build` as `tsc -p tsconfig.build.json`, and defining the ESM package metadata plus `engines.node: ">=22"` expectations in `package.json`.
- Addressed the acceptance-bar comments by requiring zero-warning ESLint output and by naming the MVP documentation surface explicitly as a short module-level note in `src/index.ts`, backed by tests.
- Addressed the coverage comments by expanding the required test plan to include `bigint`, `symbol`, invalid `Date`, class-instance comparisons, `Map`/`Set` rejection, and cycles returning `false`.
