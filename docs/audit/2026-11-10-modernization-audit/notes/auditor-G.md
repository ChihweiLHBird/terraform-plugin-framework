# Area G working notes: test infrastructure, repo-wide test hygiene, build tooling, CI

Auditor G. Repo at 00badb1 (read-only; nothing in /home/user/terraform-plugin-framework was modified).
All experiments ran on a full copy of the repository (no .git) at
`drafts/G/copy`,
with a scratch-only git repo inside it so every step has an exact diffstat (branches listed at the end).
Draft patches for every measured item are in `.../audit/work/G/patches/`.

## 0. Corrections to the shared brief (please propagate)

- CI Go matrix is `['1.26', '1.27']` (.github/workflows/ci-go.yml:82), NOT "1.26 and 1.25". go.mod says `go 1.26.0`
  and README says "Go 1.26 or later": all three are consistent. No finding.
- "newexpr=218 call sites" is 205 call sites + 13 helper *definitions* (each definition is its own go fix diagnostic).
  `go fix -inline` then removes 7 more calls (the int32/int64/float32 typed helpers), so 212 call sites in total
  disappear (golangci `modernize` reports newexpr=212, which matches).
- "14 typed newXPointer helpers" in tfsdk/value_as_test.go: there are 13 definitions (`grep -n '^func new' tfsdk/value_as_test.go`);
  the 14th name (`newBigFloatPointer`) only appears inside a commented-out test case (value_as_test.go:173-174).
- ST1000 x15 = 14 hits for the single package `statestore` (one per file) + 1 malformed comment in internal/testing/testtypes/doc.go.

## 1. go fix verification

Method: on the copy, one analyzer at a time, `GOTOOLCHAIN=auto go fix -p 2 -<analyzer> ./...` (apply mode), then
`gofmt -l` and `gofmt -s -l` on the changed files, `go vet -p 2 ./...` (whole module, compiles all test packages),
`go test -p 2 -count=1 <packages containing changed files>`, commit. After all analyzers, ONE whole-suite run:
`go vet -p 2 ./...` + `GOTOOLCHAIN=auto go test -p 2 -count=1 ./...` on the cumulative copy, and
`staticcheck -checks all ./...` on the cumulative copy. golangci-lint was NOT run on the copy (only `golangci-lint linters`
and `golangci-lint config verify`, which do not analyse code).

| step | files | lines | sites | vet | tests (changed pkgs) | gofmt / gofmt -s |
|---|---|---|---|---|---|---|
| `go fix -newexpr ./...` (raw) | 42 (all *_test.go) | +225 / -212 | 205 calls rewritten (192 lines of `pointer(...)`, 1 `strPtr`, 6 `newBool/Float64/StringPointer`); 13 helper bodies rewritten to `return new(x)` + `//go:fix inline` added | clean | 7 pkgs ok | clean |
| delete the 10 helpers newexpr left dead (6 pointer_test.go files, strPtr, 3 tfsdk helpers) | 8 | -74 | | clean | ok | clean |
| `go fix -inline ./...` (consumes the new directives; 7 calls -> `new(int64(0))`, `new(float32(12.3))`...) + delete 3 more dead helpers | 1 | +7 / -22 | 7 | clean | tfsdk ok | clean |
| **= complete newexpr PR** | 42 (36 modified, 6 deleted) | **+206 / -282** | 212 calls, 13 helpers gone | clean | ok | clean |
| optional: 6 `newXPointerPointer` helpers -> `new(new(x))` | 1 | +12 / -42 | 12 | clean | tfsdk ok (TestValueAs 23 subtests) | clean |
| `go fix -any ./...` | 269 (47 test, 222 non-test) | +553 / -553 | 554 occurrences of `interface{}` (185 in tests; 4 in internal/testing/testtypes) | clean | 22 pkgs ok | clean |
| `go fix -reflecttypefor ./...` | 13 (11 test, 2 non-test: internal/reflect/generic_attr_value.go, into.go) | +30 / -30 | 31 (26 test) | clean | 3 pkgs ok | clean |
| `go fix -fmtappendf ./...` | 1 (internal/privatestate/data_test.go) | +6 / -6 | 6 | clean | ok | clean |
| `go fix -rangeint ./...` | 5 test files | +5 / -5 | 5 | clean | 3 pkgs ok | clean |
| `go fix -waitgroup ./...` | 2 (proto5server/serve_test.go, proto6server/serve_test.go) | +4 / -8 | 2 | clean | 2 pkgs ok | clean |
| `go fix -mapsloop ./...` | 3 (types/basetypes map_value.go, object_type.go, object_value.go) | +7 / -12 | 4 (all `maps.Copy` into a `make`d map) | clean | ok | clean |
| `go fix -slicescontains ./...` | 2 (internal/fwschema/attribute_name_validation.go x2, internal/fwtype/missing_underlying_type_validation.go x1) | +25 / -37 | 3 | clean | 2 pkgs ok | clean |
| `go fix ./...` (ALL default analyzers) afterwards | 0 | 0 | nothing else pending (forvar, minmax, stringscut*, stringsseq, testingcontext, slicessort... all empty) | | | |
| **cumulative copy** | 303 | +836 / -933 | | `go vet ./...` clean | **whole suite: 56 ok, 22 no-test, 0 FAIL (34 s wall, warm cache)** | clean |

