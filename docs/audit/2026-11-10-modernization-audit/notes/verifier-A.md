# Verifier notes, area A (protocol layer)

Repository at 00badb1, read-only throughout (git status clean at the end). Work dir:
`drafts/V-A`, called `$V` below.

## Method
- Two sandbox copies made with tar (no .git): `$V/repo0` stays pristine for differential runs; `$V/repo` receives one draft at a time.
  Both are restored and checked with `diff -rq` against the repository after every item. They are pristine now.
- All probes are my own, kept in `$V/scripts/` and copied in only for a run:
  - zz_va_a3probe_test.go: A2/A3 differential, 7 edge cases
  - leak_test.go: A5, real grpc + tf6server stack
  - zz_va_a5_test.go: A5 semantics and -race stress
  - zz_va_a6_test.go: A6
  - zz_va_a7_test.go, zz_va_a8_test.go, zz_va_a9_test.go: A7, A8, A9
  - zz_va_a10_test.go, zz_va_a10diff_test.go, zz_va_a14v5diff_test.go: A10/A14 nondeterminism and success-path digests
  - zz_va_a11_test.go, zz_va_a12_test.go: A11, A12
  - zz_va_b2_test.go, zz_va_b2null_test.go: B2 hand-off
  - sepscan.py: A11 broader scan
  - norm_cases.py: A3 case normaliser
- Lint uses the audit's go1.26 build of golangci-lint (`audit/bin/golangci-lint`, project .golangci.yml, -j 1, only the touched packages).
  The system golangci-lint is built with go1.25 and refuses the go 1.26.0 module.
- Saved outputs:
  - `$V/a1_fixed_test.txt`: A1 sandbox fix, 30 failures.
  - `$V/a2_before.txt`, `$V/a2_after.txt`, `$V/a3_after.txt`: test-identity lists.
  - `$V/a3probe_repo0.txt`, `$V/a3probe_repo.txt`, `$V/a3probe_a2only.txt`: differential outputs.
  - `$V/heap_repo0.pprof`: A5 heap profile.

## Verdicts in one line each
- **A1 CONFIRMED.** Impact corrected: no user-visible output change today. Tier 2 kept. The core claim is unverifiable context.
- **A2 CONFIRMED.** Size is +627/-1178, not -1084/+573. The protocol-line wording is corrected.
- **A3 CONFIRMED.** Size is +135/-384, not -372/+128.
- **A4 CONFIRMED.**
- **A5 CONFIRMED.**
  - The leak is 11.5 KB/RPC on the real stack, not 0.5 KB.
  - "No regression" fails for never-canceled parents (106 B → 624 B per RPC).
  - Call sites are 27/35, size is +19/-2 per package, and the tier goes from 3 to 2.
- **A6 CONFIRMED.** The precondition is refined, and the size is +14 per package.
- **A7 CONFIRMED.**
- **A8 CONFIRMED.** Reachable only by provider misuse (embedding).
- **A9 CONFIRMED.** The proposed guard misses the nil-Block case (reproduced), so the replacement is corrected.
- **A10 CONFIRMED.** The fromproto part of the scope is unreachable. Size is +53/-92 on toproto6.
- **A11 CONFIRMED.**
- **A12 CONFIRMED.**
- **A13 CONFIRMED.**
- **A14 CONFIRMED.** Size is -117/+36, not -95/+36.
- **A15 PLAUSIBLE (medium).**
- **A16 CONFIRMED.**
- **A17 CONFIRMED.**
- **A18 CONFIRMED.**
- **B2 hand-off confirmed.** It is already filed as B2-6 part (4) and is reported as A-V1 for merging.

