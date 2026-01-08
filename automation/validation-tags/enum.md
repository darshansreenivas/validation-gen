# +k8s:enum

## Description
Marks a string type as an enumeration. All `const` values defined for this type in the same package are considered valid values.

## Scope
`Type` (Must **NOT** be applied to struct fields)

## Supported Go Types
`string` (and any alias of `string`)

## Stability
**Beta**

## Usage

### Type (Required)
The `+k8s:enum` tag MUST be applied to the type definition. All `const` values defined for this type in the same package are automatically considered valid values for the enumeration.

```go
// +k8s:enum
type Protocol string

const (
    TCP Protocol = "TCP"
    UDP Protocol = "UDP"
)
```

### Field
To use the enum, reference the type in a struct field. **Do not** apply the `+k8s:enum` tag to the field itself; the validation is automatically inherited from the type definition. Both value and pointer fields are supported.

```go
type ServicePort struct {
    // Validation is inherited from the Protocol type definition
    Protocol Protocol  `json:"protocol,omitempty"`
    Mode     *Protocol `json:"mode,omitempty"`
}
```

### Map & Slice
To validate items in a map or slice, the enum type must be used as the key or value.

```go
type ProtocolList struct {
    // Validates that map values are valid Protocol values
    Protocols map[string]Protocol `json:"protocols,omitempty"`
}

type ProtocolSlice struct {
    // Validates that all items in the slice are valid Protocol values
    Protocols []Protocol `json:"protocols,omitempty"`
}
```

## Conditional Exclusions
Enum values can be conditionally excluded based on feature gates or other options using `+k8s:ifEnabled` or `+k8s:ifDisabled` combined with `+k8s:enumExclude`. See [enumExclude.md](enumExclude.md) for more details.

## Migrating from Handwritten Validation

When converting a field with handwritten enum validation to use the declarative `+k8s:enum` tag, follow these steps:

1.  **Define the Enum Type**: Create a new type alias for `string` and apply the `+k8s:enum` tag. Define the valid constant values for this new type.
2.  **Update Field**: Change the field in your API struct to use the new enum type.
3.  **Mark Errors**: In the handwritten validation function, mark the error as covered by declarative validation using `.MarkCoveredByDeclarative()`.

## Detailed Example: Validating a Taint Effect

This example demonstrates how to migrate the `Effect` field in a `Taint` struct from handwritten validation to a declarative enum.

### 1. Define the Enum Type in `types.go`
Create a new `TaintEffect` type with the `+k8s:enum` tag and define its constant values.

**File:** `staging/src/k8s.io/api/core/v1/types.go`
```go
// +k8s:enum
type TaintEffect string

const (
	TaintEffectNoSchedule TaintEffect = "NoSchedule"
	TaintEffectNoExecute  TaintEffect = "NoExecute"
)
```

### 2. Update the API Struct
Modify the `Taint` struct to use the new `TaintEffect` enum type for the `Effect` field.

**File:** `staging/src/k8s.io/api/core/v1/types.go`
```go
type Taint struct {
    Key string `json:"key" protobuf:"bytes,1,name=key"`
    Value string `json:"value,omitempty" protobuf:"bytes,2,opt,name=value"`
    Effect TaintEffect `json:"effect" protobuf:"bytes,3,name=effect,casttype=TaintEffect"`
}
```

### 3. Update Handwritten Validation
In the existing validation logic, mark the validation for the `Effect` field as covered by declarative validation.

**File:** `pkg/apis/core/validation/validation.go`

```go
func validateTaint(taint *core.Taint, fldPath *field.Path) field.ErrorList {
    var allErrs field.ErrorList
    // ... other validation for Key and Value ...

    // Original handwritten validation for Effect:
    //
    // validEffects := sets.New(string(core.TaintEffectNoSchedule), string(core.TaintEffectNoExecute))
    // if !validEffects.Has(string(taint.Effect)) {
    //     allErrs = append(allErrs, field.NotSupported(fldPath.Child("effect"), taint.Effect, sets.List(validEffects)))
    // }

    // Mark as covered by declarative validation:
    allErrs = append(allErrs, field.Invalid(fldPath.Child("effect"), taint.Effect, "invalid value").MarkCoveredByDeclarative())

    return allErrs
}
```

By following this pattern, you ensure that the declarative validation is correctly implemented and can be verified by the validation tooling, while maintaining compatibility with existing handwritten checks during the transition period.

## Test Coverage

When using `+k8s:enum`, you should add declarative validation tests to verify that only the defined constant values are accepted.

### Example

Suppose you have a `TaintEffect` enum defined as follows:

**File:** `staging/src/k8s.io/api/core/v1/types.go`
```go
// +k8s:enum
type TaintEffect string

const (
	TaintEffectNoSchedule TaintEffect = "NoSchedule"
	TaintEffectNoExecute  TaintEffect = "NoExecute"
)
```

Your `declarative_validation_test.go` should include test cases for both valid and invalid enum values.

**File:** `pkg/apis/core/validation/declarative_validation_test.go` (hypothetical example)
```go
func TestDeclarativeValidateTaint(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        core.Taint
        expectedErrs field.ErrorList
    }{
        "valid taint effect": {
            input: core.Taint{
                Key: "key",
                Effect: core.TaintEffectNoSchedule,
            },
            expectedErrs: field.ErrorList{},
        },
        "invalid taint effect": {
            input: core.Taint{
                Key: "key",
                Effect: "InvalidEffect",
            },
            expectedErrs: field.ErrorList{
                field.NotSupported(
                    field.NewPath("effect"),
                    "InvalidEffect",
                    []string{"NoExecute", "NoSchedule"},
                ).MarkCoveredByDeclarative(), // Assuming there is also handwritten validation
            },
        },
    }
    // ...
}
```

In this example:
1.  We test both a valid enum value (`TaintEffectNoSchedule`) and an invalid one (`"InvalidEffect"`).
2.  For the invalid case, we expect a `field.NotSupported` error, which lists the allowed values.
3.  The error can be marked with `.MarkCoveredByDeclarative()` if there is also a handwritten validation for the same field. If the validation is purely declarative, you would use `.MarkDeclarativeNative()` instead.

