# V-B1 working notes (verifier of findings-B1.json)

Repo /home/user/terraform-plugin-framework at 00badb1 (read-only; untouched). The clone is SHALLOW (124 commits,
grafted at e43fb0b, 2025-05-16), so `git log -S`/blame cannot reach the original authorship of any line in this area.
All work under drafts/V-B1/ (below: $V).

Method: one pristine copy ($V/pristine, `tar` of the repo minus .git; `diff -rq` against the repo: identical) and
`go test -overlay <json>` for every probe, fix and draft, so the pristine tree is never edited. Overlay files:
$V/ov-*.json. Author artefacts used read-only from work/B1 (snap-after-generic, snap-after-arms,
snap-before-nestedobj, repo = cumulative, golden/). Full draft copies for golangci-lint: $V/draft-generic,
$V/draft-msgfix, $V/draft-cumul (== author's cumulative internal/, checked with diff -rq), $V/draft-armsdirect.

## B1-1 (PathExpression) -- reproduced independently, CONFIRMED

Probe $V/probes/zz_vb1_pathexpr_test.go (external test package; ValidateResourceConfig then PlanResourceChange;
recording String/Object/List plan modifiers and String validators; relative expression
MatchRelative().AtParent().AtName("sibling"); schema with and without a root-level "sibling"). Output
$V/out-pathexpr-pristine.txt vs $V/out-pathexpr-fixed.txt. Pristine, with root "sibling":

    PM  battr  path=blk.battr  expr="battr"        siblingMatch=[sibling]           (wrong: root attribute)
    PM  inner  path=outer.inner expr="inner"       siblingMatch=[sibling]           (wrong: root attribute)
    PM  lbe    path=lblk[0].lbe expr="lblk[0].lbe" siblingMatch=[lblk[0].sibling]  (correct)
    PM  le     path=lst[0].le  expr="lst[0].le"    siblingMatch=[lst[0].sibling]   (correct)
    PM  top    path=top        expr=""             siblingMatch=[]  diags=Error:Invalid Path Expression for Schema
    PML lblk   expr=""   PMO blk expr=""   PMO outer expr=""   PMO lblk[] expr="lblk[0]"
    VAL ...    every validator gets the full expression and the right match
Pristine, WITHOUT a root "sibling": inner and battr also fail with "Invalid Path Expression for Schema" (so "silently
reads a root-level attribute" only holds when a same-named root attribute exists). With my 2-line fix
($V/fix-b1-1/fix.diff, +14/-12 incl. gofmt realignment): every PM gets the validator's expression and match.
`go test -overlay ov-fix1.json ./internal/fwserver/ ./internal/proto5server/ ./internal/proto6server/` ok (1278 PASS
lines in -v). No schema/server test mentions PathExpression (only the 4 per-type test files, which inject
path.MatchRoot("test"): 12 + 3 sites).

Intent/history: CHANGELOG 0.10.0 (#396) "Added AttributePathExpression field to ModifyAttributePlanRequest and
ValidateAttributeRequest types"; developer.hashicorp.com/terraform/plugin/framework/handling-data/path-expressions:
"Provider-defined functionality that is schema-based, such as attribute validation and attribute plan modification, are
provided an accurate current path expression ...". UPSTREAM (the author said none found; wrong):
- #1258 "Empty `PathExpression` in Plan Modifiers breaks `path.MatchRelative()`", open, 2026-01-21, v1.17.0,
  error "Invalid Path Expression for Schema" for a top-level attribute; fix PR #1260 (open, 2026-01-23) adds
  `AttributePathExpression: path.MatchRoot(name)` in both loops + a schema test.
- #1219 "Incorrect `PathExpression` Passed into Plan Modifier of an Attribute in a 2nd Layer Nested Object", open,
  2025-09-10: nested.double_nested.str_a gets "double_nested.str_a"; fix PR #1311 (open, 2026-07-06), same fix + test;
  notes that providers matching the truncated expression will see the full one.
(GitHub API is blocked for this session; status read with WebFetch on the public pages.)

## B1-2 (tf_attribute_path for blocks) -- reproduced, CONFIRMED

Probe $V/probes/zz_vb1_logpath_test.go (tfsdklogtest; ListNestedAttribute vs ListNestedBlock, each with list
validator/modifier, object validator/modifier on the nested object, and a string child). Pristine:
`validator.List blk_list-v`, `validator.Object blk_list[]-objv`, `planmodifier.List blk_list-pm`,
`planmodifier.Object blk_list[]-objpm` and the `Type Validate` entry of BlockValidate's ValueAtPath: tf_attribute_path
ABSENT; attribute twins carry `attr_list`. Fix ($V/fix-b1-2/fix.diff: +2 statements +2 blank = +4) -> all carry
`blk_list`. fwserver tests pass. Other FrameworkWithAttributePath callers (server_planresourcechange.go:416,
write_only_nullification.go:24, fwschemadata get/set/semantic-equality) set it for every path, blocks included, so the
omission does not look like a design choice.

## B1-3 (Block Deprecated dropped on error) -- reproduced, CONFIRMED

Probe $V/probes/zz_vb1_deprecation_test.go. Pristine diagnostics:
    Error list validator error attr_list_err / Warning Attribute Deprecated attr_list_err
    Error list validator error blk_list_err   / (no Block Deprecated)
    Error object validator error blk_single_err / (no Block Deprecated)
    Error nested string validator error blk_list_nested_err[0].a / Warning Block Deprecated blk_list_nested_err
    Warning Block Deprecated blk_list_ok
With my extraction ($V/fix-b1-3/fix.diff, +12/-8; author measured +14/-9 with a different hunking) all five
deprecation warnings appear; fwserver tests pass. Deliberate? AttributeValidate emits the warning after errors;
the block arms are copies of AttributeValidateNestedAttributes' arms, whose returns end only that function; the comment
names only null/unknown; only 3 deprecation test cases exist (known/null/unknown), none with an error. History not
available (shallow clone). Judged unintended.

## B1-4 / B1-5 / B1-6 (generic per-type bodies) -- CONFIRMED with corrections

Twins (my own extraction: doc comment + func through closing brace, `sed s/<Type>/TYPE/g`, diff vs Bool/List):
AttributePlanModify x12, BlockPlanModify x3, AttributeValidate x12, BlockValidate x3: 0 differing lines in every pair.
Function sizes 158 (plan modify) and 64 (validate). Attribute vs block plan-modify: only doc/signature, receiver name,
and "Attribute"/"attribute" -> "Block"/"block" in the 3 diagnostics; attribute vs block validate: variable names only
(block validators say "Attribute"). Read value_plan_modification.go and value_validation.go line by line against
AttributePlanModifyBool/AttributeValidateBool: same statements, order, texts, nil handling.

Golden harness (author's generator, re-run by me; GOLDEN_OUT per test, overlay builds):
    pm: pristine == generic (snap-after-generic) == cumulative (drafts/B1/repo) : 585 lines, byte-identical
    v : pristine == generic == cumulative                                     : 255 lines, byte-identical
    my pristine outputs == author's pm-repo-base.txt / v-repo-base.txt
    counts: pm 60 invalid-type diags, 120 log lines, 60 calls; v 30 diags, 90 log lines, 45 calls.
Gap: the harness only produces "Invalid <T> <Kind> Plan Modifier Value Type" and the modifier error "e2"; the
ToValue / coerce<T>Typable / ValueFrom<T> error returns are not exercised (also uncovered by the suite, see B1-13).

Tests: `go test -overlay ov-generic.json` and `ov-cumul.json` on ./internal/fwserver/ ./internal/proto5server/
./internal/proto6server/: ok. go vet ok. golangci-lint v2.15 (work/../bin, go1.26 build) with the repo config on
./internal/fwserver/...: 0 issues on pristine, draft-generic, draft-msgfix, draft-cumul, draft-armsdirect.
Sizes: attribute_plan_modification.go 2720 -> 859 (+12/-1873), block_plan_modification.go 1082 -> 616 (+3/-469),
attribute_validation.go 1243 -> 511 (+12/-744), block_validation.go 465 -> 282 (+3/-186), new files 325 + 205:
3,802 -> 1,800 and 1,708 -> 998 as claimed. map[string]interface{} in the 7 area files: 68 -> 8 (60 removed).

HOT PATH (asked by the caller). Microbenchmark $V/probes/zz_vb1_bench_test.go (internal package; 2 no-op modifiers /
validators; testing.AllocsPerRun + b.Loop):
    pristine: AttributePlanModifyString 5 allocs/op 272 B/op ~380-440 ns; AttributeValidateString 3 allocs 144 B ~250-280 ns
    generic : AttributePlanModifyString 9 allocs/op 464 B/op ~1130-1350 ns; AttributeValidateString 7 allocs 336 B ~580-830 ns
`go build -gcflags=-m`: `"Calling provider defined planmodifier." + t.name escapes to heap` (and Called, and the
validator twins): 2 runtime concatenations per modifier/validator call, made even with logging off; the original
passes string constants. No reflection added; dictionary calls are not the cost: variant $V/generic-msgfix
(callingMsg/calledMsg literal fields per spec; diff $V/generic-msgfix/msgfix.diff) gives 5 / 3 allocs, 272 / 144 B,
timing within noise of pristine; golden identical; lint clean; files 351 + 231 lines (+26 each), so 3,802 -> 1,826
and 1,708 -> 1,024. End to end ($V/probes/zz_vb1_bench_e2e_test.go, PlanResourceChange, 30 string attributes x 2
no-op modifiers): pristine 8,302 allocs/op 944,757 B/op; generic 8,422 / 950,518; msgfix 8,302 / 944,756 (timing too
noisy on the shared machine). Also: @caller of the trace entries changes to value_plan_modification.go (golden
excludes @caller; any edit moves it).

## B1-7 .. B1-10 (splits and helpers)

B1-7 verbatim check (my own python: extract each `case` arm of pristine AttributeModifyPlan/BlockModifyPlan, de-indent
one tab, replace the nested call with `nestedObjectPlanModify(ctx, objectReq, objectResp)`, diff against the helper
bodies of snap-after-arms): list 150/150, set 150/150, map 150/150, single 84/84 lines, 0 differing lines; block
List/Set/Single arms == attribute arms after substituting NestedBlockObjectPlanModify -> NestedAttributeObjectPlanModify
(0 lines). Sizes: snap-after-generic -> snap-after-arms attribute +337/-309 (62 lines with -w; 859 -> 887), block
+7/-384 (616 -> 239): net -349. AttributeModifyPlan 602 -> 76, BlockModifyPlan 425 -> 48. Variable capture: the
helpers compile with only ctx/req/resp and the closure; only one arm runs; the switch is the last statement.
B1-8 elem.diff +84/-189; B1-9 +105/-76 and +12/-123 (net -82); B1-10 +53/-85 and +38/-71 (net -65): reproduced.
B1-8 order argument (state lookup closure at the original point; element assignment after merges on a local copy)
holds by reading; B1-9 `&req` handling holds; B1-10 helper is a faithful move.

ALLOCATIONS (escape analysis, `go build -gcflags=-m`): closures themselves do not escape, but values passed THROUGH
them do. Pristine/draft-generic: no ModifyAttributePlanResponse escapes. draft-cumul:
    attribute_plan_modification.go: &ModifyAttributePlanResponse{...} escapes to heap in singleNestedPlanModify (B1-7),
      nestedObjectElementPlanModify (B1-8; before B1-8 the same escape sits in the list/set/map helpers), nestedPlanModify (B1-9)
    schema_plan_modification.go: moved to heap: attrResp (schemaModifyPlanAtRoot, B1-10)
Nested fixture $V/probes/zz_vb1_bench_nested_test.go (10 root strings, ListNestedAttribute x10 elements x3 attrs,
SingleNestedAttribute x3, ListNestedBlock x5 x2; no modifiers): allocs/op pristine 17,741; generic 17,741;
+B1-7 17,757 (+16 = one per nested object); +B1-8 17,757; cumulative w/o B1-1 fix 17,813 (+56 = 43 nested children
(B1-9) + 13 roots (B1-10)); B1-1 fix alone +29 (root expressions; inherent to the fix). Direct-dispatch variant of
B1-7 ($V/arms-direct/direct.diff: struct{attributeObject, blockObject, isBlock} with a planModify method calling the
right function directly): 17,741 (parity), no escapes, fwserver tests ok, lint 0 issues; attribute file 899 lines,
block 237 -> net -339 instead of -349.

## B1-11 / B1-12

B1-11: my own deletion ($V/fix-b1-11/fix.diff): -3, -5, -4 = -12; vet ok; fwserver/proto5server/proto6server ok;
`go build ./...` ok. Readers: none (grep ProviderMeta in internal/fwserver non-test: only the declarations and the
writes); tests mention ProviderMeta only on server-level request types, which stay.
B1-12: `grep -rn "issues/365" --include=*.go .` -> 7, all in area files; each paragraph is 5 comment lines (35).
#365 (WebFetch): "Consider Refactoring tfsdk Schema/Attribute/Block Logic into internal Package(s)", closed as
completed 2022-09-22 by bflad: "This `internal` package refactoring was completed previously. Other issues can be used
to track additional efforts." #709: open; closed via #711 on 2023-04-06, reopened 2023-06-06 with revert commits
("maintainers were not able to rectify the second half of the changes"). 14 test TODOs reference it (8 + 6).

## B1-13 coverage

`go test ./internal/fwserver/ -coverprofile` (79.5% of statements): AttributePlanModify<T> 76.1% each,
BlockPlanModify<T> 76.1%, AttributeValidate<T>/BlockValidate<T> 80.0%. Uncovered blocks in AttributePlanModifyBool:
696-707, 721-732, 746-757 (the 3 invalid-type diagnostics), 715-717, 740-742, 765-767 (To<T>Value error returns),
775-777 (typable error return), 840-842 (ValueFrom<T> error return); AttributeValidateBool: 212-223 (diagnostic),
231-233 (ToValue error return). Greps: invalid-type summaries in tests 0 (60 in non-test code), "Calling provider
defined planmodifier|validator" in tests 0, tf_attribute_path in tests 1 (internal/logging).

## B1-14

Sizes confirmed (8 x 633 at 3803-6971, 8604-9236, 11087-12353; 8 x 227 at 2088-3226, 3735-3961, 4480-4934);
normalised pairs differ in 138/150 lines (pm) and 18/22 lines (v), all fixture literals. Each case embeds typed
testplanmodifier.<T>{PlanModify<T>Method: func(..., planmodifier.<T>Request, *planmodifier.<T>Response)} and
testschema.AttributeWith<T>PlanModifiers, so one table needs adaptors that mirror B1-4's struct-conversion trick.
Verdict PLAUSIBLE low -> drop.

## Rejected candidates and exhaustive leads (author's notes)

- #740: my probe $V/probes/zz_vb1_singlenull_test.go: null SingleNestedBlock with a Required child -> 1 error
  "Missing Configuration for Required Attribute" at blk.req; the null SingleNestedAttribute -> nothing. #740
  (WebFetch): "Allow Optional SingleNestedBlocks and Required Attributes within", closed not planned 2023-06-12, bflad:
  "we have concluded that the current behavior is desirable/correct". Rejection correct.
  HCL claim checked (WebFetch of hashicorp/hcl hcldec/spec.go): BlockSpec.decode has no UnknownBody check,
  BlockListSpec.decode has one. Correct.
- #993 guard missing in NestedBlockObjectPlanModify: correct by reading; small imprecision: besides a NestedBlockObject
  plan modifier, a block-level List/Set plan modifier that returns null/unknown elements also reaches it. Same
  conclusion (Terraform rejects null/unknown block elements; the framework currently turns them into known objects).
  #993 (WebFetch) concerns nested attributes only; fixed by #995 (v1.9.0).
- exhaustive x4: correct false positives. NestingModeUnknown/BlockNestingModeUnknown (= 0) are only declared in
  internal/fwschema; every production GetNestingMode returns a constant (grep: 62 constant returns); only
  internal/testing/testschema returns a configurable mode. Each switch has an error-reporting default arm.

## New item found while checking (B1-15)

SingleNestedAttribute/SingleNestedBlock object validators run twice. Probe $V/probes/zz_vb1_singlevalidators_test.go
(known values): attribute 2 calls, block 2 calls; $V/probes/zz_vb1_singlevalidators_null_test.go (null values):
attribute 1 call (guarded arm), block 2 calls. Cause: GetNestedObject() of 13 single-nested types (action, datasource,
ephemeral, list, provider, resource schema packages x attribute+block, statestore attribute) copies Validators into
the nested object; fwserver runs them as attribute/block validators and again in NestedAttributeObjectValidate /
NestedBlockObjectValidate. Identical diagnostics are deduplicated by diag.Diagnostics.Append (B1-3 probe shows
"object validator error" once for blk_single_err). PlanModifiers are not copied, so plan modification runs once.
No upstream issue found (three GitHub issue searches); CHANGELOG has no rationale.

## Not equivalent (checked)
- generic message concatenation and func-value indirection: output-identical, allocation profile not (numbers above).
- the author's six rejected rewrites: endorsed by reading (2, 3, 5, 6) or by probe (4: #740 behaviour reproduced).
