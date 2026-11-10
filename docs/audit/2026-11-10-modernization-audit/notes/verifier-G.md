# Verifier notes, area G (test infrastructure, test hygiene, tooling, CI, docs links)

Verifier: independent. Repo /home/user/terraform-plugin-framework at 00badb1, untouched (`git status --short` = 0 lines at the end).
Scratch: `drafts/V-G/`.
- `copy/`: fresh `tar --exclude=.git` copy, with a scratch-only git (baseline 1fcc9f2). Commits: newexpr raw, inline, delete helpers (=G1), G2 mine, G13 mine, then the author patches G3, G5, G6, G7a, G7b, G8, G9, G12 (HEAD 594f4b3).
- Logs: `suite-cumulative.json` (go test -json), `baseline-6pkgs.json` and `baseline-9pkgs.json` (baseline identity runs), `sc-default-cumulative.txt`, `sc-all-cumulative.txt`, `golangci-changed.txt`, `gofix-pristine.json`, `diff-{fmtappendf,rangeint,waitgroup}.patch`, `G1-mine.patch`, `g4-bench-before.txt`, `g4-bench-after-fix.txt`, `g12-bench-after.txt`, `cover-reflect-{base,g3}.out`.
- Probe programs (stdlib only, each in its own dir): `probe-newexpr/` (static types), `probe-g4/`, `emptytests/` (assertion-free Test sweep), `docprobe/` (go/doc example attachment), `tablefields/` (never-read table fields).

Verdicts: all 13 items are CONFIRMED (0 PLAUSIBLE, 0 INVALID). The corrections are listed in verified-G.json.

## Whole-suite run (the one allowed) on the cumulative copy

Copy state: G1 + G2 + G13 (re-applied by me) + author patches G3, G5, G6, G7a, G7b, G8, G9, G12 = 113 files, +365/-676.

| check | result |
|---|---|
| `gofmt -l .` and `gofmt -s -l .` (Go 1.26 gofmt from the toolchain dir; PATH gofmt is 1.24.7) | clean |
| `GOTOOLCHAIN=auto go vet -p 2 ./...` | exit 0, no output |
| `GOTOOLCHAIN=auto go test -p 2 -count=1 -json ./...` | 28 s; 56 pkgs pass, 22 no-test; 15,387 test+subtest IDs pass, 0 fail, 0 skip |
| `staticcheck ./...` (default) | U1000 only the 4 pre-existing intentional fields (internal/reflect/helpers_test.go:24,25; struct_test.go:858,2196, shifted -1 by G5) |
| `staticcheck -checks all ./...` vs baseline staticcheck-all.txt (line:col normalised) | 285 -> 283; removed = testtypes/doc.go ST1000 (G7) and 1 SA1019 on the deleted SetTypeWithValidateError assertion (G2); 0 new |
| `golangci-lint run --allow-parallel-runners -j 2` on the 18 changed packages only (project config) | `0 issues.` |

