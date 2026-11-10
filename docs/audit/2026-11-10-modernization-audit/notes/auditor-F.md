# Area F working notes: public lifecycle packages and provider-defined functions

Auditor F. Repo /home/user/terraform-plugin-framework @ 00badb1. The repository was not modified; `git status --short` is empty after the work.
All scratch work is under `drafts/F` (written `W` below).
- Full copies of the repo with draft fixes: `W/fwcopy` (F1 + F3 + F5 drafts) and `W/fwcopy2` (F11 draft).
- Probe modules use `replace` to point at either the repo or a copy: `W/probe-listreturn`, `W/probe-lockrace`, `W/probe-listnoident`, `W/probe-nilparam`, `W/probe-args`, `W/probe-assert`.
- AST field scanner: `W/fieldscan`. Stale type name scan: `W/stale_type_names.sh`. Findings generator: `W/build_findings.py`.

## What I swept

Read exhaustively:
- function/: all 47 non-test files. The 36 twins (12 `_parameter.go`, 12 `_return.go`, 12 `_parameter_validator.go`) were checked by normalised diff (type name replaced by a placeholder, then md5 and pairwise diffs). arguments_data.go, result_data.go, func_error.go, definition.go, function.go, run.go, metadata.go, parameter.go, parameter_validation.go, return.go and doc.go were read line by line.
- list/ (6 files), statestore/ (14 files), resource/ (21 files) and provider/ (9 files): read in full.
- datasource/ (9), ephemeral/ (11), action/ (10): read in full except the long field docs of create/update/open/renew/modify_plan. I skimmed those for pre-population claims and checked them with the AST field scan.

The coverage of these 4 reads (internal/ only):
- AST scan of every exported `*Request` struct field in area F. It lists which fields internal/ non-test code sets through composite literals; fields missing from literals were then grepped for assignments.
- AST scan of every exported `*Response`/`*Result`/`*Stream` field. It lists which fields internal/ code never reads.
- Result: every field is populated or read, except the 4 unused list.* types (F8) and provider.MetadataResponse.Version (documented as unused, so intentional).

Sampled or not done:
- The fromproto5/6 ArgumentsData type switch: only read where it fills function validator requests (positions). That code belongs to another area.
- fwserver files: read only where they consume area F types (callfunction, listresource, listresources, configureprovider, configurestatestore, import, move, upgrade nil handling).

## Commands reproducing the counts

```
# twin normalisation (function/)
for t in bool dynamic float32 float64 int32 int64 list map number object set string; do U=<Capitalised>; for k in parameter return parameter_validator; do sed -e "s/$U/TYPE/g" -e "s/$t/type/g" function/${t}_$k.go > W/norm/${t}_$k.go; done; done; md5sum W/norm/*
#   -> parameter: 12 distinct; return: 12 distinct; parameter_validator: 10 identical + int64 (normalisation artefact) + object (doc typo, F9)
# identical ValidateImplementation bodies (F11)
for t in list map set object; do sed -n '/^func (p [A-Za-z]*Parameter) ValidateImplementation/,/^}/p' function/${t}_parameter.go | sed "s/[A-Z][a-z]*Parameter)/XParameter)/" | md5sum; done   # 4 equal; same for Return
# NewResultData callers (F1/F2)
grep -rn "\.NewResultData(" --include=*.go . | grep -v _test.go        # only internal/fwserver/server_callfunction.go:34
grep -rn --include=*_test.go -o "function\.[A-Z][A-Za-z0-9]*Return{" . | grep -v StringReturn | awk -F: '{print $1}' | sort | uniq -c   # only function/ files
go test ./function/ -count=1 -coverprofile=c.out; go tool cover -func=c.out | grep NewResultData         # 0.0% x12
go test ./internal/fwserver/ -count=1 -run CallFunction -coverpkg=./function/ -coverprofile=c2.out       # String 60.0%, other 11 0.0%
go test ./internal/proto6server/ -count=1 -run CallFunction -coverpkg=./function/ -coverprofile=c3.out   # same
# list unreferenced API (F8)
grep -rn "ListResultsStreamDiagnostics" . | grep -v "^./.git/"           # 2 lines (definition + doc)
grep -rn "list\.ConfigureRequest\|list\.ConfigureResponse\|list\.MetadataRequest\|list\.MetadataResponse" --include=*.go .   # 1 (comment, internal/fwserver/server.go:48)
diff <(sed -e 's/ephemeral resource/list resource/g; s/Ephemeral resource/List resource/g; s/EphemeralResource/ListResource/g; s/ephemeral/list/g; s/Ephemeral/List/g' ephemeral/doc.go) list/doc.go   # empty; same for configure.go, metadata.go
# stale default parameter name docs (F9)
grep -n 'default to "param"\|default name will' function/*.go | grep -v _test     # 27 lines, 13 files
grep -rn "varparam\|param%d" --include=*.go internal/ | grep -v _test.go            # 0: no defaulting code exists
grep -ln "does not have a name" function/*_test.go | wc -l                           # 13 test files pin the error
# stale type names in docs (F10)
W/stale_type_names.sh     # 5 real hits; the 2 provider/configure.go hits are qualified statestore.* refs (false positives)
# Configure call sites / deferral scope (F10)
grep -rln "resource.ResourceWithConfigure" internal/fwserver/*.go | grep -v _test   # 9 RPC files
grep -rn "s.deferred" internal/fwserver/*.go | grep -v _test | awk -F: '{print $1}' | sort | uniq -c
# unexported identifiers in area F non-test code: 3 (rngSource, generateLockID, generateLockInfoWho), all referenced
# typos: grep -rn --include=*.go satisifies . | wc -l -> 148 (24 files in function/); implmentation x2; provider-specifc x6 (area F)
```

