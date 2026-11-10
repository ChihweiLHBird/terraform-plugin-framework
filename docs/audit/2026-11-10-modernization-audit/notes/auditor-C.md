# Area C notes: schema data, reflection, schema internals

Auditor C. Repository /home/user/terraform-plugin-framework at 00badb1 (unchanged; `git status --porcelain` empty before and after).
All paths below starting with `drafts/C/` are relative to
``.

Output: 20 items in findings-C.json. Bugs: 2 tier 1 (crashes), 2 tier 2, 5 tier 3. Refactors: 2 tier 1, 5 tier 2.
Modernizations: 3 tier 3. Cleanup: 1 tier 3.

## 1. Coverage of the area

Read in full (every non-test line):
- internal/reflect: all 14 files (2858 lines).
- internal/fwschemadata: data.go, data_description.go, data_value.go, data_set_at_path.go, data_default.go, data_get.go,
  data_get_at_path.go, data_set.go, data_terraform_value.go, tftypes_value.go, data_path_exists.go, data_path_matches.go,
  data_valid_path_expression.go, data_nullify_collection_blocks.go, value_semantic_equality.go, _bool, _dynamic, _list,
  _set. data_reify_null_collection_blocks.go, _map, _object and the 6 other scalar files were compared by normalised diff
  against the files above (differences read line by line).
- internal/fwschema: attribute_name_validation.go, schema.go, block.go, attribute_validate_implementation.go,
  block_validate_implementation.go, write_only_nested_attribute_validation.go, underlying_attributes.go, errors.go,
  attribute_nesting_mode.go, block_nested_mode.go, validate_implementation.go; skimmed (declarations and loops only)
  attribute.go, attribute_default.go, diagnostics.go, nested_attribute.go, nested_attribute_object.go,
  nested_block_object.go.
- internal/fwschema/fwxschema: skimmed (interface declarations only, nothing to report).
- internal/fwtype, internal/fwfunction, internal/privatestate: all files read.
- Tests: read where a finding depends on what they pin (slice_test.go set/tuple cases, primitive_test.go alias tests,
  helpers_test.go TestCommaSeparatedString/TestIsValidFieldName, privatestate TestProviderDataEqual and typo
  expectations, provider/schema TestSchemaValidateImplementation, fwschema TestIsValidAttributeName).

Swept exhaustively (scripted, every site): the validation blocks in internal/reflect, the TransformDefaults arms, the
semantic-equality family, dead top-level declarations, the 13 exhaustive leads, the 23 dupword leads, the 3
slicescontains and 17 reflecttypefor leads. Sampled: test-file dupl leads (290; see section 5).

## 2. Counts and the commands that reproduce them

Repository commands run from the repository root; commands on leads/ and staticcheck-all.txt run from the audit
directory.

| Count | Value | Command |
|---|---|---|
| SA1019 leads in area | 27 (24 reflect, 3 fwschemadata) | `grep SA1019 staticcheck-all.txt \| grep -cE 'internal/(reflect\|fwschemadata)/'` |
| validation blocks in reflect | 24 | `grep -c 'case xattr.ValidateableAttribute' internal/reflect/*.go \| grep -v _test` (7+3+5+1+2+4+2) |
| `//nolint:staticcheck` in reflect | 24 (-> 2 after C10) | `grep -c 'nolint:staticcheck' internal/reflect/*.go` |
| identical blocks after normalising 5 names | 23 of 24 | `python3 drafts/C/scripts/extract_blocks.py drafts/C/validate-blocks internal/reflect/{interfaces,map,number,pointer,primitive,slice,struct}.go; md5sum drafts/C/validate-blocks/*.norm \| sort` |
| TransformDefaults arms | 12 | `grep -c 'case fwschema.AttributeWith' internal/fwschemadata/data_default.go` |
| scalar semantic-equality files | 7 x 54 lines, identical modulo type name | `wc -l internal/fwschemadata/value_semantic_equality_{bool,float32,float64,int32,int64,number,string}.go`; normalised copies in drafts/C/semeq |
| non-test dupl leads in area | 9 (reflect number.go x3 + primitive.go x2, semeq map/object x2, fwxschema pair x2) | `grep '^golangci/dupl' leads/C-schemadata-reflect.txt \| grep -vc _test.go` |
| test dupl leads in area | 290 | same with `grep -c _test.go` |
| dupword leads | 23, all real | `grep -c '^golangci/dupword' leads/C-schemadata-reflect.txt` |
| exhaustive leads | 13 | `grep -c '^golangci/exhaustive' leads/C-schemadata-reflect.txt` |
| reflecttypefor leads | 17 (5 non-test: into.go x4, generic_attr_value.go x1) | `grep -c '^gofix/reflecttypefor' leads/C-schemadata-reflect.txt` |
| extra `reflect.TypeOf(big.NewX(0))` sites (not flagged) | 4 | `grep -n 'reflect.TypeOf(big.New' internal/reflect/*.go` |
| slicescontains leads | 3 (+3 unflagged same-shape loops) | `grep -c '^gofix/slicescontains' leads/...`; `grep -n 'for _, elemType := range attrType' internal/fwtype/*.go` |
| top-level non-method declarations checked for dead code | 216 | `drafts/C/declister/declister <dirs>` then `git grep -w` per name (drafts/C/decl-refs.tsv) |