Test identity: I ran `go test -json` at baseline in the real repo for all 15 packages any patch touches (function, tfsdk, internal/reflect, path, internal/privatestate, list, internal/fwschemadata, internal/toproto5|6, types/basetypes, internal/fromproto5|6, internal/fwserver, internal/proto5|6server; about 13 s total) and compared them with the suite JSON. 7,043 baseline IDs are all present with the same status. The only additions are `path TestPathSteps/tuple-index` and `path TestPathString/AttributeName-TupleIndex` (G9). Statically, the removed lines in the cumulative diff contain no table-case key, and the only top-level Test/Example/Benchmark change is the ExampleResource_listable -> ExampleListResource rename (examples without // Output do not run).

## G1 (newexpr): reproduced byte-identically

```
go fix -p 2 -newexpr ./...  -> 42 files changed, 225 insertions(+), 212 deletions(-)
go fix -p 2 -inline ./...   -> tfsdk/value_as_test.go only, 7 insertions(+), 7 deletions(-)
   new(float32(0.0)) new(float32(12.3)) new(int32(0)) new(int32(12)) new(int64(0)) new(int64(12)) new(int64(0))
delete 6 pointer_test.go + strPtr + 6 tfsdk single-pointer helpers -> 8 files, -89
baseline..HEAD: 42 files changed, 206 insertions(+), 282 deletions(-) (36 M, 6 D); value_as_test.go +13/-37
git apply G1-newexpr-pr.patch on baseline, then diff against mine: EMPTY
```
- newexpr diagnostics (my `go fix -json`): 205 "call of F(x) can be simplified to new(x)" + 13 "F can be an inlinable wrapper around new(expr)" = 218.
- Call counts: `grep -rho --include='*_test.go' -E '\bpointer\('` = 198 (on 192 lines; definitions are `pointer[`, so they do not match). Adding 1 strPtr, 6 newBool/Float64/StringPointer calls and 7 newFloat32/Int32/Int64Pointer calls gives 212.
- Static types: TestValueAs's `target`/`expected` are `interface{}` (tfsdk/value_as_test.go:102-107). newexpr left the float32/int32/int64 helpers alone, and -inline added the conversions. A script checked that all 206 paired -/+ lines are pure renames (with conversions on exactly the 7 inline lines). No `pointer[T](...)` explicit instantiations exist. Probe output on Go 1.26 (pointer vs new): `*float64 *float64`, `*int *int`, `*int32 *int32` ('x'), `*int *int` (1<<40), `*main.I *main.I`, `**struct { X int }` twice, `*string *string` (typed const), and `true true` for distinct allocations.
- Deletion is mandatory. After newexpr+inline with the helpers kept: `staticcheck -checks U1000` flags all 13 (6x `func pointer is unused`, strPtr, newBool/Float32/Float64/Int32/Int64/StringPointer). `golangci-lint run ./function/ ./tfsdk/` with the project config: 8 issues, `unused: 8`. This answers the author's open question 1: golangci `unused` behaves like U1000.
- `//go:fix inline` directives: 0 in the repo and 0 in go-cmp v0.7.0, go-uuid v1.0.3, terraform-plugin-go v0.31.0 and terraform-plugin-log v0.10.0.
- Optional `new(new(x))` patch: applies, +12/-42, keeps the float32/int32/int64 conversions, and TestValueAs passes.
- Minor: "tfsdk/value_as_test.go:24-85" should be 24-81 (line 85 is inside `type personFrameworkTypes`).
- Tier: I suggest 2. It touches 42 files and is idiom adoption in test code only; the brief's tier-2 wording is "touches many files". It is otherwise maximal proof and zero risk.

## G2: reproduced byte-identically
- Script-applied deletion: 11 files, +11/-206. Applying the author patch to the G1 tree and diffing against mine gives an empty diff.
- 236 import lines of testtypes, all with the default name. Qualified refs `testtypes\.<T>\b` = 0 for all 12 types.
- Unqualified hits in testtypes: own decl, assertion and Equal (6 per wrapper), plus `XValue: in` composite keys in *withsemanticequals.go:53. Every *WithSemanticEquals struct embeds `basetypes.XValue` (grep of embedded-field lines), so these keys refer to the embedded basetypes field.
- Each of the 9 XType defines only `Equal` (method grep), and the XTypes are used 24-104 times each, so the wrappers can never be produced.

## G3
- AST sweep (`emptytests/`): 3,294 Test funcs, 3 never use t beyond Parallel/Helper: exactly TestNumber_int16/uint16/uint32.
- int16/uint16/uint32 appear in no test file except number_test.go.
- Coverage of internal/reflect, number.go:92.4 / 125.4 / 130.4 (in-range returns): count 0 at baseline, 1 with G3.

## G4 (= E12)
- Probe: `elementCount=10 len=0 iterations=0`, `elementCount=1000 len=0 iterations=0`.
- Baseline at -benchtime=2000x: 10 -> 44.99 ns, 24 B, 1 alloc; 1000 -> 40.90 ns, 40 B, 1 alloc; 1e6 -> 9507 ns, 16027 B, 1 alloc.
- With the fix (`make([]tftypes.Value, elementCount)`) plus G12, -benchtime=5x: 10 -> 6.9 us, 2184 B, 91 allocs; 100 -> 1.25 ms, 238 KB, 9,901 allocs; 1000 -> 79.4 ms, 24 MB, 999,005 allocs. Quadratic, so 10000 is about 8 s/op as well. The missingType element type is not a TypeWithValidate, so only duplicate detection runs and there is no error path.

## G5
- Every family was opened. Counts: providerMetaSchema 10 decls with 0 other occurrences; reflect struct_test.go:51 declared, never set, shadowed at :401; configureProviderReq 1 occurrence; Get* `request` 3+6+8+1 (proto5) + 3+7+9+1 (proto6) = 38 one-line `&tfprotovN.XRequest{}`, with the loop passing `new(...)`; parentType 16 assignments in TestUpsertChildTerraformValue (11 more in TestCreateParentTerraformValue are read via tc.parentType at :163); expectedError 1 assignment, and 'config cannot be nil' has 0 production hits.
- Correction: proto6server/server_listresource_test.go has 4 `expectedDiagnostics: diag.Diagnostics{}` (214, 233, 257, 274), not 3. The patch already removes all 9 lines (patch per-file stats: proto5 -10/+6, proto6 -12/+7).
- My AST sweep (`tablefields/`) of map/slice-of-anonymous-struct tables and named structs inside Test funcs reproduces the same 24 entries. All other hits are tfsdk-tagged reflection targets compared by cmp.Diff.
- Asymmetric comparer: go-cmp v0.7.0 `dynChecker.Next` checks the 1st and 2nd calls (then 4, 7, 11, ...). `callTTBFunc` runs f(y,x) in a goroutine whose panic is recovered, so the result is invalid and it panics "non-deterministic or non-symmetric function detected". The author's rejection #8 holds while each case has at most 2 comparer calls.

## G6
- `grep -rnE --include='*.go' '\.(AddError|AddWarning|AddAttributeError|AddAttributeWarning)\("[^"]*%[a-zA-Z]' .` gives 51 lines in 27 files (12 fwserver, 6 proto5server, 8 proto6server, 1 planmodifiers).
- 8 distinct summaries; each appears 0 times outside the AddError calls. Extra greps: path-first AddAttribute* = 0, New*Diagnostic with % = 0, calls split over lines = 0.
- This is a fixture defect (API misuse), not test data. The impact is cosmetic and only on failing runs, and the value is in the detail. Tier 3 is right.

## G7
- (1) The use sites `internal/fwserver/attribute_validation_test.go:1625,1740` assign it to `testschema.NestedAttribute.Type attr.Type`. So there is an implicit attr.Type check, but no ObjectTypable/ObjectValuable check. Wording fix only.
- (2) The 24 hits of 'to ListNestedAttribute' in tests are all in */schema/list_nested_attribute_test.go across 8 packages (provider/metaschema, provider/schema, resource, datasource, list, action, ephemeral, statestore) for the real types. None observes the fixture.
- (3) Baseline staticcheck has `internal/testing/testtypes/doc.go:4:1: package comment should be of the form "Package testtypes..." (ST1000)`, and it is gone after the patch.

## G8
`docprobe/` output. Baseline: `found example: ExampleResource_listable` / `attached: 0 of 1`. Renamed copy: `type ListResource <- ListResource` / `attached: 1 of 1`. It is the only `^func Example` in the repo.

## G9
- Script over the 14 functions gives (expected entries / tautological): expression x8 = 2/2 each; PathAtListIndex 3/3, AtTupleIndex 3/3, AtMapKey 3/3, AtName 3/2, AtSetValue 3/3; NewDynamicValue 4/3. That is 33 of 35 entries; TestPathAtName/empty and TestNewDynamicValue/nil are real.
- Mutation run (path/path.go AtTupleIndex appends `PathStepAttributeName("MUTANT")`). Without G9, `go test ./path/` is ok. With G9, TestPathSteps and TestPathString FAIL (`"test.MUTANT"`). `go test -run TestFromSlice ./internal/reflect/` also FAILs. Reverted afterwards.
- In-package pins: TestPathString covers AtListIndex x5, AtMapKey x5, AtName x6, AtSetValue x3. TestExpressionString covers all AtAny* and At*. TestExpressionSteps covers AtParent. AtTupleIndex appears only in TestPathAtTupleIndex.

## G10
- `make website` on the copy gives `make[1]: *** website: No such file or directory.  Stop.` and `make: *** [Makefile:8: website] Error 2`, exit 2. `make -n` is the same, because recursive $(MAKE) still runs.
- `git show d2548a9` ("chore: remove old docs (#1212)", 2025-08-28; 160 files, -26986): "Documentation has moved to https://github.com/hashicorp/web-unified-docs". This is the target for CONTRIBUTING.md:50.
- The finding misses some residue: `.github/CODEOWNERS:3-17` (/website/, data, public, content, docs, img, README.md rules; CODEOWNERS was last touched in #1320 without removing them) and `.gitignore:1-3` (website-preview, node_modules).
- Siblings: terraform-plugin-testing, -sdk and -log Makefiles have 3 `$(MAKE) -C website` targets and website/Makefile returns HTTP 404. terraform-plugin-go has 0 such targets.

## G11
`go list -f '{{.ImportPath}}|{{.Doc}}' ./...` gives 78 packages with 1 lacking a doc (statestore). list/doc.go equals sed(ephemeral/doc.go): its Renew/Close text is wrong, and area F already reports it as F7. Mirror only its first sentence, as G11 proposes.

## G12
- Go 1.26 `src/testing/benchmark.go` B.Loop doc: resets the timer on the first call; keeps alive the "arguments to and results from function calls and assigned variables within the loop"; runs the benchmark function once per measurement.
- After the patch (2000x): SetTypeValidate10 = 35.88 ns, 24 B, 1 alloc; SetTypeValidate1000000 = 44.39 ns, 24 B, 1 alloc. Before: 9507 ns and 16027 B. ObjectValueToTerraformValue1000 = 279,744 ns, 293,059 B, 2016 allocs, against 280,170 ns, 293,149 B, 2017 allocs at baseline.

## G13
- `go fix -diff`: fmtappendf touches 1 file. Its diff printer shows a non-minimal +35/-35 hunk, but there are 6 real sites (data_test.go:49, 114, 233, 263, 758, 775). rangeint: 5 sites in 5 files. waitgroup: 2 sites in 2 files. After applying: 6 files, +15/-19, byte-identical to the author patch.
- Go 1.26 `sync.WaitGroup.Go` source: `defer func(){ if x := recover(); x != nil { panic(x) }; wg.Done() }(); f()`. So Done is skipped on panic (the original deferred Done would run), and it runs on a normal return or Goexit. The two bodies only `t.Error` and `return`, so the change is equivalent. The author's "documented as Add(1) + go with deferred Done" is slightly inaccurate.
- Overlap: A16 (area A) edits the same TestServerCancelInFlightContexts loops (proto5server serve_test.go:29-43, proto6server :28-42).

## "go fix verification" table: every row re-measured
My own `go fix -json` on a pristine worktree, deduplicated by position:
any 554 diags / 269 files / 185 in tests; reflecttypefor 31 / 13 / 26; mapsloop 4 / 3 / 0; slicescontains 3 / 2 / 0; newexpr 218 / 42 / 218; fmtappendf 6 / 1; rangeint 5 / 5; waitgroup 2 / 2.
Applied one at a time: any 269 files +553/-553 (47 test, 222 non-test, 0 `interface{}` left); reflecttypefor 13 files +30/-30 (11 test, 2 non-test, 12 `reflect.TypeOf(` left); mapsloop 3 files +7/-12; slicescontains 2 files +25/-37. `go vet ./...` on that state is clean and gofmt is clean. Everything matches the author's table.

## Author's rejected notes: fact checks
- Fixture duplication: the 4 testschema nested_attribute_object_*custom_type.go files are 93 lines each and identical after normalising the type-name prefix. The outer ListNestedObjectCustomType / inner NestedObjectCustomType nesting in fwserver/attribute_plan_modification_test.go:244-320 is real. The rejection is a judgement call and its facts are correct.
- perfsprint: 673 constant fmt.Errorf in 175 test files (+8 non-test = 681), 0 backquoted. Go 1.26 `fmt.Errorf` returns `errors.New(format)` when no formatting is needed, so the rewrite is equivalent. perfsprint is not enabled in .golangci.yml. "Churn" is fair.
- copywrite `tool` directive: tools/go.mod has 51 lines (43 `// indirect`), tools/go.sum 504 lines, main go.sum 97 lines, go-isatty is v0.0.20 in main and v0.0.21 in tools, and copywrite is v0.25.3. terraform-plugin-testing, -sdk and -log have the same tools/copywrite.go (modulo the header) and no `tool` directives; terraform-plugin-go has none either. Correct.
- DescriptionMethod: 43 fixture files declare it, there are 0 setters outside internal/testing, and fwserver logs `Description(ctx)` under logging.KeyDescription (block_validation.go:224; 82 non-test sites). Correct.
- CI matrix `['1.26', '1.27']` at .github/workflows/ci-go.yml:82. Correct, so new(expr) in tests is safe on every CI toolchain.
- The 9 pinned action SHAs: `git ls-remote` with peeled tags gives MATCH for all 9 (checkout v7.0.1, setup-go v7.0.0, upload-artifact v7.0.1, lock-threads v6.0.2, golangci-lint-action v9.3.0, goreleaser-action v7.2.3, setup-copywrite v1.1.3, setup-terraform v4.0.1, changie-action v3.0.1).

## Cross-area leads (not filed)
- SetType.Validate duplicate detection allocates about n^2 (999,005 allocs/op and 24 MB/op for 1000 elements) through tftypes.Value.Equal. Area E (E12 mentions the cost) should decide whether it matters for real configurations.
