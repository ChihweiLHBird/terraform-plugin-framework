# Area E notes: types, types/basetypes, attr, attr/xattr, path, diag, tfsdk

Auditor E. Repository read-only; all drafts, probes and test runs were done in scratch copies:
- baseline snapshot: `git archive HEAD | tar -x -C .../work/E/repo` (commit 00badb1)
- drafts: `.../work/E/draft` (E5 primitive helpers), `draft2` (E4+E9), `draft3` (E2, E3, E7, E8),
  `draft4` (rejected list/set helper), `draft5` (E1), `draft6` (E10), `draft7` (E4 alone), `draft8` (E11, E13)
- probe module `.../work/E/probe` (replace directive -> the real repo, read-only) and `probe2`/`probe3`
  (replace -> draft2/draft3) for before/after behaviour.
- AST tools: `.../work/E/twinprobe` (normalised per-method twin comparison), `.../work/E/fwdcheck`
  (checks every `types.X` func is `return basetypes.NewX(args...)`).
- Diffs kept: `.../work/E/{primitive_core,collections,listset,mustpanic,r1,c1,t2}.diff`.

Scratch is outside the repo, so `go test` there never touched the repository. No whole-repo test or
lint runs: only single packages (`./types/basetypes/`, `./path/`, `./diag/`, `./tfsdk/`, `./internal/fwserver/`,
`./internal/fwschemadata/`, `./internal/reflect/`, `./internal/proto5server/`, `./internal/proto6server/`, ...).

## Coverage: exhaustive vs sampled
Exhaustive (every non-test file read): types/ (27 files), types/basetypes primitives (bool, float32, float64,
int32, int64, number, string, dynamic: value + type), basetypes collections (list, map, set, object, tuple:
value + type, Equal/String/Elements/ToTerraformValue/Must/From), missing_*, terraform_type_to_framework_type,
collection_length_options, attr/, attr/xattr/, path/ (all 21 files), diag/ (all 9 files), tfsdk/ (all 10 files).
Sampled: tests (only where a lead or finding pointed: value_as_test.go, set_value_test.go benchmarks,
*_type_test.go error cases, path Equal/String tests, diag Errors tests, tfsdk wrapper tests for duplication).
All 490 leads were classified (see "Lead disposition").

## Findings summary (details in findings-E.json)
| id | kind | tier | one line |
|----|------|------|----------|
| E1 | bug | 2 | `%s` on *big.Float prints `%!s(*big.Float=80.5)` in user-facing Int32/Int64/Float32/Float64 errors (13 sites) |
| E2 | bug | 2 | Path.Equal / Expression.Equal asymmetric for nil vs empty steps: wrong ImportStatePassthrough guard, order-dependent diag dedup |
| E3 | bug | 2 | (*ResourceIdentity).Set / (*Resource).Set panic on nil receiver; SetAttribute returns a diagnostic |
| E4 | refactor | 1 | SetValue.Equal / MapValue.String / ObjectValue.String copy the whole collection per element (quadratic) |
| E5 | refactor | 2 | unexported generic helpers for the primitive value state machine (-192/+88 measured) |
| E6 | bug | 3 | Float32Type debug log fires exactly when float32/float64 are identical (condition inverted) |
| E7 | bug | 3 | Paths/Expressions.String emit "[,a]" when the first element is skipped as empty |
| E8 | bug | 3 | ObjectValue.Equal ignores attribute types for null/unknown (contradicts its own doc and siblings) |
| E9 | modernization | 3 | mapsloop x4 -> maps.Copy, sort.Strings x3 -> slices.Sorted(maps.Keys(..)) |
| E10 | refactor | 3 | 5 copies of the diagnostics->panic-string loop in New*ValueMust -> one helper |
| E11 | cleanup | 3 | 13 doc comments contradict code or name removed/nonexistent identifiers |
| E12 | test | 3 | BenchmarkSetTypeValidate* benchmarks an empty set at every size |
| E13 | test | 3 | tfsdk/value_as_test.go: 13 typed pointer helpers -> new(T(v)); type-preservation trap |

## Key measurements (commands reproducible)
- E1 sites: `grep -rn 'Value %s' --include=*.go types/basetypes | grep -v _test.go` -> 13 (float64_type.go 4,
  float32_type.go 2, int32_type.go 3, int64_type.go 4). Tests pinning the garbled text:
  `grep -rn 'cannot be represented\|is not an integer' --include=*_test.go .` -> 9 (3 literal `%!s(*big.Float=...)`
  strings in int32_type_test.go; 6 computed with the same Sprintf on a *big.Float in float32/float64 tests).
  End-to-end via providerserver.NewProtocol6WithError + ValidateResourceConfig with `port = 80.5` on an
  Int64Attribute: `Configuration Read Error ... Error: Value %!s(*big.Float=80.5) is not an integer.`
  (probe `.../work/E/probe/e2e`). big.Float.Format supports b,e,E,f,F,g,G,x,X,p,v only.
