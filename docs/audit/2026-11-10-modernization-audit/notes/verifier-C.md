# Verifier notes, area C (internal/fwschemadata, internal/reflect, internal/fwschema, fwxschema, fwtype, fwfunction, privatestate)

Verifier V-C. Repository /home/user/terraform-plugin-framework at 00badb1, untouched (`git status --porcelain` empty before
and after). All scratch work is under `audit/work/V-C/` (absolute: drafts/V-C/).

Layout of work/V-C:
- `orig/`: pristine copy of the tracked files (`git ls-files | cp --parents`), diff -rq against the repo is empty.
- Draft copies, one per item: `fix1/` (author's C1 draft), `fix1b/` (my corrected C1 fix), `fix2/` (my gated C2 fix),
  `fix3/` (C3 list/set re-validation removed), `fix4/` (C4 import of fwschema, does not compile), `c10/`, `c11/`, `c12/`,
  `c13/` (= c10 + C13), `c14/`, `c15/`, `c16/` (C15+C16 combined), `c17/`, `c18/`, `c19/`, `c20/`.
- `probe/`: my own public-API probe module (`replace github.com/hashicorp/terraform-plugin-framework => /home/user/terraform-plugin-framework`),
  tests c1_test.go, c1b_test.go, c2_test.go, c2b_test.go, c3_test.go, c3b_test.go, c3c_test.go, c4_test.go, c5_test.go,
  c6_test.go, c6b_test.go, c7_test.go, c8_test.go. `probe-<draft>/` are the same module re-pointed at a draft copy.
- `bench/`: public-API benchmarks (State.Get/Set of a 10-field struct, Get of one int64 field).
- `convprobe/`: stdlib probes (reflect.Convert same-kind exactness, TypeFor vs TypeOf(big.NewX) allocs, DeepEqual vs maps.EqualFunc).
- `extract_blocks.py`, `blocks/`: my own extraction and normalisation of the 24 validation blocks.
- `ext/provider.go`: Terraform core internal/configs/provider.go (main branch) fetched from raw.githubusercontent.com for C8.

Commands: run probes with `cd drafts/V-C/probe && GOTOOLCHAIN=auto GOFLAGS=-mod=mod go test -count=1 -run TestCx -v .`.

## Per item

### C1 (named numerics / map keys panic in Into): CONFIRMED, fix needs correction
Public probe (TestC1), original code:
```
State.Get working model        diags=0 {Port:8080 Ratio:0.5 Name:n Flag:true Ports:[1 2] Labels:map[a:x]}
State.Get named int field      PANIC: reflect.Set: value of type int64 is not assignable to type vcprobe.Port
Config.Get named int field     PANIC: (same)
State.Get named float field    PANIC: reflect.Set: value of type float64 is not assignable to type vcprobe.Ratio
State.Get []Port element       PANIC: reflect.Set: value of type int64 is not assignable to type vcprobe.Port
State.Get map[Key]string       PANIC: reflect.Value.SetMapIndex: value of type string is not assignable to type vcprobe.Key
State.Get *Port / GetAttribute *Port / ElementsAs []Port   PANIC (reflect.Set)
GetAttribute map[int]string    PANIC: reflect.Value.SetMapIndex: value of type string is not assignable to type int
State.Set named int model      diags=0          (Set direction works)
SetAttribute map[Key]string    diags=0;  SetAttribute map[int]string -> diag "map keys must be strings, got int"
GetAttribute empty map into map[int]string   diags=0 map[] nil=false   (works today)
```
No recover() anywhere in non-test code: a panic in an RPC kills the provider process.

Documentation: the in-repo website docs were removed by d2548a9 "chore: remove old docs (#1212)" (moved to the unified docs
site). In d2548a9^: types/{int32,int64,float32,float64,number}.mdx: "Numbers can be automatically converted from the following
Go types, pointers to these types, or any aliases of these types, such `type MyNumber int`"; types/object.mdx: "Objects can be
automatically converted to and from any Go struct type ... Every struct type must be an acceptable conversion type according to
the type documentation"; types/map.mdx: "A Go built-in map of string key type (`map[string]T`) or type alias of a map of string
key type such as `type MyMapType map[string]T`". Code docs: number.go:22-25 says "any of the built-in int, uint, or float types".
TestPrimitive_stringAlias / boolAlias pin named string/bool reads. Conclusion: named numeric struct fields are a documented
(implied) model shape, so tier 1 is justified for them; named-string keys and non-string keys are undocumented, so for maps
the defect is "panic instead of a diagnostic".

Draft (author's diff applied to `fix1/`): go test ./internal/reflect ./tfsdk ./types/basetypes pass; the probe shows all panics gone,
BUT:
```
(probe c1b_test.go, original -> author's draft)
GetAttribute map[any]string        diags=0 map[a:x]   -> diags=1 map[]
GetAttribute map[KeyIface]string   diags=0 map[a:x]   -> diags=1 map[]   (type KeyIface interface{})
ElementsAs map[any]types.String    diags=0 map[a:"x"] -> diags=1 map[]
GetAttribute empty map into map[int]string  diags=0 map[] nil=false -> diags=1 nil=true
```
So the draft's `Key().Kind() != reflect.String` check changes inputs that work today. Corrected fix in `fix1b/`
(diff saved as drafts/V-C/c1-corrected-map.diff): check only when `len(values) > 0 && !string assignable to key && key kind != String`;
in the loop keep `reflect.ValueOf(key)` when string is assignable to the key type, else Convert. With fix1b all four rows above
match the original, every panic is gone, tests pass, golangci-lint 0 issues. Size: +31/-14 including the author's no-op
`*big.Int` Convert, so about -13/+30 without it.
Convert exactness (convprobe TestConvertSameKind): int8 min/max -> named int8, uint16 max, float32 max/smallest, int min, uint max:
bit-identical; identity Convert of int64 returns the same type; CanSet/CanAddr false as with reflect.ValueOf. Each of the 12
returns converts from the predeclared type of the same Kind as the target, so Convert can only relabel the type; it cannot truncate.
Plain int64 (identity) and named string/bool (Primitive, untouched) are unaffected.

### C2 (nil *big.Int and typed-nil interface pointers panic in FromValue): CONFIRMED, fix needs narrowing
Probe TestC2: `Get` of a null number into `*big.Int` -> nil, diags=0; `State.Set(&m)` -> PANIC nil pointer dereference;
`SetAttribute(count, (*big.Int)(nil))` PANIC; `SetAttribute(id, (*types.String)(nil))` PANIC "value method ...StringValue.Type
called using nil *StringValue pointer"; non-nil `*types.String` works; `ListValueFrom(NumberType, []*big.Int{nil})` PANIC;
`[]*big.Float{nil}` -> [<null>]; Get into a `*types.String` field -> type-mismatch diagnostic.
Attack on the fix: TestC2NilSafe, pointer-receiver Unknownable whose GetUnknown returns true for nil:
`ListValueFrom(StringType, []*nilSafeU{nil})` = [<unknown>] today, no panic. The proposed "nil pointer -> FromPointer before the
interface dispatch" would return <null>. Gated fix in `fix2/` (+18 lines): FromBigInt(nil) -> FromBigFloat(nil); in FromValue
route a nil pointer to FromPointer only when its ELEMENT type implements attr.Value/ValueCreator/Unknownable/Nullable (value
receivers, so the call is guaranteed to panic today). Result: round trip diags=0, (*types.String)(nil) -> null, nilSafeU still
<unknown>, nil *big.Float still handled by FromBigFloat; reflect/tfsdk/basetypes tests pass. (A mixed type, value-receiver
Unknownable plus pointer-receiver attr.Value, would still differ; gate on the first interface in dispatch order to be exact.)

### C3 (double validation; set elements at index paths): CONFIRMED bug, fix has a dependency hole
Probe TestC3 (vStrType: custom string type whose value ValidateAttribute warns, errors on "bad", counts calls):
```
SetValueFrom {a,b}        calls=4 n=4 [Warning @ [0]] [Warning @ [Value("a")]] [Warning @ [1]] [Warning @ [Value("b")]]
SetValueFrom {bad}        calls=1 n=1 [Error "bad value" @ [0]]
ListValueFrom {a,b}       calls=4 n=2 [Warning @ [0]] [Warning @ [1]]
MapValueFrom {k:a}        calls=2 n=1;  ObjectValueFrom {s:a} calls=2 n=1
State.SetAttribute set {a}   calls=2 n=2 [Warning @ tags[0]] [Warning @ tags[Value("a")]]
State.SetAttribute set {bad} calls=1 n=1 [Error @ tags[0]]
State.SetAttribute name a    calls=2 n=1
SetValueFrom set(object{s: vStrType}) calls=2 n=1 [Warning @ [0].s]   (nested paths also index-based)
```
Coverage (orig, go test -coverprofile ./internal/reflect): uncovered blocks include map.go:191,199; slice.go:275,283,365,373;
struct.go:289,297, i.e. the second-pass early returns, as claimed.
Hole: FromUnknownable (both branches) and FromNullable (null branch) discard their validation diagnostics (C6), so the loop
re-validation and the SetAtPath re-validation are the ONLY source of those warnings today. Probe TestC3Unknownable (uStr:
pointer-receiver Unknownable, as the repo's own test types):
```
ListValueFrom []*uStr{a}   calls=2 n=1 [Warning @ [0]]       original
ListValueFrom []*uStr{a}   calls=1 n=0                         fix3 (list/set loop re-validation removed)
State.SetAttribute name *uStr{a}  calls=2 n=1                  original (the warning comes from SetAtPath's re-validation)
```
`go test ./internal/reflect/` still passes on fix3, so nothing pins this. C3 must land after C6 (or together).

### C4 (underscore tags): CONFIRMED (medium), replacement wrong
Probe TestC4: schema with "_internal": ValidateImplementation -> []; State.Get -> error "...error retrieving field names from struct
tags: _internal: invalid tfsdk tag, must only use lowercase letters, underscores, and numbers, and must start with a letter";
State.Set -> error; GetAttribute(_internal) into types.String works. TestIsValidAttributeName pins "_test" valid;
TestIsValidFieldName has no underscore-first case. CHANGELOG #705 entries confirmed (lines 490-492).
Replacement attack: `fix4/` makes helpers.go use fwschema.ValidAttributeNameRegex:
```
package .../internal/reflect
	imports .../internal/fwschema from helpers.go
	imports .../resource/schema/defaults from attribute_default.go
	imports .../types from bool.go
	imports .../types/basetypes from bool_type.go
	imports .../internal/reflect from list_value.go: import cycle not allowed
```
So the "no import cycle" claim is false; copy the pattern instead. Same applies to C11's note.

### C5 (random order of missing field names): CONFIRMED
Probe TestC5 (3 struct fields x1..x3 vs 3 schema attributes a1..a3), 500 runs each: State.Get 9 distinct Detail texts, State.Set 9,
types.ObjectValueFrom 9. The author's internal repro gave 8 and 9 over 200 runs in my rerun (random). TestCommaSeparatedString
expects "red and blue" for ["red","blue"], so sorting must happen at the 4 call sites (struct.go:105,108,220,224).

### C6 (dropped warnings): CONFIRMED, impact wording corrected
GetAtPath part, probe TestC6: `GetAttribute(name, &attr.Value)` n=0, `GetAttribute(name, &vStr)` n=1 warning.
Reflect part: lines interfaces.go:110,151 (`return res, nil`), :244, confirmed; author's repro TestReproDroppedWarnings reproduces.
Every public FromValue caller listed (`grep -rn 'FromValue(' --include=*.go`): basetypes List/Map/Set/ObjectValueFrom ->
FromSlice/FromMap/FromStruct loops re-validate; fwschemadata Data.Set -> FromStruct loop re-validates; SetAtPath re-validates;
tfsdk.ValueFrom drops all FromValue warnings anyway (new item C21); function ResultData.Set -> FuncErrorFromDiags logs warnings
with tflog.Warn. So the reflect-part drops are masked today on every path that returns diagnostics; they become visible only
when C3 removes the re-validation (or via function results, as a missing log line).

### C7 (DataDescription.Title): CONFIRMED
Probe TestC7: tfsdk.Resource.SetAttribute(missing) -> "Data Write Error"; Resource.GetAttribute(missing) -> "Data Read Error";
State.SetAttribute(missing) -> "State Write Error". DataDescriptionResource added in 9cafd72 (#1157, ListResource RPC).

### C8 (provider root blocks with resource meta-argument names): PLAUSIBLE, fix needs rework
Probe TestC8 (provider/schema ValidateImplementation): blocks count, lifecycle, provider, depends_on, connection -> 1 error each with
"When validating the resource or data source schema..."; blocks locals, source -> 0; attributes of all those names -> 0.
All 7 resource-like schema packages call IsReservedResourceAttributeName before ValidateBlockImplementation (grep), so removing the
inner call is a no-op for them (duplicate dropped by Append; ordering unchanged).
Terraform core (ext/provider.go, main): `providerBlockSchema` reserves attribute names count, depends_on, for_each, source and
block types lifecycle, locals ("Reserved argument name in provider block" / "Reserved block type name in provider block").
So a provider block named "lifecycle" is correctly rejected today (wrong message only); deleting the inner call would accept a
schema practitioners cannot configure. connection/provider/provisioner blocks are not reserved by core: false positives there.
provider/schema TestSchemaValidateImplementation "block-using-reserved-field-name" pins the resource message for "connection".

### C9 (privatestate typos): CONFIRMED, size corrected
data.go:68-69, 82-83, 163, 176, 380 and fwschema/schema.go:70-71 confirmed; `grep -c 'is is not valid\|a a period'
internal/privatestate/data_test.go` = 17, no other package. The missing-newline fix also changes the 4 expectations
`"An error was encountered when validating private state value."+` at data_test.go:55,70,121,138: 21 test lines, 7 data.go lines.

### C10 (validation-block helper): CONFIRMED
My own extraction (drafts/V-C/extract_blocks.py, output in blocks/): 24 blocks; after normalising switched value, path, type,
Validate argument and early-return value, 23 share md5 67e84229; FromAttributeValue (interfaces.go:458) differs only by the 4
lazy ToTerraformValue lines. Per-site names and returns match the author's table exactly (NewAttributeValue returns `target`,
every other site `nil`). No-error-on-entry invariant checked per site by reading the preamble of each block: every earlier error
path returns; loops HasError-check the FromValue result and return on ToTerraformValue errors.
Draft (author's diff on `c10/`): +111/-558 (git diff --shortstat), internal/reflect non-test 2858 -> 2411, nolint:staticcheck
24 -> 2, each of the 23 calls has the original arguments and the original early-return value; go vet, go test
./internal/reflect ./tfsdk ./types/basetypes ./internal/fwschemadata pass; golangci-lint (project config) on internal/reflect
0 issues; gocognit FromSlice 89->46, FromMap 44->17, FromStruct 49->32, FromUnknownable/FromNullable/FromPointer 25/25/16 -> <15.
Differential run of my whole probe suite (TestC1..TestC8) original vs c10: identical output (68 lines), incl. validator call
counts and diagnostic paths. Allocations identical (bench: Get10 480, Set10 719, GetInt64 75 allocs both).

### C11 (regexp per call): CONFIRMED, speed-up slightly optimistic for Set
Public benchmarks (bench/, 2000x, 3 runs): State.Get 10-field struct 480 -> 110 allocs, ~45-52 us -> ~14 us;
State.Set 719 -> 349 allocs, ~74-85 us -> ~25-37 us; Get of one int64 field 75 -> 38 allocs. 370 allocs saved per 10 fields =
37 per field (matches the author). *regexp.Regexp is documented safe for concurrent use (only Longest() mutates); pattern identical.

### C12 (eager rounding diagnostic): CONFIRMED
Draft +30/-25, 19 lazy uses (lines 56..170 as cited; 44 is the declaration). No mutating big.Float call on `result` in number.go.
Public bench GetInt64: 75 -> 65 allocs, 5259 -> 4897 B (10 allocs, ~360 B per numeric field; author's 14 -> 4 for Number alone).

### C13 (fromPrimitive): CONFIRMED
My normalisation of the 7 functions: 6 identical (md5 b3b04b9f), FromBigInt differs only by `fl := big.NewFloat(0).SetInt(val)`.
Draft applied on my c10: number.go 312 -> 221, primitive.go 108 -> 96, +17/-120; the author's base also had C12 (+5), hence their
317 -> 226. Tests and lint pass; differential probe output identical to the original.

### C14 (TransformDefaults): CONFIRMED
My normalisation: Bool, Float32, Float64, Int32, Int64, Number, Object, String, Dynamic identical (md5 32890bd7); List, Map, Set
identical (915fc4f4, Map differs by a blank line). Helper checks `respDiags.HasError()` (the response's, not the accumulated diags,
as the original). PlanValue fields are concrete basetypes structs; ElementType is `return l.elementType`. Draft -174/+49, file
412 -> 287, tests (fwschemadata, fwserver, proto6server) and lint pass. Coverage of TransformDefaults 96.7%, uncovered blocks only
at 48, 60, 69, 78, 91 (preamble).

### C15 / C16 (semantic equality helpers): CONFIRMED with a logging caveat
Scalar files identical modulo type name; list/map/set outer functions identical; object differs in comments and the Attributes
function. C15 draft -280/+74; C16 on top of C15 +81/-256; combined +155/-536, non-test value_semantic_equality*.go 1400 -> 1019.
Tests and lint pass. Nested helper keeps the `resp.Diagnostics.HasError()` return before usePriorValue; scalar helper has none.
@caller: terraform-plugin-log defaults IncludeLocation=true (internal/logging/options.go:211) and the framework subsystem adds
WithAdditionalLocationOffset(1), so the JSON log field @caller changes. Not disqualifying: line numbers change with any edit,
the framework's own log tests (e.g. proto5server server_getproviderschema_test.go:720) compare entries without @caller, and
no test asserts these lines. But for C15, the 7 scalar functions emit the same message text and tf_value_type carries the VALUE's
String() ("true", "\"x\"", "<null>"), so the per-type file in @caller is currently the only clue which typed function ran.

### C17 (reflect idioms): CONFIRMED, overlap
Draft +16/-15, tests and lint pass; TypeFor identity and 0 allocs either way (convprobe: 7.5 vs 6.7 ns, 0 B). The 5 go-fix
TypeFor sites (into.go:70,76,80,95, generic_attr_value.go:14) are already in the coordinator's audit/m-rtf.diff; the 3
big.NewX sites are not.

### C18 (slices.Contains): CONFIRMED
Draft +30/-61; gofmt flags only the import order in static_collection_validation.go (as the author said); after gofmt, tests for
fwschema/fwtype/fwschemadata and lint pass.

### C19 (maps.EqualFunc): CONFIRMED
maps.EqualFunc = len check + range m1 + `m2[k]` lookup + eq, short-circuit: the same algorithm as the hand-written loops, so nil
and empty maps compare equal in both (len 0). GetAttributes()/GetBlocks() return internal types (fwschema.UnderlyingAttributes,
map[string]fwschema.Block), so only framework types (47 implementations, all `return schemaAttributes(...)`-style builders) or
promoted embeds implement them. Draft -64/+7; tests (fwschema + 5 schema packages) and lint pass.
Probe convprobe/mapseq_test.go (hand-written loop vs maps.EqualFunc with an interface method expression `eqer.Equal`):
nil-nil, nil-empty, empty-nil true/true; same-length different keys, same keys different values, different lengths
false/false; equal maps in different insertion order true/true. No case differs.

### C20 (degenerate switches): CONFIRMED, draft does not compile
Applying draft-fwschema-switch.diff (identical to drafts/C/fwschema-draft/*.go):
```
internal/fwschema/attribute_validate_implementation.go:10:2: ".../path" imported and not used
internal/fwschema/block_validate_implementation.go:10:2: ".../path" imported and not used
```
and a double blank line (gofmt -l lists both files). Removing the import and gofmt: -56/+3; go test for fwschema and the 7
schema packages pass; lint 0 issues.

## New item
C21 tfsdk.ValueFrom drops FromValue warnings (tfsdk/value_from.go:20-25: `return ValueAs(ctx, v, target)`). Probe TestC6b:
`tfsdk.ValueFrom(ctx, "a", vStrType{}, &s)` -> validator called once (warning), 0 diagnostics returned; same with an *attr.Value
target. With a validating target (*vStr) ValueAs re-validates and the warning reappears. ValueAs uses path.Empty() like FromValue,
so `diags.Append(ValueAs(...)...)` cannot duplicate. TestValueFrom has no warning cases.

## Rejected rewrites checked
Author's 10: all endorsed (probes for 1; code reads for 2-10). #2 is theoretical today: diags is always empty at BuildValue's kind
switch (NewUnknownable/NewNullable only add errors and return) and no callee returns an empty non-nil, nil-containing or duplicate
Diagnostics; line 168 already uses `return Number(...)`. Mine: C1 key-kind check, C2 ungated nil check, C3 removal before C6,
C4 fwschema regexp reuse (cycle), C8 deletion (lifecycle).
