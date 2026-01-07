# +k8s:ifDisabled

## Description
Applies the chained validation only if a specific feature gate/option is disabled.

## Scope
`Field`, `Type`

## Supported Go Types
Any Go type. This tag acts as a conditional wrapper for other validation tags applied to the field or type.

## Arguments
`<OptionName>` (Required): The name of the feature gate or option.

## Payload
`+<validation-tag>`: The validation tag(s) to apply if the option is disabled.

## Stability
**Alpha**

## Usage

### Field
```go
type MyStruct struct {
    // If "MyFeature" is disabled, this field is required.
    // +k8s:ifDisabled(MyFeature)=+k8s:required
    // +k8s:optional
    Config string `json:"config,omitempty"`
}
```

### Type
```go
// If "LegacySupport" feature is disabled, this type is validated as an enum.
// +k8s:ifDisabled(LegacySupport)=+k8s:enum
type Protocol string

const (
    NewProtocol    Protocol = "New"
    LegacyProtocol Protocol = "Legacy"
)
```
In the above example, if `LegacySupport` is disabled, the `Protocol` type will be validated as an enum.

## Migrating from Handwritten Validation

The `+k8s:ifDisabled` tag directly translates conditional logic based on feature gates into declarative validation. This eliminates the need for boilerplate `if featuregate.IsDisabled(...)` checks in handwritten validation functions, making the validation rules more explicit and maintainable.

## Detailed Example: Conditional Enum Exclusion

This example demonstrates how `+k8s:ifDisabled` can be used to conditionally exclude an enum value based on the state of a feature gate, replacing handwritten checks for feature status.

### 1. Define the Enum Type with Conditional Exclusion in `types.go`
Apply the `+k8s:ifDisabled` tag to a `const` value, chaining it with `+k8s:enumExclude`.

**File:** `pkg/apis/example/v1/types.go`
```go
// +k8s:enum
type FeatureDependentState string

const (
	StateEnabled  FeatureDependentState = "Enabled"
	StateDisabled FeatureDependentState = "Disabled"
	// This state is only valid if "MyFeatureGate" is enabled.
	// If "MyFeatureGate" is disabled, "ConditionalState" will not be a valid option.
	// +k8s:ifDisabled(MyFeatureGate)=+k8s:enumExclude
	ConditionalState FeatureDependentState = "Conditional"
)
```

### 2. Update Handwritten Validation (if applicable)
If there was prior handwritten validation that manually checked `MyFeatureGate`'s status to validate `ConditionalState`, that logic can now be removed or marked as covered.

**File:** `pkg/apis/example/validation/validation.go`
```go
func ValidateFeatureDependentState(state FeatureDependentState, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}

    // Original handwritten logic (example):
    // if state == ConditionalState && utilfeature.DefaultFeatureGate.Disabled(features.MyFeatureGate) {
    //     allErrs = append(allErrs, field.Invalid(fldPath, state, "ConditionalState is not allowed when MyFeatureGate is disabled"))
    // }

    // With +k8s:ifDisabled, the generated code handles this. If still present for backward compatibility:
    // This check would ideally be removed, or its error marked as covered by the declarative tag.
    // For simplicity, we'll show how it would be marked if kept during transition.
    // if state == ConditionalState && utilfeature.DefaultFeatureGate.Disabled(features.MyFeatureGate) {
    //     allErrs = append(allErrs, field.Invalid(fldPath, state, "ConditionalState is not allowed when MyFeatureGate is disabled").MarkCoveredByDeclarative())
    // }

    return allErrs
}
```
The `+k8s:ifDisabled` tag makes the conditional validation rule explicit in the API definition, reducing the need for imperative code.

## Test Coverage

When using `+k8s:ifDisabled`, your declarative validation tests should cover both scenarios: when the specified feature gate is enabled and when it is disabled. This ensures that the conditional validation is applied correctly in both cases.

To control feature gates within a test, you can use the `featuregatetesting.SetFeatureGateDuringTest` helper function.

### Example: Conditional Enum Exclusion

Suppose you have a `Protocol` enum where `"Legacy"` is a valid option only when the `LegacySupport` feature gate is disabled.

**File:** `pkg/apis/example/v1/types.go`
```go
// +k8s:enum
type Protocol string

const (
    ProtocolModern Protocol = "Modern"
    // +k8s:ifDisabled(LegacySupport)=+k8s:enumExclude
    ProtocolLegacy Protocol = "Legacy"
)

type MyResource struct {
    Protocol Protocol `json:"protocol"`
}
```

Your `declarative_validation_test.go` should include tests for both states of the `LegacySupport` feature gate.

**File:** `pkg/apis/example/validation/declarative_validation_test.go` (hypothetical example)
```go
import (
    "testing"
    "k8s.io/apimachinery/pkg/util/validation/field"
    "k8s.io/apiserver/pkg/features"
    utilfeature "k8s.io/apiserver/pkg/util/feature"
    featuregatetesting "k8s.io/component-base/featuregate/testing"
    "..." // other imports
)

func TestDeclarativeValidateConditionalEnum(t *testing.T) {
    //
    // Scenario 1: LegacySupport feature gate is ENABLED
    //
    t.Run("LegacySupport=true", func(t *testing.T) {
        featuregatetesting.SetFeatureGateDuringTest(t, utilfeature.DefaultFeatureGate, features.LegacySupport, true)

        // Test that "Legacy" is an allowed value
        validObj := &example.MyResource{Protocol: example.ProtocolLegacy}
        if errs := validateObject(validObj); len(errs) > 0 {
            t.Errorf("expected no errors, but got: %v", errs)
        }
    })

    //
    // Scenario 2: LegacySupport feature gate is DISABLED
    //
    t.Run("LegacySupport=false", func(t *testing.T) {
        featuregatetesting.SetFeatureGateDuringTest(t, utilfeature.DefaultFeatureGate, features.LegacySupport, false)

        // Test that "Legacy" is now a forbidden value
        invalidObj := &example.MyResource{Protocol: example.ProtocolLegacy}
        expectedErrs := field.ErrorList{
            field.NotSupported(field.NewPath("spec", "protocol"), "Legacy", []string{"Modern"}),
        }
        if errs := validateObject(invalidObj); !reflect.DeepEqual(errs, expectedErrs) {
            t.Errorf("expected errors %v, but got: %v", expectedErrs, errs)
        }
    })
}
```

In this example:
1.  We define two sub-tests, one for each state of the `LegacySupport` feature gate.
2.  `featuregatetesting.SetFeatureGateDuringTest` is used to enable or disable the feature gate for the duration of each sub-test.
3.  When the feature is enabled, the `ifDisabled` condition is not met, so `ProtocolLegacy` is a valid enum value, and no error is expected.
4.  When the feature is disabled, the `ifDisabled` condition is met, `+k8s:enumExclude` takes effect, and `ProtocolLegacy` becomes an invalid value, resulting in a `field.NotSupported` error.