# B2 notes: fwserver RPC handlers and registries

Area: `internal/fwserver/*.go` except `attribute_*`, `block_*`, `schema_*` (B1).
Scratch root: `drafts/B2` (below: `$W`).
Repository untouched (clean `git status` at 00badb1); all edits were made in copies under `$W`.

## 1. Coverage

- **Read in full (exhaustive):** all 50 non-test files of the area (7,784 lines). That is every RPC handler (`server_*.go`), `server.go`, the per-concept registry files, `attr_type.go`, `attr_value.go`, `diagnostics.go`, `write_only_nullification.go` and `server_capabilities.go`.
- **Tests (40 files, 37,404 lines): sampled.**
  - Read: the deferral cases in the PlanResourceChange, ReadResource and ImportResourceState tests; all of `server_listresource_test.go`; the multi-mover diagnostics cases of the MoveResourceState test; the sort calls in the GetMetadata test.
  - Grepped every handler test for its cmp options, `ConfigureMethod` coverage, `AddWarning`, write-only coverage and semantic-equality coverage.
  - The 188 test dupl leads are table twins (one per RPC) and are not itemised.
- **Tools** (all under `$W`):
  - `funclen/`: function lengths.
  - `fieldscan/`: a go/types scan that lists struct fields never read anywhere in the module, tests included.
  - `audit/bin/golangci-lint`: run per package on each draft, with the project `.golangci.yml` (all drafts report 0 issues) and with `$W/gocog.yml` (gocognit and nestif).
  - `go test`, one package at a time (fwserver, proto5server, proto6server, providerserver).
- **Terraform core sources** (main branch, fetched read-only into `$W/ext/` to complete the bug traces):
  - node_resource_abstract_instance.go, node_resource_plan_instance.go, node_provider.go, upgrade_resource_state.go;
  - internal/providers/provider.go (`provider_iface.go`);
  - internal/plugin6/grpc_provider.go (`grpc_provider6.go`).

## 2. Lead classification (494 leads)

