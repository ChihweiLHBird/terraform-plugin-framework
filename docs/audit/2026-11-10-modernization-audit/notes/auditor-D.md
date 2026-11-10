# Area D notes: public schema package families

Auditor D. Repository untouched (`git status --porcelain` = 0 lines, checked after every overlay run).
All scratch work is under `drafts/D`
(written `$W` below). Drafts were compiled and tested with `go test -overlay=<json>` on single packages, which
reads replacement files from scratch without writing to the repo (the overlay can also add new files, which is how
the new `internal/fwschema` / `internal/fwplanmodifier` files and the probe tests were tested).

## 0. Method and coverage

| What | How | Exhaustive? |
|---|---|---|
| Same-package typed attribute twins | AST comparer `$W/astcmp` (normalises the type token in identifiers, optional comment stripping; per-declaration and per-field) | yes, resource/schema all families; spot checks elsewhere |
| Cross-package siblings (9 packages) | astcmp per file name across packages; getter matrix (1613 single-statement getters, `$W/getters.tsv`); method-set matrix (`$W/methods.tsv`); interface-assertion matrix; ValidateImplementation bodies per file | yes |
| 12 typed planmodifier + 12 default packages | astcmp per file across the 12 types, with and without docs | yes |
| schema.go (9) | astcmp + classification of all 123 `Schema` methods (`$W/schema_methods.tsv`) | yes |
| perfsprint (663) | every line read by script (`$W/perfsprint_lines.txt`) | yes |
| dupl (364), funlen (36), godox (2), dupword (1), any (148+148) | classified by script, sample read | yes (classification) |
| Coverage | `go test -coverprofile` per schema package (9 packages, one at a time) | yes |
| Dead code | coordinator deadcode.txt (0 hits in area D) + reference count of every unexported package-level func | yes |

## 1. Q1: shared unexported implementation for the attribute structs

### Measured duplication within resource/schema (code only, type token normalised)
- 8 primitive attribute files (bool, float32, float64, int32, int64, number, string, dynamic): 33 of 33 declarations
  identical (imports, var block, all 13 fields with docs, all 19 methods). With docs, the only differences are the
  struct doc (8 distinct, expected) and one doc bug (Bool `GetType` says `types.StringType`, item D7).
- list/map/set attributes: 31 of 34 identical; differences are Set's missing `WriteOnly` field and its `IsWriteOnly`
  returning false (deliberate: write-only unsupported in sets).
- list/map/set nested attributes: 31 of 36 identical; differences: step type in `ApplyTerraform5AttributePathStep`
  (ElementKeyInt/String/Value), Set write-only handling.
