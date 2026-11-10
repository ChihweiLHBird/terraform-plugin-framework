# Verifier notes, area E (types, types/basetypes, attr, attr/xattr, path, diag, tfsdk)

Verifier: independent (did not write findings-E). Repo /home/user/terraform-plugin-framework at 00badb1, untouched
(`git status --short | wc -l` = 0 before and after). Toolchain: `GOTOOLCHAIN=auto go version` -> go1.26.0 linux/amd64.
Scratch: drafts/V-E/ (paths below are relative to .../scratchpad/audit/work/V-E):
- repo/   : `git archive HEAD | tar -x` snapshot of 00badb1 (never edited; probes replace -> ../repo)
- dN/     : full copies of repo/ with one draft change each (d1 = E1, ...)
- probe/  : probe module `vprobe` (go.mod: replace github.com/hashicorp/terraform-plugin-framework => ../repo)
Session was interrupted once (rate limit + machine restart) after E1; files survived, work resumed at E2.

## E1 (%s on *big.Float) -- CONFIRMED high
- Pinned Go: $GOROOT/src/math/big/ftoa.go:458 Format(): verbs e,E,f,b,p,x,F,v,g,G handled; `default:` prints
  `fmt.Fprintf(s, "%%!%c(*big.Float=%s)", format, x.String())`. So %s is unsupported and the inner text is exactly
  x.String() == x.Text('g', 10).
- Sites: `grep -rn 'Value %s' --include=*.go types/basetypes | grep -v _test.go` -> 13 (float64_type.go 103,114,151,157;
  float32_type.go 93,99; int32_type.go 82,88,92; int64_type.go 98,109,142,148). All 13 opened: each passes a *big.Float.
- Tests pinning the garbled text: `grep -rn 'cannot be represented\|is not an integer' --include=*_test.go .` -> 9, all in
  types/basetypes (3 literal "%!s(*big.Float=...)" in int32_type_test.go:51,55,59; 6 computed with the same %s Sprintf in
  float64_type_test.go:72,88,176,186 and float32_type_test.go:90,100). `grep -rn '%!s(\*big.Float'` -> only the 3 literals.
  No non-Go file (website docs) contains the text.
- probe/bigfmt (Go 1.26.0):
    in=80.5            %s="%!s(*big.Float=80.5)"            String()="80.5"            %v="80.5"
    in=3000000000      %s="%!s(*big.Float=3000000000)"      String()="3000000000"      %v="3e+09"
    in=2^63            %s="%!s(*big.Float=9.223372037e+18)" String()="9.223372037e+18" %v="9.223372036854775808e+18"
    in=2147483647.5    %s="%!s(*big.Float=2147483648)"      String()="2147483648"      %v="2.1474836475e+09"
    in=12345678901.5   %s="%!s(*big.Float=1.23456789e+10)"  String()="1.23456789e+10"  %v="1.23456789015e+10"
  Int32/Int64 ValueFromTerraform(80.5) -> "Value %!s(*big.Float=80.5) is not an integer."; Int64Type.Validate and
  Float64Type.Validate put the same garbled text in the diagnostic Detail.
- probe/e2e: providerserver.NewProtocol6WithError + ValidateResourceConfig, Int64Attribute port=80.5 ->
  "Configuration Read Error ... Error: Value %!s(*big.Float=80.5) is not an integer."; Int32Attribute 3000000000 ->
  "... Value %!s(*big.Float=3000000000) cannot be represented as a 32-bit integer."
- Draft d1 (sed: add .String() at the 13 sites, 13 lines changed): `go test ./types/basetypes/ -run
  'TypeValueFromTerraform|TypeValidate'` -> exactly 9 failing subtests in 4 tests (TestFloat64TypeValidate x2,
  TestInt32TypeValueFromTerraform x3, TestFloat32TypeValueFromTerraform x2, TestFloat64TypeValueFromTerraform x2).
  internal/reflect, internal/fwschemadata, function, tfsdk pass on d1.
- Caveat to add (not a verdict change): String() is 10 significant digits, so a fractional value with more than 10
  significant digits renders as an integer: 2147483647.5 -> "Value 2147483648 is not an integer."; 12345678901.5 ->
  "Value 1.23456789e+10 is not an integer.". This is already the text inside today's wrapper, so .String() is the
  minimal fix as claimed; for the two "is not an integer" sites (and arguably all), Text('g', -1)/%v prints the exact
  value at the cost of "3e+09" style for round integers. Implementer's choice; record it.
