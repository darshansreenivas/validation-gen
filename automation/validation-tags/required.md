# +k8s:required

## Description
Indicates that a field must be specified by the client. If not provided, the API server will reject the object.

## Scope
`Field`

## Supported Go Types
Any Go type. The tag indicates that the field must be present in the submitted object.

## Stability
**Stable**

## Usage

### Field
```go
type PodSpec struct {
    // This field must be specified by the client.
    // +k8s:required
    Containers []Container `json:"containers"`
}
```
If a field is a pointer type (e.g., `*string`), `+k8s:required` implies that the pointer itself must be non-nil. For `string` and slice types, it implies non-empty (length > 0) unless additional validation rules allow empty values.

## Migrating from Handwritten Validation

The `+k8s:required` tag declaratively enforces that a field must be supplied by the client. This directly replaces handwritten validation logic that would check for the absence of a field and report a `field.Required` error.

## Detailed Example: Required Pod Containers

This example demonstrates how `+k8s:required` ensures that the `Containers` field within a `PodSpec` is always provided by the client.

### 1. Define the Tag in `types.go`
Apply the `+k8s:required` tag to the `Containers` field in the `PodSpec` struct definition.

**File:** `staging/src/k8s.io/api/core/v1/types.go`
```go
type PodSpec {
    // ...
    // List of containers belonging to the pod.
    // There must be at least one container in a Pod.
    // +k8s:required
    Containers []Container `json:"containers" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,2,rep,name=containers"`
    // ...
}
```

### 2. Update Handwritten Validation
In the corresponding handwritten validation function (`ValidatePodSpec`), remove or mark as covered the explicit checks for the `Containers` field's presence.

**File:** `pkg/apis/core/validation/validation.go`
```go
func ValidatePodSpec(spec *core.PodSpec, fldPath *field.Path, opts PodValidationOptions) field.ErrorList {
    allErrs := field.ErrorList{}

    // ... other validations ...

    // Original handwritten validation (example):
    // if len(spec.Containers) == 0 {
    //     allErrs = append(allErrs, field.Required(fldPath.Child("containers"), "must have at least one container"))
    // }

    // After adding +k8s:required, the generated code handles this.
    // If still present for backward compatibility during migration, mark the error as covered:
    if len(spec.Containers) == 0 {
        allErrs = append(allErrs, field.Required(fldPath.Child("containers"), "must have at least one container").MarkCoveredByDeclarative())
    }

    // ... other validations ...
    return allErrs
}
```
The `+k8s:required` tag makes the mandatory nature of the `Containers` field explicit in the API definition, simplifying validation logic and ensuring that clients always provide this essential information.

## Test Coverage

When using `+k8s:required`, you should add declarative validation tests to verify that an error is returned when the required field is not set.

### Example

Suppose you have a `name` field that must always be provided.

**File:** `pkg/apis/example/v1/types.go`
```go
type MyStruct struct {
    // +k8s:required
    Name string `json:"name"`
}
```

Your `declarative_validation_test.go` should include a test case that fails when `name` is empty.

**File:** `pkg/apis/example/validation/declarative_validation_test.go`
```go
func TestDeclarativeValidateRequired(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyStruct
        expectedErrs field.ErrorList
    }{
        "required field is present": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Name = "a-valid-name"
            }),
            expectedErrs: field.ErrorList{},
        },
        "required field is empty": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Name = ""
            }),
            expectedErrs: field.ErrorList{
                field.Required(field.NewPath("spec", "name"), "").WithOrigin("required"),
            },
        },
    }
    // ...
}
```

In this example:
1.  We test the valid case where the required field is set.
2.  We test the invalid case where the required field is empty and expect a `field.Required` error.
3.  The error origin is `required`, corresponding to the `+k8s:required` tag.