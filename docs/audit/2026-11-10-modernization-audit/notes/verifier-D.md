# Verifier notes, area D (public schema packages)

Verifier V-D. Repository untouched: `git status --porcelain` printed 0 lines after all overlay runs.
All probes are under `drafts/V-D` (written `$V` below). Every
probe is a `zz_*_test.go` added through `go test -overlay`, or a standalone module (`$V/neq`).
External sources were read only from the public Go module proxy into a private module cache (`$V/modcache`):
terraform-plugin-framework v1.15.0, hashicorp/terraform v1.14.0 and v1.17.1, zclconf/go-cty v1.18.1.

## Verdicts

| Item | Verdict | Conf. | Change vs author |
|---|---|---|---|
| D1 | CONFIRMED | high | replacement has a hole (see below); corrected replacement tested |
| D2 | CONFIRMED | high | confidence medium -> high (end-to-end + v1.15.0 evidence) |
| D3 | CONFIRMED | high | impact narrowed; line refs :182/:44; internal fix drafted (-4/+16) |
| D4 | CONFIRMED | medium | impact: latent because of D-N1; import-group nit |
| D5 | CONFIRMED | medium | PLAUSIBLE -> CONFIRMED (Terraform core + go-cty sources) |
| D6 | CONFIRMED | high | none |
| D7 | CONFIRMED | high | +1 defect the draft misses (defaults/doc.go:32); size -65/+67 |
| D8 | CONFIRMED | high | none |
| D9 | CONFIRMED | high | size re-measured exactly with an independent script |
| D10 | CONFIRMED | high | 34 files, not 33 (43 call sites in 31 files + 3 definition-only files) |
| D11 | CONFIRMED | high | none; order question answered (random today, unchanged) |
| D-N1 (new) | CONFIRMED | high | tier 1 bug: ValidateActionConfig rejects all configured write-only action attributes |

## D1: ObjectAttribute default with CustomType + AttributeTypes

Probe `$V/d1/zz_vd1_test.go`, 8 cases, run with three overlays:
`go test -vet=off -count=1 -overlay=$V/d1/overlay-{before,after,alt}.json -run TestZZVD1 -v ./resource/schema/`

| case | today | author draft | corrected variant |
|---|---|---|---|
| 1 strict custom(a) + AttrTypes(a) + default(a) [valid] | **1** (identical type strings) | 0 | 0 |
| 2 strict custom(a), no AttrTypes, default(a) [valid] | 0 | 0 | 0 |
| 3 no custom + AttrTypes(a) + default(a) [valid] | 0 | 0 | 0 |
| 4 no custom + AttrTypes(a) + default(b) [mismatch] | 1 | 1 | 1 |
| 5 strict custom(a) + AttrTypes(a) + default(b) [mismatch] | 1 | 1 | 1 |
| 6 strict custom(b) + AttrTypes(a) + default(a) [mismatch vs effective type] | 1 | **0** | 1 |
| 7 loose custom(a) (no Equal override) + AttrTypes(a) + default(a) [valid] | 0 | 0 | 0 |
| 8 loose custom(b) + AttrTypes(a) + default(a) [mismatch vs effective type] | 1 | **0 (regression)** | 1 |

Through `Schema.ValidateImplementation` (what fwserver.go:789 runs) case 1 gives haserror=true.
The corrected variant (`$V/d1/object_attribute.go`) keeps the author's AttributeTypes comparison, returns after a
mismatch, and when CustomType != nil also requires `a.CustomType.TerraformType(ctx).Equal(actualType.TerraformType(ctx))`.
`go test ./resource/schema/` passes with both overlays.
At runtime the default value is placed into the plan with `resp.PlanValue.ToTerraformValue(ctx)`
(internal/fwschemadata/data_default.go:320), so the Terraform type is what has to match.

## D2: dynamic UseNonNullStateForUnknown

The md5 of the normalised PlanModify body is identical across all 12 packages, dynamic included (stamped in 18140ec).
End to end through `fwserver.PlanResourceChange` (`$V/d2/zz_vd2_test.go`, overlay-before/after):

