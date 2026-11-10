# B1 notes: fwserver plan modification and validation

Area files (production, read in full, 5,938 lines): internal/fwserver/attribute_plan_modification.go (2720),
block_plan_modification.go (1082), attribute_validation.go (1243), block_validation.go (465),
schema_plan_modification.go (197), schema_semantic_equality.go (146), schema_validation.go (85).
Tests (34,689 lines): read every TODO site, the case lists of the per-type tests, the PathExpression/deprecation/
custom-type cases; the 270 test dupl leads were sampled (20 read) and then measured as families by normalisation.
attr_type.go / attr_value.go (coerce helpers) are B2's; I only read them to trace behaviour.

All scratch artefacts: drafts/B1/
- repo-base/  pristine copy (every probe/fix was reverted; `diff -rq <repo>/internal repo-base/internal` is clean)
- repo/       cumulative drafts of B1-4..B1-10 (+ the B1-1 fix applied last)
- golden/     differential harness (see below) and its outputs
- probes/     the bug reproductions (copy one into internal/fwserver of a pristine copy and `go test ./internal/fwserver/ -run <Name> -v`)
- draft-*/ fix-*/  per-item diffs used for the size numbers

Tooling note: /usr/local/bin/golangci-lint is built with go1.25 and refuses this go1.26.0 module ("can't load config:
the Go language version (go1.25) used to build golangci-lint is lower than the targeted Go version (1.26.0)").
Drafts were checked with go build, go vet, gofmt -s -l, `go test ./internal/fwserver/` (plus proto5server/proto6server
for the cumulative state), and by reading against the enabled linters (forcetypeassert: every assertion uses ok;
unparam: every new parameter receives varying non-constant values; unused: every spec var is referenced).

## Counts and the commands that reproduce them (run in the repo root)

- Function sizes: `awk '/^func /{n=$0;s=NR} /^}/{if(n!=""){print NR-s+1, n; n=""}}' internal/fwserver/attribute_plan_modification.go`
  AttributeModifyPlan 602, 12 x AttributePlanModify<T> 158, NestedAttributeObjectPlanModify 113; BlockModifyPlan 425,
  3 x BlockPlanModify<T> 158, NestedBlockObjectPlanModify 152; AttributeValidate 143, 12 x AttributeValidate<T> 64,
  AttributeValidateNestedAttributes 164, NestedAttributeObjectValidate 77; BlockValidate 152, 3 x BlockValidate<T> 64,
  NestedBlockObjectValidate 91; SchemaModifyPlan 135; SchemaSemanticEquality 95; SchemaValidate 41.
- Twins (example for the 12 plan-modify functions; ranges are doc comment through closing brace):
  `for spec in Bool:687:846 Float32:847:1006 ... Dynamic:2447:2606; do T=${spec%%:*}; ...; sed -n "${s},${e}p" F | sed "s/$T/TYPE/g" > apm_$T.txt; done; diff apm_Bool.txt apm_$T.txt`
  Result: 0 differing lines for all 11 pairs. Same 0 for BlockPlanModify List/Object/Set, AttributeValidate x12,
  BlockValidate x3. Attribute vs block plan-modify twins differ in 5 hunks only: doc+signature, receiver name, and
  "Attribute"/"attribute" -> "Block"/"block" in the 3 invalid-type diagnostics. Attribute vs block validate twins differ
  in variable names only; NOTE the block validators keep the word "Attribute" in their diagnostic
  ("Invalid List Attribute Validator Value Type", block_validation.go:188/254/320).
- Public struct shapes (decides the conversion trick in B1-4/B1-6): all 12 planmodifier.<T>Request have the field
  sequence Path, PathExpression, Config, ConfigValue, Plan, PlanValue, State, StateValue, Private; all 12 <T>Response
  have PlanValue, RequiresReplace, Private, Diagnostics; all 12 validator.<T>Request have Path, PathExpression, Config,
  ConfigValue, ClientCapabilities; <T>Response = {Diagnostics}. types.Bool etc. are aliases (types/*.go `type Bool = basetypes.BoolValue`).
  Command: awk over resource/schema/planmodifier/<t>.go and schema/validator/<t>.go, `sort | uniq -c` gives 12 identical rows each.
- Untested branches: `grep -rhoE "Invalid [A-Za-z0-9]+ (Attribute|Block) (Plan Modifier|Validator) Value Type" --include=*_test.go .` -> 0;
  `grep -rn "Calling provider defined planmodifier\|Calling provider defined validator" --include=*_test.go .` -> 0;
  `grep -rn "tf_attribute_path" --include=*_test.go .` -> 1 (internal/logging only).
- Callers (dead-code sweep): `grep -rnE --include=*.go "\b$fn\(" . | grep -v _test.go | grep -v ":func $fn("` for every
  func in the 7 files: every function has >= 1 production caller; no dead functions.
- #365 references: `grep -rn "issues/365" --include=*.go .` -> 7, all in this area. #709: 14, all in the two plan-modification test files.
- Non-test dupl leads: 44, every one maps to B1-4/5 (per-type), B1-6 (validate per-type), B1-7/B1-8 (nesting arms),
  B1-9 (nested object loops, dupl attribute_plan_modification.go:2665-2713 vs block_plan_modification.go:977-1025/1027-1075),
  B1-10 (schema loops), or the rejected validation nested-object pair (attribute_validation.go:1170-1228 vs block_validation.go:378-436).

## Item 1: the twelve AttributePlanModify<Type> functions and the generic version

Generic design drafted (drafts/B1/draft-generic-pm/value_plan_modification.go, 325 lines):
- valuePlanModifyRequest[V] / valuePlanModifyResponse[V]: same fields in the same order as every planmodifier.<T>Request/
  <T>Response, so the per-type adaptor converts with `planmodifier.BoolRequest(req)` and `(*planmodifier.BoolResponse)(resp)`.
  Struct conversion requires identical underlying types, so adding/reordering a field in any public struct breaks the
  build instead of silently dropping data. The provider still receives a *planmodifier.BoolResponse (same dynamic type
  under reflection) that points at the framework's struct, exactly like today.
- valuePlanModifyType[Valuable, Value, Typable, Modifier] spec per type, all method expressions:
  toValue: basetypes.BoolValuable.ToBoolValue; typable: coerceBoolTypable; valueFrom: basetypes.BoolTypable.ValueFromBool;
  modify: func(m planmodifier.Bool, ...) { m.PlanModifyBool(ctx, planmodifier.BoolRequest(req), (*planmodifier.BoolResponse)(resp)) }.
  12 specs serve all 15 functions (List/Object/Set are shared by attributes and blocks).
- valuePlanModify(ctx, spec, kind "Attribute"|"Block", planModifiers func() []Modifier, req, resp): the 158-line body once.
  The getter is passed as a method value (attribute.BoolPlanModifiers) and called at the same point as today (after the
  three coercions and the typable check). Passing the slice instead would call the getter before the coercions and also
  when they fail: not equivalent for an embedding type that overrides the getter (rejected variant).
- Diagnostics: "Invalid "+name+" "+kind+" Plan Modifier Value Type" and the detail with strings.ToLower(kind) (ASCII,
  identical bytes) and fmt.Sprintf("Incoming Value Type: %T", value) on the same attr.Value interface (same %T output,
  "<nil>" for a nil interface). Log messages "Calling provider defined planmodifier."+name, map[string]any (same type).
Measured (diff -u vs pristine): attribute_plan_modification.go +12/-1873, block_plan_modification.go +3/-469, new file +325.
Area plan-modification files 3,802 -> 1,800 lines. The wrapper doc comments and signatures are unchanged lines.
Equivalence evidence: build/vet/gofmt clean; `go test ./internal/fwserver/` passes; golden harness byte-identical (below).
Is it clearly simpler? I think yes for maintenance: the algorithm exists once instead of 15 times and the per-type knowledge is a
9-line table of method expressions per type. Costs a reviewer should weigh: 4 type parameters; message literals become
templates, so grepping a user-reported message no longer lands on the code; the conversion trick needs a comment.
Smaller step if generics are rejected (estimated, not drafted): one helper for the 45 identical invalid-value-type
diagnostic blocks (~-360 lines) and one for the 30 trace-log pairs (~-180 lines); no per-type body sharing is possible
without generics because each type's Valuable/To<T>Value/ValueFrom<T>/PlanModify<T> differ by name.
Validation family: same design (value_validation.go, 205 lines; no kind parameter because block validators also say "Attribute").
Measured +12/-744 and +3/-186; 1,708 -> 998 lines.

Golden differential harness (drafts/B1/golden/zz_golden_b1_planmodify_test.go and zz_golden_b1_validate_test.go, generated):
for each of the 15+15 per-type functions: wrong-typed config, plan, state; nil config; no modifiers; a 3-modifier chain
(warning + RequiresReplace + PlanValue := ConfigValue; error + Private = nil; a third modifier that must not run) and a
2-modifier chain (PlanValue := StateValue, RequiresReplace twice, Diagnostics = diag.Diagnostics{}). Records every
modifier's request (path, expression, values, Private nil-ness) and initial response, the final response (plan %T and
value, every diagnostic's type/severity/summary/detail/path, RequiresReplace nil-ness and value, Private nil-ness), and
every trace log entry (tfsdklogtest). pm: 585 lines incl. 60 invalid-type diagnostics, 120 log lines, 60 calls;
v: 255 lines incl. 30 diagnostics, 90 log lines, 45 calls. Outputs identical between pristine and refactored copies
(pm-repo-base.txt = pm-repo.txt = pm-cumulative.txt; v-repo-base.txt = v-repo.txt = v-cumulative.txt).

## Item 2: phase maps of the long functions

AttributeModifyPlan (602 lines; gocognit 133):
 P0 setup: ctx = FrameworkWithAttributePath; resp.Private = req.Private if non-nil.
 P1 value plan modifiers: type switch -> AttributePlanModify<T> (mutates resp.AttributePlan/Diagnostics/Private/RequiresReplace).
 P2 gates: resp has error -> return; resp.AttributePlan null/unknown -> return; not fwschema.NestedAttribute -> return.
 P3 nested walk by nesting mode: List / Set / Map arms (~150 lines each): coerce config collection, plan collection
    (valuable, typable, value; from resp.AttributePlan, i.e. after P1), state collection; per element: config element,
    plan object, plan object valuable + typable, state element, ObjectRequest, NestedAttributeObjectPlanModify,
    re-wrap with the element custom type (#821), assign element, merge nested diagnostics/Private/RequiresReplace;
    rebuild the collection and re-wrap with the collection custom type (#754). Single arm (~85 lines) merges the nested
    diagnostics/Private/RequiresReplace BEFORE the coerce + ValueFromObject steps (collection arms merge AFTER).
    Default arm: error diagnostic.
 State that pins the order: resp.Diagnostics (every check in P3 is the cumulative resp.Diagnostics.HasError(), so an
 error from P1 or from element i stops element i+1 at its first check and leaves resp.AttributePlan un-rebuilt);
 resp.Private (threaded element to element through objectReq.Private); resp.RequiresReplace (dedup Append order);
 resp.AttributePlan (P3 reads the post-P1 value); ctx (P0 before any provider call). The switch is the function's last
 statement, so moving each arm verbatim into its own function keeps every return equivalent (B1-7, drafted).
BlockModifyPlan (425; gocognit 95): same without the logging field (B1-2) and without the Map arm; its List/Set/Single
 arms are byte-identical to the attribute arms after replacing the nested-object call (asserted by the B1-7 generator).
AttributeValidate (143; gocyclo 39): V0 ctx; V1 three definition checks (each returns); V2 ValueAtPath (returns on error);
 V3 null/unknown classification incl. the dynamic underlying value (returns on error); V4 read-only / required /
 write-only-not-allowed checks (append only); V5 req.AttributeConfig = value; V6 per-type validators; V7
 AttributeValidateNestedAttributes; V8 deprecation warning (known values only). Pinning state: resp.Diagnostics (V7's
 arms use the cumulative HasError, so an error from V4/V6 skips nested validation: probe nestedskip), req.AttributeConfig
 (V5 before V6/V7), ctx. A split into definition/classification/presence helpers is possible with the same order but low
 value (143 linear lines); not reported.
BlockValidate (152): ValueAtPath; per-type validators; an INLINE nesting switch (copy of AttributeValidateNestedAttributes'
 arms minus Map and minus the single-arm null/unknown guard); deprecation warning. Because the switch is inline, its
 early returns skip the deprecation warning (B1-3). A pure refactor that extracts the switch must keep that skip
 (`if !blockValidateNested(...) { return }`); extracting it as a plain call IS the B1-3 fix.

## Item 3: validation vs plan-modification traversal overlap (measured)

Normalised (trimmed, comments/brace/return-only lines dropped) distinct lines: plan switch (attribute_plan_modification.go:137-686)
81, validation switch (attribute_validation.go:1011-1164) 49, common 10, all boilerplate (`case fwschema.NestingModeX:`,
`default:`, `switch nm {`, `resp.Diagnostics.Append(diags...)`, `if resp.Diagnostics.HasError() {`, `resp.Diagnostics.AddAttributeError(`,
`req.AttributePath,`). Validation iterates config elements and derives expressions from the parent expression
(req.AttributePathExpression.AtListIndex(idx)); plan modification zips plan elements with config/state lookups by
index/key and derives expressions from the exact path (attrPath.Expression()). No shared walker recommended.

## Item 4: godox (21)
- 7 production TODOs "Clean up this abstraction back into an internal Attribute/Block/Schema type method ... #365"
  (attribute_plan_modification.go:81, attribute_validation.go:56 and :998, block_plan_modification.go:21,
  block_validation.go:20, schema_plan_modification.go:59, schema_validation.go:41): STALE. #365 was closed as completed on
  2022-09-22 ("This internal package refactoring was completed previously. Other issues can be used to track additional
  efforts."). The described end state is also not reachable without moving fwserver logic into the public schema
  packages (fwschema.Attribute is an interface implemented only by public types). -> B1-12.
- 14 test TODOs "Rework list/set element alignment during plan ... #709" (attribute_plan_modification_test.go:909, 921, 975,
  987, 1158, 2250, 2262, 2433; block_plan_modification_test.go:2330, 2342, 2513, 2903, 2915, 3086): STILL VALID. #709 is
  open (closed 2023-04-06 via #711, reopened 2023-06-06, #711 reverted); the code still aligns list/set elements by index
  (listElemObject/setElemObject with idx). The expectations they annotate document the known-wrong current behaviour.
  Not a new bug; no action.

## Item 5: exhaustive / gocognit / gocyclo / funlen
- exhaustive x4 (attribute_plan_modification.go:137, attribute_validation.go:1012, block_plan_modification.go:51,
  block_validation.go:52): FALSE POSITIVES. Each switch has a default arm that emits an error diagnostic for
  NestingModeUnknown/BlockNestingModeUnknown. No production type returns the Unknown mode (grep NestingModeUnknown:
  only the fwschema declarations). The default arms are untested (no test references their texts); not worth a finding.
- gocognit AttributeModifyPlan 133 / BlockModifyPlan 95: addressed by B1-7 (76 and 48 lines after the move).
- gocyclo AttributeValidate 39: see phase map; not reported.
- funlen x18: 17 test functions (funlen is not an enabled linter) + AttributeValidateNestedAttributes 162 (> 150): not reported.

## Item 6: dead code
- No function without callers (sweep above). No always-true guards found; the redundant trailing `return`s at the end of the
  default arms of AttributeModifyPlan/BlockModifyPlan/AttributeValidateNestedAttributes and in AttributeValidate's final
  deprecation block are harmless style (the one in BlockValidate's default arm is NOT redundant: it skips the warning).
- Dead data: ModifyAttributePlanRequest.ProviderMeta and ModifySchemaPlanRequest.ProviderMeta are written, never read
  (deleting both compiles and passes; no reference outside internal/fwserver) -> B1-11.

## Item 7: bugs and twin inconsistencies (traced)
Reported:
- B1-1 PathExpression (probes pathexpr, pathexpr2, pathexpr3).
- B1-2 blocks never set tf_attribute_path (probe logpath).
- B1-3 BlockValidate inline-switch returns drop "Block Deprecated" (probe deprecation).
Investigated and NOT reported (verifier: please do not resurrect these without new evidence):
- Null SingleNestedBlock with a Required nested attribute raises "Missing Configuration for Required Attribute" (blk.req),
  while the same null value for a SingleNestedAttribute raises nothing (AttributeValidateNestedAttributes has
  `if o.IsNull() || o.IsUnknown() { return }`, BlockValidate's single arm does not). Reproduced (probe singleblock), but
  INTENDED: hashicorp/terraform-plugin-framework#740 closed "not planned" by the maintainer: "current behavior is
  desirable/correct". The unknown-block variant is not producible by Terraform: hcldec BlockSpec (NestingSingle) has no
  UnknownBody check (unlike BlockListSpec etc.), so a dynamic single block with unknown for_each decodes to a known
  object with unknown attributes. Any refactor that shares the single arm MUST keep the guard difference.
- NestedBlockObjectPlanModify lacks the #993 guard that NestedAttributeObjectPlanModify has (skip children when the object
  is null/unknown after object-level modifiers). Reachable only if a provider's NestedBlockObject plan modifier turns a
  block element null/unknown, which Terraform rejects anyway (block elements cannot be null/unknown in a valid plan;
  dynamic blocks with unknown for_each make the whole collection unknown, which BlockModifyPlan already gates). Keep the
  divergence in any refactor (B1-9 keeps it); adding the guard would be a behaviour change.
- Single arm vs collection arms merge order: Single merges objectResp diagnostics/Private/RequiresReplace before the
  coerce + typable.ValueFromObject steps; List/Set/Map merge after, so when a custom ObjectTypable.ValueFromObject fails
  the collection arms drop the nested diagnostics (only reachable with a failing custom object type; the response is an
  error either way). Inconsistent but negligible; any unification must keep per-arm order.
- Nested validation is skipped when the parent attribute's own validators/checks produced an error (cumulative
  resp.Diagnostics.HasError() in AttributeValidateNestedAttributes and BlockValidate; probe nestedskip). Same design as plan
  modification (AttributeModifyPlan returns after P1 errors). Looks intentional; changing the checks to diags.HasError()
  (the "only return early on new errors" convention used elsewhere in the file) is NOT a refactor.
- SchemaModifyPlan starts every root attribute from req.Private and keeps only the last root's resp.Private. A plan
  modifier that REPLACES resp.Private (instead of calling SetKey, which mutates the shared *ProviderData) would have its
  replacement lost, depending on map iteration order. Documented usage (SetKey) is unaffected; not reported.
- path.Paths aliasing: nested responses are seeded with RequiresReplace: resp.RequiresReplace (shared backing array) and
  merged back with the dedup Append; both writers write the same values in the same slots; benign.
- Block validators' invalid-type diagnostic says "Attribute" (block_validation.go:188/254/320) whereas block plan
  modifiers say "Block". Cosmetic, practically unreachable (needs a custom block type whose value is not ListValuable);
  B1-6 preserves the text; changing it would be a separate behaviour change.
- Response fields copied back: PlanValue, RequiresReplace, Private, Diagnostics are handled in all 15+2 plan-modify
  loops (byte-identical twins); no field dropped.

## Rejected rewrites ("looks equivalent but is not")
1. Passing attribute.<T>PlanModifiers() (the slice) to the generic: moves the getter call before the coercions and runs it
   when they fail. Used the method value instead (same call point).
2. Pre-computing the state element lookup in the per-element helper (B1-8): reorders which diagnostic is reported when
   both a plan-element coercion and the state lookup fail. Used a closure called at the original point.
3. Unifying the Single arm with the collection arms (merge before/after coerce): see item 7.
4. Sharing BlockValidate's single arm with AttributeValidateNestedAttributes': drops or adds the null/unknown guard (#740).
5. Switching validation early returns to diags.HasError(): runs nested validation after parent errors (probe nestedskip).
6. Adding the #993 guard while deduplicating NestedAttribute/NestedBlockObjectPlanModify: behaviour change for blocks.
7. Building planmodifier requests field-by-field per type instead of the struct conversion: equivalent but +~130 lines
   and loses the compile-time guarantee that every field is copied; kept the conversion.
Rejected as not clearly simpler (measured or estimated):
- One walker shared by validation and plan modification: 10 shared boilerplate lines (item 3).
- Sharing the List/Set arms of BlockValidate and AttributeValidateNestedAttributes: ~-55 lines (estimated) for three extra
  parameters (two diagnostic wordings + the single-arm guard flag). After B1-3's extraction it becomes easier; still marginal.
- Sharing NestedAttributeObjectValidate / NestedBlockObjectValidate (dupl 1170-1228 vs 378-436): object-validator parts
  differ in diagnostic wording ("Attribute Validation Walk Error" vs "Block Validation Walk Error"); ~-40 lines; marginal.
- AttributeValidate split (gocyclo 39): linear, readable; low value.

## Leads discarded
- gofix/any 68 + golangci modernize any 68: the same 68 sites, all `map[string]interface{}` literals in logging.FrameworkTrace
  calls; byte-identical with `any`; none changes behaviour; coordinator owns them. (B1-4/B1-6 would remove 60 of them.)
- golangci dupl 270 test hits: per-type test twins (TestAttributePlanModify<T> 633 lines for the 8 scalar-like types;
  normalised pairs differ in 138-150 lines, all literal fixture values such as types.BoolValue(true) vs types.Float32Value(1.2);
  TestAttributeValidate<T> 227 lines) and attribute/block test twins -> B1-14 (estimated only).

## For the verifier
- B1-1: run probes zz_probe_b1_pathexpr_test.go (top "", outer.inner "inner", blk.battr "battr", lst[0].le correct),
  zz_probe_b1_pathexpr2_test.go (validator matches [outer.sibling], plan modifier matches [sibling]),
  zz_probe_b1_pathexpr3_test.go (top-level relative sibling -> "Invalid Path Expression for Schema" error). Contract:
  resource/schema/planmodifier/bool.go:30-32 "PathExpression contains the expression matching the exact path of the
  attribute for modification"; per-type unit tests feed path.MatchRoot("test") (attribute_plan_modification_test.go:3863 etc.).
- B1-7: re-assert the arms are identical: extract attribute_plan_modification.go:138-686 and block_plan_modification.go:52-449,
  de-indent, replace the nested-object call, diff (only the Map arm and the default-arm texts differ).
- B1-4/5/6: re-run the golden harness on a pristine copy and on the draft; byte-identical outputs expected.
- B1-11: delete both ProviderMeta fields + the 3 writes; `go build ./... ` scope: only internal/fwserver references them.
