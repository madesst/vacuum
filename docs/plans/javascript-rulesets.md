# JavaScript rulesets for vacuum

Status: proposed implementation plan. This PR changes no runtime behavior.

Let an agent bring an executable ruleset to vacuum, run it through the existing
engine, and prove what it did. Keep the rules. Keep the closures. Do not make the
agent rewrite a policy merely because its ruleset exports JavaScript.

## Outcome

`vacuum lint api.yaml -r ruleset.js` accepts supported CommonJS and ESM rulesets,
including imported custom functions. The same prepared ruleset works through
reports, the Go API, and the agent test harness. Existing YAML/JSON rulesets and
`getSchema()` / `runRule(input)` functions keep their behavior and fast path.

This is ruleset and function compatibility, not an implementation of Node.js or
an in-process replacement for `@stoplight/spectral-core`.

Implementation and companion documentation are separate linked PRs. The plan PR
does not publish documentation that claims this feature already exists.

## Evidence and existing owners

Reviewed against vacuum main `424f9daf`, Spectral CLI 6.17.0's installed package
contracts, and the agent-harness branch `ed8bf3a9` (PR #999, not on this base).

- `rulesets/remote_ruleset.go`: local/remote loading, source locations and extends.
- `rulesets/rulesets.go`: validation, rule merging, load errors and built-ins.
- `cmd/shared_functions.go`: location/config handling and function loading.
- `plugin/javascript/js_plugin.go`: Goja, CommonJS registry, custom-function bridge.
- `plugin/javascript/eventloop/`: async work and cancellation.
- `plugin/javascript/`: fetch restrictions and runtime error tests.
- `model.RuleFunction`, `model.RuleFunctionContext` and `motor.ExecutionOptions`:
  existing function execution contracts. Do not create another lint engine.
- `cmd/lint_shared.go`, `cmd/build_results.go`: shared execution paths.
- `harness/` and `cmd/test*.go` on #999: exact assertions, typed errors and manifests.
- Spectral `packages/core/src/types/function.ts`, `ruleset/types.ts`,
  `documentInventory.ts`: callable rules, options, context and format contracts.

The current loader parses data, not executable exports. The current JS bridge
has a different function signature. CommonJS support in Goja does not provide an
ESM loader or a complete Node environment. These are the gaps to close.

## Supported contract

| Surface | Initial implementation |
| --- | --- |
| Entry files | Explicit local `.js`, `.cjs`, `.mjs` passed through existing ruleset options. |
| Exports | `module.exports = ruleset` or ESM `export default ruleset`; export must be a ruleset object, not a factory or Promise. |
| Imports | Static local JS/JSON imports and compatible, already-installed npm packages; no package install or network module fetch. |
| Functions | Direct callable `then.function`, closures, `(input, options, context)`, sync results and Promises. |
| Results | `undefined` or an array of `{message, path?}`; malformed results fail execution. Paths retain numeric indices and missing-property locations. |
| Context | `path`, `document.data`, `document.source`, and an explicitly tested read-only rule metadata subset. Document data reflects Spectral's contract, not whichever tree is convenient. |
| Formats | Callable predicates, including predicates used by scoped aliases; evaluate under the same timeout/error rules. |
| Inheritance | Exported parent objects, ordered extends tuples, severity overrides, aliases and action arrays. Reuse the existing merge owner after normalization. |
| TypeScript | Deferred; reject `.ts` with a clear instruction to compile to supported JavaScript. |
| Remote executable code | Rejected, including a remote YAML ruleset extending a local executable ruleset. Local explicitly selected executable graphs are the trust boundary. Existing remote data rulesets retain their behavior. |
| Node-specific behavior | Unsupported built-ins/native addons and unsupported package exports fail with the module and importer identified. No general Node shim layer. |
| Dynamic loading | Nonliteral/dynamic require/import and top-level await are outside the initial contract; reject rather than leave an uncontrolled runtime resolver. |

An installed npm package is not automatically compatible. Pure JavaScript is
necessary but not sufficient: it must use the supported module and runtime APIs.

`documentInventory`, resolver graphs, document methods, full `Spectral` instances,
parser-option parity and file-override parity are not initial compatibility claims.
Detect unsupported exported fields at load time. Expose unsupported context APIs
through throwing accessors/proxies, not empty objects that can manufacture a pass.
For supported context fields, pin fixtures for resolved and unresolved rules,
external references, missing fields, array indices and root selections.

### Spectral package imports

Add only named compatibility exports backed by fixtures:

- `@stoplight/spectral-functions`: the documented core functions that vacuum
  already implements. Dispatch directly to existing functions where their
  contracts match. Do not create independent copies of built-in validation logic.
- `@stoplight/spectral-core`: `createRulesetFunction` only, including its input and
  options validation behavior. Unsupported exports, including `Spectral`, fail.
- `@stoplight/spectral-formats`: explicitly tested built-in format predicates.
- `@stoplight/spectral-rulesets`: defer `oas`/`asyncapi` object compatibility until
  the pinned corpus proves it. Do not substitute vacuum's recommended ruleset:
  our default checks differ. Existing string `extends` behavior remains unchanged.

The function-export list must be checked into fixtures and documentation, not
inferred from a broad claim of Spectral compatibility. A fixture that needs an
unsupported helper gets a clear migration error, not an approximate replacement.

## Implementation design

### 1. One module preparation path

Use the Go API of esbuild to prepare supported module graphs into CommonJS for
Goja. It earns its dependency by handling ESM syntax and static dependency graphs;
do not write a second JavaScript parser or resolver. Do not run a Node subprocess.
Pin its version when implementing, and verify its package-resolution behavior
with `.js`/`.cjs`/`.mjs`, package `type`, exports conditions and JSON fixtures.

Use one controlled resolver policy for local files, installed packages and the
explicit compatibility modules. Preserve module identities and source mappings
in errors. Reject unsupported imports before evaluation. Audit transformed output
for unresolved dynamic module calls; no runtime escape into Goja's default
filesystem resolver. Resolve relative paths from the importing module, never cwd.

Record files actually read and their hashes, including package metadata used in
resolution. Keep this record in the prepared ruleset; no global dependency cache.
Compilation is per loaded source graph, never per selected document node.
Replace any remaining runtime require/import fallback with an explicit unsupported
operation error, including nonliteral calls that static analysis cannot classify.

### 2. Normalize once, preserve executable values

Add a location-aware preparation boundary in the existing ruleset-loading flow.
The prepared result contains the normal rule model, function adapters, module
provenance, and execution-lifetime creation/cleanup. Keep the data-only API valid;
a byte-only parser must not guess a module filename or working directory.

Walk the exported object while inside its VM. Extract callable values before
validating declarative fields. Never JSON-serialize functions or use
`Function.toString()` to recreate them: either loses closures and module state.
Module inspection and execution-scope initialization must produce the same rule
and callable topology. Verify that identity when a fresh scope is initialized;
fail if initialization changes the rule contract. Top-level module initialization
may run during preparation and once per document; document that it must be free of
external side effects. Do not claim arbitrary stateful Spectral module parity.

Normalize callable references to internal collision-free handles, scoped to the
prepared result; preserve user rule IDs in all diagnostics and evidence. Reject
ambiguous collisions with explicit `--functions` registrations.

Reuse existing merge and validation semantics for the normalized data. Support
object inheritance by converting it into that path, with cycle detection and
stable parent order. Unsupported fields and invalid function options fail before
any rule executes. A failed import cannot yield a partial accepted ruleset.

### 3. Own VMs per document execution

The prepared graph may share immutable compiled programs. Goja values and module
instances stay inside a document execution scope. Evaluate the module graph once
for that scope, retain its closures, and dispose of it at completion. Do not retain
live VMs across documents, harness cases, baseline/current comparisons or repairs.

Serialize JS access within a scope using the existing runtime/event-loop ownership
pattern. Native rules retain existing concurrency. Start the execution deadline
before waiting for JS access, so queue time cannot escape the caller's timeout.
Cancellation must stop module initialization, getters during normalization, format
predicates, function calls and Promise work. Reuse existing interrupt cleanup.

Mutable module state is shared inside one document scope, as normal modules imply;
cross-rule invocation order is not promised. Document this limitation and require
order-independent policy checks. Fixtures prove closure preservation and isolation
between documents. Do not add a VM pool, a worker service or global mutable state.

Runtime binding must be passed through the existing execution owner, not hidden in
process-global registries. The first implementation spike must prove creation,
cleanup, cancellation and race safety before exporting a public Go API shape.

### 4. Adapt functions at the existing motor boundary

Implement `model.RuleFunction` adapters. Preserve function options, targeted input,
format filtering, resolved/unresolved selection, ignores and reference-origin
mapping. Keep Spectral result path arrays until the canonical path conversion;
do not flatten them with dotted string concatenation.

Validate the exact meaning of `then.field`, omitted return paths, missing values,
null, root paths and referenced nodes against Spectral fixtures. Do not infer
these semantics from a visually similar report. Add options/input validation for
`createRulesetFunction` using the existing schema validator after checking its
behavior against Spectral's helper.

Reuse the existing Promise/fetch implementation and its HTTPS/private-network
controls. Console output must go to stderr or the existing log sink, never corrupt
machine JSON. Script exceptions, unsupported context access, timeouts and malformed
results are tool errors (exit 2), not clean results or ordinary policy findings.

### 5. Carry the prepared result through every caller

Audit the callers of ruleset and custom-function loading. CLI lint, multifile lint,
JSON Schema, reports, dashboard, language server and docs diagnostics must either
support the prepared result correctly or return an explicit unsupported error;
none may accidentally discard its callable adapters. Preserve file/config path
resolution and current public data-only Go APIs.

Integrate with the harness after #999 is merged or on an explicitly stacked branch.
Use its typed error model and execution evidence rather than inventing a parallel
status/error schema. JS ruleset invocations must count as actual calls; an error
cannot pass a zero-diagnostic assertion.

Track the entry file, imported files and resolution metadata in the manifest.
Keep executable rulesets non-reproducible and reject portable bundle requests in
the initial implementation, consistently with existing custom-function limits.
A recorded import graph does not prove absence of external runtime state.

## Tests that prove the feature

Create a small checked-in compatibility corpus, pinned to Spectral CLI 6.17.0 and
its lockfile for the differential runner. Spectral is a development oracle, not a
runtime dependency of vacuum. Use synthetic policies; add at least one licensed,
pinned real ruleset example if redistribution is permitted. Record unsupported
cases honestly. Do not claim ecosystem compatibility from synthetic cases alone.

The positive anchor is a module importing a local closure-based function that
requires a nonblank operationId. Test valid, missing, empty and whitespace values,
POST exclusion, a broken selector, deliberate logic mutation and a repaired spec.
Cover CommonJS, ESM, mixed imports, installed pure-JS helper packages, inherited
objects, options, two actions, async results and callable formats/scoped aliases.

Negative fixtures cover invalid exports, missing imports, unsupported package/API
access, dynamic loading, inheritance cycles, malformed results, module/function
exceptions, infinite loops and cancellation while queued. Explicitly assert exit 2
and absence of a passing harness case. Remote-to-local executable inheritance must
fail before a marker side effect executes.

Compare rule IDs, severities, messages, duplicate counts, canonical paths and source
locations against the pinned oracle for the supported subset. Explicitly document
any intentional difference; never drop fields merely to make a fixture pass.
Run cases in repeated and concurrent document executions to detect leaked state.

For implementation validation, start with affected `rulesets`, `plugin/javascript`,
`motor` and `cmd` tests; add `harness` tests when available. Run race tests across
runtime ownership and cancellation. Then run release-shaped `GOWORK=off go test ./...` and tagged build checks from AGENTS.md. These are future checks, not evidence
that this planning PR implements the feature.

## Performance acceptance

Before edits, capture data-only CLI baselines on the six speed-test documents with
fixture hashes, compiler settings and module versions. Repeat after implementation:
one warmup, at least five shuffled serial runs, wall time and peak RSS, same output.
Reject timeout/partial-work results. Preserve full diagnostic comparisons.

Also compare equivalent YAML and JS rulesets on the same documents, separating
module initialization from rule execution. Prove module compilation/evaluation
counts do not grow with matched-node count. Profile material regressions in the
unchanged data-only path; no JS compilation or VM creation belongs on that path.
Report measurements and variability rather than promise a percentage improvement.
Do not trade correctness for a benchmark, enable turbo, or change timeouts to make
results look better. Repeated concurrency and memory tests must show no retained
per-document VM growth after scopes end.

## Documentation in ../quobix.com

Create a companion documentation branch/worktree and linked PR. Read that repo's
AGENTS.md before edits. Reuse current front matter (`type: vacuum`), menu hierarchy,
terminal-window, info-box/warn-box, relref links, tables and existing layouts. No
CSS, components, navigation redesign or new visual builder.

- Extend `content/vacuum/rulesets/custom-rulesets.md` with executable exports and
  the existing `-r` entry point; link the migration example.
- Extend `content/vacuum/rulesets/sharing.md` with installed package imports and
  precise package/module limitations.
- Extend `content/vacuum/api/custom-javascript-functions.md` to distinguish existing
  vacuum functions from Spectral-style functions. Keep current examples working.
- Extend `content/vacuum/api/loading-ruleset.md` with the final public Go entry
  point, execution scope and cleanup contract, after those are implemented.
- Add one focused page under `content/vacuum/rulesets/` only if the compatibility
  matrix and migration guide do not fit the existing page. Match sibling metadata.
- Update the agent-testing suites/results/API pages introduced by docs PR #18 with
  JS ruleset tests, typed failure examples and executable bundle restrictions.
- Put complete downloadable fixtures under the existing
  `static/vacuum/examples/` convention. Exercise downloads with the built binary
  and harness; include every referenced helper, suite and input file.

Use the user's established direct language: show the command, show the rule,
show the failure, then explain it. Say exactly what works and what needs a port.
Avoid claims of complete Spectral/Node compatibility or unreleased version numbers.
Build Hugo, check links and menu placement, inspect the affected rendered pages,
and verify examples as part of the companion PR. Keep unsupported examples clearly
marked, with asserted errors rather than commands presented as successful.

## Commit and delivery sequence

1. Pin the compatibility corpus and prove the module/runtime spike. Resolve the
   lifecycle and context mapping against real tests before broad caller changes.
2. Add module preparation and normalization with load/error tests.
3. Add callable adapters, context and formats, with differential/race tests.
4. Wire shared CLI and Go callers; prove every affected output surface and failure.
5. Add harness evidence/provenance integration after its prerequisite lands.
6. Add the companion documentation and runnable examples; run performance checks.
7. Apply de-slop and independent dave-review to implementation and docs. Commit
   each tested chunk and open linked implementation/docs PRs; do not merge them.

Completion means a fresh agent can load the supported module forms, diagnose a
broken selector and broken function, verify a repair, and encounter an explicit
error for unsupported APIs using only the docs. Tests, measured performance,
correctness comparisons and rendered docs accompany the code. No runtime feature
is considered delivered by merging this plan alone.