- E2: `grep -n "steps == nil" path/path.go path/expression.go` -> 4 lines (2 per Equal). Probe output (before):
  `Path{}.Equal(Empty())=false`, `Empty().Equal(Path{})=true`, `Path{}.Equal(Path{}.Copy())=false`,
  `Expression{}.Equal(MatchRelative())=false` (reverse true), `Append(zeroPathDiag, emptyPathDiag)` keeps 2 but
  reversed keeps 1, `ImportStatePassthroughID(ctx, path.Path{}, ...)` -> "Value Conversion Error" instead of
  "Resource Import Passthrough Missing Attribute Path". After the draft fix all symmetric; path, diag, tfsdk,
  fwserver, fwschemadata, fwschema tests pass in draft3.
- E4: allocation probe (testing.AllocsPerRun / testing.Benchmark, `.../work/E/probe/setalloc`):
  SetValue.Equal n=1000: 1000 allocs, 16,384,096 B/op -> 0 allocs, 0 B; n=3000: 147 MB/op -> 0.
  MapValue.String n=1000: 10,084 allocs, ~83 ms -> 5,030 allocs, ~0.6 ms; n=3000: ~680 ms -> ~1.7 ms.
  `grep -rn "\.Elements()\|\.Attributes()" types/basetypes/*.go | grep -v _test.go` -> 10 sites; 3 are per-element
  (map_value.go:359, object_value.go:391, set_value.go:309), 4 are extra whole copies in the same String methods
  (map 345/346, object 377/378), 3 are a single harmless copy per call (list 337, set 345, tuple 219).
- E5: twin comparison (`twinprobe ... value bool float32 float64 int32 int64 number string dynamic`): IsNull/IsUnknown
  identical in all 8; NewXNull/NewXUnknown identical in all 8; Equal identical for bool,int32,int64,string (float32/
  float64 share a big.Float variant; number and dynamic differ); ToTerraformValue identical for 6 (bool/string
  with tftypes.Bool/String, float32/float64/int32/int64 with Number); String has 6 variants (format verbs differ);
  Value<T>Pointer identical for bool,int32,int64,string. Draft: -192/+88 incl. a 70-line helper file; probe of
  invalid-state panic text, String, Equal, pointers identical before/after; basetypes, tfsdk, reflect tests pass;
  micro-benchmarks show no extra allocations (Pointer goes from 16 B to 8 B because only the value escapes).
- E10: `grep -rn '"%s | %s | %s"' --include=*.go . | grep -v _test.go` -> 5 (list, map, object, set, tuple).
  Draft -60/+33; panic texts identical for all 5 (probe test in scratch).
- E11: `grep -rn "Setting the deprecated" types/basetypes` -> 5; `grep -n ElementsAs types/basetypes/object_value.go` -> 3.
- E12: `go test ./types/basetypes/ -run XXX -bench BenchmarkSetTypeValidate -benchtime 2000x` (scratch copy):
  10 -> 111 ns/op, 1,000,000 -> 4,959 ns/op (all sizes ~constant: the set is empty). With len-allocated slice:
  10 -> 10 us, 100 -> 0.68 ms, 1000 -> 82.5 ms and 24 MB / 999,249 allocs per op (quadratic duplicate check).
- E13: `grep -c "^func new" tfsdk/value_as_test.go` -> 13 helpers (the brief said 14; the 14th name,
  newBigFloatPointer, only appears in a commented-out case). Live call sites: 42 matches - 13 defs - 2 comments = 27.
  Draft -96/+27; TestValueAs 21 subtests pass.
- types/ forwards: `fwdcheck /home/user/terraform-plugin-framework/types` -> 54 funcs, 0 non-forward; each
  `types.X(args)` is exactly `return basetypes.NewX(args)` with args in order; 11 `type X = basetypes.XValue`
  and 13 `var/alias` type names checked.

## "Looks equivalent but is not" (rejected rewrites)
1. `maps.Clone(m.elements)` for MapValue.Elements / ObjectValue.Attributes / AttributeTypes / ObjectType.AttributeTypes:
   maps.Clone(nil) returns nil; today null/unknown values return an empty non-nil map. Use maps.Copy into the
   existing `make(..., len)` (E9), which is equivalent.
