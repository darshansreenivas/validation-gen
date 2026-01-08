# +k8s:unique

## Description
Enforces uniqueness of items in a list without implying ownership semantics (unlike `listType`). This tag ensures that all elements within a list are unique according to a defined strategy.

## Scope
`Field`, `Type`

## Supported Go Types
`[]any`, `*[]any` (and any alias of these types).

## Payload
`set` | `map`
*   `set`: Enforces uniqueness where each item in the list is treated as a distinct value. For structs, uniqueness is based on the entire struct's value.
*   `map`: Enforces uniqueness based on specific key fields within each item. Requires `+k8s:listMapKey` to specify the key(s) to use for uniqueness.

## Stability
**Alpha**

## Usage

### Field (unique=set)
```go
type MyStruct struct {
    // Each string in this list must be unique.
    // +k8s:listType=atomic // The actual merge strategy
    // +k8s:unique=set
    Tags []string `json:"tags"`
}
```

### Field (unique=map)
```go
type MyStruct struct {
    // Each item in this list must be unique by its 'name' field.
    // +k8s:listType=atomic // The actual merge strategy
    // +k8s:unique=map
    // +k8s:listMapKey=name
    Items []MyItem `json:"items"`
}

type MyItem struct {
    Name  string `json:"name"`
    Value string `json:"value"`
}
```

### Type (unique=set)
```go
// Each element in this type of list must be unique.
// +k8s:listType=atomic
// +k8s:unique=set
type UniqueStrings []string

type MyStruct struct {
    UniqueNames UniqueStrings `json:"uniqueNames"`
}
```
`+k8s:unique` is typically used in conjunction with `+k8s:listType` to refine list behavior, especially for patch strategies and validation.

## Migrating from Handwritten Validation

The `+k8s:unique` tag declaratively enforces uniqueness constraints on list items, directly replacing handwritten validation logic that would manually check for duplicate entries. This includes cases where uniqueness is determined by the entire item (`unique=set`) or by specific key fields within an item (`unique=map`).

When `+k8s:unique` is used, the generated validation code automatically handles the uniqueness check. Any corresponding handwritten logic for uniqueness can then be removed or marked as covered.

## Detailed Example: Ensuring Unique Tags and Named Items

This example demonstrates how `+k8s:unique=set` enforces uniqueness for a list of simple strings, and `+k8s:unique=map` (with `+k8s:listMapKey`) enforces uniqueness based on an `ID` field for a list of complex objects.

### 1. Define the Tag in `types.go`
Apply `+k8s:unique=set` to a string slice and `+k8s:unique=map` with `+k8s:listMapKey` to a slice of structs.

**File:** `pkg/apis/example/v1/types.go`
```go
type MyResourceSpec struct {
    // Each string in this slice must be unique.
    // +k8s:listType=atomic // The actual merge strategy for Server-Side Apply
    // +k8s:unique=set
    StringTags []string `json:"stringTags,omitempty"`

    // Each item in this slice must be unique by its 'id' field.
    // +k8s:listType=atomic // The actual merge strategy for Server-Side Apply
    // +k8s:unique=map
    // +k8s:listMapKey=id
    NamedItems []MyNamedItem `json:"namedItems,omitempty"`
}

type MyNamedItem struct {
    ID    string `json:"id"`
    Value string `json:"value"`
}
```

### 2. Update Handwritten Validation
In the corresponding handwritten validation function, remove or mark as covered the explicit checks for uniqueness for `StringTags` and `NamedItems`.

**File:** `pkg/apis/example/validation/validation.go`
```go
func ValidateMyResourceSpec(spec *MyResourceSpec, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}

    // Original handwritten uniqueness check for StringTags (example):
    // seenStringTags := sets.New[string]()
    // for i, tag := range spec.StringTags {
    //     if seenStringTags.Has(tag) {
    //         allErrs = append(allErrs, field.Duplicate(fldPath.Child("stringTags").Index(i), tag))
    //     }
    //     seenStringTags.Insert(tag)
    // }

    // After adding +k8s:unique=set, this check is generated automatically.
    // If still present for backward compatibility during migration:
    seenStringTags := sets.New[string]()
    for i, tag := range spec.StringTags {
        if seenStringTags.Has(tag) {
            allErrs = append(allErrs, field.Duplicate(fldPath.Child("stringTags").Index(i), tag).MarkCoveredByDeclarative())
        }
        seenStringTags.Insert(tag)
    }

    // Original handwritten uniqueness check for NamedItems (example, by ID):
    // seenItemIDs := sets.New[string]()
    // for i, item := range spec.NamedItems {
    //     if seenItemIDs.Has(item.ID) {
    //         allErrs = append(allErrs, field.Duplicate(fldPath.Child("namedItems").Index(i).Child("id"), item.ID))
    //     }
    //     seenItemIDs.Insert(item.ID)
    // }

    // After adding +k8s:unique=map and +k8s:listMapKey=id, this check is generated automatically.
    // If still present for backward compatibility during migration:
    seenItemIDs := sets.New[string]()
    for i, item := range spec.NamedItems {
        if seenItemIDs.Has(item.ID) {
            allErrs = append(allErrs, field.Duplicate(fldPath.Child("namedItems").Index(i).Child("id"), item.ID).MarkCoveredByDeclarative())
        }
        seenItemIDs.Insert(item.ID)
    }

    // ... other validations ...
    return allErrs
}
```
The `+k8s:unique` tag eliminates the need for manual uniqueness checks, making the API definition more declarative and reducing the amount of handwritten validation code.

## Test Coverage

The `+k8s:unique` tag is a structural hint to the validation system and does not, by itself, enforce a validation rule. Its correctness is tested indirectly through other validation tags that rely on it, such as `+k8s:item` and `+k8s:unique`.

### Example: `listType=set`

When `listType=set`, uniqueness is based on the entire value of each item in the list.

**File:** `pkg/apis/example/v1/types.go`
```go
type MyStruct struct {
    // +k8s:listType=set
    // +k8s:unique
    Values []string `json:"values,omitempty"`
}
```

Your `declarative_validation_test.go` should include a test case with duplicate items.

**File:** `pkg/apis/example/validation/declarative_validation_test.go`
```go
func TestDeclarativeValidateUniqueSet(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyStruct
        expectedErrs field.ErrorList
    }{
        "unique items in set": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Values = []string{"a", "b", "c"}
            }),
            expectedErrs: field.ErrorList{},
        },
        "duplicate items in set": {
            input: mkMyStruct(func(obj *example.MyStruct) {
                obj.Values = []string{"a", "b", "a"}
            }),
            expectedErrs: field.ErrorList{
                field.Duplicate(field.NewPath("spec", "values").Index(2), "a"),
            },
        },
    }
    // ...
}
```

### Example: `listType=map`

When `listType=map`, uniqueness is based on the fields specified by `+k8s:listMapKey`. See the `listMapKey` documentation for a detailed example.