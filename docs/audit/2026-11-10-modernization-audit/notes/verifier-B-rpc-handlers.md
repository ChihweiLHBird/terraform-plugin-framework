# V-B2 verification notes: fwserver RPC handlers, registries, Validate*Config twins, attr_type.go

Verifier scratch: `drafts/V-B2` (below: `$V`).
Author scratch: `.../audit/work/B2` (below: `$W`). Repository untouched (`git status --short` empty at 00badb1, checked at the end).

## Verdicts

| Id | Verdict | Conf. | Changes to the finding |
|---|---|---|---|
| B2-1 | CONFIRMED | high | impact: reach needs a client that allows deferrals |
| B2-2 | CONFIRMED | high | proof: core check also in released v1.12.0/v1.14.0; impact: deferral-capable client |
| B2-3 | CONFIRMED | high | risk: meta-schema warnings will then show on every apply/read RPC |
| B2-4 | CONFIRMED | high | why_equivalent: same multiset of log lines, not byte order; proof: no time gain without logging |
| B2-5 | CONFIRMED | high | notes: with B2-11, Clip is one line in load + the two ResourceBehaviors returns |
| B2-6 | CONFIRMED | high | adds a 6th provider-reachable panic: UpgradeResourceIdentity :181 |
| B2-7 | CONFIRMED | medium | size measured +11/-1; core check exists since v1.11.0 |
| B2-8 | CONFIRMED | high | none |
| B2-9 | CONFIRMED | high | none |
| B2-10 | CONFIRMED | high | none |
| B2-11 | CONFIRMED | high | proof: verifier runtime probe identical (242 lines) |
| B2-12 | CONFIRMED | high | none |
| B2-13 | CONFIRMED | high | risk note: method value evaluated before the error check (safe: struct receivers) |
| B2-14 | CONFIRMED | high | notes: coordinate with D-N1 (actions must pass true) |
| B2-15..B2-19 | CONFIRMED | high | none |
| B2-20 | CONFIRMED | high | where :42 not :40; 1 PlannedPrivate key not 2; size +8/-61 in 15 files (not -30/+0 in ~12) |
| B2-21 | CONFIRMED | high | none |
| B2-22 | CONFIRMED | high | notes: 4 more identical sites in toproto5/6 tests |
| D-N1 | CONFIRMED | high | tier 1; replacement must handle the validator-visible capability |
| A18 | confirmed | | `UpgradeResourceIdentityRequest.TypeName`: zero reads and writes, tests included (rename-and-compile) |

No item fell. I attacked each one with the checklist and with probes I wrote myself; what I found are corrections of numbers and scope, and one missed site (B2-6 (6)).

## Bugs

### B2-1: EnablePlanModification without ModifyPlan drops the provider deferral

Probe `$V/repo/internal/proto6server/zz_vb2_test.go` `TestVB2_B21`. My own resource types are used, with a schema plan modifier that sets `id`. The run goes through proto6server: ConfigureProvider with DeferralAllowed, then PlanResourceChange.

```
enable=true,modifyplan=no:  Deferred=<nil>                    id="set-by-schema-plan-modifier"
enable=true,modifyplan=yes: Deferred=&{PROVIDER_CONFIG_UNKNOWN} id="set-by-schema-plan-modifier"
enable=false,modifyplan=no: Deferred=&{PROVIDER_CONFIG_UNKNOWN} id=<null>
fixes draft (copy $V/repo-fixes): all three -> PROVIDER_CONFIG_UNKNOWN
```

- **Documented contract.** `provider/configure.go:88-96` says Deferred means "Terraform should automatically defer all resources and data sources". `resource/metadata.go:52-55` says "Framework will then automatically return a deferred response along with the modified plan."
- **The draft diff** moves the block out of the ModifyPlan `if` (13 lines added, 13 removed).
- **gocognit,** measured with `$W/gocog.yml` on a clean copy: 71 at HEAD, 69 on the fixes draft, 51 on handlers_a.
- **Reach.** Core's ConfigureProviderResponse carries only Diagnostics (`$W/ext/provider_iface.go:462`), so the per-RPC deferral from the framework is the only signal.

### B2-2: automatic import deferral omits the identity

Probe `TestVB2_B22` (proto6server):

```
auto deferral, identity import:   Deferred=PROVIDER_CONFIG_UNKNOWN Identity=<nil>
manual deferral, identity import: Deferred=ABSENT_PREREQ           Identity={"id":"abc"}
auto deferral, ID import:         Identity=<nil>       (unchanged on the fixes draft)
fixes draft: auto deferral, identity import -> Identity={"id":"abc"}
```

