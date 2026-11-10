# Area A (protocol layer): working notes

Repo at 00badb1, read-only. All drafts and probes ran in a sandbox copy:
`drafts/A/repo`
(a tar copy without .git, restored to pristine after each draft; `diff -rq` against the repo is clean at the end).
Work dir: `drafts/A` (called `$W` below).

## 1. Coverage: what was swept exhaustively and what was sampled

Exhaustive:
- **Every non-test file of the area was read:**
  - toproto6, fromproto6 and proto6server in full;
  - the v5 twins only where they differ after normalisation (the rest are identical);
  - totftypes, fromtftypes, logging and providerserver in full.
- **Twin diff:** every same-name pair, plus the renamed pairs (Prepare/ValidateProviderConfig, ValidateResourceTypeConfig/ValidateResourceConfig, ValidateDataSourceConfig/ValidateDataResourceConfig), was diffed after 5/6 normalisation. Section 2 classifies every difference.
- **Scripted field audits:**
  - every tfprotov5/6 request field against what fromproto reads;
  - every protocol response field against what toproto sets;
  - every fwserver response field against what toproto reads;
  - every fwserver request field against what fromproto sets;
  - every `Field: protoN.Other` / `Field: fw.Other` mapping where the names differ.
  
  Results: GenerateResourceConfig dropped (A1), UpgradeResourceIdentityRequest.TypeName dead (A18). All other gaps are intentional: TypeName and ActionType are used for lookup; UnsafeToUseLegacyTypeSystem, SchemaBlock.Version and nested-block MinItems/MaxItems are deliberately unset; ValidateListResourceConfig IncludeResourceObject/Limit have no public framework field to land in (feature gap, not a converter bug); v5 GetMetadata ignores StateStores (protocol delta).
- **Client capabilities:** every tfprotov5/6 *ClientCapabilities field is mapped, and every converter wires its capability (grep `ClientCapabilities(`).
- **Handler wiring:** for every proto6server handler, the type name, schema and converter passed to each step were checked. All are consistent.
- **Unused identifiers:** every exported identifier in the internal packages was checked for non-test use. Only EphemeralResultData is dead (A4). Unexported identifiers are all used.
- **Mechanical scans:**
  - consecutive identical if-blocks, repo-wide: only A13;
  - string concatenation missing a separator, repo-wide: only A11;
  - range over a map whose order is observable: A10;
  - RPC names inside string literals: one, and it is correct.
- **All 8 nilnil and all 8 exhaustive leads were read** (section 4).

Sampled: test files. I read the lead sites (rangeint, waitgroup, newexpr, gocritic, godox, perfsprint), the InvokeAction, ListResource, WriteStateBytes, serve and arguments_data tests, and the getproviderschema test cases I reused. I did not read the remaining ~70k test lines of table data.

## 2. Twin differences (item 7): protocol delta or drift?

Normaliser (`$W/norm.sh`):
`sed -E -e 's/tfprotov[56]/tfprotovN/g' -e 's/toproto[56]/toprotoN/g' -e 's/fromproto[56]/fromprotoN/g' -e 's/proto[56]server/protoNserver/g' -e 's/tf[56]server/tfNserver/g' -e 's/Proto[56]/ProtoN/g' -e 's/proto[56]/protoN/g' -e 's/ProtocolVersion[56]/ProtocolVersionN/g' -e 's/protocol version [56]/protocol version N/g' -e 's/ProviderServer[56]/ProviderServerN/g' -e 's/protov[56]/protovN/g'`