| Lead | Count | Disposition |
|---|---|---|
| gofix/any + modernize any | 107 + 106 | Coordinator-owned. Every site is an alias swap (`map[string]interface{}` log fields, signatures). None changes behaviour: `%T` prints `interface {}` for both spellings. |
| dupl | 204 | 16 non-test leads, all covered: validate twins (B2-14, with B2-10), registries (B2-11/12/13), unlockstate vs writestatebytes (their Configure preamble, B2-10). The 188 test leads are table twins; not itemised. |
| funlen | 32 | 2 non-test (CreateResource 176, UpgradeResourceState 188), reduced by B2-10/15/16; phase maps in §4. 30 test functions: not itemised. |
| gocognit | 12 | Non-test: PlanResourceChange 71 (B2-17: 51; B2-1 alone: 69); MarkComputedNilsAsUnknown 62 (12-case default-value switch, no shared interface, §6); GenerateResourceConfig 47 (B2-19 does not move it); MoveResourceState 31; ReadResource 31; UpdateResource 32. These last three shrink with B2-10/15/16. The 6 test leads are not itemised. |
| nestif | 4 | planresourcechange.go:200 removed by B2-17; :313 removed by the B2-1 fix; generateresourceconfig.go:77 (10) and listresource.go:105 (8): the gain is small, see §6. |
| errorlint | 1 | Rejected, not a bug. planresourcechange.go:451 `err != tftypes.ErrInvalidStep`: tftypes v0.31.0 returns the sentinel unwrapped (attribute_path.go:364-366, value.go:133-170), and tftypes' own diff.go:97 compares with `!=`. `errors.Is` would only differ if a future version wrapped the error. |
| gocritic newDeref | 2 | B2-22. |
| perfsprint | 16 | Style only; perfsprint is not in the project linter set. 4 non-test sites build diagnostic or debug text, so the output is identical either way and there is no value. |
| godox | 10 | listresources.go:76 `TODO: y?` is B2-9. upgraderesourcestate.go:78 is a feature idea (#113). The 8 test TODOs reference upstream issues. |

## 3. Registries (task 1)

- **Funcs methods.** Normalised diff of the seven `<Concept>Funcs` methods (`$W/reg/*.norm`): pairs share 21-32 of 41-67 non-blank lines. The differences are only:
  - strings (quoted in B2-11's notes) and types;
  - the optional With* interface check (ephemeral, action, state store, function);
  - functions skip ProviderTypeName;
  - ListResourceFuncs checks the interface before the lock and returns `nil, nil` uncached;
  - list's extra matching-managed-resource check.
- **Duplicate detection order.** The loop iterates the provider's returned slice in order, so the first-seen name wins. The map is only a membership test, and nothing sorts. The generic keeps this.
- **Locking.** The lock trace is logged before `Lock()`. `defer Unlock` covers the provider calls, ProviderTypeName and, for list, `s.ResourceFuncs(ctx)`, which takes the resource lock while holding the list lock. That is the same lock order as today and the same as ResourceBehaviors, so there is no cycle.
- **Not covered by the generic:**
  - ResourceBehaviors (caches values, not factories).
  - Function lookups (FuncError).
  - ListResourceType, which returns `nil` diags on success where every sibling returns the cached registry diags. That is an inconsistency, but the cached errors already fail GetProviderSchema and GetMetadata; not filed.
- **Measured drafts** (`git diff --no-index --stat` against the repo):
  - registry `+257/-423`;
  - schemaCache `+58/-141`;
  - lookups, metadatas and schemas `+146/-300`.
  - All three touch the same files, so apply them in the order B2-11, then B2-12, then B2-13.

## 4. Long handlers: phase maps (task 3)

**PlanResourceChange** (54-412, 359 lines):

| Phase | Lines | What it does |
|---|---|---|
| P0 | 55-57 | nil guard |
| P1 | 59-75 | Automatic-deferral short-circuit. Writes PlannedState, PlannedPrivate and Deferred; Debug trace; returns. |
| P2 | 77-94 | Configure (B2-10). |
| P3 | 96-119 | Null-fills req.Config, ProposedNewState and PriorState (mutates req). |
| P4 | 121-137 | Null PriorIdentity, then a copy into resp.PlannedIdentity (B2-16). |
| P5 | 139-151 | PlannedPrivate. Aliases req.PriorPrivate and may write its Provider field. |
| P6 | 153 | PlannedState from the proposed state. |
| P7 | 155-177 | TransformDefaults. Appends diags; may return. |
| P8 | 179-189 | Write-only nullification. May return. |
| P9a | 205-248 | Changed-path Debug traces. Pure; B2-17. |
| P9b | 250-267 | MarkComputedNilsAsUnknown transform. May return. |
| P10 | 270-303 | Schema plan modifiers. The response is seeded with resp.Diagnostics, which it then replaces; appends RequiresReplace; may return. |
| P11 | 305-372 | Resource ModifyPlan plus provider deferral (B2-1). |
| P12 | 374-397 | Identity validation. |
| P13 | 399-400 | NormaliseRequiresReplace. |
| P14 | 402-411 | Destroy-plan null check. |

Order pins:
- P7 must precede P8, and P9 reads P8's plan.
- P10 and P11 seed their responses with resp.Diagnostics, so earlier warnings survive and their errors stop the handler.
- P12-P14 read P11's output.

Only P9a can be extracted without touching shared state, so only it is proposed.

**UpgradeResourceState** (47-268, 222 lines):

| Phase | Lines | What it does |
|---|---|---|
| U0 | 48-57 | Guards. |
| U1 | 59-66 | unmarshalOpts. |
| U2 | 86-110 | Same-version passthrough (B2-7). |
| U3 | 112-129 | Configure. |
| U4 | 131-164 | Upgrader lookup (two errors). |
| U5 | 166-190 | Unmarshal of the prior state. |
| U6 | 192-212 | Upgrader call. |
| U7 | 214-244 | DynamicValue path. |
| U8 | 246-267 | State path. |

Extracting U2 is mechanical: it returns `(*tfsdk.State, diag.Diagnostics)` and shares no state. That moves about 25 lines out but does not shrink the code. U7 and U8 share an identical nullify block, which could become one helper (about -8 lines). Not drafted: low value.

**ReadResource, UpdateResource, CreateResource** (209, 208 and 181 lines) have the same shape:
1. guards and auto-deferral (Read only);
2. Configure;
3. request/response initialisation, including private state;
4. identity pre-population and copies;
5. provider call, which overwrites resp.Diagnostics (B2-3);
6. identity checks (wording differs per RPC, so they are not parameterised);
7. semantic equality (B2-15);
8. write-only nullification (identical 11 lines in all three, -17 if extracted; not filed).

With B2-10, B2-15 and B2-16 the three handlers come to about 165, 165 and 140 lines.

**MoveResourceState** (90-276, 187 lines):
1. guards;
2. interface check;
3. private state;
4. the mover loop, which has three returns that end the RPC and the diagnostics overwrite that test 'response-Diagnostics-first-error-always-responds' pins;
5. the "no match" error.

Extracting the loop body is not mechanical. The source-state unmarshal (163-214) could become a pure helper that returns `*tfsdk.State`, which would bring gocognit below 30. Not filed.

**ImportResourceState** (69-237, 169 lines):
1. guards;
2. auto-deferral (B2-2);
3. Configure;
4. interface check;
5. initialisation and identity (B2-16);
6. provider call;
7. nullify;
8. empty-state check;
9. private state with ImportBeforeReadKey;
10. identity check;
11. response.

**GenerateResourceConfig** (44-184, 141 lines): the Transform callback has three identical error blocks (B2-19). The nesting comes from the attribute-or-block if/else.

## 5. Dead code sweep (task 5)

- **fieldscan.** Run on `internal/fwserver` against the whole module with tests: `$W/fieldscan/fieldscan github.com/hashicorp/terraform-plugin-framework/internal/fwserver /home/user/terraform-plugin-framework`. Output, area files only:
  - `ServerCapabilities.GenerateResourceConfig` (reads=0, keys=66). This is area A's A1: toproto5/6 drop the capability. Independently confirmed by probe `$W/repo/internal/proto6server/zz_probe_caps_test.go`: v5 and v6 GetProviderSchema and v6 GetMetadata all report `GenerateResourceConfig:false`. Core gates the RPC on it (`$W/ext/node_resource_plan_instance.go:1191`). Not refiled.
  - `CloseEphemeralResourceRequest.EphemeralResourceSchema`, `RenewEphemeralResourceRequest.EphemeralResourceSchema`, `CreateResourceRequest.PlannedPrivate`: B2-20.
  - `UpgradeResourceIdentityRequest.TypeName` (reads=0, writes=0, keys=0): area A's A18; not refiled.
- **Exported identifiers with 0 references:** `fwserver.NoListResults` (B2-20).
- **Callers of every unexported helper in the area:** all have callers.
  - `stateToConfig`, `planToState` and `stateToPlan` are used.
  - The `attr_value.go` helpers are used by B1's files. `listElemObjectFromTerraformValue` and `setElemObjectFromTerraformValue` are identical except for the List/Set parameter type; merging them saves about 12 lines. Left to B1, as a note.
- **Unreachable branches:** ResourceBehaviors' missing and duplicate errors (server.go:619-638) are reachable only if a provider's Metadata is non-deterministic (it runs on a second instance). They are not provably dead and were not filed.

## 6. Considered and rejected ("looks equivalent but is not")

1. **Configure helper returning only diagnostics, with an unconditional `if resp.Diagnostics.HasError() { return }` at each site.** If resp already holds an error on entry and the type has no Configure, today's code continues and this would return. Only direct callers do that, but it is still not equivalent. B2-10 uses `(diags, ok)`.
2. **Accumulating validator diagnostics locally, then one Append** (B2-14 variant). Append de-duplicates through `Diagnostic.Equal` against what is already there. Local accumulation changes which pairs are compared and in what order, so it is equivalent only if Equal is an equivalence relation, and providers can supply their own Diagnostic types. B2-14 keeps one Append per validator.
3. **`sort.Slice` to `slices.SortFunc` in NormaliseRequiresReplace.** Both are the same generated pdqsort with the same comparisons, so results are identical even with ties. One site, no simplification; not filed (area A's A14 covers sort.Slice sites elsewhere).
4. **`errors.Is` for the errorlint lead.** See §2. Neither a bug nor a modernisation.
5. **Sorting the GetMetadata type names** (`slices.Sorted(maps.Keys(funcs))`).
   - Today the order of the 7 metadata slices comes from map iteration, so it is random. The protocol treats them as sets, and the fwserver, proto5server and proto6server tests sort before comparing (7+8+8 `sort.Slice` calls; proto6server's test sorts ListResources twice, at lines 419 and 427).
   - Sorting narrows the behaviour and would let those test sorts go, but it is a behaviour change, not a refactor. Left as a suggestion.
   - The same map-order nondeterminism affects the order of GetProviderSchema diagnostics and of PlanResourceChange's 'Detected value change' traces. diag.Diagnostics documents its order as unreliable.
6. **Building ResourceBehaviors in the same pass as ResourceFuncs.** This saves a second instantiation and Metadata call per resource, and a second set of 'Found resource type' traces emitted in random order. It changes the provider call count and the log sequence. Rejected.
7. **Caching an empty ProviderTypeName.** It would change the log sequence that TestServerGetProviderSchema_logging pins ('Checking ProviderTypeName lock' and 'Calling provider defined Provider Metadata' once per registry). Rejected.
8. **Generic for MarkComputedNilsAsUnknown's 12 `AttributeWith<T>DefaultValue` cases.** fwschema has no common interface (different method names and return types), so it would need 12 closures. No gain.
9. **Using `list.ListResultsStreamDiagnostics` in ListResource (area F's F8 option) and simplifying the error-path iterator** (server_listresource.go:124-137, `if !push(x) { return }` as the last statement). Equivalent, but small (about -6 lines). Not filed; folds into B2-9's PR.
10. **MoveResourceState's diagnostics overwrite** looked like B2-3, but it is intended and test-pinned.
11. **Automatic ReadResource deferral** returns NewState=CurrentState with Private and NewIdentity nil, while manual deferral keeps both.
    - Core copies `resp.Private` and `resp.Identity` into the refreshed object (`$W/ext/node_resource_abstract_instance.go:809-810`).
    - But it does not write a deferred refresh to the refresh state (`node_resource_plan_instance.go:446`), so no persisted loss could be shown.
    - Not filed. The verifier could check refresh-only mode (`node_resource_plan_instance.go:~512` writes the working state).
12. **Unsynchronised writes** of s.deferred, *ConfigureData and StateStoreConfigureData in ConfigureProvider and ConfigureStateStore. The protocol orders those RPCs first. Not filed.
13. **Deleting `CreateResourceRequest.PlannedPrivate` versus wiring it into Create.** Wiring it in is a feature (resource.CreateRequest has no Private field; CreateResponse documents it as not pre-populated). B2-20 deletes it.

## 7. Cross-area hand-offs (verified here)

- **From A (notes-A.md:137): ListResource builds a missing list Config from the managed-resource schema.** Verified by probe `zz_probe_nil2_test.go`:
  - With ResourceSchema set, the provider's List receives `Config.Schema == the resource schema`.
  - With ResourceSchema nil and an identity schema set, the handler panics at `:144`.
  - Unreachable from Terraform: core always sends a config (`$W/ext/grpc_provider6.go:1362`).
  - Filed as part (4) of B2-6.
- **From A (notes-A.md:139): `UpgradeResourceIdentityRequest.TypeName` is dead.** Confirmed by fieldscan; kept as A18 and referenced from B2-20's notes.
- **From A (notes-A.md:138):** `InvokeActionResponse.SendProgress` (server_invokeaction.go:37-41) does an unconditional send on the consumer's channel. Confirmed as the root cause of A6. The fix belongs to A6 (a protoNserver drain); a fwserver-side alternative is a select on a done channel. Not refiled.
- **From F (notes-F.md:106, :109):** NoListResults is dead (B2-20) and ConfigureProvider dereferences a nil request (B2-6 part 1, probe-confirmed).
- **To F, for F4** (list resource for a framework resource without identity panics in NewListResult): Terraform core rejects this client-side before calling the provider. `$W/ext/grpc_provider6.go:1343` returns 'Identity schema not found for resource type %s; this is a bug in the provider'. So the panic is reachable only from non-Terraform clients.
- **To B1:** B2-18 keeps the `coerce<Type>Typable` names as function values, so B1's spec tables compile unchanged.

## 8. For the verifier

- **B2-1:** run `$W/repo/internal/fwserver/zz_probe_deferral_test.go`, then the same probe against `$W/drafts/fixes`.
- **B2-2:** core lines `$W/ext/node_resource_plan_instance.go:949-951` (identity check, after the deferral bookkeeping at :871-877) and `:903-927` (only an empty ImportedResources skips the checks).
- **B2-3:**
  - The fwserver probe lists 8 handlers. RenewEphemeralResource has the same `=` at :92 but was not in the probe.
  - The broader scope comes from the proto6server probe `zz_probe_metawarn_test.go`: the overwrite also drops warnings the protoNserver wrappers appended before calling (provider meta schema warnings on every ReadResource).
  - The fix draft changes 12 lines, including ApplyResourceChange :65/:93/:120, which repeat the overwrite one level up.
- **B2-4:** rerun `zz_probe_ctxchain_test.go` (about 6 s with logging on) and `zz_probe_ctxchain_eq_test.go` (log bytes equal).
- **B2-11:** the string-equality check script is inline in this session. Re-derive by diffing the string literals of each original Funcs method against the factoryRegistryText values in `$W/drafts/registry`. Also run proto5server/proto6server `TestServerGetProviderSchema_logging` on the draft.
- **B2-7:** confidence is medium only because of how likely the trigger is. The fwserver behaviour and the core check (`$W/ext/upgrade_resource_state.go:129-145`) are both confirmed.

## 9. Drafts and probes

- **Drafts** (each a full repo copy; diff with `git diff --no-index /home/user/terraform-plugin-framework/internal/fwserver <draft>/internal/fwserver`):

  | Draft | Item(s) |
  |---|---|
  | `$W/drafts/configure` | B2-10 |
  | `$W/drafts/registry` | B2-11 |
  | `$W/drafts/schemas` | B2-12 |
  | `$W/drafts/bulk` | B2-13 |
  | `$W/drafts/validate` | B2-14 |
  | `$W/drafts/attrtype` | B2-18 |
  | `$W/drafts/handlers` | Cumulative: `handlers_a` = B2-17; `handlers_b` = +B2-15; `handlers_c` = +B2-16; `handlers` = +B2-19. Diff consecutive copies to isolate one item. |
  | `$W/drafts/fixes` | Cumulative bug fixes: B2-1, B2-2, B2-3, B2-4, B2-5. All suites pass and the project lint reports 0 issues. |

- **Probes** (`$W/repo` is a full copy):
  - `$W/repo/internal/fwserver/zz_probe_{deferral,import_deferral,configure_warn,ctxchain,ctxchain_eq,misc,nilidentity,nil2}_test.go`
  - `$W/repo/internal/proto6server/zz_probe_caps_test.go` and `zz_probe_metawarn_test.go`
  - `zz_probe_misc_test.go` defines the `catch` helper that the nil probes use.

## 10. Commands that reproduce the counts

```sh
cd /home/user/terraform-plugin-framework/internal/fwserver
ls *.go | grep -v _test.go | grep -Ev '^(attribute_|block_|schema_)' | wc -l              # 50 files
ls *.go | grep -v _test.go | grep -Ev '^(attribute_|block_|schema_)' | xargs cat | wc -l # 7784 lines
grep -n -E 'WithConfigure\); ok \{' $(ls *.go | grep -v _test) | wc -l                    # 27 Configure preambles
grep -n 'resp.Diagnostics = ' *.go | grep -v _test                                         # 19 overwrites; 12 in B2-3 (9 handlers + ApplyResourceChange x3)
grep -n -E 'return s\.[a-zA-Z]+(Funcs|Behaviors), s\.[a-zA-Z]+Diags$' *.go | grep -v _test | wc -l  # 20 (B2-5)
grep -n -E 'Mutex\.(R?Lock|R?Unlock)\(\)' *.go | grep -v _test | grep -i schema            # schema cache sites (B2-12)
grep -c 'nullIdentityTfValue := tftypes.NewValue(req.IdentitySchema' server_*.go | grep -v ':0'  # 4 files, 5 sites; +2 nullTfValue sites (import, read) = 7 (B2-16)
grep -rn -B3 'ctx = logging.FrameworkWith' --include=*.go ../.. | grep -v _test            # 2 closures (B2-4)
grep -rn 'UpgradeResourceIdentity() method' --include=*.go ../..                           # 5 (B2-8)
grep -rn NoListResults --include=*.go ../.. | grep -v '/list/'                             # declaration only + list.NoListResults uses (B2-20)
for f in server_*_test.go; do printf "%s=%s " $f $(grep -c ConfigureMethod $f); done      # Configure test coverage (B2-10)
# configure block grouping: extract 18 lines from each site and md5sum (groups 9/6+1/4/3/2/1+1)
```