- TestInt64TypeValueFromTerraform has only the String-type error case today (confirmed), so "add Int64 cases" is right.

## E2 (Path.Equal / Expression.Equal asymmetry) -- CONFIRMED high
- Read path/path.go:105-120 and path/expression.go:150-169: `if steps == nil && o.steps == nil {true}; if steps == nil
  {false}; return steps.Equal(o.steps)` (Expression checks root first). PathSteps/ExpressionSteps.Equal are len-based.
- Exhaustive static caller list (compile trick: copy repo to rn/, rename Path.Equal/Expression.Equal to EqualRENAMED,
  iterate `go build ./...` patching each reported call, tools/iter_rename.py): 9 non-test sites, exactly the finding's
  list: path/paths.go:34,53; path/expressions.go:35,66; diag/with_path.go:27; resource/import_state.go:104,129,137;
  internal/fwserver/server_planresourcechange.go:563 (NormaliseRequiresReplace dedup). Plus reflective use by go-cmp
  (cmp.Diff calls x.Equal(y)) in tests.
- Nothing else distinguishes nil from empty steps: `grep -rn 'steps == nil|steps != nil'` -> only the 4 lines in the two
  Equal methods; totftypes.AttributePath uses fw.Steps() (PathSteps{} for both). No non-test code outside path/ builds
  `path.Path{}`/`path.Expression{}` (grep: 0); the only in-package zero-value use is Expressions.String's
  `path.Equal(Expression{})`, where the zero value is the ARGUMENT (unaffected: receiver-empty/arg-nil already true).
- probe/pathequal (before): Path{}.Equal(Empty())=false, reverse=true; Path{}.Equal(Path{}.Copy())=false;
  Path{}.Equal(Root(a).ParentPath())=false; Expression{}.Equal(MatchRelative())=false, reverse=true;
  Diagnostics.Append(zeroPathDiag, emptyPathDiag) len 2 vs reversed len 1; Paths{Path{}}.Contains(Empty())=false vs
  Paths{Empty()}.Contains(Path{})=true; Paths{Root(a),Path{}}.String()="[a,]";
  ImportStatePassthroughID(ctx, path.Path{}, ID "abc") -> "Value Conversion Error" (guard skipped);
  ImportStatePassthroughWithIdentity(Path{},Path{}) -> only "Value Conversion Error" (both guards skipped).
- Draft d2 = the two one-line bodies from the finding. Measured diff: path.go -13/+1, expression.go -17/+1 (= -30/+2,
  as claimed). gofmt clean.
- Before/after Equal matrix (probe/pathmatrix vs probe_d2/pathmatrix; 13 paths x 13 + 16 expressions x 16 = 425 pairs
  incl. zero, Empty, Copy, ParentPath, AtParent().Resolve(), Merge, AtName on zero): exactly 10 rows change, all
  receiver=zero-value vs argument=non-nil-empty, false->true. Asymmetric pairs: 10 before, 0 after. Every other result
  identical, so the "behaviour-preserving for all other inputs" claim holds. Step Equal calls (attr.Value.Equal inside
  PathStepElementKeyValue) happen in the same order and number for non-nil cases; none for the changed case.
- probe_d2/pathequal (after): all symmetric; Append keeps 1 either order; ImportStatePassthroughID(Path{}) ->
  "Resource Import Passthrough Missing Attribute Path"; WithIdentity -> both guard diagnostics.
- Tests on d2 (`go test -p 2`): path, diag, tfsdk, internal/fwserver, fwschemadata, fwschema, proto5server, proto6server,
  totftypes, fromtftypes, reflect, list, function, resource/schema/... (26 pkgs), datasource/schema, provider/schema: all
  ok. `resource` has no test files: ImportStatePassthroughID/WithIdentity guards are untested today.
- Reliance on the asymmetric answer: none in the repo (above). go-cmp can only flip a currently-unequal pair to equal,
  so no currently-passing equality assertion can break; only an assertion of INequality on a nil-vs-empty pair could,
  and none exists (tests pass). Downstream reliance would be on `zero.Equal(Empty()) == false` while
  `Empty().Equal(zero) == true` already: implausible but possible, so it is a public behaviour change -> bug fix with
  a CHANGELOG entry (as the finding says), not a refactor.