## Findings (11)

| id | kind | tier | one line |
|---|---|---|---|
| F1 | bug | 1 | List/Set/MapReturn{CustomType only}: NewResultData uses nil ElementType, so the provider crashes on any Run error (both protocols); fix measured -12/+36 |
| F2 | test | 2 | NewResultData: 11/12 types never run in tests, CustomType branch never runs; the 10 deadcode leads are a test gap |
| F3 | bug | 2 | statestore.NewLockInfo races on a shared math/rand.Rand; 14 races under -race, duplicate IDs in 3/5 runs; fix with crypto/rand |
| F4 | bug | 2 | List resource for a framework resource without identity: GetProviderSchema accepts it, then NewListResult panics (nil schema) |
| F5 | bug | 3 | Custom Parameter with nil GetType: GetProviderSchema panics; ValidateImplementation checks the Return type only |
| F6 | bug | 3 | GetArgument(-1) and zero ResultData.Set panic instead of returning a FuncError |
| F7 | cleanup | 2 | list/ docs copied from ephemeral (Renew/Close in the package doc), wrong method names, ListResourceData doc points at action |
| F8 | cleanup | 3 | list.ConfigureRequest/Response and MetadataRequest/Response unused; ListResultsStreamDiagnostics unreferenced and untested (deprecate/test, never delete) |
| F9 | cleanup | 3 | 13 docs promise default parameter names removed in v1.7.0, plus 5 copy-paste doc errors |
| F10 | cleanup | 3 | about 25 stale or copy-paste doc comments across resource/datasource/provider/ephemeral/action/statestore |
| F11 | refactor | 3 | Twin measurement: only the collection/object ValidateImplementation bodies can be shared, net -56 lines (measured) |

## Rejected rewrites and wrong first passes

1. **Shared base struct for the 12 typed Parameter structs (or the Return structs).** Promoted fields cannot be set in composite literals, so `BoolParameter{Name: "x"}` would stop compiling. It would also change the reflect-visible field list and order. Rejected.
2. **Generic NewResultData helper.** It needs a per-type constructor and a `ValueFrom<T>` method value, so it ends up longer than the 9-line twins. Rejected.
3. **recvcheck on ResultData (mixed receivers).** Equal and Value must keep value receivers. go-cmp only uses a `(T) Equal(T) bool` method when the receiver kind matches; with a pointer receiver, cmp.Diff on a `RunResponse` in provider tests would fall through to the unexported `value` field and panic. Method sets are public API. No action.
4. **`return NewFuncError(msg)` instead of `funcErr = ConcatFuncErrors(funcErr, NewFuncError(msg)); return funcErr`** (arguments_data.go, 4 early returns). Equal by value only because every message is a non-empty constant: ConcatFuncErrors returns nil for an empty text with no argument, and it allocates a new pointer. It is equivalent, but too small to file. It could be folded into an arguments_data.go PR as a separate commit, not into F6.
5. **NoListResults as `var NoListResults iter.Seq[ListResult]`.** This changes the exported var's type (%T, reflection). Rejected.
6. **Nil guard in list.NewListResult as the fix for F4.** Provider code then calls `result.Identity.Set` on a nil `*tfsdk.ResourceIdentity` and panics there instead. Not a fix.
7. **F1 variants.** (a) Always preferring the CustomType's element type changes behaviour when both fields are set and disagree. (b) Building via `CustomType.ValueFromTerraform` calls a different user method. The nil-only guard is minimal.
8. **F3 via `uuid.GenerateUUID()`.** Its error path would produce an empty LockID, which the StateStore docs define as "locking unsupported". Rejected in favour of `crypto/rand.Read` + `uuid.FormatUUID`.
9. **First-pass claim that 4 numeric parameter types lack `ParameterWith<T>Validators` assertions.** False: my regex `[A-Za-z.]+` excluded digits. All 12 parameter types have all 3 assertions, and a compile probe (`W/probe-assert`) confirms they satisfy the interfaces. Withdrawn.
10. **DeferredReason.String (5 files) switching on literals `case 0: case 1:` instead of the named constants.** Equivalent, but negligible value. Not filed.
11. **Consolidating the 24 parameter/return test files (5,324 lines) into one generic table test.** It would save about 3,000 lines but breaks the per-type file layout used across the whole repo, including the schema packages. That is a test-wide style decision for the coordinator. Not filed.
12. **ObjectReturn in F1.** Its doc requires AttributeTypes, and the error path does not panic. Left as an optional extension, noted in F1.

## Leads (232), classified