## Reproduction commands (selection)
```sh
R=/home/user/terraform-plugin-framework
# A1 count
grep -rc "PlanDestroy: *true" --include=*_test.go $R/internal/proto5server $R/internal/proto6server    # 6+8+7+9 = 30
# A2/A3/A10/A14/A5/A6 sizes
git diff --no-index --numstat <orig> <draft>
# A5 call sites
for p in 5 6; do grep -l . $R/internal/proto${p}server/*.go | grep -v _test.go | xargs grep -h 'registerContext(' | grep -vc 'func (s \*Server) registerContext'; done   # 27, 35
# A11 broader scan
python3 -I $V/scripts/sepscan.py $R      # 2 hits
# A14 sites
grep -rn "sort.Slice(" --include=*.go $R | grep -v _test.go | wc -l   # 12 (11 in toproto5/6)
# A5 realistic leak (copy leak_test.go into <sandbox>/internal/vaprobe/)
VA_N=20000 GOTOOLCHAIN=auto go test -count=1 -v -run TestVALeakRealStack ./internal/vaprobe/
```

## Working log (chronological, as recorded during the run)
# V-A running notes (verifier of area A). Sandbox: drafts/V-A/repo (tar copy of repo at 00badb1, no .git)

## A1
- toproto5/6 server_capabilities.go:20-24 copy only 3 fields; fwserver/server_capabilities.go:41-48 sets GenerateResourceConfig: true.
- tfprotov5/6.ServerCapabilities.GenerateResourceConfig exists in terraform-plugin-go v0.31.0 (tfprotov5/server_capabilities.go:29, tfprotov6 same),
  and tfprotov5/6/internal/toproto/server_capabilities.go:20 forwards it to the wire (tfplugin5.pb.go field 4).