`staticcheck -checks all ./...` on the cumulative copy: 285 findings, identical (after normalising line:col) to the
baseline staticcheck-all.txt, i.e. no new U1000 once the dead helpers are deleted.

Key facts the coordinator needs:
1. `go fix -newexpr` alone BREAKS CI: it keeps the helpers (rewrites them to `//go:fix inline` wrappers around `new(x)`)
   while rewriting every caller, so staticcheck U1000 (= golangci `unused`, enabled in .golangci.yml) reports 10 dead
   helpers: function/pointer_test.go:7, internal/fwschemadata/pointer_test.go:7, internal/toproto5/pointer_test.go:7,
   internal/toproto6/pointer_test.go:7, tfsdk/pointer_test.go:7, types/basetypes/pointer_test.go:7,
   internal/reflect/pointer_test.go:180 (strPtr), tfsdk/value_as_test.go:25/45/80 (newBoolPointer, newFloat64Pointer,
   newStringPointer). The PR must delete them (6 of the 8 files become empty and are deleted).
2. newexpr does not rewrite calls whose argument's default type differs from the parameter type
   (`newInt64Pointer(0)` would become `*int`). `go fix -inline ./...` then inlines those 7 calls with explicit conversions
   (`new(int64(0))`), after which newFloat32Pointer/newInt32Pointer/newInt64Pointer are dead too.
   `go fix -inline ./...` touches nothing else in this module: no `//go:fix inline` directives exist in the repo or its
   4 direct deps, and the stdlib ones (reflect.Ptr, reflect.PtrTo, io/ioutil, go/types) are not used (grep = 0).
3. Equivalence of `pointer(x)` -> `new(x)`: both allocate a fresh variable initialised to x and return its address;
   the static type is the same because `pointer[T]` infers T from the argument exactly as `new(expr)` takes the
   expression's (default) type: `pointer(1.2)` and `new(1.2)` are both *float64; `pointer(attr.Value(v))` and
   `new(attr.Value(v))` are *attr.Value; `pointer(&struct{...}{})` -> `new(&struct{...}{})` is **struct. Verified by compile + tests.
4. Other analyzers: fmtappendf `[]byte(fmt.Sprintf(f, x))` -> `fmt.Appendf(nil, f, x)`: equal bytes; nil-vs-empty differs
   only for an empty result, and all 6 formats are non-empty literals. waitgroup: `wg.Add(1); go func(){defer wg.Done()...}()`
   -> `wg.Go(func(){...})` is the documented equivalent. rangeint: loop vars not modified in bodies (analyzer checks).
   mapsloop: destination is always a fresh `make(map, len(src))`, so non-nil-ness is preserved. slicescontains: the loops
   had `break` after the single AddError / `return true` on first match, same short-circuit order.
   reflecttypefor: only rewrites when the operand's static type is not an interface (`reflect.TypeOf(s)` with `s string`).
   any: alias, byte-identical types (allowed by the brief), `%T` prints `interface {}` either way.
5. Not covered by go fix 1.26 but reported by golangci `modernize` v2.15: `reflecttypeassert` x12, all in internal/reflect
   tests (interfaces_test.go:276,414,562,929; pointer_test.go:54,74,94; pointer_zero_test.go:18,34,50,66,82):
   `v.Interface().(T)` -> `reflect.TypeAssert[T](v)` (Go 1.25). Area C's call; needs a manual or gopls rewrite.

## 2. Dead test fixtures (item 1) - swept exhaustively