The core check `import of %s didn't return an identity` sits after the deferral bookkeeping in every version I checked:

| Version | Location |
|---|---|
| main | `$W/ext` :949-951 |
| v1.12.0 | :757-760 |
| v1.14.0 | :758-761 |

The release files were fetched to `$V/corefetch/npi-*.go`.

### B2-3: warnings collected before the provider call are overwritten

Probe `TestVB2_B23` (proto6server).

| Case | HEAD | fixes draft |
|---|---|---|
| ReadResource, resource Configure warning | 0 diags | 1 warning |
| PlanResourceChange (control) | 1 warning | 1 warning |
| ApplyResourceChange (create) | 0 diags | 1 warning |
| ReadResource x2, provider_meta schema warning | 0 diags each call | 1 warning each call |
| PlanResourceChange x2, provider_meta schema warning | 1 warning each call | 1 warning each call |

Recount of `grep -n 'resp.Diagnostics = ' *.go | grep -v _test` gives 19 hits:

- the 12 cited lines;
- configurestatestore:71 and generateresourceconfig:183;
- moveresourcestate:224 and :243;
- 3 seeded sites: planresourcechange:295 and :350, planaction:93.

Duplication check:

- Every overwritten response is a fresh struct. That includes ApplyResourceChange's `&CreateResourceResponse{}` and its siblings.
- The proto wrappers add nothing after the call.
- `Append` de-duplicates.

So no diagnostic can appear twice; on the draft each warning appears exactly once.

### B2-4: ctx reassignment in transform callbacks

The real functions were run in `$V/repo` (HEAD) and in `$V/repo-ctx` (the 2-line change only). Test file: `$V/zz_vb2_ctx_test.go.txt`, best of 3.

| Logging | n | Nullify HEAD | Nullify fixed | MarkComputed HEAD | MarkComputed fixed | Retained, closure alive (HEAD → fixed) |
|---|---|---|---|---|---|---|
| on | 1000 | 27 ms | 17 ms | 73 ms | 31 ms | |
| on | 2000 | 88 ms | 30 ms | 269 ms | 62 ms | |
| on | 4000 | 653 ms | 72 ms | 2.29 s | 105 ms | |
| on | 8000 | 4.16 s | 120 ms | 15.2 s | 178 ms | |
| off | 8000 | 91 ms | 102 ms | 103 ms | 114 ms | 25,634 KB → about 0 KB |

Log equivalence was captured as tfsdklogtest JSON, 3 runs per tree:

- **Identical:** `sort | md5sum` is the same for all 6 runs (`dba69a3f...`), and the transformed values (first 4 lines) are the same in all 6 runs.
- **Not identical:** the raw files differ between two runs of the same tree (cmp: line 5). tftypes v0.31.0 `transform` ranges over `map[string]Value` for maps and objects (walk.go:263, :295), so the line order is random in both variants.

Correction: "byte-identical logs" should read "identical sorted lines and values". No time gain is measurable without logging; the gain there is memory.

Repo-wide check, `grep -rn --include=*.go -E '^\s+ctx = ' . | grep -v _test`:

- 2 closures;
- 5 function-scope parameter reassignments;
- `logging/context.go` and `logging/framework.go`;
- proto6server/server_writestatebytes.go:85-87, which runs once, after the loop.

### B2-5: cached registry diagnostics are aliased

Probe `$V/repo/internal/fwserver/zz_vb2_alias_test.go`.

```
cached len=3 cap=4
d1 last detail before second lookup: No data source type named "first" ...
d1 last detail after  second lookup: No data source type named "second" ...
VB2_RACE=1 go test -race: WARNING: DATA RACE (x2), test FAIL
fixes draft (Clip): cap=3, d1 unchanged, -race ok
```

The 20 return sites were recounted with the author's grep. ResourceBehaviors (:600, :643) is outside B2-11's generic, so the composed fix needs Clip there too.

### B2-6: nil dereferences (one more site found)

Probes:

