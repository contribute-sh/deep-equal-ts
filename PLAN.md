**Overview**
This MVP builds a tiny TypeScript utility library that exports a single pure function, `deepEqual(a, b)`, for recursive structural equality across primitives, arrays, plain objects, `Date`, `RegExp`, `null`, and `undefined`, with explicit handling for `NaN` equality and `+0`/`-0` equality. The implementation stays within the constitution boundaries: no runtime dependencies, no extra public API, strict TypeScript, and a small but complete toolchain using `pnpm`, flat ESLint, Vitest, and a build that emits `dist/index.js` and `dist/index.d.ts`.

**Architecture**
The library consists of one public entry point in `src/index.ts` plus a small set of internal comparison branches inside the same module or private helpers local to that module. Data flow is straightforward: callers pass two `unknown` values into `deepEqual`, the function first resolves fast-path primitive and identity checks, then dispatches by runtime shape (`null`/`undefined`, array, `Date`, `RegExp`, plain object, mixed types), and finally recurses into nested array elements or object properties until it can return a single boolean result. Tooling files (`package.json`, `tsconfig.json`, `eslint.config.js`, `vitest.config.ts`) support typechecking, linting, testing, and build output, while `src/index.test.ts` exercises both accepted and rejected comparisons so each explicit branch in the comparator is covered.

**User experience**
Public API:
- `export function deepEqual(a: unknown, b: unknown): boolean`

Public vs internal:
- Public: `deepEqual` is the only exported symbol and the only semver-stable API promised by the package.
- Internal: helper functions, type guards, and normalization utilities used to implement `deepEqual` remain unexported and can change without a major version bump.

Behavior and result shape:
- The function returns `true` when two supported values are structurally equal under the MVP rules and `false` otherwise.
- The function performs no I/O, no network access, no filesystem writes, and no global mutation; it is pure and deterministic.
- The function should not throw for ordinary supported inputs because the API accepts `unknown` and communicates comparison outcome through the boolean return value.
- Unsupported MVP shapes such as `Set` and `Map` should resolve to `false` when compared against supported shapes or non-equivalent values unless implementation details require a stricter branch; if a stricter behavior is chosen for cyclic references, it must be documented and tested explicitly.

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
- Cyclic reference behavior is the only constitution-level ambiguity and must be documented once implementation confirms whether the MVP returns `false` or throws for cycles.

**File tree**
Files to create or modify for the MVP implementation:

```text
.
|-- package.json
|-- pnpm-lock.yaml
|-- tsconfig.json
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
- `pnpm eslint .`
- `pnpm tsc --noEmit`
- `pnpm vitest run`
- `pnpm build`

**Data model**
Primary function types:
- `deepEqual(a: unknown, b: unknown): boolean`

Runtime value categories the implementation must distinguish:
- Primitive values: `string`, `number`, `boolean`, `bigint`, `symbol`, `undefined`, `null`
- Special numeric cases: `NaN`, `+0`, `-0`
- Indexed collections: arrays of nested `unknown` values
- Structured objects: plain objects with own enumerable string keys and nested `unknown` property values
- Built-ins with custom comparison rules: `Date`, `RegExp`

Comparison rules by shape:
- Primitives compare by value, except `NaN` equals `NaN` and signed zero values are treated as equal.
- Arrays compare by kind, then length, then ordered element-wise recursive equality.
- Plain objects compare by kind, then own enumerable key count and key set, then recursive equality for each matching property value.
- `Date` compares by timestamp value from its internal time representation.
- `RegExp` compares by `source` and `flags`.
- Mixed categories compare as `false`.

Non-goals reflected in the model:
- No special support for `Map` or `Set` in MVP.
- No class-instance identity checks beyond comparison of own enumerable properties, per constitution boundary.
- No cycle-aware graph model in MVP; cycle handling remains a documented edge policy rather than a supported structure.

**Implementation phases**
1. Create project scaffolding with `package.json`, `pnpm` scripts, `tsconfig.json` in strict mode, flat `eslint.config.js`, and `vitest.config.ts` aligned to the constitution commands.
2. Implement the single exported `deepEqual` function in `src/index.ts` with explicit branches for primitive equality, `NaN`, signed zero normalization, `null`/`undefined`, arrays, plain objects, `Date`, `RegExp`, and mixed-type rejection.
3. Keep comparison internals private and minimal, extracting only the smallest helper functions needed to make each branch named and testable.
4. Add Vitest coverage in `src/index.test.ts` with at least 12 cases spanning positive and negative paths, including nested arrays/objects, mixed types, `NaN`, signed zero, `Date`, `RegExp`, and out-of-scope values.
5. Run `pnpm eslint .`, `pnpm tsc --noEmit`, `pnpm vitest run`, and `pnpm build`, then fix remaining configuration or branch gaps until all quality standards pass.
6. Confirm the built package emits `dist/index.js` and `dist/index.d.ts`, and ensure the final implementation still exposes only `deepEqual`.

**Acceptance criteria**
- `src/index.ts` exports exactly one public function: `deepEqual(a: unknown, b: unknown): boolean`.
- The implementation returns correct boolean results for all MVP-supported categories: primitives, `NaN`, signed zero, arrays, plain objects, `Date`, `RegExp`, `null`, and `undefined`.
- Mixed-type comparisons return `false`.
- No runtime dependencies are introduced.
- Tooling exists and uses the constitution-defined stack: `pnpm`, strict `tsconfig`, flat ESLint, and Vitest.
- `src/index.test.ts` contains at least 12 Vitest cases covering both accepted and rejected paths.
- `pnpm eslint .`, `pnpm tsc --noEmit`, `pnpm vitest run`, and `pnpm build` all pass.
- `pnpm build` emits `dist/index.js` and `dist/index.d.ts`.
- Unsupported MVP areas (`Map`, `Set`, and cycle handling policy) are explicitly documented rather than silently left undefined.

**Open questions**
- For cyclic references, should the MVP standardize on returning `false` or throwing an error? `CONSTITUTION.md` allows either, but the final implementation and tests need one documented choice.
- Should comparisons involving unsupported container types such as `Map` and `Set` always return `false`, or should they fall through to structural own-property comparison if they have enumerable properties? The constitution says they are out of scope, but the exact rejection behavior should be made explicit in implementation notes/tests.
- What package name should be used in `package.json` and examples? The constitution defines behavior and tooling, but not the publish/install name.
