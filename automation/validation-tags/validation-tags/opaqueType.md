# +k8s:opaqueType

## Description
Instructs the generator to ignore any validations defined on the referenced type's definition. This is useful when the referenced type itself has validation tags that should not apply in the current context, or when custom validation is preferred.

## Scope
`Field`

## Supported Go Types
Any Go type, including primitive types, structs, slices, and maps. The tag acts on the reference to the type in a field.

## Stability
**Alpha**

## Usage

### Field
```go
// Assume MyType has validation tags defined on its fields.
type MyType struct {
    // +k8s:required
    Value string `json:"value"`
}

type MyStruct struct {
    // The validations defined on MyType will be ignored for this field.
    // +k8s:opaqueType
    OpaqueField MyType `json:"opaqueField"`
}
```

### Chaining
`+k8s:opaqueType` can be chained with `+k8s:eachVal` or `+k8s:eachKey` to suppress validation on elements of lists or keys/values of maps, respectively.

```go
type MyStruct struct {
    // Validations defined on MyType will be ignored for each item in this slice.
    // +k8s:eachVal=+k8s:opaqueType
    OpaqueSliceField []MyType `json:"opaqueSliceField"`

    // Validations defined on MyKeyType and MyValueType will be ignored for keys and values respectively.
    // +k8s:eachKey=+k8s:opaqueType
    // +k8s:eachVal=+k8s:opaqueType
    OpaqueMapField map[MyKeyType]MyValueType `json:"opaqueMapField"`
}

## Migrating from Handwritten Validation

The `+k8s:opaqueType` tag is crucial when a field's type has its own set of validation tags, but these validations should not apply to the field in the current context. Instead, the validation for this specific field might be handled by:
*   **Custom Handwritten Logic**: The field's validation is entirely managed by a dedicated handwritten function.
*   **Contextual Validation**: The validity of the field depends on other fields in the parent struct, requiring custom logic.
*   **External Validation**: The type is validated by an external system or library, and duplicating validation is unnecessary or undesirable.

By applying `+k8s:opaqueType`, you prevent the code generator from producing potentially redundant or conflicting validation code for the referenced type, allowing your custom or external logic to take precedence.

## Detailed Example: Custom Validation for an Opaque Configuration Field

This example demonstrates how `+k8s:opaqueType` can be used to prevent generated validation on a field whose underlying type may have its own validation rules, but for which custom, context-dependent validation is desired.

### 1. Define the Tag in `types.go`
Apply the `+k8s:opaqueType` tag to the field for which generated type validation should be suppressed.

**File:** `pkg/apis/example/v1/types.go`
```go
// Assume ComplexConfig has its own validation tags, e.g., +k8s:required
type ComplexConfig struct {
    // +k8s:required
    ParameterA string `json:"parameterA"`
    ParameterB int    `json:"parameterB"`
}

type MyResourceSpec struct {
    // The validation tags defined on ComplexConfig (e.g., +k8s:required on ParameterA)
    // will be ignored for this specific field. Custom validation will apply.
    // +k8s:opaqueType
    CustomConfig ComplexConfig `json:"customConfig"`
}
```

### 2. Implement Handwritten Validation
Implement a custom handwritten validation function that handles the logic for the `CustomConfig` field, taking into account any specific contextual requirements.

**File:** `pkg/apis/example/validation/validation.go`
```go
func ValidateMyResourceSpec(spec *MyResourceSpec, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}

    // Perform custom validation for CustomConfig, ignoring any tags on ComplexConfig itself.
    if spec.CustomConfig.ParameterA == "" && spec.CustomConfig.ParameterB < 0 {
        allErrs = append(allErrs, field.Invalid(fldPath.Child("customConfig"), spec.CustomConfig, "parameterA cannot be empty and parameterB must be non-negative simultaneously"))
    }
    // ... further custom validation logic ...

    return allErrs
}
```
In this scenario, `+k8s:opaqueType` ensures that the `+k8s:required` tag on `ComplexConfig.ParameterA` is not enforced by generated code for `MyResourceSpec.CustomConfig`, allowing the handwritten `ValidateMyResourceSpec` function to define more nuanced or context-specific rules.

## Test Coverage

The `+k8s:opaqueType` tag is used to suppress declarative validation for a type. Testing for this tag involves demonstrating that fields of this type are not validated, even if they contain data that would otherwise be considered invalid.

### Example

Suppose you have a `RawExtension` type that should be treated as opaque, and it is used in a struct that also contains a validated field.

**File:** `pkg/apis/example/v1/types.go`
```go
// +k8s:opaqueType
type RawExtension struct {
	// This struct can contain anything, and it won't be validated.
	InnerField string `json:"innerField,omitempty"`
}

type MyStruct struct {
    OpaqueData RawExtension `json:"opaqueData,omitempty"`
    // +k8s:maxLength=5
    ValidatedField string `json:"validatedField,omitempty"`
}
```

Your `declarative_validation_test.go` should show that validation is skipped for `OpaqueData` but still applied to `ValidatedField`.

**File:** `pkg/apis/example/validation/declarative_validation_test.go`
```go
func TestDeclarativeValidateOpaque(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyStruct
        expectedErrs field.ErrorList
    }{
        "opaque field with invalid-looking data is ignored": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.OpaqueData = example.RawExtension{
                    // If RawExtension were not opaque, this might be invalid
                    InnerField: "some data that could be invalid under other rules",
                }
                obj.ValidatedField = "valid"
            }),
            expectedErrs: field.ErrorList{},
        },
        "validation error from non-opaque field is still reported": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.OpaqueData = example.RawExtension{}
                // This field violates its maxLength rule.
                obj.ValidatedField = "this is too long"
            }),
            expectedErrs: field.ErrorList{
                field.TooLong(field.NewPath("spec", "validatedField"), "", 5).WithOrigin("maxLength"),
            },
        },
    }
    // ...
}
```

In this example:
1.  The first test case shows that no errors are produced for the `OpaqueData` field, demonstrating that it is not being validated.
2.  The second test case confirms that validation rules on other fields (`ValidatedField`) are still enforced, proving that `+k8s:opaqueType` only affects the type it is applied to.