- `$V/repo/internal/fwserver/zz_vb2_nil_test.go`: cases (1) to (4) all PANIC. Case (4b), with ResourceSchema set, shows the provider's List receiving a `schema.Schema` (the resource schema).
- proto6server `TestVB2_B26_5`: ReadResource, Apply create and Apply update panic at server_readresource.go:202, server_createresource.go:173 and server_updateresource.go:203 (from the stack traces).
- **New site, proto6server `zz_vb2_upgid_test.go`:** UpgradeResourceIdentity with an IdentityUpgrader that sets `resp.Identity = nil` PANICs at server_upgraderesourceidentity.go:181. The control, which leaves Identity set without Raw, returns 'Missing Upgraded Resource Identity' as designed.

Other identity dereferences were swept. They are guarded, or the request side is pre-populated whenever IdentitySchema != nil (PlanResourceChange :374/:386, ReadResource :188, UpdateResource :189, ImportResourceState :218).

terraform-plugin-go v0.31.0 tf6server has no `recover()` (grep), so these panics kill the provider process.

### B2-7: same-version UpgradeResourceState returns write-only values

Probe `$V/repo/internal/proto6server/zz_vb2_upg_test.go`.

```
schema version 0, state version 0: password="stored-secret"   (passthrough)
schema version 1, state version 0: password=<null>            (upgrader path)
verifier draft $V/repo-b27 (+11/-1): passthrough -> <null>; fwserver/proto5server/proto6server ok
```

Docs, fetched from developer.hashicorp.com/terraform/plugin/framework/resources/write-only-arguments:

- "the plugin framework will handle the nullification of write-only arguments for all RPCs";
- the same page recommends a separate write-only attribute (PreferWriteOnlyAttribute). That is why the trigger is uncommon, and why confidence stays medium.

Core: upgrade_resource_state.go in v1.11.0 (:126-130) and v1.14.0 (:125-129) rejects non-null write-only values, and calls the provider for same-version states (":75/:74 ... minor fixups").

### B2-8, B2-9

Re-grepped. 5 hits for 'UpgradeResourceIdentity() method'. The interface method is `UpgradeIdentity` (resource/resource.go:239). The pinned 'Found resource type' lines are ResourceFuncs' messages (`tf_resource_type`), not the list message.

### D-N1 (from area D): ValidateActionConfig rejects write-only action attributes

Probe `$V/repo/internal/proto6server/zz_vb2_dn1_test.go`.

```
GetProviderSchema action attr wo write_only=true, plain write_only=false, 0 diags
ValidateActionConfig wo="secret":  ERROR "WriteOnly Attribute Not Allowed" "The resource contains a non-null value for WriteOnly attribute wo. Write-only attributes are only supported in Terraform 1.11 and later."
ValidateActionConfig wo=<unknown>: same ERROR
ValidateActionConfig wo=<null>:    0 diags
copy $V/repo-dn1 (true): 0 diags for all three; fwserver/proto5server/proto6server ok
```

**Evidence that `false` is deliberate: none found.**

- git blame shows cca5c5aa (2025-07-28) for lines 100-106. The comment predates #1233 (e0355d6), which touched only `action/schema/*` (`git show --stat`).
- tfprotov6.ValidateActionConfigRequest has no ClientCapabilities.
- None of the 3 action validate test files contains WriteOnly.
- Core sends the unmarked config:
  - main node_action.go:211-219 (fetched to `$V/corefetch`);
  - v1.14.0 node_action_validate.go:105-116.
- In both versions this follows `validateResourceForbiddenEphemeralValues`, which allows ephemeral values only in write-only attributes.

**Side effect of the one-line fix.** Probe `$V/zz_vb2_dn1cap_test.go.txt`: an action attribute validator sees `req.ClientCapabilities.WriteOnlyAttributesAllowed` = `[false]` at HEAD and `[true]` with the fix. `schema/validator/client_capabilities.go:10-16` documents the field as "only populated during managed resource schema validation". The fix must therefore update that public doc and the changelog, or bypass the check with an internal flag.

Tier 1 is endorsed: a released feature is unusable, the fix is one line, and the proof is strong.

## Refactors

### Draft sizes (re-measured)

Commands: `git diff --no-index --stat /home/user/terraform-plugin-framework/internal <draft>/internal`, and diffs between consecutive handler drafts.

| Item | Draft | Measured |
|---|---|---|
| B2-10 | configure | 27 files +193/-339 |
| B2-11 | registry | 7 files +257/-423 (net -166) |
| B2-12 | schemas | 7 files +58/-141 |
| B2-13 | bulk | 6 files +146/-300 |
| B2-14 | validate | 8 files +93/-263 |
| B2-17 | handlers_a | +49/-42 |
| B2-15 | handlers_a → handlers_b | 4 files +36/-103 |
| B2-16 | handlers_b → handlers_c | 6 files +36/-99 |
| B2-19 | handlers_c → handlers | 1 file +18/-30 |
| B2-18 | attrtype | 1 file +22/-169 |