Exported identifiers under internal/testing: 268 (266 types, 2 funcs; all 319 package-level vars are `var _ X = Y{}`
assertions). Method: AST lister over every internal/testing file, then for each name: qualified references
(`\bpkg\.Name\b`) in all 1887 .go files minus the defining file, and unqualified references in sibling files of the same
package. 14 names have zero references outside the defining file; 10 more only "same-package" hits, all of which were
false positives (field names of embedded basetypes values, e.g. `DynamicValue: in` in dynamicwithsemanticequals.go:53).
Reading every one:

- Truly dead (no reference, never constructed): `SetTypeWithValidateError` (setwithvalidate.go:24, +Validate :32, +assertion :19);
  `SetTypeWithValidateAttributeError` + `SetValueWithValidateAttributeError` (setwithvalidateattribute.go:17-45).
  Their List/Map siblings are used by internal/reflect tests and the Set*Warning variants by fwschemadata, so this is
  leftover fixture surface, not a missing test (reflect's set path shares FromSlice/Into with lists).
- Never instantiated: `DynamicValue, Float32Value, Float64Value, Int32Value, Int64Value, ListValue, MapValue, ObjectValue,
  SetValue` (testtypes/{dynamic,float32,float64,int32,int64,list,map,object,set}.go:30-42). The matching `XType` only
  overrides `Equal`, so its inherited `ValueFromTerraform`/`ValueType` return basetypes values; the `XValue` wrapper is
  referenced solely by its own `var _ basetypes.XValuable = XValue{}` line. The 9 files are identical modulo the type name.
- The 12 deadcode leads: 9 x `XValue.Equal` -> orphaned (needed only so the never-used wrapper satisfies attr.Value);
  `SetTypeWithValidateError.Validate`, `SetTypeWithValidateAttributeError.ValueFromTerraform`,
  `SetValueWithValidateAttributeError.ValidateAttribute` -> orphaned. None is required by an interface the type is
  actually used as. -> finding G2 (11 files, +11/-206, vet of whole module clean, reflect/fwschemadata/tfsdk tests ok).
- NOT dead although zero-ref by name: `BoolValueWithValidateParameterError`, `StringValueWithValidateParameterError`,
  `ListValueWithValidateAttributeError`, `MapValueWithValidateAttributeError`, `NumberValueWithValidateAttributeError`,
  `ObjectValueWithValidateAttributeError`, `List/Map/Set/SingleNestedAttributesCustomValue` (all returned by a used
  sibling Type's ValueFromTerraform); `DynamicTypeWithSemanticEquals` (returned by the used
  `DynamicValueWithSemanticEquals.Type()`).
- list/*_test.go (deadcode: ComputeInstanceResource.*, NoOpListResource.*, NoOpResource.*, ExampleResource_listable):
  NOT dead: they are compile-time interface checks inside an Example that has no `// Output:` (compiled, never run).
  But see G8: go/doc drops that example (package list has no identifier `Resource`).
- Unexported helpers in *_test.go: staticcheck -checks all reports U1000 only for 4 struct fields; all 4 are deliberate
  reflection fixtures (internal/reflect/helpers_test.go:24-25, struct_test.go:859, struct_test.go:2197; three of them
  carry `//nolint:unused`) used to prove that unexported fields are ignored. 0 unused unexported test functions/types.
- Exported identifiers in *_test.go (U1000 ignores exported names): 14 (DynamicValueMust x4, reflect embed fixtures,
  list example types); all referenced.
- `*Method` func fields of fixtures that no test ever sets: of 83 distinct names only `DescriptionMethod` and
  `MarkdownDescriptionMethod` (43 types: 12 testdefaults, 12 testplanmodifier, 12 testvalidator, 7 testprovider
  config validators). Rejected as a cleanup, see section 10.

## 3. Fixture duplication (item 2) - measured, nothing mechanically mergeable

All 112 dupl fragments in internal/testing were classified by normalising type names and diffing:
- testdefaults/*, testplanmodifier/*, testvalidator/* (12 each): one type per typed public interface
  (`DefaultBool`, `PlanModifyString`, `ValidateParameterInt64`...). Method names differ per type, Go generics cannot
  parameterise method names -> cannot merge.
- testprovider/*config validators (7) and *with* combos: each satisfies a distinct interface set; merging into an
  "implements everything" type changes which optional-interface paths fwserver takes (type assertions) -> not equivalent.
  Verified one pair: ephemeralresourcewithconfigureandclose.go vs ...andrenew.go differ only in the Close/Renew interface.
- testschema/attributewith{Type}{default,planmodifiers,validators}.go: per fwxschema interface (AttributeWithXDefaultValue,
  AttributeWithXPlanModifiers, AttributeWithXValidators). A merged type would make fwserver run plan-modification paths
  for validator-only fixtures (different log lines/paths) -> not equivalent.
- testtypes/*withsemanticequals.go (12), *withvalidateattribute.go: per typed interface (XSemanticEquals method names)
  and per embedded base type; Go cannot embed a type parameter -> cannot merge.
- testschema/nested_attribute_object_{,list_,map_,set_}custom_type.go: 4 files byte-identical modulo the type name and
  String() text (93 lines each). Looked mergeable, REJECTED: fwserver plan-modification tests deliberately nest two
  different custom types (outer ListNestedObjectCustomType, inner NestedObjectCustomType; e.g.
  internal/fwserver/attribute_plan_modification_test.go:244-320) to prove each nesting level keeps its own CustomType.
  Merging would weaken that test.
- Optional micro-dedupe (Description/MarkdownDescription boilerplate via an embedded exported `Describer` struct):
  rejected; an embedded *unexported* type would make cmp.Diff panic on schemas holding these fixtures, an exported one
  adds API to every fixture, and no call site sets the fields anyway (see section 10).

## 4. tfsdk/value_as_test.go typed helpers (item 4)

13 helpers. After `go fix -newexpr` + `go fix -inline` + deletion: the 6 single-pointer helpers vanish
(newBool/Float32/Float64/Int32/Int64/StringPointer); call sites become `new(false)`, `new(0.0)`, `new("")` or, where the
default type differs, `new(float32(0.0))`, `new(int32(0))`, `new(int64(0))`. File diff: +13/-37.
The 6 `newXPointerPointer` helpers can optionally become `new(new(x))` / `new(new(int64(0)))` (+12/-42 more, tests pass);
readability is a reviewer call, so it is listed as optional inside G1. `newBigFloatPointerPointer` should stay
(`new(new(big.Float).SetUint64(0))` is worse).

## 5. perfsprint (item 5)

golangci perfsprint (scratch config): 704 hits = error-format 681 (673 test, 8 non-test), bool-format 13 (12 test,
1 non-test), string-format 8 (1 test, 7 non-test), integer-format 2 (non-test). Test total 686 as in the brief.
- `fmt.Errorf("<literal without %>")` with no args in *_test.go: exactly 673 sites in 175 files
  (`grep -rhoE --include='*_test.go' 'fmt\.Errorf\("([^"\\%]|\\.)*"\)' . | wc -l`), 0 backquoted, 0 with a `%`.
  Equivalent to errors.New (fmt.Errorf without %w returns errors.New(Sprintf(format)), same *errors.errorString and text).
  REJECTED as churn: perfsprint is not enabled in .golangci.yml, tests are not hot paths, 175 files of conflicts for
  zero behaviour or readability gain. The 3 in internal/testing/testtypes/invalid.go:23,41,71 can be folded into any PR
  that touches that file.
- bool-format: 12 in internal/fwserver/server_upgraderesourcestate_test.go (144,148,238,258,322,331,396,575,579,646,716,720)
  on `bool` fields/vars -> `strconv.FormatBool` identical (fmt only consults Formatter for %t, plain bool has none);
  1 non-test types/basetypes/bool_value.go:153 (`b.value` is bool) identical.
- integer-format: types/basetypes/int32_value.go:153 (`%d` on int32 -> needs `strconv.FormatInt(int64(i.value), 10)` or
  `strconv.Itoa(int(i.value))`), int64_value.go:153 -> `strconv.FormatInt(i.value, 10)`; identical output.
- string-format: all 8 operands are plain `string` (no Stringer): server_configurestatestore_test.go:73,
  fwserver/server_planresourcechange.go:392, server_readresource.go:194, server_updateresource.go:195 (`.Raw.String()`),
  privatestate/data.go:416 (strings.Join), toproto5/getproviderschema.go:143 (strings.Join), stringdefault/static_value.go:31
  (`defaultVal string`), basetypes/object_value.go:109 (`name` from ranging a map[string]) -> concatenation identical.
  Verdict: equivalent, but style-only; fold into PRs that touch those files (non-test ones belong to areas A/B/C/E).

## 6. Test-hygiene sweeps (item 6) - what was swept vs sampled

Exhaustive (AST or grep over all 340k test lines):
- Loop-variable copies: `x := x` 0, `a, b := a, b` 0 (copyloopvar works); `go fix -forvar` finds nothing.
- reflect.DeepEqual in tests: 0 (cmp.Diff 3431 uses, cmp.Equal 47).
- Tautological `cmp.Diff(x, x)` / `x.Equal(x)`: 0.
- t.Skip: 0. Benchmarks: 7 (2 helpers).
- Table fields never read (custom AST checker: fields of struct types declared inside a Test func that never appear in a
  selector in that func): 390 raw hits, almost all reflection target structs populated via tfsdk tags (NestedString x149,
  Name, ID...). Real table-case fields: 7 families -> G5. Includes one misleading expectation (fwserver
  TestServerListResource `expectedError: "config cannot be nil"` - never asserted; server.ListResource replaces a nil
  Config with a null config and emits no error, internal/fwserver/server_listresource.go:91-101).
- Test funcs with no assertion and no t.Run: 3 -> G3 (TestNumber_int16/uint16/uint32 bodies are only t.Parallel()).
- Expected computed with the function under test (heuristic over 2846 table-driven tests): 14 -> G9.
- `make([]T, 0, n)` followed by index-assignment in `range` over that slice: 1 -> G4 (benchmark bug).
- AddError/AddWarning with a printf verb in the *summary*: 51 sites / 27 files -> G6.
Sampled:
- Large duplicated expected literals: 1517 dupl fragments in tests, 880 within the same file. Sampled the 3 largest
  (server_planresourcechange_test.go 8609-10513 vs 13600-15504 = 1905 lines; fromproto5/dynamic_value_test.go 154-825 vs
  826-1497; all 33 dupl pairs in toproto6/getproviderschema_test.go): every pair differs only in list/set/map naming
  (`list_attribute` vs `set_attribute`, NestingModeList vs Set): per-type coverage, 0 literally identical pairs. Not a
  mechanical shared-fixture opportunity.
- Asymmetric custom comparer in fwserver TestServerListResource (server_listresource_test.go:298-306) loops only over
  `a`: initially looked like "a missing expected diagnostic compares equal". Probe showed cmp's dynamic symmetry check
  (always run on the 1st and 2nd comparer call of a Diff) panics "non-deterministic or non-symmetric function detected",
  and every case has <= 2 events, so the test does fail loudly. Downgraded to a note inside G5 (fragile, misleading panic).
- `expected, _ := types.SetValue(...)` style (48 sites in fromproto5/6 arguments_data_test.go, inside validator fixtures):
  if construction failed the validator would return an error and the test would fail -> not a swallowed error.

## 7. Tooling and CI (item 7)

- tools/ module: tools/go.mod (51 lines, 43 indirect) + tools/go.sum (504 lines) + tools/copywrite.go (blank import under
  `//go:build generate` + `//go:generate go run github.com/hashicorp/copywrite headers -d .. --config ../.copywrite.hcl`).
  Trade-off of Go 1.24 `tool` directives (tier-3 note, NOT recommended):
  (a) tool directive in the MAIN go.mod: clearly worse for a library. Tool requirements become part of the library's
      module graph; the main go.sum (97 lines) would absorb most of tools/go.sum (504 lines), and MVS would raise
      consumers' minimums for shared deps (e.g. github.com/mattn/go-isatty main v0.0.20 vs tools v0.0.21). Rejected.
  (b) tool directive inside the existing tools/ module: `go get -tool github.com/hashicorp/copywrite@v0.25.3` in tools/,
      delete tools/copywrite.go (13 lines), Makefile `generate:` becomes `cd tools; go tool copywrite headers -d .. --config ../.copywrite.hcl`,
      CONTRIBUTING.md:110 text updated; dependabot keeps updating /tools. Net about -12 lines. Same copywrite version,
      same flags. But terraform-plugin-testing, -sdk, -log use the byte-identical tools/copywrite.go + go:generate layout
      and none of them (nor terraform-plugin-go) has a `tool` directive (fetched their tools/go.mod and
      tools/copywrite.go from GitHub raw). Divergence from the shared HashiCorp devex layout outweighs ~12 lines.
      Only worth doing across all devex repos at once.
- Workflows: all 9 pinned action SHAs match their version comments upstream (checked with `git ls-remote --tags` +
  peeled tags: checkout v7.0.1, setup-go v7.0.0, upload-artifact v7.0.1, lock-threads v6.0.2, golangci-lint-action v9.3.0,
  goreleaser-action v7.2.3, setup-copywrite v1.1.3, setup-terraform v4.0.1, changie-action v3.0.1). No stale comments.
- `go-version-file: 'go.mod'` used consistently everywhere except the `test` job, which correctly uses the explicit
  1.26/1.27 matrix.
- Path filters: all named files exist. Gaps (notes only): ci-go.yml paths omit `go.sum`; nothing builds the tools/
  module, so daily dependabot bumps of tools/go.mod are merged without any CI compiling them (compliance.yml installs
  copywrite via hashicorp/setup-copywrite, not from tools/go.mod, so the CI copywrite version and `make generate`'s
  v0.25.3 can drift). golangci-lint-action has no `version:` (installs latest) and actionlint is `@latest`: non-reproducible
  but deliberate; notes only.
- terraform-provider-corner jobs: 2 x 26 lines, exactly 4 differing lines (job key, name, test path, matrix var).
  A matrix dimension cannot express it (TF_VERSIONS_PROTOCOL_V5 and _V6 are different org-level lists); a reusable
  workflow would save <= 8 lines and add a file + indirection. Rejected.
- Makefile: `website`, `website/local`, `website/build-local` call `$(MAKE) -C website`, and website/ does not exist;
  `make website` fails ("make[1]: *** website: No such file or directory.  Stop."). CONTRIBUTING.md:50 still says the
  repo contains the docs and links ../website/README.md (missing). -> G10. (Same residue exists in terraform-plugin-testing,
  -sdk, -log Makefiles; terraform-plugin-go's Makefile has none, so there is precedent for removing it.)
- .copywrite.hcl header_ignore: `.github/labeler-*.yml` and `.release/*.hcl` match 0 files (others match >= 1;
  `.goreleaser.yml` exists). NOT proposed: the file is byte-identical (modulo copyright_year) to terraform-plugin-testing's,
  and terraform-plugin-go and -sdk carry the same two entries: a shared template; the entries are harmless no-ops.
- dependabot.yml: gomod `/` and `/tools` daily, github-actions weekly grouped. Fine.
- README: all 7 links resolve (5 via one redirect to developer.hashicorp.com / go.dev). Canonicalising is optional.

## 8. .golangci.yml (item 8)

- `golangci-lint config verify` (v2.15.0) exit 0; all 16 enabled linters exist in v2.15 and none is deprecated
  (`golangci-lint linters -c .golangci.yml`).
- `exclusions.paths` / `formatters.exclusions.paths` (`third_party$`, `builtin$`, `examples$`): no such directories exist
  (`find . -type d -name third_party -o -name builtin -o -name examples` = 0); `golangci-lint migrate` boilerplate, no-op.
- Presets: `comments` masks staticcheck ST1000 (15), ST1020 (51), ST1021 (115) (doc-comment form; out of scope except
  ST1000 below). `legacy` / `std-error-handling` mask errcheck on Close/Flush/print/os.Setenv/Remove: grep finds 0 such
  unchecked calls in the module, so they hide nothing today. `common-false-positives` targets gosec (not enabled): no-op.
  SA1019 x49 and U1000 x4 are suppressed by per-line //nolint with reasons (intentional).
- ST1000: 77 of 78 packages have a package comment (`go list -f '{{.Doc}}'`); the only one without is the public package
  `statestore` -> G11. internal/testing/testtypes/doc.go has the malformed `// types contains ...` -> folded into G7.

## 9. Bugs in tests (item 9)

- Benchmark bug G4 (types/basetypes/set_value_test.go:66-89): all six BenchmarkSetTypeValidate{10..1000000} validate an
  EMPTY set. Probe mirroring the helper prints `len(elements)=0 set elements=0`; benchmark output shows B/op equal to the
  amortised setup slice (1e6 x 32 B / b.N: 16027 B/op at -benchtime=2000x, 8025 B/op at 4000x) and 1 alloc/op.
- Fixture `Equal` methods: every `Equal` in internal/testing asserts its own receiver type (awk scan) and the schema
  fixtures delegate to fwschema.AttributesEqual like production types. 232 nil-guarded fixture methods all call the same
  field they guard (AST check `fieldcheck`).
- `var _` assertion coverage: every testprovider type has one. Missing only on testtypes *TypeWithValidateAttribute* /
  *WithValidateParameter* Type fixtures (their Value types are asserted; fine) and on the SingleNested fixture, whose
  assertion block is a copy of the Set file's and asserts the wrong types (G7). Only copy-paste assertion in the tree
  (scan: every `var _ X = T{}` in internal/testing names a type declared in the same file, except that one).
- testschema.NestedAttribute.ApplyTerraform5AttributePathStep says "ListNestedAttribute" for Map and Set nesting
  (nested_attribute.go:51,59); real types say MapNestedAttribute / SetNestedAttribute; no test observes the fixture text
  (fix applied on a branch, 18 testschema-using packages pass) -> G7.
- internal/testing/planmodifiers/planmodifiers.go:23,36 uses AddError like printf (part of G6).
- Empty tests G3; tautological tests G9; misleading expectedError in G5.

## 10. Rejected ideas ("looks equivalent / worthwhile but is not")

1. Merge the 4 NestedObjectCustomType variants: distinct types are load-bearing in nested custom-type tests (section 3).
2. Remove never-set DescriptionMethod/MarkdownDescriptionMethod from 43 fixtures (draft: ~43 files, roughly -300 net lines;
   the regex draft was incomplete and discarded): fwserver logs `validator.Description(ctx)` under logging.KeyDescription
   (internal/fwserver/block_validation.go:225 etc.), so the fields are the only hook for future log-content tests, and every
   declarative fixture in the package follows the one-func-field-per-interface-method convention. Not worth it.
3. Merge per-type testdefaults/testplanmodifier/testvalidator/testprovider fixtures: per-interface method names.
4. errors.New for 673 constant fmt.Errorf in tests: equivalent but churn (section 5).
5. `go tool` directive for copywrite: diverges from sibling repos; main-module variant leaks tool deps into the library graph.
6. Reusable workflow for the two corner jobs: <= 8 lines saved.
7. Delete .copywrite.hcl stale entries: shared template across devex repos, harmless.
8. "Asymmetric comparer means missing diagnostics pass": false, cmp's symmetry check panics (section 6).
9. "Path.AtTupleIndex has no real test": false. A mutant (append PathStepAttributeName) passes the tautological
   TestPathAtTupleIndex but is caught by internal/reflect TestFromSlice/tupleElemWithValidateAttributeError. G9 kept at
   tier 3 for the path package's own tests only.
10. Delete the 4 U1000 struct fields in internal/reflect tests: intentional (//nolint:unused) reflection fixtures.
11. Treat list/*_test.go no-op resources as dead: they are compile-time interface checks.
12. Replace the 48 `expected, _ := types.XValue(...)` with XValueMust: style only; a failed construction already fails the test.
13. A bloop/b.Loop rewrite folded into the G4 bug fix: kept separate (rule: never fold a bug into a refactor); G12.

## 11. Cross-area leads for the coordinator (not filed as G findings)

- internal/privatestate/data.go:69,83,163,176 "is is not valid UTF-8/JSON" and :380 "does not use a a period":
  duplicated-word typos in diagnostic Detail text (observable; 17 test lines mirror them). Area C/B.
- perfsprint non-test sites verified equivalent (section 5): types/basetypes Bool/Int32/Int64 String(),
  stringdefault/static_value.go:31, fwserver identity diagnostics, privatestate panic text, toproto5:143.
- reflecttypeassert x12 in internal/reflect tests (section 1, point 5). Area C.
- `*new(time.Time)` -> `time.Time{}` (gocritic newDeref) in 6 test files: internal/fwserver/server_openephemeralresource_test.go:223,
  server_renewephemeralresource_test.go:197, internal/toproto5|6/openephemeralresource_test.go:93,
  renewephemeralresource_test.go:43. Fold-in only.
- Brief correction: CI matrix is 1.26/1.27 (section 0).

## 12. Please double-check (verifier)

- G1: that golangci-lint `unused` (project config, tests linted by default) flags the 10 helpers exactly like staticcheck
  U1000 did. I did not run golangci-lint on the copy (instruction), only staticcheck.
- G4: maintainers must pick sizes after the fix; SetType.Validate's duplicate detection is a nested loop
  (types/basetypes/set_type.go:185 outer loop, :206 inner loop from indexOuter+1), so 100000/1000000 elements would be O(n^2) and very slow.
- G5: whether maintainers prefer `testCase.request` in the 8 Get* calls (chosen in the draft, 8 one-line edits) or
  deleting the field (8 decls + 38 assignments).
- G9: proposal only adds cases; converting the 14 tautological tests to literal Steps()/String() expectations is optional.
- G11: wording of the new statestore package doc is a maintainer choice (mirrors list/doc.go).

## 13. Reproduction commands (run at repo root unless noted)

- pointer helper calls: `grep -rn --include='*_test.go' -E '\bpointer(\[[^]]*\])?\(' . | grep -v 'func pointer' | wc -l` -> 192 lines (198 occurrences with -o, minus 6 definitions)
- helper defs: `grep -rn --include='*_test.go' '^func pointer\[T any\]' . | wc -l` -> 6; `grep -n '^func new' tfsdk/value_as_test.go | wc -l` -> 13
- interface{}: `grep -rho --include='*.go' 'interface{}' . | wc -l` -> 554 (tests 185); after go fix -> 0
- reflect.TypeOf: 43 before, 12 after (31 rewritten)
- constant fmt.Errorf in tests: `grep -rhoE --include='*_test.go' 'fmt\.Errorf\("([^"\\%]|\\.)*"\)' . | wc -l` -> 673; `-l` -> 175 files
- loop copies: `grep -rnE --include='*_test.go' '^\s*([A-Za-z_][A-Za-z0-9_]*) := \1\s*$' . | wc -l` -> 0
- reflect.DeepEqual in tests: `grep -rn --include='*_test.go' 'reflect.DeepEqual' . | wc -l` -> 0
- printf-style AddError: `grep -rnE --include='*.go' '\.(AddError|AddWarning|AddAttributeError|AddAttributeWarning)\("[^"]*%[a-zA-Z]' . | wc -l` -> 51 (`-l` 27 files)
- DescriptionMethod setters: `grep -rn --include='*.go' -E '^\s*(Markdown)?DescriptionMethod:' . | grep -v '^./internal/testing/' | wc -l` -> 0
- zero-ref testtypes: `grep -rn --include='*.go' -E 'testtypes\.(Set)TypeWith(Validate|ValidateAttribute)Error\b' . | wc -l` -> 0
- empty number tests: `grep -n -A2 '^func TestNumber_\(int16\|uint16\|uint32\)(' internal/reflect/number_test.go`
- packages without doc: `for p in $(go list -f '{{.Dir}}' ./...); do go list -f '{{.Doc}}' $p | grep -q . || echo $p; done` -> statestore only
- stale header_ignore: `bash -O globstar -O nullglob -c 'f=(.github/labeler-*.yml); echo ${#f[@]}; g=(.release/*.hcl); echo ${#g[@]}'` -> 0 0
- website dir: `ls -d website` -> No such file or directory
- benchmark: `go test -run '^$' -bench BenchmarkSetTypeValidate -benchtime=2000x -benchmem ./types/basetypes` (flat ns/op, 1 alloc/op)

## 14. Scratch artifacts

- copy: `.../audit/work/G/copy` (scratch git; master = all go fix steps; branches dead-testtypes, dead-table-fields,
  adderror-printf, example-rename, fixture-msg, empty-number-tests, tuple-index-test, pp-experiment; all from baseline b404da0
  except pp-experiment which sits on the newexpr PR commit 0b13e18)
- logs: `.../audit/work/G/analyzers.log`, `final-suite.log`, `test-final.log`, `vet-final.log`, `staticcheck-copy.txt`
- patches: `.../audit/work/G/patches/*.patch`
- tools: `.../audit/work/G/{lister,docprobe,fieldcheck,tablecheck}` (small stdlib-only AST programs)

## 15. G12 measurement (finished after the restart) and findings index

- G12 draft (branch `bloop`, patch `.../work/G/patches/G12-bloop.patch`): 2 files, +7/-8. `go vet` and `go test ./types/basetypes`
  ok. BenchmarkSetTypeValidate at -benchtime=2000x: before 24 B/op (size 10) vs 16027 B/op (size 1e6); after b.Loop
  24 B/op for both sizes, so the setup allocation is no longer measured. (ns/op for the 1e6 case stays higher, about 1.7 us,
  most likely GC scanning the 32 MB live, mostly-unused slice that G4's bug creates.)
- Findings index: G1 newexpr PR (tier 1); G2 dead testtypes fixtures (tier 2); G3 empty TestNumber_* tests (tier 2);
  G4 benchmark empty-set bug (tier 3, bug); G5 never-read table fields; G6 printf-style AddError; G7 fixture copy-paste
  slips; G8 example rename; G9 tautological At* tests; G10 website Makefile targets; G11 statestore package doc;
  G12 b.Loop; G13 test-only go fix analyzers (all tier 3).