## 3. Results of the exhaustive sweeps

### 3.1 Validation blocks in internal/reflect (C10)
Per-site variables (from drafts/C/validate-blocks/*.raw):

| site | value | path | type | Validate arg | early return |
|---|---|---|---|---|---|
| interfaces.go:83,124 FromUnknownable | res | path | typ | tfVal | nil |
| interfaces.go:217,258 FromNullable | res | path | typ | tfVal | nil |
| interfaces.go:349 FromValueCreator | res | path | typ | tfVal | nil |
| interfaces.go:392 NewAttributeValue | res | path | typ | val | target |
| interfaces.go:458 FromAttributeValue (variant, stays inline) | val | path | typ | tfVal computed lazily | val |
| map.go:118,226 FromMap (nil / tail) | attrVal | path | typ | tfVal | nil |
| map.go:178 FromMap element loop | val | mapKeyPath | elemType | tfVal | nil |
| number.go:205,251,297,343,390 FromInt..FromBigInt | num | path | typ | tfNum | nil |
| pointer.go:108 FromPointer | attrVal | path | typ | tfVal | nil |
| primitive.go:77,123 FromString/FromBool | str / b | path | typ | tfStr / tfBool | nil |
| slice.go:200,409 FromSlice (nil / tail) | attrVal | path | typ | tfVal | nil |
| slice.go:262 FromSlice list/set loop | val | valPath | elemType | tfVal | nil |
| slice.go:352 FromSlice tuple loop | val | valPath | elemAttrType | tfVal | nil |
| struct.go:276 FromStruct field loop | attrVal | path (field) | attrTypes[name] | tfObjVal | nil |
| struct.go:325 FromStruct tail | ret | path | typ | tfVal | nil |

Equivalence invariant for the helper: at all 23 replaced sites `diags` contains no error on entry (every earlier error
returns immediately; in the 4 loops the preceding FromValue result is HasError-checked and the ToTerraformValue error
returns). The helper version runs `diags.HasError()` also when neither interface applies; that is false under the
invariant. A future site entered with an error in `diags` would behave differently: the verifier should re-check the
invariant per site.

### 3.2 TransformDefaults (C14) phase map
- 27-31 build configData (Description Configuration).
- 33-61 Transform callback: skip root; schema attribute lookup; ignore ErrPathInsideAtomicAttribute, ErrPathIsBlock,
  ErrPathInsideDynamicAttribute; other errors abort the Transform.
- 63-71 tftypes path -> fw path (diagnostics appended, skip on error).
- 73-100 config value at path; skip unless null (dynamic: underlying value null).
- 102-396 12-arm switch. Normalised: Bool, Float32, Float64, Int32, Int64, Number, Object, String, Dynamic identical (23
  lines, Dynamic only differs by the switch's closing brace in the extraction); List == Set (29 lines); Map == List minus
  one blank line. drafts/C/default-arms/*.norm.
- 401-411 Transform error -> "Error Handling Schema Defaults".

### 3.3 internal/reflect function phases (task 3)
- FromSlice (257 lines): nil slice -> null + validate (186-228); list/set loop: FromValue(elem, index path) [validates],
  ToTerraformValue, set path fix-up, re-validate, append (230-290); tuple: empty element types, homogeneous-type check,
  same loop (297-380); default error; ValidateValue + NewValue + ValueFromTerraform + validate (391-436).
- FromMap (155): nil -> null + validate; key-kind check, FromValue(elem, key path), ToTerraformValue, re-validate;
  ValidateValue + NewValue + ValueFromTerraform + validate.
- FromStruct (176): struct tags; missing-field check (map iteration, C5); field loop with FromValue, ToTerraformValue,
  re-validate, dynamic type fix-up; NewValue + ValueFromTerraform + validate.
- BuildValue (158): invalid target; attr.Value / ValueConverter dispatch; Unknownable and Nullable pre-processing;
  unhandled unknown / null; dynamic guard; *big.Float/*big.Int; kind switch.
- Number (157): val.As(*big.Float); eager rounding diagnostic (C12); *big.Float/*big.Int; int kinds with range checks;
  uint kinds; float32 exactness/underflow/overflow; float64 exactness; fallback error. Inner kind switches are exhaustive
  for the outer case lists (exhaustive leads number.go:61,68,110 are false positives).
- Duplicated sub-blocks between From* functions: the validation block (C10), the element tail "FromValue + ToTerraformValue
  + re-validate" (4 copies, source of C3), the null branch "NewValue(nil) + ValueFromTerraform + validate" (FromSlice,
  FromMap, FromPointer: identical except the "from slice/map/pointer value" error text, so not proposed), and the tuple
  preamble shared by reflectSlice and FromSlice (same logic, different diagnostic constructors and text; not proposed).
- Order pins: in every element loop FromValue diagnostics are appended before ToTerraformValue errors and before the
  re-validation; set paths are computed after ToTerraformValue. C10 keeps all of this.

### 3.4 Semantic equality family (C15, C16)
Scalar files identical modulo the type name (md5 5837055f after normalisation). Outer functions of list/map/set
identical modulo type name (71 lines); object differs in comments and the nested function name. Element functions:
list (index), map (key), set (O(n*m) matching with candidate removal), object (attribute name) are structurally
different and stay.

### 3.5 Dead code
No top-level declaration in the area lacks a non-test reference. Only-comment or test-only references:
- NestingModeUnknown, BlockNestingModeUnknown: zero-value enum sentinels, intentionally kept.
- privatestate.MustMarshalToJson / MustProviderData: production-file test helpers used by tests in 8 packages
  (231 and 160 test references). They cannot move into a _test.go file; moving them to internal/testing is churn.
- Many exported functions in internal/reflect and internal/fwschemadata are only used in their package (NewUnknownable,
  FromInt, ValueSemanticEqualityBool, ...); they are exported for the external `_test` packages. Not worth unexporting.
- getStructTags carries `//nolint:unparam // False positive, ctx is used below.`: ctx is only passed to its own
  recursive call; harmless, not filed.
- Struct fields: none never read.

### 3.6 exhaustive (13 leads)
- data_description.go:48 Title(): real bug, C7.
- attribute_validate_implementation.go:77, block_validate_implementation.go:71,104: degenerate switches, C20.
- block.go:103 BlockPathExpressions: BlockNestingModeUnknown reaches `panic(fmt.Sprintf("unhandled BlockNestingMode: %T",
  nestingMode))`, which prints the type name, not the value. Unreachable: fwschema.Block cannot be implemented outside the
  framework and every framework block returns a valid mode. Latent cosmetic, not filed.
- data_description.go:38 String(), reflect helpers.go:142 canBeNil, into.go:170, number.go:61,68,110, outof.go:45,
  primitive.go:26: defaults are correct (error diagnostic or false for every unlisted kind).

### 3.7 dupword (23 leads)
All real: 1 comment (fwschema/schema.go:70-71), 5 diagnostic strings in privatestate/data.go, 17 test expectations that
pin those strings. Filed as C9.

## 4. Rejected rewrites ("looks equivalent but is not")

1. privatestate ProviderData.Equal: `reflect.DeepEqual(d.data, o.data)` -> `maps.EqualFunc(d.data, o.data, bytes.Equal)`.
   Not equivalent: DeepEqual distinguishes a nil map from an empty map (and nil from empty []byte values), EqualFunc and
   bytes.Equal do not. `(&ProviderData{}).Equal(EmptyProviderData(ctx))` would flip from false to true, and cmp.Diff in
   many fwserver tests calls this Equal method. `maps.Equal` does not compile ([]byte is not comparable). Equal has no
   production caller; keep DeepEqual.
2. BuildValue kind-switch arms `val, valDiags := X(...); diags.Append(valDiags...); return val, diags` -> `return X(...)`:
   not equivalent. Append drops nil and duplicate diagnostics and leaves `diags` nil when valDiags is an empty non-nil
   slice.
3. Using the C10 helper for FromAttributeValue with an eagerly computed tfVal: adds a ToTerraformValue call (and a new
   error path) for every attr.Value. Kept inline.
4. Removing the element-loop re-validation (slice.go:262, 352; map.go:178; struct.go:276) as a refactor: changes
   validator call counts (observable side effects) and, for sets, removes the set-path diagnostics. Filed as bug C3.
5. One helper for scalar and nested semantic equality (scalar = nested with a no-op nested function): the nested variant
   returns on `resp.Diagnostics.HasError()` before applying usePriorValue, the scalar variant applies the prior value even
   when the provider returned an error diagnostic. Two helpers.
6. Generic TransformDefaults helper over Request/Response types: not expressible (no field access through type
   parameters); the typed request/response construction stays in each arm.
7. Method expression `basetypes.BoolValuableWithSemanticEquals.BoolSemanticEquals` as the C15 callback: does not
   type-check (its parameter is BoolValuable, not the WithSemanticEquals interface), hence the closure.
8. Sorting inside commaSeparatedString to fix C5: breaks TestCommaSeparatedString, which expects input order ("red and
   blue"). Sort at the call sites.
9. A shared helper for the 3 fwschemadata validation blocks (data_value.go:92, data_set_at_path.go:118, :269): the third
   site logs "Calling provider defined Type ValidateAttribute" where the others log "Type Validate"; a helper would change
   that log text (and the log @caller of all three). Low value (~50 lines); not filed. The wrong log text itself is
   cosmetic and not filed.
10. Folding FromValueCreator into C13's fromPrimitive: it calls typ.TerraformType(ctx) twice; the helper would call it
    once. Left out.

Log caveat for C14/C15/C16: the framework subsystem logger includes the call location (internal/logging/context.go:
WithAdditionalLocationOffset(1)), so moving a logging.Framework* call into a helper changes the `@caller` field of that
log line. Message, level and fields are identical; no test captures these lines.

## 5. Discarded leads

- golangci/dupl in tests (290): table-driven test cases (data_default_test.go 102, data_set_at_path_test.go 43, ...). Not
  a refactor target.
- funlen (35, all tests), gocognit on tests (2), godox (7 TODOs with issue links), nilnil (test), U1000 x4 (deliberate
  unexported fields in test fixtures, two already `//nolint:unused`).
- errorlint struct.go:146,248 (`%s` -> `%w`): message text identical; the error is only used through `.Error()`
  (DiagIntoIncompatibleType.Detail/Equal). No observable change, no value.
- prealloc schema.go:186 (result length unknown), tftypes_value.go:40 (micro), privatestate/data.go:410 (test helper).
- perfsprint privatestate/data.go:416 (test helper). into.go:61 folded into C17.
- recvcheck fwschemadata.Data (mixed receivers): pointer receivers for the mutating methods; intentional.
- gofix any/newexpr/fmtappendf: coordinator-owned.
- reflecttypeassert in tests (12): test-only, low value.
- NewValueConverter error text names "FromTerraform5Type" while it looks up "FromTerraform5Value"
  (interfaces.go:298), and NewUnknownable/NewNullable "cannot find method" branches: unreachable (BuildValue only calls
  them when the type implements the interface). Not filed.
- ImportBeforeReadKey is a never-reassigned `var` (could be `const`): trivial, not filed.
- ValidPathExpression builds ElementKeyValue steps from the collection type instead of the element type
  (data_valid_path_expression.go): harmless because ApplyTerraform5AttributePathStep ignores the step's value for sets.
- toTerraformValueErrorDiag in FromSlice (:255, :349) and FromMap (:175) uses the parent path although the element path
  is known (FromStruct uses the field path). Latent: needs ToTerraformValue to fail on a value just built. Not filed.

## 6. Systemic observation, not filed

Map iteration order leaks into the order of separate diagnostics in: reflect Map, FromMap, Struct, FromStruct field
loops; map/object semantic-equality element loops; every schema ValidateImplementation (top-level attributes and nested
attributes); privatestate Bytes/NewData/NewProviderData. With two or more failing elements the first error returned (or
the order of warnings) varies between runs. Only the case where the order changes the text of a single diagnostic is
filed (C5).

## 7. For the verifier

- C1, C2 are crashes reproduced through the public tfsdk API (drafts/C/probe-public, a module with
  `replace github.com/hashicorp/terraform-plugin-framework => /home/user/terraform-plugin-framework`; run
  `GOFLAGS=-mod=mod go test -count=1 -v .` inside drafts/C/probe-public). TestPrimitive_stringAlias/boolAlias show alias
  support is intended for strings and bools.
- C3: run drafts/C/probe-public TestSetValueFromValidation and drafts/C/repros (see below). Check the proposed fix design for
  sets; the repro numbers are the evidence, the fix is a direction.
- C4: confidence medium; check CHANGELOG #705 wording and decide whether the tag rule was deliberate.
- C8: PLAUSIBLE; provider/schema TestSchemaValidateImplementation pins the current message.
- C10: re-check the no-error-on-entry invariant per site (section 3.1) and that apply_helper.py kept each site's early
  return value. Drafts were tested in a full copy of the repository (drafts/C/repo) with go vet, go test (reflect,
  fwschemadata, tfsdk, types/basetypes, function, fwserver, proto6server) and golangci-lint (project config) on
  internal/reflect and internal/fwschemadata: 0 issues.
- C11/C12 benchmark numbers came from a shared 4-core machine: allocation counts are exact, timings are noisy.

## 8. Reproduction files and drafts

- Repros (all reproduce at 00badb1): drafts/C/repros/reflect/zz_*_test.go and drafts/C/repros/fwschemadata/zz_*_test.go.
  Run `sh drafts/C/repros/run.sh <path-to-a-COPY-of-the-repo>` (copies the files into the copy and runs
  `go test -run 'TestRepro|TestProbe'` plus the struct benchmarks). Never point it at the real repository.
- Public-API repros: drafts/C/probe-public/*.go.
- Draft diffs against the repository: drafts/C/draft-validate-helper.diff (C10), draft-fromprimitive.diff (C13, on top of
  C10), draft-regexp-hoist.diff (C11), draft-lazy-rounding.diff (C12), draft-default-helper.diff (C14),
  draft-semeq-helper.diff (C15), draft-semeq-nested.diff and draft-semeq-all.diff (C16), draft-reflect-idioms.diff (C17),
  draft-slices-contains.diff (C18), draft-fwschema-mapsequal.diff (C19), draft-fwschema-switch.diff (C20),
  draft-fix-named-types.diff (C1 fix sketch).
- Complexity before/after (golangci gocognit/gocyclo/dupl, threshold 15, config drafts/C/gocog.yml): TransformDefaults
  112 -> 67, FromSlice 89 -> 46, FromMap 44 -> 17, FromStruct 49 -> 32, FromUnknownable/FromNullable/FromPointer
  25/25/16 -> below 15; non-test dupl hits in reflect + fwschemadata 7 -> 2.