### Tests and lint on the drafts

`go test -count=1 ./internal/fwserver/ ./internal/proto5server/ ./internal/proto6server/ ./providerserver/` reports 4 ok each for configure, registry, schemas, bulk, validate, attrtype, handlers_a, handlers (cumulative) and fixes.

`$AUDIT/bin/golangci-lint run --config /home/user/terraform-plugin-framework/.golangci.yml ./internal/fwserver/...` was run with a FRESH cache (`GOLANGCI_LINT_CACHE=$V/lintcache`; the warm-cache run took 0.4 s, so the cache was cleared). It reports 0 issues on all 10 drafts.

### B2-10 (Configure preambles)

- `$V/cfg/*.txt` holds 18 lines per site, grouped by md5:
  - 9 resource;
  - 6 state store;
  - 4 ephemeral;
  - 3 action;
  - 2 data source;
  - 1 list-validate;
  - validatestatestoreconfig, unique only by `statestoreWithConfigure`;
  - ListResource, unique (Called/Called text, ListResult streaming).
- The 26 draft call sites are uniform.
- The helper strings and ProviderData expressions are identical to the originals.
- Only the `@caller` location of the moved traces changes, which is inherent to any extraction.
- Configure coverage: 20 test files have a type-level ConfigureMethod; the 6 Validate*Config tests have 0.

### B2-11 (registry generic)

