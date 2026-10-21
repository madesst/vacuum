# JavaScript rulesets: plan-crush review

Scope: the design proposed in the conversation, checked against vacuum main
`424f9daf` and the Spectral 6.17.0 package contracts. Findings below are applied in
[javascript-rulesets.md](javascript-rulesets.md). This is a design review, not a
runtime test or implementation approval.

## [P1] Treating Spectral package names as aliases can change the policy

**Plan section:** Spectral package imports.

**Evidence:** `rulesets/remote_ruleset.go` and `rulesets/rulesets.go` select vacuum's
recommended rules for the Spectral string alias. The default CLI comparison also
returned different findings. Spectral's executable rulesets export actual objects
and functions, not only a string alias.

**Problem:** substituting vacuum's broader recommended set for an imported `oas`
object would silently change what a migrated executable ruleset checks.

**Why it does not earn its complexity:** a package-wide emulation claim adds a
second compatibility contract before its behavior has been demonstrated.

**Smaller design:** explicit, fixture-backed named exports. Defer imported
`@stoplight/spectral-rulesets` objects until differential coverage exists. Preserve
the existing string-alias behavior, without presenting it as exact object parity.

**Outcomes preserved:** executable local policies and supported functions migrate;
unsupported package exports fail clearly instead of changing policy.

**Verify:** imported unsupported exports fail before execution; supported exports
match pinned expected diagnostics. A documentation matrix names the boundary.

Confidence: high. Real-package coverage is still required before broader claims.

## [P1] Returning incomplete context can manufacture a passing rule

**Plan section:** Supported contract and function adaptation.

**Evidence:** Spectral `types/function.ts` exposes `documentInventory`;
`documentInventory.ts` includes a resolver graph and referenced-document lookup.
Vacuum's JS bridge supplies a different context in `plugin/javascript/js_plugin.go`.

**Problem:** an empty inventory object or a guessed document view can cause a rule
to check nothing and return no findings.

**Why it does not earn its complexity:** recreating the entire Spectral resolver
object model would duplicate libopenapi ownership without evidence it is needed
for the initial migration corpus.

**Smaller design:** a tested context subset with throwing unsupported accessors.
Use canonical vacuum source/path mapping for supported results.

**Outcomes preserved:** supported rules receive correct data; unsupported rules
cannot pass by reading an empty substitute.

**Verify:** referenced-document and missing-property fixtures; unsupported API
access produces a tool error and a failing harness case.

Confidence: high. Exact supported document semantics remain a differential test
obligation, not an assumption.

## [P2] A VM pool adds state ownership before there is a measured need

**Plan section:** Own VMs per document execution.

**Evidence:** Goja values belong to a runtime; the existing bridge and event loop
already own runtime execution, interruption and async completion.

**Problem:** pooling live closures across documents creates state leakage, reset
rules, queue semantics and lifetime problems on top of existing engine concurrency.

**Why it does not earn its complexity:** there is no measurement showing a pool is
required for the accepted outcome.

**Smaller design:** immutable compiled programs with one isolated module instance
per document execution scope; serialize its JS calls, retain native concurrency.

**Outcomes preserved:** closures, asynchronous functions, cancellation and parallel
independent documents work without sharing mutable VM state.

**Verify:** scope isolation, race tests, cancellation while queued, no per-node
module evaluation, and measured YAML/JS overhead.

Confidence: high. The lifecycle spike must resolve the adapter ownership before
publishing a new Go API.

## [P2] Separate CommonJS and ESM loaders multiply resolution behavior

**Plan section:** One module preparation path.

**Evidence:** `BuildVM` enables a CommonJS registry; its default source loader reads
local files. That does not implement ESM imports or package-export semantics.

**Problem:** an unrestricted CommonJS path plus an independent ESM bundle path can
resolve different dependencies and bypass import/provenance restrictions.

**Why it does not earn its complexity:** the required module forms can share one
preparation and resolver policy. A handwritten syntax detector is also unnecessary.

**Smaller design:** one esbuild-based preparation path, explicit compatibility
modules, and no fallback runtime filesystem loader for unresolved dynamic imports.

**Outcomes preserved:** CommonJS/ESM source support, local imports, module origins
and fail-closed unsupported dependency handling.

**Verify:** equivalent module-form fixtures; dynamic imports and remote executable
inheritance fail; all resolution inputs appear in provenance.

Confidence: medium-high. esbuild package-conditions behavior must be pinned and
proved during the first implementation chunk.

## Crushed plan

- Outcome: load supported executable rulesets through vacuum's existing engine,
  then verify them through the harness with correct findings and explicit errors.
- Reuse: rule models, merge/validation, custom-function interface, Goja event loop,
  fetch restrictions, canonical paths, shared command helpers and harness evidence.
- Sequence: compatibility corpus/runtime spike; module preparation; function and
  format adapters; shared callers; harness; companion quobix.com docs and examples.
- Remove: VM pools, a Node subprocess, duplicate validators/linters, global caches,
  separate module resolvers and silent approximate Spectral package aliases.
- Defer: full Node APIs, TypeScript, remote executable modules, documentInventory,
  in-process Spectral API replacement and unproved ruleset-package exports.
- Verify: pinned differential fixtures, cancellation/race/state isolation, all
  relevant command surfaces, full findings preservation, repeated performance
  measurements, runnable downloads and existing Hugo layouts.

The final plan includes documentation in ../quobix.com as a delivery requirement,
not an optional follow-up. No implementation or runtime checks were performed by
this plan-only review. Public ecosystem compatibility remains limited to the
explicit contract and must be demonstrated before the implementation is released.
