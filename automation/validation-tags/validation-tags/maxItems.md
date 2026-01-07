# +k8s:maxItems

## Description
Limits the maximum number of items in a list or array.

## Scope
`Field`, `Type`

## Supported Go Types
`[]any`, `*[]any` (and any alias of these types)

## Payload
`<non-negative integer>`

## Stability
**Stable**

## Usage

### Field
```go
type PodSpec struct {
    // +k8s:maxItems=1000
    HostAliases []HostAlias `json:"hostAliases,omitempty"`
}
```

### Type
```go
// +k8s:maxItems=10
type AliasSlice []string

type MyStruct struct {
    Aliases AliasSlice `json:"aliases,omitempty"`
}
```

## Migrating from Handwritten Validation

When adding `+k8s:maxItems` to a field that already has handwritten validation, follow this pattern:

1.  **Add the Tag**: Add the `+k8s:maxItems=<value>` tag to the struct field.
2.  **Mark Errors**: Update the handwritten validation logic to mark the item count check error as covered by declarative validation using `.MarkCoveredByDeclarative()`.

## Detailed Example: Validating HostAliases

This example demonstrates how to apply `maxItems` validation to the `HostAliases` field of a `PodSpec` struct.

### 1. Define the Tag in `types.go`
Add the `+k8s:maxItems=1000` tag to the `HostAliases` field in the `PodSpec` struct definition.

**File:** `staging/src/k8s.io/api/core/v1/types.go`
```go
type PodSpec struct {
    // +k8s:maxItems=1000
    HostAliases []HostAlias `json:"hostAliases,omitempty" patchStrategy:"merge" patchMergeKey:"ip" protobuf:"bytes,23,rep,name=hostAliases"`
    // ...
}
```

### 2. Update Handwritten Validation
In the existing validation logic for `PodSpec`, mark the `TooMany` error for the `hostAliases` field as covered by declarative validation.

**File:** `pkg/apis/core/validation/validation.go`

```go
func ValidatePodSpec(spec *core.PodSpec, fldPath *field.Path, opts PodValidationOptions) field.ErrorList {
    allErrs := field.ErrorList{}

    // ... other validation ...

    if len(spec.HostAliases) > 1000 {
        allErrs = append(allErrs, field.TooMany(fldPath.Child("hostAliases"), len(spec.HostAliases), 1000).MarkCoveredByDeclarative())
    }

    // ... other validation ...

    return allErrs
}
```
This ensures that the declarative validation tooling can verify the `maxItems` constraint while maintaining compatibility with the existing handwritten validation.

## Test Coverage

When using `+k8s:maxItems`, you should add declarative validation tests to verify that lists with more than the specified number of items are rejected.

### Example

Suppose you have a slice that can contain at most 2 items.

**File:** `pkg/apis/example/v1/types.go`
```go
type MyStruct struct {
    // +k8s:maxItems=2
    Values []string `json:"values,omitempty"`
}
```

Your `declarative_validation_test.go` should include test cases to cover lists that are within the limit, at the limit, and over the limit.

**File:** `pkg/apis/example/validation/declarative_validation_test.go`
```go
func TestDeclarativeValidateMaxItems(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyStruct
        expectedErrs field.ErrorList
    }{
        "list with fewer than max items": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Values = []string{"one"}
            }),
            expectedErrs: field.ErrorList{},
        },
        "list with exactly max items": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Values = []string{"one", "two"}
            }),
            expectedErrs: field.ErrorList{},
        },
        "list with more than max items": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Values = []string{"one", "two", "three"}
            }),
            expectedErrs: field.ErrorList{
                field.TooMany(field.NewPath("spec", "values"), 3, 2).WithOrigin("maxItems"),
            },
        },
    }
    // ...
}
```

In this example:
1.  We test three scenarios for the list length relative to `maxItems`.
2.  For the case that exceeds the limit, we expect a `field.TooMany` error, which clearly states the actual and maximum allowed number of items.
3.  The error origin is `maxItems`, corresponding to the `+k8s:maxItems` tag.