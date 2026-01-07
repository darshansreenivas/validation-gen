# +k8s:subfield

## Description
Targets a subfield of a struct for validation, allowing validation tags to be applied to nested fields.

**Important Usage Constraint:**
*   **Use `+k8s:subfield` ONLY** when the type of the field is defined **outside** of the current `types.go` file (e.g., targeting fields in `metav1.ObjectMeta` or types from other packages).
*   **Do NOT use `+k8s:subfield`** if the field's type is a subtype defined **within the same `types.go` file**. In such cases, apply the validation annotations directly to the fields within that subtype's definition.

## Scope
`Type`, `Field`

## Supported Go Types
Any Go type. This tag targets a field within a struct, regardless of its own type.

## Arguments
`name=<field-json-name>` (Required): The JSON name of the subfield to target.

## Payload
`+<validation-tag>`: The validation tag(s) to apply to the targeted subfield.

## Stability
**Stable**

## Usage

### Field (External Type)
Use this pattern for types defined in other packages, such as `metav1.ObjectMeta`.

```go
type MyResource struct {
    // Targets the "name" subfield of the embedded metav1.ObjectMeta,
    // which is defined in the apimachinery repository.
    // +k8s:subfield(name="name")=+k8s:optional
    // +k8s:subfield(name="name")=+k8s:format=k8s-long-name
    metav1.ObjectMeta `json:"metadata,omitempty"`
}
```

### Type (Internal Subtype - Preferred Pattern)
If the type is defined locally, put the tags on the subtype fields directly.

```go
// CORRECT: Tags on the subtype definition
type MyCustomField struct {
    Key   string `json:"key"`
    // +k8s:minLength=5
    Value string `json:"value"`
}

type MyStruct struct {
    Field MyCustomField `json:"field"`
}

// INCORRECT: Avoid using subfield for local types
type MyIncorrectStruct struct {
    // +k8s:subfield(name="value")=+k8s:minLength=5
    Field MyCustomField `json:"field"`
}
```
This enables granular validation on complex nested structures without requiring extensive boilerplate code.

## Migrating from Handwritten Validation

The `+k8s:subfield` tag significantly simplifies validation of nested fields by allowing direct application of validation tags to specific sub-elements within a struct. This replaces handwritten validation logic that would otherwise require manually constructing `field.Path` objects and traversing nested structures to apply checks.

## Detailed Example: Validating a DeviceClass Metadata Name

This example demonstrates how `+k8s:subfield` is used to apply format validation (`+k8s:format=k8s-long-name`) and optionality to the `name` field within the embedded `metav1.ObjectMeta` of a `DeviceClass`.

### 1. Define the Tag in `types.go`
Apply the `+k8s:subfield` tags to the embedded `metav1.ObjectMeta` field in the `DeviceClass` struct.

**File:** `staging/src/k8s.io/api/resource/v1/types.go`
```go
type DeviceClass struct {
    metav1.TypeMeta `json:",inline"`
    // Standard object metadata
    // +optional
    // +k8s:subfield(name)=+k8s:optional // The name field of ObjectMeta is optional for DeviceClass
    // +k8s:subfield(name)=+k8s:format=k8s-long-name // The name must conform to k8s-long-name format
    metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

    // Spec defines what can be allocated and how to configure it.
    Spec DeviceClassSpec `json:"spec" protobuf:"bytes,2,name=spec"`
}
```

### 2. Update Handwritten Validation
In the corresponding handwritten validation function (e.g., for `DeviceClass`), remove or mark as covered any explicit logic that constructs paths to `metadata.name` and applies format checks.

**File:** `pkg/apis/resource/validation/validation.go`
```go
func ValidateDeviceClass(obj *resource.DeviceClass, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}

    // ... other validations ...

    // Original handwritten validation for metadata.name (example):
    // if obj.ObjectMeta.Name != "" { // Check if name is provided
    //     for _, msg := range validation.IsDNS1123Subdomain(obj.ObjectMeta.Name) {
    //         allErrs = append(allErrs, field.Invalid(fldPath.Child("metadata", "name"), obj.ObjectMeta.Name, msg))
    //     }
    // }

    // After adding +k8s:subfield(name)=+k8s:optional and +k8s:subfield(name)=+k8s:format=k8s-long-name,
    // the generated code handles this. If still present for backward compatibility:
    namePath := fldPath.Child("metadata", "name")
    if obj.ObjectMeta.Name != "" {
        for _, msg := range validation.IsDNS1123Subdomain(obj.ObjectMeta.Name) { // Assuming IsDNS1123Subdomain aligns with k8s-long-name
            allErrs = append(allErrs, field.Invalid(namePath, obj.ObjectMeta.Name, msg).MarkCoveredByDeclarative())
        }
    }
    // Note: The optionality is handled automatically by the generated code due to +k8s:subfield(name)=+k8s:optional.

    // ... other validations ...
    return allErrs
}
```
The `+k8s:subfield` tag makes the validation of nested fields explicit and easily readable in the API definition, reducing the verbosity of handwritten validation code.

## Test Coverage

When using `+k8s:subfield`, your declarative validation tests should verify that the validation rule is correctly applied to the specified sub-field.

### Example

Suppose you have a nested struct and you want to apply a `maxLength` validation to a sub-field.

**File:** `pkg/apis/example/v1/types.go`
```go
type InnerSpec struct {
    TargetField string `json:"targetField,omitempty"`
}

type OuterSpec struct {
    Inner InnerSpec `json:"inner,omitempty"`
}

// +k8s:subfield(inner.targetField)=+k8s:maxLength=5
type MyResource struct {
    Outer OuterSpec `json:"outer,omitempty"`
}
```

Your `declarative_validation_test.go` should include test cases to confirm that the validation is applied to `inner.targetField`.

**File:** `pkg/apis/example/validation/declarative_validation_test.go` (hypothetical example)
```go
func TestDeclarativeValidateSubfield(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyResource
        expectedErrs field.ErrorList
    }{
        "subfield within length limit": {
            input: example.MyResource{
                Outer: example.OuterSpec{
                    Inner: example.InnerSpec{
                        TargetField: "valid",
                    },
                },
            },
            expectedErrs: field.ErrorList{},
        },
        "subfield exceeds length limit": {
            input: example.MyResource{
                Outer: example.OuterSpec{
                    Inner: example.InnerSpec{
                        TargetField: "this-is-too-long",
                    },
                },
            },
            expectedErrs: field.ErrorList{
                field.TooLong(
                    field.NewPath("outer", "inner", "targetField"),
                    "", 5,
                ).WithOrigin("maxLength"),
            },
        },
    }
    // ...
}
```

In this example:
1.  We test a case where the `targetField`'s length is within the limit, which should pass.
2.  We test a case where the `targetField`'s length exceeds 5 characters and expect a `field.TooLong` error.
3.  The error path correctly reflects the nested structure: `outer.inner.targetField`, demonstrating that `+k8s:subfield` has correctly targeted the validation.