- Tier: all user-visible consequences need provider code to pass a zero-value path.Path/Expression (legal Go zero
  value, but the framework never builds one). Not tier 1; tier 2 is defensible ("wrong output": misleading diagnostic,
  order-dependent dedup) though tier 3 (latent) would also be fair. Kept 2.

## E3 (nil-receiver Set panics) -- CONFIRMED high
- Read tfsdk/resource_identity.go:45-59 (Set, no guard) and :72-81 (SetAttribute guard "Missing Identity Definition");
  tfsdk/resource.go:47-61 and :74-83 ("Missing Resource Definition"). Only these two tfsdk types have the guard
  (`grep -n '== nil {' tfsdk/*.go`), and they are the only tfsdk types handed to providers through pointer-typed fields:
  resource Create/Read/Update/Delete/ModifyPlan/ImportState/UpgradeIdentity *Response.Identity, MoveStateResponse
  .TargetIdentity, list.ListResult.Identity/.Resource (`grep -rn '\*tfsdk\.ResourceIdentity\|\*tfsdk\.Resource\b'`).
- Reachability (the question to attack): fwserver allocates resp.Identity only `if req.<X>Identity != nil` after
  pre-populating a null identity `if ... == nil && req.IdentitySchema != nil` (server_createresource.go:104-124, same
  shape in read:134-149, update:127-143, importresourcestate:152-167, planresourcechange:122-133/335-340, delete:117/140).
  So the provider's resp.Identity is nil exactly when the resource has no identity schema. list.NewListResult always
  allocates both; nil Resource needs a ListResult literal (field doc anticipates nil: "a nil value will raise a warning").