2. `slices.Clone(l.elements)` for List/Set/Tuple Elements(): nil for nil (null/unknown) instead of empty non-nil.
3. Diagnostics.Errors()/Warnings() via `slices.DeleteFunc(slices.Clone(diags), ...)`: nil receiver -> nil instead of
   `Diagnostics{}`. Trap: TestDiagnosticsErrors/TestDiagnosticsWarnings would still pass because cmp.Diff calls
   `Diagnostics.Equal` (len-based), so the suite is blind to nil-vs-empty here.
4. Embedding a generic core struct with exported methods (IsNull/IsUnknown promoted) into BoolValue etc.: moves
   those methods one embedding level deeper. Downstream custom types embed basetypes values (documented pattern);
   a provider type embedding StringValue next to another embedded type that reaches IsNull at depth 2 would go
   from "StringValue wins at depth 1" to an ambiguous selector (compile error, and attr.Value no longer satisfied).
   It also changes `%#v` and reflect field layout. E5 therefore uses free generic functions and keeps every
   exported method declared on each type with an unchanged struct layout.
5. Folding Float32/Float64/Number/Dynamic Equal into `primitiveEqual`: known-with-nil-value semantics differ
   (float: nil==nil true; number: Cmp on nil panics; dynamic: false) and *big.Float needs Cmp, not ==. Excluded.
6. `new(12.3)` / `new(12)` for `newFloat32Pointer(12.3)` / `newInt32Pointer(12)`: untyped constants default to
   float64/int, so the pointer type silently changes; the table field is `interface{}` so it still compiles and the
   test stops exercising float32/int32/int64. go fix only flagged the bool/float64/string call sites for this reason.
   (`pointer(x)` calls in basetypes/tfsdk tests are fine: generic inference uses the same default types, and the
   float32/int32/int64 sites already carry explicit conversions.)
7. Shared list/set ToTerraformValue helper: measured -82/+59 (net -23) in draft4; the adaptor (file header,
   imports, merged MAINTAINER NOTE) eats the savings. Rejected.
8. Primitive *_type.go twins: every method except ValueFromTerraform (8 distinct variants) and the deprecated
   Validate (Float64/Int64 only) is 1-2 statements; only Bool/String ValueFromTerraform form an identical
   non-trivial pair (19 lines each). A generic helper would cost about what it saves. Rejected.
9. path step Equal helper `stepEqual[T comparable]`: 10 of 12 step types could use it; about -36 net lines spread
   over 10 public files. Rejected as churn.
10. tfsdk Set/SetAttribute shared helper: 10 methods x 8 lines, about -38 net; E3 adds per-type nil guards with
    per-type texts anyway. Rejected. Get/GetAttribute/PathMatches are already 1-line forwards.
11. Paths.String separator fix via `result.Len() > 1`: wrong when a non-empty path renders "" (e.g. Root("")).
    E7 collects and joins instead.
12. Fixing E2 by making Copy() preserve nil instead: does not fix `Path{}.Equal(path.Empty())`.
13. sort.Strings -> slices.Sort is equivalent for []string (unique map keys, so stability is moot); E9 goes one step
    further with slices.Sorted(maps.Keys(..)), whose nil result for an empty map is unobservable (only ranged over).

## Lead disposition (490 leads)
- golangci/modernize 137, gofix/any 74, gofix/newexpr 46, gofix/reflecttypefor 14, intrange 3, rangeint 1:
  coordinator-owned patterns. Checked for behaviour changes: reflecttypefor sites all take concrete-typed
  expressions (equivalent); newexpr: only the tfsdk typed helpers have the type trap (E13); intrange/rangeint
  loops do not modify the index (equivalent).