Same-name non-test pairs with my normaliser (they differ by one from the brief's 42/21 because normalisers differ):
- toproto: 49 pairs, 43 identical;
- fromproto: 36 pairs, 27 identical;
- protoNserver: 26 pairs, 22 identical.

### toproto5 vs toproto6

| File | Difference | Verdict |
|---|---|---|
| getmetadata.go | v6 adds StateStores | protocol delta |
| getproviderschema.go | v5 emits "Unsupported State Store(s)"; v6 converts state store schemas | protocol delta |
| getproviderschema.go | v5 list-resource loop has no `return` | **DRIFT, bug A7** |
| identity_schema_attribute.go | message wording ("...and protocol version 5...") | intentional |
| schema_attribute.go | v5 rejects NestedAttribute; v6 converts it | protocol delta |
| schema_attribute.go | v5 has the `GetType()==nil` guard and v6 does not (v6 identity converter does) | **DRIFT, bug A8** |
| server_capabilities.go | v6 adds StateStoreServerCapabilities | delta. Both drop GenerateResourceConfig (A1); that is a shared bug, not drift |
| validatedatasourceconfig.go | DataSource vs DataResource type rename | delta |
| v5-only / v6-only files | renamed RPCs or state store RPCs | renamed pairs are identical after renaming |

### fromproto5 vs fromproto6

| File | Difference | Verdict |
|---|---|---|
| client_capabilities.go | rename; v6 adds ConfigureStateStore | delta |
| identity_schema.go, resource_schema.go | comment mentions SDKv2 only in v5 | correct, SDKv2 is v5-only |
| resource_schema.go | v6 handles NestedType | delta |
| moveresourcestate.go, upgraderesourcestate.go, upgraderesourceidentity.go | v5 converts `*tfprotov5.RawState` to `*tfprotov6.RawState`, because fwserver standardises on v6 RawState | delta |
| plan.go | import order | cosmetic |
| readdatasource.go | struct literal field order | cosmetic |
| moveresourcestate.go (v5) | import block has no blank line between groups | cosmetic |

### proto5server vs proto6server

| File | Difference | Verdict |
|---|---|---|
| server_configureprovider.go, server_readresource.go | import order | cosmetic |
| server_moveresourcestate.go | one blank line | cosmetic. Both have the duplicate check (A13) |
| server_listresource.go | RawV5 vs RawV6 | delta |
| server_listresource.go | v5-only comment claiming validation guarantees both raw schemas | the claim is false (A9) |
| renamed RPC files | identical after renaming | — |

## 3. ArgumentsData analysis (item 1)

- `fromproto5/arguments_data.go` and `fromproto6/arguments_data.go` are identical after normalisation.
- Lines that mention tfprotovN: 4 (import, doc comment, signature).
- Lines that touch the protocol value: 2 (the `range arguments` clause and `argument.Unmarshal(...)`).
- Everything else is framework-typed. Protocol-specific lines that must stay per protocol after A2: the `[]*tfprotovN.DynamicValue` parameter and `arguments[position].Unmarshal(typ)`.

Measurements:
- A2 draft: -1084/+573. Byte diff of the core against today's fromproto6 file: 11 hunks (`diff $O/internal/fromproto6/arguments_data.go $W/drafts/A2_fromprotocommon_arguments_data.go`).
- A3 on top of A2: -372/+128. The 12 validator cases were proven identical by normalising type names (`$W/gen` scripts, run inline). Bool/Dynamic/.../String all match; the two "DIFF"s were artifacts: "basetypes" contains "set", and the last case carries the closing brace.
- Pinning tests: `go test -count=1 -v -run 'TestArgumentsData' ./internal/fromproto5/ ./internal/fromproto6/ | grep -c -- '--- PASS: TestArgumentsData.*/'` gives 200.

Other protocol-neutral candidates were considered and not filed:
- **fromproto DynamicValue/Config/Plan/State/EphemeralResultData/IdentityData/ProviderMeta.**
  - About 35 non-blank lines each.
  - Protocol touch points: the parameter type, the nil check, and Unmarshal.
  - The results are distinct tfsdk structs, so sharing needs an unmarshal callback, an isNil flag and a constructor callback.
  - Saves about 25 lines per file pair and reads worse. Rejected.
- **fromproto ResourceSchemaAttributes/IdentitySchemaAttributes.**
  - The logic reads protocol struct fields throughout.
  - Sharing needs a neutral DTO ({Name, Type, Required, Optional, Computed, WriteOnly, Sensitive}) built per protocol.
  - Estimated about -135 lines plus a new indirection per attribute. Not recommended unless that switch needs to change again.
- **toproto converters and protoNserver handlers.** They build protocol structs or wire protocol types end to end; there is no shared core without struct-field generics (which Go lacks).

## 4. Lead classification (728 leads)

| Lead kind | Count | Disposition |
|---|---|---|
| golangci/dupl | 379 | 351 in tests: table-driven, deliberate. 28 non-test, all in protoNserver handlers: same shape, different types. Measured: server_validateresourceconfig.go vs server_closeephemeralresource.go differ on 22 of 50 lines, every type-bearing line. A shared helper would need about 5 type parameters and 3-4 function values. Rejected as not simpler. Apply/Plan/Read share a 5-lookup prelude; a helper saves about 55 lines per package. Rejected: the explicit per-RPC wiring is the convention and reads better. |
| golangci/modernize + gofix/any | 101 + 95 | Coordinator-owned. Non-test: logging/framework.go:18,23,28,33 (`map[string]interface{}` in signatures; alias, so byte-identical for callers). |
| gofix/rangeint + golangci/intrange | 4 + 4 | Same 4 sites: serve_test.go:29/28 (i unused) and server_invokeaction_test.go:239 (i only read). Rewrite is behaviour-preserving. |
| gofix/waitgroup | 2 | serve_test.go:31/30: `wg.Add(1); go func(){ defer wg.Done(); ... }()` becomes `wg.Go`. Identical. |
| gofix/newexpr | 4 | toproto5/6 pointer_test.go + callfunction_test.go:41. Identical; coordinator-owned. |
| golangci/funlen | 88 | 87 in tests (table data). The 1 non-test is fromproto6 ResourceSchemaAttributes (163 lines). Splitting out the NestedType branch is possible but low value; not filed. |
| golangci/gocognit | 10 | ArgumentsData (addressed by A2/A3); the rest are test functions. |
| golangci/exhaustive | 8 | All intentional: every switch has a default that either returns an error ("unrecognized nesting mode", "no supported block/nested attribute") or maps to Invalid (DiagnosticSeverity). fromproto Block nesting Group and Map are unsupported by design. Nit: the MAINTAINER NOTE in fromproto*/resource_schema.go says only map nested blocks are unsupported, but Group is unsupported too. Comment accuracy only; not filed. |
| golangci/nilnil | 8 | Intentional nil-in, nil-out convention. toproto Schema(nil) yields a nil ProviderMeta when there is no meta schema. The only harmful consequence is the typed nil at the interface boundary in ListResource (A9). |
| golangci/perfsprint | 14 | Linter not enabled in CI; trivial. fmt.Errorf without verbs returns an `*errors.errorString` just like errors.New, so the rewrite would be equivalent. Not filed. |
| golangci/gocritic newDeref | 4 | `*new(time.Time)` vs `time.Time{}` in tests. Trivial; not filed. |
| golangci/godox | 6 | "TODO tuple support" (issue #54). Not actionable. |
| deadcode | 1 | providerserver.Serve: public API. False positive. |

## 5. "Looks equivalent but is not": rejected rewrites

1. **Generic `ArgumentsData[T interface{ Unmarshal(tftypes.Type) (tftypes.Value, error) }]`.** A probe (`$W/probe_generic`) shows that calling a value-receiver method through a type parameter on a nil pointer panics with "value method main.DV.Unmarshal called using nil *DV pointer". A direct call panics with "invalid memory address or nil pointer dereference". Hence A2 uses a callback, which keeps the direct call.
2. **MoveResourceState: moving the first HasError check up** between the ResourceSchema and ResourceIdentitySchema lookups to match other handlers. Not equivalent: it skips the identity lookup and its diagnostics when ResourceSchema errors. Only deleting the duplicate (A13) is equivalent.
3. **Deleting registerContext and relying on terraform-plugin-go's stoppableContext.** Not equivalent for direct ProviderServer users (mux, tests, NewProtocol5/6 consumers). TestServerCancelInFlightContexts pins it.
4. **Sorted-key iteration as a "refactor".** It changes which error is reported on the error path, so it is filed as a bug fix (A10). The equivalent alternative is A14.
5. **Generic helper for GetProviderSchemaResponse's 6-7 identical loops.** It cannot cover the v5 list loop until A7 is fixed, because that loop differs. About -55 lines per package. Not filed; do it after A7/A10 if at all.
6. **A3 nested accumulation of FuncErrors.** It looked risky but is equivalent: ConcatFuncErrors is associative (text join skips empties; the first non-nil FunctionArgument wins).
7. **Hoisting `tfType.Steps()` in fromtftypes.AttributePath.** Equivalent, but the gain is negligible (Steps() copies a short slice). Not filed.
8. **Removing ReadStateBytes' provider call before the ChunkSize check** (it wastes a provider call). That reorders side effects: the provider's Read would no longer run. Not equivalent; not filed.

## 6. Cross-area hand-offs (not filed as area-A findings unless noted)

- **B2:** fwserver/server_listresource.go:91-100. When fwReq.Config is nil, it fabricates a null *list config* typed with `fwReq.ResourceSchema` (the managed-resource schema), not the list resource config schema, and sets `Schema: fwReq.ResourceSchema`. This looks wrong (provider `req.Config.Get` would decode against the wrong schema), but it is only reachable when Terraform sends no Config. Please verify.
- **B2:** fwserver.InvokeActionResponse.SendProgress does an unconditional send on an unbuffered channel. This is the root cause of A6; A6's fix is in protoNserver.
- **B2:** UpgradeResourceIdentityRequest.TypeName is dead. Filed as A18 because it was found here; drop it if B2 also has it.
- **Coordinator:** all modernizer leads in this area (any, rangeint, intrange, waitgroup, newexpr) are behaviour-preserving at every site; I checked each non-`any` site.

## 7. For the verifier

- **A1.** Check Terraform core gating. I fetched raw main-branch sources: internal/terraform/node_resource_plan_instance.go:1191 `if providerSchema.ServerCapabilities.GenerateResourceConfig {` falls back to genconfig.ExtractLegacyConfigFromState. Re-confirm against the v1.14 tag if possible. Also confirm the 1.19.0 CHANGELOG line (CHANGELOG.md:5).
- **A5.** The fix assumes grpc-go cancels a unary RPC's stream context when the handler returns, so that AfterFunc fires. If it did not, growth would simply continue as today (no regression).
- **A9.** The precondition is a provider misconfiguration that passes fwserver ListResourceFuncs validation: an identity-only raw schema plus a framework managed resource of the same type, or a complete v5 pair with a half v6 pair when served over v6.
- **A10.** The success-path claim rests on names being unique map keys equal to the Name/TypeName that the old comparator sorted on (SchemaAttribute sets `Name: name`, Block sets `TypeName: name`, IdentitySchemaAttribute sets `Name: name`).
- **A2/A3.** Re-run `go test ./internal/fromproto5 ./internal/fromproto6` and `golangci-lint run ./internal/fromprotocommon/... ./internal/fromproto5/... ./internal/fromproto6/...` after copying the drafts into a sandbox.

## 8. Commands that reproduce the counts

```sh
cd /home/user/terraform-plugin-framework
W=drafts/A
# twin pairing (non-test, same name)
for pair in "toproto5 toproto6" "fromproto5 fromproto6" "proto5server proto6server"; do set -- $pair; p=0; s=0; for f in $(ls internal/$1/*.go | grep -v _test.go | xargs -n1 basename); do [ -f internal/$2/$f ] && { p=$((p+1)); diff -q <($W/norm.sh internal/$1/$f) <($W/norm.sh internal/$2/$f) >/dev/null && s=$((s+1)); }; done; echo "$1 $p $s"; done
# A1: converters never mention the field; 30 test expectations encode it as false
grep -n GenerateResourceConfig internal/toproto5/server_capabilities.go internal/toproto6/server_capabilities.go   # (none)
grep -rc "PlanDestroy: *true" --include=*_test.go internal/proto5server internal/proto6server                    # 6+8+7+9 = 30
# A4: dead converters
grep -rn "EphemeralResultData(" --include=*.go . | grep -v _test.go
# A11: missing separator (repo-wide)
for f in $(find . -name '*.go' -not -name '*_test.go'); do awk -v F=$f '{ if (prev ~ /[A-Za-z0-9)]"[ \t]*\+[ \t]*$/ && prev !~ /\\n"[ \t]*\+[ \t]*$/ && $0 ~ /^[ \t]*"[A-Z]/) print F":"NR-1; prev=$0 }' $f; done
# A13: consecutive identical if-blocks (repo-wide)
find . -name '*.go' -not -name '*_test.go' | while read f; do awk -v F=$f '{ gsub(/^[ \t]+/, "", $0); if ($0=="") next; l[++n]=$0 } END { for (i=1;i+5<=n;i++) if (l[i] ~ /^if / && l[i+2]=="}" && l[i]==l[i+3] && l[i+1]==l[i+4] && l[i+5]=="}") print F }' $f; done
# A14: sort.Slice sites
grep -rn "sort.Slice(" --include=*.go . | grep -v _test.go | wc -l          # 12 repo-wide, 11 in toproto5/6
# A5: registerContext call sites
grep -c registerContext internal/proto5server/*.go internal/proto6server/*.go | awk -F: '{s+=$2} END {print s}'
# twin test duplication
# (same loop as above over *_test.go): toproto 30 identical pairs/4870 lines, fromproto 28/5183, protoNserver 12/3996
```

## 9. Scratch artifacts

All under `$W` (= `drafts/A`):

- **Drafts** (`$W/drafts/`):
  - A2: A2_fromprotocommon_arguments_data.go, A2_fromproto5_arguments_data.go, A2_fromproto6_arguments_data.go
  - A3: A3_fromprotocommon_arguments_data_validators.go
  - A5: A5_proto6server_serve.go
  - A6: A6_proto6server_server_invokeaction.go
  - A10: A10_sorted_iteration_toproto6/
  - A14: A14_sortfunc/
- **Diffs** (in `$W`): A2_argumentsdata.diff, A3_validators.diff, A5_registercontext.diff, A6_invokeaction.diff, A10_toproto6_sorted.diff, A14_sortfunc.diff.
- **Probes** (`$W/probes/`):
  - zz_probe_test.go (InvokeAction leak, registry growth, ArgumentsData text)
  - zz_probe_mem_test.go (bytes per RPC)
  - zz_probe_list_test.go (typed-nil ListResource)
  - zz_probe_log_test.go (WriteStateBytes log fields)

  The toproto5/6 probes for A1/A7/A8/A10 were inline and are reproduced in the findings' proof text.
- **Generic panic probe:** `$W/probe_generic/`.
- **Terraform core sources fetched for A1:** `$W/tf_node_resource_plan_instance.go`, `$W/tf_provider.go`.