- **deadcode 23**
  - 10 `<T>Return.NewResultData`: test gap (F2); production dispatch at fwserver/server_callfunction.go:34.
  - `list.ListResultsStreamDiagnostics`: unreferenced public API (F8).
  - 12 methods in list/*_test.go: compile-time interface scaffolding for `ExampleResource_listable` (no `// Output:` line, so it is compiled, not run). No action.
- **dupl 39** (function twins and their tests): measured in F11. The honest answer: no shareable core beyond ValidateImplementation. Tests mirror the per-type layout.
- **godox 2** (resource/upgrade_state.go:28, :59): live design TODOs for issue #340 (protocol-specific types in public API). Not done, so keep.
- **exhaustive 1** (function/func_error.go:120): SeverityInvalid is ignored on purpose, matching HasError semantics everywhere else. No action.
- **gocritic 1** (resource/upgrade_identity.go:36): `//schema version.`, folded into F10.
- **recvcheck 1** (function/result_data.go:19): intentional, see rejected rewrite 3.
- **funlen 3**: table-driven tests. No action.
- **gofix/any 5 + modernize/any**: coordinator owns, including func_error.go:124 `map[string]interface{}`.
- **gofix/newexpr 76 + modernize/newexpr 76**: coordinator owns (tests, `pointer_test.go`).

Modernization in area F beyond the coordinator-owned items: nothing, apart from F3's move off `math/rand` + `NewSource(time.Now().UnixNano())` seeding. No range-over-int, slices, maps or min/max candidates exist in the non-test code; all non-trivial non-test logic in area F is in function/ (about 260 lines outside the twins), list (2 helpers), statestore/lock_info.go and resource/import_state.go.

## Cross-area observations (not filed, outside area F; for the coordinator)

- `internal/fwserver/server_listresource.go:85` `var NoListResults`: an exported var in an internal package with 0 references (`grep -rn NoListResults --include=*.go .`). Deletable dead code.
- `internal/fwserver/server_listresource.go:124-128` inlines `list.ListResultsStreamDiagnostics(diagsResp.Diagnostics)`; equivalence is argued in F8.
- `internal/fwserver/server_listresource.go:91-101`: when `fwReq.Config == nil`, the fallback Config is built from the *resource* schema, not the list config schema, which looks wrong. If ResourceSchema is nil while ResourceIdentitySchema is non-nil, Config stays nil and `*fwReq.Config` at :144 would panic. Reachability not verified.
- `internal/fwserver/server_configureprovider.go:17-26`: a nil `req` is handled for the Configure call but then dereferenced in `if resp.Deferred != nil { if !req.ClientCapabilities.DeferralAllowed`. Latent: proto servers pass nil only for a nil proto request.
- `internal/fwserver/server.go:47-48`: comment says `[list.ConfigureRequest.ProviderData]`; the data actually goes to resource.ConfigureRequest.ProviderData.
- fwserver.ListResult and ListResultsStream duplicate list.ListResult and ListResultsStream (converted with `ListResult(result)`).
- The fromproto5/6 ArgumentsData "Parameter type missing at position %d" guard is unreachable for Terraform callers, because GetProviderSchema panics first (F5).
- MoveResourceState errors on a nil SourceRawState, while UpgradeResourceState and UpgradeResourceIdentity silently return on a nil RawState. The comments in both upgrade RPCs say this is deliberate, so not drift.
- `resource.ImportStatePassthroughWithIdentity` (resource/import_state.go:159) calls `req.Identity.GetAttribute` without a nil check. For a resource without identity and an empty ID it would panic. The doc says the helper is only for identity-enabled resources (documented precondition, latent).

## For the verifier

All probe modules point their `replace` at the unmodified repo, so `cd W/<probe> && GOFLAGS=-mod=mod go run .` reproduces each bug. To see a fix, change the `replace` target to `W/fwcopy`, which holds the F1, F3 and F5 drafts.

- F1: `W/probe-listreturn` (`main` runs runPlain, runGeneric, runFixed and runObject). Against the repo it prints 11 PANIC lines covering List/Set/Map, v5 and v6, the error path and the embedding custom types. Against `W/fwcopy` it prints 0. The `errE2E` cases are the minimal reproduction: Run returns `NewFuncError("boom")` and the return is CustomType-only.
- F3: `W/probe-lockrace`. Run `GOFLAGS=-mod=mod go run -race .` for the race report. Repeat without -race to see occasional duplicate IDs.
- F4: `W/probe-listnoident`.
- F5: `W/probe-nilparam` panics with a stack through toproto6.FunctionParameter (no recover, by design).
- F6: `W/probe-args`.
- F11: draft in `W/fwcopy2/function` (validate_implementation.go plus the 8 edited files). Use `diff -r /home/user/terraform-plugin-framework/function W/fwcopy2/function`.
- The F3 tier depends on how likely concurrent NewLockInfo calls are: provider tests with t.Parallel under -race are realistic, production much less so.
- Every probe module was run from inside its own scratch directory. GOFLAGS=-mod=mod only touches the probe's own go.mod/go.sum, never the repo's (checked with `git status --short`: clean).
