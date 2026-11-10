# terraform-plugin-framework modernization and simplification audit

Commit 00badb1 (origin/main, 'chore: Upgrade to Go 1.26 (#1326)') · 2026-11-10 (audit started 2026-10-24; the commit was still the tip of main on 2026-11-10) · ChihweiLHBird/terraform-plugin-framework (fork of hashicorp/terraform-plugin-framework), module github.com/hashicorp/terraform-plugin-framework

Audit by Claude Code with the codebase-modernization-audit skill: eight area auditors and eight independent verifiers (Opus, maximum effort), coordinated in one session. The repository was read-only throughout; every size marked "measured" comes from a draft applied to a scratch copy.

**Verdict.** The code base passes every gate it sets itself: golangci-lint with the project configuration, go vet and gofmt report nothing, and the 56-package suite is green in under 30 seconds. The audit still found 48 behaviour bugs. Four are tier 1: a provider crash through documented use of collection function returns (F1), two reflection panics when a model struct uses a named numeric type or holds a nil pointer (C1, C2), and action write-only attributes that have rejected every non-null value since they shipped in 1.17.0 (B37). Fifteen more are tier 2, among them warnings that ten RPC handlers silently drop (B17), automatic deferral that is lost or incomplete (B15, B16), and a GenerateResourceConfig capability the framework computes but never advertises (A1). The simplification potential is in internal packages, where per-type and protocol-5/6 twins are byte-identical after renaming. Generic bodies shrink the plan-modification files from 3,802 to 1,826 lines and the validation files from 1,708 to 1,024 (B4 to B6). A shared ArgumentsData core removes 551 net lines (A2), and one helper replaces 23 copied validation blocks in internal/reflect (C10). The public schema and value packages are duplicated per concept on purpose; the report records why sharing their struct definitions would break every provider. Moving to Go 1.26 idioms is mechanical and proven: any, new(expr) and reflect.TypeFor apply cleanly with go fix and the suite stays green.

Of the 47 entries under "Removed after verification", 1 is an item its verifier dropped, 7 are cross-area duplicates merged into a surviving item, 14 are ideas their own authors withdrew, and 25 are sub-claims that failed verification inside items that were kept and corrected.


## Inventory and scope

Go only: 1887 files, 450,062 lines (109,321 non-test, 340,741 test). Build: Go modules, go.mod `go 1.26.0`; CI lints with the go.mod version and runs `go test` on a Go 1.26 and 1.27 matrix. Direct dependencies: github.com/google/go-cmp v0.7.0, github.com/hashicorp/go-uuid v1.0.3, github.com/hashicorp/terraform-plugin-go v0.31.0, github.com/hashicorp/terraform-plugin-log v0.10.0; a separate tools/ module pins github.com/hashicorp/copywrite v0.22.0 for licence headers. Test harness: standard `testing`, table-driven, cmp.Diff comparisons, t.Parallel() everywhere (paralleltest lint); `go test ./...` = 56 packages, ~27 s on 4 cores, all green at this commit. CI proofs (.github/workflows/ci-go.yml): golangci-lint v2 with .golangci.yml (copyloopvar durationcheck errcheck forcetypeassert govet ineffassign makezero misspell nilerr paralleltest predeclared staticcheck unconvert unparam unused usetesting; gofmt), `go test` with coverage, and terraform-provider-corner acceptance tests for tfprotov5 and tfprotov6 against real Terraform versions. At this commit: golangci-lint (project config) 0 issues, go vet clean, gofmt -s clean. Library with a strict compatibility promise: every exported identifier outside internal/ is public API.

Swept exhaustively: every non-test Go file in internal/ (fwserver, fwschemadata, reflect, fwschema, fwtype, fwfunction, privatestate, toproto5/6, fromproto5/6, proto5server/6server, totftypes, fromtftypes, logging), the public packages (types, types/basetypes, attr, path, diag, tfsdk, resource, datasource, provider, ephemeral, action, list, statestore, function, providerserver) and the nine schema package families with their plan-modifier and default sub-packages; internal/testing fixtures; tooling and CI files. Test files (340k lines) were swept by tool (go fix, golangci-lint scratch config, staticcheck, deadcode) and read by sample where a lead pointed at them. Excluded: CHANGELOG.md, .changes/, docs/design (historical design records), META.d, LICENSE. No generated or vendored code exists in the module.

## Summary

- Items kept: 123 (120 confirmed, 3 plausible with a caveat)
- Bug groups: 48
- Claims removed after verification: 47
- Tier-1 items: 10

## Start here

- **A2** Move the protocol-neutral 587-line ArgumentsData body out of fromproto5/fromproto6 into one shared internal function; keep both fromprotoN.ArgumentsData as 5-line wrappers (+627/-1178 physical lines (net -551), measured with git diff --no-index --numstat on the author's drafts (fromproto5 +5/-589, fromproto6 +5/-589, core +611, doc.go +6) and matching the author's own A2_argumentsdata.diff; the stated '-1084/+573 (net -511)' is a non-blank-line count (1110 -> 599).)
- **A4** Delete fromproto5.EphemeralResultData and fromproto6.EphemeralResultData: no production caller, only their own tests (about -346/+0 lines (measured with wc: 53+120 per protocol))
- **B7** Split AttributeModifyPlan (602 lines) into named nesting-mode helpers and let BlockModifyPlan (425 lines) reuse the List/Set/Single ones (attribute_plan_modification.go +337/-309 and block_plan_modification.go +7/-384 as drafted (net -349). With direct dispatch, net -339.)
- **B37** ValidateActionConfig rejects every configured write-only action attribute (WriteOnlyAttributesAllowed hard-coded false), so action schemas' WriteOnly field (1.17.0, #1233) is unusable (1 line plus comment/doc lines (measured on a copy), plus one TestServerValidateActionConfig case with a WriteOnly attribute (known and unknown value))
- **C1** reflect.Into panics (reflect.Set) for named numeric Go types and for map targets whose key type is not plain string (about -13/+30 lines with the corrected map handling (measured on drafts/V-C/fix1b minus the no-op *big.Int Convert); the author's draft is -13/+22 but changes working inputs)
- **C2** reflect.FromValue panics on a nil *big.Int and on typed-nil pointers that implement attr.Value, Unknownable or Nullable (about +18 lines (measured on drafts/V-C/fix2, including a 7-line helper and a comment))
- **C10** internal/reflect: replace 23 copies of the ValidateableAttribute / TypeWithValidate validation block with one helper (about -560/+113 lines (measured, includes the new 42-line validate.go); internal/reflect non-test lines 2858 -> about 2410; //nolint:staticcheck in internal/reflect 24 -> 2; gocognit FromSlice 89 -> 46, FromMap 44 -> 17, FromStruct 49 -> 32, FromUnknownable/FromNullable 25 -> under 15)
- **C11** isValidFieldName compiles its regexp on every call: hoist it to a package variable (about 3x faster struct Get, 2.5x faster Set; 37 fewer allocations per struct field) (-2/+5 lines (measured: drafts/C/draft-regexp-hoist.diff))
- **E4** SetValue.Equal, MapValue.String and ObjectValue.String copy the whole collection once per element (quadratic allocation) (about -7/+7 lines in 3 files (measured on draft7))
- **F1** ListReturn/SetReturn/MapReturn.NewResultData ignore the CustomType's element type: a CustomType-only collection return crashes the provider on every Run error (Author's draft measured exactly -12/+36 (-4/+12 per file). A tighter draft that only reassigns value when ElementType is nil is +24/-0 (8 lines per file including a 2-line comment) with identical probe results. Tests about +120 (estimated).)

### PR batches for the tier-1 items

Each bullet is one PR unless it says otherwise. Land the bugs first, each with the failing test its item names.

1. **Crash and broken-feature fixes, one PR each:** B37 (action write-only: a one-line fix plus the doc and changelog updates its item lists), F1 (element type for CustomType-only list, set and map returns), C1 (named numeric types and map keys, with the corrected map check) and C2 (nil pointers, gated on the element type).
2. **Wins with no behaviour change, one PR each:** A4 (delete the dead EphemeralResultData converters), C11 (compile the field-name regexp once) and E4 (stop copying collections once per element).
3. **Larger refactors with byte-identical or golden-output proof:** A2 (shared ArgumentsData core, then A3 and A15), C10 (one validation helper in internal/reflect) and B7 (split AttributeModifyPlan and BlockModifyPlan, calling the helpers directly).

**Before opening any PR,** check upstream. B1's fix already exists in open PRs #1260 and #1311 (issues #1258 and #1219), so review one of those instead. C3 must wait for C6. The repo-wide go fix items (M1, M2, G1, G13, E9 and C18) should each land as their own PR, together right after a release to limit conflicts, and M3 after them.

## A. Protocol layer (toproto, fromproto, proto5/6server, providerserver)

### A2. Move the protocol-neutral 587-line ArgumentsData body out of fromproto5/fromproto6 into one shared internal function; keep both fromprotoN.ArgumentsData as 5-line wrappers

- Scope: internal/fromproto5, internal/fromproto6, new internal package (e.g. internal/fromprotocommon) or internal/fwserver | Kind: refactor | Tier: 1 (do first) | Verdict: CONFIRMED
- Where: internal/fromproto5/arguments_data.go:20-608; internal/fromproto6/arguments_data.go:20-608
- Current: Correction to the count: only 3 lines mention tfprotovN (import, doc comment, signature); the 4th line that differs between the twins is the package clause. With the range clause (55) and argument.Unmarshal (81) that makes 6 protocol-touching lines.
- Replacement: Create `func ArgumentsData(ctx context.Context, numArguments int, unmarshal func(position int, typ tftypes.Type) (tftypes.Value, error), definition function.Definition) (function.ArgumentsData, *function.FuncError)` in a protocol-neutral internal package (the draft uses a new internal/fromprotocommon; internal/fwserver, already imported by fromproto5/6, also works; internal/fwfunction does not, because `function` imports it). Its body is the current body with len(arguments) -> numArguments (6 sites), the range clause -> `for position := range numArguments`, and `argument.Unmarshal(t)` -> `unmarshal(position, t)`. fromprotoN.ArgumentsData keeps its signature: `unmarshal := func(position int, typ tftypes.Type) (tftypes.Value, error) { return arguments[position].Unmarshal(typ) }; return fromprotocommon.ArgumentsData(ctx, len(arguments), unmarshal, definition)`. Protocol-specific lines that must stay per protocol: the []*tfprotovN.DynamicValue parameter and the `arguments[position].Unmarshal(typ)` call.
- Why equivalent: A move plus parameterisation: statement order, every FuncError text, every logging.FrameworkTrace call and every return value are unchanged. The wrapper reads len(arguments) once and the slice is never mutated, so numArguments equals each former len(arguments); arguments[position] is read at the point where the range variable was used. A callback is used on purpose instead of a generic `T interface{ Unmarshal(tftypes.Type) (tftypes.Value, error) }`: a probe showed that calling a value-receiver method through a type parameter turns the nil-element panic 'runtime error: invalid memory address or nil pointer dereference' into 'value method ... called using nil *DynamicValue pointer'. The callback keeps the direct call in protocol code, so even that panic text is identical. Only the @caller file:line metadata of the 6 trace log entries changes, as it does for any edit.
- Size: +627/-1178 physical lines (net -551), measured with git diff --no-index --numstat on the author's drafts (fromproto5 +5/-589, fromproto6 +5/-589, core +611, doc.go +6) and matching the author's own A2_argumentsdata.diff; the stated '-1084/+573 (net -511)' is a non-blank-line count (1110 -> 599).
- Risk: low: internal packages only, mechanical, existing tests unchanged. Design note: it would be the first package shared across the deliberately twinned fromproto5/6 packages; internal/fwserver is a cycle-free alternative home (verified).
- Proof: Pinned by TestArgumentsData and TestArgumentsData_ParameterValidators in both fromproto5 and fromproto6 (200 subtests), TestCallFunctionRequest, and TestServerCallFunction in proto5server/proto6server. In a sandbox copy: go build ./internal/..., go vet, go test ./internal/fromproto5 ./internal/fromproto6 and go test -run CallFunction ./internal/proto5server ./internal/proto6server pass unchanged; golangci-lint with the project .golangci.yml on the three packages reports 0 issues. Drafts: drafts/A/drafts/A2_*.go; diff: drafts/A/A2_argumentsdata.diff.
- Follow-ups that build on this: A3 (validator loop collapse, -244 more lines) and A15 (test consolidation, about -3000 test lines).
- The same callback trick could share fromproto DynamicValue/Config/Plan/State/EphemeralResultData/IdentityData, but each pair saves only about 25 lines and needs a constructor callback for the distinct tfsdk result types. Rejected; see notes.

### A4. Delete fromproto5.EphemeralResultData and fromproto6.EphemeralResultData: no production caller, only their own tests

- Scope: internal/fromproto5, internal/fromproto6 | Kind: cleanup | Tier: 1 (do first) | Verdict: CONFIRMED
- Where: internal/fromproto5/ephemeral_result_data.go:1-53 (+ ephemeral_result_data_test.go:1-120); internal/fromproto6/ephemeral_result_data.go:1-53 (+ ephemeral_result_data_test.go:1-120)
- Current: `grep -rn "EphemeralResultData(" --include=*.go . | grep -v _test.go` finds only the two fromproto definitions and the toproto5/6 converters (which go the other way). No protocol request carries ephemeral result data: Renew/CloseEphemeralResourceRequest only have TypeName and Private. deadcode -test misses them because their own tests are roots.
- Replacement: Delete both files and both test files. fwschemadata.DataDescriptionEphemeralResultData stays (still used by toproto5/6 and tfsdk).
- Why equivalent: No production reference anywhere in the module, and the packages are internal, so nothing outside can call them. Removing them changes no reachable behaviour.
- Size: about -346/+0 lines (measured with wc: 53+120 per protocol)
- Risk: low: internal, unreferenced
- Proof: grep shows zero non-test references; deleting them in a sandbox and running go build ./... plus go test ./internal/fromproto5 ./internal/fromproto6 is enough (no other package references them).
- Systematic sweep: every exported identifier in toproto5/6, fromproto5/6, totftypes, fromtftypes and logging was checked for non-test references (same-file uses counted). These two were the only ones with none.

### A3. Replace the 12 copy-pasted ParameterWith<Type>Validators loops in ArgumentsData with one generic helper

- Scope: ArgumentsData core (after A2), or both fromproto5/6 twins if A2 is not taken | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/fromproto5/arguments_data.go:162-559; internal/fromproto6/arguments_data.go:162-559
- Current: Addendum: the 12 blocks are identical except a local variable name in the List case (listValue vs listValuable), which has no behavioural effect.
- Replacement: `func runParameterValidators[Valuable, Value, Validator any](ctx, pos int64, attrValue attr.Value, validators []Validator, valuableName string, toValue func(Valuable, context.Context) (Value, diag.Diagnostics), validate func(context.Context, Validator, Value) *function.FuncError) *function.FuncError` holding the loop once. Each case becomes `funcError = function.ConcatFuncErrors(funcError, runParameterValidators(ctx, pos, attrValue, p.GetValidators(), "basetypes.BoolValuable", basetypes.BoolValuable.ToBoolValue, func(ctx context.Context, v function.BoolParameterValidator, value basetypes.BoolValue) *function.FuncError { resp := &function.BoolParameterValidatorResponse{}; v.ValidateParameterBool(ctx, function.BoolParameterValidatorRequest{ArgumentPosition: pos, Value: value}, resp); return resp.Error }))`.
- Why equivalent: Same case order (first matching interface still wins), same per-validator steps and texts (the valuable name strings match all 12 originals, verified by grep), same validator request values. The type assertion to a type-parameter interface has the same semantics as the explicit one. The only structural change is accumulating into a local FuncError and concatenating once. function.ConcatFuncErrors is associative: it joins non-empty Text with \n and keeps the first non-nil FunctionArgument, so Concat(F, Concat(e1..en)) equals Concat(...Concat(F, e1)..., en) in Text and FunctionArgument. The validate callback returns *function.FuncError, not error, so a nil response error cannot become a non-nil interface.
- Size: +135/-384 on the shared core (net -249), measured with numstat (the author's A3_validators.diff agrees); stated '-372/+128 (net -244)'. Twice that if applied to both twins without A2.
- Risk: low-medium: behaviour is pinned by tests, but it introduces a 3-type-parameter generic plus 12 closures; maintainers may prefer the explicit form
- Proof: TestArgumentsData_ParameterValidators (about 50 cases per package, covering every type, the wrong-Valuable path and validator errors) and TestArgumentsData pass unchanged on the sandbox draft; golangci-lint (project config): 0 issues; gofmt clean. Draft: drafts/A/drafts/A3_fromprotocommon_arguments_data_validators.go; diff: drafts/A/A3_validators.diff.
- Independent of A2 in principle, but much cheaper after it (one copy instead of two).

### A13. Remove the duplicated back-to-back `if fwResp.Diagnostics.HasError()` block in MoveResourceState (both servers)

- Scope: internal/proto5server/server_moveresourcestate.go, internal/proto6server/server_moveresourcestate.go | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/proto5server/server_moveresourcestate.go:43-49; internal/proto6server/server_moveresourcestate.go:42-48
- Current: Two identical consecutive blocks, `if fwResp.Diagnostics.HasError() { return toprotoN.MoveResourceStateResponse(ctx, fwResp), nil }`, with nothing in between. The second can never be taken when the first was not. A repo-wide scan for consecutive identical if-blocks finds only these two sites.
- Replacement: Delete the second block (and its blank line).
- Why equivalent: The condition reads only fwResp.Diagnostics, which nothing modifies between the two checks, so the second check is always false when reached.
- Size: about -8/+0 lines (measured: 4 per file)
- Risk: low
- Proof: Pinned by TestServerMoveResourceState in both packages. Rejected variant: moving the first check up between the ResourceSchema and ResourceIdentitySchema lookups (to match other handlers) is NOT equivalent, because it would skip the identity-schema lookup and its diagnostics when the resource-schema lookup errors.

### A14. Replace the 11 schema-conversion sort.Slice calls, whose nil checks cannot be reached, with slices.SortFunc + strings.Compare

- Scope: internal/toproto5 (block.go, identity_schema.go, schema.go), internal/toproto6 (same + schema_attribute.go) | Kind: modernization | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/toproto5/block.go:73,85; identity_schema.go:37; schema.go:48,60; internal/toproto6/block.go:73,85; identity_schema.go:37; schema.go:48,60; schema_attribute.go:80
- Current: Each site is an 11-line `sort.Slice(x, func(i, j int) bool { if x[i] == nil { return true }; if x[j] == nil { return false }; return x[i].Name < x[j].Name })`. Elements can never be nil: SchemaAttribute, Block and IdentitySchemaAttribute return either (nil, err), which makes the caller return, or a non-nil value. These are 11 of the repo's 12 non-test sort.Slice calls.
- Replacement: `slices.SortFunc(x, func(a, b *tfprotovN.SchemaAttribute) int { return strings.Compare(a.Name, b.Name) })` (element type per site); imports sort -> slices, strings. If A10 lands, delete these sorts instead.
- Why equivalent: Both sorts are unstable pdqsorts, but the keys (map-derived Name/TypeName) are unique, so the sorted order is unique and identical. The removed nil branches cannot be reached. Nil versus empty slices are untouched.
- Size: -117/+36 measured (the author's own A14_sortfunc.diff has 124 '-' and 43 '+' lines including 7 headers each: 11 sites x -10/+2 plus 7 import swaps x -1/+2); stated '-95/+36'.
- Risk: low
- Proof: Sandbox draft: go vet plus go test ./internal/toproto5 ./internal/toproto6 ./internal/proto5server ./internal/proto6server pass; golangci-lint (project config) on toproto5/6: 0 issues. Draft: drafts/A/drafts/A14_sortfunc; diff: drafts/A/A14_sortfunc.diff.
- Superseded by A10 if A10 is accepted (sorted iteration makes the post-sorts redundant).

### A15. After A2, keep one copy of the 3134-line twin ArgumentsData tests in the shared package and only a few wrapper tests per protocol

- Scope: internal/fromproto5/arguments_data_test.go, internal/fromproto6/arguments_data_test.go | Kind: test | Tier: 3 (fold-in) | Verdict: PLAUSIBLE, medium confidence
- Where: internal/fromproto5/arguments_data_test.go:1-3134; internal/fromproto6/arguments_data_test.go:1-3135
- Current: Two test files, identical after 5/6 normalisation except one blank line, each with 100 subtests (TestArgumentsData, TestArgumentsData_ParameterValidators). Inputs are built with DynamicValueMust and tfprotovN.DynamicValue.
- Replacement: Move the fromproto6 file to the shared package. It can keep building tfprotov6 DynamicValues; its two runner lines call the core with `len(in)` and `func(i int, t tftypes.Type) (tftypes.Value, error) { return in[i].Unmarshal(t) }`. Replace each protocol's file with about 60 lines of wrapper smoke tests (argument-count mismatch, one success, one unmarshal error, one variadic).
- Why equivalent: Test-only. The core is protocol-neutral after A2, so the 100 cases still run once against it. The wrappers are 4-line functions covered by smoke tests. tfprotov5/6 DynamicValue.Unmarshal are twin implementations in terraform-plugin-go.
- Size: about -3134/+150 if the fromproto6 file is moved as a rename (about -6269/+3255 without rename detection) (estimated).
- Risk: low-medium: less end-to-end coverage per protocol; keep both wrapper smoke tests
- Proof: Depends on A2. Measure by running both packages' tests plus the shared package's tests with -cover before and after.
- Other twin test duplication is deliberate and left alone: 30 toproto, 28 fromproto and 12 protoNserver test-file pairs are identical after normalisation (4870/5183/3996 lines per side).
- Verifier caveat: Depends on A2 being accepted; size estimated, not drafted; trades per-protocol end-to-end coverage for de-duplication.

### A16. TestServerCancelInFlightContexts relies on a 200 ms sleep to know that 10 goroutines have registered; use a WaitGroup barrier

- Scope: internal/proto5server/serve_test.go, internal/proto6server/serve_test.go | Kind: test | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/proto5server/serve_test.go:18-56; internal/proto6server/serve_test.go:17-55
- Current: Goroutines call s.registerContext concurrently; the test sleeps 200 ms ('avoid any race conditions around canceling the contexts before they're all set up') and then cancels. A goroutine that registers after the cancel is never canceled and fails after 10 s.
- Replacement: Add `registered := new(sync.WaitGroup)`; each goroutine calls registered.Done() right after registerContext; replace the sleep with registered.Wait(). Concurrent registration is still exercised.
- Why equivalent: Test-only. It checks the same property without timing assumptions and saves 2 x 200 ms. Combine with the coordinator's WaitGroup.Go / range-over-int modernization of the same loop (both behaviour-preserving).
- Size: about -6/+6 lines per package (estimated)
- Risk: low
- Proof: The existing test passes; run it with -race -count=50 after the change.
- Edits the same test loops as G13 (range-over-int and WaitGroup.Go): land one, then rebase the other.

### A17. providerserver public docs are inaccurate: Serve does not stop when ctx is canceled, and ServeOpts.ProtocolVersion refers to the removed tfsdk.Attribute

- Scope: providerserver (public API doc comments only) | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: providerserver/providerserver.go:78; providerserver/serve_opts.go:31-35
- Current: 'Serve serves a provider, blocking until the context is canceled.' But ctx reaches only opts.validate, and validateAddress ignores it (`_ context.Context`). tf5server.Serve/tf6server.Serve take no context and block until the plugin process is stopped. The ProtocolVersion doc says '- tfsdk.Attribute cannot use Attributes field (nested attributes).', but tfsdk.Attribute no longer exists (`grep -rn '^type Attribute' tfsdk/` finds nothing).
- Replacement: Reword: 'Serve serves a provider and blocks until the plugin server exits; ctx is currently only used while validating opts.' and '- Nested attributes (for example schema.ListNestedAttribute) are not supported.'
- Why equivalent: Comment-only change; no signature or behaviour change.
- Size: about -3/+4 lines (estimated)
- Risk: low
- Proof: Read providerserver.go:79-128 and serve_opts.go:46-87; tf5server.Serve signature in terraform-plugin-go v0.31.0 tfprotov5/tf5server/server.go:257.
- The deadcode lead 'providerserver.Serve unreachable' is a false positive: it is public API called from providers' main packages.

## B. Framework server (internal/fwserver)

### B7. Split AttributeModifyPlan (602 lines) into named nesting-mode helpers and let BlockModifyPlan (425 lines) reuse the List/Set/Single ones

- Scope: 2 files in internal/fwserver | Kind: refactor | Tier: 1 (do first) | Verdict: CONFIRMED
- Where: internal/fwserver/attribute_plan_modification.go:137-686; internal/fwserver/block_plan_modification.go:51-449
- Current: AttributeModifyPlan = setup, per-type dispatch, three gates and a 550-line switch with List/Set/Map arms (~150 lines each) and a Single arm (~85). BlockModifyPlan repeats the switch without Map: after de-indenting and replacing NestedBlockObjectPlanModify(ctx, nestedBlockObject, objectReq, objectResp) with the attribute call, its List/Set/Single arms are byte-identical to the attribute arms (asserted by the drafting script; dupl leads 138-288, 289-439, 440-590, block 52-202, 203-353).
- Replacement: type nestedObjectPlanModifyFunc func(context.Context, planmodifier.ObjectRequest, *ModifyAttributePlanResponse); listNestedPlanModify, setNestedPlanModify, mapNestedPlanModify and singleNestedPlanModify(ctx, nestedObjectPlanModify, req, resp) hold the arm bodies verbatim. AttributeModifyPlan builds `nestedObjectPlanModify := func(ctx, req, resp) { NestedAttributeObjectPlanModify(ctx, nestedAttributeObject, req, resp) }` and calls them from a 4-case switch (default arm unchanged); BlockModifyPlan does the same with NestedBlockObjectPlanModify for List/Set/Single. Result: AttributeModifyPlan 76 lines, BlockModifyPlan 48.
- Why equivalent: Each arm moves verbatim (de-indented one level); the only edit is calling the nested-object function through a closure that forwards the same arguments. The switch is the last statement of both functions, so a return inside a moved arm (now returning from the helper) ends the outer function's work exactly as before. ctx (with the attribute logging field for attributes, without it for blocks, as today), req and resp are passed through; all order-pinning state (cumulative resp.Diagnostics checks, resp.Private threading, RequiresReplace append order, resp.AttributePlan after the per-type modifiers) flows through the same resp pointer.
- Size: attribute_plan_modification.go +337/-309 and block_plan_modification.go +7/-384 as drafted (net -349). With direct dispatch, net -339.
- Risk: low: verbatim move, reviewable with git diff -w --color-moved. Allocation note: the nested-object function is reached through a func value, so objectResp now escapes. That is +1 heap allocation (about 80 B) per nested-object element and per single nested object (nested fixture: 17,741 -> 17,757 allocs/op for 16 nested objects). A direct-dispatch value avoids this: a struct holding the NestedAttributeObject, the NestedBlockObject and an isBlock flag, with a planModify method that calls NestedAttributeObjectPlanModify or NestedBlockObjectPlanModify directly. It keeps parity (17,741), has no escapes, and passes the fwserver tests and lint (drafts/V-B1/arms-direct/direct.diff).
- Proof: TestAttributeModifyPlan (list/set/map/single nested attribute cases), TestBlockModifyPlan (list/set/single nested blocks), TestSchemaModifyPlan and TestServerPlanResourceChange* pass on the draft; go test ./internal/proto5server/ ./internal/proto6server/ also pass.
- Resolves the gocognit leads (133 and 95).
- Independent of B4: the arm bodies do not touch the per-type functions (the draft happened to be applied on top of B4).
- Must keep the per-arm order differences: the Single arm merges nested diagnostics/Private/RequiresReplace before re-wrapping, the collection arms after (see notes).

### B4. Replace the 12 byte-identical AttributePlanModify<Type> bodies (158 lines each) with one generic valuePlanModify; keep the 12 exported names as one-line wrappers

- Scope: 1 file changed (12 functions) + 1 new file in internal/fwserver | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/fwserver/attribute_plan_modification.go:688-2606
- Current: AttributePlanModifyBool, Float32, Float64, Int32, Int64, List, Map, Number, Object, Set, String, Dynamic: 158 lines each; after `sed s/<Type>/TYPE/g` over each (doc comment through closing brace) all 12 are byte-identical (0 differing lines). Each asserts config/plan/state to basetypes.<T>Valuable (3 identical diagnostic blocks), calls To<T>Value, coerce<T>Typable, builds planmodifier.<T>Request, loops over <T>PlanModifiers() with two trace logs, merges Diagnostics/Private/RequiresReplace and re-wraps with ValueFrom<T>.
- Replacement: As drafted, plus one change: each spec carries the two literal trace messages (callingMsg "Calling provider defined planmodifier.Bool", calledMsg "Called provider defined planmodifier.Bool"), and these are passed to logging.FrameworkTrace. Do not build them at run time with "Calling provider defined planmodifier."+t.name. Verified variant: drafts/V-B1/generic-msgfix/msgfix.diff.
- Why equivalent: The 158-line body exists once with the same statements in the same order: the assertion to a type parameter whose argument is the same interface type; diagnostics built as "Invalid "+name+" "+kind+" Plan Modifier Value Type" and the detail with strings.ToLower(kind) (ASCII, byte-identical) and the same %T on the same attr.Value ("<nil>" for nil); trace messages "Calling/Called provider defined planmodifier."+name with the same description field (map[string]any is the same type); a fresh response per modifier holding only PlanValue and Private; the same Private/RequiresReplace/Diagnostics merge and early returns. The modifier getter is passed as a method value and called at the same point as today (after the coercions and only if they succeed), so an embedding type that overrides the getter sees the same calls. The provider receives an identical request value and a *planmodifier.<T>Response (same dynamic type under reflection) pointing at a struct of identical layout, as today.
- Size: attribute_plan_modification.go +12/-1873 and new value_plan_modification.go +351 with the message fields (+325 as drafted). With B5, the plan-modification files go from 3,802 to 1,826 lines (1,800 as drafted).
- Risk: medium: a large mechanical deletion; 4 type parameters and the struct-conversion idiom need a comment; diagnostic literals become templates. Performance: as drafted, the runtime concatenation of the trace messages costs 2 heap allocations per provider plan-modifier call, even with logging off. AttributePlanModifyString with 2 modifiers goes from 5 to 9 allocs/op and 272 to 464 B/op (about 3x ns/op in a microbenchmark); PlanResourceChange with 30 attributes x 2 modifiers goes from 8,302 to 8,422 allocs/op. With literal messages per spec: parity (5 allocs/op; 8,302), golden identical, lint clean.
- Proof: As stated, plus a gap: the golden harness only produces the invalid-type diagnostics and the modifier error. The To<T>Value, coerce<T>Typable and ValueFrom<T> error returns that the generic body also moves are exercised by neither the harness nor the suite (per-type functions are 76.1% covered), so they are verified by reading only. B13 should cover them first.
- Golden output includes 60 invalid-type diagnostics, 120 trace entries and 60 modifier calls, all identical.
- If generics are not wanted, the smaller step is one helper for the 45 identical invalid-value-type diagnostic blocks (about -360 lines, estimated) and one for the 30 trace-log pairs (about -180, estimated); per-type bodies cannot be shared without generics because every method name differs.
- Removes 24 of the area's 68 map[string]interface{} sites (B5 6 more, B6 30 more).
- Can be reviewed in type groups if preferred; the generic core must land with the first group.
- Also: the @caller of the trace entries moves to value_plan_modification.go. The golden harness excludes @caller, and any edit moves it anyway. The map[string]interface{} count holds: 68 in the 7 area files, 8 left after B4/5/6.

### B5. BlockPlanModifyList/Object/Set: reuse valuePlanModify with kind "Block" (depends on B4)

- Scope: 1 file, 3 functions | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/fwserver/block_plan_modification.go:451-929
- Current: Three 158-line functions, identical to each other after normalisation and to the attribute twins except the receiver name and 'Block'/'block' in the three invalid-value-type diagnostics.
- Replacement: `func BlockPlanModifyList(ctx, block, req, resp) { valuePlanModify(ctx, listPlanModify, "Block", block.ListPlanModifiers, req, resp) }` (and Object/Set), sharing B4's specs.
- Why equivalent: As B4; the kind argument reproduces 'Invalid List Block Plan Modifier Value Type' and '... perform List block plan modification.' byte for byte.
- Size: +3/-469 (measured)
- Risk: low once B4 is in. It inherits B4's allocation fix (literal trace messages per spec).
- Proof: TestBlockPlanModifyList/Object/Set pass on the draft; the BlockPlanModify* sections of the golden harness output (including the 'Block' diagnostics) are identical.
- Separate PR only to keep B4's review focused; could be folded into it.

### B6. Replace the 15 byte-identical AttributeValidate<Type>/BlockValidate<Type> bodies (64 lines each) with one generic valueValidate

- Scope: 2 files changed (15 functions) + 1 new file in internal/fwserver | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/fwserver/attribute_validation.go:203-994; internal/fwserver/block_validation.go:176-373
- Current: 12 AttributeValidate<T> byte-identical after normalisation; 3 BlockValidate<T> identical to them except variable names (block validators also say 'Attribute' in their diagnostic, block_validation.go:188/254/320).
- Replacement: value_validation.go (205 lines, drafted in drafts/B1/draft-generic-v): valueValidateRequest[V] with the field order of validator.<T>Request (Path, PathExpression, Config, ConfigValue, ClientCapabilities), valueValidateResponse{Diagnostics}, a 7-line spec per type (To<T>Value method expression and a Validate<T> adaptor using struct conversion), one generic body; 15 one-line wrappers. No kind parameter is needed because the block texts are the attribute texts.
- Why equivalent: Same statements in the same order once: the same assertion and diagnostic, a fresh response per validator, both trace logs, every validator runs even after an error (no early exit), diagnostics appended in order. The validator getter is a method value called at the same point.
- Size: attribute_validation.go +12/-744, block_validation.go +3/-186, new file +231 with the message fields (+205 as drafted). 1,708 -> 1,024 lines (998 as drafted).
- Risk: low-medium: same idiom as B4. As drafted, the 'Calling/Called provider defined validator.'+t.name concatenation adds 2 heap allocations per validator call: AttributeValidateString with 2 validators goes from 3 to 7 allocs/op and 144 to 336 B/op. Literal messages per spec restore parity (verified).
- Proof: TestAttributeValidate<Type> x12 and TestBlockValidateList/Object/Set pass on the draft; golden differential harness drafts/B1/golden/zz_golden_b1_validate_test.go (wrong/nil config, a 3-validator chain with warning, error and an empty non-nil Diagnostics, request recording including ClientCapabilities, log capture): 255-line outputs identical (30 diagnostics, 90 trace entries, 45 calls). Add first: B13.
- Keep the 'Attribute' wording for blocks; changing it would be a separate behaviour change.

### B13. Pin the invalid-value-type diagnostics and the trace messages of the 30 per-type plan-modify/validate functions

- Scope: 4 test files in internal/fwserver | Kind: test | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: grep -rhoE 'Invalid [A-Za-z0-9]+ (Attribute|Block) (Plan Modifier|Validator) Value Type' --include=*_test.go . (0 hits); grep -rn 'Calling provider defined planmodifier\|Calling provider defined validator' --include=*_test.go . (0 hits)
- Current: No test exercises the 60 invalid-value-type branches (3 per plan-modify function x 15, 1 per validate function x 15) or asserts the 'Calling/Called provider defined ...' trace messages.
- Replacement: As stated, plus cases for the To<T>Value error, the coerce<T>Typable error and the ValueFrom<T> error returns, using custom types that return error diagnostics or a non-Typable Type(). These branches are equally uncovered and are also moved by B4/B6.
- Why equivalent: Tests only.
- Size: about +400-500 lines (estimated): the 60 literal expected diagnostics alone are about 300 lines, plus the error-return cases and the log-capture test.
- Risk: low
- Proof: Gap measured by the greps above; the generated golden harness in drafts/B1/golden covers the same ground and can seed the expected values.
- Prerequisite for B4, B5 and B6; useful on its own.

### B24. Replace 26 byte-identical Configure preambles with one helper per concept (configureResource, configureDataSource, configureEphemeralResource, configureAction, configureStateStore, configureListResource)

- Scope: internal/fwserver, 26 handler files + 1 new file | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: grep -n -E 'WithConfigure\); ok \{' internal/fwserver/*.go (non-test): 27 sites; resource x9 (create, delete, importresourcestate, planresourcechange, readresource, updateresource, upgraderesourceidentity, upgraderesourcestate, validateresourceconfig), statestore x7 (deletestate, getstates, lockstate, readstatebytes, unlockstate, writestatebytes, validatestatestoreconfig), ephemeral x4 (close, open, renew, validate), action x3 (invoke, plan, validate), datasource x2 (readdatasource, validate), list x1 (validatelistresourceconfig); server_listresource.go:105 excluded
- Current: Each site is the same 18-line block: type assertion, '<Concept> implements <Concept>WithConfigure' trace, ConfigureRequest{ProviderData: s.<Concept>ConfigureData} (state store: StateStoreData: s.StateStoreConfigureData.StateStoreConfigureData), Calling/Called traces, Append, `if resp.Diagnostics.HasError() { return }`. md5 of the extracted blocks: 9+6+4+3+2+1 identical groups; validatestatestoreconfig differs only in the local variable name (statestoreWithConfigure); ListResource differs (B23 log text, diagnostics streamed as a ListResult).
- Replacement: func (s *Server) configureResource(ctx, r resource.Resource) (diag.Diagnostics, bool) returning (nil, false) when r does not implement ResourceWithConfigure, otherwise the same traces, call and (configureResp.Diagnostics, true); each site becomes `if diags, ok := s.configureResource(ctx, req.Resource); ok { resp.Diagnostics.Append(diags...); if resp.Diagnostics.HasError() { return } }`. Draft: drafts/B2/drafts/configure (new server_configure.go, 141 lines).
- Why equivalent: Same type assertion on the same value, same three trace messages and ProviderData field (36 strings and fields compared in order between the originals and the helpers: identical), same Append and the HasError check only when Configure exists, so a response that already holds an error on entry behaves as before. No other statements move. (A variant returning only diagnostics with an unconditional HasError check was rejected: it returns early for a pre-existing error when the type has no Configure.)
- Size: +193/-339 lines, 27 files (measured on draft)
- Risk: low: mechanical; Configure traces are not pinned by tests
- Proof: Draft builds, go vet clean, project .golangci.yml 0 issues, go test ./internal/fwserver ./internal/proto5server ./internal/proto6server pass. ProviderData plumbing and the early return are pinned by the configure cases in 20 handler tests (grep -c ConfigureMethod per test file); the 6 Validate*Config tests have none (count 0): add one per concept first, ideally asserting the 3 trace lines with tfsdklogtest.
- Independent of the B17 bug fix (different lines).
- Do B23 first if ListResource should use configureListResource too.

### B25. One generic factoryRegistry[T] for the seven lazily built <Concept>Funcs registries, with the per-concept log and diagnostic text kept as data

- Scope: internal/fwserver, 6 files + 1 new file | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: fields server.go:85-97, 108-120, 131-143, 159-171, 173-182, 248-260, 284-296; methods server.go:317-370 (DataSourceFuncs), :646-699 (ResourceFuncs); server_actions.go:35-95; server_ephemeralresources.go:35-96; server_functions.go:121-179; server_listresources.go:34-116; server_statestore.go:35-95
- Current: Seven copies (420 lines plus 21 map/diags/mutex fields) of: trace lock message, Mutex Lock with defer Unlock, return the cached map and diags when non-nil, (except functions) s.ProviderTypeName(ctx), make the map, (ephemeral, action, state store, function) With* interface check returning the empty map, Calling/Called traces around the provider's factory method, then per factory: instantiate, Metadata, empty-name error, 'Found' trace, duplicate-name error, (list only) matching-managed-resource check, store. After normalising concept names, pairs share 21-32 of 41-67 non-blank lines; the rest are strings and types.
- Replacement: factoryRegistry[T]{mu sync.Mutex; funcs map[string]func() T; diags diag.Diagnostics} with load(ctx, text *factoryRegistryText, prepare func() (factories func(context.Context) []func() T, typeName func(T) string), accept func(T, string, *diag.Diagnostics) bool). prepare runs under the lock after the cache check and does what each concept does today before the loop (ProviderTypeName, interface check; nil factories = not implemented); accept carries the list-only check. ListResourceFuncs keeps its interface check before the lock (returns nil, nil uncached). Draft: drafts/B2/drafts/registry (registry.go 91 lines).
- Why equivalent: Same lock scope and order (lock trace, Lock, cache check, then ProviderTypeName and interface check, then Calling/Called), same iteration over the provider's slice in order (first-seen name wins, no sorting, the map is only used for membership), same nil-vs-empty results (non-nil empty map cached when the With* interface is missing; nil, nil for list; nil diags when no errors), same AddError dedup behaviour, same %T/%s formatting (the detail formats contain exactly one verb and the constant tail is appended unformatted). All texts verified equal programmatically, per concept, including logging keys (see notes).
- Size: +257/-423 lines, 7 files (measured on draft)
- Risk: medium: central code path; mitigated by existing pins
- Proof: Draft builds, vet clean, project lint 0 issues; go test ./internal/fwserver ./internal/proto5server ./internal/proto6server ./providerserver pass. Verifier runtime probe (drafts/V-B2/zz_vb2_registry_test.go.txt, run in drafts/V-B2/repo and drafts/V-B2/repo-registry) produces byte-identical output (242 lines: all registry diagnostics, map keys, nil-ness, and the full trace log including every 'Found ...' line) before and after; it can be added as the missing 'Found' trace test.
- Diagnostics kept per concept (Summary / Detail prefix, all followed by 'This is always an issue with the provider and should be reported to the provider developers.'): DataSource 'Data Source Type Name Missing' / 'The %T DataSource returned an empty string from the Metadata method. ', 'Duplicate Data Source Type Defined' / 'The %s data source type name was returned for multiple data sources. Data source type names must be unique. '; Resource 'Resource Type Name Missing' / 'The %T Resource returned ...', 'Duplicate Resource Type Defined' / 'The %s resource type name was returned for multiple resources. Resource type names must be unique. '; Ephemeral 'Ephemeral Resource Type Name Missing' / 'The %T EphemeralResource returned ...', 'Duplicate Ephemeral Resource Type Defined' / 'The %s ephemeral resource type name was returned for multiple ephemeral resources. Ephemeral resource type names must be unique. '; Action 'Action Type Missing' / 'The %T Action returned ...', 'Duplicate Action Defined' / 'The %s action type was returned for multiple actions. Action types must be unique. '; StateStore 'State Store Type Missing' / 'The %T state store returned ...', 'Duplicate State Store Type Defined' / 'The %s state store type was returned for multiple state stores. State store type names must be unique. '; Function 'Function Name Missing' / 'The %T Function returned ...', 'Duplicate Function Name Defined' / 'The %s function name was returned for multiple functions. Function names must be unique. '; ListResource 'ListResource Type Name Missing' / 'The %T ListResource returned ...', 'Duplicate ListResource Type Defined' / 'The %s ListResource type name was returned for multiple list resources. ListResource type names must be unique. ' plus 'ListResource Type Defined without a Matching Managed Resource Type' (in accept).
- Traces kept per concept: lock 'Checking DataSourceTypes lock' / 'Checking ResourceTypes lock' / 'Checking EphemeralResourceFuncs lock' / 'Checking ActionFuncs lock' / 'Checking StateStoreFuncs lock' / 'Checking FunctionTypes lock' / 'Checking ListResourceFuncs lock'; provider call 'Calling provider defined Provider DataSources' / 'Provider Resources' / 'Provider EphemeralResources' / 'Actions' / 'StateStores' / 'Provider Functions' / 'ListResources'; found 'Found data source type' (tf_data_source_type) / 'Found resource type' (tf_resource_type) / 'Found ephemeral resource type' / 'Found action' / 'Found state store' / 'Found function' / 'Found resource type' (tf_list_resource_type, see B23).
- With B19 the Clip fix becomes one line in load.
- Apply before B26/B27 (same files).

### B28. Validate*Config twins: one generic runConfigValidators loop and one validateConfigSchema tail for the seven handlers

- Scope: internal/fwserver/server_validate*config.go (7 files) + 1 new file | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/fwserver/server_validate{action,datasource,ephemeralresource,listresource,provider,resource,statestore}config.go: config-validator loops (e.g. server_validatedatasourceconfig.go:58-84) and schema tails (e.g. :100-118)
- Current: Seven handlers of 80-90 lines; after normalising concept names, pairs share 56-82 of 86-95 non-blank lines (drafts/B2/val/*.norm). The validator loop (27 lines) differs only in the validator interface, its method (ValidateDataSource, ValidateResource, ValidateProvider, ValidateEphemeralResource, ValidateAction, ValidateStateStore, ValidateListResourceConfig) and the trace name ('ConfigValidator' for data source and provider, '<Concept>ConfigValidator' otherwise); the 18-line schema tail differs only in WriteOnlyAttributesAllowed (req.ClientCapabilities for resources, false otherwise) and comments.
- Replacement: runConfigValidators[V interface{ Description(context.Context) string }](ctx, name string, validators []V, diags *diag.Diagnostics, validate func(V) diag.Diagnostics) and validateConfigSchema(ctx, config tfsdk.Config, writeOnlyAttributesAllowed bool) diag.Diagnostics. Each loop becomes a call with a 6-line closure that creates the fresh response and calls the concept's method; each tail becomes `resp.Diagnostics.Append(validateConfigSchema(ctx, *req.Config, false)...)`. Draft: drafts/B2/drafts/validate. The proto5server/proto6server callers only need the unchanged ValidateXConfig(ctx, *Request, *Response) signatures and request structs, which stay.
- Why equivalent: ConfigValidators(ctx) is evaluated once before the loop as before; Description(ctx) is still called twice per validator around the call; a fresh response per validator; diagnostics are appended to resp.Diagnostics after each validator (not accumulated locally, which would change the order of Diagnostic.Equal calls in Append's de-duplication); SchemaValidate gets the same schema value (config.Schema of the copied config equals req.Config.Schema) and the same request. The trace name is captured from the original text by the rewrite (Calling and Called names asserted equal).
- Size: +93/-263 lines, 8 files (measured on draft)
- Risk: low
- Proof: Draft builds, vet and project lint clean; fwserver, proto5server, proto6server tests pass (the 7 TestServerValidate*Config tables cover validators, ValidateConfig and schema validation). Validator trace lines are not pinned: add a tfsdklogtest case for one concept first.
- Composes with B24: together the seven handlers shrink to about 35-45 lines each.
- A fully generic ValidateConfig handler was not pursued: request types, configure data and ValidateConfig method names differ per concept and would need 5+ closures per call.
- Coordinate with B37: if B37 lands, ValidateActionConfig must pass true (not false) to validateConfigSchema, and the helper's doc comment ('must be false for all other schema types') must say resources and actions.

### B35. TestServerListResource compares diagnostics with a custom comparer that ignores Detail and length, and declares an expectedError that is never checked

- Scope: internal/fwserver/server_listresource_test.go | Kind: test | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/fwserver/server_listresource_test.go:298-307 (cmp.Comparer over diag.Diagnostics), :78 and :185 (expectedError field and value)
- Current: The comparer loops over a only, compares Severity and Summary, ignores Detail, returns true when b is longer and indexes past b when a is longer (go-cmp then panics with 'non-symmetric function' only when its random symmetry check fires). expectedError: "config cannot be nil" is set but never read.
- Replacement: Delete the comparer (cmp uses diag.Diagnostics.Equal, which compares severity, summary and detail) and the expectedError field and value.
- Why equivalent: Test-only strengthening; production unchanged.
- Size: about -12 lines (estimated)
- Risk: low
- Proof: Scratch copy: with the comparer removed all cases still pass; mutating the expected detail 'This is a test warning diagnostic' passes with the current comparer and fails without it (drafts/B2/server_listresource_test.go.orig is the saved original). Probe of the comparer: a 1-diag actual vs a 2-diag expected with a different detail compares equal one way and panics the other.

### B8. Optional after B7: one per-element helper for the List/Set/Map nesting helpers

- Scope: 1 file (attribute_plan_modification.go after B7) | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED, medium confidence
- Where: the element loops of listNestedPlanModify, setNestedPlanModify and mapNestedPlanModify (today internal/fwserver/attribute_plan_modification.go:183-267, 334-418, 485-569 and their block copies at internal/fwserver/block_plan_modification.go:97-181, 248-332)
- Current: The ~85-line element body (config element, plan object, valuable, typable, state element, ObjectRequest, nested call, coerce, ValueFromObject, merge) is identical in the three arms except the path step and the element lookup.
- Replacement: nestedObjectElementPlanModify(ctx, nestedObjectPlanModify, req, resp, attrPath, planElem, configObject, stateObject func() (types.Object, diag.Diagnostics)) (attr.Value, bool); each loop keeps its path step and config lookup, passes the state lookup as a closure and assigns the returned element.
- Why equivalent: The config lookup stays first in the loop and the state-lookup closure runs at its original point (after the plan coercions), so the same diagnostic is reported when several steps fail; the element assignment moves after the merges, which is unobservable because planElements is a local copy (List/Set/MapValue.Elements return copies).
- Size: +84/-189 (measured): net -105
- Risk: low-medium: an 8-parameter helper; moderate value
- Proof: The B7 test set passes on the draft.
- Pre-computing the state lookup would be simpler but changes which diagnostic is reported when both a plan coercion and the state lookup fail (rejected).

### B9. NestedAttributeObjectPlanModify / NestedBlockObjectPlanModify: share the object-plan-modifier loop and the per-child loop body

- Scope: 2 files in internal/fwserver | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwserver/attribute_plan_modification.go:2608-2720; internal/fwserver/block_plan_modification.go:931-1082 (dupl attribute_plan_modification.go:2665-2713 vs block_plan_modification.go:977-1025 and 1027-1075)
- Current: Two identical copies of the object plan modifier loop (~43 lines) and three identical copies of the per-child loop (nested attributes in both functions, nested blocks in the block one; ~48 lines each).
- Replacement: runObjectPlanModifiers(ctx, modifiers []planmodifier.Object, req *planmodifier.ObjectRequest, resp) bool and nestedPlanModify(ctx, name, modifyPlan func(context.Context, ModifyAttributePlanRequest, *ModifyAttributePlanResponse), req, resp) (attr.Value, bool); the callers keep the type assertion, the #993 null/unknown guard (attribute version only) and the final types.ObjectValue.
- Why equivalent: Statements move unchanged; req is passed by pointer so the req.PlanValue updates stay visible to the caller as before; ObjectPlanModifiers() is evaluated once at the same point (loop start); the newPlanValueAttributes assignment moves after the merges, unobservable because the map is a copy from ObjectValue.Attributes(). The block version still has no #993 guard.
- Size: attribute file +105/-76, block file +12/-123 (measured): net -82
- Risk: low. Allocation note: nestedAttrResp is passed through the modifyPlan func value and escapes, costing +1 heap allocation per nested child attribute/block (nested fixture: +43 allocs/op for 43 children). Passing the attribute or block and calling AttributeModifyPlan/BlockModifyPlan directly would keep parity (not drafted).
- Proof: TestNestedAttributeObjectPlanModify, TestNestedBlockObjectPlanModify, TestAttributeModifyPlan and TestBlockModifyPlan pass on the draft.
- Adding the #993 guard to the block version while deduplicating would be a behaviour change; keep it out of this PR.

### B10. SchemaModifyPlan and SchemaSemanticEquality: one per-root helper instead of identical attribute and block loops

- Scope: 2 files in internal/fwserver | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwserver/schema_plan_modification.go:84-196; internal/fwserver/schema_semantic_equality.go:55-145
- Current: Each function has an attribute loop and a block loop whose bodies are identical except AttributeModifyPlan vs BlockModifyPlan (and one comment word); dupl leads 84-139/141-196 and 55-99/101-145.
- Replacement: schemaModifyPlanAtRoot(ctx, name, configData, planData, stateData, modifyPlan, req, resp) bool and schemaSemanticEqualityAtRoot(ctx, name, req, resp) bool; the loops call them (with a closure for the plan case) and return when they return false.
- Why equivalent: Bodies move verbatim; each `return` becomes `return false`, the loop continuation becomes `return true`; `var diags` moves into the helper; map iteration and the cumulative HasError checks are unchanged.
- Size: schema_plan_modification.go +53/-85, schema_semantic_equality.go +38/-71 (measured): net -65
- Risk: low. Allocation note: attrResp moves to the heap because modifyPlan is a func value, costing +1 allocation per root attribute/block (nested fixture: +13 allocs/op). Calling AttributeModifyPlan/BlockModifyPlan directly, for example by passing the attribute or block, would avoid it (not drafted).
- Proof: TestSchemaModifyPlan, TestSchemaSemanticEquality and TestServerPlanResourceChange* pass on the draft.
- Land B1 first; afterwards its fix is a single line in the helper.

### B11. Remove the never-read ProviderMeta fields of ModifyAttributePlanRequest and ModifySchemaPlanRequest

- Scope: 3 files in internal/fwserver (one is B2's server_planresourcechange.go) | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwserver/attribute_plan_modification.go:55-56; internal/fwserver/schema_plan_modification.go:29-30, :90, :147; internal/fwserver/server_planresourcechange.go:283-285
- Current: PlanResourceChange copies req.ProviderMeta into ModifySchemaPlanRequest.ProviderMeta, SchemaModifyPlan copies it into each root ModifyAttributePlanRequest.ProviderMeta, and nothing reads that field (planmodifier requests have no ProviderMeta; the nested request builders do not copy it).
- Replacement: Delete both fields and the three writes; the resource-level ModifyPlan path (server_planresourcechange.go:324-326) keeps its own ProviderMeta. Optionally fix the stale doc of ModifyAttributePlanRequest ('supplied as an argument to the Modify function of an attribute's plan modifier(s)'; plan modifiers receive planmodifier.<T>Request).
- Why equivalent: Write-only data: with both fields deleted the package and its tests compile and all fwserver tests pass; neither struct is referenced outside internal/fwserver (grep).
- Size: -12 lines (measured: -3, -5, -4)
- Risk: low: internal package only
- Proof: Deleted in a scratch copy (drafts/B1/draft-providermeta): go build, go vet and go test ./internal/fwserver/ pass.
- Touches 4 lines of B2's server_planresourcechange.go; coordinate.

### B12. Delete the 7 stale 'Clean up this abstraction ... issues/365' TODO paragraphs

- Scope: 7 doc comments in 7 files | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwserver/attribute_plan_modification.go:80-84; internal/fwserver/attribute_validation.go:55-59 and :997-1001; internal/fwserver/block_plan_modification.go:20-24; internal/fwserver/block_validation.go:19-23; internal/fwserver/schema_plan_modification.go:58-62; internal/fwserver/schema_validation.go:40-44
- Current: Each doc comment carries '// TODO: Clean up this abstraction back into an internal Attribute|Block|Schema type method. The extra ... parameter is a carry-over of creating the proto6server package from the tfsdk package ... Reference: .../issues/365'.
- Replacement: Delete the TODO paragraph and its preceding '//' line, keeping the first doc sentence.
- Why equivalent: Comments only.
- Size: about -35 lines (7 x 5 comment lines, counted)
- Risk: low
- Proof: #365 status verified via the public issue page (closed as completed on 2022-09-22, with the quoted maintainer comment). #709 verified as open (reopened on 2023-06-06 after the #711 revert).
- The 14 test TODOs referencing #709 (list/set element alignment) are still valid: #709 is open and the code still aligns elements by index. Leave them.

### B26. Generic schemaCache[S] for the eight RWMutex-protected schema and definition caches

- Scope: internal/fwserver, 6 files + 1 new file | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: caches actionSchemas, dataSourceSchemas, ephemeralResourceSchemas, functionDefinitions, listResourceSchemas, resourceSchemas, resourceIdentitySchemas, statestoreSchemas: fields in server.go:76-83, 99-106, 122-129, 150-157, 184-192, 230-246, 275-282; 8 read sites (RLock/lookup/RUnlock) and 9 write sites (Lock/lazy make/assign/Unlock) in server.go (DataSourceSchema, ResourceSchema, ResourceIdentitySchema x2), server_actions.go:116-118,147-155, server_ephemeralresources.go, server_functions.go:36-38,63-71, server_listresources.go, server_statestore.go
- Current: 16 fields (map + RWMutex, each with a comment) and 17 hand-written lock sequences of 3 and 6-9 lines.
- Replacement: schemaCache[S]{mu sync.RWMutex; schemas map[string]S} with get(typeName) (S, bool) and set(typeName, S) (lazy make inside set). Sites become `x, ok := s.dataSourceSchemas.get(typeName)` and `s.dataSourceSchemas.set(typeName, schemaResp.Schema)`. Draft: drafts/B2/drafts/schemas.
- Why equivalent: Same lock kinds and critical sections (read under RLock, lazy make and assignment under Lock); the stored values are the same expressions (implicit conversion to fwschema.Schema happens at set's parameter instead of at the map assignment); nil identity schemas are still cached as nil.
- Size: +58/-141 lines, 7 files (measured on draft)
- Risk: low
- Proof: Draft builds, vet and project lint clean; fwserver, proto5server, proto6server tests pass. Fold into the B25 PR (same files).
- providerSchema/providerMetaSchema/providerTypeName/resourceBehaviors use plain Mutex with different shapes and stay as they are.

### B27. Generic helpers for the per-concept lookups, GetMetadata lists and GetProviderSchema loops (newInstance, typeNameMetadatas, validatedSchemas)

- Scope: internal/fwserver, 5 files + 1 new file | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: lookups server.go:299-315 (DataSource), :556-572 (Resource); server_actions.go:17-33; server_ephemeralresources.go:17-33; server_statestore.go:17-33. Metadatas server.go:372-386, :701-715; server_actions.go:97-111; server_ephemeralresources.go:98-112; server_statestore.go:97-111; server_listresources.go:118-132. Schemas server.go:435-472, :764-801; server_actions.go:160-197; server_ephemeralresources.go:161-198; server_statestore.go:160-197; server_listresources.go:178-210
- Current: 5 lookups (80 lines), 6 TypeName-only metadata builders (78 lines) and 6 'call Schema, append diags, ValidateImplementation, keep valid' loops (199 lines) that differ only in types, request/response structs and trace texts.
- Replacement: newInstance[T](funcs, diags, typeName, summary, detailFormat string) (T, diag.Diagnostics); typeNameMetadatas[M ~struct{ TypeName string }, F any](funcs) []M (composite literal M{TypeName: n}); validatedSchemas[T, S](ctx, funcs, diags, logKey, callingLog, calledLog, getSchema func(T) (S, func(context.Context) diag.Diagnostics, diag.Diagnostics)). Draft: drafts/B2/drafts/bulk (server_concepts.go 82 lines). ListResourceType, ResourceBehavior, FunctionMetadatas, FunctionDefinitions and ResourceIdentitySchemas keep their own code (different returns or validation).
- Why equivalent: Same summary and %q detail per concept (passed as literals copied from the originals; the five 'Not Found' texts were asserted present before replacement), zero T is the nil interface the originals return, factory() called at the same point; metadata slices built from the same map range (random order, as today) with the same capacity; validatedSchemas keeps the Calling/Called traces with the same key around the Schema call, the same Append order and skip conditions; ValidateImplementation is passed as a method value bound to the same schema value.
- Size: +146/-300 lines, 6 files (measured on draft)
- Risk: low: note for the implementer: the method value schemaResp.Schema.ValidateImplementation is evaluated before the error check; this is equivalent only because every SchemaResponse.Schema is a struct with a value receiver (an interface-typed schema would panic when nil)
- Proof: Draft builds, vet and project lint clean; fwserver/proto5server/proto6server/providerserver tests pass. 'Not Found' text is pinned only for Resource, ListResource and StateStore (grep in *_test.go); add DataSource/EphemeralResource/Action not-found cases first.
- validatedSchemas' three-value callback is the least readable part; dropping it keeps about -60 lines for lookups and metadatas alone.
- Fold into the B25 PR.

### B29. One stateSemanticEquality helper for the semantic-equality step of CreateResource, UpdateResource, ReadResource and ReadDataSource

- Scope: internal/fwserver, 4 handler files | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwserver/server_createresource.go:183-211; server_updateresource.go:213-241; server_readresource.go:212-240; server_readdatasource.go:114-144
- Current: Four copies of a 29-line block: build SchemaSemanticEqualityRequest (prior data with a description, proposed new data from the new state), call SchemaSemanticEquality, append diagnostics, return on error, and replace the new state's Raw with a debug trace when it changed.
- Replacement: func stateSemanticEquality(ctx, priorDescription fwschemadata.DataDescription, priorSchema fwschema.Schema, priorRaw tftypes.Value, newState *tfsdk.State) diag.Diagnostics, which updates newState only when the diagnostics have no error; sites: `resp.Diagnostics.Append(stateSemanticEquality(ctx, fwschemadata.DataDescriptionPlan, req.PlannedState.Schema, req.PlannedState.Raw, resp.NewState)...)` plus the existing HasError return (ReadDataSource: the call is the last statement). Draft: drafts/B2/drafts/handlers_b (diff against drafts/handlers_a, which holds only B31).
- Why equivalent: At each site resp.Diagnostics holds no error (every handler returned on errors earlier), so resp.Diagnostics.HasError() after the Append is equivalent to the semantic-equality diagnostics having an error, the condition the helper uses before touching newState. The only reordering is between appending warnings and updating newState.Raw plus its debug trace, two independent effects. ReadDataSource's `if Equal { return }; trace; set` equals `if !Equal { trace; set }` at the end of the function.
- Size: +36/-103 lines, 4 files (measured on draft)
- Risk: low
- Proof: Draft builds, vet and project lint clean; tests pass; semantic equality is exercised in each handler's tests (grep -ci semantic: 27, 31, 23, 25 lines).
- Handler sizes with B24, B29 and B30 combined (estimated): ReadResource 209->~165, UpdateResource 208->~165, CreateResource 181->~140.

### B30. nullIdentity and copyIdentity helpers for the 7 null-identity and 13 identity-copy literals in the resource handlers

- Scope: internal/fwserver, 6 handler files | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwserver/server_createresource.go:103-124; server_deleteresource.go:102-121,137-144; server_importresourcestate.go:151-171; server_planresourcechange.go:121-137,335-344; server_readresource.go:133-153; server_updateresource.go:126-147
- Current: 7 sites build `nullIdentityTfValue := tftypes.NewValue(schema.Type().TerraformType(ctx), nil)` then `&tfsdk.ResourceIdentity{Schema: schema, Raw: nullIdentityTfValue.Copy()}`; 13 sites build `&tfsdk.ResourceIdentity{Schema: x.Schema, Raw: x.Raw.Copy()}`.
- Replacement: func nullIdentity(ctx, identitySchema fwschema.Schema) *tfsdk.ResourceIdentity and func copyIdentity(identity *tfsdk.ResourceIdentity) *tfsdk.ResourceIdentity. Draft: drafts/B2/drafts/handlers_c (diff against drafts/handlers_b).
- Why equivalent: Same expressions evaluated in the same order; Copy() of a freshly created null value returns NewValue(type, nil), the same value, so dropping that Copy is unobservable; copyIdentity keeps Raw.Copy().
- Size: +36/-99 lines, 6 files (measured on draft)
- Risk: low
- Proof: Draft builds, vet and project lint clean; fwserver, proto5server, proto6server tests pass (identity cases exist in every resource handler test).

### B31. PlanResourceChange: extract the best-effort changed-attribute debug logging (44 lines) into logPlannedValueChanges

- Scope: internal/fwserver/server_planresourcechange.go | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwserver/server_planresourcechange.go:205-248 (inside the nestif lead at :200)
- Current: PlanResourceChange is 359 lines with cognitive complexity 71; phase map in notes-B2.md. The changed-path logging block only reads resp.PlannedState and req.PriorState, ignores diagnostics and emits Debug traces.
- Replacement: `if !req.PriorState.Raw.IsNull() { logPlannedValueChanges(ctx, resp.PlannedState, req.PriorState) }` with the block moved verbatim into the helper. Draft: drafts/B2/drafts/handlers_a.
- Why equivalent: Contiguous block with no returns and no writes outside its locals, called at the same point, so its GetAttribute calls and traces stay before the 'Marking Computed attributes...' trace; same pointers passed.
- Size: +49/-42 lines (measured); PlanResourceChange 359 -> 318 lines, gocognit 71 -> 51, nestif at :200 gone (golangci-lint on the draft)
- Risk: low
- Proof: Draft builds and fwserver tests pass. The logs are not pinned by tests.
- Other phases are not purely mechanical to extract (they append to resp.Diagnostics and return); see the phase map in notes-B2.md.

### B32. attr_type.go: twelve identical coerce<Type>Typable functions become one generic coerceTypable with twelve named instantiations

- Scope: internal/fwserver/attr_type.go (1 file; 27 call sites in B1's files unchanged) | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwserver/attr_type.go:14-192; callers attribute_plan_modification.go and block_plan_modification.go (27 calls, area B (plan modification and validation))
- Current: The file is a header plus 12 instances of one template (verified by regenerating the file from the template: identical).
- Replacement: func coerceTypable[V attr.Value, T attr.Type](ctx, schemaPath, valuable V) (T, diag.Diagnostics) with the same comment, plus `var ( coerceBoolTypable = coerceTypable[basetypes.BoolValuable, basetypes.BoolTypable]; ... )`. Draft: drafts/B2/drafts/attrtype.
- Why equivalent: Type assertion to the type parameter behaves as the concrete assertion; the zero T is the nil interface the originals return; valuable keeps its dynamic type when passed as attr.Value, so %T in the diagnostic is unchanged; the named vars have the exact function types, so call sites and B1's method-expression spec tables still compile.
- Size: +22/-169 lines, 1 file (measured on draft)
- Risk: low
- Proof: Draft builds, vet and project lint clean, fwserver and proto6server tests pass. The error branch ('Unexpected Attribute Plan Modifier Type Conversion Error') has no test (grep in *_test.go: 0): add one with a Valuable whose Type() is not the Typable.
- Coordinate with B1 item 1 (generic AttributePlanModify), which references coerce<Type>Typable as function values; both changes compose.

### B33. GenerateResourceConfig: one local addPathError closure for the three identical 'Generate Resource Config Error' blocks

- Scope: internal/fwserver/server_generateresourceconfig.go | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwserver/server_generateresourceconfig.go:81-91, 127-137, 150-160
- Current: Three copies of: convert the tftypes path with fromtftypes.AttributePath, append its diagnostics, add the same attribute error with err.Error().
- Replacement: `addPathError := func(path *tftypes.AttributePath, err error) {...}` declared before the Transform call; each site becomes `addPathError(path, err)` followed by its existing return. Draft: drafts/B2/drafts/handlers (diff against drafts/handlers_c).
- Why equivalent: Same calls with the same arguments in the same order; the closure captures the same diags variable.
- Size: +18/-30 lines (measured); gocognit stays 47 (the nesting comes from the if/else structure)
- Risk: low
- Proof: Draft builds; TestServerGenerateResourceConfig passes.
- The RPC is currently never called by Terraform because the capability is not advertised (area A, A1).

### B34. Dead code: three request fields that are set but never read, and the unreferenced exported var fwserver.NoListResults

- Scope: internal/fwserver (4 declarations), their test literals, fromproto5/6 (4 assignments) | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwserver/server_closeephemeralresource.go:20; server_renewephemeralresource.go:21; server_createresource.go:24 (written only at server_applyresourcechange.go:53); server_listresource.go:85; writes to drop at fromproto5/6 closeephemeralresource.go:42 and renewephemeralresource.go:42 (not :40); internal/fwserver/server_upgraderesourceidentity.go:24-26 (UpgradeResourceIdentityRequest.TypeName, never written or read; from A18)
- Current: A go/types scan over the whole module, tests included (drafts/B2/fieldscan), finds these fields with 0 reads (keys in composite literals only: 9, 12 and 1; verifier rename-and-compile agrees). NoListResults has 0 references (the code uses list.NoListResults). CreateResource never reads PlannedPrivate (resource.CreateResponse documents that no private state is passed to Create).
- Replacement: Delete the 3 fields and the var; drop the keys from server_applyresourcechange.go:53 and fromproto5/6 closeephemeralresource.go:42 / renewephemeralresource.go:42 and the 17 test literal lines; also delete the now-unused internal/fwschema imports in server_closeephemeralresource.go and server_renewephemeralresource.go, and the now-unused `testSchema` variables and ephemeral/schema imports in server_closeephemeralresource_test.go and server_renewephemeralresource_test.go; run gofmt (struct and literal re-alignment). fromproto keeps its schema nil check. Fold in the two server.go comment fixes and A18 (UpgradeResourceIdentityRequest.TypeName, server_upgraderesourceidentity.go:24-26).
- Why equivalent: Nothing reads the fields or the var; removing them changes no value, log or call.
- Size: +8/-61 lines in 15 files (measured on a verifier copy, gofmt applied); with A18 +8/-64 in 16 files
- Risk: low: internal package
- Proof: fieldscan output (reads=0 for the 3 fields); grep -rn NoListResults --include=*.go . -> only list/ and the fwserver declaration; go build after deletion is the check.
- Area A filed UpgradeResourceIdentityRequest.TypeName as A18 (fieldscan here also shows reads=0 writes=0 keys=0); land it in the same PR.
- Area F also noted NoListResults (notes-F.md:106) and the server.go:47-48 comment (notes-F.md:110).
- Alternative for PlannedPrivate: pass planned private data to Create (feature change, needs public API).
- Includes area A's A18: UpgradeResourceIdentityRequest.TypeName is never written or read. Both auditors confirmed it.
- The area F verifier also independently found that NoListResults has zero references.

### B36. Replace `*new(time.Time)` with the zero value in two ephemeral resource tests (gocritic newDeref leads)

- Scope: internal/fwserver/server_openephemeralresource_test.go, server_renewephemeralresource_test.go | Kind: test | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwserver/server_openephemeralresource_test.go:223; internal/fwserver/server_renewephemeralresource_test.go:197
- Current: RenewAt: *new(time.Time),
- Replacement: Drop the field (zero value) or write `RenewAt: time.Time{},`.
- Why equivalent: *new(time.Time) is the zero time.Time.
- Size: 2 lines
- Risk: low
- Proof: gocritic newDeref leads; read both sites.
- Same pattern at internal/toproto5/openephemeralresource_test.go:93, renewephemeralresource_test.go:43 and the toproto6 twins (other area); fold all 6 into one change if convenient.

## C. Schema data, reflection and schema internals

### C10. internal/reflect: replace 23 copies of the ValidateableAttribute / TypeWithValidate validation block with one helper

- Scope: internal/reflect (interfaces.go, map.go, number.go, pointer.go, primitive.go, slice.go, struct.go) | Kind: refactor | Tier: 1 (do first) | Verdict: CONFIRMED
- Where: internal/reflect/interfaces.go:83,124,217,258,349,392; map.go:118,178,226; number.go:205,251,297,343,390; pointer.go:108; primitive.go:77,123; slice.go:200,262,352,409; struct.go:276,325 (each a 26-line `switch t := X.(type) { case xattr.ValidateableAttribute: ... default: //nolint:staticcheck ... }`); interfaces.go:458-488 (FromAttributeValue) is the one variant and stays inline
- Current: 24 blocks, one per From* exit path. Extracted to drafts/C/validate-blocks and normalised over 5 per-site names (switched value res/attrVal/val/num/str/b/ret, path path/valPath/mapKeyPath, type typ/elemType/elemAttrType/attrTypes[name], Validate argument tfVal/tfNum/tfStr/tfBool/tfObjVal/val, early-return value nil/target): 23 are byte-identical (md5 e9aac39a), FromAttributeValue differs only by computing tfVal lazily inside the TypeWithValidate branch (ToTerraformValue + toTerraformValueErrorDiag). Each block carries its own //nolint:staticcheck; these are 24 of the 27 SA1019 leads in area C.
- Replacement: func validateAttributeValue(ctx, typ attr.Type, val attr.Value, tfVal tftypes.Value, path path.Path) diag.Diagnostics { switch t := val.(type) { case xattr.ValidateableAttribute: resp := xattr.ValidateAttributeResponse{}; t.ValidateAttribute(ctx, xattr.ValidateAttributeRequest{Path: path}, &resp); return resp.Diagnostics; default: //nolint:staticcheck; if v, ok := typ.(xattr.TypeWithValidate); ok { return v.Validate(ctx, tfVal, path) } }; return nil }, and at each site `diags.Append(validateAttributeValue(ctx, TYPE, VALUE, TFVAL, PATH)...); if diags.HasError() { return RET, diags }`. Draft (drafts/C/draft-validate-helper.diff) was produced mechanically by drafts/C/scripts/apply_helper.py, which asserts one return value per block.
- Why equivalent: Same dispatch (value type switch first, then the deprecated type interface), same arguments, same single call to ValidateAttribute/Validate, diagnostics appended with the same diag.Diagnostics.Append (same de-duplication), same early-return values. The only structural difference is that the HasError check now also runs when neither interface applies; this is unobservable because at all 23 sites diags holds no error on entry (every earlier error returns immediately: checked per site, including the loop sites map.go:178, slice.go:262/352, struct.go:276 where the preceding FromValue result is HasError-checked). ValueFromTerraform and ToTerraformValue call counts are unchanged (the helper calls neither). FromAttributeValue stays inline because using the helper there would add an eager ToTerraformValue call. Allocation counts on the struct benchmarks are unchanged (helper adds no allocation).
- Size: about -560/+113 lines (measured, includes the new 42-line validate.go); internal/reflect non-test lines 2858 -> about 2410; //nolint:staticcheck in internal/reflect 24 -> 2; gocognit FromSlice 89 -> 46, FromMap 44 -> 17, FromStruct 49 -> 32, FromUnknownable/FromNullable 25 -> under 15
- Risk: low: mechanical, in an internal package; the equivalence argument relies on the no-error-on-entry invariant, which a reviewer can check per site
- Proof: Pinned by TestFromSlice, TestFromMap, TestFromStruct_*, TestFromInt/Uint/Float/BigFloat/BigInt, TestFromString/TestFromBool, TestFromPointer, TestFromUnknownable, TestFromNullable, TestFromValueCreator, TestNewAttributeValue (200+ *WithValidate* cases across interfaces/map/number/pointer/primitive/slice/struct _test.go). Coverage run (go test -coverprofile ./internal/reflect/): every arm of every block is executed except the early returns of the 4 loop re-validations (map.go:191-193,199-201; slice.go:275-277,283-285,365-367,373-375; struct.go:289-291,297-299), which are unreachable for deterministic validators (see C3). Ran on the draft: go vet and go test for internal/reflect, internal/fwschemadata, tfsdk, types/basetypes, function, internal/fwserver, internal/proto6server all pass; golangci-lint with the project config on internal/reflect and internal/fwschemadata: 0 issues.
- Update the 'before reaching the switch statement below' comments (slice.go:243-245, 337-339, map.go:163-165, struct.go:261-263) when applying.
- Keep the FromUnknownable/FromNullable `return res, nil` lines as they are in this PR; changing them is bug C6.
- The 3 similar blocks in internal/fwschemadata (data_value.go:92, data_set_at_path.go:118, :269) log around the calls and one logs a different message, so they are not folded into this helper (see notes-C.md).

### C11. isValidFieldName compiles its regexp on every call: hoist it to a package variable (about 3x faster struct Get, 2.5x faster Set; 37 fewer allocations per struct field)

- Scope: internal/reflect/helpers.go | Kind: refactor | Tier: 1 (do first) | Verdict: CONFIRMED
- Where: internal/reflect/helpers.go:135-138 (`re := regexp.MustCompile("^[a-z][a-z0-9_]*$")` inside isValidFieldName, called once per struct field by getStructTags:117, which runs on every Struct/FromStruct call, i.e. every Get/Set/GetAttribute/SetAttribute with a struct model and every struct element of a list/set/map)
- Current: Each call compiles the pattern: 4.3 us and 37 allocations per field name.
- Replacement: var validFieldNameRegex = regexp.MustCompile("^[a-z][a-z0-9_]*$") at package level; isValidFieldName returns validFieldNameRegex.MatchString(name). (fwschema already does this for ValidAttributeNameRegex.)
- Why equivalent: Same pattern, same MatchString result; *regexp.Regexp is safe for concurrent use; MustCompile of this constant pattern cannot panic, so moving it to package initialisation changes no behaviour.
- Size: -2/+5 lines (measured: drafts/C/draft-regexp-hoist.diff)
- Risk: low
- Proof: Pinned by TestIsValidFieldName, TestGetStructTags, TestNewStruct_errors (invalid-tag messages). Benchmarks in scratch (drafts/C/repros/reflect/zz_bench_into_test.go and an internal benchmark): isValidFieldName 4277 ns/37 allocs -> 425 ns/0 allocs; getStructTags on a 10-field struct 38.6 us/405 allocs -> 5.6 us/35 allocs; reflect.Into of a 10-field struct 542 -> 172 allocs and FromValue of it 636 -> 266 allocs (timings 3-4x faster, noisy on the shared machine; allocation counts are exact). go test ./internal/reflect/ passes on the draft.
- Combine with C4 if the tag rule is aligned with fwschema.ValidAttributeNameRegex: the fix then reuses that package variable.
- C4 must duplicate the pattern rather than reuse fwschema.ValidAttributeNameRegex: internal/reflect cannot import internal/fwschema (import cycle via types/basetypes).

### C12. reflect.Number formats the rounding-error diagnostic eagerly on every conversion: build it only on failure

- Scope: internal/reflect/number.go | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/reflect/number.go:43-48 (roundingError := fmt.Errorf(... result.String() ...) and roundingErrorDiag := diag.NewAttributeErrorDiagnostic(...) before the type switch); 19 uses at :56,66,71,74,79,82,87,90,95,98,108,113,118,123,128,140,150,158,170
- Current: Every Into of a Go int/uint/float/*big.Int target formats the big.Float to a string and builds an error diagnostic that is discarded on success: 990 ns, 440 B, 14 allocations per numeric field.
- Replacement: roundingErrorDiag := func() diag.Diagnostic { roundingError := fmt.Errorf("cannot store %s in %s", result.String(), target.Type()); return diag.NewAttributeErrorDiagnostic(path, "Value Conversion Error", "...\n\n"+roundingError.Error()) } and roundingErrorDiag() at the 19 uses.
- Why equivalent: result and target are not modified between line 43 and any use (result.Int, Int64, Uint64, Float64, Float32 and Text are read-only; big.NewFloat makes a new value), so the lazily built text is byte-identical; the appended value has the same dynamic type (diag.DiagnosticWithPath). Success paths no longer allocate it.
- Size: -25/+30 lines (measured: drafts/C/draft-lazy-rounding.diff)
- Risk: low
- Proof: 29 test cases in internal/reflect/number_test.go pin the 'cannot store ... in ...' text (TestNumber_*OverflowError/UnderflowError, TestNumber_bigIntRoundingError); all pass on the draft. Benchmark (scratch BenchmarkNumberInt64): 990.5 ns/440 B/14 allocs -> 147.2 ns/72 B/4 allocs.
- Independent of C1; if C1 lands first its Convert calls do not touch these lines.

### C13. internal/reflect: FromInt, FromUint, FromFloat, FromBigFloat, FromBigInt, FromString and FromBool share one body

- Scope: internal/reflect/number.go, primitive.go | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/reflect/number.go:192-418 (5 functions), primitive.go:64-151 (2 functions); dupl leads number.go:192,238,284 and primitive.go:64,110
- Current: Seven ~42-line functions that differ only in the tftypes type (Number, String, Bool) and in FromBigInt converting with big.NewFloat(0).SetInt(val) first: ValidateValue, NewValue, ValueFromTerraform, validation block, return.
- Replacement: func fromPrimitive(ctx, typ attr.Type, tfType tftypes.Type, val any, path path.Path) (attr.Value, diag.Diagnostics) with the shared body (using C10's helper); each exported function becomes a one-line call, FromBigInt passes big.NewFloat(0).SetInt(val).
- Why equivalent: tftypes.ValidateValue and tftypes.NewValue already take interface{}; boxing val at the call site produces the same dynamic value (int64, uint64, float64, *big.Float, string, bool). Same call order and counts, same diagnostics constructors, same returns. FromBigInt evaluates SetInt before anything else in both versions (so a nil *big.Int still panics until C2 is fixed).
- Size: about -120/+17 on top of C10 (measured); number.go 312 -> 221 and primitive.go 108 -> 96 on C10 alone (the author's 317 -> 226 included C12's +5 lines)
- Risk: low
- Proof: Same tests as C10 (TestFromInt, TestFromUint, TestFromFloat, TestFromBigFloat, TestFromBigInt, TestFromString, TestFromBool, TestFromValue_go_types); go test ./internal/reflect/ passes on the stacked draft.
- Depends on C10. FromValueCreator has the same tail but calls typ.TerraformType(ctx) twice; folding it in would change that call count, so it is left out.

### C14. TransformDefaults: collapse the 12 near-identical default-value arms with one helper

- Scope: internal/fwschemadata/data_default.go | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/fwschemadata/data_default.go:102-396 (12 `case fwschema.AttributeWith<T>DefaultValue:` arms inside the tftypes.Transform callback of TransformDefaults, 390 lines, gocognit 112)
- Current: Phases of TransformDefaults: build configData (27-31); Transform callback: skip root (35-37), look up the schema attribute and ignore atomic/block/dynamic-nested paths (39-61), convert the path (63-71), read the config value and skip non-null or non-null-dynamic values (73-100), 12-arm type switch (102-396), wrap a Transform error (401-411). Arms normalised over the type name: Bool, Float32, Float64, Int32, Int64, Number, Object, String and Dynamic are byte-identical (23 lines); List and Set are identical (29 lines: plus an ElementType(ctx) == nil warn-and-skip); Map equals List except one blank line.
- Replacement: Each arm keeps its typed request/response: `defaultValue := a.<T>DefaultValue(); if defaultValue == nil { return tfTypeValue, nil }; resp := defaults.<T>Response{}; defaultValue.Default<T>(ctx, defaults.<T>Request{Path: fwPath}, &resp); return applyDefault(ctx, &diags, fwPath, tfTypeValue, resp.Diagnostics, resp.PlanValue, <false | resp.PlanValue.ElementType(ctx) == nil>)`, and applyDefault does Append, HasError-skip, the no-element-type warn-skip, the trace log and planValue.ToTerraformValue(ctx). A generic helper is not useful: Go cannot reach Path/PlanValue/Diagnostics fields through type parameters.
- Why equivalent: Same calls in the same order: diags.Append(resp.Diagnostics...), resp.Diagnostics.HasError() skip, collection warn-skip, trace log with fmt.Sprintf("%s", planValue) (same String() via the dynamic type), ToTerraformValue. The only reordering is that ElementType(ctx) for List/Map/Set is evaluated before the Append; it is a pure field getter (basetypes ListValue/MapValue/SetValue.ElementType return l.elementType). Log message text, level and fields are unchanged; only the log @caller location moves into the helper (the framework subsystem logs with location, offset 1), as it does for any edit that shifts lines.
- Size: -174/+49 lines (measured: drafts/C/draft-default-helper.diff); file 412 -> 287 lines, TransformDefaults 390 -> 241 lines, gocognit 112 -> 67
- Risk: low: pinned by a large table test; review the 3 collection arms
- Proof: TestDataDefault (internal/fwschemadata/data_default_test.go, 378 default-related lines) executes every arm, including the nil-default, error-diagnostic and missing-element-type branches (coverage: TransformDefaults 96.7%, the uncovered lines are all in the preamble: 48-52, 60, 69-71, 78-80, 91-93). Draft passes go vet and go test for internal/fwschemadata, internal/fwserver and internal/proto6server; golangci-lint project config: 0 issues.
- No test asserts these log lines (grep 'setting attribute .* to default value' in *_test.go: none).

### C15. Semantic equality: the 7 scalar ValueSemanticEquality<Type> functions are identical modulo the type name; share one generic helper

- Scope: internal/fwschemadata/value_semantic_equality_{bool,float32,float64,int32,int64,number,string}.go, value_semantic_equality.go | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/fwschemadata/value_semantic_equality_bool.go:13-54 and the 6 sibling files (54 lines each)
- Current: After replacing the type name the 7 files are byte-identical (md5 5837055f; string differs only through the normalisation of map[string]/String()): assert prior and proposed to <T>ValuableWithSemanticEquals, trace 'Calling...', call <T>SemanticEquals, trace 'Called...', append diags, set NewValue to the prior value if usePriorValue.
- Replacement: func valueSemanticEqualityScalar[V attr.Value](ctx, req, resp, semanticEquals func(ctx context.Context, proposedNewValuable, priorValuable V) (bool, diag.Diagnostics)) with the shared body; each exported function keeps its name and becomes `valueSemanticEqualityScalar(ctx, req, resp, func(ctx context.Context, p, q basetypes.<T>ValuableWithSemanticEquals) (bool, diag.Diagnostics) { return p.<T>SemanticEquals(ctx, q) })`. The closure is needed because <T>SemanticEquals takes <T>Valuable, not the WithSemanticEquals interface, so a method expression does not unify with V.
- Why equivalent: x.(V) with V the same interface type is the same assertion (a nil PriorValue fails it in both versions); the closure calls the same method with the same receiver/argument order; resp.NewValue receives the same dynamic value; log text and fields are identical (only the log @caller location moves into value_semantic_equality.go).
- Size: -280/+74 lines (measured: drafts/C/draft-semeq-helper.diff)
- Risk: low; the only observable difference is the @caller field of 2 TRACE lines per call. Because the 7 functions log identical messages and tf_value_type holds the value's String() (not its type), the per-type file in @caller is currently the only clue to which typed function ran; after the change that clue is gone. State this in the PR.
- Proof: TestValueSemanticEqualityBool/Float32/Float64/Int32/Int64/Number/String and TestValueSemanticEquality pass on the draft, as do internal/fwserver and internal/proto6server tests; golangci-lint project config: 0 issues.
- Do not merge with C16 into one helper: see the rejected rewrite in notes-C.md (the scalar variant applies the prior value even when the provider returned an error diagnostic; the nested variant returns first).

### C16. Semantic equality: the List/Map/Set/Object outer functions are identical modulo the type name; share one helper that takes the nested-element function

- Scope: internal/fwschemadata/value_semantic_equality_{list,map,set,object}.go, value_semantic_equality.go | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/fwschemadata/value_semantic_equality_list.go:15-85 and the same lines of _map.go, _set.go, _object.go (dupl lead value_semantic_equality_map.go:4-212 vs _object.go)
- Current: List, Map and Set outer functions are byte-identical after replacing the type name (71 lines); Object differs only in comments and in calling ValueSemanticEqualityObjectAttributes: fall back to the element/attribute function when either value lacks the interface, otherwise call <T>SemanticEquals, append diags, return on error, use the prior value if signalled, else check the elements.
- Replacement: func valueSemanticEqualityNested[V attr.Value](ctx, req, resp, semanticEquals func(ctx, proposed, prior V) (bool, diag.Diagnostics), nested func(context.Context, ValueSemanticEqualityRequest, *ValueSemanticEqualityResponse)); each outer function becomes a 5-line call passing ValueSemanticEquality<T>Elements / ObjectAttributes. The Elements/Attributes functions stay as they are (their loops differ: index, key, set matching, attribute name).
- Why equivalent: Same three call sites of the nested function, same HasError early return before the usePriorValue check, same trace logs (text and fields; only @caller moves).
- Size: about -256/+81 lines (measured: 4 files -256/+16 in drafts/C/draft-semeq-nested.diff plus the 65-line helper; combined with C15 -536/+155, value_semantic_equality*.go 1400 -> 1019 lines)
- Risk: low
- Proof: TestValueSemanticEqualityList/Map/Set/Object and TestValueSemanticEquality pass on the draft (plus internal/fwserver, internal/proto6server); golangci-lint project config: 0 issues; dupl leads in non-test fwschemadata drop from 2 pairs to the Map/Object Elements pair.
- Map and object element loops range over Go maps, so the order of element SemanticEquals calls (and of their warnings) is random; unchanged by this refactor, noted in notes-C.md.

### C17. internal/reflect idioms: reflect.TypeFor, range over int, len == 0, errors.New

- Scope: internal/reflect (into.go, number.go, generic_attr_value.go, helpers.go, pointer.go, slice.go) | Kind: modernization | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: Owned here: reflect.TypeOf(big.NewFloat(0)) / reflect.TypeOf(big.NewInt(0)) at into.go:167 and number.go:51,53; for i := 0; i < N; i++ at helpers.go:71, pointer.go:68, slice.go:235,334; len(...) <= 0 at slice.go:107,298; fmt.Errorf("invalid target") at into.go:61. The TypeOf((*X)(nil)).Elem() sites into.go:70,76,80,95 and generic_attr_value.go:14 belong to the coordinator's repo-wide go fix -reflecttypefor item.
- Current: Pre-1.22 idioms.
- Replacement: reflect.TypeFor[attr.Value]() / [tftypes.ValueConverter] / [Unknownable] / [Nullable] / [*attr.Value] / [*big.Float] / [*big.Int]; for i := range typ.NumField(), for range pointers, for i := range val.Len(); len(...) == 0; errors.New("invalid target").
- Why equivalent: TypeFor[T]() returns the identical reflect.Type (probe drafts/C/probe-typefor TestSame: equal for an interface type and for *big.Float/*big.Int). NumField(), pointers and val.Len() are loop-invariant (in FromSlice the body re-declares val, but the loop bound always referred to the outer reflect.Value). len never returns a negative number. errors.New gives the same message (no format verbs). No allocation change: the compiler already avoids allocating in reflect.TypeOf(big.NewFloat(0)) (probe benchmark: 0 allocs either way), so this is readability only.
- Size: -10/+11 lines without the 5 coordinator-owned TypeFor lines (whole draft: -15/+16, measured)
- Risk: low
- Proof: go vet and go test ./internal/reflect/ pass on the draft. The 11 test-file reflecttypefor leads in this area (fwschemadata data_get_at_path_test.go:102,136,137, data_get_test.go:74,117,118; reflect into_test.go:102,130,146, pointer_test.go:31,32) all pass concrete static types and are equally safe to convert.
- Fold into any PR that touches these files (C10 touches all of them).
- Its reflect.TypeFor part is the same rewrite as M2, which covers the whole repo. Land whichever comes first and drop the overlap from the other.

### C18. Membership loops -> slices.Contains / slices.ContainsFunc (fwschema, fwtype, fwschemadata)

- Scope: internal/fwschema/attribute_name_validation.go, internal/fwtype/*.go, internal/fwschemadata/tftypes_value.go | Kind: modernization | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwschema/attribute_name_validation.go:76-91,109-124 (go fix slicescontains); internal/fwtype/missing_underlying_type_validation.go:40-46 (go fix slicescontains -> ContainsFunc); internal/fwtype/static_collection_validation.go:33-39,66-72 and internal/fwschemadata/tftypes_value.go:192-204 (same shape, not flagged)
- Current: Hand-written 'found' loops with break or early return.
- Replacement: if slices.Contains(ReservedProviderAttributeNames, name) { diags.AddError(...) } (same for resource names); return slices.ContainsFunc(attrType.ElemTypes, ContainsMissingUnderlyingType); return slices.ContainsFunc(attrType.ElementTypes(), ContainsCollectionWithDynamic) / containsDynamic; if !slices.ContainsFunc(parentElems, func(e tftypes.Value) bool { return e.Equal(childValue) }) { parentElems = append(parentElems, childValue) }.
- Why equivalent: Same element order, same short-circuit on the first match, same nil/empty result (false). The predicates are pure (string ==, recursive type switches, tftypes.Value.Equal with the original receiver/argument order). The reserved-name diagnostic used `name`, which equals the matched element, so its text is unchanged; ElementTypes() is still evaluated exactly once.
- Size: -61/+30 lines (measured: drafts/C/draft-slices-contains.diff; run gofmt for the import order)
- Risk: low
- Proof: TestIsReservedProviderAttributeName, TestIsReservedResourceAttributeName, TestContainsMissingUnderlyingType, TestTypeContainsCollectionWithDynamic, TestUpsertChildTerraformValue; go test ./internal/fwschema/ ./internal/fwtype/ ./internal/fwschemadata/ pass on the draft.
- The ObjectType/AttributeTypes loops in the same fwtype functions range over maps and stay as loops.

### C19. fwschema Equal helpers -> maps.EqualFunc

- Scope: internal/fwschema/underlying_attributes.go, nested_attribute_object.go, nested_block_object.go | Kind: modernization | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwschema/underlying_attributes.go:39-57 (UnderlyingAttributes.Equal), nested_attribute_object.go:55-77 (NestedAttributeObjectEqual), nested_block_object.go:64-102 (NestedBlockObjectEqual, attribute and block loops)
- Current: len check plus a range loop that looks each key up in the other map and calls Equal; the loops call b.GetAttributes()/b.GetBlocks() once per key, and the framework schema types rebuild a new map on every GetAttributes()/GetBlocks() call (resource/schema/schema.go:179-187 schemaAttributes).
- Replacement: maps.EqualFunc(u, o, Attribute.Equal); maps.EqualFunc(a.GetAttributes(), b.GetAttributes(), Attribute.Equal) and maps.EqualFunc(a.GetBlocks(), b.GetBlocks(), Block.Equal) after the existing Type().Equal check.
- Why equivalent: maps.EqualFunc performs exactly the len check, key lookup and eq(v1, v2) = v1.Equal(v2) per key with short-circuit. The only difference is fewer calls to GetAttributes()/GetBlocks(), which are pure map-building getters for every implementation (fwschema.NestedBlockObject/NestedAttributeObject can only be implemented inside the framework because their methods return internal types).
- Size: -64/+7 lines (measured: drafts/C/draft-fwschema-mapsequal.diff, including 3 import lines)
- Risk: low
- Proof: go test ./internal/fwschema/ ./resource/schema/ ./datasource/schema/ ./provider/schema/ pass on the draft (their *_Equal tests exercise these helpers through every schema type).

### C20. fwschema: drop the degenerate `switch nestingMode { default: ... }` blocks (commented-out cases) in ValidateAttributeImplementation / ValidateBlockImplementation

- Scope: internal/fwschema/attribute_validate_implementation.go, block_validate_implementation.go | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/fwschema/attribute_validate_implementation.go:62,77-89; internal/fwschema/block_validate_implementation.go:56,71-81,104-114 (golangci exhaustive leads at :77, :71, :104)
- Current: Each loop declares `var p path.Path`, then switches on a nestingMode computed only for this purpose, with every case commented out and a default that does `p = req.Path.AtName(name)` 'purely to preserve the prior logic'.
- Replacement: As proposed, and also delete the now-unused "github.com/hashicorp/terraform-plugin-framework/path" import in both files and run gofmt (the measured draft, drafts/C/draft-fwschema-switch.diff = drafts/C/fwschema-draft/*.go, does not compile).
- Why equivalent: The switch always takes the default branch. GetNestingMode() is a pure getter on framework schema types (fwschema.NestedAttribute and Block cannot be implemented outside the framework because GetNestingMode returns an internal type), so removing the call changes nothing observable.
- Size: -56/+3 lines (measured on a compiling draft, drafts/V-C/c20)
- Risk: low: maintainers may prefer to keep the commented-out cases as design notes; issue #574 holds that context
- Proof: Exercised by every schema package's TestSchemaValidateImplementation and nested attribute/block ValidateImplementation tests; no behaviour to pin.

## D. Public schema packages

### D7. Misleading doc comments across the schema packages (wrong package doc for provider/schema, wrong types and method names, nonexistent identifiers)

- Scope: about 38 files, comments only | Kind: cleanup | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: as cited, plus resource/schema/defaults/doc.go:32 ('desired custom type in a plan modifier.' in the defaults package)
- Current: Godoc text that names the wrong package/type/method. The identifiers StringPlanModifier, StringValidator and StringDefault do not exist (the interfaces are planmodifier.String, validator.String, defaults.String).
- Replacement: Fix each comment: provider/schema/doc.go -> 'Package schema contains all available schema functionality for providers. Provider schemas define the structure and value types for provider configuration data. Schemas are implemented via the provider.Provider type Schema method.'; types.BoolType; 'ObjectType or CustomType'; 'AttributeAtTerraformPath returns the Attribute at the passed tftypes path ...'; 'GetBlocks returns the Blocks field value.'; 'IsSensitive always returns false as action schema attributes cannot be Sensitive.' (wording used by the 11 sibling IsSensitive docs in action/schema); list all concept packages in schema/doc.go; use the real interface names and fix the typos. Draft: drafts/D/diffs/D-docs.diff (drafts/D/rewrite_docs.py).
- Why equivalent: Comments only; no code, signature, string literal or diagnostic changes. go build output is byte-identical.
- Size: about -65/+67 lines in 38 files, comments only (draft -64/+66 plus the defaults/doc.go:32 line)
- Risk: low: documentation only
- Proof: Counts: grep -l 'GetType returns types.StringType' */schema/bool_attribute.go -> 9 (string_attribute.go legitimately has it x9); grep -l 'GetType returns ListType of ObjectType' */*/single_nested_attribute.go -> 8; 'AttributeAtTerraformPath' doc mismatch -> 9 and 'GetBlocks' -> 6 (also reported by staticcheck ST1020 in staticcheck-all.txt, out of the project's lint scope); grep 'type StringPlanModifier\|type StringValidator\|type StringDefault' -> 0 results. A sweep of all 1613 single-statement getters found no other getter whose doc names a field it does not return.
- Highest-visibility item is provider/schema/doc.go: pkg.go.dev shows the data source description for the provider schema package.
- ST1020/ST1021 style hits that are only about form (e.g. NestedAttribute type comments) are not included.

### D6. resource/schema SingleNestedAttribute.ValidateImplementation appends the non-computed-default diagnostic twice; the inner append is dead

- Scope: resource/schema/single_nested_attribute.go, 4 lines | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: resource/schema/single_nested_attribute.go:321-323 (outer check) and :333-336 (inner duplicate inside 'if a.ObjectDefaultValue() != nil')
- Current: if !a.IsComputed() && a.ObjectDefaultValue() != nil { Append(nonComputedAttributeWithDefaultDiag(req.Path)) } ... if a.ObjectDefaultValue() != nil { if !a.IsComputed() { Append(nonComputedAttributeWithDefaultDiag(req.Path)) } ... }. It is the only file in resource/schema with two calls (grep -c nonComputedAttributeWithDefaultDiag per file).
- Replacement: Delete the inner 'if !a.IsComputed() { ... }' block (lines 334-336 plus the blank line). Do NOT delete the outer block instead: that would move the diagnostic after the write-only diagnostics and change order. Draft: drafts/D/diffs/D-sna.diff.
- Why equivalent: Both conditions read the same fields of the value receiver (IsComputed returns a.Computed, ObjectDefaultValue returns a.Default) and nothing between them can mutate a. The second Append receives an ErrorDiagnostic with the same summary and detail (same req.Path), and diag.Diagnostics.Append skips diagnostics for which Contains (ErrorDiagnostic.Equal compares summary and detail) is true, so it never adds anything. Probe with Default + WriteOnly violation: 2 diagnostics today (default first, then write-only), identical after the change.
- Size: about -4/+0 lines in 1 file (measured on draft)
- Risk: low: dead append removal; order preserved
- Proof: go test -overlay=drafts/D/drafts/D-sna/overlay.json ./resource/schema/ passes; TestSingleNestedAttributeValidateImplementation (resource/schema/single_nested_attribute_test.go:686) pins the single-diagnostic output.

### D8. provider/schema MapNestedAttribute lacks the fwschema.AttributeWithValidateImplementation compile-time assertion that all 6 sibling packages have

- Scope: provider/schema/map_nested_attribute.go var block | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: provider/schema/map_nested_attribute.go:22-25 (method at :253); siblings: {resource,datasource,ephemeral,action,list,statestore}/schema/map_nested_attribute.go var blocks
- Current: var ( _ NestedAttribute = MapNestedAttribute{}; _ fwxschema.AttributeWithMapValidators = MapNestedAttribute{} )
- Replacement: Add '_ fwschema.AttributeWithValidateImplementation = MapNestedAttribute{}' (gofmt realigns the block).
- Why equivalent: Blank-identifier assertion: compile-time only, no runtime code. The method already satisfies the interface (probe: provider schema with a map nested attribute containing a dynamic child gets the MapNestedAttribute's own collection-with-dynamic diagnostic).
- Size: about -2/+3 lines in 1 file (estimated)
- Risk: low: compile-time check only
- Proof: Cross-package assertion matrix (notes-D.md) shows this is the only file where a ValidateImplementation method exists without the assertion; probe drafts/D/probe6 confirms the method runs.
- Fold into D7 or any PR touching the file.

### D9. Replace the 148 per-type Equal method bodies with generic fwschema helpers (type check + existing AttributesEqual/NestedAttributesEqual/BlocksEqual)

- Scope: 148 Equal methods in 148 files across the 9 schema packages, plus one new file in internal/fwschema | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: grep -rn 'func (a [A-Za-z]*) Equal(o fwschema.Attribute) bool' and 'func (b [A-Za-z]*) Equal(o fwschema.Block) bool' in resource/schema datasource/schema provider/schema provider/metaschema ephemeral/schema action/schema list/schema statestore/schema resource/identityschema: 100 x 'if _, ok := o.(T); !ok { return false }; return fwschema.AttributesEqual(a, o)', 31 x 'other, ok := o.(T); if !ok {return false}; return fwschema.NestedAttributesEqual(a, other)', 17 x the block form with fwschema.BlocksEqual
- Current: Each exported type carries a 5-6 statement Equal body that differs from its 147 siblings only in the asserted type name.
- Replacement: internal/fwschema/typed_equal.go: func TypedAttributesEqual[T Attribute](a T, o Attribute) bool { if _, ok := o.(T); !ok { return false }; return AttributesEqual(a, o) }, func TypedNestedAttributesEqual[T NestedAttribute](a T, o Attribute) bool { other, ok := o.(T); if !ok { return false }; return NestedAttributesEqual(a, other) }, func TypedBlocksEqual[T Block](b T, o Block) bool {...}. Each method body becomes 'return fwschema.TypedAttributesEqual(a, o)' (resp. Nested/Blocks). Exported method names, signatures, receivers and doc comments are untouched. Draft: drafts/D/diffs/D-equal-all.diff (generated by drafts/D/rewrite_equal.py).
- Why equivalent: T is inferred from the receiver's static type (e.g. BoolAttribute), so o.(T) is the same concrete-type assertion as o.(BoolAttribute), including for a nil o (ok=false) and for a promoted Equal on a provider type embedding BoolAttribute (receiver is still BoolAttribute). The helpers pass the same dynamic values to the same AttributesEqual/NestedAttributesEqual/BlocksEqual functions (a boxed into the interface exactly as before; 'other' is the asserted copy as before). No struct, field, method set or exported signature changes; only internal/fwschema gains exported-to-the-module helpers. The only observable difference is one extra stack frame if a comparison panics.
- Size: about -802/+148 lines in 148 files plus a 36-line helper file, net about -618 (measured on draft)
- Risk: low: mechanical regex rewrite; touches many files, so split per package (about 16 files / 110 changed lines each) if reviewers prefer
- Proof: go test -overlay=drafts/D/drafts/D-equal-all/overlay.json for all 9 schema packages and ./internal/fwschema/ passes; go vet and gofmt clean on the overlay. 162 Test*Equal functions in the 9 packages pin the behaviour (e.g. TestBoolAttributeEqual).
- Value is boilerplate reduction only; recommend doing it package by package alongside other edits rather than as a standalone 149-file PR.
- internal/testing/testschema has 53 more call sites of the same pattern (area G) that could use the same helpers.
- GetType bodies were measured too and NOT proposed (see notes-D.md: eager evaluation trap).

### D10. Replace the 15 copies of schemaAttributes/schemaBlocks with two generic fwschema conversion helpers

- Scope: 9 schemaAttributes + 6 schemaBlocks definitions and 43 call sites in 33 files, plus one new file in internal/fwschema | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: 15 definitions as cited; 43 call sites in 31 files, plus 3 definition-only files: 34 repo files in total (the item says 33), plus the new internal/fwschema helper
- Current: Each package defines func schemaAttributes(map[string]Attribute) map[string]fwschema.Attribute (and schemaBlocks for blocks): make(len) + copy loop, identical except for the concept-specific element type.
- Replacement: internal/fwschema: func AttributesMap[T Attribute](attributes map[string]T) map[string]Attribute and func BlocksMap[T Block](blocks map[string]T) map[string]Block with the same make+loop body; callers become fwschema.AttributesMap(s.Attributes) etc.; the 15 local functions are deleted. Draft: drafts/D/diffs/D-maps.diff (drafts/D/rewrite_maps.py).
- Why equivalent: Same allocation and copy: make(map, len(in)) so a nil input still yields a non-nil empty map; converting a type-parameter value whose type argument is the concept's Attribute interface to fwschema.Attribute keeps the dynamic value and keeps nil entries nil (probe drafts/D/probe4: nil-entry==nil true, %T unchanged, nil map -> non-nil len 0 in both). GetAttributes on NestedAttributeObject still returns fwschema.UnderlyingAttributes via assignability, as today. Map iteration order was already unspecified.
- Size: about -208/+43 lines in 34 files plus a 27-line helper file, net about -138 (measured on draft)
- Risk: low: unexported functions only; public GetAttributes/GetBlocks signatures unchanged
- Proof: go test -overlay=drafts/D/drafts/D-maps/overlay.json for the 9 schema packages passes; go vet/gofmt clean. Pinned by 38 Test*GetAttributes/Test*GetBlocks tests plus every Schema/NestedObject test that walks attributes.

### D11. Move the 9 copies of the Schema.ValidateImplementation loop into one fwschema helper, completing the existing forward-to-fwschema pattern of schema.go

- Scope: 9 Schema.ValidateImplementation methods (one per schema package) plus one new file in internal/fwschema | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: ValidateImplementation in resource/schema/schema.go:152, datasource/schema/schema.go, provider/schema/schema.go, provider/metaschema/schema.go, ephemeral/schema/schema.go, action/schema/schema.go:131, list/schema/schema.go, statestore/schema/schema.go, resource/identityschema/schema.go
- Current: Four textual variants of one loop: attributes then blocks, each building ValidateImplementationRequest{Name, Path: path.Root(name)}, appending a reserved-name check (IsReservedResourceAttributeName in 7 packages, IsReservedProviderAttributeName in provider/schema, none in provider/metaschema) and ValidateAttributeImplementation/ValidateBlockImplementation. 54 of the other 123 Schema methods already forward to fwschema.Schema* helpers.
- Replacement: internal/fwschema.SchemaValidateImplementation(ctx, s Schema, reservedName func(string, path.Path) diag.Diagnostics) diag.Diagnostics containing the loop (reservedName may be nil); each method body becomes 'return fwschema.SchemaValidateImplementation(ctx, s, fwschema.IsReservedResourceAttributeName)' (provider: IsReservedProviderAttributeName; metaschema: nil). action/schema keeps its TODO:Actions comment above the call. Draft: drafts/D/diffs/D-schemavi.diff.
- Why equivalent: Same calls in the same order per entry (reserved-name check, then implementation validation; attributes before blocks). provider/metaschema (GetBlocks returns nil) and resource/identityschema / statestore (GetBlocks returns an empty map) make the added blocks loop a no-op; metaschema passes nil so no reserved-name check runs, as today. GetAttributes/GetBlocks are the same methods called through the fwschema.Schema interface. Map iteration order was already random; Append's deduplication is order-independent for identical diagnostics.
- Size: about -194/+14 lines in 9 files plus a 47-line helper file, net about -133 (measured on draft)
- Risk: low: internal mechanics only; exported method signature and doc unchanged
- Proof: go test -overlay=drafts/D/drafts/D-schemavi/overlay.json for the 9 schema packages, ./internal/fwschema/ and ./internal/fwserver/ passes; go vet/gofmt clean. Pinned by TestSchemaValidateImplementation in all 9 packages and the fwserver GetProviderSchema tests.
- The two action/schema TODO:Actions comments (godox leads at action/schema/schema.go:140,154) are a pending design question; removing the reserved-name check for actions would change behaviour and is not proposed.

## E. Value types, paths, diagnostics and tfsdk

### E4. SetValue.Equal, MapValue.String and ObjectValue.String copy the whole collection once per element (quadratic allocation)

- Scope: types/basetypes: set_value.go, map_value.go, object_value.go (7 lines) | Kind: refactor | Tier: 1 (do first) | Verdict: CONFIRMED
- Where: types/basetypes/set_value.go:309 (contains: range s.Elements()); types/basetypes/map_value.go:345,346,359 (m.Elements() twice plus once per key); types/basetypes/object_value.go:377,378,391 (o.Attributes() twice plus once per key)
- Current: Elements()/Attributes() return defensive copies. SetValue.contains (called once per element by SetValue.Equal) iterates s.Elements(), copying the n-element slice n times; MapValue.String and ObjectValue.String call m.Elements()/o.Attributes() n+2 times, copying the whole map each time. Measured: SetValue.Equal with 1,000 elements allocates 1,000 times / 16.4 MB per call (3,000: 147 MB); MapValue.String with 1,000 entries takes ~83 ms (3,000: ~680 ms). Equal runs on semantic-equality paths (internal/fwschemadata/value_semantic_equality.go:90, internal/fwserver/schema_semantic_equality.go:90,136) and String runs whenever a set-element path is rendered (PathStepElementKeyValue.String embeds the value's String; 7 FrameworkWithAttributePath(ctx, path.String()) sites).
- Replacement: Index the receiver's own fields: set_value.go:309 `for _, elem := range s.elements {`; map_value.go:345-346 `len(m.elements)` / `range m.elements` and :359 `m.elements[k].String()`; object_value.go:377-378 `len(o.attributes)` / `range o.attributes` and :391 `o.attributes[k].String()`.
- Why equivalent: The copies hold the same keys and the same attr.Value interface values as the fields they copy and are only read, so iterating or indexing the field directly yields the same elements in the same order, the same number of Equal/String calls on each element (one per comparison/key, same receiver/argument order: other.contains(elem) still calls otherElem.Equal(elem)), and the same output. No method signature, nil-vs-empty result or panic changes (a nil element still panics on .String() in both versions).
- Size: about -7/+7 lines in 3 files (measured on draft7)
- Risk: low: no observable change other than allocation/time.
- Proof: Pinned by TestSetValueEqual, TestMapValueString, TestObjectValueString (basetypes tests pass on draft2). Benchmarks in scratch (drafts/E/probe/setalloc vs drafts/E/probe2/setalloc): SetValue.Equal n=1000 1000 allocs/16 MB -> 0 allocs/0 B (~16.4 ms -> ~3.4 ms); MapValue.String n=1000 10,084 allocs ~83 ms -> 5,030 allocs ~0.6 ms. Suggest adding BenchmarkSetValueEqual and BenchmarkMapValueString with b.Loop().
- Land before E9: E9 rewrites the same key-collection lines with slices.Sorted(maps.Keys(m.elements)).
- ListValue.String, SetValue.String and TupleValue.String copy once per call (harmless) and are left alone.

### E5. Unexported generic helpers for the primitive value state machine (ToTerraformValue, Equal, String, Value<T>Pointer)

- Scope: types/basetypes: bool, int32, int64, string, float32, float64 *_value.go + new primitive_value.go | Kind: refactor | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: types/basetypes/{bool,int32,int64,string}_value.go ToTerraformValue/Equal/String/Value<T>Pointer; types/basetypes/{float32,float64}_value.go ToTerraformValue (24 method bodies)
- Current: Each primitive repeats the same null/unknown/known state machine. Normalised AST comparison (drafts/E/twinprobe): ToTerraformValue is identical for 6 types modulo tftypes kind and the type name in the panic; Equal is identical for bool/int32/int64/string; Value<T>Pointer identical for the same 4; String differs only in the format of the known value. 16-line ToTerraformValue, 17-line Equal, 11-line String and 7-line pointer bodies are duplicated.
- Replacement: Add types/basetypes/primitive_value.go with primitiveToTerraformValue(typ tftypes.Type, name string, state attr.ValueState, value any) (tftypes.Value, error), primitiveEqual[T comparable](state, value, otherState, otherValue) bool, primitiveString(state, known func() string) string and primitiveValuePointer[T any](state, value T) *T. Keep every exported method declared on each type with its doc comment; bodies become e.g. `return primitiveToTerraformValue(tftypes.Bool, "Bool", b.state, b.value)` and `o, ok := other.(BoolValue); return ok && primitiveEqual(b.state, b.value, o.state, o.value)`. Number and Dynamic stay as they are (different nil semantics).
- Why equivalent: Struct layouts (unexported state/value fields), zero values (null), exported method sets, value receivers and doc comments are untouched, so reflection, %#v and downstream embedding are unaffected. Equal still asserts the concrete type first, so BoolValue.Equal(Int64Value) stays false. The helper switch is the same switch: the panic text is built with the same format and the type name, giving byte-identical "unhandled Bool state in ToTerraformValue: ..." (verified for an invalid state 3, including the nested %!s(PANIC=String method ...) text). tftypes.ValidateValue/NewValue receive the same boxed value. String checks unknown before null and calls the formatter only for other states, as before. Value<T>Pointer returns a fresh pointer to a copy (nil for null), as &b.value on a value receiver does.
- Size: about -192/+88 lines (measured on a draft: 6 files shrink by 37 (bool/int32/int64), 37 (string), 13 (float32/float64) lines; the new helper file is 70 lines incl. header and docs)
- Risk: low-medium: no API change, but it touches the most-read public files of the type system (many custom types copy basetypes as a template; helpers make them less self-contained). Maintainers may prefer explicitness.
- Proof: Reachable behaviour is already pinned: each of the 6 types has Test<T>ValueToTerraformValue, Test<T>ValueEqual (with cross-type candidates such as NewStringValue), Test<T>ValueString and Test<T>Value<T>Pointer (24 test functions; pass on the draft, plus tfsdk and internal/reflect). The only unpinned branch is the invalid-state panic, unreachable from outside the package (state is unexported and every constructor sets a valid state); a verifier's internal dump of 68 values incl. invalid states showed identical output. The suggested basetypes-internal panic-text test is cheap and nice to have, not a prerequisite.
- Rejected variant: embedding a generic core struct with promoted IsNull/IsUnknown changes selector depth for downstream types that embed basetypes values (possible ambiguous selectors) and changes %#v/reflect layout; see notes.
- Float32/Float64/Number/Dynamic Equal are excluded because their known-with-nil semantics differ (and big.Float needs Cmp).

### E9. mapsloop x4 -> maps.Copy; sort.Strings x3 -> slices.Sorted(maps.Keys(...))

- Scope: types/basetypes: map_value.go, object_type.go, object_value.go | Kind: modernization | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: types/basetypes/map_value.go:177-179 (Elements), :345-349 (String keys); types/basetypes/object_type.go:46-48 (AttributeTypes), :148-153 (String keys); types/basetypes/object_value.go:238-240 (Attributes), :250-252 (AttributeTypes), :377-381 (String keys)
- Current: result := make(map[K]V, len(src)); for k, v := range src { result[k] = v } (4 defensive copies) and keys := make([]string, 0, len(m)); for k := range m { keys = append(keys, k) }; sort.Strings(keys) (3 String methods).
- Replacement: maps.Copy(result, src) after the existing make; keys := slices.Sorted(maps.Keys(m.elements)) (ObjectType.String can range over slices.Sorted(maps.Keys(o.AttrTypes)) directly). Import maps/slices, drop sort.
- Why equivalent: maps.Copy assigns every key/value of src into dst exactly like the loop (it is the same loop); dst is always a fresh non-nil make(...), so a nil src still yields an empty non-nil map. slices.Sorted collects the same unique keys and calls slices.Sort, which is what sort.Strings itself calls in Go 1.26 (sort/sort.go:181), so the order is identical; its nil result for an empty map is unobservable because keys is only ranged over. Do NOT use maps.Clone: it returns nil for a nil source.
- Size: about -31/+13 lines in 3 files when applied after E4 (measured; net -18; imports -sort +maps +slices); E4+E9 together -34/+16 (measured). The earlier '-27/+9' is the arithmetic difference: the 4 key-collection lines E4 edits are edited again by E9, so they count in both diffs.
- Risk: low.
- Proof: Pinned by TestMapValueElements, TestObjectValueAttributes, TestObjectValueAttributeTypes, TestObjectTypeAttributeTypes_immutable, TestMapValueString, TestObjectValueString, TestObjectTypeString (basetypes tests pass on draft2).
- Apply after E4, which changes the same String() lines.
- go fix (mapsloop) produces the maps.Copy part mechanically.

### E10. Five copies of the diagnostics-to-panic-string loop in New{List,Map,Object,Set,Tuple}ValueMust

- Scope: types/basetypes: list/map/object/set/tuple_value.go + one small helper file | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: types/basetypes/list_value.go:135-148; map_value.go:140-153; object_value.go:165-178; set_value.go:135-148; tuple_value.go:92-105; grep -rn '"%s | %s | %s"' --include=*.go . | grep -v _test.go (5)
- Current: Each Must function repeats a 12-line loop formatting each diagnostic as "severity | summary | detail" and joining with "\n", under the comment "This could potentially be added to the diag package."
- Replacement: One unexported func diagnosticsPanicString(diags diag.Diagnostics) string holding the loop; each Must keeps its own prefix: panic("NewListValueMust received error(s): " + diagnosticsPanicString(diags)).
- Why equivalent: Same Sprintf format, same iteration order over diags, same "\n" join and the same per-type prefix (including the inconsistent "MapValueMust"/"ObjectValueMust"/"SetValueMust" vs "NewListValueMust"/"NewTupleValueMust" names, which must not be unified). Panic messages compared byte-for-byte for all five types before/after in a scratch test: identical.
- Size: about -60/+33 lines (measured on draft6, incl. a 27-line helper file)
- Risk: low.
- Proof: No test pins the panic text (grep 'ValueMust received error' in tests: 0). Add one test per Must function asserting the recovered panic message before refactoring.
- Low value on its own; fold into a PR that touches these files (e.g. E4/E9).

### E11. Doc comments that contradict the code or name removed/nonexistent identifiers

- Scope: types/basetypes (6 files), path/path.go, diag/warning_diagnostic.go: 13 comment sites | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: types/basetypes/object_value.go:66,128,157; types/basetypes/number_value.go:156-157; types/basetypes/float32_value.go:94; types/basetypes/float64_value.go:57-58,68-69; types/basetypes/string_value.go:48-49,59-60,70-71; types/basetypes/missing_value.go:53; path/path.go:27; diag/warning_diagnostic.go:40
- Current: ObjectValue docs point to "Elements or ElementsAs methods" (ObjectValue has Attributes and As; types/object_value.go already says so); ValueBigFloat says it returns 0.0 for null/unknown but returns nil (pinned by TestNumberValueValueBigFloat); Float32SemanticEquals says float32 has "53 bits of precision" (24); five paragraphs warn about "the deprecated Float64/String type Null, Unknown, or Value fields", which no longer exist; missingValue.ToTerraformValue says it "always returns an error" but returns (tftypes.Value{}, nil); the Path type doc example uses path.MatchRoot (an Expression constructor) instead of path.Root; NewWarningDiagnostic's doc starts with "NewErrorDiagnostic".
- Replacement: Correct each sentence ("Attributes or As methods", "returns nil", "24 bits", delete the five stale paragraphs, "returns an empty tftypes.Value and no error", path.Root(...), "NewWarningDiagnostic returns ...").
- Why equivalent: Comments only; no code change.
- Size: about -23/+8 lines in 8 files (measured on draft8)
- Risk: low.
- Proof: Facts checked: grep -rn 'Setting the deprecated' types/basetypes (5); ObjectValue method list (no Elements/ElementsAs); TestNumberValueValueBigFloat expects nil for null/unknown; missing_value.go:54 returns nil error; float32 has a 24-bit significand.
- Only content errors are listed; pure doc-comment style (ST1020 names such as "Float64Null creates" on NewFloat64Null) is out of scope, except the warning-diagnostic one, which names a different exported function.

### E12. BenchmarkSetTypeValidate* always benchmarks an empty set

- Scope: types/basetypes/set_value_test.go (benchmark helper) | Kind: test | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: types/basetypes/set_value_test.go:66-89 (benchmarkSetTypeValidate), :91-113 (6 size variants)
- Current: elements := make([]tftypes.Value, 0, elementCount); for idx := range elements { elements[idx] = ... } ranges over a zero-length slice, so every size validates an empty set: 10 -> 112 ns/op and 1,000,000 -> 4,959 ns/op (the growth is only the amortised setup allocation).
- Replacement: elements := make([]tftypes.Value, elementCount) so the loop fills it; drop (or skip) the 10000, 100000 and 1000000 variants, which become infeasible once the benchmark works because SetType.Validate's duplicate check is quadratic (measured: 1,000 elements 72 ms and 24 MB per op; 10,000 elements 7.8 s and 2.4 GB per op); switch to b.Loop() (or call b.ResetTimer()) so the setup is not timed.
- Why equivalent: Test-only; production code untouched. The benchmark starts measuring what its name says.
- Size: about -13/+1 lines (fix the make call, drop three variants)
- Risk: low: benchmarks do not run in CI.
- Proof: Ran in scratch: go test ./types/basetypes/ -run XXX -bench BenchmarkSetTypeValidate -benchtime 2000x (flat timings), then with the len-allocated slice: 10 -> 10 us, 100 -> 0.68 ms, 1000 -> 82.5 ms, 999,249 allocs/op.
- Once fixed, the benchmark exposes SetType.Validate's O(n^2) tftypes.Value.Equal duplicate scan (called from fwschemadata for every set attribute); see notes. Optionally switch the loops to b.Loop().
- Area G found the same bug independently (its G4), with the same one-line fix and the same advice to cap or drop the 100,000 and 1,000,000 variants.
- Land before G12 (b.Loop) and keep the two as separate PRs.

## F. Lifecycle packages and provider-defined functions

### F2. Return.NewResultData is a test gap, not unreferenced API: 11 of 12 implementations never run in tests, and no test runs the CustomType branch for any type

- Scope: function/*_return.go NewResultData (12 methods) | Kind: test | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: function/{bool,dynamic,float32,float64,int32,int64,list,map,number,object,set,string}_return.go NewResultData; production caller internal/fwserver/server_callfunction.go:34 (req.FunctionDefinition.Return.NewResultData through the function.Return interface)
- Current: deadcode -test reports 10 NewResultData methods as unreachable. RTA counts ListReturn and StringReturn as reachable only because tests put them into a function.Return interface; ListReturn's method never actually runs. Measured coverage: go test ./function/ -coverprofile shows NewResultData at 0.0% for all 12 types. go test ./internal/fwserver/ -run CallFunction -coverpkg=./function/ shows StringReturn at 60.0% (only the CustomType == nil branch) and the other 11 at 0.0%. go test ./internal/proto6server/ -run CallFunction -coverpkg=./function/ gives the same result. All 103 Return literals in internal/ tests are StringReturn.
- Replacement: Add a table-driven TestReturnNewResultData in function/ (or one test per <type>_return_test.go) that covers each of the 12 types with CustomType unset (expect NewResultData(basetypes.New<T>Unknown(...))) and with CustomType set (expect the value from ValueFrom<T>). Add one internal/fwserver CallFunction case with a non-String return.
- Why equivalent: Tests only. Production code is unchanged.
- Size: about +250 test lines (estimated)
- Risk: low: tests only
- Proof: Reproduce with: go test ./function/ -count=1 -coverprofile=c.out && go tool cover -func=c.out | grep NewResultData; go test ./internal/fwserver/ -run CallFunction -coverpkg=./function/ -coverprofile=c2.out; and grep -rn '\.NewResultData(' --include=*.go . | grep -v _test.go, which shows internal/fwserver/server_callfunction.go:34 as the only non-test caller.
- The 10 deadcode leads are resolved as a test gap. These are public interface methods dispatched in production, so they must not be deprecated or removed.
- The CustomType branch of these tests would have caught F1.

### F7. Doc bug: list/ docs were copied from ephemeral/ (the package overview describes Renew/Close and values not stored in plan/state) and several list docs name the wrong methods

- Scope: list/doc.go, list/config_validator.go, list/list_resource.go, list/schema.go, provider/configure.go, provider/provider.go (comments only) | Kind: cleanup | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: list/doc.go:4-19; list/config_validator.go:23; list/list_resource.go:92, :129-130, :260; list/schema.go:13, :18; provider/configure.go:75-77; provider/provider.go:128
- Current: (1) The false statements are the Renew/Close methods, 'offer practitioners list values' and 'can only be referenced in other list values, such as provider configuration attributes'; 'not stored in plan/state' is accurate for list results. (7) The data is passed to resource.ConfigureRequest.ProviderData in both ListResource (internal/fwserver/server_listresource.go:108-110) and ValidateListResourceConfig (server_validatelistresourceconfig.go:39).
- Replacement: Rewrite list/doc.go to describe list resources: listing managed resource instances by identity for `terraform query`, matched to a managed resource type by type name, implemented through ProviderWithListResources.ListResources. Fix the method names and links in (2) through (5), document (6) and (8), and point (7) at [resource.ConfigureRequest.ProviderData] for each ListResource type.
- Why equivalent: Comments only. No code, signature or behaviour changes.
- Size: about -20/+30 comment lines in 6 files (estimated)
- Risk: low: comments only
- Proof: Shown by reading plus the diff command above. Package overview as rendered: go doc ./list | head -12. grep -n 'Renew\|Close' list/*.go | grep -v _test matches only list/doc.go:10-11.
- list/configure.go and list/metadata.go are the same ephemeral copy. They are handled in F8 because the types themselves are unused.
- The internal comment at internal/fwserver/server.go:47-48 also says [list.ConfigureRequest.ProviderData]. It belongs to another area.

### F8. Unreferenced public API in list/: ConfigureRequest/Response and MetadataRequest/Response are never used (ListResource takes resource.* types), and ListResultsStreamDiagnostics has zero references and no test

- Scope: list/configure.go, list/metadata.go, list/list_resource.go:172-180 (public API: deprecate or test, never delete) | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: list/configure.go:9-34; list/metadata.go:6-24; list/list_resource.go:172-180; grep -rn 'list\.ConfigureRequest\|list\.ConfigureResponse\|list\.MetadataRequest\|list\.MetadataResponse' --include=*.go . (1 hit, a comment at internal/fwserver/server.go:48); grep -rn ListResultsStreamDiagnostics . | grep -v '^./.git/' (2 hits: the definition and its doc)
- Current: list.ConfigureRequest, list.ConfigureResponse, list.MetadataRequest and list.MetadataResponse are copies of the ephemeral/ files (identical after the substitution in F7), and their docs say they are 'supplied as an argument to the ListResource type Configure/Metadata method'. By design, ListResource.Metadata and ListResourceWithConfigure.Configure take resource.MetadataRequest/Response and resource.ConfigureRequest/Response ('One implementation ... can satisfy both interfaces', list_resource.go:33-36, :72-76). Nothing constructs or reads the list.* types. ListResultsStreamDiagnostics, added in 7d8104c (#1167) as a utility for providers, has no caller and no test (0.0% coverage in go test ./list/). Its body ends with a redundant 'if !push(x) { return }'.
- Replacement: Add '// Deprecated: ListResource implementations receive [resource.ConfigureRequest] (or MetadataRequest/Response); this type is unused.' to the four types and fix their docs. Keep ListResultsStreamDiagnostics and add a test that it yields exactly one ListResult{Diagnostics: diags} and stops when push returns false. Optionally simplify its body to 'push(ListResult{Diagnostics: diags})', and optionally use it at internal/fwserver/server_listresource.go:124-128, which inlines the same closure.
- Why equivalent: Deprecation notices and docs do not change behaviour. In the body simplification the if statement is the closure's last statement, so push is still called exactly once and its result ignored either way. For the fwserver reuse, the inline closure yields diagsResp, a ListResult whose only set field is Diagnostics, and diagsResp is not modified after the closure is created, so ListResultsStreamDiagnostics(diagsResp.Diagnostics) yields an identical value.
- Size: about +12 comment lines, +30 test lines, -3/+1 code lines (estimated)
- Risk: low: compatibility is preserved. Downstream code that names the four types gets SA1019 warnings, which is intended.
- Proof: Grep counts are in 'where'. The ephemeral copy is shown by diff <(sed 's/ephemeral resource/list resource/g; s/EphemeralResource/ListResource/g; s/ephemeral/list/g; s/Ephemeral/List/g' ephemeral/configure.go) list/configure.go (no output), and the same for metadata.go. Coverage: go test ./list/ -coverprofile shows ListResultsStreamDiagnostics and NewListResult at 0.0% in-package; NewListResult is covered from the proto5/6server tests.
- This is the only exported identifier in area F with zero references anywhere (deadcode lead confirmed). It stays as public API.
- The list package carries no experimental notice, so deletion would be a breaking change. Deprecate first.

### F9. Doc bug in function/: 13 docs still promise default parameter names ('param1'/'varparam') that were removed in v1.7.0, plus copy-paste errors in validator, Definition and MapReturn docs

- Scope: function/parameter.go, function/*_parameter.go (12), function/definition.go, function/parameter_validation.go, function/object_parameter_validator.go, function/map_return.go (comments only) | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: function/parameter.go:32-34; Name field docs at bool_parameter.go:70-72, dynamic_parameter.go:65-67, float32_parameter.go:67-69, float64_parameter.go:67-69, int32_parameter.go:66-68, int64_parameter.go:66-68, list_parameter.go:77-79, map_parameter.go:77-79, number_parameter.go:65-67, object_parameter.go:79-81, set_parameter.go:77-79, string_parameter.go:66-68; definition.go:14; parameter_validation.go:38, :83; object_parameter_validator.go:15; map_return.go:21
- Current: (1) Parameter.GetName's doc and every typed Name field say an empty name defaults to 'param<N>' or 'varparam'. Since v1.7.0 ('All parameters must be explicitly named via the Name field', CHANGELOG.md:338-339), each typed ValidateImplementation adds the error 'Function %q - Parameter at position %d does not have a name' (internal/fwfunction/diagnostics.go:12-28), and no code anywhere applies a default (grep -rn 'varparam\|param%d' --include=*.go internal/ | grep -v _test: 0 hits). (2) definition.go:14 says 'Always set at least the Result field'; the field is Return. (3) parameter_validation.go:38 has ParameterWithFloat32Validators 'should return a list of Float64 validators', and :83 has ParameterWithNumberValidators 'should return a list of Map validators'. (4) object_parameter_validator.go:15 reads 'ValidateParameterObject ValidateParameterSet performs the validation.' (5) map_return.go:21 calls MapReturn 'an ordered collect of a single element type'; maps are unordered, and 'collect' is a typo. (6) The 8 scalar <T>Parameter.ValidateImplementation methods have no doc comment, while the 4 collection/object ones do.
- Replacement: Typed Name fields: 'This field is required: an empty name raises an error diagnostic during GetProviderSchema.' Parameter.GetName (parameter.go:32-34): say the name must be non-empty and no default is applied; the built-in parameter types raise an error diagnostic for an empty name, custom implementations are not checked. Fix (2)-(5) and add the 8 doc comments as proposed.
- Why equivalent: Comments only.
- Size: about -44/+60 comment lines in 17 files (39 stale lines in 13 files, 5 one-line fixes, 8 new 4-line doc comments; estimated by counting), not -30/+30 in 16 files.
- Risk: low: comments only
- Proof: grep -n 'default to "param"\|default name will' function/*.go | grep -v _test gives 27 lines in 13 files. The missing-name error is pinned by 13 test files (grep -ln 'does not have a name' function/*_test.go | wc -l).
- Checked and not a gap: all 12 parameter files assert Parameter, ParameterWith<T>Validators and fwfunction.ParameterWithValidateImplementation, and all 12 return files assert Return (plus ReturnWithValidateImplementation for list/map/set/object). A first-pass regex that excluded digits wrongly reported Float32/Float64/Int32/Int64 as missing; a compile probe confirmed those types satisfy their interfaces.
- 'satisifies' appears in 24 function/ files (148 repo-wide), but only in comments on blank var declarations, which godoc does not render. That is a repo-wide typo sweep for the coordinator, not part of this item.

### F10. Doc bug: copy-paste and stale doc comments across lifecycle packages (wrong concept names, fields that do not exist, removed types, stale method names, missing docs)

- Scope: 39 sites (33 locations plus 6 'provider-specifc'), not about 25; 25 files (24 edited plus a new statestore/doc.go), not about 20. | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: resource/configure.go:30-31; action/configure.go:28-29; resource/schema.go:20, :23-24; datasource/schema.go:23-24; provider/schema.go:23-24; provider/metaschema.go:11-13, :16-17, :23-24; resource/identity_schema.go:16; resource/resource.go:41-42, :51-52, :65-66, :71-72, :227; datasource/data_source.go:37-38, :43-44, :50; ephemeral/ephemeral_resource.go:62; resource/read.go:40, :43, :73; resource/update.go:52; resource/state_upgrader.go:23; provider/provider.go:41; resource/move_state.go:39; resource/state_mover.go:49; resource/upgrade_identity.go:36; action/action.go:8; provider/configure.go:88-89; statestore/statestore.go:10-12, :75; statestore (no package doc); 'provider-specifc' at resource/configure.go:17, datasource/configure.go:17, ephemeral/configure.go:17, action/configure.go:15, list/configure.go:17, statestore/initialize.go:22
- Current: Wrong concept: resource and action ConfigureResponse.Diagnostics say 'configuring of the Datasource'. resource.SchemaResponse.Schema says 'the schema of the data source'. The SchemaResponse.Diagnostics docs in resource, datasource and provider, and provider.MetaSchemaResponse.Diagnostics, say 'related to validating the data source configuration'. provider.MetaSchemaRequest/Response say 'Provider type Schema method' instead of MetaSchema. resource.IdentitySchemaResponse says it is 'a response to a SchemaRequest'. Fields that do not exist: Resource.Delete says 'Config values may be read from the DeleteRequest' and EphemeralResourceWithClose.Close says 'Config values may be read from the CloseRequest', but neither request has a Config field. Resource.Read says 'Planned state values should be read from the ReadRequest', which holds current state. Removed API: ResourceWithConfigure and DataSourceWithConfigure say they are 'intended to replace the provider.ResourceType/DataSourceType ... method in a future release', but those types were removed (CHANGELOG.md:721). Their Configure is described as 'separately executed for each ReadResource/ReadDataSource RPC', but fwserver calls Resource Configure in 9 RPCs. Stale type names: ReadResourceResponse/ReadResourceRequest (resource/read.go), UpdateResourceRequest (resource/update.go:52), UpgradeResourceStateRequest (resource/state_upgrader.go:23), ConfigureProviderRequest (provider/provider.go:41). Wrong identifiers: [MoveStateResponse.State] should be TargetState (move_state.go:39), and 'StateMove defines' should be StateMover (state_mover.go:49). Formatting: '//schema version.' (upgrade_identity.go:36, the gocritic lead). Missing docs: ResourceWithUpgradeIdentity, action.Action, and statestore.StateStore (its comment is only a NOTE); statestore also has no package doc while every sibling package has a doc.go. Incomplete: provider.ConfigureResponse.Deferred says 'all resources and data sources', but ephemeral resources and actions are deferred too. Typos: 'implmentation' (x2), 'provider-specifc' (x6).
- Replacement: Correct each comment to name the right concept, field or type: DeleteRequest has State/Identity/ProviderMeta/Private, CloseRequest has Private, Configure runs before every RPC for that type. Remove the ResourceType/DataSourceType sentences, add the missing doc comments and a statestore/doc.go, and fix the typos.
- Why equivalent: Comments, plus one new doc-only file. No code changes.
- Size: about -55/+80 (estimated): about -45/+45 for the one- and two-line fixes, about +10 for three new doc comments, about +20 for statestore/doc.go (sibling doc.go files are 15-25 lines).
- Risk: low: comments only
- Proof: The stale type names come from a scan of doc comments for *Request/*Response words not declared in the package (script in notes-F.md). The Configure call sites come from grep -rln 'resource.ResourceWithConfigure' internal/fwserver/*.go | grep -v _test (9 files). Deferral scope comes from grep -rn 's.deferred' internal/fwserver/*.go | grep -v _test (openephemeralresource, planaction, planresourcechange, readdatasource, readresource, importresourcestate). Request fields are confirmed by the AST field scan (drafts/F/fieldscan).
- Field-population check is clean. Every exported field of every *Request struct in area F is set by internal/fwserver, except the four unused list.* types (F8). Every *Response field is read, except provider.MetadataResponse.Version, which its doc says is not connected to framework functionality (intentional).

### F11. Twin measurement: the only logic the 24 typed Parameter/Return files can share is the collection/object ValidateImplementation bodies; two unexported helpers save about 56 lines

- Scope: function/{list,map,set,object}_parameter.go and function/{list,map,set,object}_return.go ValidateImplementation, plus a new unexported function/validate_implementation.go | Kind: refactor | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: function/list_parameter.go:136-154, map_parameter.go:136-154, set_parameter.go:136-154, object_parameter.go:138-156; function/list_return.go:75-85, map_return.go:75-85, set_return.go:75-85, object_return.go:71-81
- Current: The Parameter bodies are 19 lines each including signature and brace (not 18); the '84 one-line getters' are 72 one-line getters plus 12 multi-line GetType methods.
- Replacement: Add unexported validateCollectionParameterImplementation(p Parameter, hasCustomType bool, req, resp) and validateCollectionReturnImplementation(typ attr.Type, hasCustomType bool, resp), whose bodies are the current ones with p.CustomType == nil replaced by !hasCustomType. Each of the 8 methods becomes one call: validateCollectionParameterImplementation(p, p.CustomType != nil, req, resp) or validateCollectionReturnImplementation(p.GetType(), p.CustomType != nil, resp). The exported methods, signatures and doc comments stay as they are.
- Why equivalent: Same diagnostics, in the same order, built by the same fwtype/fwfunction helpers. GetType and GetName are framework value-receiver methods with no side effects, so calling them through the Parameter interface on a copy of the struct returns the same values. In the return helper, p.GetType() is now evaluated even when CustomType != nil; in that case it just returns r.CustomType without calling user code. The CustomType nil check moves to the caller unchanged.
- Size: about -112/+8 lines in 8 files plus a new 48-line file, net -56 (measured on a scratch draft)
- Risk: low: TestList/Map/Set/ObjectParameterValidateImplementation and TestList/Map/Set/ObjectReturnValidateImplementation pin all branches
- Proof: The draft in drafts/F/fwcopy2 passes go test ./function/ and go vet, and golangci-lint with the project config reports 0 issues on ./function/. Twin hashes: for t in list map set object; do sed -n '/^func (p [A-Za-z]*Parameter) ValidateImplementation/,/^}/p' function/${t}_parameter.go | sed 's/[A-Z][a-z]*Parameter)/XParameter)/' | md5sum; done prints 4 equal hashes, and the same holds for Return.
- Optional and low value. The project deliberately keeps per-type twins. Only worth doing if F1 already touches these files. The dupl leads (39) are resolved by this measurement.
- Rejected: embedding a shared base struct (breaks composite literals and reflection), and a generic NewResultData helper (longer than the 9-line twins).

## G. Test infrastructure, tooling and CI

### G1. Replace the local pointer[T]/strPtr/newXPointer test helpers with Go 1.26 new(expr) (go fix -newexpr + go fix -inline), then delete the 13 helpers it leaves dead

- Scope: test files only: 42 *_test.go files in function, internal/fwschemadata, internal/reflect, internal/toproto5, internal/toproto6, tfsdk, types/basetypes | Kind: modernization | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: function/pointer_test.go:6; internal/fwschemadata/pointer_test.go:6; internal/toproto5/pointer_test.go:6; internal/toproto6/pointer_test.go:6; tfsdk/pointer_test.go:6; types/basetypes/pointer_test.go:6; internal/reflect/pointer_test.go:179 (strPtr); tfsdk/value_as_test.go:24-81 (13 helper definitions, of which the 6 single-pointer ones are deleted); call sites: 198 pointer( calls on 192 lines + 1 strPtr + 13 typed-helper calls = 212
- Current: Six packages each define an identical `func pointer[T any](value T) *T { return &value }` (6 files of 8 lines), internal/reflect defines `strPtr`, tfsdk/value_as_test.go defines 6 single-pointer helpers (newBool/Float32/Float64/Int32/Int64/StringPointer); 212 call sites use them.
- Replacement: Run `go fix -newexpr ./...` then `go fix -inline ./...`, then delete the 13 helpers that are now unused (the 6 pointer_test.go files become empty and are deleted). Calls become `new(int64(0))`, `new("test")`, `new(types.ListNull(...))`, `new(float32(12.3))`. Optional add-on in the same PR: the 6 newXPointerPointer helpers become `new(new(x))` (+12/-42, readability call); keep newBigFloatPointerPointer.
- Why equivalent: pointer(x) and new(x) both allocate a fresh variable initialised to x and return its address; the pointer's static type is identical because pointer[T] infers T from the argument exactly as new(expr) takes the expression's (default) type (pointer(1.2) and new(1.2) are *float64; pointer(attr.Value(v)) is *attr.Value). newexpr leaves calls whose default type differs (newInt64Pointer(0)) to -inline, which adds the explicit conversion new(int64(0)). Each call still yields a distinct allocation, so no aliasing changes. Only test code changes.
- Size: about -282/+206 lines across 42 files (36 modified, 6 deleted) (measured; patch G1-newexpr-pr.patch); raw go fix -newexpr alone is +225/-212
- Risk: low: test-only and fully automated; the only trap is that go fix -newexpr ALONE breaks CI (see notes)
- Proof: On a full scratch copy: go vet -p 2 ./... clean after each step; go test of the 7 changed packages ok (TestValueAs, TestArgumentsData*, TestDataGetAtPath, TestFromPointer, function parameter tests...); whole suite on the cumulative copy 56 ok / 0 FAIL; staticcheck -checks all on the cumulative copy identical to baseline (no new U1000); gofmt -l / -s -l clean. Nothing to add first: the rewritten tests are themselves the pins.
- go fix -newexpr keeps the helpers, rewrites their bodies to `return new(value)` and adds `//go:fix inline`; with every caller rewritten, staticcheck U1000 (= golangci `unused`, enabled in .golangci.yml) reports 10 dead helpers, so the PR must delete them. go fix -inline then makes newFloat32Pointer/newInt32Pointer/newInt64Pointer dead too (3 more).
- go fix -inline ./... touches nothing else in this module: no //go:fix inline directives in the repo or its 4 direct deps; stdlib ones (reflect.Ptr/PtrTo, io/ioutil, go/types) are unused here.
- Counts reconcile with the brief: newexpr's 218 diagnostics = 205 call sites + 13 helper definitions; -inline removes 7 more calls (212 total, = golangci modernize newexpr count).
- tfsdk/value_as_test.go has 13 helper definitions, not 14 (the 14th name is in a commented-out case); file diff +13/-37.
- Coordinator: this is the repo-wide newexpr item; dedupe with any other area that lists the same helpers.
- Fold in from E13: delete the dead commented-out 'primitive number pointer' case at tfsdk/value_as_test.go:160-170, which uses the removed types.Number{Value: ...} API and a nonexistent newBigFloatPointer.
- Do not hand-write new(12.3) or new(12) for the float32/int32/int64 helpers: untyped constants default to float64/int, and the interface{} table field hides the type change. go fix -inline writes new(float32(12.3)) correctly.
- Tier 2 rather than 1, following the verifier: the change touches 42 files and only changes idioms in test code. It is still fully mechanical: rerun the two go fix commands and compare.

### G2. Delete 12 unreachable internal/testing/testtypes fixtures: 9 never-instantiated XValue wrappers, SetTypeWithValidateError, SetTypeWithValidateAttributeError/SetValueWithValidateAttributeError

- Scope: internal/testing/testtypes (11 files); internal test fixtures with no compatibility promise | Kind: cleanup | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/testing/testtypes/{dynamic,float32,float64,int32,int64,list,map,object,set}.go:11-14 and :30-42; internal/testing/testtypes/setwithvalidate.go:17-34; internal/testing/testtypes/setwithvalidateattribute.go:17-45
- Current: Each of 9 files defines `XType` (overrides only Equal) plus `XValue` whose only reference is its own `var _ basetypes.XValuable = XValue{}`; XType inherits ValueFromTerraform/ValueType from basetypes so it never produces XValue. SetTypeWithValidateError and SetTypeWithValidateAttributeError (+ its SetValueWithValidateAttributeError) have zero references anywhere.
- Replacement: Remove the 9 XValue types with their Equal methods and assertion lines (the var block becomes `var _ basetypes.XTypable = XType{}`), remove SetTypeWithValidateError (type, Validate, assertion) and SetTypeWithValidateAttributeError + SetValueWithValidateAttributeError.
- Why equivalent: The removed code is unreachable: deadcode -test reports all 12 methods unreachable; grep finds 0 qualified references (`testtypes.SetTypeWithValidateError`, `testtypes.ListValue{`...) and the unqualified same-package hits are field names of embedded basetypes values (e.g. `DynamicValue: in`). No interface in actual use requires them; the kept XType types behave identically (their methods are untouched). No reflection or registration can reach a type that is never instantiated.
- Size: -206/+11 lines, 11 files (measured; patch G2-dead-testtypes.patch)
- Risk: low: compiler-checked deletion of unreachable test fixtures
- Proof: go vet -p 2 ./... on the draft (compiles every test package) clean; go test ./internal/reflect ./internal/fwschemadata ./tfsdk ok. deadcode -test ./... lists exactly these 12 methods (brief's deadcode.txt).
- Their siblings are used: List/Map *WithValidate*Error by internal/reflect slice/map tests, Set*Warning by fwschemadata; reflect's set path shares FromSlice/Into with lists, so this is leftover surface, not a missing test.
- Not dead despite zero name references: Bool/StringValueWithValidateParameterError, List/Map/Number/ObjectValueWithValidateAttributeError, List/Map/Set/SingleNestedAttributesCustomValue (returned by a used Type's ValueFromTerraform) and DynamicTypeWithSemanticEquals (returned by DynamicValueWithSemanticEquals.Type()).

### G3. Fill the three empty tests TestNumber_int16, TestNumber_uint16 and TestNumber_uint32 (body is only t.Parallel())

- Scope: internal/reflect/number_test.go | Kind: test | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: internal/reflect/number_test.go:201-203; internal/reflect/number_test.go:455-457; internal/reflect/number_test.go:497-499
- Current: `func TestNumber_int16(t *testing.T) { t.Parallel() }` (same for uint16, uint32) while the 11 sibling TestNumber_<kind> tests have 14-line bodies. The in-range conversion paths for reflect.Int16/Uint16/Uint32 in internal/reflect/number.go:85,121,126 are only exercised by the overflow/underflow error tests.
- Replacement: Give each the TestNumber_int8 body with its own kind: convert tftypes Number 123 into the kind via refl.Number, assert no diagnostics and n == 123.
- Why equivalent: Adds assertions only; no production change.
- Size: +33 lines, 1 file (measured; patch G3-empty-number-tests.patch)
- Risk: low: new tests pass on the current code
- Proof: Draft run: go test -run 'TestNumber_(int16|uint16|uint32)$' -v ./internal/reflect -> 3 PASS. Found by sweeping all Test funcs with no t.Error/t.Fatal/t.Run (3 hits repo-wide, all here).

### G5. Remove or wire up table-test fields that are never read (7 families), including a misleading expectedError the server never produces

- Scope: 24 test files in internal/fromproto5|6, internal/reflect, internal/fwserver, internal/proto5server|6server, internal/fwschemadata | Kind: test | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: ... internal/proto5server/server_listresource_test.go:177 (expectedDiagnostics, 3 empty assignments) and internal/proto6server/server_listresource_test.go:197 (expectedDiagnostics, 4 empty assignments) ... (the finding says '3x empty each'; proto6 has 4, which the patch already removes: 9 expectedDiagnostics lines in total)
- Current: Fields declared in table-test structs and set in cases but never read by the loop, so they look like inputs/expectations but have no effect. Worst case: TestServerListResource case "zero-results-on-nil-config" sets `expectedError: "config cannot be nil"`, which is never asserted, and fwserver.ListResource does not produce it (a nil Config is replaced by a null config, internal/fwserver/server_listresource.go:91-101).
- Replacement: Delete the dead declarations and assignments; for the 8 Get* tests pass `testCase.request` instead of `new(tfprotovN.XRequest)` (8 one-line edits; all 38 values are empty requests); drop the `provider` import that becomes unused in server_generateresourceconfig_test.go.
- Why equivalent: Removed fields are never read; for the Get* tests every request value is `&tfprotovN.XRequest{}`, identical to `new(...)`, and each case has its own pointer. Test behaviour is unchanged.
- Size: -103/+24 lines, 24 files (measured; patch G5-dead-table-fields.patch)
- Risk: low: compile-checked, tests unchanged
- Proof: Custom AST sweep over all *_test.go: fields of struct types declared inside a Test func that never appear in a selector in that func (390 raw hits, almost all reflection targets; these 7 families are the real table fields). Draft: go vet ./internal/... clean; go test of fromproto5/6, fwschemadata, fwserver, proto5server/6server, reflect ok.
- Same TestServerListResource (internal/fwserver/server_listresource_test.go:298-306) has an asymmetric cmp.Comparer that loops only over `a`. It is NOT a silent pass: cmp's symmetry check (always run on the first comparer calls of a Diff, and every case has <= 2 events) panics 'non-deterministic or non-symmetric function detected'. Worth a `len(a) != len(b)` guard in the same PR for a readable failure.

### G6. Stop using Diagnostics.AddError like printf in test fixtures: 51 summaries contain a literal "%s"

- Scope: 27 files: internal/fwserver, internal/proto5server, internal/proto6server tests and internal/testing/planmodifiers/planmodifiers.go | Kind: test | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: grep -rnE --include='*.go' '\.(AddError|AddWarning|AddAttributeError|AddAttributeWarning)\("[^"]*%[a-zA-Z]' . (51 sites, 27 files; e.g. internal/fwserver/server_readdatasource_test.go:171, internal/testing/planmodifiers/planmodifiers.go:23,36)
- Current: `resp.Diagnostics.AddError("unexpected req.Config value: %s", config.TestRequired.ValueString())`: AddError(summary, detail) is not a formatter, so the summary literally reads "...: %s" (the value lands in the detail).
- Replacement: Drop the ": %s" from the 8 distinct summaries (`AddError("unexpected req.Config value", ...)`), one sed over 27 files.
- Why equivalent: These diagnostics are only produced when a fixture's own check fails (never on a passing run) and no test expects these summaries (grep for the summary strings outside AddError = 0), so every currently passing test is unaffected; only failure output improves.
- Size: -51/+51 lines, 27 files (measured; patch G6-adderror-printf.patch)
- Risk: low: failure-message text only
- Proof: Draft: go vet and go test of internal/fwserver, internal/proto5server, internal/proto6server ok; re-grep finds 0 remaining.
- govet printf does not catch this because AddError is not a printf wrapper.

### G7. Fix three copy-paste slips in internal/testing fixtures (wrong interface assertion, wrong error text, malformed package comment)

- Scope: internal/testing/testtypes and internal/testing/testschema (3 files) | Kind: test | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: internal/testing/testtypes/singlenestedattributescustom.go:17-20; internal/testing/testschema/nested_attribute.go:51,59; internal/testing/testtypes/doc.go:4
- Current: (1) singlenestedattributescustom.go's assertion block is a copy of setnestedattributescustom.go's and asserts `_ basetypes.SetTypable = SetNestedAttributesCustomTypeType{}` / `_ basetypes.SetValuable = &SetNestedAttributesCustomValue{}`, so the SingleNested fixtures have no compile-time check at all. (2) testschema.NestedAttribute.ApplyTerraform5AttributePathStep returns "cannot apply step %T to ListNestedAttribute" for the Map and Set nesting modes (the real types say MapNestedAttribute / SetNestedAttribute). (3) `// types contains shared attr.Type ...` is not a valid package comment (staticcheck ST1000).
- Replacement: (1) `_ basetypes.ObjectTypable = SingleNestedAttributesCustomTypeType{}` and `_ basetypes.ObjectValuable = &SingleNestedAttributesCustomValue{}`; (2) MapNestedAttribute / SetNestedAttribute; (3) `// Package testtypes contains ...`.
- Why equivalent: (1) compile-time only. The corrected assertions compile because the types embed types.ObjectType and types.Object. Today the fixture's only check is the implicit attr.Type one at its 2 use sites (fwserver/attribute_validation_test.go:1625,1740), not 'none at all'. (2) No test observes the fixture text: the 24 hits for 'to ListNestedAttribute' are all expectations for the real ListNestedAttribute types in 8 schema packages' list_nested_attribute_test.go, not only provider/metaschema. (3) comment only.
- Size: -5/+5 lines, 3 files (measured; patches G7a-singlenested-assert-and-doc.patch, G7b-testschema-msg.patch)
- Risk: low
- Proof: go vet ./internal/testing/testtypes ok; staticcheck -checks ST1000 on testtypes clean after (3); go test of all 18 packages that import testschema ok after (2). Scan: every other `var _ X = T{}` in internal/testing names a type declared in its own file.

### G8. Rename list.ExampleResource_listable so pkg.go.dev shows it (go/doc drops it: package list has no identifier Resource)

- Scope: list/list_resource_test.go | Kind: test | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: list/list_resource_test.go:39-41
- Current: `func ExampleResource_listable()` in package list_test. go vet accepts the name because `Resource` resolves to the imported resource.Resource, but go/doc (used by pkg.go.dev) only attaches examples to identifiers of the documented package, so this example is silently dropped. It is the only Example function in the repository.
- Replacement: `func ExampleListResource()` (and its comment), which attaches to type list.ListResource.
- Why equivalent: The example has no `// Output:` comment, so it is compiled but never run; renaming changes only documentation placement.
- Size: -2/+2 lines, 1 file (measured; patch G8-example-rename.patch)
- Risk: low
- Proof: go/doc probe (doc.NewFromFiles over list/*.go): before 'ExampleResource_listable NOT ATTACHED', after 'ExampleListResource -> type ListResource'; go vet ./list clean.
- The deadcode leads for ComputeInstanceResource/NoOpListResource/NoOpResource methods in list/*_test.go are compile-time interface checks used by this example: not dead.

### G9. 14 table tests compute most `expected` values with the method under test (12 fully, 2 partially: 33 of 35 entries); pin Path.AtTupleIndex in the path package's own Steps/String tests

- Scope: path/expression_test.go (8 funcs), path/path_test.go (5 funcs), types/basetypes/dynamic_value_test.go (1 func) | Kind: test | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: path/expression_test.go:15,45,75,105,138,171,204,234; path/path_test.go:15,53,91,129,167; types/basetypes/dynamic_value_test.go:18
- Current: e.g. TestPathAtTupleIndex: `path: path.Root("test"), index: 1, expected: path.Root("test").AtTupleIndex(1)` and the loop computes `testCase.path.AtTupleIndex(testCase.index)`: same call, same input, so the test can only fail on non-determinism. TestNewDynamicValue does the same in 3 of 4 cases.
- Replacement: Minimal: add AtTupleIndex cases to TestPathSteps (expected `path.PathSteps{PathStepAttributeName("test"), PathStepElementKeyInt(1)}`) and TestPathString (expected `test[0]`). Optional: convert the 14 tautological expectations to literal Steps()/String() forms or delete them as redundant.
- Why equivalent: Adds independent assertions only.
- Size: +11 lines, 1 file for the minimal change (measured; patch G9-tuple-index-cases.patch); optional rewrite not measured
- Risk: low
- Proof: Mutation: making AtTupleIndex append PathStepAttributeName("MUTANT") still passes TestPathAtTupleIndex, while the new TestPathSteps/TestPathString cases fail. The other At* builders are pinned independently by TestPathSteps/TestPathString/TestExpressionSteps/TestExpressionString; AtTupleIndex is only caught outside the package (internal/reflect TestFromSlice/tupleElemWithValidateAttributeError).
- Found with a heuristic over 2846 table-driven tests (expected entries that call the same function the loop calls): 14 hits, all read.
- The fully tautological cases can fail only on a panic or non-determinism; 'only on non-determinism' overstates it slightly. TestPathAtName/empty and TestNewDynamicValue/nil are real expectations.

### G10. Remove the Makefile website targets and the CONTRIBUTING link to a website/ directory that does not exist

- Scope: Makefile, .github/CONTRIBUTING.md | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: Makefile:4-19; .github/CONTRIBUTING.md:50; optionally the same residue in .github/CODEOWNERS:3-17 and .gitignore:1-3
- Current: `website`, `website/local` and `website/build-local` run `$(MAKE) -C website ...`, but there is no website/ directory: `make website` fails with "make[1]: *** website: No such file or directory.  Stop.". CONTRIBUTING.md:50 says the repository contains the documentation markdown and links ../website/README.md (missing).
- Replacement: Delete Makefile lines 4-19 (keep `generate`); reword CONTRIBUTING.md:50 to point to where the framework docs now live (maintainers to confirm the repository).
- Why equivalent: The removed targets cannot succeed today; `make generate` is untouched.
- Size: Makefile -16 lines; CONTRIBUTING.md 1 line (estimated)
- Risk: low
- Proof: `ls -d website` -> no such directory; `make -n website` on the scratch copy fails as quoted.
- terraform-plugin-testing, -sdk and -log Makefiles carry the same residue; terraform-plugin-go's Makefile has no website targets, so there is precedent for removing it.
- Not included: .copywrite.hcl header_ignore entries .github/labeler-*.yml and .release/*.hcl match no file, but the file is a shared template byte-identical to sibling repos (notes).
- The replacement for CONTRIBUTING.md:50 does not need to wait for maintainers to name the repository: the removal commit d2548a9 says the docs moved to https://github.com/hashicorp/web-unified-docs. Size if the optional residue is included: Makefile -16, CONTRIBUTING 1 line, CODEOWNERS about -15, .gitignore -4.

### G11. Add a package comment to statestore, the only one of 78 packages without one

- Scope: statestore (public package documentation) | Kind: cleanup | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: statestore/*.go (14 files, staticcheck ST1000 once per file; no doc.go)
- Current: pkg.go.dev shows no package overview for statestore; 77 of 78 packages have one (most via doc.go, e.g. list/doc.go).
- Replacement: statestore/doc.go: `// Package statestore contains all interfaces, request types, and response types for a state store implementation.` (wording mirrors list/doc.go; maintainers' choice).
- Why equivalent: Comment only; no code or API change.
- Size: +5 lines, 1 new file (estimated)
- Risk: low
- Proof: `for p in $(go list -f '{{.Dir}}' ./...); do go list -f '{{.Doc}}' $p | grep -q . || echo $p; done` -> statestore only.
- ST1020/ST1021 (166 doc-comment form hits) are masked by the golangci `comments` preset and are out of scope.
- Cross-reference F7: do not copy more of list/doc.go than its first sentence, because the rest describes ephemeral-resource semantics.
- Base the package comment on the first sentence of list/doc.go only: the rest of that file is the copied ephemeral text reported in F7.

### G12. Use testing.B.Loop (Go 1.24) in the two benchmark helpers so their setup is no longer timed

- Scope: types/basetypes/object_value_test.go, types/basetypes/set_value_test.go (benchmarks only) | Kind: test | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: types/basetypes/object_value_test.go:23-45; types/basetypes/set_value_test.go:64-89
- Current: `for n := 0; n < b.N; n++ {...}` with no b.ResetTimer, so building 1000 attributes / the set is included in ns/op and B/op; set_value_test.go keeps a package-level `benchDiags` sink to defeat dead-code elimination; neither helper calls b.Helper() (thelper).
- Replacement: `for b.Loop() {...}`, drop the `benchDiags`/`diags` sink (b.Loop keeps call results alive), add `b.Helper()`.
- Why equivalent: Deliberately not benchmark-equivalent: setup leaves the measurement (more accurate). `go test` (no -bench) is unaffected because benchmarks do not run.
- Size: -8/+7 lines, 2 files (measured; patch G12-bloop.patch)
- Risk: low: benchmark-only
- Proof: Draft: go vet and go test ./types/basetypes ok. BenchmarkSetTypeValidate at -benchtime=2000x: before 24 B/op (size 10) vs 16027 B/op (size 1000000); after 24 B/op for both, i.e. the setup allocation is no longer measured.
- Apply after the G4 bug fix; keep the two PRs separate.
- Sequence after E12 (the empty-set benchmark fix).

### G13. Apply the three go fix modernizers whose hits are all in tests: fmtappendf (6), rangeint (5), waitgroup (2)

- Scope: 6 test files: internal/privatestate/data_test.go, internal/proto5server|6server/serve_test.go, internal/proto5server|6server/server_invokeaction_test.go, types/basetypes/object_value_test.go | Kind: modernization | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: go fix -fmtappendf / -rangeint / -waitgroup ./... (sites listed in notes-G.md section 1)
- Current: `[]byte(fmt.Sprintf(...))`, `for i := 0; i < 10; i++` with i unused or unmodified, `wg.Add(1); go func(){ defer wg.Done(); ... }()`.
- Replacement: `fmt.Appendf(nil, ...)`, `for range 10` / `for i := range 5`, `wg.Go(func(){ ... })`.
- Why equivalent: fmt.Appendf returns the same bytes, and nil-vs-empty differs only for an empty result, which cannot happen because all 6 formats are non-empty literals. rangeint fires only when the loop variable and the bound are not modified in the body. In Go 1.26 wg.Go(f) equals Add(1) + go with a deferred Done, except that Done is skipped if f panics (fatal either way); the two bodies never panic or Goexit, so the Add/Done pairing is preserved.
- Size: -19/+15 lines, 6 files (measured; patch G13-test-only-analyzers.patch)
- Risk: low: automated, test-only
- Proof: Per analyzer on the scratch copy: go vet ./... clean, tests of changed packages ok, gofmt clean; included in the whole-suite run (56 ok / 0 FAIL).
- Coordinator: dedupe with the repo-wide go fix items; the any/reflecttypefor/mapsloop/slicescontains numbers are in notes-G.md 'go fix verification'.
- Overlap: area A's A16 reworks the same TestServerCancelInFlightContexts loops in internal/proto5server/serve_test.go and internal/proto6server/serve_test.go (the 200 ms sleep). Sequence the two PRs, or fold the wg.Go change into A16.
- Edits the same TestServerCancelInFlightContexts loops in internal/proto5server and proto6server as A16: land one, then rebase the other.

## M. Repo-wide mechanical modernization (go fix, Go 1.26)

### M1. Replace interface{} with any in one go fix -any run (554 sites in 269 files)

- Scope: all | Kind: modernization | Tier: 2 (solid) | Verdict: CONFIRMED
- Where: `go fix -diff -any ./...` lists every site: 47 internal non-test files (177 lines), 175 public non-test files (191 lines), 47 test files (185 lines). Examples: internal/logging/framework.go:18-33, function/func_error.go:124, types/basetypes/*_type.go ApplyTerraform5AttributePathStep.
- Current: 553 lines spell the empty interface as interface{}. Nine public schema files already use any in the same method (ApplyTerraform5AttributePathStep), so the code base mixes both spellings.
- Replacement: Run `go fix -any ./...` and commit the result on its own.
- Why equivalent: any is a predeclared alias for interface{} (Go spec, since 1.18), so the types are identical. Exported signatures, method sets, reflect.Type values, %T output and compiled code do not change. No package declares an identifier named any that could shadow the builtin (checked by grep). The only visible difference is that godoc prints any.
- Size: ±553 lines in 269 files (measured on a git-archive copy; matches area G's independent measurement)
- Risk: low: no semantic change. The cost is merge conflicts with open PRs on the same lines, so land it alone, right after a release.
- Proof: Applied on a fresh copy: go vet ./... clean, gofmt -l empty, go test -p 2 ./... green (56 packages ok, 0 failed). Area G separately ran the whole suite on its own copy after applying every analyzer: 56 packages ok, 0 failed. After the PR, `go fix -diff -any ./...` must print nothing.
- If reviewers want PRs under about 300 lines, split by directory: internal/ (177 lines), public packages (191 lines), tests (185 lines). Because the tool regenerates the diff, a single PR is also reviewable: rerun it and compare.

### M2. reflect.TypeOf((*T)(nil)).Elem() and similar -> reflect.TypeFor[T]() (31 sites in 13 files)

- Scope: all | Kind: modernization | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: `go fix -diff -reflecttypefor ./...`: internal/reflect/into.go:70,76,80,95; internal/reflect/generic_attr_value.go:14; 26 sites in 11 test files (e.g. internal/fwschemadata/data_get_at_path_test.go:102,136,137).
- Current: Interface types are obtained with the pre-1.22 idiom reflect.TypeOf((*X)(nil)).Elem(), and one pointer type with reflect.TypeOf((*attr.Value)(nil)).
- Replacement: Run `go fix -reflecttypefor ./...`.
- Why equivalent: reflect.TypeFor[T]() returns the identical reflect.Type: it is defined as TypeOf((*T)(nil)).Elem() for interface T. The removed operands are nil conversions with no side effects, so dropping their evaluation changes nothing. The analyzer rewrites TypeOf(x) only when x has no side effects and a non-interface static type.
- Size: ±30 lines in 13 files (measured)
- Risk: low
- Proof: Applied on top of M1 on a copy: go vet and gofmt clean, full suite green (56 packages ok, 0 failed). internal/reflect's Into/BuildValue tests exercise the four Implements checks.
- C17's reflect.TypeFor part is the same rewrite, limited to internal/reflect. Land whichever comes first and drop the overlap from the other.

### M3. After the go fix items land, enable golangci-lint's modernize linter so the old idioms do not come back

- Scope: tooling | Kind: modernization | Tier: 3 (fold-in) | Verdict: CONFIRMED
- Where: .golangci.yml linters.enable. Residual sites the linter would still flag: 12 reflecttypeassert hits in internal/reflect/interfaces_test.go:276,414,562,929; internal/reflect/pointer_test.go:54,74,94; internal/reflect/pointer_zero_test.go:18,34,50,66,82.
- Current: The enabled linters do not include modernize, so new code can bring back interface{}, local pointer helpers and pre-1.22 reflect idioms. At 00badb1, modernize reports 800 diagnostics: 541 any, 212 newexpr, 26 reflecttypefor, 12 reflecttypeassert, 4 mapsloop, 3 slicescontains and 2 waitgroupgo.
- Replacement: After M1, M2, G1, G13, E9 and C18 land, add modernize to linters.enable. In the same PR, convert the 12 test sites from v.Interface().(T) to reflect.TypeAssert[T](v) (Go 1.25).
- Why equivalent: CI configuration only, so no library behaviour changes. reflect.TypeAssert[T](v) is documented as semantically equivalent to v.Interface().(T) in its comma-ok form.
- Size: +1 configuration line and about ±12 test lines (estimated)
- Risk: low: contributors must then write current idioms. CI already runs golangci-lint v2 through golangci-lint-action v9.3.0 with no version pin, and modernize ships with golangci-lint v2.
- Proof: Scratch golangci-lint v2.15 run with modernize enabled at 00badb1 (the counts above). Area G confirmed that once every go fix analyzer is applied, go fix reports nothing more.
- This is a maintainer decision: the project curates its linter list deliberately. Propose it as its own PR, after the conversions.

## Bugs (separate PRs, each with a test)

### B37. ValidateActionConfig rejects every configured write-only action attribute (WriteOnlyAttributesAllowed hard-coded false), so action schemas' WriteOnly field (1.17.0, #1233) is unusable

- Where: internal/fwserver/server_validateactionconfig.go:100-106; rejection at internal/fwserver/attribute_validation.go:155-161; documentation of the capability at schema/validator/client_capabilities.go:10-16
- Impact: Every action whose schema marks an attribute WriteOnly fails validation as soon as the practitioner sets it (including from an ephemeral value, which is unknown during validate), with a misleading 'only supported in Terraform 1.11 and later' error.
- Current: SchemaValidate for actions runs with ClientCapabilities.WriteOnlyAttributesAllowed=false. AttributeValidate then adds 'WriteOnly Attribute Not Allowed: The resource contains a non-null value for WriteOnly attribute X. Write-only attributes are only supported in Terraform 1.11 and later.' for every action attribute (at any depth) with IsWriteOnly() and a non-null value, known or unknown. Meanwhile GetProviderSchema advertises write_only=true for the same attribute.
- Replacement: Set WriteOnlyAttributesAllowed: true in ValidateActionConfig (every client that sends ValidateActionConfig supports write-only) and fix the comment; because the value is also handed to provider-defined attribute validators, update the public doc comment of validator.ValidateSchemaClientCapabilities.WriteOnlyAttributesAllowed to say it is populated for managed resource and action schemas, and add a changelog note. (Alternative that keeps validators seeing false: give SchemaValidate a separate internal flag that skips the line-155 rejection for actions.) If B28 lands first, pass true to validateConfigSchema for actions.
- Size: 1 line plus comment/doc lines (measured on a copy), plus one TestServerValidateActionConfig case with a WriteOnly attribute (known and unknown value)
- Risk: low: actions require Terraform 1.14+; the only observable side effect is the capability value seen by action attribute validators (documented above)
- Proof: Verifier probes: fwserver level, go test -overlay=drafts/V-D/actwo/overlay.json -run TestZZVActWO -v ./internal/fwserver/ (a 'secret' value gives 1 error and null gives 0). Protocol level, go test -overlay=drafts/V-D/actwo/overlay6.json -run TestZZVActWO6 -v ./internal/proto6server/: a Required+WriteOnly string attribute gives an error for both known 'hunter2' and unknown values, and GetProviderSchema reports write_only=true with 0 diags. With overlay-fix.json both give 0 diags. Core sends the real (unmarked) config values to ValidateActionConfig during validate (terraform@v1.14.0 internal/terraform/node_action_validate.go:108-114; v1.17.1 internal/terraform/node_action.go:213-218). Add first: a ValidateActionConfig test with a WriteOnly attribute set to a known and to an unknown value, expecting no diagnostics.
- Makes D4 and D5 fully latent until fixed.
- The error text also says 'The resource contains ...' for an action; consider a neutral wording when touching it.
- Found by the independent verifier of area D while checking that area.
- Re-checked independently by the verifier of area B (RPC handlers): confirmed.

### C1. reflect.Into panics (reflect.Set) for named numeric Go types and for map targets whose key type is not plain string

- Where: internal/reflect/number.go:76,84,92,100,102,115,120,125,130,132,163,175 (successful returns not converted to the target type); internal/reflect/map.go:89 (key not converted); the panic is raised by the caller at into.go:46, struct.go:166, pointer.go:51, slice.go:90,159 or map.go:89
- Impact: Provider process crash whenever a model field, list/set element, map value or GetAttribute/ElementsAs target uses a named numeric type (documented as an acceptable conversion type), and for map targets with a named string key or a non-string, non-interface key (undocumented shapes, which should get the diagnostic FromMap already gives).
- Current: Number returns reflect.ValueOf(int8(x)), reflect.ValueOf(intResult), reflect.ValueOf(floatResult)... i.e. a Value of the predeclared type, and Map calls m.SetMapIndex(reflect.ValueOf(key), result) with a plain string key. For a named target type (type Port int64, type Ratio float64, map[KeyT]string) or a non-string key kind (map[int]string) the caller's reflect.Set / reflect.Append / SetMapIndex panics: 'reflect.Set: value of type int64 is not assignable to type Port'. Primitive already does reflect.ValueOf(s).Convert(target.Type()) for strings and bools (primitive.go:38,50) and TestPrimitive_stringAlias / TestPrimitive_boolAlias pin alias support; the Set direction (FromValue) already accepts named numerics and named string keys.
- Replacement: Number: append .Convert(target.Type()) to the 12 successful returns (not the *big.Int one). Map: keyType := underlyingValue.Type().Key(); before the loop, only when len(values) > 0 and string is neither assignable to keyType nor keyType.Kind() == reflect.String, return DiagIntoIncompatibleType 'map keys must be strings, got %s'; in the loop keep reflect.ValueOf(key) when reflect.TypeFor[string]().AssignableTo(keyType), otherwise .Convert(keyType). Draft: drafts/V-C/fix1b (diff drafts/V-C/c1-corrected-map.diff): all panics gone, map[any]T, map[EmptyIface]T and empty-map-into-map[int]T unchanged, tests and golangci-lint pass.
- Size: about -13/+30 lines with the corrected map handling (measured on drafts/V-C/fix1b minus the no-op *big.Int Convert); the author's draft is -13/+22 but changes working inputs
- Risk: low with the corrected map check: only inputs that panic today change
- Proof: Reproduced through the public API: drafts/C/probe-public (module with replace => repo) TestNamedIntPanicsViaPublicAPI and TestNamedIntGetAttribute: tfsdk.State.Get and State.GetAttribute panic with 'reflect.Set: value of type int64 is not assignable to type probepublic.Port'. Internal repros drafts/C/repros/reflect/zz_repro_namednum_test.go, zz_repro_namedkey_test.go, zz_repro_intkey_test.go (run with drafts/C/repros/run.sh <copy-of-repo>): Into of a struct with myInt/myFloat fields, Into *myInt, Into map[keyT]string and map[int]string all panic. With the draft fix the repros pass and go test ./internal/reflect/ passes. Nothing pins the panic. Add first: TestNumber_int64Alias / float64Alias mirroring TestPrimitive_stringAlias, a named-key case in TestReflectMap_string, and a map[int]string diagnostic case.
- Same class as C2 (panics instead of diagnostics in reflect) but a different code path; keep as separate commits.
- number.go:56 returns reflect.ValueOf(result) (a *big.Float) together with an error diagnostic for a *big.Int target; every caller checks HasError before Set, so it does not panic.

### C2. reflect.FromValue panics on a nil *big.Int and on typed-nil pointers that implement attr.Value, Unknownable or Nullable

- Where: internal/reflect/number.go:378 (FromBigInt: big.NewFloat(0).SetInt(val) with val == nil); internal/reflect/outof.go:25-36 (interface dispatch happens before any nil-pointer check) then interfaces.go:444 (val.Type(ctx) on a nil *basetypes.StringValue) or the Unknownable/Nullable methods
- Impact: Provider crash on an ordinary Read/Set round trip of an optional number modelled as *big.Int when the value is null.
- Current: Get of a null number into a *big.Int field yields nil (BuildValue canBeNil, documented support for *big.Int at into.go:165), but Set of that same model panics with 'invalid memory address or nil pointer dereference' inside big.Float.SetInt. A nil *big.Float becomes a null value (FromBigFloat), and a nil *string goes through FromPointer and becomes null. A (*types.String)(nil) passed to SetAttribute satisfies attr.Value and panics in FromAttributeValue ('value method basetypes.StringValue.Type called using nil *StringValue pointer'); nil pointers to Unknownable/Nullable implementations panic in GetUnknown/GetNull.
- Replacement: FromBigInt: if val == nil { return FromBigFloat(ctx, typ, nil, path) } (identical to today's nil *big.Float handling). FromValue: before the attr.Value / ValueCreator / Unknownable / Nullable checks, route a nil pointer to FromPointer only when its element type implements the interface by value receivers (reflect.ValueOf(val).Type().Elem().Implements(...)), because only then is today's method call a guaranteed panic; pointer-receiver implementations may be nil-safe and must keep their current dispatch. To be exact for mixed types, test the first interface in dispatch order. Draft: drafts/V-C/fix2.
- Size: about +18 lines (measured on drafts/V-C/fix2, including a 7-line helper and a comment)
- Risk: low: only panicking inputs change
- Proof: Public API: drafts/C/probe-public TestNilBigIntRoundTrip: State.Get returns Count == nil without diagnostics, then State.Set(model) panics; State.SetAttribute(ctx, path.Root("name"), (*types.String)(nil)) panics. Internal: drafts/C/repros/reflect/zz_probe_nilbig_test.go (nil *big.Float -> <null>, nil *big.Int -> panic, Into-then-FromValue round trip panics), zz_probe_ptrattr_test.go, zz_probe_nilunknownable_test.go. Add first: nil cases in TestFromBigInt and TestFromValue_go_types.
- *types.String fields are rejected by Get (NewAttributeValue type mismatch diagnostic), so that part is less likely in practice; the *big.Int round trip is the realistic trigger.

### F1. ListReturn/SetReturn/MapReturn.NewResultData ignore the CustomType's element type: a CustomType-only collection return crashes the provider on every Run error

- Where: function/list_return.go:59-69; function/set_return.go:59-69; function/map_return.go:59-69; caller internal/fwserver/server_callfunction.go:34; crash site internal/toproto6/function.go:104 (twin internal/toproto5/function.go) -> types/basetypes/list_value.go:225
- Impact: The provider process crashes when Run returns an error, and also when ResultData.Set fails, for any function whose Return is a ListReturn, SetReturn or MapReturn with only CustomType set (both protocols; no recover() in terraform-plugin-go v0.31.0). Parameter-validator failures do NOT crash: they return before Run with no Result (verifier probe: error 'param validation failed', result nil). Which calls crash depends on the custom type: if its value's Type() is derived from the embedded value (the internal/testing/testtypes pattern and the usual embed-basetypes pattern), every call crashes, because Set fails ('missing type information; cannot create value' or a type mismatch) and the placeholder is then converted; if Type() returns a fixed type, only calls whose Run returns an error crash; if ValueFromList rebuilds unknown values with its own element type, nothing crashes.
- Current: NewResultData builds the unknown placeholder from the ElementType field (value := basetypes.NewListUnknown(r.ElementType)) even when CustomType is set, then wraps it with CustomType.ValueFromList. The type docs say 'Either the ElementType or CustomType field must be set' (list_return.go:22, set_return.go:22, map_return.go:22), and the repo's own tests build ListReturn{CustomType: testtypes.ListType{...}} without ElementType. With CustomType only, the placeholder has a nil element type. fwserver.CallFunction always copies runResp.Result into the response, and toproto5/6.FunctionResultData always calls attrValue.ToTerraformValue, which dereferences the nil element type.
- Replacement: In the CustomType branch of the three methods, take the element type from the custom type when ElementType is unset: 'elementType := r.ElementType; if elementType == nil { if t, ok := r.CustomType.(attr.TypeWithElementType); ok { elementType = t.ElementType() } }; valuable, diags := r.CustomType.ValueFromList(ctx, basetypes.NewListUnknown(elementType))' (ValueFromSet/ValueFromMap and NewSetUnknown/NewMapUnknown in the twins). The CustomType == nil branch keeps NewListUnknown(r.ElementType).
- Size: Author's draft measured exactly -12/+36 (-4/+12 per file). A tighter draft that only reassigns value when ElementType is nil is +24/-0 (8 lines per file including a 2-line comment) with identical probe results. Tests about +120 (estimated).
- Risk: low: only the CustomType-only configuration changes, and today it panics or errors. The new code also calls the custom type's ElementType(), which is usually the method promoted from basetypes.ListType.
- Proof: End-to-end reproduction through providerserver.NewProtocol5WithError/NewProtocol6WithError and CallFunction (probe in drafts/F/probe-listreturn). The return is ListReturn, SetReturn or MapReturn with CustomType set to a type that embeds basetypes.ListType/SetType/MapType{ElemType: String}, and Run only sets resp.Error = function.NewFuncError("boom"). Both protocols panic with 'invalid memory address or nil pointer dereference' (stack: basetypes.ListValue.ToTerraformValue list_value.go:225 <- toproto6.FunctionResultData function.go:104 <- toproto6.CallFunctionResponse callfunction.go:22 <- proto6server.CallFunction server_callfunction.go:54). With ElementType also set, the call returns the clean error 'boom'. With a type built like internal/testing/testtypes.ListType (embedding, no overrides), ResultData.Set(ctx, []string{"a"}) fails with 'missing type information; cannot create value' and the response then panics. With the scratch draft fix there are 0 panics across all probe cases, Set succeeds, go test ./function/ passes, and golangci-lint with .golangci.yml reports 0 issues on ./function/. No test exercises this path today: outside function/, every Return literal in tests is a StringReturn (grep -rn --include=*_test.go -o 'function\.[A-Z][A-Za-z0-9]*Return{' . | grep -v StringReturn lists only function/*_return_test.go and function/definition_test.go). Tests to add first: TestListReturnNewResultData, TestSetReturnNewResultData and TestMapReturnNewResultData, each with a 'CustomType' case expecting the custom type's element type; and a proto5server/proto6server CallFunction case with Return ListReturn{CustomType: testtypes.ListType{ListType: basetypes.ListType{ElemType: basetypes.StringType{}}}} whose Run returns an error.
- ObjectReturn is left out because its doc (object_return.go:24) says 'The AttributeTypes field must be set'. With CustomType only, its error path does not panic (ObjectValue.ToTerraformValue tolerates nil attribute types), but Set of a struct fails with 'Struct defines fields not found in object'. The same guard using attr.TypeWithAttributeTypes would fix it if maintainers want CustomType-only objects.
- Rejected variant: always preferring the CustomType's element type. That changes behaviour when both fields are set and disagree. The nil-only guard is the minimal fix.
- Rejected variant: building the placeholder with CustomType.ValueFromTerraform(tftypes unknown). It mirrors the schema path but calls a different user method (ValueFromTerraform instead of ValueFromList).
- Related: F2 (NewResultData test gap). Land F1's tests in the same PR.
- Add: fix the contradictory ElementType field doc ('This field must be set', list/set/map_return.go:32-33) in the same PR.

### A1. GenerateResourceConfig server capability is dropped by toproto5/toproto6, so Terraform never calls the framework's GenerateResourceConfig RPC

- Where: internal/toproto5/server_capabilities.go:15-25; internal/toproto6/server_capabilities.go:15-25; source of truth internal/fwserver/server_capabilities.go:41-48
- Impact: No user-visible change in generated configuration today: fwserver.GenerateResourceConfig is a transcription of Terraform core's legacy logic meant to have 'no functionality changes' (fwserver/server_generateresourceconfig.go:39-43; CHANGELOG 1.19.0). The defect is that the protocol response contradicts fwserver.ServerCapabilities, so the about 434-line GenerateResourceConfig path (fwserver 192, protoNserver 82, fromproto 98, toproto 62) never runs in production, and the planned replacement of that logic ('will be replaced in a future release') would silently not take effect.
- Current: fwserver.(*Server).ServerCapabilities() returns GenerateResourceConfig: true (added in 2ee75aa, 'Enable GenerateResourceConfig server capability', released in 1.19.0 whose CHANGELOG says config generation moved from Terraform core into the framework for Terraform >= 1.14). Both toproto ServerCapabilities converters copy only GetProviderSchemaOptional, MoveResourceState and PlanDestroy, so the tfprotov5/6.ServerCapabilities sent in GetMetadata and GetProviderSchema always carries GenerateResourceConfig=false. Terraform core (internal/terraform/node_resource_plan_instance.go:1191, main branch) calls provider.GenerateResourceConfig only when providerSchema.ServerCapabilities.GenerateResourceConfig is true and otherwise falls back to genconfig.ExtractLegacyConfigFromState, so the framework's GenerateResourceConfig RPC (proto5server/proto6server/fwserver, ~300 lines) never runs in production.
- Replacement: Add `GenerateResourceConfig: fw.GenerateResourceConfig,` to both converters, and add `GenerateResourceConfig: true,` to the 30 expected ServerCapabilities values in proto5server/proto6server TestServerGetMetadata and TestServerGetProviderSchema (they currently encode the dropped flag).
- Size: about -0/+32 lines (measured: 2 converter lines; 30 subtests fail after the fix until their expectation gains one line)
- Risk: medium: it switches on, for Terraform >= 1.14, a path that has only run in unit tests; also unverified: how terraform-plugin-mux merges this capability for muxed SDKv2+framework providers. Run terraform-provider-corner (and a muxed provider) with an import block and -generate-config-out on Terraform >= 1.14 and compare with the legacy output before release.
- Proof: As stated, plus: the Terraform core gating (node_resource_plan_instance.go:1191 on the main branch) is unverifiable context from a fetched file, not proof. Verifier reproduced: probe fw true -> proto false for both protocols; sandbox fix -> exactly 30 failing subtests, every diff '- GenerateResourceConfig: false / + true'.
- Field audit: every other tfprotov5/6 response field is either set or intentionally unset (UnsafeToUseLegacyTypeSystem, SchemaBlock.Version, SchemaNestedBlock.MinItems/MaxItems); GenerateResourceConfig is the only fwserver field the converters never read.
- Git: commit 2ee75aa touched fwserver/server_capabilities.go and its tests but no toproto file.
- Area B's auditor reproduced this independently while checking the fwserver side.

### A5. proto5server/proto6server keep every RPC's cancel func and context chain until StopProvider: unbounded growth per RPC

- Where: internal/proto5server/serve.go:19-39; internal/proto6server/serve.go:19-39; registerContext is called by 27 handlers in proto5server and 35 in proto6server (the stated 29/37 also count the definition line and serve_test.go).
- Impact: Measured on the real stack (grpc-go server + tf6server + framework, loopback TCP): about 11.5 KB retained per completed RPC (219.9 MiB per 20,000 GetMetadata RPCs), about 23 times the stated 0.5 KB lower bound, released only by StopProvider or process exit. Each Terraform walk keeps one provider process and sends several RPCs per resource (estimate: about 3 in a plan), i.e. on the order of 35 KB per managed resource, hundreds of MB for workspaces with 10k resources.
- Current: registerContext appends the context.WithCancel cancel func of every RPC to s.contextCancels. The slice is cleared only by cancelRegisteredContexts, called from StopProvider, which Terraform sends only on interrupt. Each closure keeps its cancelCtx alive, and through it the parent chain (gRPC stream context, terraform-plugin-go logger contexts). terraform-plugin-go's own stoppableContext (tf5server/server.go:441-452) avoids this by using a goroutine that exits when the request context is done.
- Replacement: Make the registry forget completed RPCs: store the cancel funcs in a map[uint64]context.CancelFunc (or a set) and register `context.AfterFunc(ctx, func() { lock; delete(s.contextCancels, id) })` in registerContext. cancelRegisteredContexts keeps iterating and canceling, then resets the map. Draft: drafts/A/drafts/A5_proto6server_serve.go.
- Size: +19/-2 per package measured on the v6 draft (+38/-4 total); stated '-2/+15'.
- Risk: medium: touches concurrency in the hot path of every RPC; covered by TestServerCancelInFlightContexts (passes under -race on the draft) and a new growth test
- Proof: Verifier: registry len 1000 (pristine) vs 0 (draft) after 1000 completed RPCs; real-stack heap delta 11,528 vs 2-3 bytes/RPC; -race stress clean; in-flight cancellation preserved. Add first: the growth test (N RPCs with canceled parents, assert empty registry).
- Rejected alternative: delete registerContext and rely on terraform-plugin-go's stoppableContext. That is not equivalent for callers that use the ProviderServer directly (providerserver.NewProtocol5/6 consumers, mux, tests); TestServerCancelInFlightContexts pins the framework's own cancellation.

### B1. SchemaModifyPlan never sets AttributePathExpression: plan modifiers get an empty PathExpression for top-level attributes/blocks and one missing the parent inside SingleNestedAttribute/SingleNestedBlock

- Where: internal/fwserver/schema_plan_modification.go:85-92; internal/fwserver/schema_plan_modification.go:142-149; propagated by internal/fwserver/attribute_plan_modification.go:638 (Single arm) and :783 (+ the 11 per-type twins), internal/fwserver/block_plan_modification.go:401; contrast internal/fwserver/schema_validation.go:48-53 and :64-69
- Impact: Provider plan modifiers that resolve relative expressions from req.PathExpression are affected in two ways. (1) For every top-level attribute or block of any type, including object/list/set modifiers on top-level nested attributes and blocks, the expression is empty: MatchRelative().AtParent().AtName(x) resolves to nothing and Plan/Config/State.PathMatches returns the error 'Invalid Path Expression for Schema' (upstream #1258). (2) For attributes reached through a chain of SingleNestedAttribute/SingleNestedBlock from the root, the parent steps are missing: the lookup silently matches a root-level attribute of the same name only if one exists, and otherwise fails with the same error (upstream #1219: nested.double_nested.str_a receives 'double_nested.str_a'). Any provider message that prints req.PathExpression shows '' or a truncated path. Validators on the same attributes receive the correct value.
- Current: attrReq := ModifyAttributePlanRequest{AttributePath: path.Root(name), Config: req.Config, State: req.State, Plan: req.Plan, ProviderMeta: req.ProviderMeta, Private: req.Private} (blockReq identical): AttributePathExpression stays the zero path.Expression{}. The per-type functions copy it into planmodifier.<T>Request.PathExpression, and the Single nesting arms pass it down (PathExpression: req.AttributePathExpression), so children get path.Expression{}.AtName(child). List/Set/Map element children are correct because those arms use attrPath.Expression(). SchemaValidate sets AttributePathExpression: path.MatchRoot(name), so validators on the same attributes get the right value.
- Replacement: Add `AttributePathExpression: path.MatchRoot(name),` to both literals (exactly what SchemaValidate does), plus TestSchemaModifyPlan cases asserting req.PathExpression for a top-level attribute, a SingleNestedAttribute child and a SingleNestedBlock child.
- Size: about +2 semantic lines (+14/-12 including gofmt realignment of the two literals, measured) plus a ~60-line test
- Risk: low: only the PathExpression handed to plan modifiers changes, to the documented value. Plan modifiers that ignore it are unaffected; those resolving relative expressions go from an error or a wrong match to the correct match. A provider that worked around the bug by matching the truncated expression will now see the full expression (PR #1311 notes this).
- Proof: Add the verifier probe drafts/V-B1/probes/zz_vb1_pathexpr_test.go (outputs out-pathexpr-pristine.txt and out-pathexpr-fixed.txt in the same directory). Also add the docs sentence: developer.hashicorp.com/terraform/plugin/framework/handling-data/path-expressions says schema-based 'attribute validation and attribute plan modification, are provided an accurate current path expression'.
- Only SchemaModifyPlan builds root ModifyAttributePlanRequest values (grep 'ModifyAttributePlanRequest{' in non-test code: schema_plan_modification.go:85/142 plus the nested builders that derive from them).
- If B10 lands first the fix is one line in schemaModifyPlanAtRoot (verified in the cumulative draft).
- No upstream issue found for this.
- Replace 'No upstream issue found for this.' with: known upstream as #1258 (open, 2026-01-21, empty expression at top level) and #1219 (open, 2025-09-10, missing parent in nested single objects). Open fix PRs #1260 (2026-01-23) and #1311 (2026-07-06) add the same path.MatchRoot(name) line to both loops, with schema-level tests. The plan should review and land one of these rather than open a new PR. If B10 lands first, the fix is one line in schemaModifyPlanAtRoot (verified in the cumulative draft).

### B15. PlanResourceChange drops the automatic provider deferral when ProviderDeferred.EnablePlanModification is true and the resource has no ModifyPlan method

- Where: internal/fwserver/server_planresourcechange.go:61 (skip of the automatic-deferral return), :313-372 (provider deferral attached only inside `if resourceWithModifyPlan, ok := ...`, at :357-371); contract in resource/metadata.go:51-57
- Impact: A provider whose Configure defers (only possible when the client allows deferrals: server_configureprovider.go:25-31 rejects it otherwise, so today Terraform Stacks or experimental CLI builds) returns a normal, non-deferred plan for every resource that opted into EnablePlanModification without a ModifyPlan method, so Terraform plans (and later applies) it instead of deferring it.
- Current: With s.deferred set (provider Configure deferred) and ResourceBehavior.ProviderDeferred.EnablePlanModification true, the early return at :61 is skipped so plan modification can run, but the provider deferral is copied into resp.Deferred only inside the ResourceWithModifyPlan block (:357-371). A resource that only has schema/attribute plan modifiers gets an ordinary, non-deferred plan. resource/metadata.go:52-55: 'framework will still execute provider-defined resource plan modification logic if provider.Configure defers. Framework will then automatically return a deferred response along with the modified plan.'
- Replacement: Move the `if s.deferred != nil { ...; return }` block out of the ModifyPlan `if`, directly after it, and log `resp.Deferred.Reason` instead of `modifyPlanResp.Deferred.Reason` (same value). Draft: drafts/B2/drafts/fixes/internal/fwserver/server_planresourcechange.go. Add a TestServerPlanResourceChange case 'create-response-deferral-automatic-plan-modification-without-modifyplan' (testprovider.Resource, EnablePlanModification true, provider ConfigureMethod sets Deferred) expecting Deferred{ProviderConfigUnknown}.
- Size: about +13/-13 lines in 1 file (measured on draft), plus one ~45-line test case
- Risk: low: deferral support is experimental (resource/metadata.go:49-50 says no compatibility guarantee); the 3 existing EnablePlanModification cases pass on the draft
- Proof: Probe drafts/B2/repo/internal/fwserver/zz_probe_deferral_test.go at 00badb1: without-modifyplan Deferred=<nil>, with-modifyplan Deferred=&{Provider Config Unknown}; on the fix draft both are &{Provider Config Unknown}. go test ./internal/fwserver ./internal/proto5server ./internal/proto6server pass on the draft. Terraform core does not act on ConfigureProvider deferral itself: hashicorp/terraform main internal/providers/provider.go ConfigureProviderResponse has only Diagnostics, so the framework's per-RPC deferral is the only signal.
- Not changed: the early return in the moved block still skips identity validation, NormaliseRequiresReplace and the destroy-plan null check for deferred plans, as it does today on the ModifyPlan path.
- Side effect: gocognit of PlanResourceChange 71 -> 69 and the nestif lead at :313 disappears (golangci-lint on the draft).

### B16. Automatic ImportResourceState deferral omits the request identity, so Terraform rejects identity-based imports instead of deferring them

- Where: internal/fwserver/server_importresourcestate.go:74-95 (auto-deferred ImportedResource has no Identity); manual-deferral path for comparison :151-171 and :228-236
- Impact: With a deferring provider Configure (requires a client that allows deferrals: Terraform Stacks or experimental CLI builds), every `import { identity = {...} }` of a framework resource fails the plan with "didn't return an identity" instead of being deferred (Terraform 1.12+).
- Current: When s.deferred is set the response is one ImportedResource{State: unknown, TypeName, Private: &privatestate.Data{}} without Identity. When the resource's own ImportState defers, the framework returns Identity (pre-populated from req.Identity). Terraform core (hashicorp/terraform main, internal/terraform/node_resource_plan_instance.go:949, evaluated for deferred responses too) fails with 'import of <addr> didn't return an identity' when importing by identity and imported[0].Identity is null; internal/plugin6/grpc_provider.go only decodes Identity when the proto field is non-nil.
- Replacement: Add `Identity: req.Identity,` to the automatically deferred ImportedResource (draft drafts/B2/drafts/fixes/internal/fwserver/server_importresourcestate.go). Alternative: return no ImportedResources when auto-deferring; core then fabricates a null state for deferred imports (node_resource_plan_instance.go:903-927).
- Size: +1 line (measured on draft), plus one test case
- Risk: low
- Proof: Probes drafts/B2/repo/internal/fwserver/zz_probe_import_deferral_test.go and (verifier) drafts/V-B2/repo/internal/proto6server/zz_vb2_test.go TestVB2_B22 at 00badb1: automatic deferral Identity=<nil>, manual deferral Identity=<id=abc>; on the fix draft the automatic path returns the identity. The core check is present in released Terraform v1.12.0 and v1.14.0 (fetched), not only on main.
- PlanResourceChange's automatic deferral also returns no PlannedIdentity, but core skips identity checks for deferred plans (node_resource_abstract_instance.go:1071-1073), so that one is harmless.

### B17. Warnings collected before the provider call (from the type's Configure method, or appended by the proto layer such as provider meta schema warnings) are discarded by 9 RPC handlers and by ApplyResourceChange

- Where: internal/fwserver/server_createresource.go:130; server_readresource.go:159; server_updateresource.go:153; server_deleteresource.go:147; server_readdatasource.go:106; server_openephemeralresource.go:104; server_renewephemeralresource.go:92; server_closeephemeralresource.go:75; server_invokeaction.go:86; server_applyresourcechange.go:65, :93, :120
- Impact: Provider warnings (Configure-time deprecation or misconfiguration notices, provider_meta schema warnings) never reach practitioners on apply, refresh, data source reads, ephemeral open/renew/close and action invoke, although plan and validate show them.
- Current: These handlers overwrite resp.Diagnostics with the provider method's response (e.g. `resp.Diagnostics = createResp.Diagnostics`), discarding everything already in resp: warnings from the type's Configure (appended a few lines earlier) and warnings the proto5server/proto6server wrappers appended before calling the handler (e.g. ProviderMetaSchema diagnostics, which are returned on every call, or first-call Schema warnings); ApplyResourceChange repeats the overwrite one level up. Siblings keep them: PlanResourceChange (:288, :329) and PlanAction (:86) seed the method response with resp.Diagnostics, while ImportResourceState, UpgradeResourceState/Identity, the 7 Validate*Config handlers and the 6 state store handlers Append.
- Replacement: `resp.Diagnostics.Append(xResp.Diagnostics...)` at the 12 sites (draft drafts/B2/drafts/fixes). Strict alternative that also keeps exact duplicates inside the provider's own response: `resp.Diagnostics = append(resp.Diagnostics, xResp.Diagnostics...)`.
- Size: 12 lines in 10 files (measured on draft), plus a 'configure-warning' case per handler test and one proto-level case
- Risk: low: after the change the cached provider_meta schema diagnostics (returned on every ProviderMetaSchema call) also surface on every ReadResource, ApplyResourceChange, ReadDataSource and ephemeral/action response, as they already do on PlanResourceChange; mention it in the changelog
- Proof: Probe drafts/B2/repo/internal/fwserver/zz_probe_configure_warn_test.go at 00badb1: a Configure warning is missing for CreateResource, ReadResource, UpdateResource, DeleteResource, ReadDataSource, OpenEphemeralResource, CloseEphemeralResource and InvokeAction and present for the controls PlanResourceChange and PlanAction; on the draft it is present for all. Probe drafts/B2/repo/internal/proto6server/zz_probe_metawarn_test.go: a provider meta schema warning is absent from ReadResource (0 diagnostics) but present in PlanResourceChange; on the draft ReadResource returns it. No test pins the current behaviour. fwserver, proto5server, proto6server and providerserver tests pass on the draft.
- MoveResourceState's `resp.Diagnostics = moveStateResp.Diagnostics` (:224, :243) looks similar but is intended: test 'response-Diagnostics-first-error-always-responds' documents that earlier movers' warnings are dropped. It also drops warnings the proto layer appended before the call; leave it to maintainers.
- GenerateResourceConfig (:183) and ConfigureStateStore (:71) also overwrite resp.Diagnostics; they have no Configure step, so they only drop proto-layer warnings (first-fetch schema warnings, conversion warnings). Add them to the same PR (2 more lines) if consistency is wanted.

### B18. NullifyWriteOnlyAttributes and MarkComputedNilsAsUnknown re-wrap the captured context for every visited value: O(n) retained heap per transform and O(n^2) time with SDK logging on

- Where: internal/fwserver/write_only_nullification.go:24; internal/fwserver/server_planresourcechange.go:416
- Impact: Every managed resource RPC response goes through NullifyWriteOnlyAttributes (Plan, Create, Read, Update, Import, Upgrade, Move); with TF_LOG=trace or debug, large nested states make each RPC quadratically slower, and without logging the transform keeps about 800 bytes per value node alive.
- Current: Inside the tftypes.Transform callback, `ctx = logging.FrameworkWithAttributePath(ctx, path.String())` assigns to the enclosing function's captured ctx, so each call wraps the previous call's context (a context.WithValue layer holding a copied LoggerOpts per value node). Lookups of other context keys, such as the SDK logger on every log call, walk the whole chain.
- Replacement: `ctx := logging.FrameworkWithAttributePath(ctx, path.String())` at both sites (shadow instead of assign).
- Size: 2 lines in 2 files (measured)
- Risk: low
- Proof: Verifier benchmark on the real functions (drafts/V-B2/zz_vb2_ctx_test.go.txt in drafts/V-B2/repo vs drafts/V-B2/repo-ctx): logging on, n=8000: NullifyWriteOnlyAttributes 4.16 s vs 0.12 s, MarkComputedNilsAsUnknown 15.2 s vs 0.18 s (quadratic: x3-x6 per doubling); logging off: no measurable time difference, 25.6 MB vs ~0 KB retained while the transform runs. Author's figures (3.72 s vs 0.16 s) reproduce in kind. fwserver/proto5server/proto6server pass with the change (drafts/B2/drafts/fixes).
- Performance defect with identical output; it can also be reviewed as a refactor because the logs and values are byte-identical.
- Repo-wide only these two closures reassign a captured ctx (grep -rn -B3 'ctx = logging.FrameworkWith' --include=*.go . | grep -v _test: the other 5 hits reassign a function parameter at function scope, which is fine).

### C3. Every element and field is validated twice in FromSlice/FromMap/FromStruct (and SetAtPath); set elements get diagnostics at a list-index path plus duplicated warnings

- Where: internal/reflect/slice.go:235-290 (list/set loop: FromValue with the index path at :246, set path fix-up at :258-260, re-validation at :262-287), slice.go:334-380 (tuple loop), map.go:150-206, struct.go:240-314; internal/fwschemadata/data_set_at_path.go:97-153 (top-level value re-validated after reflect.FromValue, as its MAINTAINER NOTE at :97-98 acknowledges)
- Impact: Wrong diagnostic path for set elements (an index step on a set is not a valid path), duplicated warnings shown to practitioners, and double validator cost (e.g. JSON or timestamp parsing in custom types) on every Set, SetAttribute and *ValueFrom.
- Current: FromValue already runs ValidateAttribute / TypeWithValidate.Validate for each element (FromString, FromAttributeValue, ...); the loop then runs it again on the result. For lists, tuples, maps and struct fields both runs use the same path, so diag.Diagnostics.Append drops the duplicates and only the validator call count doubles. For sets the first run uses the 'technically incorrect index-based path' (slice.go:236-241 comment) and its diagnostics are kept: a set element error is reported at attr[0] (the correctly-pathed second run is never reached because of the early return), and a warning is reported twice, at attr[0] and at attr[Value("a")].
- Replacement: First land C6 (return the dropped diagnostics). Then drop the loop re-validation for list, tuple, map and struct and the SetAtPath re-validation; for sets, validate each element once at path.AtSetValue after conversion (this needs a non-validating conversion entry point). Note that nested diagnostics inside set elements ([0].attr) stay index-based unless the conversion is restructured, and that dropping the SetAtPath block also drops its 3 trace log lines.
- Size: about -40/+15 lines on top of C10 (estimated: 4 loop call sites, the set path change and the SetAtPath block)
- Risk: medium: validators with side effects (logging, counters) run once instead of twice; set diagnostic paths change (intended)
- Proof: Public API: drafts/C/probe-public TestSetValueFromValidation: types.SetValueFrom(ctx, warnStringType{}, []string{"a","b"}) returns 4 warnings at [0], [Value("a")], [1], [Value("b")] and calls ValidateAttribute 4 times; types.ListValueFrom returns 2 warnings but also 4 calls. Internal: drafts/C/repros/reflect/zz_repro_set_validate_test.go (set element error reported at attr[0]), zz_repro_count_test.go (calls=4 for 2 elements in list, set, map, tuple and struct; 1 for a single string), drafts/C/repros/fwschemadata/zz_repro_setatpath_count_test.go (SetAtPath of one string: 2 calls). Coverage (go test -coverprofile ./internal/reflect/): the early-return branches of the second validation (map.go:191-193,199-201; slice.go:275-277,283-285,365-367,373-375; struct.go:289-291,297-299) never execute, consistent with them being unreachable for deterministic validators. Pinned today: TestFromSlice listElemWithValidate*/tupleElemWithValidate* expect one warning per element (de-duplicated). Add first: set cases with element warnings/errors to TestFromSlice, asserting AtSetValue paths.
- Land after C10: with the helper each re-validation is a 5-line call that can simply be deleted for list/tuple/map/struct.
- Removing the re-validation is NOT a pure refactor (call count and set paths change); recorded as rejected-refactor in notes-C.md.
- Depends on C6 (hard prerequisite) and C10 (convenience). Removing the re-validation is not a refactor: call counts, set paths and SetAtPath trace logs change.

### C4. tfsdk struct tags reject a leading underscore that schema attribute names allow

- Where: internal/reflect/helpers.go:115-119 and 133-138 (regexp ^[a-z][a-z0-9_]*$); internal/fwschema/attribute_name_validation.go:61 (ValidAttributeNameRegex ^[a-z_][a-z0-9_]*$)
- Impact: A provider whose schema uses an underscore-prefixed attribute (valid for Terraform and for the framework's own schema validation) cannot use struct models with Get/Set for that schema: every call returns an error.
- Current: ValidateImplementation accepts an attribute named _internal (pinned by fwschema TestIsValidAttributeName case ascii-lowercase-alphabet-leading-underscore), but Get/Set of a struct tagged tfsdk:"_internal" fails with 'invalid tfsdk tag, must only use lowercase letters, underscores, and numbers, and must start with a letter'.
- Replacement: Duplicate the pattern `^[a-z_][a-z0-9_]*$` in internal/reflect (or move the regexp into a leaf package both can import) and reword the tag error message. internal/reflect cannot import internal/fwschema: internal/fwschema -> types/basetypes -> internal/reflect is an import cycle (compile error reproduced in drafts/V-C/fix4).
- Size: about +3/-3 lines plus test cases (estimated)
- Risk: low: widens the accepted tag set to exactly the names the schema already accepts
- Proof: drafts/C/repros/reflect/zz_repro_underscore_test.go: IsValidAttributeName("_internal") returns no diagnostics, while Into and FromValue both return the invalid-tag error. CHANGELOG entry for #705 shows schema validation was made to reject only leading digits. Pinned today: TestIsValidFieldName (no underscore-first case), TestGetStructTags and TestNewStruct_errors messages. Add first: TestIsValidFieldName "_a": true and a struct round trip with an underscore tag.
- Confidence medium: the tag rule predates #705 and could be argued to be intentional, but the two validators disagree today.
- Confidence medium: the tag rule's own error text ('must start with a letter') shows it was deliberate, but it predates #705 and disagrees with the schema validator; the old website object docs say the tag 'must name an attribute in the object'.

### D1. resource/schema ObjectAttribute.ValidateImplementation raises a false 'default type mismatch' error when CustomType and AttributeTypes are both set

- Where: resource/schema/object_attribute.go:314 (in ValidateImplementation at :283); field doc :54 says AttributeTypes 'must be set'; sibling guards: resource/schema/single_nested_attribute.go (a.CustomType == nil && ...), resource/schema/list_attribute.go / map_attribute.go / set_attribute.go (compare ElementType only)
- Impact: A provider that follows the field doc (AttributeTypes 'must be set') and uses a custom object type with a strict Equal plus a Default gets an error diagnostic during GetProviderSchema (internal/fwserver/server.go:789 runs Schema.ValidateImplementation), so the resource cannot be used. Workaround: omit AttributeTypes when CustomType is set.
- Current: if a.AttributeTypes != nil && !a.GetType().Equal(defaultResp.PlanValue.Type(ctx)) { resp.Diagnostics.Append(fwschema.AttributeDefaultTypeMismatchDiag(req.Path, a.GetType(), defaultResp.PlanValue.Type(ctx))) }. GetType() returns CustomType when set, while defaults.ObjectResponse.PlanValue is always basetypes.ObjectValue, so a custom type whose Equal requires its own type (the usual custom-type pattern, and the repo's own internal/testing/testtypes.ObjectType) never compares equal.
- Replacement: if a.AttributeTypes != nil { expectedType := types.ObjectType{AttrTypes: a.AttributeTypes}; if !expectedType.Equal(actualType) { Append(AttributeDefaultTypeMismatchDiag(req.Path, expectedType, actualType)); return } }; if a.CustomType != nil && !a.CustomType.TerraformType(ctx).Equal(actualType.TerraformType(ctx)) { Append(AttributeDefaultTypeMismatchDiag(req.Path, a.CustomType, actualType)) } where actualType := defaultResp.PlanValue.Type(ctx). The author's draft alone (compare only against AttributeTypes) is not enough (see risk). Verified variant: drafts/V-D/d1/object_attribute.go (about -2/+19 lines).
- Size: about -2/+6 lines in 1 file (measured on draft) plus 2 test cases
- Risk: low-medium: the author's draft also removes a correct rejection. With CustomType set, AttributeTypes does not define the schema type, so a default that matches AttributeTypes but not the custom type's own attribute types (AttributeTypes and CustomType disagree, which nothing else validates) is accepted after the draft. It is rejected today when the custom type's Equal is not strict (probe case 8). The TerraformType comparison in the corrected replacement closes this. 'A genuinely mismatched default is still rejected' only holds when AttributeTypes matches the CustomType.
- Proof: Verifier probe: go test -overlay=drafts/V-D/d1/overlay-before.json -run TestZZVD1 -v ./resource/schema/ (case 1: diags=1 with identical type strings; via Schema.ValidateImplementation haserror=true). With overlay-after.json (author draft): cases 6 and 8 become 0. With overlay-alt.json (corrected variant): all 8 cases correct, and go test ./resource/schema/ passes. Add first: customtype-attributetypes-default (expect 0), customtype-attributetypes-default-mismatch (expect 1), customtype-disagreeing-attributetypes-default (expect 1).
- Only resource/schema has defaults, so no other package carries this guard.
- ObjectAttribute is the only collection-like attribute that compares the whole (possibly custom) type; list/map/set compare element types and nested attributes guard with CustomType == nil.

### D2. dynamicplanmodifier.UseNonNullStateForUnknown ignores underlying null/unknown values and copies a typed-null prior state into the plan

- Where: resource/schema/dynamicplanmodifier/use_non_null_state_for_unknown.go:37-55 (guards at :39, :44, :49); sibling in the same package: use_state_for_unknown.go:50 and :56 (IsUnderlyingValueUnknown), requires_replace_if_configured.go (IsUnderlyingValueNull)
- Impact: For a Computed dynamic attribute whose prior state holds a typed null (provider stored e.g. types.DynamicValue(types.StringNull()); Terraform's dynamic encoding keeps the concrete type of the null), the modifier plans null instead of leaving (known after apply). This is the exact behaviour the modifier exists to avoid (1.17.0 notes: copying nulls causes plan inconsistency errors for child attributes); if apply then returns a non-null value Terraform reports 'Provider produced inconsistent result after apply'. A typed-unknown plan value set by an earlier modifier is also left unknown, unlike the sibling UseStateForUnknown.
- Current: if req.StateValue.IsNull() {return}; if !req.PlanValue.IsUnknown() {return}; if req.ConfigValue.IsUnknown() {return}; resp.PlanValue = req.StateValue. This body is byte-identical (after type-name normalisation) to the 11 static-type packages; it was stamped from one template in commit 18140ec (#1242). For types.Dynamic, IsNull()/IsUnknown() only look at the wrapper: DynamicValue(StringNull()) reports IsNull()=false.
- Replacement: if req.StateValue.IsNull() || req.StateValue.IsUnderlyingValueNull() {return}; if !req.PlanValue.IsUnknown() && !req.PlanValue.IsUnderlyingValueUnknown() {return}; if req.ConfigValue.IsUnknown() || req.ConfigValue.IsUnderlyingValueUnknown() {return} (same shape as the package's UseStateForUnknown). Draft: drafts/D/diffs/D-dyn-nonnull.diff.
- Size: about -3/+6 lines in 1 file (measured on draft) plus 3 test cases
- Risk: low-medium: changes plan output only for dynamic attributes with typed-null state or typed-unknown plan/config values; needs a changelog BUG FIXES entry
- Proof: Verifier: end-to-end through fwserver.PlanResourceChange (go test -overlay=drafts/V-D/d2/overlay-before.json -run TestZZVD2 -v ./internal/fwserver/): typed-null prior gives a planned tftypes.String<null>, untyped-null prior gives <unknown>. With overlay-after.json both give <unknown>. Historical: the v1.15.0 dynamic UseStateForUnknown (the documented 'pre-1.15.1 behavior') has the same three guards as the draft. Add first: the three underlying-value cases from use_state_for_unknown_test.go, with non-null expectations.
- End-to-end reproduction through Terraform was not run (no terraform binary); the modifier-level behaviour is reproduced and the package's own UseStateForUnknown tests model this exact state ('null-underlying-state-value-unknown-plan').
- Keep this fix separate from any refactor of the plan modifier packages (see notes: shared-helper refactor rejected).
- The end-to-end reproduction through the framework server is done; only the final Terraform 'inconsistent result after apply' was not run (no terraform binary). The fix makes the dynamic UseNonNullStateForUnknown code-identical to v1.15.0's dynamic UseStateForUnknown.

### E1. Int32/Int64/Float32/Float64 conversion errors print "%!s(*big.Float=80.5)": %s verb on *big.Float

- Where: types/basetypes/float64_type.go:103,114,151,157; types/basetypes/float32_type.go:93,99; types/basetypes/int32_type.go:82,88,92; types/basetypes/int64_type.go:98,109,142,148; grep -rn 'Value %s' --include=*.go types/basetypes | grep -v _test.go (13)
- Impact: Every practitioner who sets a fractional or out-of-range number on an Int32/Int64/Float32/Float64 attribute, or passes one to a provider-defined function with an Int32/Int64/Float32/Float64 parameter, sees a garbled Go formatting artifact: ValidateResourceConfig -> 'Configuration Read Error ... Error: Value %!s(*big.Float=80.5) is not an integer.'; CallFunction with Int64Parameter and 1.5 -> '... Unable to convert tftypes to framework type at position 0: Value %!s(*big.Float=1.5) is not an integer.' (reproduced). Wrong output, no crash.
- Current: fmt.Errorf("Value %s is not an integer.", bigF) / fmt.Sprintf("Value %s cannot be represented as a 64-bit integer.", value) where bigF/value is *big.Float. big.Float implements fmt.Formatter and supports only b,e,E,f,F,g,G,x,X,p,v, so %s renders as "%!s(*big.Float=80.5)". The text reaches practitioners: ValidateResourceConfig on an Int64Attribute set to 80.5 returns "Configuration Read Error ... Error: Value %!s(*big.Float=80.5) is not an integer."; Int64Type/Float64Type.Validate put the same text in the "Int64/Float64 Type Validation Error" diagnostic Detail.
- Replacement: Pass bigF.String() / value.String() at all 13 sites (keeps exactly the text currently embedded inside the %!s(...) wrapper, e.g. "Value 80.5 is not an integer.", "Value 3000000000 cannot be represented as a 32-bit integer."). Update the 3 literal expectations in int32_type_test.go:51,55,59 and add .String() to the 6 fmt.Sprintf(..., testMustParseFloat(...)) expectations in float64_type_test.go:72,88,176,186 and float32_type_test.go:90,100; add Int64 cases (none exist today).
- Size: about -13/+13 production lines in 4 files, -9/+9 test lines in 3 files (measured on a draft: 13 sites rewritten, exactly the 9 pinned test cases fail until updated)
- Risk: low: user-facing Detail/err text changes (intended). No other package's tests contain the text (grep for '%!s(*big.Float' and 'cannot be represented' outside basetypes: 0).
- Proof: Pinned today (with the wrong text) by TestInt32TypeValueFromTerraform (3 cases), TestFloat64TypeValueFromTerraform, TestFloat64TypeValidate, TestFloat32TypeValueFromTerraform (2 cases each). Reproduced end-to-end through providerserver.NewProtocol6WithError + ValidateResourceConfig (probe drafts/E/probe/e2e) and through tfsdk.Config.GetAttribute (probe drafts/E/probe/bigfmt). Add: TestInt64TypeValueFromTerraform cases for 80.5 and 2^63, and assert the new texts.
- internal/reflect/number.go:43 already formats with result.String(); this is the only family with the bug.
- Alternative formats (%v gives shortest 'g', e.g. 2.147483648e+09) change more text; .String() is the minimal fix.
- Add: big.Float.String() is Text('g', 10), so the fixed text rounds to 10 significant digits: 2147483647.5 renders 'Value 2147483648 is not an integer.' and 12345678901.5 'Value 1.23456789e+10 is not an integer.' (already the text inside today's wrapper, so .String() is still the minimal fix; Text('g', -1)/%v would print the exact value at the cost of '3e+09' for round integers; implementer's choice). Cross-area: the function-argument path also hits area A's missing ': ' separator ('Unable to Convert Function ArgumentAn unexpected error ...').

### E2. Path.Equal and Expression.Equal are asymmetric for zero-value (nil steps) vs empty steps

- Where: path/path.go:105-120; path/expression.go:150-169; affected callers: resource/import_state.go:104,129,137; diag/with_path.go:27; path/paths.go:34,53; path/expressions.go:35,66
- Impact: Equal violates symmetry and the Copy contract; provider code passing an uninitialised path.Path gets a misleading conversion error instead of the intended guard diagnostic, and diagnostic de-duplication depends on append order. Wrong output, no crash.
- Current: if p.steps == nil && o.steps == nil { return true }; if p.steps == nil { return false }; return p.steps.Equal(o.steps) (Expression adds a root check first). PathSteps.Equal is length-based, so: path.Path{}.Equal(path.Empty()) == false but path.Empty().Equal(path.Path{}) == true; zero.Equal(zero.Copy()) == false (Copy turns nil steps into empty); Expression{}.Equal(path.MatchRelative()) == false, reverse true. Consequences (all reproduced): ImportStatePassthroughID(ctx, path.Path{}, ...) skips its "Resource Import Passthrough Missing Attribute Path" guard and fails later with "Value Conversion Error" (same for both guards in ImportStatePassthroughWithIdentity); Diagnostics.Append keeps both of two otherwise-identical attribute diagnostics whose paths are Path{} and Empty() in one order and dedups them in the other; Paths.String does not skip a zero-value Path although its doc says empty paths are skipped; Paths/Expressions.Contains are order-dependent.
- Replacement: func (p Path) Equal(o Path) bool { return p.steps.Equal(o.steps) } and func (e Expression) Equal(o Expression) bool { return e.root == o.root && e.steps.Equal(o.steps) }
- Size: about -30/+2 lines in 2 files (measured on draft3: path.go -13/+1, expression.go -17/+1) plus new test cases
- Risk: low: semantic change limited to zero-value Path/Expression comparisons; all path, diag, tfsdk, internal/fwserver, internal/fwschemadata, internal/fwschema tests pass with the fix (draft3).
- Proof: No test covers nil-vs-empty today (TestPathEqual/TestExpressionEqual only use constructed values). Probe drafts/E/probe/pathequal shows the asymmetry and the ImportStatePassthroughID diagnostic before, and the symmetric results after (probe3 against draft3). Add: TestPathEqual cases zero-empty, empty-zero, zero-copy; TestExpressionEqual zero-relative; a diag Append order test; an ImportStatePassthroughID test with path.Path{}.
- Fixing Copy() to preserve nil instead would not fix Path{} vs path.Empty().
- internal/fwserver NormaliseRequiresReplace (outside area) dedups with Path.Equal after sorting by String; it inherits the fix.
- Add: every user-visible consequence needs provider code to pass a zero-value path.Path/Expression (legal Go zero value; the framework never builds one), so tier 2 is the ceiling and tier 3 would also be defensible. The resource package has no test files: the ImportStatePassthroughID/WithIdentity guards are untested today.

### E3. (*tfsdk.ResourceIdentity).Set and (*tfsdk.Resource).Set panic on a nil receiver while SetAttribute returns a diagnostic

- Where: tfsdk/resource_identity.go:47-58 (Set, no guard) vs :72-81 (SetAttribute guard "Missing Identity Definition"); tfsdk/resource.go:50-61 (Set, no guard) vs :74-83 (SetAttribute guard "Missing Resource Definition")
- Impact: A provider bug (calling resp.Identity.Set on a resource without an identity schema) crashes the plugin process instead of producing the diagnostic that SetAttribute already gives.
- Current: Set calls s.data() (value receiver) through the pointer, so a nil *ResourceIdentity / *Resource panics with "invalid memory address or nil pointer dereference". Reachable: resource Create/Read/Update/Delete/ModifyPlan/ImportState responses carry Identity *tfsdk.ResourceIdentity that the server leaves nil when the resource has no identity schema (internal/fwserver/server_createresource.go:104-123), and list.ListResult.Resource is a *tfsdk.Resource that is nil when a ListResult is built as a literal. resp.Identity.SetAttribute(...) returns the friendly diagnostic; resp.Identity.Set(ctx, &model) crashes the provider.
- Replacement: Add the same nil guard at the top of both Set methods, returning an error diagnostic with the same summary as the sibling SetAttribute guard ("Missing Identity Definition" / "Missing Resource Definition") and a detail that says "set the resource identity" / "set the resource".
- Size: about -0/+20 lines in 2 files (10 per file, measured) plus 2 test cases
- Risk: low: only the previously-panicking path changes. tfsdk and list package tests pass with the guard (draft3).
- Proof: Probe: (*ResourceIdentity)(nil).SetAttribute -> 'Missing Identity Definition'; .Set -> nil pointer dereference; same pair for *Resource. End to end: providerserver.NewProtocol6WithError + ApplyResourceChange, resource without identity schema, Create calls resp.Identity.Set -> plugin process crashes (SIGSEGV, exit 2); with the guard -> ERROR diagnostic, normal return. No test covers either existing guard (grep in tfsdk tests: 0); add nil-receiver cases to TestResourceIdentitySet/TestResourceSet and the SetAttribute tests.
- By the rubric's letter this is a crash (tier 1), but it is only reachable through provider misuse (no identity schema, or a ListResult literal); the framework already treats the same misuse via SetAttribute as worth a diagnostic, so tier 2.
- Get/GetAttribute have value receivers and cannot be guarded the same way; out of scope.

### F3. statestore.NewLockInfo races on a shared math/rand.Rand and can return duplicate lock IDs; use crypto/rand

- Where: statestore/lock_info.go:88-96
- Impact: Data race (go test -race failures in provider suites) and, under sustained concurrent calls, duplicate lock IDs. Duplicates are close to harmless in practice: lock IDs are compared only for the same state, and Terraform serialises Lock calls within a provider process.
- Current: var rngSource = rand.New(rand.NewSource(time.Now().UnixNano())) (math/rand) is shared package-wide, and generateLockID calls rngSource.Read(buf). A *rand.Rand is not safe for concurrent use. NewLockInfo is a public helper for StateStore.Lock implementations (lock.go:9-12, lock_info.go:43-51), and its docs give no concurrency caveat.
- Replacement: Import crypto/rand instead of math/rand, delete rngSource, and write '_, _ = rand.Read(buf)' (since Go 1.24, crypto/rand.Read never returns an error, always fills the buffer and is safe for concurrent use; go.mod is 1.26 and CI also tests 1.25). Keep uuid.FormatUUID(buf).
- Size: about -4/+3 lines (measured on a scratch draft) plus a test (about +25, estimated)
- Risk: low. One behavioural difference worth a line in the PR: crypto/rand.Read terminates the process if the OS RNG fails (legacy Linux without getrandom and without /dev/urandom), whereas math/rand cannot fail; negligible, since go-plugin's TLS setup already depends on crypto/rand.
- Proof: Race reproduced by the verifier (9, 10 and 4 races in three -race runs of 64 goroutines). The duplicate rate did not reproduce at 64 calls (0 of 40 runs); duplicates occur under sustained concurrency: 14-24 per 64,000 IDs (16 goroutines x 4000) and 274-717 per 1.2M IDs (8 x 150,000). With the crypto/rand draft: 0 races, 0 duplicates in 320,000 IDs, go test -race ./statestore/ passes, golangci-lint 0 issues.
- Rejected alternative: uuid.GenerateUUID() from go-uuid, which is already a dependency. It returns an error when crypto/rand fails, and with this function shape an error would yield an empty LockID. The StateStore docs (lock.go:34-36) define an empty LockID as 'locking not supported', so locking would be silently disabled.
- The math/rand + NewSource(time.Now().UnixNano()) seeding is also an outdated idiom (math/rand is auto-seeded since Go 1.20). Terraform core's statemgr has the same pattern; it was copied, not designed for concurrent library use.
- Not a finding, notes only: generateLockInfoWho's function comment ('will return an empty string if any errors are encountered') and its inline comment ('return any information we have') contradict each other and the code. A host error returns the username; a user error returns '' even when the host is known.

### A6. InvokeAction: if the event stream consumer stops, the provider's Invoke blocks forever in SendProgress (goroutine leak)

- Where: internal/proto5server/server_invokeaction.go:85-88; internal/proto6server/server_invokeaction.go:85-88; internal/fwserver/server_invokeaction.go:37-41
- Impact: Precise trigger: at least one SendProgress after the push that returned false, i.e. two or more progress events after a cancel, or any event after a Send error. Code before the blocking call runs; code after it and Invoke's deferred cleanup never run, and the goroutine leaks until the provider process exits (normally at the end of the Terraform command).
- Current: The Events iterator runs fwserver.InvokeAction in a goroutine and relays progress from the unbuffered fwResp.ProgressEvents channel. When push returns false, the iterator returns and nobody receives from the channel again. push returns false whenever tf5/6server's range loop exits: on ctx.Done (StopProvider or cancellation) or a Send error. SendProgress (`r.ProgressEvents <- ...`, no select) then blocks forever. The provider's Invoke never returns and its cleanup never runs, even if it watches ctx.
- Replacement: In the progress case, before returning on !push, start a drain: `go func() { for { select { case <-fwResp.ProgressEvents: case <-completedChan: return } } }()`. Draft: drafts/A/drafts/A6_proto6server_server_invokeaction.go. A fwserver-side alternative is to make SendProgress select on a done channel.
- Size: +14/-0 per package measured on the v6 draft (+28 total); stated +13.
- Risk: low: only runs after the consumer has gone away
- Proof: Probe (sandbox): an action whose Invoke sends 3 progress events, consumed by a push that returns false after the first event. Today Invoke is still blocked 2 s later; with the draft it returns at once and TestServerInvokeAction still passes. Add first: that probe as a regression test in both packages.

### A7. Drift: toproto5.GetProviderSchemaResponse does not return after a list resource schema conversion error (toproto6 and every other loop do)

- Where: internal/toproto5/getproviderschema.go:121-131 (vs internal/toproto6/getproviderschema.go:118-130)
- Impact: Protocol-5 providers with several list resources that use nested attributes get N duplicate error diagnostics (plus the state-store diagnostic) instead of one. Cosmetic: the response already fails.
- Current: All six other loops (provider, provider_meta, action, data source, resource, ephemeral resource), and the same loop in toproto6, append the 'Error converting ... schema' diagnostic and `return protov5`. The v5 list-resource loop only appends, so it keeps converting, emits one diagnostic per failing list resource, and then also runs the state-store check. Both files came from the same commit (ca23057) and v5 never had the return.
- Replacement: Add `return protov5` (after a blank line) inside the `if err != nil` block, as in toproto6.
- Size: about -0/+2 lines (trivial)
- Risk: low
- Proof: Probe (sandbox): 3 list resource schemas with a nested attribute (invalid in protocol 5) give 3 'Error converting list resource schema' diagnostics, while 3 failing resource schemas give 1. No test covers the list error path (`grep -rn "Error converting list resource schema" --include=*_test.go` finds nothing). Add a test like the data-source nested-attribute case in getproviderschema_test.go.
- Classified as drift, not a protocol delta: no reason in the protocol for the asymmetry, and toproto6 has the return.

### A8. Drift: toproto6.SchemaAttribute lacks the nil-Type guard that toproto5.SchemaAttribute and both IdentitySchemaAttribute have, so it panics instead of returning 'must have Type set'

- Where: internal/toproto6/schema_attribute.go:19-31 (vs internal/toproto5/schema_attribute.go:23-25, internal/toproto6/identity_schema_attribute.go:22-24)
- Impact: A provider schema attribute whose GetType() returns nil crashes the provider in GetProviderSchema over protocol 6 (protocol 5 returns a diagnostic). Reachable only by provider misuse: embedding a framework attribute type and overriding GetType to return nil.
- Current: toproto6 evaluates `a.GetType().TerraformType(ctx)` without checking GetType() for nil. toproto5 returns path.NewErrorf("must have Type set") (pinned by the toproto5 test case 'attr-and-nested-attr-unset'), but toproto6 has no such test case and panics with a nil pointer dereference.
- Replacement: Add `if a.GetType() == nil { return nil, path.NewErrorf("must have Type set") }` before the Required/Optional/Computed check, and port the 'attr-and-nested-attr-unset' case to toproto6/schema_attribute_test.go.
- Size: about -0/+4 lines code, +8 lines test (estimated)
- Risk: low
- Proof: Probe (sandbox): toproto6.SchemaAttribute(ctx, "whoops", path, testschema.Attribute{Optional: true}) panics with 'invalid memory address or nil pointer dereference'; toproto5 returns 'must have Type set'. History is truncated at e43fb0b, so when the guard diverged cannot be traced.

### A9. ListResource passes a typed-nil *resourceschema.Schema to the provider when RawV5/RawV6Schemas returns an identity schema but no resource schema; NewListResult then panics

- Where: internal/proto5server/server_listresource.go:67-84; internal/proto6server/server_listresource.go:67-83; internal/fromproto5/resource_schema.go:20-23; internal/fromproto6/resource_schema.go:20-23
- Impact: A provider misconfiguration that validation was meant to reject crashes the provider during ListResource instead of returning a diagnostic.
- Current: The guard checks only `schemaResp.ProtoV6IdentitySchema != nil` and then assigns `req.ResourceSchema, err = fromproto6.ResourceSchema(ctx, schemaResp.ProtoV6Schema)`. For a nil schema (or nil Block) that returns (nil *resourceschema.Schema, nil), which stored in the fwschema.Schema field is a non-nil interface. The v5 comment claims 'There's validation in ListResources that ensures both are set if either is provided'. That is not true: fwserver/server_listresources.go:98-110 validates only when no managed resource of that type exists, and accepts either a complete v5 pair or a complete v6 pair.
- Replacement: Check the converted schema, not only the raw pointers: after fromprotoN.ResourceSchema (and IdentitySchema), treat a nil result (nil raw schema OR nil Block) as a partial pair, i.e. return a 'Converting Resource Schema' diagnostic (or fall back to the framework schemas), and correct the v5 comment. The proposed `ProtoV6Schema != nil && ProtoV6IdentitySchema != nil` guard alone still panics for a schema with a nil Block (reproduced).
- Size: about -2/+2 lines per package plus about 60 lines of new test (estimated)
- Risk: low
- Proof: Probe (sandbox proto6server test): a list resource implementing RawV6Schemas with only ProtoV6IdentitySchema, plus a framework managed resource of the same type, makes the provider's List see req.ResourceSchema != nil, and req.NewListResult panics with 'value method .../resource/schema.Schema.Type called using nil *Schema pointer'. No proto5server/proto6server test exercises the raw-schema branch at all (grep for RawV5Schemas/RawV6Schemas/ProtoV6IdentitySchema in *_test.go finds only fwserver/server_getmetadata_test.go); add one first.
- This is the only real consequence found for the 4 nilnil leads in fromproto5/6 (resource_schema.go:22, identity_schema.go:21): the nil, nil convention itself is intentional; the bug is the caller's one-sided guard.

### A10. Schema conversion errors depend on map iteration order: the same invalid schema produces a different diagnostic (and different partial response) on each run

- Where: Drop internal/fromproto5/resource_schema.go:136 and internal/fromproto6/resource_schema.go:191: their only error, basetypes.TerraformTypeToFrameworkType's 'unsupported tftypes.Type detected', is unreachable (all 9 tftypes kinds are handled and tftypes.Type has unexported methods, so no other implementation can exist). Keep them only as consistency changes if at all.
- Impact: Diagnostics for invalid provider schemas are non-deterministic, which makes provider-side tests of schema errors flaky and repeated runs confusing. No effect on valid schemas.
- Current: Each loop ranges over a map and returns on the first conversion error, so when two or more entries are invalid, the reported attribute, block or resource (and, in GetProviderSchemaResponse, which entries already made it into the maps) depends on Go's randomised map order. Success-path output is deterministic, because the slices are sorted afterwards and the maps are unordered anyway.
- Replacement: Iterate keys in sorted order: `attributes := s.GetAttributes(); for _, name := range slices.Sorted(maps.Keys(attributes)) { attr := attributes[name]; ... }`. Call GetAttributes()/GetBlocks() once, because some schema types build a new map on each call. The post-loop sort.Slice calls then become no-ops and can be deleted (this also supersedes A14).
- Size: +53/-92 measured with numstat on the 6 toproto6 draft files (stated '-74/+47'); with toproto5 about +100/-170 in total (estimated).
- Risk: low: success outputs unchanged (all toproto6 and proto6server tests pass on the draft)
- Proof: Probe: toproto6.Schema on 8 invalid attributes returned 8 distinct error messages over 200 calls; GetProviderSchemaResponse with 5 invalid resource schemas produced 5 distinct outcomes. With the toproto6 draft (drafts/A/drafts/A10_sorted_iteration_toproto6, diff drafts/A/A10_toproto6_sorted.diff): 1 distinct outcome each, and go test ./internal/toproto6 ./internal/proto6server pass.
- Do A7 first (or together): A7 changes the same v5 loop.
- The same map-order effect also decides the order of separate diagnostics in internal/reflect (Map, FromMap, Struct, FromStruct), map/object semantic equality, every schema ValidateImplementation, privatestate, the GetMetadata lists and PlanResourceChange traces (areas B and C). diag.Diagnostics documents its order as unreliable, so only the cases where order changes a single diagnostic's text are filed: A10 and C5.

### A11. ArgumentsData error text is missing its separator: 'Unable to Convert Function ArgumentAn unexpected error ...'

- Where: internal/fromproto5/arguments_data.go:101-104; internal/fromproto6/arguments_data.go:101-104
- Impact: Garbled error text shown to practitioners when a function argument's custom type fails ValueFromTerraform.
- Current: `"Unable to Convert Function Argument"+ "An unexpected error was encountered ..."`. The two sibling messages at lines 71 and 86 use 'Unable to Convert Function Argument: '. A repo-wide scan for a string literal ending in a letter followed by + and a literal starting with a capital finds only these two sites.
- Replacement: "Unable to Convert Function Argument: "+ (optionally also add the 'This is always an issue ...' sentence the siblings carry), plus a test using a parameter whose type's ValueFromTerraform fails (testtypes.InvalidType).
- Size: about -2/+2 lines, +25 lines test (estimated)
- Risk: low
- Proof: Probe (sandbox, custom function.Parameter whose GetType() is testtypes.InvalidType): FuncError.Text = "Unable to Convert Function ArgumentAn unexpected error was encountered when converting the function argument from the protocol type. Please report this to the provider developer:\n\nUnable to convert tftypes to framework type at position 0: intentional ValueFromTerraform error". No test covers this path (grep for 'Unable to convert tftypes to framework type' in *_test.go finds nothing).
- The area E verifier reached this with a stock function.Int64Parameter called with 1.5: an ordinary provider-defined function call shows the run-together text, combined with E1's garbled number.

### A12. WriteStateBytes: framework-subsystem log lines lack tf_state_store_type because the field is set on the root, proto and provider loggers after logging.InitContext created the framework subsystem

- Where: internal/proto6server/server_writestatebytes.go:20-21 (registerContext/InitContext) and 85-87 (SetField calls; 81-84 is the MAINTAINER NOTE).
- Impact: Framework-level trace logs for WriteStateBytes cannot be filtered by state store type, unlike every other state-store RPC.
- Current: The MAINTAINER NOTE says the fields are set here because the type name is only known after the stream is read. tfsdklog.NewSubsystem copies root fields into a subsystem only when it is created (WithRootFields), so the later tfsdklog.SetField on the root logger does not reach the 'framework' subsystem that InitContext created at the top of the handler. The string literal "tf_state_store_type" is also hard-coded three times instead of logging.KeyStateStoreType (same value).
- Replacement: Add `ctx = tfsdklog.SubsystemSetField(ctx, logging.SubsystemFramework, logging.KeyStateStoreType, typeName)` next to the existing three calls, and use logging.KeyStateStoreType in all four.
- Size: about -3/+4 lines
- Risk: low: log fields only
- Proof: Probe (sandbox, tfsdklogtest.RootLogger): during WriteStateBytes, 8 of 9 sdk.framework entries ('Checking StateStoreFuncs lock' ... 'Called provider defined StateStore Write') have no tf_state_store_type (the 9th, 'Found state store', passes it explicitly). For GetStates, where terraform-plugin-go sets the root field before the handler, every framework entry has it. Add first: a _logging test like TestServerGetProviderSchema_logging.

### B2. BlockModifyPlan and BlockValidate never set tf_attribute_path on the framework logger, so block-level plan modifier and validator trace logs carry no path

- Where: internal/fwserver/block_plan_modification.go:25-28; internal/fwserver/block_validation.go:24-25; contrast internal/fwserver/attribute_plan_modification.go:86 and internal/fwserver/attribute_validation.go:61
- Impact: TF_LOG=trace output for block-level validators and plan modifiers cannot be attributed to a block; observability only.
- Current: AttributeModifyPlan and AttributeValidate begin with `ctx = logging.FrameworkWithAttributePath(ctx, req.AttributePath.String())`; their block twins do not, so the 'Calling/Called provider defined planmodifier.List|Object|Set' and 'validator.List|Object|Set' entries for blocks (and the object plan modifiers/validators of nested block objects) have no tf_attribute_path field.
- Replacement: Add the same first statement to BlockModifyPlan and BlockValidate.
- Size: +4 lines (measured)
- Risk: low: logging field only
- Proof: Probe drafts/B1/probes/zz_probe_b1_logpath_test.go (tfsdklogtest capture through ValidateResourceConfig and PlanResourceChange): before, attr-v/attr-pm entries carry tf_attribute_path=attr_list and block-v/block-pm entries carry none; with the fix the block entries carry tf_attribute_path=blk_list. go test ./internal/fwserver/ passes with the fix. No test outside internal/logging asserts tf_attribute_path (grep: 1 hit). Add first: one log-capture test for a block validator and a block plan modifier.
- Attributes nested inside blocks already get their own path from AttributeModifyPlan/AttributeValidate.

### B3. BlockValidate: the early returns of its inline nesting-mode switch skip the 'Block Deprecated' warning whenever block-level validation produced an error

- Where: internal/fwserver/block_validation.go:49-165 (returns at :64, :71, :99, :106, :134, :141, :164); warning at internal/fwserver/block_validation.go:167-174; contrast internal/fwserver/attribute_validation.go:191-201 (AttributeValidateNestedAttributes call, then the warning) with AttributeValidateNestedAttributes at :1002-1165
- Impact: Practitioners do not see a block's deprecation warning while any block-level validation error exists for it; the nested-attribute equivalent is shown.
- Current: BlockValidate inlines a copy of AttributeValidateNestedAttributes' arms and then emits the deprecation warning. The arms return when resp.Diagnostics.HasError() (cumulative, so also after a failing block-level validator), on a value-type mismatch, and in the default arm, which also skips the warning. AttributeValidate keeps the switch in a separate function, so 'Attribute Deprecated' is emitted in the same situation.
- Replacement: Move the switch (from `nestedBlockObject := b.GetNestedObject()` through the closing brace of the switch) verbatim into `func BlockValidateNestedBlockObject(ctx, b, req, resp)` and call it before the warning, mirroring AttributeValidateNestedAttributes (drafted).
- Size: +12/-8 (verifier draft) or +14/-9 (author); same change, different diff hunking. Plus one test case.
- Risk: low: adds a warning that the same configuration already gets when no error occurs
- Proof: Probe drafts/B1/probes/zz_probe_b1_deprecation_test.go (ValidateResourceConfig; a deprecated ListNestedAttribute and a deprecated ListNestedBlock, both with a failing list validator): before, 'Attribute Deprecated' is present and 'Block Deprecated' is missing; with the draft both are present. go test ./internal/fwserver/ passes with the draft; no test combines a block deprecation message with a validation error. Add first: a TestBlockValidate case 'deprecation-message-known-with-validator-error'.
- The comment above the warning ('Show deprecation warning only on known values.') names null/unknown as the only intended suppression.
- A pure refactor that extracts this switch must keep the skip (`if !f(...) { return }`); extracting it as a plain call is this fix, so keep it a separate PR from B6/B7-style work.

### B19. Cached registry diagnostics are returned without copying; lookups append into the shared backing array, so a later lookup rewrites an earlier caller's diagnostics (data race under concurrent RPCs)

- Where: returns of the cached slice: internal/fwserver/server.go:325,369,600,643,654,698; server_actions.go:43,53,94; server_ephemeralresources.go:43,54,95; server_functions.go:129,139,178; server_listresources.go:47,115; server_statestore.go:43,53,94. Appends into it: server.go:306,563,581; server_actions.go:24; server_ephemeralresources.go:24; server_statestore.go:24; server_listresources.go:23; and the two Append calls in each of DataSourceSchemas, ResourceSchemas, ResourceIdentitySchemas, ActionSchemas, EphemeralResourceSchemas, ListResourceSchemas, StateStoreSchemas, FunctionDefinitions
- Impact: Only for a provider whose registry already holds errors (missing or duplicate type names) with spare capacity: concurrent RPCs (Terraform parallelism defaults to 10) can return another request's 'Type Not Found' text, and `go test -race` would report a race. Such a provider already fails GetProviderSchema, so practical impact is low.
- Current: <Concept>Funcs returns s.<x>Diags itself. DataSource/Resource/EphemeralResource/Action/StateStore/ListResourceType then call diags.AddError on it, and the <Concept>Schemas methods call diags.Append on it. When len < cap (3 cached errors give cap 4), the new element is written into the cached backing array, where the next lookup overwrites it.
- Replacement: Return `slices.Clip(s.<x>Diags)` from the 20 return statements (draft drafts/B2/drafts/fixes; with B25 it is one line in factoryRegistry.load).
- Size: 20 return lines + 6 import lines in 6 files (measured on draft)
- Risk: low
- Proof: Probe drafts/B2/repo/internal/fwserver/zz_probe_misc_test.go (aliasing block): a provider with data source type names a, a, b, b and one empty name caches 3 errors with cap 4; d1 := DataSource(ctx, "first") followed by DataSource(ctx, "second") changes d1[3] from 'No data source type named "first"...' to '..."second"...'. With the Clip draft d1 is unchanged. Suites pass on the draft.
- Function lookups convert the cached diags with function.FuncErrorFromDiags and do not append, so they are unaffected.
- With B25 the Clip fix is one line in factoryRegistry.load PLUS the two ResourceBehaviors returns (server.go:600, :643), which stay outside the generic and are also appended into by ResourceBehavior (server.go:581).

### B20. Latent nil dereferences: four handlers dereference what an earlier guard allowed to be nil, and Create/Update/Read/UpgradeResourceIdentity panic when a provider sets resp.Identity to nil

- Where: (1)-(5) as listed; add (6) internal/fwserver/server_upgraderesourceidentity.go:181 (`UpgradeResourceIdentityResponse.Identity.Raw.Type()` after the provider's IdentityUpgrader, which can set resp.Identity = nil)
- Impact: A panic terminates the provider process. (5) turns a provider bug that the framework otherwise reports as 'Missing Resource Identity After ...' into a plugin crash; (1)-(4) are latent.
- Current: Each site panics with a nil pointer dereference instead of returning a diagnostic. (4) also builds the fallback list Config from the managed resource schema: with ResourceSchema set, the provider's List receives Config.Schema == the resource schema (not the list config schema) and a null value of the resource type (hand-off from areas A and F, verified).
- Replacement: (1)-(5) as proposed; (6) `if UpgradeResourceIdentityResponse.Identity == nil || UpgradeResourceIdentityResponse.Identity.Raw.Type() == nil || UpgradeResourceIdentityResponse.Identity.Raw.IsNull()` so the existing 'Missing Upgraded Resource Identity' diagnostic is returned.
- Size: about +11/-7 lines in 8 files (estimated), plus one test case per site
- Risk: low
- Proof: Author probes plus verifier probes drafts/V-B2/repo/internal/fwserver/zz_vb2_nil_test.go ((1)-(4) panic, (4b) provider sees resource/schema.Schema) and drafts/V-B2/repo/internal/proto6server/zz_vb2_test.go TestVB2_B26_5 and zz_vb2_upgid_test.go ((5) Read/Create/Update panic at server_readresource.go:202, server_createresource.go:173, server_updateresource.go:203; (6) UpgradeResourceIdentity panics at server_upgraderesourceidentity.go:181). Reachability: (1)-(4) need a nil request or config that Terraform never sends (ListResource config: core grpc_provider.go:1362); (5) and (6) are reachable from provider code.
- (1) was also noted by area F (notes-F.md:109) and (4) by areas A (notes-A.md:137) and F (notes-F.md:108); filed once here.
- UpdateResource has the same shape as (3) at :216-217, but ApplyResourceChange only calls it with a non-nil PlannedState.
- The area F verifier independently confirmed part (1), the ConfigureProvider nil-request dereference: tier 3, because Terraform never sends a nil request.
- The area A verifier independently reproduced part (4): with no protocol Config, the provider's list config carries the managed-resource schema, and reading the list resource's own attribute fails.

### B21. UpgradeResourceState's same-version passthrough returns stored write-only attribute values that the other two upgrade paths null out; Terraform rejects them

- Where: internal/fwserver/server_upgraderesourcestate.go:86-110 (passthrough, no NullifyWriteOnlyAttributes) vs :229 and :257 (both other paths nullify)
- Impact: A provider that turns an existing attribute into a write-only attribute without bumping the schema version breaks every plan for existing resources with 'Invalid resource state upgrade' until it ships a state upgrader.
- Current: When req.Version equals the schema version the raw state is unmarshalled and returned as is. A value stored for an attribute that is now WriteOnly (an existing attribute converted to write-only without a schema version bump) is returned non-null. Terraform core validates upgraded state: hashicorp/terraform internal/terraform/upgrade_resource_state.go:129-145 'Invalid resource state upgrade ... returned a value for the write-only attribute'.
- Replacement: Run `tftypes.Transform(rawStateValue, NullifyWriteOnlyAttributes(ctx, req.ResourceSchema))` in the passthrough branch with the same 'Error Modifying Upgraded Resource State' diagnostic the other branches use.
- Size: +11/-1 lines in 1 file (measured on a verifier draft), plus one test case
- Risk: low
- Proof: Probe drafts/B2/repo/internal/fwserver/zz_probe_misc_test.go and verifier probe drafts/V-B2/repo/internal/proto6server/zz_vb2_upg_test.go: same-version passthrough returns password="stored-secret", the upgrader path returns null; verifier draft drafts/V-B2/repo-b27 nullifies and passes the three suites. Existing write-only tests cover only the DynamicValue and PriorSchema paths (server_upgraderesourcestate_test.go:362, :602). Core check present since Terraform v1.11.0.
- Confidence medium because the trigger needs that specific provider change; the inconsistency itself is reproduced.

### B22. UpgradeResourceIdentity diagnostics name a method that does not exist: 'UpgradeResourceIdentity()' instead of the interface method UpgradeIdentity()

- Where: internal/fwserver/server_upgraderesourceidentity.go:102 and :125; tests internal/fwserver/server_upgraderesourceidentity_test.go:303, :332, :517; interface resource/resource.go:227-239 (UpgradeIdentity(context.Context) map[int64]IdentityUpgrader)
- Impact: Provider developers are told to implement a method name that no framework interface has.
- Current: Detail text: 'This resource was implemented without an UpgradeResourceIdentity() method, ...' and '...with an UpgradeResourceIdentity() method, ...'. The sibling UpgradeResourceState diagnostics name the real method ('UpgradeState()').
- Replacement: Use 'UpgradeIdentity()' in both details and update the 3 test expectations.
- Size: 2 source lines + 3 test lines (estimated)
- Risk: low: Detail text is observable but only for misconfigured providers
- Proof: grep -rn 'UpgradeResourceIdentity() method' --include=*.go . -> 5 hits (2 source, 3 test); grep -n 'UpgradeIdentity(context.Context)' resource/resource.go -> :239.
- Fold-in when touching the file: the locals `UpgradeResourceIdentityRequest` (:132) and `UpgradeResourceIdentityResponse` (:159) shadow the package's own type names inside the method; renaming them is a pure cleanup.

### B23. Two wrong trace messages in the list resource paths: 'Called ... Configure' is logged before the call, and list registrations log 'Found resource type'

- Where: internal/fwserver/server_listresource.go:114 (should be 'Calling provider defined ListResource Configure', as in server_validatelistresourceconfig.go:43); internal/fwserver/server_listresources.go:76 ('Found resource type' with KeyListResourceType and the comment `// TODO: y?`)
- Impact: Misleading TF_LOG=trace output when debugging list resources.
- Current: ListResource emits 'Called provider defined ListResource Configure' twice around the call; ListResourceFuncs logs 'Found resource type' for list resources, so its trace lines cannot be told apart from ResourceFuncs' identical message.
- Replacement: 'Calling provider defined ListResource Configure' at :114 and 'Found list resource type' at :76 (dropping the TODO).
- Size: 2 lines (estimated)
- Risk: low
- Proof: Read the sites; godox lead server_listresources.go:76; compare the 26 other Configure sites (all log 'Calling ...' first).
- Fix this before applying B24 to ListResource; B24 leaves ListResource out because of this text and its different diagnostics handling.

### C5. Struct/object mismatch diagnostic lists missing field names in random order

- Where: internal/reflect/struct.go:91-108 (Struct) and 198-224 (FromStruct): objectMissing/targetMissing/structMissing are appended while ranging over maps and joined by commaSeparatedString (helpers.go:34) without sorting
- Impact: Non-reproducible error text; provider unit tests that compare diagnostics become flaky.
- Current: With three missing fields on each side the Detail text takes 9 different forms over 200 runs.
- Replacement: slices.Sort each of the lists before commaSeparatedString at the 4 call sites. Do not sort inside commaSeparatedString: TestCommaSeparatedString expects the input order to be kept ('red and blue').
- Size: about +4 lines (estimated)
- Risk: low
- Proof: drafts/C/repros/reflect/zz_repro_order_test.go prints 'Struct (Into) distinct detail texts over 200 runs: 9' and the same for FromStruct. Tests at struct_test.go:81-159, 1352, 1375 use a single field per side. Add first: a 2+2 missing-fields case.
- Ordering of separate diagnostics produced inside map-ranging loops (FromMap, Map, Struct, FromStruct, map/object semantic equality, schema ValidateImplementation) is also random; not reported, see notes-C.md.

### C6. Warning diagnostics dropped: FromUnknownable, FromNullable (null branch) and GetAtPath with a *attr.Value target return nil instead of diags

- Where: internal/reflect/interfaces.go:110 and :151 (FromUnknownable, both branches), interfaces.go:244 (FromNullable null branch); internal/fwschemadata/data_get_at_path.go:38-42 (generic attr.Value target)
- Impact: GetAttribute into an attr.Value (State/Plan/Config) silently drops custom-type validation warnings (reproduced). The FromUnknownable/FromNullable drops have no user-visible effect today because the duplicate validation in the element/field loops and SetAtPath (C3) re-adds them (except function results, where warnings were only logged); they surface as soon as C3 removes that duplicate validation.
- Current: Validation warnings appended to diags are discarded by `return res, nil` / `return nil`; the sibling paths return them (FromNullable non-null branch :285, FromValueCreator :376, GetAtPath with a typed target :56-60).
- Replacement: Return diags at the 4 sites.
- Size: 4 changed lines
- Risk: low
- Proof: drafts/C/repros/reflect/zz_repro_dropwarn_test.go: for the same warning-producing type FromUnknownable(unknown)=0, FromUnknownable(known)=0, FromNullable(null)=0, FromNullable(non-null)=1, FromString=1 diagnostics; drafts/C/repros/fwschemadata/zz_repro_generic_target_test.go: GetAtPath(*attr.Value) diags=0 while ValueAtPath diags=1. Add first: warning cases in TestFromUnknownable, TestFromNullable, TestDataGetAtPath.
- The C10 helper keeps these returns unchanged; fix them in a separate commit.
- Hard prerequisite of C3. The C10 helper keeps these returns unchanged; fix them in a separate commit.

### C7. DataDescription.Title() has no case for DataDescriptionResource, so tfsdk.Resource diagnostics are titled 'Data ... Error'

- Where: internal/fwschemadata/data_description.go:47-62 (Title switch); DataDescriptionResource is used at tfsdk/resource.go:99, internal/toproto5/resource.go:22, internal/toproto6/resource.go:22
- Impact: Misleading diagnostic summary for list-resource results built through tfsdk.Resource.
- Current: Title() covers Configuration, Plan, State, EphemeralResultData and ResourceIdentity; DataDescriptionResource (added with the ListResource RPC, commit 9cafd72) falls to the default 'Data'. tfsdk.Resource.SetAttribute errors therefore read 'Data Write Error' while their detail says 'the resource'.
- Replacement: Add `case DataDescriptionResource: return "Resource"`.
- Size: +2 lines
- Risk: low
- Proof: drafts/C/probe-public TestResourceDescriptionTitle: tfsdk.Resource.SetAttribute(ctx, path.Root("missing"), "x") returns summary "Data Write Error". No test pins it (tfsdk/resource_test.go has no Read/Write Error expectations). golangci exhaustive lead data_description.go:48 points at exactly this case. Add first: a Title() table test over all DataDescription constants.
- The other exhaustive lead in this file (String(), data_description.go:38) is fine: its default returns string(d).

### C8. Provider schema root blocks named count/lifecycle/provider/connection/depends_on/for_each/provisioner are rejected with a resource-schema message

- Where: internal/fwschema/block_validate_implementation.go:39 (ValidateBlockImplementation calls IsReservedResourceAttributeName itself); provider/schema/schema.go:152-160 (provider root blocks: IsReservedProviderAttributeName, then ValidateBlockImplementation)
- Impact: The error text names the wrong schema kind for all 7 names. It is a false positive for provider blocks named connection, provider or provisioner (and likely count, depends_on, for_each, which core reserves as argument names, not block types). For 'lifecycle', Terraform core itself rejects the block in provider configuration ('Reserved block type name in provider block'), so the rejection is right and only the message is wrong.
- Current: Every resource-like schema (resource, datasource, ephemeral, list, action, statestore schema.go) already calls IsReservedResourceAttributeName before ValidateBlockImplementation, so for them the inner call only yields a duplicate that diag.Diagnostics.Append drops. For a provider schema the inner call adds 'When validating the resource or data source schema ... "count" is a reserved root attribute/block name' for root blocks with resource meta-argument names, while root attributes with the same names are accepted (the provider check only reserves alias and version).
- Replacement: Do not just delete the inner call. Move the root reserved-name check fully to the callers and give provider schemas their own block reservation aligned with core's providerBlockSchema (lifecycle; locals is unreserved today, and adding it would be a new error), with the provider wording; resource-like schemas keep IsReservedResourceAttributeName. Confirm intent with maintainers: TestSchemaValidateImplementation 'block-using-reserved-field-name' pins the resource text for a provider 'connection' block.
- Size: about -1/+10 lines plus provider/schema test updates (estimated)
- Risk: low
- Proof: drafts/C/probe-public TestProviderSchemaBlockNamedCount: provider block "count", "lifecycle" or "provider" -> 1 error with the resource message; provider attribute with the same name -> 0. Pinned by provider/schema TestSchemaValidateImplementation case block-using-reserved-field-name, which expects the resource message for a provider block named "connection"; that expectation reads like a copy of the resource test, but confirm intent with maintainers before changing it.
- PLAUSIBLE rather than CONFIRMED only because a test pins the current output.
- Verifier caveat: The bug (wrong message, block/attribute asymmetry) reproduces, but the proposed deletion regresses 'lifecycle' (reserved by core in provider blocks), and a test pins the current output, so intent needs a maintainer decision.

### C9. privatestate diagnostics contain doubled words ('is is', 'a a period') and a missing sentence break

- Where: internal/privatestate/data.go:68-69 and 82-83 ('...private state value.'+'The value associated with key %q is is not valid ...': no separator and 'is is'), 163 and 176 ('is is'), 380 ('does not use a a period'); comment internal/fwschema/schema.go:70-71 ('at the / the given path')
- Impact: Typos in error messages shown to provider developers (Restricted Resource Private State Namespace, Error Encoding/Decoding Private State).
- Current: Provider-developer-facing error details read e.g. 'An error was encountered when validating private state value.The value associated with key "k" is is not valid UTF-8.' and 'Please check the key you are supplying does not use a a period ('.') as a prefix.'
- Replacement: Fix the five strings ('is not valid', 'use a period', add '\n' after 'value.' as NewData already does at :162) and the comment; update the 17 test expectations in internal/privatestate/data_test.go.
- Size: 7 changed lines in internal/privatestate/data.go (68, 69, 82, 83, 163, 176, 380) + 1 comment line in internal/fwschema/schema.go + 21 test lines in data_test.go (17 'is is' or 'a a' expectations plus the 4 lines at 55, 70, 121, 138 that pin the missing newline)
- Risk: low: only internal/privatestate tests pin these strings (grep for 'is is not valid' and 'a a period' finds no other package)
- Proof: golangci dupword: 23 leads in area C, all real: 5 in data.go strings, 17 in data_test.go expectations that pin them (lines 55-56, 70-71, 121-122, 138-139, 328, 339, 350, 361, 372, 383, 394, 405, 502, 664, 735, 751, 959) and 1 comment in fwschema/schema.go:70.
- Cosmetic; bundle as one typo PR.

### D3. SingleNestedAttribute and SingleNestedBlock object validators run twice per validation

- Where: resource|datasource|provider|ephemeral|action|list|statestore single_nested_attribute.go and 6 single_nested_block.go GetNestedObject (lines as cited); internal/fwserver/attribute_validation.go:182 (AttributeValidateObject; cited :181) + :191 -> :1118-1155 (null/unknown return at :1139); internal/fwserver/block_validation.go:44 (cited :43) + :123-155 (no null/unknown guard)
- Impact: An object validator on a single nested attribute runs twice when the configured object is known and the validator (and earlier checks on that attribute) produced no error. On a single nested block it runs twice whenever it produced no error, including null and unknown blocks. A validator that returns an error runs once. Deterministic diagnostics are deduplicated by Append; non-deterministic ones (counters, timestamps) show up twice.
- Current: GetNestedObject() copies the attribute's Validators into the synthesized NestedAttributeObject/NestedBlockObject. fwserver.AttributeValidate runs attribute.ObjectValidators() and then, for NestingModeSingle, NestedAttributeObjectValidate runs nestedObject.ObjectValidators(), which is the same slice. BlockValidate does the same. PlanModifiers are not copied, so plan modifiers run once.
- Replacement: Preferred (internal only): in the NestingModeSingle branches of fwserver.AttributeValidateNestedAttributes and fwserver.BlockValidate, skip the nested object's object-level validator loop (e.g. split NestedAttributeObjectValidate/NestedBlockObjectValidate into a validators part and a children part and call only the children part for single nesting). Alternative: drop the Validators line from the 13 GetNestedObject methods, which changes the value returned by an exported method and needs a changelog note.
- Size: about -4/+16 lines in internal/fwserver/attribute_validation.go and block_validation.go (measured on a verifier draft: drafts/V-D/d3/fix.diff) plus 2 tests
- Risk: low: visible output normally identical; coordinate with area B (plan modification and validation), which owns internal/fwserver
- Proof: Overlay-added test (drafts/D/drafts/D-snvalid/zz_probe_test.go; go test -overlay=drafts/D/drafts/D-snvalid/overlay.json -run TestZZProbe -v ./internal/fwserver/) with a counting validator.Object: 'single nested attribute: object validator invoked 2 time(s); diagnostics=1', control 'list nested attribute: invoked 1 time(s)', 'single nested block: invoked 2 time(s); diagnostics=1'. No existing test counts invocations. Add first: a counting-validator test in internal/fwserver/attribute_validation_test.go and block_validation_test.go asserting exactly 1 call.
- Cross-area: the execution site is internal/fwserver (area B (plan modification and validation)); the cause is the schema packages' GetNestedObject (area D).
- provider/metaschema has no validators, so it is unaffected.
- Keep the existing early return on resp.Diagnostics.HasError() unchanged: it already prevents the second run for error validators, and it also suppresses child validation after a parent error. That is area B (plan modification and validation) behaviour, out of scope for this fix.
- The area B verifier independently reproduced the double run (2 calls per validation for known values, and for a null block) without knowing about this item.

### D4. action/schema SingleNestedAttribute accepts WriteOnly with non-write-only children; its list/map siblings and resource/schema reject this

- Where: action/schema/single_nested_attribute.go:126 (WriteOnly field), :232 (IsWriteOnly returns a.WriteOnly), no ValidateImplementation; siblings: action/schema/list_nested_attribute.go:255, action/schema/map_nested_attribute.go:256, resource/schema/single_nested_attribute.go:325
- Impact: Latent. The framework accepts an action schema that breaks the rule it enforces for the sibling action List/Map nested attributes. Terraform core would accept such a configuration (its ephemeral check walks up to the write-only ancestor). Today no action write-only attribute can be configured non-null at all (see B37), so nothing user-visible changes until B37 is fixed.
- Current: Commit e0355d6 (#1233, 'Add support for write-only attributes in actions') gave List/MapNestedAttribute both the field doc sentence 'If WriteOnly is true for a nested attribute, all of its child attributes must also set WriteOnly to true' and the ValidateImplementation check; SingleNestedAttribute got the field and IsWriteOnly only.
- Replacement: As drafted, but put "context" in the same import group as "fmt". The draft separates them with a blank line, which gofmt accepts but differs from the file's style.
- Size: about -2/+18 lines in 1 file (measured on draft) plus 1 test case
- Risk: low-medium: providers that already declare such a schema would start failing GetProviderSchema; needs a changelog BUG FIXES entry
- Proof: Probe (drafts/D/probe, public API): action SingleNestedAttribute{WriteOnly: true} with StringAttribute{Optional: true} child -> 0 diagnostics; action ListNestedAttribute same shape -> 1 ('"obj" is a WriteOnly nested attribute that contains a non-WriteOnly child attribute.'); resource SingleNestedAttribute -> 1. With the draft overlay: 1, and go test ./action/schema/ passes. action/schema/single_nested_attribute_test.go has no ValidateImplementation test at all (only TestSingleNestedAttributeIsWriteOnly at :500). Add first: TestSingleNestedAttributeValidateImplementation mirroring TestListNestedAttributeValidateImplementation (action/schema/list_nested_attribute_test.go:609).
- fwschema.ValidateAttributeImplementation only calls the attribute's own ValidateImplementation (internal/fwschema/attribute_validate_implementation.go), so nothing else enforces the rule.

### D5. action/schema SetNestedAttribute and SetNestedBlock accept write-only children, although action docs say write-only is unsupported for set-based data and resource/schema rejects it

- Where: action/schema/set_nested_attribute.go:239, action/schema/set_nested_block.go:200; action docs: action/schema/set_attribute.go:188, set_nested_attribute.go:213 ('write-only attributes are not supported for sets and set-based data'); resource analogues: resource/schema/set_nested_attribute.go:299, resource/schema/set_nested_block.go:227
- Impact: Latent. Terraform core (v1.14.0 through v1.17.1) accepts the schema, then rejects at validate time every ephemeral value assigned to such a child ('Invalid use of ephemeral value'), because set element marks are hoisted to the set. The flag is therefore inert, and its doc ('can accept ephemeral values') is misleading.
- Current: Action SetNestedAttribute/SetNestedBlock only check collection-with-dynamic; a child StringAttribute{WriteOnly: true} is accepted.
- Replacement: Add 'if fwschema.ContainsAnyWriteOnlyChildAttributes(a) { Append(fwschema.InvalidSetNestedAttributeWithWriteOnlyDiag(req.Path)) }' and 'if fwschema.BlockContainsAnyWriteOnlyChildAttributes(b) { Append(fwschema.SetBlockCollectionWithWriteOnlyDiag(req.Path)) }', as resource/schema does. Draft: drafts/D/diffs/D-action-wo.diff.
- Size: about +8 lines in 2 files (measured on draft) plus 2 test cases
- Risk: low-medium: the check turns schemas that are accepted today, but whose write-only flag inside a set can never do anything, into a GetProviderSchema error. Today such a child also cannot be set to a non-null value, because ValidateActionConfig rejects it (B37). No working configuration depends on it.
- Proof: Verifier probe drafts/V-D/d45 (action SetNestedAttribute/SetNestedBlock with a write-only child: 0 diags; resource: 1; with the draft overlay: 1). Core evidence: terraform@v1.17.1 internal/configs/configschema/internal_validate.go:100-104 and :182-186 (v1.14.0: :80-84 and :153-157), internal/schemarepo/loadschemas/plugins.go:133-186 (InternalValidate at :134/:143/:167/:178/:184, no action loop, both versions), internal/terraform/node_action.go:205 -> node_resource_validate.go:985 validateResourceForbiddenEphemeralValues (v1.14.0: node_action_validate.go:105 -> node_resource_validate.go:850); go-cty@v1.18.1 cty/value_init.go:264 SetVal (marks hoisted).
- Kept separate from D4 because its correctness depends on an unverified Terraform core rule; D4 only aligns siblings within action/schema.

### E6. Float32Type.ValueFromTerraform debug log fires when float32 and float64 are identical, never when they differ

- Where: types/basetypes/float32_type.go:82-88
- Impact: Misleading DEBUG noise for every exactly-representable Float32 value read from Terraform, and no log for the case it was written to flag. Latent, log-only.
- Current: f, accuracy := bigF.Float32(); f64, f64accuracy := bigF.Float64(); if accuracy == big.Exact && f64accuracy == big.Exact { logging.FrameworkDebug(ctx, "... big.Float value has distinct float32 and float64 representations (float32 value: %f, float64 value: %f)") }. Both conversions exact means both representations equal the value, so the message is false whenever it is logged (e.g. for 1.5 or 3: "(float32 value: 1.500000, float64 value: 1.500000)"), and the genuinely distinct case (0.1 parsed as float64: float32 inexact, float64 exact) logs nothing.
- Replacement: Log when the representations actually differ: `if float64(f) != f64 {` (or `accuracy != big.Exact` if the intent is "float32 lost precision").
- Size: about -1/+1 line
- Risk: low: debug-level log only.
- Proof: No test pins the log (grep 'distinct float32' in tests: 0). Probe drafts/E/probe/f32log with tfsdklogtest: 1.5 and 3 log the "distinct" message; float64(0.1) (acc32=Above, acc64=Exact) logs nothing. Add a tfsdklogtest-based test for 1.5 (no log) and 0.1 (log).
- Add: the message formats with %f, so after the fix 0.1 logs 'float32 value: 0.100000, float64 value: 0.100000'; consider %g/%v when touching the line (optional).

### E7. Paths.String / Expressions.String emit a stray comma when the first element is skipped as empty ("[,a]")

- Where: path/paths.go:47-67; path/expressions.go:60-80
- Impact: Malformed lists in log and error text that print path collections (e.g. validators formatting path.Expressions). Cosmetic.
- Current: The loop skips empty elements but writes the separator based on the element index (`if pathIndex != 0`), so Paths{path.Empty(), path.Root("a")}.String() == "[,a]" and Expressions{path.MatchRelative(), path.MatchRoot("a")}.String() == "[,a]", although the doc says empty paths/expressions are skipped. (Combined with E2, Paths{Root("a"), Path{}} prints "[a,]".)
- Replacement: Collect the non-empty element strings and return "[" + strings.Join(parts, ",") + "]" (same skip test as today).
- Size: about -24/+8 lines in 2 files (measured on draft3)
- Risk: low: logging/error text only.
- Proof: TestPathsString and TestExpressionsString pass with the change (draft3); probe drafts/E/probe/pathsstring shows "[,a]" before and "[a]" after. Add cases empty-first for both types.
- Edge left open: a root expression with zero steps (path.Empty().Expression()) is not equal to Expression{} (root differs) and still renders an empty entry; decide whether "empty" should mean "no steps".
- Add: until E2 lands, zero-value Path elements are not skipped, so Paths{Path{}, Path{}, Root("a")} still prints '[,,a]' after E7 (verified); land E2 first or together.

### E8. ObjectValue.Equal returns true for null/unknown objects with different attribute types

- Where: types/basetypes/object_value.go:300-352
- Impact: Equality is not type-aware for null/unknown objects, unlike every other collection type; latent wrong comparisons in provider code and plan logic that rely on Equal implying identical types.
- Current: Equal compares state first and returns true for two null (or two unknown) objects before comparing attributeTypes, so NewObjectNull({a: String}).Equal(NewObjectNull({b: Bool})) == true even though their Type()s are unequal. Its own doc says it requires "the same value state, and ... exactly the same attribute types/values", the attr.Value.Equal contract asks for type and value equivalence, and List/Map/Set/Tuple all compare element types before state (their null values with different element types are unequal).
- Replacement: Move the attributeTypes length/key/Equal loop above the state comparison (same order as TupleValue.Equal).
- Size: about -8/+8 lines (a block move; measured on draft3) plus test cases
- Risk: medium: Equal results flip (true -> false) for null/unknown objects with different attribute types, and that flips built-in replace planning: dynamicplanmodifier.RequiresReplaceIf and objectplanmodifier.RequiresReplace start requiring replacement when a null/unknown object's attribute types change between state and plan (e.g. a dynamic attribute whose object shape changed); downstream comparisons against types.ObjectNull(nil) also change. Framework suites pass but none covers the case.
- Proof: TestObjectValueEqual has null-null/unknown-unknown cases only with identical attribute types. Probe drafts/E/probe/objequal: ObjectNull(a).Equal(ObjectNull(b)) = true while ListNull/MapNull/SetNull/TupleNull with different element types are false; after the draft fix the object case is false. Add the two different-type cases.
- If maintainers consider the lenient behaviour intended, fix the doc comment instead; either way code and doc disagree today.
- Needs a CHANGELOG bug-fix entry because provider-visible Equal results change.
- Add: Terraform's own cty treats typed nulls as Equal regardless of type (Equals; RawEquals does not), so the lenient behaviour may be intended; present both options to maintainers (fix the code with a CHANGELOG breaking-behaviour note and replace-planning tests, or fix the doc to say attribute types are compared only for known values, which is zero risk).
- Verifier caveat: The code/doc/sibling inconsistency is confirmed, but that the current behaviour is wrong is not demonstrable (cty precedent; no wrong user-visible outcome shown for today's code), and the proposed fix changes replace planning in built-in plan modifiers, which the finding's risk text does not mention.

### E14. tfsdk.ValueFrom discards the warnings returned by reflect.FromValue

- Where: tfsdk/value_from.go:20-25
- Impact: Custom-type validation warnings are silently lost when providers use tfsdk.ValueFrom with Go-native or attr.Value targets.
- Current: v, diags := reflect.FromValue(ctx, targetType, val, path.Empty()); if diags.HasError() { return diags }; return ValueAs(ctx, v, target). Warnings in diags (custom type ValidateAttribute / TypeWithValidate warnings) are discarded.
- Replacement: diags.Append(ValueAs(ctx, v, target)...); return diags
- Size: +2/-1 lines plus a TestValueFrom warning case
- Risk: low
- Proof: Public probe drafts/V-C/probe c6b_test.go: tfsdk.ValueFrom(ctx, "a", vStrType{}, &s) with s a string calls the validator once (it emits a warning) and returns 0 diagnostics; the same happens with an *attr.Value target. With a validating target (the custom value type) ValueAs re-validates, so the warning reappears. TestValueFrom (tfsdk/value_from_test.go) has no warning cases. Add first: a warning case with a string target.
- Same class as C6; not reported by any other area (grep of findings/*.json for value_from.go: none).
- Found by the independent verifier of area C while checking that area.
- Re-checked independently by the verifier of area M: confirmed.

### F4. A list resource for a framework resource without identity passes GetProviderSchema, then panics in list.ListRequest.NewListResult (nil identity schema)

- Where: list/list_resource.go:135-151 (panics at :137); internal/fwserver/server_listresources.go:98-110 (checks only that a framework resource of the same type exists); internal/fwserver/server.go:826-837 (returns a nil identity schema without error for resources without identity); internal/proto6server/server_listresource.go:90 (twin internal/proto5server/server_listresource.go:91) passes the nil schema on to list.ListRequest
- Impact: Not reachable from Terraform: core v1.14.0 and main refuse to send ListResource when the managed resource type has no identity schema and report 'Identity schema not found for resource type <type>; this is a bug in the provider - please report it there'. The framework only fails to diagnose the misconfiguration (GetProviderSchema, GetMetadata and ValidateListResourceConfig return no diagnostics); the NewListResult panic is reachable only by direct ProviderServer callers such as provider unit tests driving providerserver.
- Current: If a ListResource matches a framework Resource that does not implement resource.ResourceWithIdentity, GetProviderSchema returns 0 diagnostics. In the ListResource RPC, ListRequest.ResourceIdentitySchema is nil, and the documented constructor req.NewListResult(ctx) calls r.ResourceIdentitySchema.Type() on a nil interface. Non-framework resources must supply ProtoV5/V6IdentitySchema (server_listresources.go:100-108), so identity is clearly required; the requirement just is not checked for framework resources.
- Replacement: Prefer a per-RPC guard that changes only the call that panics today: in proto5/6server.ListResource's framework branch (after ResourceIdentitySchema, proto6 :90 / proto5 :91) return ListRequestErrorDiagnostics with an error that names the missing resource.ResourceWithIdentity when the schema is nil, or the same guard in fwserver.ListResource before calling List. If registration-time strictness is wanted, emit a warning from ListResourceFuncs (or an error only with a breaking-change changelog entry), and skip it when the list resource supplies a raw ProtoV5/V6 identity schema.
- Size: about +15 lines in internal/fwserver/server_listresources.go plus a test (estimated)
- Risk: medium for the registration-time variant: an error from ListResourceFuncs surfaces in GetProviderSchema (server_getproviderschema.go:84-87), which Terraform treats as fatal for every command, so a provider that today works for plan/apply and only fails `terraform query` on that type would break entirely after upgrading. low for a per-RPC guard.
- Proof: Probe in drafts/F/probe-listnoident: a provider with Resource p_thing (no identity) and ListResource p_thing whose List calls req.NewListResult(ctx). GetProviderSchema returns 0 diagnostics and 1 list resource schema. Calling ListResource through tfprotov6.ProviderServerWithListResource and iterating Results panics with 'invalid memory address or nil pointer dereference' at list.ListRequest.NewListResult list_resource.go:137. Test to add first: an fwserver ListResourceFuncs/GetProviderSchema test with this provider, expecting the new error diagnostic.
- Rejected: a nil guard in NewListResult alone. Typical provider code next calls result.Identity.Set(...) on the nil *tfsdk.ResourceIdentity (see internal/proto6server/server_listresource_test.go:108-111), so the panic just moves into provider code.
- Not verified: whether Terraform core would send ListResource for a type without an identity schema (no core source here). The reproduction is at the framework RPC boundary.
- Replace the note 'Not verified: whether Terraform core would send ListResource...' with: verified, core does not send it (see impact).

### F5. Definition.ValidateImplementation checks for a nil Return type but not for nil parameter types: a custom Parameter with GetType() == nil panics in GetProviderSchema

- Where: Add: the GetFunctions RPC panics the same way (internal/toproto6/getfunctions.go:26 -> toproto6.Function -> FunctionParameter function.go:56; v5 twin).
- Impact: A custom Parameter implementation with an undefined type crashes the provider in GetProviderSchema. This is latent: custom Parameter implementations are rare.
- Current: 'that guard can never run' should read 'that guard can never run for Terraform callers' (Terraform always calls GetProviderSchema first; direct callers can reach it).
- Replacement: In the Parameters loop and in the VariadicParameter branch, add an error diagnostic when GetType() == nil, worded like the Return check, for example "Function %q - Parameter at position %d data type is undefined" and "Function %q - Definition variadic parameter data type is undefined".
- Size: about +18 lines (measured on a scratch draft) plus 2 test cases in TestDefinitionValidateImplementation (about +40, estimated)
- Risk: low
- Proof: Probe in drafts/F/probe-nilparam: a custom Parameter whose GetType() returns nil, used in a provider function. providerserver GetProviderSchema panics at toproto6.FunctionParameter function.go:56 <- toproto6.Function function.go:33 <- toproto6.GetProviderSchemaResponse getproviderschema.go:87. With the draft check, GetProviderSchema returns 1 error diagnostic and go test ./function/ passes.
- Add: custom Parameter implementations also skip the empty-name check (only the 12 built-in types call MissingParameterNameDiag); an optional extension of the same Definition check (see F9 correction).

### F6. Two misuse paths in function/ panic instead of returning a FuncError: ArgumentsData.GetArgument with a negative position, and ResultData.Set on a zero ResultData

- Where: function/arguments_data.go:128 (guard is only position >= len(d.values)); function/result_data.go:34-35 (d.value.Type(ctx) with nil d.value)
- Impact: A provider programming error crashes the provider instead of returning a function error. Latent: in production fwserver always supplies a non-nil ResultData, so the zero-value case is mostly a unit-test pitfall.
- Current: GetArgument(ctx, -1, &t) panics with 'index out of range [-1]', while GetArgument past the end returns the documented 'Invalid Argument Data Position' FuncError. (*ResultData).Set on a zero ResultData panics with a nil pointer dereference; one example is resp.Result.Set on a bare function.RunResponse{} in a provider unit test. ArgumentsData, by contrast, already returns 'Function does not have argument data.' for its empty case.
- Replacement: arguments_data.go: check position < 0 before the empty-data branch (:118-126) and return NewFuncError (no argument position) for it; keep 'position >= len(d.values)' with NewArgumentFuncError. Widening only the :128 guard leaves GetArgument(-1) on empty data returning FunctionArgument=-1. result_data.go Set: return NewFuncError('Invalid Result Data Usage: ...') when d.value == nil.
- Size: about -1/+10 lines plus 2 test cases (estimated)
- Risk: low
- Proof: Probe in drafts/F/probe-args: GetArgument(1) on one argument returns the 'Invalid Argument Data Position ... Given argument position: 1, last argument position: 0' FuncError. GetArgument(-1) panics with 'runtime error: index out of range [-1]'. Both new(ResultData).Set and RunResponse{}.Result.Set panic with 'invalid memory address or nil pointer dereference'. Tests to add first: cases in TestArgumentsDataGetArgument and a new TestResultDataSet zero-value case.

## Looks equivalent but is not

- **Generic ArgumentsData[T interface{ Unmarshal(tftypes.Type) (tftypes.Value, error) }] instead of a callback**: Calling the value-receiver Unmarshal through a type parameter on a nil *DynamicValue panics with 'value method github.com/hashicorp/terraform-plugin-go/tfprotov5.DynamicValue.Unmarshal called using nil *DynamicValue pointer' instead of 'runtime error: invalid memory address or nil pointer dereference' (verifier probe, Go 1.26). (internal/fromproto5|6/arguments_data.go:81 (A2))
- **Moving MoveResourceState's first HasError check up between the ResourceSchema and ResourceIdentitySchema lookups**: When the resource-schema lookup errors, the identity-schema lookup and its diagnostics would be skipped, changing the response diagnostics; only deleting the duplicate second check (A13) is equivalent. (internal/proto5server/server_moveresourcestate.go:35-49; internal/proto6server/server_moveresourcestate.go:34-48)
- **Deleting registerContext and relying on terraform-plugin-go's stoppableContext**: stoppableContext exists only inside tf5server/tf6server's gRPC wrapper; direct ProviderServer callers (tests, providerserver.NewProtocol5/6 consumers) would lose StopProvider cancellation, which TestServerCancelInFlightContexts pins. (internal/proto5server/serve.go, internal/proto6server/serve.go (A5))
- **Sorted-key iteration of schema maps presented as a pure refactor**: It changes which conversion error is reported when two or more entries are invalid (2 distinct errors in 50 runs become 1), so it is a bug fix (A10); only success-path output is byte-identical. (internal/toproto5|6 schema.go, block.go, schema_attribute.go, identity_schema.go, getproviderschema.go, getresourceidentityschemas.go)
- **Hoisting the ReadStateBytes ChunkSize check above the provider Read call**: The provider's Read runs at server_readstatebytes.go:43 before the ChunkSize check at :56; hoisting the check would skip a provider call (side-effect order changes). (internal/proto6server/server_readstatebytes.go:43-56)
- **Build the trace message at run time in the generic bodies ('Calling provider defined planmodifier.'+t.name) instead of passing a constant**: Output is identical, but the result escapes and costs 2 heap allocations per provider modifier/validator call, even with logging off (5 -> 9 and 3 -> 7 allocs/op with 2 modifiers/validators). Literal per-spec messages restore parity. (B4/B5/B6 drafts (value_plan_modification.go:149/159, value_validation.go:91/101))
- **Reach NestedAttributeObjectPlanModify/NestedBlockObjectPlanModify, AttributeModifyPlan/BlockModifyPlan through a func value in the extracted helpers**: Output is identical, but escape analysis cannot see through the indirect call, so the response struct passed to it moves to the heap: +1 allocation per nested object (B7/B8), per nested child (B9) and per root attribute/block (B10). Direct dispatch keeps parity (verified for B7). (B7..B10 drafts: singleNestedPlanModify, nestedObjectElementPlanModify, nestedPlanModify, schemaModifyPlanAtRoot)
- **Pass attribute.<T>PlanModifiers() (the slice) to the generic instead of the method value**: Calls the getter before the coercions, and also when they fail (author's rejection; endorsed by reading) (B4/B6)
- **Share BlockValidate's single arm with AttributeValidateNestedAttributes'**: Adds or drops the null/unknown guard. A null SingleNestedBlock with a Required child yields 'Missing Configuration for Required Attribute' while the attribute yields nothing; this is intended per #740 (reproduced by verifier probe zz_vb1_singlenull_test.go). (internal/fwserver/block_validation.go:123-155 vs attribute_validation.go:1118-1154)
- **Extract BlockValidate's nesting switch as a plain call inside a pure refactor**: That extraction is the B3 behaviour change ('Block Deprecated' then appears after block-level errors); a pure refactor must keep 'if !f(...) { return }' (internal/fwserver/block_validation.go:49-174)
- **Unify the Single arm with the collection arms (merge nested diagnostics/Private/RequiresReplace before vs after re-wrapping), or precompute the state lookup in B8, or add the #993 guard to blocks while deduplicating (B9)**: Each changes which diagnostics are reported, or changes block behaviour (author's rejections; endorsed by reading) (internal/fwserver/attribute_plan_modification.go:650-675 vs :242-266; B8; B9)
- **Configure helper that returns only diagnostics, with an unconditional `if resp.Diagnostics.HasError() { return }` at each call site (B24 variant)**: If resp already holds an error on entry and the type does not implement <Concept>WithConfigure, today's code continues and the variant returns; B24's (diags, ok) form keeps the HasError check inside the ok branch. (internal/fwserver, the 26 Configure preambles)
- **Accumulating Validate*Config validator diagnostics locally and appending once (B28 variant)**: Diagnostics.Append de-duplicates through Diagnostic.Equal against what is already in resp; local accumulation changes which pairs are compared and in what order, which is equivalent only if every provider-supplied Diagnostic's Equal is an equivalence relation. (internal/fwserver/server_validate*config.go)
- **Sorting the GetMetadata type-name slices (slices.Sorted(maps.Keys(funcs)))**: Behaviour change, not a refactor: today the order comes from map iteration; tests sort before comparing (fwserver 7, proto5server 7, proto6server 8 sort.Slice calls; both proto5server :364/:372 and proto6server :419/:427 sort ListResources twice). (internal/fwserver/server*.go <Concept>Metadatas)
- **Building ResourceBehaviors in the same pass as ResourceFuncs**: ResourceBehaviors (server.go:594-645) instantiates every resource again, calls Metadata again and re-emits 'Found resource type' traces in map order; merging changes the provider call count and the log sequence. (internal/fwserver/server.go:594-699)
- **Caching an empty ProviderTypeName**: ProviderTypeName only caches a non-empty name (server.go:480-494); caching '' changes the 'Checking ProviderTypeName lock'/'Calling provider defined Provider Metadata' sequence pinned by TestServerGetProviderSchema_logging (6 occurrences each in proto6server). (internal/fwserver/server.go:475-495)
- **MoveResourceState `resp.Diagnostics = moveStateResp.Diagnostics` -> Append (looks like B17)**: Intended and pinned by test 'response-Diagnostics-first-error-always-responds' (server_moveresourcestate_test.go:612). (internal/fwserver/server_moveresourcestate.go:224, :243)
- **errors.Is(err, tftypes.ErrInvalidStep) for the errorlint lead**: Not a fix and not a modernisation: tftypes v0.31.0 returns the sentinel unwrapped (attribute_path.go:364-366, value.go Value.ApplyTerraform5AttributePathStep) and its own diff.go:97/:117 compares with !=; identical today. (internal/fwserver/server_planresourcechange.go:451)
- **Comparing B18 log output byte-for-byte between variants**: tftypes.Transform walks object attributes and map elements in Go map order (walk.go:263, :295), so even the unchanged code emits a different line order run to run; equivalence must be checked on sorted lines plus values. (internal/fwserver/write_only_nullification.go:24, server_planresourcechange.go:416)
- **privatestate ProviderData.Equal: reflect.DeepEqual(d.data, o.data) -> maps.EqualFunc(d.data, o.data, bytes.Equal)**: Endorsed with a probe (drafts/V-C/convprobe deq_test.go): nil vs empty map gives DeepEqual=false, EqualFunc=true; {k:nil} vs {k:[]byte{}} gives DeepEqual=false, EqualFunc=true. fwserver tests build &privatestate.ProviderData{} (nil map) and compare via cmp.Diff, which calls Equal. (internal/privatestate/data.go:267)
- **BuildValue kind-switch arms `val, valDiags := X(...); diags.Append(valDiags...); return val, diags` -> `return X(...)`**: Endorsed as a conservative rejection, but the difference is theoretical today: diags is always empty at the switch (NewUnknownable/NewNullable add only errors and return) and no callee returns an empty non-nil, nil-containing or duplicate Diagnostics; line 168 already uses `return Number(...)`. Not provably equivalent by construction (Append filters nil and duplicates and keeps nil for an empty input). (internal/reflect/into.go:170-202)
- **C10 helper for FromAttributeValue with an eagerly computed tfVal**: Endorsed: the original calls val.ToTerraformValue only in the TypeWithValidate branch, and its error returns early; eager computation adds a call, and a new error path, for every attr.Value. (internal/reflect/interfaces.go:458-488)
- **Removing the element-loop / SetAtPath re-validation as a pure refactor**: Endorsed, with a stronger reason: besides call counts and set paths, it drops warnings for Unknownable and Nullable(null) values, because FromUnknownable/FromNullable discard their own validation diagnostics (C6). Probe: ListValueFrom(vStrType, []*uStr{{S:"a"}}) gives 1 warning, 0 after removal. (internal/reflect/slice.go:262,352; map.go:178; struct.go:276; internal/fwschemadata/data_set_at_path.go:118-153)
- **One semantic-equality helper for scalar and nested types (scalar = nested with a no-op nested function)**: Endorsed: nested functions return on resp.Diagnostics.HasError() before applying usePriorValue (value_semantic_equality_list.go), scalar functions apply the prior value even after an error diagnostic. (internal/fwschemadata/value_semantic_equality_*.go)
- **Generic TransformDefaults helper over the typed defaults Request/Response**: Endorsed: Go does not allow field access (Path, PlanValue, Diagnostics) through a type parameter, even when every type in the set has the field. (internal/fwschemadata/data_default.go:102-396)
- **Method expression basetypes.BoolValuableWithSemanticEquals.BoolSemanticEquals as the C15 callback**: Endorsed, with an additional reason: the method expression's signature puts the receiver first, func(BoolValuableWithSemanticEquals, context.Context, BoolValuable), so it matches neither the parameter order nor the parameter type of func(ctx, V, V). (internal/fwschemadata/value_semantic_equality_bool.go)
- **Sort inside commaSeparatedString to fix C5**: Endorsed: TestCommaSeparatedString expects input order ('red and blue' for [red blue]; 'red, blue, and green'). (internal/reflect/helpers.go:34-46)
- **One helper for the 3 fwschemadata validation blocks**: Endorsed: data_set_at_path.go:294/298 logs 'Calling/Called provider defined Type ValidateAttribute' where data_value.go:117/121 and data_set_at_path.go:143/147 log 'Type Validate'. (internal/fwschemadata/data_value.go:92; data_set_at_path.go:118,269)
- **Fold FromValueCreator into C13's fromPrimitive**: Endorsed: FromValueCreator calls typ.TerraformType(ctx) twice (interfaces.go:338,342); the helper would call it once. (internal/reflect/interfaces.go:330-376)
- **C1 map fix: reject every map target whose key Kind is not String before iterating**: New: map[any]T and map[EmptyIface]T targets work today (string is assignable to the key) and an empty Terraform map read into map[int]T succeeds today; both become errors (probe c1b_test.go). Gate on assignability and on len(values) > 0. (internal/reflect/map.go:52-90)
- **C2 fix: send every nil pointer to FromPointer before the attr.Value/ValueCreator/Unknownable/Nullable dispatch**: New: nil-safe pointer-receiver implementations do not panic today; ListValueFrom(StringType, []*nilSafeU{nil}) is <unknown> today and would become <null>. Gate on the element type implementing the interface (value receivers). (internal/reflect/outof.go:22-36)
- **C4/C11: reuse fwschema.ValidAttributeNameRegex inside internal/reflect**: New: does not compile: import cycle internal/reflect -> internal/fwschema -> types/basetypes -> internal/reflect. (internal/reflect/helpers.go:135-138)
- **C8 fix: delete the IsReservedResourceAttributeName call from ValidateBlockImplementation**: New: equivalent for resource-like schemas, but for provider schemas it also stops rejecting a 'lifecycle' block, which Terraform core reserves in provider blocks (providerBlockSchema), so the framework would accept a schema practitioners cannot configure. (internal/fwschema/block_validate_implementation.go:39)
- **Move the common fields (Required, Optional, Description, ...) into an unexported embedded struct**: Keyed composite literals stop compiling ('unknown field Required in struct literal of type lib.BoolAttributeEmbedded'), and every provider writes schema.BoolAttribute{Required: true}. (all attribute structs in the 9 schema packages (verifier probe drafts/V-D/neq/user1))
- **Keep every exported field and add an unexported embedded helper struct**: reflect NumField changes (3 -> 4), and a bare cmp.Diff on the struct panics on the unexported field. The repo's tests and provider tests do bare cmp.Diff on these structs. (attribute structs (verifier probe drafts/V-D/neq/user2))
- **Generic type alias, e.g. type BoolAttribute = primitiveAttribute[...]**: %T changes to lib.primitive[bool]. A generic type cannot have per-instantiation method names (BoolValidators, BoolDefaultValue, ...) that the fwxschema interfaces require. (primitive attribute types (verifier probe drafts/V-D/neq/user2))
- **GetType via a value-taking helper customTypeOr(a.CustomType, <default type>)**: The default type is evaluated eagerly. For SingleNestedAttribute that calls GetType() on every child, which panics on a nil child entry even when CustomType is set; today it returns CustomType without touching the children. (the 63 non-primitive GetType bodies (verifier probe: eager panics=true, today returns 'custom'))
- **D6: delete the OUTER non-computed-default check instead of the inner duplicate**: Moves the default diagnostic after the write-only diagnostic (probe: [Invalid Schema Implementation, Schema Using Attribute Default...] instead of the reverse). (resource/schema/single_nested_attribute.go:321-336)
- **D3: drop the attribute-level object-validator run (AttributeValidateObject/BlockValidateObject) instead of the nested-object run**: The single-nesting attribute branch returns early for null and unknown objects (attribute_validation.go:1139), so validators on null or unknown single nested attributes would never run, and their diagnostics would be lost. (internal/fwserver/attribute_validation.go:182 vs :1152)
- **D1: compare the Default only against types.ObjectType{AttrTypes: a.AttributeTypes} (author's fix), or guard with a.CustomType == nil (author's alternative)**: With CustomType set, the effective type is CustomType. The first form accepts a default that matches AttributeTypes but not the custom type, and loses today's correct rejection for custom types with a non-strict Equal (probe case 8). The second form drops type validation for every custom type (cases 5, 6, 8). (resource/schema/object_attribute.go:314)
- **maps.Clone(m.elements) / maps.Clone(o.attributes) for MapValue.Elements and ObjectValue.Attributes (also AttributeTypes)**: maps.Clone(nil) returns nil; NewMapNull/NewObjectNull leave the field nil, so null/unknown values would return a nil map instead of an empty non-nil one. Drafted: TestMapValueElements/{null,unknown} and TestObjectValueAttributes/{null,unknown} fail (the suite protects this one). Use maps.Copy into the existing make. (types/basetypes/map_value.go:173-182; object_value.go:234-254; object_type.go:41-50)
- **slices.Clone(l.elements) for List/Set/Tuple Elements()**: slices.Clone(nil) returns nil (Go 1.22+ preserves nilness) while the current make+append returns an empty non-nil slice for null/unknown values. Drafted for ListValue: TestListValueElements/{null,unknown} fail. (types/basetypes/list_value.go:168-174; set_value.go:168-174; tuple_value.go:126)
- **Diagnostics.Errors()/Warnings() as slices.DeleteFunc(slices.Clone(diags), ...)**: A nil receiver returns nil instead of Diagnostics{} (json.Marshal 'null' vs '[]', reflect.DeepEqual false). The diag suite is blind to it: drafted, `go test ./diag/` passes, because cmp.Diff uses the len-based Diagnostics.Equal. (diag/diagnostics.go Errors()/Warnings())
- **new(12.3) / new(12) / new(0) in place of newFloat32Pointer(12.3) / newInt32Pointer(12) / newFloat64Pointer(0)**: Untyped constants take their default type: new(12.3) is *float64, new(12) and new(0) are *int (probed on Go 1.26.0); the table fields are interface{}, so it compiles and the test silently stops exercising float32/int32/int64. Needs new(float32(12.3)), new(int32(12)), and keeping 0.0 for float64. (tfsdk/value_as_test.go:112-183, 351)
- **new(new(new(big.Float).SetUint64(n))) for newBigFloatPointerPointer(n) (mechanically applying 'PointerPointer -> new(new(...))')**: The helper's base value is already *big.Float, so it needs one extra new: the triple form is ***big.Float, compiles (interface{} field) and fails only via the test comparer's type assertion panic. Correct: new(new(big.Float).SetUint64(n)). Found while drafting E13. (tfsdk/value_as_test.go:69-72 (helper), 172-173 (calls))
- **Generic core struct embedded in BoolValue/StringValue/... so IsNull/IsUnknown are promoted**: Moves the methods one embedding level deeper. Compile probe: a downstream type embedding StringValue next to a type that reaches IsNull at depth 2 compiles today and fails with 'ambiguous selector MyString{...}.IsNull' under the embedded variant. E5's free generic functions avoid it. (types/basetypes/{bool,int32,int64,string,float32,float64}_value.go (rejected E5 variant))
- **Paths/Expressions.String separator via result.Len() > 1 (E7 alternative)**: Root("") renders '' but is not skipped: today Paths{Root(""), Root("a")} prints '[,a]' (verified in the E7 matrix, unchanged by collect+join), whereas Len-based separators would print '[a]'. (path/paths.go:47-67; path/expressions.go:60-80)
- **Fixing E2 by making Copy() preserve nil steps**: Does not touch Path{}.Equal(path.Empty()), which is false because Empty() has non-nil empty steps regardless of Copy; the ImportStatePassthroughID guard and Append order-dependence remain. (path/path.go:99-120)
- **Embed a shared unexported base struct in the 12 typed Parameter (or Return) structs to share the getters**: Promoted fields cannot be named in composite literals: Go 1.26.0 rejects BoolParameter{Name: "x"} with 'unknown field Name in struct literal of type BoolParameter' (verifier compile probe drafts/V-F/probe-f11lit); it also changes the reflect-visible field set. (function/*_parameter.go, function/*_return.go)
- **Pointer receiver for ResultData.Equal (recvcheck lead)**: go-cmp uses an Equal method only when it is in the value's method set; with a pointer receiver, cmp.Diff on a struct holding a ResultData panics 'cannot handle unexported field' (verifier probe drafts/V-F/probe-cmp). (function/result_data.go:24)
- **Nil guard in list.ListRequest.NewListResult as the F4 fix**: NewListResult would return a nil Identity, and typical provider code calls result.Identity.SetAttribute(...) next (the verifier's F4 probe does), so the panic moves into provider code. (list/list_resource.go:135)
- **F4 fix as an error diagnostic in fwserver.ListResourceFuncs**: That error surfaces in GetProviderSchema (server_getproviderschema.go:84-87), which Terraform treats as fatal for every command. Core already blocks `terraform query` for such types, so providers that work today for plan/apply would fail entirely after a framework upgrade. Use a per-RPC guard or a warning. (internal/fwserver/server_listresources.go:98-110)
- **F1: always take the element type from the CustomType**: Changes behaviour when ElementType and CustomType are both set and disagree; the nil-only guard keeps every ElementType-set case byte-identical (verifier probe CONTROL lines identical before and after). (function/{list,set,map}_return.go NewResultData)
- **Declare list.NoListResults as iter.Seq[ListResult]**: Changes the exported variable's type from an unnamed func type to the named iter.Seq, which is visible through %T and reflect; not a refactor in a public package. (list/list_resource.go:170)
- **Run `go fix -newexpr ./...` alone and commit**: It rewrites every caller but keeps the 13 helpers as //go:fix inline wrappers, so golangci `unused` (enabled) fails CI on them. I reproduced this: staticcheck U1000 x13, and golangci-lint on ./function ./tfsdk reports 8 unused. The helpers must be deleted, and -inline must be run for the typed helpers. (function|internal/fwschemadata|internal/toproto5|internal/toproto6|tfsdk|types/basetypes/pointer_test.go:7; internal/reflect/pointer_test.go:180; tfsdk/value_as_test.go:25-80)
- **wg.Add(1); go func(){ defer wg.Done(); f() }()  ->  wg.Go(f), treated as identical in all cases**: In Go 1.26, WaitGroup.Go recovers a panic from f, re-panics and skips Done (src/sync/waitgroup.go), while the deferred Done runs during the panic. The difference is only crash timing, so it is equivalent for bodies that cannot panic, as in serve_test.go here. (internal/proto5server/serve_test.go:29-43; internal/proto6server/serve_test.go:28-42)
- **[]byte(fmt.Sprintf(f, ...)) -> fmt.Appendf(nil, f, ...)**: This returns nil instead of []byte{} when the formatted result is empty, which json.Marshal and cmp.Diff can see. It is safe here only because all 6 formats are non-empty literals. (internal/privatestate/data_test.go:49,114,233,263,758,775)
- **Merge testschema NestedObjectCustomType / ListNestedObjectCustomType / MapNestedObjectCustomType / SetNestedObjectCustomType (byte-identical modulo the name; normalized diff is empty)**: fwserver tests nest two of them (outer ListNestedObjectCustomType, inner NestedObjectCustomType) to prove that each nesting level keeps its own CustomType, and distinct Go types are what make cmp.Diff catch a swap. Merging them weakens those tests. (internal/testing/testschema/nested_attribute_object_*custom_type.go; internal/fwserver/attribute_plan_modification_test.go:244-320)
- **Fold Float32/Float64/Number/Dynamic Equal into one primitive Equal helper**: Known-with-nil-value semantics differ (float: nil equals nil; number: Cmp on nil panics; dynamic: false) and *big.Float needs Cmp, not ==. (types/basetypes)
- **One shared helper for the New<Collection>ValueMust panics and <Collection>ValueFrom diagnostics**: The observable wording differs per type ('NewListValueMust' and 'NewTupleValueMust' but 'MapValueMust', 'ObjectValueMust', 'SetValueMust'; 'using NewListValueFrom' but 'using MapValueFrom'). A helper must take the exact text as a parameter (E10 does). (types/basetypes/*_value.go)
- **statestore.NewLockInfo via uuid.GenerateUUID()**: Its error path yields an empty LockID, which the StateStore docs define as 'locking unsupported'. crypto/rand.Read plus uuid.FormatUUID keeps the format with no new failure mode. (statestore/lock_info.go)
- **Pre-compute the prior-state element lookup in the per-element plan-modification helper**: When both a plan-element coercion and the state lookup fail, it changes which diagnostic is reported. A closure called at the original point keeps the order. (internal/fwserver/attribute_plan_modification.go)
- **Replace validation's early returns with diags.HasError() checks**: Nested validation would then run after a parent error, producing extra diagnostics. (internal/fwserver/*_validation.go)
- **Add the #993 null/unknown guard while de-duplicating NestedAttributeObjectPlanModify and NestedBlockObjectPlanModify**: Changes plan-modification behaviour for blocks. Keep the two shapes as they are in any refactor. (internal/fwserver/attribute_plan_modification.go, block_plan_modification.go)
- **Move logging.Framework* calls into shared helpers as a pure refactor**: The framework logger records the call site (WithAdditionalLocationOffset(1)), so a moved call changes the @caller field of the log line. Message, level and fields stay identical. C14, C15 and C16 state this caveat. (internal/fwschemadata)
- **Delete the four struct fields staticcheck U1000 flags in internal/reflect tests**: They are deliberate unexported fields that test how reflection handles unexported fields; two already carry //nolint:unused. (internal/reflect/helpers_test.go:24-25, struct_test.go:859, :2197)
- **Delete the no-op resources in list/*_test.go that deadcode reports as unreachable**: They are compile-time interface scaffolding for ExampleResource_listable. (list/list_resource_test.go, no_op_list_resource_test.go, no_op_resource_test.go)

## Removed after verification

- **B14: After B4/B6: fold the per-type test twins into fixture tables** -> Not drafted; the +1,100 is unmeasured. The consolidated tests would share the abstraction they are meant to prove, which weakens them as proof for B4/B6. Test IDs change, so a test-identity diff is needed. The repository's explicit per-type test style is deliberate. The only gain is test-code size. Low confidence that it is clearly simpler, so drop it. _(independent verifier, area B (plan modification and validation))_
- **Four numeric function parameter types lack the ParameterWith<T>Validators interface assertions** -> Withdrawn by its author: the regex excluded digits. All 12 parameter types carry all 3 assertions, and a compile probe confirms they satisfy the interfaces. _(author self-check, area F)_
- **provider/metaschema lacks the dynamic-in-collection guard every other schema package has** -> Not filed: harm was not demonstrated (provider_meta is read-only configuration) and dynamic support deliberately skipped metaschema. Kept as a maintainer question. _(author self-check, area D)_
- **fmt.Errorf("constant") -> errors.New("constant") at 662 perfsprint sites** -> Equivalent but not worth a PR: every hit is a verb-free expectedError literal in 171 test files, and the project does not lint for it. _(author self-check, area D)_
- **Shared List/Set ToTerraformValue helper, helpers for the primitive *_type.go twins, the path step types and the tfsdk Set wrappers** -> Not simpler: measured net savings of 23, about 0, 36 and 38 lines spread over public files once the adaptor code is counted. _(author self-check, area E)_
- **Generic internal guard helper behind the 12 typed plan-modifier packages** -> Measured net -320 over 34 files, but each package must keep its own types, docs and method names, and these files are the reference implementations provider authors copy. Rejected as churn. _(author self-check, area D)_
- **Generic NewResultData helper for the 12 function Return types** -> Needs a per-type constructor and a ValueFrom<T> method value; longer than the 9-line twins. _(author self-check, area F)_
- **deadcode -test reports 47 unreachable functions** -> Only 13 are dead or unreferenced: 12 internal test fixtures (G2) and list.ListResultsStreamDiagnostics (F8, public, so deprecate rather than delete). The other 34 are public API that providers call (11 types.* constructors, providerserver.Serve), the 10 Return.NewResultData methods reached through an interface in production (a test gap, F2), and 12 compile-time scaffolding methods in list tests. _(coordinator, from areas E, F and G)_
- **TestServerListResource's asymmetric diagnostics comparer lets missing diagnostics pass** -> False: cmp's symmetry check panics on an asymmetric comparer. The real defect is narrower (it ignores Detail and length) and is filed as B35. _(author self-check, area G)_
- **Path.AtTupleIndex has no real test** -> False: a mutant that appends the wrong step passes the tautological TestPathAtTupleIndex but fails internal/reflect's TestFromSlice. G9 only covers the path package's own tests. _(author self-check, area G)_
- **Merge the per-type test fixtures (testdefaults, testplanmodifier, testvalidator, testprovider) and the four NestedObjectCustomType variants** -> Per-interface method names prevent merging, and the distinct NestedObjectCustomType types are load-bearing in nested custom-type tests. _(author self-check, area G)_
- **Remove the never-set DescriptionMethod/MarkdownDescriptionMethod fields from 43 test fixtures** -> fwserver logs validator descriptions, so these fields are the only hook for future log-content tests, and every fixture follows the one-func-field-per-method convention. Not worth the churn. _(author self-check, area G)_
- **One traversal walker shared by fwserver validation and plan modification** -> Measured only 10 shared boilerplate lines. Not simpler. _(author self-check, area B)_
- **Automatic ReadResource deferral loses Private and NewIdentity** -> Core does not write a deferred refresh to the refresh state, so no persisted loss could be shown. Not filed. _(author self-check, area B)_
- **Unsynchronised writes in ConfigureProvider and ConfigureStateStore** -> The protocol orders those RPCs before all others. Not filed. _(author self-check, area B)_
- **A5: retention is about 0.5 KB per finished RPC, and the context.AfterFunc fix cannot regress anything** -> With the real gRPC and tf6server stack each finished RPC retains 11,528 bytes (about 23 times more), so A5 moved to tier 2. The draft also raised retention from 106 to 624 bytes per RPC for callers whose parent context is never cancelled; registering the cleanup only when in.Done() != nil avoids that. _(independent verifier, area A)_
- **A9: guard ListResource with 'both raw schemas non-nil'** -> Incomplete: a non-nil raw schema with a nil Block passes that guard and still panics (reproduced). The fix must check the converted schema. _(independent verifier, area A)_
- **A1: Terraform generates different resource configuration because the framework's GenerateResourceConfig never runs** -> Overstated: generated configuration is identical today, because the framework's logic transcribes core's own legacy logic. The real effect is about 434 framework lines that never run in production. The item and its tier stand. _(independent verifier, area A)_
- **A6: InvokeAction always blocks forever once the stream consumer stops** -> Only when the action sends a second progress event after the consumer has stopped; a single post-cancel event does not block. _(independent verifier, area A)_
- **A10 includes the two fromproto resource_schema.go sites** -> Their only error path is unreachable: every tftypes kind is handled. Removed from A10's scope. _(independent verifier, area A)_
- **B4, B5, B6: the generic plan-modification and validation bodies add no cost** -> As drafted they add 2 heap allocations per modifier or validator call, even with logging off. Storing the two literal trace messages per type restores parity; the items now say so. _(independent verifier, area B (plan modification and validation))_
- **B7, B9, B10: the extractions are allocation-neutral** -> Each added one heap allocation per call through a function value. Calling the helper directly restores parity, and B7's net size becomes -339 instead of -349. _(independent verifier, area B (plan modification and validation))_
- **B1: no upstream issue exists for the empty plan-modifier PathExpression** -> Wrong: issues #1258 and #1219 report it, and open PRs #1260 and #1311 contain the same 2-line fix (upstream tracker as read on 2026-11-10). Land one of those rather than a new PR. _(independent verifier, area B (plan modification and validation))_
- **B18: log output is byte-identical after the fix** -> Identical only as sorted lines: tftypes.Transform walks maps, so line order varies from run to run in both versions. Without logging there is no time gain, only 25.6 MB less memory held. _(independent verifier, area B (RPC handlers))_
- **B20: five nil dereferences, one reachable from provider code** -> There is a sixth provider-reachable panic: UpgradeResourceIdentity dereferences the identity at server_upgraderesourceidentity.go:181 after the provider's upgrader can set it to nil. _(independent verifier, area B (RPC handlers))_
- **B34: about -30/+0 lines in about 12 files** -> Measured +8/-61 in 15 files (+8/-64 in 16 with A18): deleting the fields orphans imports and two tests' schema variables. _(independent verifier, area B (RPC handlers))_
- **B37: the one-line fix (WriteOnlyAttributesAllowed: true for actions) needs nothing else** -> Provider-defined validators on action attributes then see the capability flip from false to true, and the public doc says it is only populated for managed resources. The fix must update that doc and the changelog. _(independent verifier, area B (RPC handlers))_
- **C1: reject every map target whose key kind is not string** -> Breaks inputs that work today: map[any]T targets and an empty map read into map[int]T. The corrected fix is about -13/+30. _(independent verifier, area C)_
- **C2: send every nil pointer to FromPointer before the interface dispatch** -> Changes a nil-safe pointer-receiver Unknownable from unknown to null. Gate on the element type implementing the interface instead (+18 lines). _(independent verifier, area C)_
- **C3 can land on its own** -> Dropping the re-validation loses warnings until C6 lands, because FromUnknownable and FromNullable discard theirs; no test catches it. C6 is a hard prerequisite. _(independent verifier, area C)_
- **C4 and C11: reuse fwschema.ValidAttributeNameRegex inside internal/reflect** -> Import cycle (compile error reproduced): the pattern has to be copied. _(independent verifier, area C)_
- **C8: delete the reserved-name check for provider schema root blocks** -> Would accept a provider lifecycle block that practitioners cannot configure; core reserves lifecycle (and locals). Only names like connection, provider and provisioner are false positives. _(independent verifier, area C)_
- **C20: the measured draft is ready** -> It did not compile (an unused path import in both files). Fixed, it is -56/+3 and all eight schema packages pass. _(independent verifier, area C)_
- **D1: compare the default only against AttributeTypes** -> Leaves a hole: with a CustomType set, a default that matches AttributeTypes but not the custom type would be accepted, regressing a config that is correctly rejected today. Adding a TerraformType comparison closes it (about -2/+19). _(independent verifier, area D)_
- **D3: every single-nested object validator runs twice** -> Narrower: validators that return an error run once, and single-nested attributes run twice only when the value is known (blocks also when null or unknown). _(independent verifier, area D)_
- **E8: fixing ObjectValue.Equal is a plain bug fix** -> It would make the built-in RequiresReplace and RequiresReplaceIf modifiers plan a replacement when a null or unknown object's attribute types change, and Terraform's value library treats typed nulls as equal. Downgraded to PLAUSIBLE: maintainers must choose between fixing the code and fixing the doc. _(independent verifier, area E)_
- **F3: 3 of 5 plain runs returned a duplicate lock ID** -> Not reproduced: 0 duplicates in 40 runs of 64 calls. Duplicates appear only under sustained load (14 to 24 per 64,000 IDs). The race itself reproduces under -race. _(independent verifier, area F)_
- **F4: fix by failing registration in ListResourceFuncs, at tier 2** -> Terraform core never sends this request, so the panic is reachable only from direct ProviderServer callers: tier 3. A registration-time error surfaces in GetProviderSchema, which Terraform treats as fatal for every command, so providers that work today would break; guard the ListResource call instead. _(independent verifier, area F)_
- **F9: the proposed doc sentence for parameter names** -> Wrong for custom Parameter.GetName implementations, which get no error for an empty name. Size is about -44/+60 in 17 files, not -30/+30 in 16. _(independent verifier, area F)_
- **G1 is tier 1** -> It touches 42 files and only changes test idioms, so it is tier 2 under this report's rules. _(independent verifier, area G)_
- **A18: fwserver.UpgradeResourceIdentityRequest.TypeName is never written or read (found by the converter field audit; outside area A)** -> Merged into B2's dead-code item, which removes the other set-but-never-read fwserver request fields in the same PR; area B's auditor confirmed it independently. _(coordinator (cross-area de-duplication))_
- **E13: tfsdk/value_as_test.go: 13 typed newXPointer/newXPointerPointer helpers -> new(T(v)) / new(new(T(v)))** -> Merged into G1, the complete repo-wide new(expr) change: it covers the same tfsdk/value_as_test.go helpers, and its go fix -inline step already writes the explicit conversions E13 called for. E13's remaining piece, the dead commented-out case, is now a G1 note. _(coordinator (cross-area de-duplication))_
- **G4: benchmarkSetTypeValidate never fills its set: all six BenchmarkSetTypeValidate{10..1000000} measure Validate on an EMPTY set** -> Duplicate of E12: areas E and G found the same empty-set benchmark bug independently. The two reports agree, which corroborates it; the details are merged into E12. _(coordinator (cross-area de-duplication))_
- **A-V1: ListResource with no protocol Config gives the provider a list config typed with the managed-resource schema (B2 hand-off confirmed; already filed as B20 part (4), merge rather than duplicate)** -> Duplicate of part (4) of B20 (the list config fallback built from the managed-resource schema). The area A verifier reproduced it independently, which corroborates that item. _(coordinator (cross-area de-duplication))_
- **BV1: Object validators of SingleNestedAttribute and SingleNestedBlock run twice per validation (as attribute/block validators and again as nested-object validators)** -> Duplicate of D3: the area B verifier found the same double run of single-nested object validators independently, with matching call counts, which corroborates D3. _(coordinator (cross-area de-duplication))_
- **FV1: Dead exported var fwserver.NoListResults (0 references)** -> Duplicate of B34 (dead request fields and NoListResults). The area F verifier found it independently, which corroborates that item. _(coordinator (cross-area de-duplication))_
- **FV2: fwserver.ConfigureProvider dereferences a nil request when the provider sets Deferred** -> Duplicate of part (1) of B20 (latent nil dereferences). The area F verifier found it independently, which corroborates that item. _(coordinator (cross-area de-duplication))_

## Verification protocol for every PR

1. Branch from main; one issue per PR; PR title names the issue ID from this report; no CHANGELOG entry for refactors and test changes (CONTRIBUTING: refactoring does not get an entry), a BUG FIXES changie entry for every bug PR.
2. Before touching code: run `go test ./<package>/...` for every package the PR touches and record the test list (`go test -run . -v ./pkg 2>&1 | grep -E '^(=== RUN|--- (PASS|FAIL))' | sort`); after the change the list of test IDs and statuses must be identical (test-identity diff), not just the pass count.
3. `GOTOOLCHAIN=auto go vet ./... && gofmt -s -l . && golangci-lint run ./...` (golangci-lint v2.15+, built with Go 1.26) must stay at 0 issues; forcetypeassert is enforced, so any new type assertion needs `, ok`.
4. For items that cite go fix: apply exactly one analyzer (`go fix -diff -<analyzer> ./...` to preview, then without -diff), commit, and verify the diff contains only that analyzer's rewrite; never run all analyzers in one PR.
5. For duplication removals: diff the extracted helper against every copy it replaces (normalise names with sed) before deleting the copies; the report's 'where' field lists every copy; a copy that differs in one line is a separate decision.
6. For items in internal/fwserver, internal/fwschemadata and internal/reflect: also run `go test ./internal/proto5server/... ./internal/proto6server/...` (they exercise the full RPC path) and, before merging, the terraform-provider-corner acceptance jobs in CI must be green for both protocols.
7. Public packages: `go doc` output for the package before and after must be identical (exported names, signatures, doc comments); `gorelease`-style API diff is the acceptance criterion for any PR touching a non-internal package.
8. Bugs: add the failing test first (named in the item), confirm it fails on main, then fix; the test name goes in the changie entry body.
9. Keep this report's issues.json next to the code review: tick items off by ID so the next audit can diff against it.

## Appendix: method and artefacts

**Method.**
- Inventory and census at commit 00badb1: the skill's census script with project-specific Go patterns; go vet; gofmt -s; golangci-lint v2.15 with the project configuration (0 issues) and with a scratch configuration adding modernize, dupl, gocognit, gocyclo, funlen, nestif, perfsprint and others (7,045 leads); staticcheck -checks all (285 reports); deadcode -test (47 reports); and go fix -diff with every Go 1.26 analyzer (827 sites). Duplication was measured by diffing files after normalising protocol and type names.
- Eight area auditors read every lead in their area, plus the code the tools cannot judge. They drafted each non-trivial change on a scratch copy to measure its size and run the affected package tests.
- Eight independent verifiers, one per area, re-checked every claim with their own probes, under instructions to find where it was wrong. Items a verifier discovered were checked again by a second verifier or by the coordinator.
- The coordinator merged cross-area duplicates, owns the repo-wide go fix items (M1 to M3), and ran the whole suite on a copy with M1 and M2 applied.
- Usage limits interrupted the run twice and a machine restart once. Every agent resumed from its saved state and no findings were lost.

**Artefacts in this folder.**
- `issues.json` is the source of every number in this report (the audit skill's schema).
- `notes/auditor-*.md` and `notes/verifier-*.md` are the per-area working notes, with the commands that reproduce each count, the leads discarded and the rewrites rejected.
- `drafts/` holds the diff and patch files behind the "measured" sizes: `drafts/<area>/` for auditors and `drafts/V-<area>/` for verifiers. They are measurement drafts, not reviewed patches. Probe programs and repository copies were not kept.

**Identifier map.** The notes use the agents' original identifiers. In this report, B1-k is Bk and B2-k is B(k+14); D-N1 is B37 and C21 is E14. B1-15 was merged into D3, A-V1 and FV2 into B20, FV1 and A18 into B34, E13 into G1, and G4 into E12.
