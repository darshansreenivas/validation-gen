# +k8s:listType

## Description
Defines how a list should be treated by Server-Side Apply and validation.

## Scope
`Field`, `Type`

## Supported Go Types
`[]any`, `*[]any` (and any alias of these types).

## Payload
`atomic` | `set` | `map`
*   `atomic`: The list is treated as a single value (replaced entirely on update).
*   `set`: The list is treated as a set (items must be unique; uniqueness is based on the entire item's value).
*   `map`: The list is treated as an associative map (requires `+k8s:listMapKey`; uniqueness is based on the specified key(s)).

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
// Defines MyList as a set. All items in lists of this type must be unique.
// +k8s:listType=set
type MyList []string

type MyStruct struct {
    Values MyList `json:"values"`
}
```
The `listType` tag is crucial for Server-Side Apply (SSA) to correctly merge updates to lists and for validation to enforce uniqueness where appropriate.

## Migrating from Handwritten Validation

The `+k8s:listType` tag is fundamental for correctly handling lists in Kubernetes APIs, particularly with Server-Side Apply (SSA) and enforcing uniqueness. It effectively replaces manual logic for:
*   **Merge behavior**: SSA uses `listType` to determine how to merge changes to lists.
*   **Uniqueness checks**: For `listType=set` and `listType=map`, the generated validation code automatically enforces uniqueness based on either the item's full value or the specified `listMapKey(s)`. This eliminates the need for handwritten loops and maps to track seen items.

When migrating from handwritten validation that performed these tasks, the explicit checks can often be removed or marked as covered by the declarative tags.

## Detailed Example: Managing Pod Containers with `listType=map`

This example demonstrates how `+k8s:listType=map` and `+k8s:listMapKey` are used to manage a list of `Container` objects within a `PodSpec`, allowing them to be uniquely identified and merged by their `name`.

### 1. Define the Tag in `types.go`
Apply `+k8s:listType=map` and `+k8s:listMapKey=name` to the `Containers` field in `PodSpec`.

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
If there was prior handwritten validation that ensured container names were unique, that logic can now be removed or marked as covered.

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
By declaring `+k8s:listType=map` and `+k8s:listMapKey=name`, the API machinery correctly handles merging `Containers` by their `name` field during updates and automatically enforces uniqueness of container names within the list.

## Test Coverage

The `+k8s:listType` tag provides structural information to the API server and validation system. It does not have a direct validation rule to test but is essential for the correct behavior of other validations on lists. Its correctness is demonstrated through the tests for tags like `+k8s:listMapKey`, `+k8s:item`, and `+k8s:unique`.

### Example: Testing a `listType=map`

When you set `listType=map`, you typically also provide one or more `+k8s:listMapKey` tags. You can then test validations that depend on this map-like behavior, such as uniqueness checks.

Suppose you have a list of environment variables where each variable must have a unique `name`.

**File:** `pkg/apis/example/v1/types.go`
```go
type EnvVar struct {
    Name  string `json:"name"`
    Value string `json:"value"`
}

type MyContainer struct {
    // +k8s:listType=map
    // +k8s:listMapKey=name
    // +k8s:unique
    Env []EnvVar `json:"env"`
}
```

Your `declarative_validation_test.go` should verify that duplicate `name` entries are rejected, which implicitly tests that `listType=map` and `listMapKey` are working correctly.

**File:** `pkg/apis/example/validation/declarative_validation_test.go` (hypothetical example)
```go
func TestDeclarativeValidateListType(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyContainer
        expectedErrs field.ErrorList
    }{
        "unique env var names": {
            input: example.MyContainer{
                Env: []example.EnvVar{
                    {Name: "VAR_A", Value: "value_a"},
                    {Name: "VAR_B", Value: "value_b"},
                },
            },
            expectedErrs: field.ErrorList{},
        },
        "duplicate env var names": {
            input: example.MyContainer{
                Env: []example.EnvVar{
                    {Name: "VAR_A", Value: "value_a"},
                    {Name: "VAR_A", Value: "another_value"},
                },
            },
            expectedErrs: field.ErrorList{
                field.Duplicate(field.NewPath("spec", "env").Key("VAR_A"), "VAR_A"),
            },
        },
    }
    // ...
}
```

In this example, the test for `+k8s:unique` on the `Env` list relies on the list being treated as a map with `name` as the key. The successful validation of uniqueness indirectly confirms that `listType=map` and `listMapKey=name` are correctly interpreted by the validation system.