- list/set nested blocks: 20 of 22 identical; differences: step type, set's write-only rejection.
- Across the 9 packages every difference lines up with a per-concept capability (Computed, Sensitive, WriteOnly,
  Validators, PlanModifiers, Default, import flags, identity's single Description). The coordinator's "9 distinct
  bodies for 9 packages" is confirmed and fully explained.
- Size context: the 9 packages are 35,604 non-test lines = 16,843 comment + 5,158 blank + 13,603 code
  (`for f in ...; grep -cE '^\s*//'`). Godoc is 47% and must stay per type.

### What can be shared without any public change (measured, all tests pass via overlay)
| Shape | Sites | Measured net | Item |
|---|---|---|---|
| `Equal` bodies (type assertion + fwschema.*Equal) | 100 attr + 31 nested + 17 block = 148 | -802/+148 +36 helper = -618 | D9 |
| `schemaAttributes`/`schemaBlocks` copies | 9 + 6 = 15 | -208/+43 +27 = -138 | D10 |
| `Schema.ValidateImplementation` loops | 9 | -194/+14 +47 = -133 | D11 |
| One-line getters (`return a.Field`) | about 1,300 | not simpler: a helper call is no shorter than `return a.Field` | rejected |
| `GetType` bodies | 131 in 5 shapes (68 primitive) | 68 x -4 = about -272 | rejected, see below |

So the honest answer is slightly better than "only one-line getters": the 5-6 line `Equal` bodies and the schema.go
loops can call generic helpers. Even with all three, the reduction is about 890 lines of 13,603 code lines (6.5%);
everything else (struct definitions, field docs, per-type method names) must stay per type.

### "Looks equivalent but is not" (rejected; proofs in `$W/probe3`)
1. **Embed an unexported `commonAttribute` struct holding Required/Optional/Computed/Sensitive/Description/...** -
   keyed composite literals stop compiling: Go does not allow promoted fields in struct literals
   (`vet: user2/main.go:5:35: unknown field Required in struct literal of type lib.BoolAttributeEmbedded`). Every
   provider writes `schema.BoolAttribute{Required: true, ...}`, so this breaks every provider, not only positional
   literals. It also changes `reflect.Type.NumField` (3 -> 2 in the probe) and makes bare `cmp.Diff` panic on the
   unexported embedded field.
2. **Keep every exported field and add an unexported embedded helper struct** (the coordinator's "breaks positional
   literals only" variant) - also not equivalent: `NumField` changes (3 -> 4), and bare `cmp.Diff` panics
   ("cannot handle unexported field"). The repo's own tests do bare `cmp.Diff` on these structs, e.g.
   `TestSchemaAttributeAtPath` (resource/schema/schema_test.go:122, expected `schema.StringAttribute{}`), and so do
   provider authors' schema tests. And it buys nothing: methods promoted from an embedded struct receive the embedded
   value and cannot read the outer struct's fields, so it cannot host the getters.
3. **Generic type alias** (`type BoolAttribute = primitiveAttribute[basetypes.BoolTypable, validator.Bool, ...]`) -
   impossible: a generic type cannot carry per-instantiation method names (`BoolValidators`, `BoolDefaultValue`,
   `BoolPlanModifiers`), which the fwxschema/fwschema interfaces require; `%T`/`reflect.Type.Name()` and godoc would
   also change.
4. **`GetType` via a value-taking helper** (`return customTypeOr(a.CustomType, <default type>)`) - equivalent only for
   the 68 primitive bodies (nil interface converts to nil `attr.Type`; typed-nil custom types stay non-nil in both).
   For the 63 others the default type must stay lazy: `SingleNestedAttribute.GetType` builds a map and calls
   `GetType()` on every child, which panics on a nil child entry; today that code is skipped when CustomType is set,
   but an eager argument would always run it. Saving 4 lines in 68 places is not worth a helper. Rejected.

## 2. Q2: the 12 typed planmodifier packages and 12 typed default packages

- Size: planmodifier family 3,363 lines (1,131 comment, 612 blank, 1,620 code); default family 565 lines (156/108/301).
- Normalised with docs, all 6 planmodifier files are identical across the 12 types except: dynamic
  `RequiresReplaceIfConfigured` (also checks `IsUnderlyingValueNull`), dynamic `UseStateForUnknown` (underlying
  unknown checks), and `ctx` vs `_` parameter name in string's `UseStateForUnknown`. `requires_replace_if.go` (73 lines)
  x12: 10 of 10 declarations identical code-only. All 12 copy diagnostics (`resp.Diagnostics.Append(ifFuncResp.Diagnostics...)`).
- `resp.RequiresReplace = ifFuncResp.RequiresReplace` looks like it could unset a prior replacement, contrary to the
  doc "Returning false will not unset any prior resource replacement", but fwserver gives every modifier a fresh
  response and accumulates with `resp.RequiresReplace.Append` (internal/fwserver/attribute_plan_modification.go:2391-2425),
  so it cannot. Not a bug.
- **Draft generic implementation, measured and REJECTED.** `$W/drafts/D-pm-all` adds `internal/fwplanmodifier`
  with `RequiresReplaceIfApplies`, `UseStateForUnknownApplies`, `UseNonNullStateForUnknownApplies` (64 lines) and
  rewrites the guard blocks of 34 files (11 static types x 3 files + dynamic requires_replace_if): -474/+90, net
  **-320 lines**; all 12 packages pass via overlay (`$W/diffs/D-pm-all.diff`). Equivalent (values are boxed into
  `attr.Value` and the same methods run; `Equal` still runs only after both null checks). Rejected because: (a) each
  package still needs its own exported funcs, `RequiresReplaceIfFunc`/`RequiresReplaceIfFuncResponse` types, docs and
  a struct with the type-specific `PlanModify<Type>` method (a generic type cannot provide per-type method names, and
  methods cannot be declared on another package's type), so only guard clauses move; (b) these packages are the
  reference implementations provider authors read and copy, and the move hides the commented guard logic in an
  internal package they cannot import; (c) dynamic already diverges and the D2 fix diverges it further, which is the
  same template-sharing that caused D2.
- Default packages: per type there is a constructor, a struct, Description/MarkdownDescription with type-specific
  verbs (`%t`, `%d`, `%f`, `%s`, `%v`) and a `Default<Type>` method. Nothing can be shared without changing the
  description text. No proposal. (float32/float64 use `%f`, so `StaticFloat64(1e-7)` describes as `0.000000`;
  cosmetic, and changing it would change observable Description text, so not reported.)

## 3. Q3: schema.go in the 9 packages
- 123 `Schema` methods: 54 forward to `fwschema.Schema*` helpers (ApplyTerraform5AttributePathStep, AttributeAtPath,
  AttributeAtTerraformPath, Type, TypeAtPath, TypeAtTerraformPath), 6 are the deprecated `Validate` forwarding to
  `ValidateImplementation(context.Background())`, 54 are one-line field getters, and 9 are multi-line
  `ValidateImplementation` bodies in 4 variants. So schema.go is already thin; the only remaining duplicated logic is
  the ValidateImplementation loop (D11) and the conversion helpers (D10).
- `AttributeAtTerraformPath` carries the `AttributeAtPath` doc in all 9 packages (D7).

## 4. Q4: perfsprint leads
- 663 leads: 662 `error-format` + 1 `string-format`. All 662 are in 171 `_test.go` files, every one is
  `expectedError: fmt.Errorf("<literal>")` with no `%` (`grep -c '%' $W/perfsprint_lines.txt` = 0), compared with
  `strings.Contains(err.Error(), testCase.expectedError.Error())`. Replacing with `errors.New` is behaviour-identical
  (fmt.Errorf without verbs or %w returns errors.New of the same text), and in all 171 files `fmt` is used only for
  these calls, so the change is about 662 lines plus 171 import swaps.
  **Rejected:** test-only churn with no behavioural or readability gain; perfsprint is not in the project's
  golangci config, so nothing would stop regressions. Could ride along only if the project adopts perfsprint.
- `resource/schema/stringdefault/static_value.go:31` `fmt.Sprintf("value defaults to %s", d.defaultVal)` ->
  concatenation is equivalent for a `string` argument, but **rejected**: the 11 sibling default packages must keep
  `Sprintf` (`%t/%d/%f/%v`), and one-off concatenation breaks the uniform pattern for no gain.
- No bool-format/integer-format leads exist in area D.

## 5. Q5: dead code and test gaps
- deadcode -test: 0 hits in area D. Unexported package-level funcs in the 9 packages: `nonComputedAttributeWithDefaultDiag`
  (used), `schemaAttributes` x9, `schemaBlocks` x6 (all used). The project's `unused` linter is clean.
- All 72 exported package-level funcs in the typed planmodifier/default packages are exercised by their own tests.
- Coverage (statements): resource/schema 99.5%, datasource 99.3, provider 99.3, provider/metaschema 99.3,
  ephemeral 99.3, action 99.3, list 98.4, statestore 98.7, resource/identityschema 100.0. The only 0% function in
  each of the 8 packages that have it is **`SingleNestedAttribute.GetAttributes`** (exported, no in-repo caller, no
  test): a test gap, cannot be removed (public API).
- Missing tests that the D-items need first: ObjectAttribute CustomType+AttributeTypes+Default (D1); dynamic
  UseNonNull underlying cases (D2); validator invocation count (D3); action SingleNestedAttribute ValidateImplementation
  (D4, the file has no ValidateImplementation test at all).
- doc.go staleness: provider/schema/doc.go (copy of the datasource doc), schema/doc.go (lists 3 of 9 concept
  packages), validator/planmodifier/defaults doc.go (nonexistent identifiers, typos): all in D7.

## 6. Q6: bug sweeps
| Sweep | Result |
|---|---|
| Getter returns the wrong field (1,613 single-statement getters grouped by method/body) | none; every minority is a per-concept constant (`return false`, `return ""`) or the documented identity `GetMarkdownDescription -> a.Description` |
| Getter doc names a field it does not return / "always false" doc on a field getter | only `NestedBlockObject.GetBlocks` x6 and action `ObjectAttribute.IsSensitive` x1 (D7) |
| Type missing an interface its siblings implement (var-block matrix vs method sets) | only provider/schema `MapNestedAttribute` lacks the ValidateImplementation assertion (D8); the method itself works |
| ValidateImplementation check present in one sibling, missing in another | action SingleNestedAttribute write-only (D4); action SetNested* write-only (D5); provider/metaschema dynamic-collection guard (not reported, below); resource ObjectAttribute default check wrong for custom types (D1) |
| Description vs MarkdownDescription swap | none (attributes, 12 plan modifiers, 12 defaults checked) |
| RequiresReplaceIf not copying diagnostics | none (12/12 identical) |
| Redundant/duplicated checks | resource SingleNestedAttribute double default diagnostic (D6) |
| Validators/modifiers run twice | single nested attribute/block validators (D3) |

### Candidates considered and NOT reported
- **provider/metaschema has no dynamic-in-collection guard** (`grep -c ContainsCollectionWithDynamic provider/metaschema/*.go` = 0,
  against 6-9 per other package). Probe `$W/probe5`: metaschema accepts `ListAttribute{ElementType: types.DynamicType}`
  and `list(object{d: dynamic})`, which provider/schema rejects, and fwserver does run metaschema ValidateImplementation
  (internal/fwserver/server.go:551). Not reported: provider_meta is read-only configuration, the dynamic-in-collection
  limitation mainly concerns provider-produced values, harm was not demonstrated, and dynamic support (#147, 1.7.0)
  deliberately skipped metaschema (no DynamicAttribute; the guard's message recommends DynamicAttribute, which does not
  exist there). Worth a maintainer question, not a finding.
- **identityschema does not reject RequiredForImport && OptionalForImport both true** - same as Required && Optional
  elsewhere, which is left to Terraform core; not a sibling gap.
- **list/schema has no Set* types, statestore/schema has no blocks** - deliberate API surface; cannot be "fixed" without new public API.
- **Equal docs say "all fields are equal"** while `fwschema.AttributesEqual` ignores validators/plan modifiers/defaults -
  long-standing internal/fwschema behaviour (area C), function values are not comparable; not a regression.

## 7. Lead accounting (1362 leads)
| Lead | Count | Disposition |
|---|---|---|
| perfsprint error-format | 662 | all tests, all verb-free; rejected (section 4) |
| perfsprint string-format | 1 | rejected (section 4) |
| dupl | 364 | 69 non-test, all same-package typed-attribute pairs (per-type design; shareable parts are D9/D10/D11); 295 test, per-type table-driven twins (e.g. float32 vs float64 attribute tests identical after normalisation; 100,398 test lines in the 9 packages). A generic test harness would cut volume but hurt failure readability; not proposed |
| gofix/any + golangci/modernize any | 148 + 148 | the same 148 sites, all `ApplyTerraform5AttributePathStep(...) (interface{}, error)` in non-test files; alias, 0 behaviour-changing sites; coordinator owns |
| funlen | 36 | test functions (table-driven); no action |
| godox | 2 | action/schema/schema.go:140,154 TODO:Actions: open design question whose resolution changes behaviour; noted in D11 |
| dupword | 1 | schema/validator/doc.go:31-32 "desired desired"; folded into D7 |

## 8. Please double-check (verifier)
1. D1: run `go test -vet=off -count=1 -overlay=$W/drafts/D-objdefault/overlay-before.json -run TestZZProbe -v ./resource/schema/`
   then with `overlay-after.json`. Confirm the custom type used is the repo's `internal/testing/testtypes.ObjectType`
   (strict `Equal`, internal/testing/testtypes/object.go:20).
2. D2: `cd $W/probe2 && GOFLAGS=-mod=mod GOPROXY=off go run .` (before) and `go run -overlay=$W/drafts/D-dyn-nonnull/overlay.json .` (after).
   Judge whether typed-null prior state for a dynamic attribute is realistic enough for tier 2 (I believe yes: the
   framework decodes it, and the package's own UseStateForUnknown tests model it).
3. D3: `go test -vet=off -count=1 -overlay=$W/drafts/D-snvalid/overlay.json -run TestZZProbe -v ./internal/fwserver/`
   (counts 2 / 1 / 2). Coordinate the fix site with area B1.
4. D5 depends on Terraform core's rule for write-only values in sets within action configs; confirm before fixing.
5. D6: confirm that removing the OUTER check instead would reorder diagnostics (probe in `$W/probe` prints 2 diagnostics,
   default first).
6. D9-D11: overlays `$W/drafts/D-equal-all/overlay.json`, `$W/drafts/D-maps/overlay.json`, `$W/drafts/D-schemavi/overlay.json`;
   the test command used is in each item's proof. They are tier 3: positive but modest value; reviewers may prefer per-package PRs.

## 9. Reproduction commands (run from /home/user/terraform-plugin-framework)
```
P="resource/schema datasource/schema provider/schema provider/metaschema ephemeral/schema action/schema list/schema statestore/schema resource/identityschema"
# Equal shapes (100 + 31) and block Equal (17)
for p in $P; do grep -h "^func (a [A-Za-z0-9]*) Equal(o fwschema.Attribute) bool {" $p/*.go; done | wc -l      # 131
for p in $P; do grep -h "^func (b [A-Za-z0-9]*) Equal(o fwschema.Block) bool {" $p/*.go; done | wc -l          # 17
# GetType shapes
for p in $P; do for f in $(ls $p/*.go | grep -v _test); do awk '/^func \(a [A-Za-z0-9]+\) GetType\(\) attr.Type \{/{p=1;b="";next} p{if($0=="}"){print b;p=0}else b=b"|"$0}' $f; done; done | sed -E 's/types\.[A-Za-z0-9]+Type$/types.XType/; s/types\.[A-Za-z0-9]+Type\{/types.XType{/' | sort | uniq -c
# double default diagnostic
grep -c nonComputedAttributeWithDefaultDiag resource/schema/*.go | grep -v ':0$\|:1$'                         # single_nested_attribute.go:2 (+ schema.go:2 = definition + doc)
# action write-only / set write-only
grep -n "ContainsAllWriteOnlyChildAttributes\|ContainsAnyWriteOnlyChildAttributes" action/schema/*.go resource/schema/*.go | grep -v _test
# metaschema dynamic guard
grep -c ContainsCollectionWithDynamic provider/metaschema/*.go | grep -v ':0$' | wc -l                         # 0
# single nested validators copied into GetNestedObject
grep -n "Validators: a.Validators\|Validators: b.Validators" */schema/single_nested_*.go
# doc defects
for p in $P; do grep -l 'GetType returns types.StringType' $p/bool_attribute.go; done | wc -l                 # 9
for p in $P; do [ -f $p/single_nested_attribute.go ] && grep -l 'GetType returns ListType of ObjectType' $p/single_nested_attribute.go; done | wc -l   # 8
grep -l 'GetAttributes returns the Blocks field value' */schema/nested_block_object.go | wc -l               # 6
grep -rn "type StringPlanModifier \|type StringValidator \|type StringDefault " --include=*.go . | wc -l     # 0
# perfsprint
grep '^golangci/perfsprint' <leads> | grep error-format | awk '{print $2}' > sites; while IFS=: read f l; do sed -n "${l}p" $f; done < sites | grep -c '%'   # 0
# missing assertion
for p in resource/schema datasource/schema provider/schema ephemeral/schema action/schema list/schema statestore/schema; do echo "$p $(sed -n '/^var (/,/^)/p' $p/map_nested_attribute.go | grep -c AttributeWithValidateImplementation)"; done   # provider/schema 0, others 1
```

## 10. Artifacts
- Drafts (all compile; tests pass via overlay): `$W/drafts/*`; unified diffs: `$W/diffs/D-{objdefault,dyn-nonnull,action-wo,sna,docs,equal-all,maps,schemavi,pm-all}.diff`.
- Rewrite scripts: `$W/rewrite_equal.py`, `$W/rewrite_maps.py`, `$W/rewrite_vi.py`, `$W/rewrite_pm.py`, `$W/rewrite_docs.py`.
- Probes: `$W/probe` (action write-only, single-nested default dedup), `$W/probe2` (dynamic UseNonNull),
  `$W/probe3` (embedding/cmp/reflect), `$W/probe4` (generic map conversion nil semantics), `$W/probe5` (metaschema dynamic),
  `$W/probe6` (provider MapNestedAttribute ValidateImplementation runs); overlay probe tests:
  `$W/drafts/D-objdefault/zz_probe_test.go`, `$W/drafts/D-snvalid/zz_probe_test.go`.
- Comparer: `$W/astcmp/astcmp [-nodoc] [-detail key] file:Token ...`.
