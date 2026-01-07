# +k8s:eachVal

## Description
Applies a validation tag to every **value** in a list or map.

## Scope
`Field`, `Type`

## Supported Go Types
`[]T`, `*[]T`, `map[K]V`, `*map[K]V` (and any alias of these types), where `T` is the element type of the slice and `V` is the value type of the map being validated.

## Payload
`+<validation-tag>`

## Stability
**Alpha**

## Usage

### Field (Slice)
```go
type MyStruct struct {
    // Validates that each int in the slice is at least 1
    // +k8s:eachVal=+k8s:minimum=1
    Ports []int `json:"ports"`
}
```

### Field (Map)
```go
type MyStruct struct {
    // Validates that each value in the map is a valid LabelValue
    // +k8s:eachVal=+k8s:maxLength=63
    Labels map[string]string `json:"labels"`
}
```

### Type (Slice)
```go
// +k8s:eachVal=+k8s:minimum=0
type MySlice []int

type MyStruct struct {
    Values MySlice `json:"values"`
}
```

## Migrating from Handwritten Validation

When applying `+k8s:eachVal` to a slice or map field that previously had handwritten validation for its elements (values), update the handwritten logic to mark the specific element validation errors as covered by declarative validation.

## Detailed Example: Validating Service Port Numbers

This example demonstrates applying `+k8s:eachVal` with `+k8s:minimum` and `+k8s:maximum` to validate the `Port` field of `ServicePort` objects within a `ServiceSpec.Ports` slice.

### 1. Define the Tag in `types.go`
Apply the `+k8s:eachVal` tag with nested `+k8s:minimum` and `+k8s:maximum` to the `Ports` field in the `ServiceSpec` struct. Note that `ServicePort` is a struct, and the validation needs to apply to its `Port` field. This is achieved by having the tags on `ServicePort` itself, and the generator will apply them to its `Port` field.

**File:** `staging/src/k8s.io/api/core/v1/types.go`
```go
type ServiceSpec struct {
    // ...
    // +k8s:eachVal=+k8s:minimum=1
    // +k8s:eachVal=+k8s:maximum=65535
    Ports []ServicePort `json:"ports,omitempty" patchStrategy:"merge" patchMergeKey:"port" protobuf:"bytes,1,rep,name=ports"`
    // ...
}

type ServicePort struct {
    // ...
    Port int32 `json:"port" protobuf:"varint,3,opt,name=port"`
    // ...
}
```

### 2. Update Handwritten Validation
In the corresponding handwritten validation function (`validateServicePort`), mark the range validation errors for the `port` field as covered by declarative validation.

**File:** `pkg/apis/core/validation/validation.go`
```go
func validateServicePort(sp *core.ServicePort, requireName, isHeadlessService bool, allNames *sets.Set[string], fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}

    // Original handwritten validation for port range:
    // if sp.Port > 65535 || sp.Port < 0 { // Note: 0 is allowed for TargetPort, but not for Port
    //     allErrs = append(allErrs, field.Invalid(fldPath.Child("port"), sp.Port, validation.InclusiveRangeError(1, 65535)))
    // }

    // After adding +k8s:eachVal=+k8s:minimum=1 and +k8s:eachVal=+k8s:maximum=65535,
    // mark the error as covered:
    if sp.Port > 65535 || sp.Port < 1 { // Assuming 0 is not a valid Port value
        allErrs = append(allErrs, field.Invalid(fldPath.Child("port"), sp.Port, validation.InclusiveRangeError(1, 65535)).MarkCoveredByDeclarative())
    }
    // ... other validations ...
    return allErrs
}
```
By marking these specific errors as covered, the system can rely on the declarative tags for port range validation while still executing other handwritten checks.

## Test Coverage

When using `+k8s:eachVal`, you should add declarative validation tests to verify that the validation is correctly applied to each value of the slice or map.

### Example: Slice Validation

Suppose you have a slice of strings where each string must have a maximum length of 5 characters.

**File:** `pkg/apis/example/v1/types.go`
```go
type MyStruct struct {
    // Validates that each string in the slice has a max length of 5
    // +k8s:eachVal=+k8s:maxLength=5
    Values []string `json:"values,omitempty"`
}
```

Your `declarative_validation_test.go` should include test cases for values with lengths less than, equal to, and greater than the maximum.

**File:** `pkg/apis/example/validation/declarative_validation_test.go`
```go
func TestDeclarativeValidate(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyStruct
        expectedErrs field.ErrorList
    }{
        "valid slice values": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Values = []string{"short", "tiny"}
            }),
            expectedErrs: field.ErrorList{},
        },
        "slice value length equal to max": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Values = []string{"exact"}
            }),
            expectedErrs: field.ErrorList{},
        },
        "slice value length too long": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Values = []string{"this-is-too-long"}
            }),
            expectedErrs: field.ErrorList{
                field.TooLong(field.NewPath("spec", "values").Index(0), "", 5).WithOrigin("maxLength"),
            },
        },
        "multiple invalid slice values": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Values = []string{"short", "this-is-too-long", "also-too-long"}
            }),
            expectedErrs: field.ErrorList{
                field.TooLong(field.NewPath("spec", "values").Index(1), "", 5).WithOrigin("maxLength"),
                field.TooLong(field.NewPath("spec", "values").Index(2), "", 5).WithOrigin("maxLength"),
            },
        },
    }
    // ...
    for k, tc := range testCases {
        t.Run(k, func(t *testing.T) {
            apitesting.VerifyValidationEquivalence(t, ctx, &tc.input, Strategy.Validate, tc.expectedErrs)
        })
    }
}
```

In this example:
1.  We test multiple scenarios, including multiple invalid values in the same slice.
2.  For invalid items, we expect a `field.TooLong` error.
3.  The `field.Path` for a slice item is constructed using `.Index()`.
4.  The error origin is `maxLength`, corresponding to the `+k8s:maxLength` tag applied by `+k8s:eachVal`.