```
BEFORE typed-null prior (string null)   planned UseNonNull=tftypes.String<null>
BEFORE untyped null prior               planned UseNonNull=tftypes.DynamicPseudoType<unknown>
AFTER  typed-null prior                 planned UseNonNull=tftypes.DynamicPseudoType<unknown>
AFTER  non-null prior "keep"            planned UseNonNull=tftypes.String<"keep">   (unchanged)
```

v1.15.0 `dynamicplanmodifier.UseStateForUnknown` (the documented "pre-1.15.1 behavior" that the #1242 release note
says UseNonNullStateForUnknown provides) is code-identical to the author's draft once comments are stripped (`diff` empty).

## D3: single nested object validators run twice

My probe `$V/d3/zz_vd3_test.go` goes through `fwserver.ValidateResourceConfig`. Calls per validation:

| | known | null | unknown |
|---|---|---|---|
| attr, validator returns nothing/warning | 2 | 1 | 1 |
| attr, validator returns error | 1 | 1 | 1 |
| block, validator returns nothing/warning | 2 | 2 | 2 |
| block, validator returns error | 1 | 1 | 1 |

Visible diagnostics: a warning validator gives 1 (deduplicated), a non-deterministic counter warning gives 2.
Error validators run once because the single-nesting branches test the accumulated `resp.Diagnostics.HasError()`
(attribute_validation.go:1134-1137, block_validation.go:140-142). That early return also skips all child validation
(child validator calls=0); it is area B1 behaviour and out of scope here.
The author's probe reproduces 2/1/2.

Fix drafted (`$V/d3/fix.diff`, -4/+16): unexported `nestedAttributeObjectValidate`/`nestedBlockObjectValidate` take an
`objectValidators bool`; the exported functions pass true, and the two single-nesting branches pass false. With
`$V/d3/overlay-fix.json` all 24 cases give 1 call, child calls and diagnostics are unchanged (the counter case drops
to 1 diagnostic), and `go test ./internal/fwserver/` passes. Removing the attribute-level run instead would lose
validators for null/unknown single nested attributes (the branch returns at :1139).

## D4/D5: action write-only

