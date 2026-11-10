# Area F verification notes

Verifier for area F (packages resource, datasource, provider, ephemeral, action, list, statestore, function; schema
sub-packages excluded). Repository /home/user/terraform-plugin-framework at 00badb1, untouched (`git status --short`
is empty at the end). Go 1.26.0 (GOTOOLCHAIN=auto). The work was interrupted once (rate limit and machine restart);
the files in work/V-F survived and every probe below was re-run after the restart.

Scratch: `W = drafts/V-F`.

- `W/fwcopy`: my own copy of the module (tar without .git) with my drafts for F1, F3 and F5 (diff in `W/drafts.diff`).
- `W/fwcopy-f11`: a clean copy with the author's F11 draft files dropped in.
- Probe modules (each `replace`s the framework with the repo, or with `W/fwcopy` in the `-fixed` twins). Run with
  `GOTOOLCHAIN=auto GOFLAGS=-mod=mod go run .` from inside the probe directory; it only touches the probe's own go.mod/go.sum.
  `probe-f1`, `probe-f1unit`, `probe-f3`, `probe-f3stress`, `probe-f4`, `probe-f5`, `probe-f6`, `probe-f9`, `probe-f11lit`,
  `probe-cmp`, `probe-xconf`.
- Terraform core sources (read-only, untrusted data, only grepped): `W/ext-terraform-src/tf`, a blobless sparse
  shallow clone of github.com/hashicorp/terraform at main 3de4756 (2026-11-10) plus tag v1.14.0. The coordinator
  also pointed me at the B2 auditor's fetch in `audit/work/B2/ext`; I read it too (see F4).