- No panic recovery on the RPC path: `grep -rn 'recover()'` in terraform-plugin-go@v0.31.0 and the framework (non-test): 0.
- probe/nilset: unit level nil *ResourceIdentity/*Resource SetAttribute -> diag, Set -> "runtime error: invalid memory
  address or nil pointer dereference". End to end (providerserver.NewProtocol6WithError -> ApplyResourceChange create,
  resource WITHOUT identity schema, Create calls resp.Identity.Set): process dies, SIGSEGV, exit 2, stack
  tfsdk.(*ResourceIdentity).Set resource_identity.go:49 <- fwserver CreateResource :127 <- proto6server. With
  resp.Identity.SetAttribute instead: ERROR "Missing Identity Definition", returns normally.
- History: the SetAttribute guard on Resource came with 0cdd87d (#1205) with the comment "If r is nil, then calling
  r.data triggers a nil pointer error"; Set predates it and never got the guard.
- Draft d3 (guard copied from SetAttribute, detail "set the resource identity"/"set the resource"): +10/-0 per file
  (+20 total, finding wrote "+10/+10"); gofmt/vet clean; tfsdk, list, internal/fwserver tests pass; probe_d3/nilset:
  Set on nil returns the diag, e2e returns ERROR "Missing Identity Definition" and no crash.
- No test covers either existing guard (grep summaries in *_test.go: 0).
- Tier 2 justified: crash, but only through provider misuse (Set on a resource without identity schema / literal
  ListResult); verifier rule says misuse-only is not tier 1.

## E4 (quadratic copies in SetValue.Equal, MapValue.String, ObjectValue.String) -- CONFIRMED high
- Lines confirmed: set_value.go:309 (contains ranges s.Elements()), map_value.go:345,346,359, object_value.go:377,378,391.
  `grep -rn "\.Elements()\|\.Attributes()" types/basetypes/*.go | grep -v _test.go` -> 10 (3 per-element, 4 extra whole
  copies in the two String methods, 3 single copies in list/set/tuple String) as stated.
- Elements()/Attributes() copy (append / make+loop) and are called statically on the concrete receiver (no dispatch:
  an embedding downstream type cannot intercept a call made inside SetValue's own method), so indexing the field is the
  same data in the same order.
- Draft d4 (7 sed edits): -7/+7 measured; `go test ./types/basetypes/` ok.
- probe/e4bench vs probe_d4/e4bench: TestFingerprint hashes String()/Equal() (both directions) for sets in forward and
  reverse order, maps with quotes/Unicode values, objects, sizes 0,1,2,7,50, null/unknown, nested object{map,set}:
  identical sha256 250d71a1...cad0b before and after (String output unchanged).
- Benchmarks (b.Loop, 20x, 4 shared cores) before -> after:
    SetValue.Equal n=100   169 us, 179 KB, 100 allocs   -> 43 us, 0 B, 0 allocs
    SetValue.Equal n=1000  9.6 ms, 16.4 MB, 1000 allocs -> 3.4 ms, 0 B, 0 allocs
    SetValue.Equal n=3000  78.6 ms, 147 MB, 3003 allocs -> 35.5 ms, 0 B, 0 allocs
    MapValue.String n=1000 48.9 ms, 82 MB, 10154 allocs -> 0.53 ms, 188 KB, 5018 allocs
    ObjectValue.String n=1000 50.1 ms, 82 MB, 9884 allocs -> 0.38 ms, 139 KB, 4752 allocs
  (author: MapValue.String ~83 ms at 1000; same order, machine noise.) SetValue.Equal stays O(n^2) in comparisons;
  only the O(n^2) allocation goes.
- Call sites cited: fwschemadata/value_semantic_equality.go:90, fwserver/schema_semantic_equality.go:90,136 (Equal);
  PathStepElementKeyValue.String embeds Value.String(); 7 `FrameworkWithAttributePath(ctx, ....String())` sites
  (eager, regardless of log level). All verified.

## E5 (generic helpers for primitive value state machine) -- CONFIRMED high
- Inspected the author's draft (drafts/E/draft) and copied ONLY its 7 changed files into my own copy d5 (diff -rq of
  drafts/E/draft vs my snapshot: only bool/int32/int64/string/float32/float64_value.go differ + new primitive_value.go).
- Size: `diff` per file: bool/int32/int64/string -41/+4 each, float32/float64 -14/+1 each, primitive_value.go +70 ->
  -192/+88 exactly as claimed (net -104).
- Exported API: `go doc -all ./types/basetypes` before vs after: identical (1753 lines) -> same exported identifiers,
  signatures, method sets, receiver kinds (all value receivers), doc comments. `go doc -all ./types` identical.
  Struct declarations of the 6 types byte-identical (unexported state/value fields unchanged -> reflect/%#v/embedding
  unaffected). IsNull/IsUnknown are `return x.state == attr.ValueStateNull/Unknown` in all 6, so String's direct state
  checks equal the old b.IsUnknown()/b.IsNull() calls (static calls on the concrete receiver, no dispatch).
- Values passed to tftypes: the helper takes `value any`; the old code boxed the same field at the call
  (bool/int32/int64/string, and *big.Float for floats, where a nil pointer boxes to the same non-nil interface).
- Behaviour dump (tools/zz_verify_e5_test.go, internal test placed in baseline copy b5 and in d5, then removed):
  68 values = 6 types x states {null, unknown, known, invalid 3, invalid 255} x 2 values (incl. "", "é\"\n", 1<<62,
  nil *big.Float) + zero values + Number/Dynamic; for each: ToTerraformValue (value, err, panic type+text), String,
  IsNull/IsUnknown, Value<T>Pointer (nil-ness, distinct per call, mutation does not alias the original),
  68x68 Equal matrix, Equal(nil). Output identical (273 lines), e.g. invalid state:
  "PANIC(string): unhandled Bool state in ToTerraformValue: %!s(PANIC=String method: unhandled ValueState in String: 3)".
- Tests on d5: types/..., tfsdk, internal/reflect, internal/fwschemadata, function: ok. gofmt, go vet clean.
  golangci-lint 2.15.0 (audit/bin, built with go1.26) with the project .golangci.yml on ./types/basetypes/: 0 issues.
  (/usr/local/bin/golangci-lint 2.5.0 refuses to load the config: built with go1.25.)
- Characterization test first? Reachable behaviour is already pinned: each of the 6 types has Test<T>ValueToTerraformValue,
  Equal (with cross-type candidates such as NewStringValue), String and Value<T>Pointer tests (24 functions). The only
  unpinned branch is the invalid-state panic, unreachable from outside the package (state is unexported, every
  constructor sets a valid state). So the suggested internal panic test is cheap and nice-to-have, not a prerequisite;
  my dump already shows the text is identical. Tier 2 stays (maintainability judgement on the most-read public files),
  not because proof is missing.

## E6 (Float32 debug log condition inverted) -- CONFIRMED high
- float32_type.go:82-88: logs "big.Float value has distinct float32 and float64 representations" iff
  accuracy == Exact && f64accuracy == Exact, i.e. iff float32(x) == x == float64(x).
- probe/f32log (tfsdklogtest.RootLogger + tfsdklog.NewSubsystem("framework", WithLevel(TRACE)), Float32Type{}.
  ValueFromTerraform):
    1.5 (float64 on the wire)        acc32=Exact acc64=Exact equal=true  LOGGED "(float32 value: 1.500000, float64 value: 1.500000)"
    3                                 Exact/Exact equal=true  LOGGED
    0.1 (float64 0.1)                 Above/Exact equal=false not logged
    0.1 (decimal string, 512-bit; how cty/msgpack sends non-float64-exact numbers) Above/Above equal=false not logged
    16777217 (2^24+1)                 Below/Exact equal=false not logged
  So the line is false whenever emitted and absent when float32 loses precision. Tests pinning it: 0.
- Fix `if float64(f) != f64` logs 0.1 (both wire forms) and 2^24+1, not 1.5/3. Note: the message formats with %f, so
  0.1 prints "float32 value: 0.100000, float64 value: 0.100000"; consider %g/%v when touching the line (optional).
- Tier 3 (DEBUG log only) right. Size -1/+1.

## E7 (Paths/Expressions.String stray comma) -- CONFIRMED high
- path/paths.go:47-67 and path/expressions.go:60-80 read; separator keyed on the element index, skip test
  `path.Equal(Empty())` / `path.Equal(Expression{})`.
- Draft d7 (collect parts, strings.Join, same skip test): -12/+4 per file = -24/+8 (as claimed); path tests pass; gofmt ok.
- Exhaustive matrix (probe/pathsstring vs probe_d7): every sequence of length 0..4 over {Empty(), Path{}, Root("a"),
  Root(""), Root("b")[0]} and {Expression{}, MatchRelative(), MatchRoot("a"), MatchRoot(""), Empty().Expression()}:
  1,563 outputs, 434 change, 0 outside the predicted class "element 0 skipped and some later element not skipped"
  (checked by script). Examples: P[Empty,Root(a)] "[,a]" -> "[a]"; P[Empty,Empty,Empty,Path{}] "[,]" -> "[]".
- Still "[,,a]" for Paths{Path{}, Path{}, Root(a)} until E2 lands (Path{} not skipped, renders ""), and "[,a]" for
  Expressions{Empty().Expression(), MatchRoot(a)} (root-only expression is not Expression{}): the author's open edge.
- Author's rejected alternative (`result.Len() > 1`) correctly rejected: Paths{Root(""), Root("a")} is "[,a]" today
  (Root("") renders "" but is not skipped); Len-based separators would print "[a]". Collect+join keeps "[,a]".
- Consumers: renaming Paths.String/Expressions.String in a copy (rn2) still builds -> no static non-test caller; no
  fmt use in non-test code. Visible only in provider/validator messages (e.g. external validators format
  path.Expressions with %s/%q). Doc: "not protected by compatibility guarantees". Tier 3 right.

## E8 (ObjectValue.Equal ignores attribute types for null/unknown) -- PLAUSIBLE medium
- object_value.go:301-352 read. Doc (verbatim): "Equal returns true if the given attr.Value is also an ObjectValue, has
  the same value state, and contains exactly the same attribute types/values as defined by the Equal method of those
  underlying types/values." Code returns true at :315-317 for two null or two unknown objects before comparing types.
  TupleValue.Equal has the same doc wording ("has the same value state, and contains exactly the same element
  types/values") and compares element types BEFORE state; List/Map/Set compare element types before state too.
  attr.Value.Equal contract: "true if the Value is considered type and data value equivalent".
- TestObjectValueEqual: null-null and unknown-unknown cases use identical attribute types only.
- probe/objequal (before -> after draft d8 = block move, -8/+8, gofmt ok):
    ObjectNull(a).Equal(ObjectNull(b)) true -> false; ObjectUnknown(a)/(b) true -> false; ObjectNull(a).Equal(ObjectNull(nil))
    true -> false; List/Map/Set/TupleNull with different element types false both before and after;
    DynamicValue(ObjectNull(a)).Equal(DynamicValue(ObjectNull(b))) true -> false.
- NEW consequence the finding's risk text misses (probe/objreplace, non-null resource State/Plan so the early returns
  are skipped): built-in plan modifiers decide replacement with PlanValue.Equal(StateValue):
    dynamicplanmodifier.RequiresReplaceIf(always) for dyn(null obj{a}) -> dyn(null obj{b}):   false -> TRUE
    same for unknown objects:                                                                   false -> TRUE
    objectplanmodifier.RequiresReplace() for null obj{a} -> null obj{b}:                        false -> TRUE
    (dyn(null list[string]) -> dyn(null list[bool]) is already TRUE before: the inconsistency is real.)
  So the fix can turn a no-op plan into a replacement for a dynamic (or mistyped object) attribute whose null/unknown
  object type changes. Plausible after a provider changes an object shape inside a dynamic attribute.
- Precedent for the lenient form: Terraform's cty (go-cty v1.16.2, fetched into an isolated scratch module ctyprobe/):
  cty.NullVal(obj{a}).Equals(cty.NullVal(obj{b})) = cty.True (also for lists), RawEquals = false. The framework chose
  the strict form for List/Map/Set/Tuple and the lenient one for Object (and Dynamic delegates).
- Tests on d8: basetypes, fwschemadata, reflect, fwserver, proto6server, fromproto6, toproto6, objectplanmodifier,
  dynamicplanmodifier, objectdefault, function: ok (no test covers the case either way).
- Verdict: the code/doc/sibling inconsistency is confirmed, but whether the code or the doc is wrong is a maintainer
  decision (cty precedent; no wrong user-visible outcome demonstrated for today's behaviour), and the fix itself
  changes replace planning. PLAUSIBLE medium, tier 3; risk must name the RequiresReplace consequence; offer
  "fix the doc" as the zero-risk alternative.

## E9 (maps.Copy x4, slices.Sorted(maps.Keys) x3) -- CONFIRMED high
- Lines confirmed: map_value.go:177 (Elements loop), :345-349 (String keys + sort.Strings :349); object_type.go:46
  (AttributeTypes loop), :148-152 (String keys); object_value.go:238 (Attributes), :250 (AttributeTypes), :377-381.
  The 4 loops are exactly the gofix/golangci mapsloop leads (map_value.go:178, object_type.go:47, object_value.go:239,251).
- Stdlib in pinned Go 1.26.0: maps.Copy = `for k, v := range src { dst[k] = v }` (identical); slices.Sorted = Collect +
  slices.Sort; sort.Strings(x) is literally `slices.Sort(x)` in sort/sort.go:181. Same comparison, same order; keys are
  unique so stability is moot. Collect of an empty map returns nil, but `keys` is only ranged over.
- Draft d9 = E9 applied on top of d4 (E4): gofmt/vet clean, basetypes tests pass. Sizes: E4+E9 vs repo -34/+16
  (matches author); E9 alone vs d4 -31/+13 (author's "-27/+9" is the arithmetic difference; the 4 key-collection lines
  E4 edits are re-edited by E9 so they count in both diffs; net -18 either way). Imports: -sort +maps +slices in 3 files.
- probe/e9 vs probe_d9/e9: Attributes()/AttributeTypes()/Elements() nil-ness and length for null, unknown, known,
  ObjectNull(nil), empty, zero-value structs; mutation of the returned copy not visible; String()/Type().String() with
  keys "", "A", "a", "b", "é", `a"q`: identical (21 lines). E4 fingerprint test also identical on d9.

## Author's "looks equivalent but is not" entries (probes)
1. maps.Clone for MapValue.Elements/ObjectValue.Attributes: ENDORSED. maps.Clone(nil) == nil; NewMapNull/NewObjectNull
   leave the field nil. Drafted in dtrap/: TestMapValueElements/{null,unknown} and TestObjectValueAttributes/{null,unknown}
   FAIL (suite protects this one).
2. slices.Clone for List/Set/Tuple Elements(): ENDORSED. slices.Clone(nil) == nil (Go 1.22+ preserves nilness);
   drafted for ListValue: TestListValueElements/{null,unknown} FAIL.
3. Diagnostics.Errors via slices.DeleteFunc(slices.Clone(diags), ...): ENDORSED and the suite is BLIND: drafted in
   dtrap/diag, `go test ./diag/` passes. probe/traps: nil receiver -> current Diagnostics{} (json "[]"), rewrite nil
   (json "null"); cmp.Diff reports equal (uses Diagnostics.Equal, len-based), reflect.DeepEqual false.
4. new(12.3)/new(12) for typed helpers: ENDORSED. Go 1.26.0: new(12.3) is *float64, new(12) is *int,
   new(float32(12.3)) is *float32, new(new(int32(1))) is **int32; an interface{} holding new(12.3) is not *float32.

## E10 (five copies of the Must panic loop) -- CONFIRMED high
- `grep -rn '"%s | %s | %s"' --include=*.go . | grep -v _test.go` -> 5 (list:141, map:146, object:171, set:141, tuple:98);
  block ranges list 135-148, map 140-153, object 165-178, set 135-148, tuple 92-105 confirmed. Normalised diff of the 5
  functions: the 12-line block is identical; only the prefix differs ("NewListValueMust", "NewTupleValueMust" vs
  "MapValueMust", "ObjectValueMust", "SetValueMust") plus the "effect"/"affect" doc typo in map/object/set.
- My own draft d10 (helper diagnosticsPanicString in a new 28-line file; each Must -> one panic line with its own
  prefix): -60/+33 (matches). gofmt/vet clean; types/... tests pass.
- probe/must vs probe_d10/must: recovered panic values for List, Set, Map (2 bad elements -> 2 lines joined by \n),
  Object (wrong type + extra attribute), Object (2 missing), Tuple (length mismatch), Tuple (2 wrong elements), and a
  non-panicking call: byte-identical. `grep -rn 'ValueMust received error' --include=*_test.go .` -> 0 (unpinned).

## E11 (13 doc comments) -- CONFIRMED high
- Every site opened: object_value.go:66 ("type ElementsAs method"), :128, :157 ("Elements or ElementsAs methods"; the
  type has Attributes() and As(); types/object_value.go:29,36,43 already say "Attributes or As");
  number_value.go:156-157 ("returns 0.0" but returns n.value, nil for null/unknown; TestNumberValueValueBigFloat expects
  nil); float32_value.go:94 ("53 bits"; the method compares ValueFloat32() values -> 24-bit significand; copy of the
  Float64 doc at float64_value.go:100); `grep -rn 'Setting the deprecated' types/` -> 5 (float64 57,68; string
  48,59,70), and no Null/Unknown/Value fields exist; missing_value.go:53 ("always returns an error", returns
  tftypes.Value{}, nil; unexported placeholder, no test file -> doc fix is the right side); path/path.go:27
  (path.MatchRoot in the Path doc example; MatchRoot returns an Expression); diag/warning_diagnostic.go:40
  ("NewErrorDiagnostic returns a new warning ...").
- Draft d11 (8 files): -23/+8 measured (matches). gofmt clean.
- Optional extra (not a content error): "potentially effect practitioners" in the Map/Object/Set Must docs.

## E12 (BenchmarkSetTypeValidate on an empty set) -- CONFIRMED high
- set_value_test.go:66-89 helper (`make([]tftypes.Value, 0, elementCount)` then `for idx := range elements`: 0
  iterations), :91-113 six variants.
- Original (scratch snapshot, 200x, -benchmem): 10 -> 102 ns, 1 alloc; 1000 -> 166 ns; 1,000,000 -> 15 us, 160 KB/op =
  only the amortised setup allocation (the timer is never reset).
- Fixed (d12, make with length): 10 -> 10.5 us, 100 allocs; 100 -> 0.60 ms, 9,982 allocs; 1000 -> 72 ms, 24 MB,
  999,985 allocs; 10000 (run once, 1x) -> 7.8 s, 2.4 GB, 100,040,109 allocs per op.
- Correction: the finding drops only the 100000 and 1000000 variants; the 10000 variant also becomes impractical
  (7.8 s and 2.4 GB per op). Drop or skip it too (size then about -13/+1), and use b.Loop() (or b.ResetTimer()) so
  the setup is not timed.

## E13 (tfsdk/value_as_test.go typed pointer helpers) -- CONFIRMED high
- Recount (regex must allow digits: 'new[A-Z][A-Za-z0-9]*Pointer[A-Za-z0-9]*\('): 42 occurrences = 13 definitions
  (lines 24-81) + 2 in the commented-out case (167,168: newBigFloatPointer, which is not defined) + 27 live calls
  (112-183 pairs and 351). Matches.
- go fix newexpr flagged only the 6 bool/float64/string call sites (112,113,132,133,177,178) plus 6 wrapper defs:
  the float32/int32/int64 sites need explicit conversions, as the finding says.
- Float64 call arguments are 0.0 and 12.3 (floating constants), so new(0.0) is *float64. Keep the ".0": new(0) would be
  *int. newInt64Pointer(0) at :351 -> new(int64(0)).
- My own draft d13 first produced new(new(new(big.Float).SetUint64(n))) for newBigFloatPointerPointer: it compiled
  (fields are interface{}) and panicked in the test's **big.Float comparer ("interface {} is ***big.Float"). The
  finding's text new(new(big.Float).SetUint64(n)) is the correct form (the helper wraps an already-pointer value
  once). Worth stating in the issue: PointerPointer of BigFloat takes ONE extra new.
- Fixed draft: go vet ok, `go test ./tfsdk/` ok; TestValueAs subtest IDs+statuses identical before/after (21/21,
  ladder step 6); per-case dynamic types (t.Logf("%T") injected in scratch copies of both versions) identical for all
  21 cases (*float32, **float32, *int32, **int32, **big.Float, ...). Size -96/+27 (matches; includes deleting the
  commented-out case).

## Additional endorsements (my own, from drafting)
5. new(new(new(big.Float).SetUint64(n))) for newBigFloatPointerPointer: ***big.Float, compiles, fails only via the test
   comparer's type assertion. Correct form has one extra new (see E13).
6. Embedded generic core (author's #4): compile probe embedprobe/{today,embedded}: a downstream
   `type MyString struct { StringValue; Other }` where Other reaches IsNull at depth 2 compiles today (prints false) and
   fails with "ambiguous selector MyString{...}.IsNull" when StringValue gets IsNull through an embedded core.
7. E7 Len-based separator (author's #11) and E2 via Copy() (author's #12): endorsed by the E7 matrix (Root("") case
   keeps "[,a]") and by construction (Empty() has non-nil steps whatever Copy does).

## Cross-area observation (not an area E item)
- probe/fnarg: provider-defined function with function.Int64Parameter called with 1.5 through
  providerserver.CallFunction returns: "Unable to Convert Function ArgumentAn unexpected error was encountered when
  converting the function argument from the protocol type. Please report this to the provider developer:\n\nUnable to
  convert tftypes to framework type at position 0: Value %!s(*big.Float=1.5) is not an integer." Two bugs in one
  message: E1 (garbled value) and the missing ": " at internal/fromproto5|6/arguments_data.go:101 (siblings at :71/:86
  have it). The separator bug is ALREADY in findings-A.json (A's proof used testtypes.InvalidType); this probe shows it
  is reachable with a stock Int64Parameter and a fractional argument, which strengthens A's item. Not added as new.
- E3 extra paths checked: MoveResourceState TargetIdentity is allocated iff the target has an identity schema
  (server_moveresourcestate.go:156-161); UpgradeResourceIdentity always allocates (:160). E3 stays misuse-only.
- E1 sweep: every non-test fmt call with %s/%q in files that use big.Float was listed; none formats a *big.Float
  outside the 13 E1 sites (reflect/number.go:43 already uses result.String()).

## Lint of every draft (audit/bin/golangci-lint 2.15.0, project .golangci.yml, one package each)
d1 types/basetypes (E1), d2 path (E2), d3 tfsdk (E3), d7 path (E7), d8 types/basetypes (E8), d9 types/basetypes
(E4+E9), d10 types/basetypes (E10), d13 tfsdk (E13), d5 types/basetypes (E5): 0 issues each.

## Verdicts
CONFIRMED: E1 E2 E3 E4 E5 E6 E7 E9 E10 E11 E12 E13 (all high). PLAUSIBLE: E8 (medium). INVALID: none.
Number corrections: E3 size -0/+20 (10 per file); E9 size -31/+13 when applied after E4 (net -18; combined -34/+16
unchanged); E12 replacement must also drop the 10000 variant (size about -13/+1). No tier changes (E2 could be argued
down to 3; E8 stays 3 but as PLAUSIBLE with a stronger risk statement).

## Repository check
`git -C /home/user/terraform-plugin-framework status --short | wc -l` -> 0 at the end; HEAD still 00badb1. All drafts,
probes and test runs were in work/V-E (replace directives pointed at scratch copies, never at the live repo).
