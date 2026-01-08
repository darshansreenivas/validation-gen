# +k8s:listMapKey

## Description
Specifies the field name(s) to use as the key for a list with `listType=map`.

## Scope
`Field`, `Type`

## Supported Go Types
`[]struct{...}` (slices of structs) or aliases of such types, where the struct contains the specified key field(s).

## Payload
`<field-name>` | `<field-name-1>,<field-name-2>,...`

## Stability
**Stable**

## Usage

### Field
```go
type PodSpec struct {
    // Defines Containers as a list that merges by the "name" field.
    // +k8s:listType=map
    // +k8s:listMapKey=name
    Containers []Container `json:"containers"`
}
```

### Type
```go
// Defines MyCustomList as a map-like list using "id" as the key.
// +k8s:listType=map
// +k8s:listMapKey=id
type MyCustomList []MyItem

type MyItem struct {
    ID    string `json:"id"`
    Value string `json:"value"`
}

type MyStruct struct {
    Items MyCustomList `json:"items"`
}
```
You can also specify multiple key fields for composite keys:
```go
// +k8s:listType=map
// +k8s:listMapKey=key1,key2
type CompositeList []CompositeItem
```
This implies that `listMapKey` is always used in conjunction with `listType=map`.

## Migrating from Handwritten Validation

The `+k8s:listMapKey` tag is used in conjunction with `+k8s:listType=map` to specify which field(s) within a struct slice should act as the unique key(s) for elements in that list. This is crucial for Server-Side Apply (SSA) to correctly merge updates to list items and for validation to enforce uniqueness. It replaces handwritten logic that manually extracts and compares these key fields to determine item identity.

## Detailed Example: Unique Container Names

This example demonstrates how `+k8s:listMapKey=name` ensures that containers within a `PodSpec` are uniquely identified by their `name` field, enabling correct merging and validation.

### 1. Define the Tag in `types.go`
Apply `+k8s:listMapKey=name` along with `+k8s:listType=map` to the `Containers` field in `PodSpec`.

**File:** `staging/src/k8s.io/api/core/v1/types.go`
```go
type PodSpec {
    // ...
    // List of containers belonging to the pod.
    // +patchMergeKey=name
    // +patchStrategy=merge
    // +k8s:listType=map
    // +k8s:listMapKey=name
    Containers []Container `json:"containers" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,2,rep,name=containers"`
    // ...
}
```

### 2. Update Handwritten Validation
If there was prior handwritten validation that manually checked for unique container names, that logic can now be removed or marked as covered.

**File:** `pkg/apis/core/validation/validation.go`
```go
func ValidatePodSpec(spec *core.PodSpec, fldPath *field.Path, opts PodValidationOptions) field.ErrorList {
    allErrs := field.ErrorList{}
    seenContainerNames := sets.New[string]() // Used for handwritten uniqueness check

    for i, container := range spec.Containers {
        idxPath := fldPath.Child("containers").Index(i)

        // Original handwritten uniqueness check (example):
        // if seenContainerNames.Has(container.Name) {
        //     allErrs = append(allErrs, field.Duplicate(idxPath.Child("name"), container.Name))
        // }
        // seenContainerNames.Insert(container.Name)

        // With +k8s:listType=map and +k8s:listMapKey=name, this check is generated automatically.
        // If still present for backward compatibility during migration, mark the error as covered:
        if seenContainerNames.Has(container.Name) { // Assuming seenContainerNames is populated before this check
             allErrs = append(allErrs, field.Duplicate(idxPath.Child("name"), container.Name).MarkCoveredByDeclarative())
        }
        seenContainerNames.Insert(container.Name) // Still insert to avoid breaking other checks that might rely on this set.

        // ... other container validations ...
    }
    return allErrs
}
```
By declaring `+k8s:listMapKey=name`, the API machinery correctly handles merging `Containers` by their `name` field during updates and automatically enforces uniqueness of container names within the list.

## Test Coverage

The `+k8s:listMapKey` tag is a structural hint to the validation system and does not, by itself, enforce a validation rule. Its correctness is tested indirectly through other validation tags that rely on it, such as `+k8s:item` and `+k8s:unique`.

### Example: Testing Uniqueness of `listMapKey`

A common use case for `listMapKey` is to enforce that all items in a list have a unique value for the specified key field(s). This is achieved by combining `+k8s:listType=map` and `+k8s:listMapKey` with `+k8s:unique`.

Suppose you have a list of containers where each container must have a unique name.

**File:** `pkg/apis/example/v1/types.go`
```go
type Container struct {
    Name string `json:"name"`
    Image string `json:"image"`
}

type MyPodSpec struct {
    // +k8s:listType=map
    // +k8s:listMapKey=name
    // +k8s:unique
    Containers []Container `json:"containers"`
}
```

Your `declarative_validation_test.go` should include test cases to verify that duplicate names are rejected.

**File:** `pkg/apis/example/validation/declarative_validation_test.go` (hypothetical example)
```go
func TestDeclarativeValidateListMapKey(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyPodSpec
        expectedErrs field.ErrorList
    }{
        "unique container names": {
            input: example.MyPodSpec{
                Containers: []example.Container{
                    {Name: "container-a", Image: "image-a"},
                    {Name: "container-b", Image: "image-b"},
                },
            },
            expectedErrs: field.ErrorList{},
        },
        "duplicate container names": {
            input: example.MyPodSpec{
                Containers: []example.Container{
                    {Name: "container-a", Image: "image-a"},
                    {Name: "container-a", Image: "image-b"},
                },
            },
            expectedErrs: field.ErrorList{
                field.Duplicate(field.NewPath("spec", "containers").Key("container-a"), "container-a"),
            },
        },
    }
    // ...
}
```

In this example:
1.  We test a list with unique container names, which is valid.
2.  We test a list with a duplicate name (`"container-a"`) and expect a `field.Duplicate` error. The error path uses `.Key("container-a")` because `name` is the `listMapKey`, demonstrating how the key is used to identify the problematic item.