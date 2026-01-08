# +k8s:optional

## Description
Indicates that a field is optional. If the field has a default value (via `+default`), it is optional for the client but required for the server (the server will populate the default).

## Scope
`Field`

## Supported Go Types
Any Go type. The tag indicates that the field does not need to be specified by the client.

## Stability
**Stable**

## Usage

### Field
```go
type ServiceSpec struct {
    // This field is optional. If not provided, the system may assign a default or leave it unset.
    // +k8s:optional
    ClusterIP string `json:"clusterIP,omitempty"`
}
```
## Migrating from Handwritten Validation

The `+k8s:optional` tag declaratively marks a field as not strictly required. This directly replaces handwritten validation logic that would otherwise report `field.Required` errors if the field were omitted. When a field is `+k8s:optional`, the generated validation code will not flag its absence as an error.

## Detailed Example: Optional Service Cluster IP

This example demonstrates how `+k8s:optional` is applied to the `ClusterIP` field of a `ServiceSpec`, allowing the system to automatically assign an IP if one is not provided.

### 1. Define the Tag in `types.go`
Apply the `+k8s:optional` tag to the `ClusterIP` field in the `ServiceSpec` struct definition.

**File:** `staging/src/k8s.io/api/core/v1/types.go`
```go
type ServiceSpec {
    // ...
    // clusterIP is the IP address of the service. It is optional; if not specified,
    // a cluster IP will be allocated by the system.
    // +k8s:optional
    ClusterIP string `json:"clusterIP,omitempty" protobuf:"bytes,3,opt,name=clusterIP"`
    // ...
}
```

### 2. Update Handwritten Validation
In the corresponding handwritten validation function (`ValidateServiceSpec`), remove or mark as covered any explicit `field.Required` checks for the `ClusterIP` field.

**File:** `pkg/apis/core/validation/validation.go`
```go
func ValidateServiceSpec(spec *core.ServiceSpec, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}

    // ... other validations ...

    // Original handwritten validation (example, if ClusterIP were required):
    // if len(spec.ClusterIP) == 0 {
    //     allErrs = append(allErrs, field.Required(fldPath.Child("clusterIP"), "must be specified for this service type"))
    // }

    // With +k8s:optional, the generated code will not produce a field.Required error for this field.
    // If a handwritten check exists for other reasons (e.g., specific conditions for setting/unsetting):
    if len(spec.ClusterIP) == 0 && someConditionMakesItRequired() { // Example: hypothetical condition
        allErrs = append(allErrs, field.Required(fldPath.Child("clusterIP"), "must be specified under certain conditions").MarkCoveredByDeclarative())
    }

    // ... other validations ...
    return allErrs
}

// someConditionMakesItRequired is a hypothetical function for demonstration.
func someConditionMakesItRequired() bool {
    // Implement logic here if ClusterIP is conditionally required
    return false
}
```
The `+k8s:optional` tag clearly communicates that the field is not mandatory, simplifying API usage and reducing the need for explicit checks in validation code.

## Test Coverage

The `+k8s:optional` tag indicates that a field is not required, which is the default behavior for most fields. Its primary testing use case is to verify that it correctly overrides a `+k8s:required` tag on a type definition, making a specific field optional even when the underlying type is generally required.

### Example: Overriding a Required Type

Suppose you have a `RequiredString` type that must not be empty, but you want to make a specific field of this type optional.

**File:** `pkg/apis/example/v1/types.go`
```go
// +k8s:required
type RequiredString string

type MyStruct struct {
    // This field would be required by default due to the RequiredString type,
    // but +k8s:optional overrides it.
    // +k8s:optional
    OptionalField RequiredString `json:"optionalField,omitempty"`

    // This field remains required.
    RequiredField RequiredString `json:"requiredField"`
}
```

Your `declarative_validation_test.go` should verify that `OptionalField` can be empty, while `RequiredField` cannot.

**File:** `pkg/apis/example/validation/declarative_validation_test.go`
```go
func TestDeclarativeValidateOptional(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyStruct
        expectedErrs field.ErrorList
    }{
        "optional field is omitted, required field is present": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.RequiredField = "I am required"
            }),
            expectedErrs: field.ErrorList{},
        },
        "required field is omitted": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                // OptionalField is omitted, which is fine.
                // RequiredField is omitted, which is an error.
            }),
            expectedErrs: field.ErrorList{
                field.Required(field.NewPath("spec", "requiredField"), "").WithOrigin("required"),
            },
        },
    }
    // ...
}
```

In this example:
1.  The first test case shows that omitting `OptionalField` is valid as long as `RequiredField` is provided.
2.  The second test case shows that omitting `RequiredField` results in a `field.Required` error, confirming that the `+k8s:required` tag on the type is still enforced for other fields.