Runtime probe `$V/zz_vb2_registry_test.go.txt`, run in `$V/repo` and in `$V/repo-registry` (the draft's fwserver files over a clean copy):

- **Providers:**
  - a full provider with names [one, "", two, one, three, two] for all 7 concepts, plus a list resource with no matching managed resource;
  - `testprovider.Provider{}`;
  - a bare provider without any optional interface.
- **Rounds:** two per provider (cached path).
- **Recorded:** map keys, nil-ness of diags, Severity/Summary/Detail of every diagnostic, and the full JSON log.

Result: 242 lines, `cmp` IDENTICAL. The run contains 36 'Found ...' traces, 17 'Checking ProviderTypeName lock' lines and all 21 registry diagnostics. TestServerGetProviderSchema_logging passes in proto5server and proto6server.

### B2-13 (lookups, metadata, schemas)

Attack: `schemaResp.Schema.ValidateImplementation` is now a method value evaluated before the HasError check. It is safe only because all 6 SchemaResponse.Schema types are structs with value receivers (checked `^type Schema struct` and `func (s Schema) ValidateImplementation` in the 6 schema packages). The 5 Not Found summary/detail pairs match the originals.

### B2-14 (Validate*Config twins)

The 7 trace names were compared with the originals. The resource tail keeps `req.ClientCapabilities.WriteOnlyAttributesAllowed`. With D-N1 applied, actions must pass true.

### B2-15 and B2-16

- **B2-15 precondition:** no error in resp at the 4 sites (create :158, update :173, read :172, datasource :110 return on errors).
- **B2-16:** tftypes `Value.Copy` (value.go:234-251) of a null value is `NewValue(val.Type(), nil)`.

### B2-17

gocognit 71 → 51 and 359 → 318 lines. Verbatim move.

### B2-18 (coerce generic)

- The 12 functions normalise to 1 body (Python).
- 27 references: 19 in attribute_plan_modification.go and 8 in block_plan_modification.go.
- The error branch is untested. Probe `$V/zz_vb2_coerce_test.go.txt` gives IDENTICAL `%#v` diagnostics before and after, for a bad BoolValuable (value receiver) and a bad ListValuable (pointer).

### B2-19

The single `var diags` at :59 is not shadowed, the callback does not reassign ctx, and `err` is passed from the correct scope.

## Cleanup and tests

### B2-20 and A18 (dead fields)

Copy `$V/repo-dead`:

1. Renamed the fields and NoListResults, and A18's TypeName.
2. Compiled all 8 importer packages with tests: `go test -gcflags=-e -run NONE`, plus `go build`.

Findings:

- **Non-test writes:**
  - `server_applyresourcechange.go:53` (PlannedPrivate, the only key);
  - fromproto5/6 `closeephemeralresource.go:42` and `renewephemeralresource.go:42`.
- **Test keys:** 17 (fwserver close 5, renew 8, fromproto5/6 close and renew 1 each).
- **References to NoListResults and to A18's TypeName:** zero.
- **Collateral compile errors:**
  - `internal/fwschema` becomes unused in server_close/renewephemeralresource.go;
  - `testSchema` and the ephemeral/schema import become unused in the two fwserver tests.

After fixing those, gofmt, vet and the 5 package tests are clean. The measured diff is +8/-61 in 15 files, and +8/-64 in 16 with A18. The two server.go comment fixes are verified:

- `list.ConfigureRequest` exists but is unused by the framework (`ListResourceWithConfigure.Configure` takes `resource.ConfigureRequest`);
- `s.deferred` is also checked by OpenEphemeralResource and PlanAction.

### B2-21 (ListResource test comparer)

On copy `$V/repo-b221`:

| Comparer | Expectations | Result |
|---|---|---|
| removed | unchanged | ok |
| removed | Detail at :210 mutated | FAIL |
| original | Detail at :210 mutated | ok (masked) |
| original | extra expected diagnostic | panic "non-deterministic or non-symmetric function detected" |

### B2-22

Both sites confirmed. The same pattern appears at internal/toproto5|6 openephemeralresource_test.go:93 and renewephemeralresource_test.go:43 (outside B2).

## Checks on the author's notes

- **§2, errorlint lead rejection.** Correct: attribute_path.go:364-366 returns the stepper error unwrapped; Value.ApplyTerraform5AttributePathStep returns the bare sentinel; diff.go:97/:117 compares with `!=`.
- **§6.5 has a factual slip.** The GetMetadata test sorts are 7+7+8, not 7+8+8. proto5server *also* sorts ListResources twice (:364 and :372), as proto6server does (:419 and :427), and has no StateStores sort. This is not filed as an item; the duplicate `sort.Slice` is a 4-line test cleanup in proto5/6server (other area).
- **§6.6 is correct.** ResourceBehaviors walks the ResourceFuncs map, calls Metadata again and logs 'Found resource type' (server.go:594-645).
- **§6.7 is correct.** ProviderTypeName caches only non-empty names; 6 + 6 pinned lines in proto6server's logging test.
- **§6.10 is correct.** Test 'response-Diagnostics-first-error-always-responds' (server_moveresourcestate_test.go:612).
- **§6.11, refresh-only.** In refresh-only mode (`skipPlanChanges`), core writes the refresh result into the WORKING state even when deferred (`$W/ext/node_resource_plan_instance.go:504`). It skips refreshState when deferred (:445-447). The nil Private and Identity of the automatic ReadResource deferral therefore reach the working state. I could not show that they are persisted, and identity and private state are not referenceable in configuration, so leaving it unfiled is reasonable.
- **§8, B2-3.** RenewEphemeralResource :92 is in the 12 lines; confirmed.

## Probe and tree index

| Path | Contents |
|---|---|
| `$V/repo` | HEAD copy plus the probes `internal/proto6server/zz_vb2_test.go`, `zz_vb2_dn1_test.go`, `zz_vb2_upg_test.go`, `zz_vb2_upgid_test.go`, `zz_vb2_dn1cap_test.go`, and `internal/fwserver/zz_vb2_alias_test.go`, `zz_vb2_nil_test.go`, `zz_vb2_ctx_test.go`, `zz_vb2_registry_test.go` |
| `$V/repo-clean` | Pristine HEAD copy (lint and gocognit baseline) |
| `$V/repo-fixes` | HEAD plus the author's fixes draft fwserver files, plus my probes |
| `$V/repo-ctx` | HEAD plus the B2-4 2-line change only |
| `$V/repo-dn1` | HEAD plus the D-N1 one-line fix |
| `$V/repo-registry` | HEAD plus the B2-11 draft files |
| `$V/repo-attrtype` | HEAD plus the B2-18 draft file |
| `$V/repo-dead` | B2-20 + A18 deletion, gofmt'd |
| `$V/repo-b27` | B2-7 fix draft |
| `$V/repo-b221` | B2-21 comparer experiments |
| `$V/corefetch` | Terraform core files fetched read-only: main node_action.go and node_action_validate.go; v1.12.0 and v1.14.0 node_resource_plan_instance.go; v1.11.0 and v1.14.0 upgrade_resource_state.go; v1.14.0 node_action_validate.go |
| `$V/docsfetch/wo.html` | Framework write-only docs page |
| `$V/out/build_json.py` | Builder for verified-B2.json |
