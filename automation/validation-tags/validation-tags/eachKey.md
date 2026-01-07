# +k8s:eachKey

## Description
Applies a validation tag to every **key** in a map.

## Scope
`Field`, `Type`

## Supported Go Types
`map[K]V`, `*map[K]V` (and any alias of these types), where `K` is the type of the key being validated.

## Payload
`+<validation-tag>`

## Stability
**Alpha**

## Usage

### Field
```go
type MyStruct struct {
    // Validates that each key in the map has a max length of 32
    // +k8s:eachKey=+k8s:maxLength=32
    Labels map[string]string `json:"labels"`
}
```

### Type
```go
// +k8s:eachKey=+k8s:maxLength=10
type MyMapKey string

type MyMap map[MyMapKey]string

type MyStruct struct {
    Data MyMap `json:"data"`
}
```

## Migrating from Handwritten Validation

When applying `+k8s:eachKey` to a map field that previously had handwritten validation for its keys, update the handwritten logic to mark the specific key validation errors as covered by declarative validation.

## Detailed Example: Validating Map Keys for Max Length

This example demonstrates applying `+k8s:eachKey` with `+k8s:maxLength` to validate the keys of a map.

### 1. Define the Tag in `types.go`
Apply the `+k8s:eachKey` tag with a nested `+k8s:maxLength` to the map field.

**File:** `pkg/apis/example/v1/types.go`
```go
type MyResourceSpec struct {
    // Validates that each key in the map is a valid label key,
    // and specifically, has a maximum length of 63 characters.
    // +k8s:eachKey=+k8s:maxLength=63
    // +k8s:eachKey=+k8s:format=k8s-label-key
    Labels map[string]string `json:"labels,omitempty"`
}
```

### 2. Update Handwritten Validation
In the corresponding handwritten validation function, mark the `TooLong` error for map keys as covered by declarative validation.

**File:** `pkg/apis/example/validation/validation.go`
```go
func ValidateMyResourceSpec(spec *MyResourceSpec, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}

    for key, _ := range spec.Labels {
        // Original handwritten validation for key length:
        // if len(key) > 63 {
        //     allErrs = append(allErrs, field.TooLong(fldPath.Child("labels").Key(key), key, 63))
        // }

        // After adding +k8s:eachKey=+k8s:maxLength=63, mark the error as covered:
        if len(key) > 63 {
            allErrs = append(allErrs, field.TooLong(fldPath.Child("labels").Key(key), key, 63).MarkCoveredByDeclarative())
        }
        // ... other handwritten validation for label keys (e.g., format checks) ...
    }

    return allErrs
}
```
By marking the specific length validation error as covered, you ensure that duplicate errors are avoided when both declarative and handwritten validations are active.

## Test Coverage

When using `+k8s:eachKey`, you should add declarative validation tests to verify that the validation is correctly applied to each key of the map.

### Example

Suppose you have a map where each key must have a maximum length of 10 characters, like this:

**File:** `pkg/apis/example/v1/types.go`
```go
type MyStruct struct {
    // Validates that each key in the map has a max length of 10
    // +k8s:eachKey=+k8s:maxLength=10
    Labels map[string]string `json:"labels"`
}
```

Your `declarative_validation_test.go` should include test cases to cover valid and invalid keys.

**File:** `pkg/apis/example/validation/declarative_validation_test.go`
```go
func TestDeclarativeValidate(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyStruct
        expectedErrs field.ErrorList
    }{
        "valid map key length": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Labels = map[string]string{"short-key": "value"}
            }),
            expectedErrs: field.ErrorList{},
        },
        "map key length equal to max": {
            input: mkMyStruct(func(obj *example.myStruct) {
                obj.Labels = map[string]string{"exactly-10": "value"}
            }),
            expectedErrs: field.ErrorList{},
        },
        "map key length too long": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Labels = map[string]string{"this-key-is-too-long": "value"}
            }),
            expectedErrs: field.ErrorList{
                field.TooLong(field.NewPath("spec", "labels").Key("this-key-is-too-long"), "", 10).WithOrigin("maxLength"),
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
1.  We test three scenarios: a key shorter than the max length, a key equal to the max length, and a key longer than the max length.
2.  For the invalid case, we expect a `field.TooLong` error.
3.  The `field.Path` for a map key is constructed using `.Key()`.
4.  The error origin is `maxLength`, which corresponds to the `+k8s:maxLength` tag that was applied by `+k8s:eachKey`.