Probe `$V/d45/zz_vd45_test.go` (overlay, then overlay-after with the author's draft):

```
action Single WO + non-WO child   0  -> 1
action List   WO + non-WO child   1
action Map    WO + non-WO child   1
action Set nested attr + WO child 0  -> 1
action Set nested block + WO child 0 -> 1
resource Single / Set attr / Set block: 1 / 1 / 1
```

Draft: `go test ./action/schema/`, `go vet` and gofmt are all clean. The draft's import block puts "context" in its own
group (nit).

Terraform core (v1.14.0 and v1.17.1):
- `internal/schemarepo/loadschemas/plugins.go` runs `InternalValidate()` for resource types (+identity), data sources,
  ephemeral and list resources only, not for actions.
- `internal/configs/configschema/internal_validate.go`: "NestingSet blocks/attributes may not contain WriteOnly
  attributes ... any marks within sets will be hoisted up the outer set value, so only the set itself can be WriteOnly"
  (v1.17.1 :100-104, :182-186).
- Action config validation calls `validateResourceForbiddenEphemeralValues` (v1.17.1 node_action.go:205; v1.14.0
  node_action_validate.go:105). It walks up to the nearest write-only attribute; if it finds none it reports
  "not a write-only attribute", or "not an assignable attribute" for blocks.
- go-cty v1.18.1 `SetVal` (value_init.go:264) deep-unmarks the elements and re-marks the set, which is the hoisting.

So for D5, a write-only child inside a set can never receive an ephemeral value (the set is never write-only in the
framework). For D4, core itself would accept ephemeral values in non-write-only children of a write-only single
nested attribute, because it walks up to the ancestor. The rule is a framework convention, which the author's
"medium" already reflects.

## D-N1 (new): ValidateActionConfig rejects every configured write-only action attribute

```
fwserver:    VACTWO value="secret" diags=1 | WriteOnly Attribute Not Allowed: The resource contains a non-null value for
             WriteOnly attribute wo. Write-only attributes are only supported in Terraform 1.11 and later.
             VACTWO value=<null> diags=0
proto6server: Required+WriteOnly attr: "hunter2" -> ERROR, <unknown> -> ERROR; GetProviderSchema write_only=true, 0 diags
with fix (WriteOnlyAttributesAllowed: true): 0 diags for both; go test ./internal/fwserver/ ./internal/proto5server/
             ./internal/proto6server/ all ok
```

Cause: server_validateactionconfig.go:100-106 copied `WriteOnlyAttributesAllowed: false` from the other non-resource
RPCs (#1188). #1233 (e0355d6, released 1.17.0) added `WriteOnly` to action schemas without touching fwserver.
tfprotov6.ValidateActionConfigRequest has no client capabilities. Core sends the real unmarked values to
ValidateActionConfig (v1.14.0 node_action_validate.go:108-114). No existing test covers it (0 `WriteOnly` in the three
action validate test files). The issue owner is area B1 (internal/fwserver); none of the other findings mention it.

## D6
Probe `$V/d6/zz_vd6_test.go`: the 4 cases are identical before and after the draft. With the outer check removed
instead (`$V/d6/sna_outer_removed.go`), case 2 reorders to [Invalid Schema Implementation, Schema Using Attribute
Default...], which confirms that the inner block is the right one to delete.

## D7
All 43 cited defects opened (not sampled): 1 + 9 + 8 + 9 + 6 + 1 + 1 + 5 + 2 + 1. The staticcheck ST1020 hits for
area D are exactly 9 (AttributeAtTerraformPath) + 6 (GetBlocks) + 1 (IsSensitive). The draft diff has 0 non-comment
changed lines and `go build -overlay` passes. Missed: `resource/schema/defaults/doc.go:32` "desired custom type in a
plan modifier." (defaults package). The other 8 concept doc.go files and the 24 typed package docs are fine.
schema/doc.go is incomplete rather than wrong.

## D8
Sweep: for each `func (x T) ValidateImplementation` in the 9 packages, is `fwschema.(Attribute|Block)WithValidateImplementation = T{}`
in the file's var block? Only provider/schema MapNestedAttribute is missing it; the Schema types use another signature.
An overlay test with the assertion compiles.

## D9
Independent rewrite `$V/d9/rewrite.py` (classifies whole bodies, 0 unmatched): attr 100, nested 31, block 17,
148 files, -802/+148, 36-line helper, net -618, which is exact. Changed-line classes are only body lines. Vet is clean;
the 10 packages pass. Equal matrix probe `$V/d9/zz_vd9_test.go`: hash 9eb7a1c8fc26cfac with 26 trues both before and
after; Bool.Equal(String)=false; wrapper/Bool asymmetry preserved.

## D10
15 definitions; 43 call sites in 31 files, plus 3 definition-only files, gives 34 files. The diff has 35 `+++` headers,
including the new maps.go. -208/+43 plus a 27-line helper. Nil probe `$V/d10/zz_vd10_test.go` is identical before and after.

## D11
4 code variants confirmed; lines :152/:131; 123 methods, 54 forwarding. Order probe `$V/d11/zz_vd11_test.go`:
3 distinct orders in 200 runs both before and after (map order is random today), and per-entry order is preserved.
Overlay tests pass including internal/fwserver.

## Looks equivalent but is not (endorsed, standalone module `$V/neq`)
```
vet: user1/main.go:7:32: unknown field Required in struct literal of type lib.BoolAttributeEmbedded
NumField today 3 helper 4
%T alias: lib.primitive[bool]
cmp.Diff on helper struct panics: true
GetType today: custom
GetType eager panics: true
```
Also endorsed from my own checks: D6 outer-removal reorders diagnostics; for D3, dropping the attribute-level run
loses null/unknown validation; for D1, comparing only against AttributeTypes (or guarding with CustomType == nil)
loses correct rejections.

## Commands (from /home/user/terraform-plugin-framework)
```
GOTOOLCHAIN=auto go test -vet=off -count=1 -overlay=$V/<dir>/<overlay>.json -run TestZZ<Name> -v ./<pkg>/
GOMODCACHE=$V/modcache GOFLAGS=-modcacherw go mod download -json github.com/hashicorp/terraform@v1.17.1   (from $V/fetch)
```