- Probe (V-A/repo/internal/toproto6/zz_va1_probe_test.go): fw={..GenerateResourceConfig:true} -> p5/p6 GenerateResourceConfig:false.
- git: 2ee75aa (#1281) "Enable `GenerateResourceConfig` server capability" touched fwserver/server_capabilities.go + fwserver tests, no toproto file. History is shallow (124 commits, oldest e43fb0b).
  No sign of deliberate omission (no comment, no version gate in toproto; fwserver comment says "This should always be enabled").
- 30 expectations: grep -rc "PlanDestroy: *true" proto5server/proto6server tests: 6+8+7+9=30; all 30 are within 4 lines of "ServerCapabilities".
- Fix applied in sandbox (both converters +1 line): exactly 30 subtests fail, 13 TestServerGetMetadata (6 v5 + 7 v6) + 17 TestServerGetProviderSchema (8+9);
  every failure diff is the single line '- GenerateResourceConfig: false / + true' (30/30). toproto5/6 tests pass. Reverted.
- Core snippet: author's fetched main-branch node_resource_plan_instance.go:1191 reads `if providerSchema.ServerCapabilities.GenerateResourceConfig {` -> UNVERIFIABLE CONTEXT (downloaded, main branch, not a release tag).
- fwserver/server_generateresourceconfig.go:39-43 MAINTAINER NOTE: logic "transcribed from Terraform Core's default ... no functionality changes ... in v1.19.0".
  => user-visible generated config is identical today either way; the bug's effect is that 434 lines (fwserver 192 + protoNserver 82 + fromproto 98 + toproto 62) never run in production.
- Callers: toproto5/6 getmetadata.go:28 and getproviderschema.go:32/28 are the only users of ServerCapabilities().

## A2
- fromproto5/6 arguments_data.go: 608 lines each; identical after s/tfprotov[56]/N/, s/fromproto[56]/N/. Raw diff = 4 lines: package clause (4), import (10), doc (20), signature (22).
  So "4 lines mention tfprotovN" is really 3 + the package clause; plus range clause (55) and argument.Unmarshal (81) = 6 protocol-touching lines. len(arguments): 6 sites; FrameworkTrace: 6 calls.
- Author draft core vs today's fromproto6: 10 hunks (author said 11), all mechanical (numArguments, range-over-int, unmarshal(position, t), doc, package, import).
- Sandbox (V-A/repo): A2 drafts installed; go build ./internal/..., go vet, tests pass. Test-identity diff (fromproto5+6 all tests + proto5/6server -run CallFunction): 737 IDs before == after, 0 FAIL, 200 TestArgumentsData*/ subtests.
- Panic probe (V-A/repo/internal/vaprobe, removed after): nil *tfprotov5.DynamicValue: direct = callback = "runtime error: invalid memory address or nil pointer dereference";
  generic T-constrained call = "value method github.com/hashicorp/terraform-plugin-go/tfprotov5.DynamicValue.Unmarshal called using nil *DynamicValue pointer". Claim confirmed.
  Reachability: plugin-go fromproto never yields nil elements (fromproto.DynamicValue(nil)->nil only for nil proto msgs, which protobuf decoding never yields in repeated fields); only direct ProviderServer callers can pass nil.
- SIZE: git diff --no-index --numstat: fromproto5 +5/-589, fromproto6 +5/-589, core +611, doc.go +6 => +627/-1178 (net -551) physical lines.
  The author's own A2_argumentsdata.diff has 631 '+' / 1182 '-' incl. 4 header lines each = +627/-1178. The finding's "-1084/+573 (net -511)" matches only a NON-BLANK count (1110 non-blank -> 599). Correct the size.

## A3
- My normaliser (scripts/norm_cases.py; T at camel boundary, lc(T) at ident start): switch at fromproto5 162-559; 12 cases x 33 lines; 11 identical to Bool;
  List differs only in a local variable name (listValue vs listValuable at 2 lines) -> no behavioural effect.
- Draft A3 (author) installed over A2 in sandbox: build, vet, gofmt clean, test-identity 737 == 737, 0 FAIL. golangci-lint (audit bin, go1.26 build, project config) on fromprotocommon+fromproto5+fromproto6 = 0 issues.
- ConcatFuncErrors (function/func_error.go:82-110): joins non-empty Text with \n, first non-nil FunctionArgument; nil iff text=="" && fa==nil -> associative; inner loop never reads funcError.
- Differential probe (scripts/zz_va_a3probe_test.go) pristine repo0 vs A3 sandbox, 7 cases (cross-arg 2x2 errors; wrong Valuable x3 validators; ToBoolValue error+warning diags;
  empty FuncError{} then text no-FA then FA=99; variadic Int64 validators; ok then wrong Valuable; all pass): output byte-identical.
- SIZE: numstat A2 core -> A3 core = +135/-384 (net -249); author's A3_validators.diff agrees (136/385 incl. headers). Finding says -372/+128 (net -244). Correct.
- Side observation (pre-existing, not filed): ToXValue diag errors become FuncErrorFromDiags with NO FunctionArgument (case 3 fa=nil), unlike the other validator errors.

## A4
- grep -rn "fromproto[56]\.EphemeralResultData" --include=*.go . -> only fromproto5/6 ephemeral_result_data_test.go:109. No unqualified in-package use. No reflection.
- wc: 53+120 per protocol = 346 total. Sandbox: deleted 4 files -> go build ./... OK, go vet (compiles tests) + go test fromproto5/6 OK. Restored.
- Only functions removed; tfsdk.EphemeralResultData type and toproto5/6.EphemeralResultData stay (used by openephemeralresource.go:26).

## A5
- serve.go identical in both packages after normalisation. registerContext appends; only cancelRegisteredContexts (StopProvider) clears.
- Counts: handler call sites = 27 (proto5server, 27 server_*.go files) and 35 (proto6server); the finding's 29/37 include the definition line and serve_test.go.
- grpc-go v1.82.1 internal/transport/http2_server.go:1331-1335 finishStream -> s.cancel() (called from writeStatus :1140): unary RPC ctx IS canceled at RPC end. Premise of the fix holds.
- REALISTIC LEAK (scripts/leak_test.go): real grpc.Server + tf6server.GRPCProviderPlugin.GRPCServer + providerserver.NewProtocol6(testprovider.Provider), loopback TCP, conn.Invoke("/tfplugin6.Provider/GetMetadata", emptypb):
    pristine: 20000 RPCs retain 219.9 MiB = 11,528 bytes/RPC (2 runs: 11529, 11528)
    A5 draft: 0.04-0.05 MiB = 2-3 bytes/RPC
  heap profile (pristine, inuse_space): terraform-plugin-log LoggerOpts.Copy 44.5MB, SetSDKSubsystemTFLoggerOpts 39MB, context.WithValue 20.5MB, bytes.growSlice 20MB,
  grpc http2Server.operateHeaders 25.5MB cum, hclog.newLogger 22MB ... i.e. per-request logger chains + grpc stream objects reachable from the retained cancelCtx parents.
  => the finding's "about 0.5 KB per RPC lower bound" understates the real stack by ~23x.
- Semantics probe (scripts/zz_va_a5_test.go, package proto6server): (a) 1000 completed RPCs w/ canceled parents: len 1000 pristine / 0 draft; (b) in-flight ctx canceled by StopProvider in both;
  (b2) registration after StopProvider canceled by a 2nd StopProvider in both; (c) NEVER-canceled parent (context.Background): 106 B/registration pristine vs 624 B draft (AfterFunc child + map entry + closure) -> "no regression" claim is wrong for that corner (direct callers w/ Background ctx). Fix: only call AfterFunc when in.Done() != nil.
  Stress under -race (8 goroutines register+cancel while 200 StopProvider calls): 0 races, 0 missed in-flight cancellations, final len 0 (draft).
- Draft: go test ./internal/proto6server OK; -race -count=20 TestServerCancelInFlightContexts OK.
- Cancellation order on StopProvider becomes map order instead of registration order: not observable (done-channel wakeups are asynchronous anyway).

## A6
- proto5/6server server_invokeaction.go identical modulo names; Events iterator (63-92) relays from unbuffered fwResp.ProgressEvents; returns when push==false.
- fwserver/server_invokeaction.go:37-41 SendProgress: `r.ProgressEvents <- ...` (no select, no ctx).
- tf6server v0.31.0 server.go:1412+: `for ev := range resp.Events { select { case <-ctx.Done(): return nil; default: ... if err := protoStream.Send(...); err != nil { return err } } }`
  -> push returns false when the body returns: ctx done (StopProvider closes stopCh -> stoppable ctx; framework ctx canceled by registerContext registry) or Send error.
  Context cancellation does NOT unblock SendProgress (plain send). So plugin-go does not already handle it.
- Probe (scripts/zz_va_a6_test.go): consumer mirrors tf6server loop; StopProvider + consumer cancel after "step 1"; ctx-aware action sends "canceled, cleaning up" then "cleanup done".
  pristine: Invoke still blocked 2s later; goroutine parked in SendProgress. Author draft A6: Invoke returns. Variant with only ONE post-cancel SendProgress: no block in pristine
  (that event is consumed by the iterator, whose push returns false). => precondition: >= 1 SendProgress AFTER the push that returned false (i.e. >= 2 progress events after cancel, or any after a Send error).
- Draft: -race -count=3 TestServerInvokeAction+probe OK. Size: numstat +14/-0 per package (finding: +13).
- Impact wording: code after the blocked SendProgress (incl. deferred cleanup) never runs; code before it does. Process is usually killed at end of the Terraform command, which bounds the leak -> tier 3 OK.

## A7
- toproto5/getproviderschema.go:121-131: list loop appends diag, no return; toproto6/getproviderschema.go:118-130 returns. Both from ca23057 (2025-07-01, #1171).
- Probe (scripts/zz_va_a7_test.go): 3 list schemas with ListNestedAttribute over v5 -> 3 "Error converting list resource schema" diags; 3 bad resource schemas -> 1 diag. No test pins it.

## A8
- toproto6/schema_attribute.go:19-21 has no GetType()==nil guard; toproto5 :23-25 and toproto6/identity_schema_attribute.go:22-24 have it.
- Probe (scripts/zz_va_a8_test.go): testschema.Attribute{Optional:true}: v5 -> "must have Type set"; v6 SchemaAttribute and toproto6.Schema -> PANIC nil pointer dereference.
- Reachability: fwschema.Attribute.Equal takes fwschema.Attribute (internal type), so an external provider can only build such an attribute by EMBEDDING a framework attribute and overriding GetType to return nil. Built-in attributes never return a nil interface. => provider misuse only; tier 3 ok.

## A9
- proto6server/server_listresource.go:67-83 guards only ProtoV6IdentitySchema != nil; fromproto6.ResourceSchema returns (nil,nil) for s==nil || s.Block==nil -> typed nil in fwschema.Schema.
- fwserver/server_listresources.go:98-110: validation only when NO managed resource of that type exists, and accepts a complete v5 pair OR complete v6 pair.
- My probe (scripts/zz_va_a9_test.go), path 2 (no managed resource; complete v5 pair + identity-only v6 pair, served over v6): ListResourceFuncs diags = 0; provider List sees
  req.ResourceSchema==nil false (dynamic *schema.Schema); NewListResult PANIC "value method .../resource/schema.Schema.Type called using nil *Schema pointer". (author reproduced path 1.)
- HOLE in proposed fix: with ProtoV6Schema = &tfprotov6.Schema{} (Block nil) both pointers are non-nil, the proposed guard passes and the same panic occurs (reproduced).
  Fix must check the converted schema (nil result -> diagnostic) or s.Block != nil.

## A10
- Cited lines verified: toproto6 schema.go:28,38; block.go:51,62; schema_attribute.go:70; identity_schema.go:27; getproviderschema.go:58,72,90,104,118,132 (:86 functions loop cannot error, rightly excluded); getresourceidentityschemas.go:27; fromproto5/resource_schema.go:136, fromproto6:191.
- Name == map key at every converter: toproto5/6 schema_attribute.go Name: name; block.go TypeName: name; identity_schema_attribute.go Name: name. Old comparator `<` and slices.Sorted are both byte-wise.
- Repro (scripts/zz_va_a10_test.go, pristine): Schema with 2 invalid attrs, 50 runs -> 2 distinct errors; 8 invalid -> 8 distinct; GetProviderSchemaResponse 5 invalid + 1 good, 50 runs -> 10-11 distinct outcomes (diag + partial map keys).
  With author's toproto6 draft: 1 / 1 / 1 distinct.
- Success-path differential (scripts/zz_va_a10diff_test.go): resource schema with 3-level nested attributes, nested list/set/single blocks, mixed-case/underscore/digit/Unicode names;
  identity schema; GetProviderSchemaResponse (12 resources, 2 data sources); GetResourceIdentitySchemasResponse; JSON-marshaled (~10 MB), SHA-256:
  pristine a415103b...cfcb4 == draft a415103b...cfcb4. Byte-identical.
- Draft: go test ./internal/toproto6 ./internal/proto6server pass.
- SIZE: numstat of the 6 toproto6 draft files = +53/-92 (finding: "-74/+47 measured on toproto6"). Whole item re-estimate ~ +100/-170 (toproto5 same loops minus schema_attribute nested + fromproto 2x small).

## A11
- fromproto5/6 arguments_data.go:101-104 confirmed. Author's awk scan reproduces exactly 2 sites. My broader literal-aware scan (scripts/sepscan.py: any "...X" + "Y..." where X not in space/colon/paren/dash/slash/quote/= and not \n, Y a letter, same line or next line), all non-test .go: 2 hits, the same two. Claim "only these two" holds.
- Probe (scripts/zz_va_a11_test.go, param whose GetType()=testtypes.InvalidType): TEXT="Unable to Convert Function ArgumentAn unexpected error was encountered ... Please report this to the provider developer:\n\nUnable to convert tftypes to framework type at position 0: intentional ValueFromTerraform error". No test covers it.

## A12
- proto6server/server_writestatebytes.go:20-21 (registerContext, InitContext) and 85-87 (three SetField with literal "tf_state_store_type"; logging.KeyStateStoreType = same, internal/logging/keys.go:47).
  The finding's "81-87" includes the MAINTAINER NOTE (81-84).
- framework InitContext (internal/logging/context.go) uses tfsdklog.NewSubsystem(..., WithRootFields()) -> root fields copied only at creation (terraform-plugin-log v0.10.0 tfsdklog/options.go:59-63).
- My probe (scripts/zz_va_a12_test.go, tfsdklogtest.RootLogger): pristine sdk.framework entries with field=1 / without=8; with proposed SubsystemSetField(framework) line: 9/0.

## A13
- proto5server/server_moveresourcestate.go HasError at 31, 43, 47, 55 -> duplicate pair 43-45 / 47-49; proto6server 30, 42, 46, 54 -> pair 42-44 / 46-48. Nothing between the pair.
- Author's consecutive-identical-if scan (repo-wide, non-test) re-run: only these two files.
- diag.Diagnostics.HasError (diag/diagnostics.go:78-86) only calls Severity() per diagnostic: pure for every framework diagnostic type; deleting the 2nd check drops a 2nd Severity() pass only.

## A14
- 12 non-test sort.Slice repo-wide; 11 in toproto5/6 (the 12th: fwserver/server_planresourcechange.go:552). Sites match the finding.
- Nil unreachable: every success return in SchemaAttribute/Block/IdentitySchemaAttribute (v5+v6) is a freshly &-allocated pointer; all error returns are (nil, non-nil err) and every caller returns on err before append.
- Keys unique (map-derived Name/TypeName == key) -> unique order; sort.Slice and slices.SortFunc both pdqsort/unstable, moot. strings.Compare is byte-wise like `<`.
- Digest differential (A10 probe on toproto6; v5 variant with no nested attrs on toproto5): pristine == A14 draft (v6 a415103b..., v5 3a91b3f1...).
- Draft: toproto5/6 + proto5/6server tests pass; golangci-lint (go1.26 build, project config) on toproto5/6: 0 issues.
- SIZE: author's own A14_sortfunc.diff = 124 '-' / 43 '+' incl. 7 headers each => -117/+36 (11 sites x -10/+2 = -110/+22; 7 import swaps -1/+2 = -7/+14). Finding says "-95/+36". Correct to -117/+36.

## A10 scope correction
- fromproto5/6 resource_schema.go:136/191 loop calls basetypes.TerraformTypeToFrameworkType, whose only error is "unsupported tftypes.Type detected: %T" (types/basetypes/terraform_type_to_framework_type.go:94).
  It handles all 9 tftypes kinds (Bool, Number, String, DynamicPseudoType, List, Map, Set, Object, Tuple), and tftypes.Type has unexported methods private()/supportedGoTypes(), so no
  other implementation can exist. => the error is unreachable; the fromproto part of A10 has no observable nondeterminism. Drop it from the bug's scope (or keep as consistency-only).

## A5 size
- numstat of author's v6 draft: +19/-2 per package (finding "-2/+15"); x2 = +38/-4.

## A15
- fromproto5/6 arguments_data_test.go: 3134/3135 lines; identical after normalisation except 1 blank line (diff "21a22 >"); 2 test funcs; 100 subtests each (200 TestArgumentsData*/ in my test-identity run).
- DynamicValueMust lives in fromproto6/dynamic_value_test.go:22 (would need copying to the shared package). Dependency on A2 is real (no shared core without it).
- Twin test stats (author's norm.sh): toproto 45 pairs / 30 identical / 4870 lines; fromproto 35/28/5183; protoNserver 25/12/3996 -> match the finding's notes.
- Size: with rename detection about -3134/+~150; without, -6269/+~3255. Estimate -3100/+150 is fine as a rename.

## A16
- proto5server/serve_test.go:18-56, proto6server 17-55 (twins). 200 ms sleep at :48 (v6). A late registrant (after cancelRegisteredContexts) is never canceled -> 10 s timeout error;
  shown by my A5 probe (b2): a post-stop registration is canceled only by a LATER StopProvider. Barrier fix is test-only.

## A17
- providerserver/providerserver.go:78 "blocking until the context is canceled": ctx only reaches opts.validate -> validateAddress(_ context.Context) ignores it; tf5server.Serve (v0.31.0 tf5server/server.go:257) has no ctx.
- serve_opts.go:31-35 (bullet at :34) names tfsdk.Attribute; no `type Attribute` in tfsdk/ (only stale mentions in internal/fwschema comments, explanatory).

## A18
- fwserver/server_upgraderesourceidentity.go:24-26. Sandbox: removed field+comment -> go build ./... and go vet (compiles tests) of fwserver, fromproto5, fromproto6, proto5server, proto6server all OK
  (these are ALL packages that mention UpgradeResourceIdentityRequest). Zero reads/writes anywhere. B2 notes line 145: "area A's A18; not refiled"; B2-20 references it. No duplicate.

## Leads
- exhaustive (8): read all; every switch has a default returning an error ("no supported block for nesting mode", "no supported nested attribute", "unrecognized nesting mode") or DiagnosticSeverityInvalid. Intentional. ENDORSED.
- nilnil (8): read 5 (fromproto5 identity_schema.go:21, fromproto6 resource_schema.go:22, toproto5 schema.go:18, toproto5/6 identity_schema.go:18), others are twins. toproto Schema(nil)->(nil,nil) feeds optional ProviderMeta etc. fromproto ResourceSchema/IdentitySchema callers: ONLY proto5/6server server_listresource.go:71/78 and :70/77 -> A9 is the only consequence. ENDORSED.

## Not-equivalent endorsements (checked)
1. Generic ArgumentsData[T] changes nil-element panic text (probe, see A2). 2. MoveResourceState: moving 1st HasError up skips identity lookup + its diags (read :34-44). 3. Deleting registerContext breaks direct ProviderServer callers (TestServerCancelInFlightContexts; stoppableContext only exists in tf5/6server's gRPC wrapper).
4. Sorted iteration changes which error is reported (probe 2->1 distinct). 8. ReadStateBytes: provider Read (:43) runs before ChunkSize check (:56); hoisting the check drops a provider call.

## B2 hand-off (fwserver/server_listresource.go:91-100)
- CONFIRMED by probe through proto6server.ListResource with protoReq.Config == nil (fromproto6.Config returns nil for nil DynamicValue): provider List sees
  req.Config.Schema type = managed resource ObjectType["id","name"]; GetAttribute(path.Root("filter")) -> 1 error "Configuration Read Error ... This is always an error in the provider".
  Contrast: protoReq.Config = msgpack null of the list config type -> Config.Schema = list config schema, GetAttribute(filter) -> 0 errors.
- History: 9cafd72 (#1157) returned error "Config cannot be nil"; 59a937e (#1178) replaced it with this fallback. fwserver test "zero-results-on-nil-config" still sets expectedError "config cannot be nil", which the runner never asserts (stale).
- Third combination (Config nil, ResourceSchema nil, identity non-nil) leaves Config nil -> *fwReq.Config at :144 panics; unreachable from proto5/6server (they pass a non-nil interface or return early).
- ALREADY FILED by B2 as B2-6 part (4). Report as merge note, do not duplicate. Reachability from Terraform (does core ever send a nil Config?) is unverifiable here.