- golangci-lint: `/usr/local/bin/golangci-lint` is 2.5.0 built with go1.25 and refuses this module ("Go language
  version (go1.25) ... lower than the targeted Go version (1.26.0)"); I used the audit's build
  `audit/bin/golangci-lint` 2.15.0 (go1.26.10) with the project `.golangci.yml`, one package at a time.

## Verdicts

| id | verdict | tier (claimed -> verified) | main corrections |
|---|---|---|---|
| F1 | CONFIRMED high | 1 -> 1 | impact: parameter-validator failures do not crash; crash scope depends on the custom type. size: author's -12/+36 exact; a +24/-0 draft also works |
| F2 | CONFIRMED high | 2 -> 2 | none |
| F3 | CONFIRMED high | 2 -> 2 (borderline) | duplicate rate only under sustained concurrency; crypto/rand fatals on RNG failure |
| F4 | CONFIRMED high | 2 -> 3 | Terraform core blocks the request; the registration-time fix would break providers; prefer a per-RPC guard |
| F5 | CONFIRMED high | 3 -> 3 | GetFunctions RPC also panics; "can never run" only for Terraform callers |
| F6 | CONFIRMED high | 3 -> 3 | replacement must put the negative check before the empty-data branch |
| F7 | CONFIRMED high | 2 -> 2 | wording of (1) and (7) |
| F8 | CONFIRMED high | 3 -> 3 | none |
| F9 | CONFIRMED high | 3 -> 3 | replacement wrong for Parameter.GetName; size -44/+60 in 17 files |
| F10 | CONFIRMED high | 3 -> 3 | 39 sites in 25 files, size about -55/+80 |
| F11 | CONFIRMED high | 3 -> 3 | 19-line bodies; 72 one-line getters + 12 GetType |

New items (cross-area, route to the internal/fwserver auditor): FV1 dead `fwserver.NoListResults`; FV2 latent nil
dereference in `fwserver.ConfigureProvider`.

## F1: CustomType-only List/Set/MapReturn

Docs: `function/list_return.go:22` "Either the ElementType or CustomType field must be set" (same in set/map returns
and in the three parameter types) versus the ElementType field doc `:32-33` "This field must be set". The validation
side settles it: `ListReturn.ValidateImplementation` skips all checks when CustomType is set, and
`TestListReturnValidateImplementation` case "customtype" (`ListReturn{CustomType: testtypes.ListType{}}`) expects no
diagnostics. Schema attributes use the same rule (`fwschema.AttributeMissingElementTypeDiag`: "One of these fields is
required"). So CustomType-only is documented usage.

recover(): `grep -rln "recover()" terraform-plugin-go@v0.31.0/tfprotov5 tfprotov6` -> 0 files; go-plugin v1.7.0 has
one, at client.go:742 (client side).

Probe `W/probe-f1` (custom types: A = docs pattern, value Type() derived from the embedded value; B = bare embedding
like internal/testing/testtypes; plain basetypes.ListType; Set and Map in style A). Output against the repo
(stack frames shortened to the crash site and the toproto frame):

```
v5 List CustomType(A)-only, Run error                         => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto5/function.go:104
v6 List CustomType(A)-only, Run error                         => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto6/function.go:104
v5 List CustomType(B testtypes-style)-only, Run error         => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto5/function.go:104
v6 List CustomType(B testtypes-style)-only, Run error         => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto6/function.go:104
v5 List CustomType(basetypes.ListType)-only, Run error        => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto5/function.go:104
v6 List CustomType(basetypes.ListType)-only, Run error        => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto6/function.go:104
v5 Set CustomType(A)-only, Run error                          => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/set_value.go:225 <- internal/toproto5/function.go:104
v6 Set CustomType(A)-only, Run error                          => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/set_value.go:225 <- internal/toproto6/function.go:104
v5 Map CustomType(A)-only, Run error                          => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/map_value.go:235 <- internal/toproto5/function.go:104
v6 Map CustomType(A)-only, Run error                          => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/map_value.go:235 <- internal/toproto6/function.go:104
v5 List CustomType(A)-only, PARAM VALIDATOR fails, Run error  => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="param validation failed" result=nil
v6 List CustomType(A)-only, PARAM VALIDATOR fails, Run error  => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="param validation failed" result=nil
v5 CONTROL List ElementType+CustomType(A), Run error          => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v6 CONTROL List ElementType+CustomType(A), Run error          => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v5 CONTROL List ElementType-only, Run error                   => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v6 CONTROL List ElementType-only, Run error                   => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v5 List CustomType(A)-only, Run Set(AListValue)               => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto5/function.go:104
v6 List CustomType(A)-only, Run Set(AListValue)               => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto6/function.go:104
v5 List CustomType(A)-only, Run Set([]string)                 => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto5/function.go:104
v6 List CustomType(A)-only, Run Set([]string)                 => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto6/function.go:104
v5 List CustomType(B)-only, Run Set([]string)                 => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto5/function.go:104
v6 List CustomType(B)-only, Run Set([]string)                 => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto6/function.go:104
v5 List CustomType(B)-only, Run Set(types.List)               => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto5/function.go:104
v6 List CustomType(B)-only, Run Set(types.List)               => PANIC: runtime error: invalid memory address or nil pointer dereference | types/basetypes/list_value.go:225 <- internal/toproto6/function.go:104
v5 CONTROL List ElementType+CustomType(A), Run Set([]string)  => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v6 CONTROL List ElementType+CustomType(A), Run Set([]string)  => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v5 CONTROL List ElementType-only, Run Set([]string)           => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v6 CONTROL List ElementType-only, Run Set([]string)           => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
```

Against `W/fwcopy` (my draft): 0 PANIC lines.

```
v5 List CustomType(A)-only, Run error                         => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v6 List CustomType(A)-only, Run error                         => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v5 List CustomType(B testtypes-style)-only, Run error         => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v6 List CustomType(B testtypes-style)-only, Run error         => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v5 List CustomType(basetypes.ListType)-only, Run error        => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v6 List CustomType(basetypes.ListType)-only, Run error        => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v5 Set CustomType(A)-only, Run error                          => GPS diags=0 returnType=tftypes.Set[tftypes.String] | CallFunction error="boom" result=tftypes.Set[tftypes.String]<unknown>
v6 Set CustomType(A)-only, Run error                          => GPS diags=0 returnType=tftypes.Set[tftypes.String] | CallFunction error="boom" result=tftypes.Set[tftypes.String]<unknown>
v5 Map CustomType(A)-only, Run error                          => GPS diags=0 returnType=tftypes.Map[tftypes.String] | CallFunction error="boom" result=tftypes.Map[tftypes.String]<unknown>
v6 Map CustomType(A)-only, Run error                          => GPS diags=0 returnType=tftypes.Map[tftypes.String] | CallFunction error="boom" result=tftypes.Map[tftypes.String]<unknown>
v5 List CustomType(A)-only, PARAM VALIDATOR fails, Run error  => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="param validation failed" result=nil
v6 List CustomType(A)-only, PARAM VALIDATOR fails, Run error  => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="param validation failed" result=nil
v5 CONTROL List ElementType+CustomType(A), Run error          => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v6 CONTROL List ElementType+CustomType(A), Run error          => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v5 CONTROL List ElementType-only, Run error                   => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v6 CONTROL List ElementType-only, Run error                   => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="boom" result=tftypes.List[tftypes.String]<unknown>
v5 List CustomType(A)-only, Run Set(AListValue)               => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v6 List CustomType(A)-only, Run Set(AListValue)               => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v5 List CustomType(A)-only, Run Set([]string)                 => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v6 List CustomType(A)-only, Run Set([]string)                 => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v5 List CustomType(B)-only, Run Set([]string)                 => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v6 List CustomType(B)-only, Run Set([]string)                 => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v5 List CustomType(B)-only, Run Set(types.List)               => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v6 List CustomType(B)-only, Run Set(types.List)               => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v5 CONTROL List ElementType+CustomType(A), Run Set([]string)  => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v6 CONTROL List ElementType+CustomType(A), Run Set([]string)  => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v5 CONTROL List ElementType-only, Run Set([]string)           => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
v6 CONTROL List ElementType-only, Run Set([]string)           => GPS diags=0 returnType=tftypes.List[tftypes.String] | CallFunction error="nil" result=tftypes.List[tftypes.String]<tftypes.String<"a">>
```

`diff <(grep CONTROL repo) <(grep CONTROL fixed)` and the same for the VALIDATOR lines: identical.

Unit probe `W/probe-f1unit` (styles A, C = value Type() is a fixed type, D = ValueFromList rebuilds unknowns with its
own element type). Repo:

```
A []string     NewResultData ferr-nil=true placeholderType=types.ListType[!!! MISSING TYPE !!!]
A []string       Run-error path (placeholder ToTerraformValue): PANIC runtime error: invalid memory address or nil pointer dereference
A []string       Run-success path: Set error "Value Conversion Error: An unexpected error was encountered trying to convert the Terraform value. This is always an error in the provider. Please report the following to the provider developer:\n\nmissing type inf
A AListValue   NewResultData ferr-nil=true placeholderType=types.ListType[!!! MISSING TYPE !!!]
A AListValue     Run-error path (placeholder ToTerraformValue): PANIC runtime error: invalid memory address or nil pointer dereference
A AListValue     Run-success path: Set error "Value Conversion Error: An unexpected error was encountered while verifying an attribute value matched its expected type to prevent unexpected behavior or panics. This is always an error in the provider. Please rep
C []string     NewResultData ferr-nil=true placeholderType=types.ListType[basetypes.StringType]
C []string       Run-error path (placeholder ToTerraformValue): PANIC runtime error: invalid memory address or nil pointer dereference
C []string       Run-success path: Set ok -> tftypes.List[tftypes.String]<tftypes.String<"a">>
D []string     NewResultData ferr-nil=true placeholderType=types.ListType[basetypes.StringType]
D []string       Run-error path (placeholder ToTerraformValue): tftypes.List[tftypes.String]<unknown>
D []string       Run-success path: Set ok -> tftypes.List[tftypes.String]<tftypes.String<"a">>
```

Fixed copy:

```
A []string     NewResultData ferr-nil=true placeholderType=types.ListType[basetypes.StringType]
A []string       Run-error path (placeholder ToTerraformValue): tftypes.List[tftypes.String]<unknown>
A []string       Run-success path: Set ok -> tftypes.List[tftypes.String]<tftypes.String<"a">>
A AListValue   NewResultData ferr-nil=true placeholderType=types.ListType[basetypes.StringType]
A AListValue     Run-error path (placeholder ToTerraformValue): tftypes.List[tftypes.String]<unknown>
A AListValue     Run-success path: Set ok -> tftypes.List[tftypes.String]<tftypes.String<"a">>
C []string     NewResultData ferr-nil=true placeholderType=types.ListType[basetypes.StringType]
C []string       Run-error path (placeholder ToTerraformValue): tftypes.List[tftypes.String]<unknown>
C []string       Run-success path: Set ok -> tftypes.List[tftypes.String]<tftypes.String<"a">>
D []string     NewResultData ferr-nil=true placeholderType=types.ListType[basetypes.StringType]
D []string       Run-error path (placeholder ToTerraformValue): tftypes.List[tftypes.String]<unknown>
D []string       Run-success path: Set ok -> tftypes.List[tftypes.String]<tftypes.String<"a">>
```

Style D lines are identical in both runs.

Size: the author's `drafts/F/fix-*_return.diff` each have 12 `>` and 4 `<` lines (-12/+36 total, as claimed). My
tighter draft (reassign `value` only when ElementType is nil) is +8/-0 per file. `go vet ./function/`, `go test
./function/`, `go test ./internal/{fwserver,proto5server,proto6server}/ -run 'CallFunction|GetProviderSchema|GetFunctions|Function'`
and golangci-lint 2.15.0 on ./function/ (0 issues) all pass on `W/fwcopy`.

Tier: kept at 1. Crash, documented configuration, proof at ladder step 5, fix only touches the crashing configuration.
Exposure caveat recorded in the JSON.

## F2: NewResultData test gap

```
go test ./function/ -count=1 -coverprofile=W/cov-function.out        -> NewResultData 0.0% x12 (result_data.go NewResultData 100%)
go test ./internal/fwserver/ -run CallFunction -coverpkg=./function/  -> string_return.go 60.0%, other 11 0.0%
go test ./internal/proto5server/ -coverpkg=./function/ (whole package) -> string_return.go 60.0% only
go test ./internal/proto6server/ -coverpkg=./function/ (whole package) -> string_return.go 60.0% only
go test ./providerserver/ -coverpkg=./function/                       -> all 0.0%
grep -rn --include=*_test.go -o "function\.[A-Z][A-Za-z0-9]*Return{" internal/ | wc -l   -> 103, all StringReturn
grep -rn "\.NewResultData(" --include=*.go . | grep -v _test.go       -> internal/fwserver/server_callfunction.go:34 only
grep -rn "\.NewResultData(" --include=*_test.go . | grep -v "function\.NewResultData("  -> 0 (all 19 test hits are the package func)
```

## F3: statestore.NewLockInfo race

Probe `W/probe-f3` (64 goroutines released together):

```
-race run 1: Found 9 data race(s)    ids=64 unique=64 duplicates=0 len=36
-race run 2: Found 10 data race(s)   ids=64 unique=64 duplicates=0
-race run 3: Found 4 data race(s)    ids=64 unique=64 duplicates=0
race frame: math/rand.(*Rand).Read rand.go:279 <- statestore.generateLockID lock_info.go:93 <- NewLockInfo lock_info.go:47
non-race, 40 runs: duplicates=0 in all 40 (author: 1 duplicate in 3 of 5 runs; not reproduced here)
```

Stress `W/probe-f3stress` (no -race):

```
16 goroutines x 4000:   duplicates=16, 18, 17, 24, 14 (5 runs, 64,000 IDs each)
8 goroutines x 150000:  duplicates=717, 274, 305 (3 runs, 1.2M IDs each), no panic
```

Fix draft (my own, `W/drafts.diff`): import crypto/rand, drop rngSource, `_, _ = rand.Read(buf)` with a one-line
comment; -4/+3 as claimed. On `W/fwcopy`: race probe 0 races in 3 runs, stress 0 duplicates in 5 x 64,000 and in a
-race stress run, IDs still 36 chars 8-4-4-4-12; `go test -race ./statestore/` ok; golangci-lint 0 issues.
`go doc crypto/rand.Read` (go1.26): "never returns an error, and always fills b entirely ... crashes the program
irrecoverably if an error is returned". Format contract: lock.go:30 calls LockID "an opaque string"; fwserver
(server_lockstate.go:65, server_unlockstate.go:50), toproto6/lockstate.go:21 and fromproto6/unlockstate.go:24 only
copy it.

## F4: list resource for a resource without identity

Probe `W/probe-f4` (Resource p_thing with or without ResourceWithIdentity; ListResource p_thing whose List calls
req.NewListResult then result.Identity.SetAttribute):

```
v6 resource WITHOUT identity: GPS diags=0 listSchemas=1 | GetMetadata diags=0 listResources=1 | identitySchemas=0 | ValidateListResourceConfig diags=0 | ListResource iteration PANIC: runtime error: invalid memory address or nil pointer dereference at list/list_resource.go:137 +0x4f
v5 resource WITHOUT identity: GPS diags=0 listSchemas=1 | ListResource iteration PANIC: runtime error: invalid memory address or nil pointer dereference at list/list_resource.go:137 +0x4f
v6 CONTROL resource WITH identity: GPS diags=0 listSchemas=1 | GetMetadata diags=0 listResources=1 | identitySchemas=1 | ValidateListResourceConfig diags=0 | result display="one" diags=0 | results=1
v5 CONTROL resource WITH identity: GPS diags=0 listSchemas=1 | result display="one" diags=0 | results=1
```

Guards searched: `grep -n Identity internal/fwserver/server_listresources.go server_listresource.go
server_getproviderschema.go server_validatelistresourceconfig.go server_getmetadata.go`: only the raw-schema check at
server_listresources.go:100-108 and the result checks at server_listresource.go:189-198. Nothing rejects a framework
resource without identity.

Terraform core (the coordinator's B2 claim, checked in two source sets):

- My clone, main 3de4756 and tag v1.14.0, `internal/plugin6/grpc_provider.go` and `internal/plugin/grpc_provider.go`,
  `GRPCProvider.ListResource`, lines 17-21 of the function (before `p.client.ListResource` at line 48):
  `resourceSchema, ok := schema.ResourceTypes[r.TypeName]; if !ok || resourceSchema.Identity == nil { ... "Identity
  schema not found for resource type %s; this is a bug in the provider - please report it there" ... return resp }`.
- B2's `drafts/B2/ext/grpc_provider6.go` has md5 b15254798822ec60478ba372fec1ee80, identical to my main copy. In its
  GetProviderSchema, `id := identResp.IdentitySchemas[name]` / `resp.ResourceTypes[name] = convert.ProtoToProviderSchema(res, id)`
  (function lines 82-83): Identity comes only from GetResourceIdentitySchemas, which the framework leaves empty for
  p_thing (probe: identitySchemas=0). B2's `genconfig.go` is a byte-identical duplicate of its
  `node_resource_plan_instance.go`, and `provider_iface.go` duplicates `provider.go`; neither changes the conclusion.
- `internal/terraform/node_resource_plan_instance_query.go:128` calls `provider.ListResource(...)` through
  providers.Interface, i.e. through the guarded client.

Conclusion: B2's claim is right. `terraform query` gets a core error naming a provider bug; the framework panic is
reachable only from direct ProviderServer callers. Impact, tier (2 -> 3), risk and replacement corrected in the JSON.
The fix-location attack: `server_getproviderschema.go:84-87` returns on any ListResourceSchemas error, and a
GetProviderSchema error is fatal for every Terraform command, so an error in ListResourceFuncs would turn a query-only
failure into a total failure for providers already in that state.

## F5: nil parameter type

Probe `W/probe-f5` (custom Parameter with GetType() == nil), repo:

```
v6 GetProviderSchema Parameters[0]      => PANIC runtime error: invalid memory address or nil pointer dereference at internal/toproto6/function.go:56 +0x90 <- internal/toproto6/function.go:33 +0x1cf <- internal/toproto6/getproviderschema.go:87 +0xcd3
v6 GetFunctions      Parameters[0]      => PANIC runtime error: invalid memory address or nil pointer dereference at internal/toproto6/function.go:56 +0x90 <- internal/toproto6/function.go:33 +0x1cf <- internal/toproto6/getfunctions.go:26 +0x266
v5 GetProviderSchema Parameters[0]      => PANIC runtime error: invalid memory address or nil pointer dereference at internal/toproto5/function.go:56 +0x90 <- internal/toproto5/function.go:33 +0x1cf <- internal/toproto5/getproviderschema.go:90 +0xc93
v6 GetProviderSchema VariadicParameter  => PANIC runtime error: invalid memory address or nil pointer dereference at internal/toproto6/function.go:56 +0x90 <- internal/toproto6/function.go:38 +0x286 <- internal/toproto6/getproviderschema.go:87 +0xcd3
v6 GetFunctions      VariadicParameter  => PANIC runtime error: invalid memory address or nil pointer dereference at internal/toproto6/function.go:56 +0x90 <- internal/toproto6/function.go:38 +0x286 <- internal/toproto6/getfunctions.go:26 +0x266
v5 GetProviderSchema VariadicParameter  => PANIC runtime error: invalid memory address or nil pointer dereference at internal/toproto5/function.go:56 +0x90 <- internal/toproto5/function.go:38 +0x286 <- internal/toproto5/getproviderschema.go:90 +0xc93
```

With my draft (+18/-0, `W/drafts.diff`):

```
v6 GetProviderSchema Parameters[0]      => diags=1 functions=0 Invalid Function Definition: Function "f" - Parameter at position 0 data type is undefined
v6 GetFunctions      Parameters[0]      => diags=1 functions=0
v5 GetProviderSchema Parameters[0]      => diags=1 functions=0
v6 GetProviderSchema VariadicParameter  => diags=1 functions=0 Invalid Function Definition: Function "f" - Definition variadic parameter data type is undefined
v6 GetFunctions      VariadicParameter  => diags=1 functions=0
v5 GetProviderSchema VariadicParameter  => diags=1 functions=0
```

fwserver/server_functions.go:96-113 drops a function with validation errors before conversion, so the diagnostic
prevents the panic for both GetProviderSchema and GetFunctions. fromproto6/arguments_data.go:66-75 checks the same
condition at CallFunction time; it runs for direct callers that skip GetProviderSchema.

## F6: GetArgument(-1) and zero ResultData

```
GetArgument(0) on 1 arg                      ok
GetArgument(1) on 1 arg                      FuncError arg=1 text="Invalid Argument Data Position: When attempting to fetch arg..."
GetArgument(-1) on 1 arg                     PANIC runtime error: index out of range [-1]
GetArgument(-1) on 0 args                    FuncError arg=-1 text="Invalid Argument Data Usage: When attempting to fetch argume..."
new(ResultData).Set                          PANIC runtime error: invalid memory address or nil pointer dereference
RunResponse{}.Result.Set                     PANIC runtime error: invalid memory address or nil pointer dereference
NewResultData(StringUnknown).Set             ok
```

The fourth line is the hole in the proposed replacement: the empty-data branch already sends a negative
FunctionArgument.

## F7, F8, F9, F10: docs and unreferenced API

Every cited line was opened (commands: `awk -v a=.. -v b=.. 'NR>=a && NR<=b'` per site).

- F7: `diff <(sed ...ephemeral->list... ephemeral/doc.go) list/doc.go` empty; same for configure.go and metadata.go.
  `grep -n 'Renew\|Close' list/*.go | grep -v _test` -> list/doc.go:10-11 only. ListResourceData is consumed at
  fwserver/server_listresource.go:109 and server_validatelistresourceconfig.go:39.
- F8: `grep -rn "list\.ConfigureRequest\|list\.ConfigureResponse\|list\.MetadataRequest\|list\.MetadataResponse" . --exclude-dir=.git`
  -> 1 (comment, internal/fwserver/server.go:48); unqualified uses inside list/ (tests included) -> 0;
  `grep -rn ListResultsStreamDiagnostics . --exclude-dir=.git` -> 2 (definition and doc); `go test ./list/
  -coverprofile` -> ListResultsStreamDiagnostics 0.0%; `git log -S ListResultsStreamDiagnostics -- list/` -> 7d8104c
  (#1167); `grep -i experimental list/*.go` -> nothing.
- F9: `grep -n 'default to "param"\|default name will' function/*.go | grep -v _test | wc -l` -> 27 in 13 files; each
  stale paragraph is 3 lines (39 lines). Non-test Go hits for varparam/param%d/"param": 0. CHANGELOG.md:338-339.
  `grep -ln "does not have a name" function/*_test.go | wc -l` -> 13. 8 scalar ValidateImplementation methods without
  a doc comment (awk scan). Probe `W/probe-f9`: custom Parameter with GetName() == "" ->
  `CUSTOM Parameter with empty name: GPS diags=0, protocol param name=""`.
- F10: per-site checks listed in the JSON. Configure call sites: `grep -c 'WithConfigure.Configure(ctx'` is 1 in each
  of the 9 resource RPC files and in server_readdatasource.go and server_validatedatasourceconfig.go. `s.deferred`
  users: configureprovider, importresourcestate, openephemeralresource, planaction, planresourcechange,
  readdatasource, readresource. Sibling doc.go sizes: ephemeral 23, action 15, resource 24, datasource 19, provider
  25, list 23, function 21 lines; statestore has none.

## F11: shared ValidateImplementation helpers

```
for t in list map set object; do sed -n '/^func (p [A-Za-z]*Parameter) ValidateImplementation/,/^}/p' function/${t}_parameter.go | sed "s/[A-Z][a-z]*Parameter)/XParameter)/" | md5sum; done
  -> 546d96d86723 x4 (19 lines each); Return twin -> 19297eb26870 x4 (11 lines each)
diff -r function drafts/F/fwcopy2/function -> 112 '<' lines, 8 '>' lines, plus validate_implementation.go (48 lines): net -56
W/fwcopy-f11: gofmt -l function/ -> empty; go vet ok; go test ./function/ ok; 8 named tests, 48 subtests PASS;
  golangci-lint 2.15.0 -> 0 issues; coverage validateCollectionParameterImplementation 100%, validateCollectionReturnImplementation 100%
parameter files: 1606 lines, 706 comment lines, 84 getters (72 one-line bodies, 12 GetType of 5-7 lines)
probe-f11lit (go1.26.0): ./main.go:19:21: unknown field Name in struct literal of type BoolParameter
```

## Author's rejected rewrites I checked

- 1 (shared base struct): endorsed, compile probe above.
- 3 (pointer receiver for ResultData.Equal): endorsed. `W/probe-cmp`: value receiver -> diff ""; pointer receiver ->
  `PANIC cannot handle unexported field at {main.respPtr}.Result.value`.
- 5 (NoListResults as iter.Seq): endorsed by reasoning (exported var type changes).
- 6 (nil guard in NewListResult): endorsed; my F4 probe's List calls result.Identity.SetAttribute right after.
- 7a (always prefer the CustomType's element type): endorsed; the nil-only guard keeps the CONTROL lines identical.
- Added: the registration-time F4 fix (see F4).

## Cross-area notes

- `internal/fwserver/server_listresource.go:85` `var NoListResults`: 0 references. `grep -rn NoListResults
  --include=*.go .` shows only list.NoListResults and tfprotov5/6.NoListResults uses besides the declaration; no
  unqualified use in package fwserver files (tests included); `grep -rn "fwserver\.NoListResults"` -> 0. Reported as FV1.
- `internal/fwserver/server_configureprovider.go:26`: probe `W/probe-xconf`:

```
v6 ConfigureProvider(nil), provider sets Deferred    PANIC runtime error: invalid memory address or nil pointer dereference
v5 ConfigureProvider(nil), provider sets Deferred    PANIC runtime error: invalid memory address or nil pointer dereference
v6 ConfigureProvider(nil), no Deferred               err=<nil> diags=0
v6 CONTROL non-nil req (no deferral cap), Deferred   err=<nil> diags=1 "Invalid Deferred Provider Response"
```

  (a later run printed the frame `internal/fwserver/server_configureprovider.go:26`). The gRPC handler allocates the
  request (`in := new(ConfigureProvider_Request)`, tfplugin6_grpc.pb.go:902) and tf6server dereferences
  `req.ClientCapabilities` itself, so Terraform never reaches it. Reported as FV2 (tier 3, latent).
- Not checked (not requested): the author's note about the fallback Config at server_listresource.go:91-101.

## My draft diffs (W/drafts.diff)

```diff
--- function/list_return.go	2026-10-24 21:30:10.434927489 +0000
+++ function/list_return.go	2026-11-10 20:59:46.478059852 +0000
@@ -63,6 +63,14 @@
 		return NewResultData(value), nil
 	}
 
+	// When only CustomType is set, it is the only source of the element
+	// type that the unknown value needs for conversion to the protocol.
+	if r.ElementType == nil {
+		if t, ok := r.CustomType.(attr.TypeWithElementType); ok {
+			value = basetypes.NewListUnknown(t.ElementType())
+		}
+	}
+
 	valuable, diags := r.CustomType.ValueFromList(ctx, value)
 
 	return NewResultData(valuable), FuncErrorFromDiags(ctx, diags)
--- function/set_return.go	2026-10-24 21:30:10.439597581 +0000
+++ function/set_return.go	2026-11-10 20:59:46.478211750 +0000
@@ -63,6 +63,14 @@
 		return NewResultData(value), nil
 	}
 
+	// When only CustomType is set, it is the only source of the element
+	// type that the unknown value needs for conversion to the protocol.
+	if r.ElementType == nil {
+		if t, ok := r.CustomType.(attr.TypeWithElementType); ok {
+			value = basetypes.NewSetUnknown(t.ElementType())
+		}
+	}
+
 	valuable, diags := r.CustomType.ValueFromSet(ctx, value)
 
 	return NewResultData(valuable), FuncErrorFromDiags(ctx, diags)
--- function/map_return.go	2026-10-24 21:30:10.434927489 +0000
+++ function/map_return.go	2026-11-10 20:59:46.478274405 +0000
@@ -63,6 +63,14 @@
 		return NewResultData(value), nil
 	}
 
+	// When only CustomType is set, it is the only source of the element
+	// type that the unknown value needs for conversion to the protocol.
+	if r.ElementType == nil {
+		if t, ok := r.CustomType.(attr.TypeWithElementType); ok {
+			value = basetypes.NewMapUnknown(t.ElementType())
+		}
+	}
+
 	valuable, diags := r.CustomType.ValueFromMap(ctx, value)
 
 	return NewResultData(valuable), FuncErrorFromDiags(ctx, diags)
--- function/definition.go	2026-10-24 21:30:10.427593807 +0000
+++ function/definition.go	2026-11-10 21:08:45.514427387 +0000
@@ -82,6 +82,15 @@
 	for pos, param := range d.Parameters {
 		parameterPosition := int64(pos)
 
+		if param.GetType() == nil {
+			diags.AddError(
+				"Invalid Function Definition",
+				"When validating the function definition, an implementation issue was found. "+
+					"This is always an issue with the provider and should be reported to the provider developers.\n\n"+
+					fmt.Sprintf("Function %q - Parameter at position %d data type is undefined", req.FuncName, pos),
+			)
+		}
+
 		if paramWithValidateImplementation, ok := param.(fwfunction.ParameterWithValidateImplementation); ok {
 			req := fwfunction.ValidateParameterImplementationRequest{
 				FunctionName:      req.FuncName,
@@ -112,6 +121,15 @@
 	}
 
 	if d.VariadicParameter != nil {
+		if d.VariadicParameter.GetType() == nil {
+			diags.AddError(
+				"Invalid Function Definition",
+				"When validating the function definition, an implementation issue was found. "+
+					"This is always an issue with the provider and should be reported to the provider developers.\n\n"+
+					fmt.Sprintf("Function %q - Definition variadic parameter data type is undefined", req.FuncName),
+			)
+		}
+
 		if paramWithValidateImplementation, ok := d.VariadicParameter.(fwfunction.ParameterWithValidateImplementation); ok {
 			req := fwfunction.ValidateParameterImplementationRequest{
 				FunctionName: req.FuncName,
--- statestore/lock_info.go	2026-10-24 21:30:10.799678047 +0000
+++ statestore/lock_info.go	2026-11-10 21:03:45.342889622 +0000
@@ -4,8 +4,8 @@
 package statestore
 
 import (
+	"crypto/rand"
 	"fmt"
-	"math/rand"
 	"os"
 	"os/user"
 	"time"
@@ -85,12 +85,11 @@
 	)
 }
 
-var rngSource = rand.New(rand.NewSource(time.Now().UnixNano()))
-
 // generateLockID returns a random UUID string
 func generateLockID() string {
 	buf := make([]byte, 16)
-	rngSource.Read(buf)
+	// crypto/rand.Read never returns an error and always fills buf.
+	_, _ = rand.Read(buf)
 
 	// The only error returned from this function is if the byte length isn't 16, so we can ignore it
 	id, _ := uuid.FormatUUID(buf)
```
