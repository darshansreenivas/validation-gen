# +k8s:maxProperties

## Description
Limits the maximum number of keys (properties) in a map.

## Scope
`Field`, `Type`

## Supported Go Types
`map[K]V`, `*map[K]V` (and any alias of these types).

## Payload
`<non-negative integer>`

## Stability
**Alpha**

## Usage

### Field
```go
type Config struct {
    // Limits the Data map to a maximum of 16 key-value pairs.
    // +k8s:maxProperties=16
    Data map[string]string `json:"data,omitempty"`
}
```

### Type
```go
// Limits instances of this map type to a maximum of 8 key-value pairs.
// +k8s:maxProperties=8
type LimitedMap map[string]string

type MyStruct struct {
    Settings LimitedMap `json:"settings"`
}
```
This tag is analogous to `+k8s:maxItems` but applies specifically to maps.

## Migrating from Handwritten Validation

The `+k8s:maxProperties` tag declaratively enforces a maximum number of entries (key-value pairs) in a map. This directly replaces handwritten validation logic that would iterate through a map and check its `len()` against a maximum allowed size.

## Detailed Example: Limiting ConfigMap Data Entries

This example demonstrates applying `+k8s:maxProperties` to the `Data` field of a `ConfigMap` object, ensuring that the map does not exceed a specified number of entries.

### 1. Define the Tag in `types.go`
Apply the `+k8s:maxProperties` tag to the `Data` field in the `ConfigMap` struct definition.

**File:** `staging/src/k8s.io/api/core/v1/types.go`
```go
type ConfigMap struct {
    // ...
    // Data contains the configuration data. Each key must consist of alphanumeric
    // characters, '-', '_' or '.'. The keys cannot represent paths in a directory tree.
    // +k8s:maxProperties=16 // Example: Limit to 16 entries
    Data map[string]string `json:"data,omitempty" protobuf:"bytes,2,rep,name=data"`
    // ...
}
```

### 2. Update Handwritten Validation
In the corresponding handwritten validation function for `ConfigMap` objects, remove or mark as covered the explicit checks for the map's size.

**File:** `pkg/apis/core/validation/validation.go`

```go
func ValidateConfigMap(configMap *core.ConfigMap, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}

    // ... other validations ...

    // Original handwritten validation for map size (example):
    // if len(configMap.Data) > 16 { // Assuming 16 is the desired limit
    //     allErrs = append(allErrs, field.TooMany(fldPath.Child("data"), len(configMap.Data), 16))
    // }

    // After adding +k8s:maxProperties=16, the generated code handles this.
    // If still present for backward compatibility during migration, mark the error as covered:
    if len(configMap.Data) > 16 { // Assuming 16 is the desired limit
        allErrs = append(allErrs, field.TooMany(fldPath.Child("data"), len(configMap.Data), 16).MarkCoveredByDeclarative())
    }

    // ... other validations ...
    return allErrs
}
```
By using `+k8s:maxProperties`, the API server automatically enforces the maximum number of entries in the `Data` map, simplifying validation logic.

## Test Coverage

When using `+k8s:maxProperties`, you should add declarative validation tests to verify that maps with more than the specified number of properties are rejected.

### Example

Suppose you have a map that can contain at most 2 properties.

**File:** `pkg/apis/example/v1/types.go`
```go
type MyStruct struct {
    // +k8s:maxProperties=2
    Labels map[string]string `json:"labels,omitempty"`
}
```

Your `declarative_validation_test.go` should include test cases to cover maps that are within the limit, at the limit, and over the limit.

**File:** `pkg/apis/example/validation/declarative_validation_test.go`
```go
func TestDeclarativeValidateMaxProperties(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyStruct
        expectedErrs field.ErrorList
    }{
        "map with fewer than max properties": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Labels = map[string]string{"key1": "value1"}
            }),
            expectedErrs: field.ErrorList{},
        },
        "map with exactly max properties": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Labels = map[string]string{"key1": "value1", "key2": "value2"}
            }),
            expectedErrs: field.ErrorList{},
        },
        "map with more than max properties": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Labels = map[string]string{"key1": "value1", "key2": "value2", "key3": "value3"}
            }),
            expectedErrs: field.ErrorList{
                field.TooMany(field.NewPath("spec", "labels"), 3, 2).WithOrigin("maxProperties"),
            },
        },
    }
    // ...
}
```

In this example:
1.  We test three scenarios for the number of properties relative to `maxProperties`.
2.  For the case that exceeds the limit, we expect a `field.TooMany` error, which clearly states the actual and maximum allowed number of properties.
3.  The error origin is `maxProperties`, corresponding to the `+k8s:maxProperties` tag.