- gofix/mapsloop 4: E9 (all 4 confirmed: dst is always a fresh `make`, nil src is a no-op either way).
- staticcheck 49: all SA1019 xattr.TypeWithValidate, intentional (//nolint present). Not findings.
- golangci/dupl 102: production hits -> int32/int64_value.go (E5), list/set_value.go ToTerraformValue (rejected #7),
  list/set_type.go ValueFromTerraform (rejected), tfsdk/plan.go vs ephemeral_result_data.go (1-8 line forwards,
  rejected #10). Test hits: not proposed. Measured tfsdk wrapper tests (normalising type names): plan_test.go vs
  resource_test.go differ in 52 of 482 lines, ephemeral 76/480, state 99/389, identity 118/482, config 202/286;
  about 2,000 near-identical lines, but these are per-type public API contract tests and a table merge is not a
  mechanical change. Recorded, not proposed.
- golangci/funlen 21, gocognit 1: long table-driven tests; not actionable.
- deadcode 11 (types.*PointerValue, *ValueFrom, TupleValue): public forwards; not dead. AST check proves each is a
  correct forward, so the missing tests are a low-value gap (not proposed).
- golangci/recvcheck 10 (Diagnostics, ExpressionSteps, Expressions, PathSteps, Paths, 5 tfsdk types): mixed
  receivers are public method-set behaviour; cannot change.
- golangci/dupword 7: test names "... pointer pointer" are intentional.
- golangci/perfsprint 6: output-identical alternatives exist (strconv.FormatBool, strconv.FormatInt, errors.New)
  but are micro-optimisations in String()/error paths; not worth a PR. (E5 keeps the Sprintf calls unchanged.)
- golangci/thelper 2: benchmark helpers; b.Helper() is cosmetic. set_value_test.go:66 is the broken benchmark (E12).
- golangci/exhaustive 1 (diag/severity.go:33): default branch already returns "Invalid" for SeverityInvalid.
- golangci/godox 1 (set_type.go:213): TODO with an upstream issue link; legitimate.

## Observations not reported as findings (for the verifier / other areas)
- SetType.Validate (deprecated xattr path, but still invoked by fwschemadata.Data.ValueAtPath / SetAtPath for every
  set attribute) is O(n^2) with tftypes.Value.Equal (deepEqual): ~82 ms, 24 MB per 1,000-element set (measured after
  fixing the benchmark). Not mechanical to fix (tftypes.Value is unhashable). Worth a perf issue.
- "Configuration Read Error ... This is always an error in the provider" is shown for practitioner input errors such
  as `port = 80.5` (fwschemadata wording; outside area E).
- NumberValue: NewNumberValue stores the caller's *big.Float and ValueBigFloat returns the internal pointer (shared by
  all copies). Design choice; changing it is a behaviour change. The doc fix in E11 only corrects "returns 0.0".
  ValueBoolPointer/ValueInt*Pointer/ValueStringPointer/ValueFloat*Pointer return fresh pointers (value receivers):
  no aliasing.
- TupleValue.ElementTypes returns its internal slice uncopied (Elements() copies); TupleValue.Type shares it with the
  returned TupleType. Latent aliasing, very low impact (tuples are rarely used); not reported.
- MapValue.Equal does not check key presence (`other.elements[key]` may be nil) and relies on `elem.Equal(nil)`
  returning false; true for every framework type, a custom element type with a bare type assertion would panic.
- NumberValue.Equal has no nil guard (Float32/Float64 do) but a known NumberValue with nil value is unconstructible
  through the public API (NewNumberValue(nil) is null); ToTerraformValue's nil branch is effectively dead.
- Paths/Expressions/PathSteps/ExpressionSteps.Append on a nil *pointer* return the input without dedup.
- Diagnostics.ErrorsCount/WarningsCount allocate a filtered slice just to count; zero framework callers.
- Wording differences that any unification must preserve (observable panic/diagnostic text): Must panics use
  "NewListValueMust", "NewTupleValueMust" but "MapValueMust", "ObjectValueMust", "SetValueMust"; From diagnostics say
  "using NewListValueFrom" but "using MapValueFrom/ObjectValueFrom/SetValueFrom"; primitive String formats differ
  (%t, %d, %q, %f for float32/float64, big.Float.String() for Number), e.g. Float64Value(1e-7).String() is "0.000000".
  ObjectValue.String does not quote-escape keys (%s inside quotes) while MapValue.String uses %q.
- ToTerraformValue default-branch panics format the invalid state with %s, which itself panics inside
  ValueState.String, giving "unhandled Bool state in ToTerraformValue: %!s(PANIC=String method: unhandled ValueState
  in String: 3)". Unreachable outside the package; E5 preserves it byte-for-byte.
- E7 leaves one edge: a root expression with zero steps (path.Empty().Expression()) is not equal to Expression{}
  (root differs), so it is still rendered as an empty entry. Decide whether "empty" means "no steps".

## Please double-check (verifier)
- E1: confirm no other test in other packages pins the `%!s(*big.Float=` text (my grep found only basetypes).
- E2: confirm nothing relies on `Path{}.Equal(path.Empty()) == false` (no test does; draft3 suites pass).
- E3: tier: a nil-pointer panic (crash) but only reachable through provider misuse (calling resp.Identity.Set on a
  resource without an identity schema, or a ListResult built as a literal); I chose tier 2.
- E8: is a behaviour change some providers may rely on (comparing against types.ObjectNull(nil)); maintainers may
  prefer to fix the doc instead. I ran basetypes, fwschemadata, reflect, fwserver, proto5/6server, fromproto5/6,
  toproto5/6, function, datasource/provider/resource schema and object plan-modifier/default tests on the fix: all pass.
- E4/E9 overlap in MapValue.String/ObjectValue.String: land E4 first (7 lines), then E9.
