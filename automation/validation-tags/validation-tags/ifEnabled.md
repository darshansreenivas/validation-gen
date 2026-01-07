# +k8s:ifEnabled

## Description
Applies the chained validation only if a specific feature gate/option is enabled.

## Scope
`Field`, `Type`

## Supported Go Types
Any Go type. This tag acts as a conditional wrapper for other validation tags applied to the field or type.

## Arguments
`<OptionName>` (Required): The name of the feature gate or option.

## Payload
`+<validation-tag>`: The validation tag(s) to apply if the option is enabled.

## Stability
**Alpha**

## Usage

### Field
```go
type MyStruct struct {
    // If "MyFeature" is enabled, this field is required.
    // +k8s:ifEnabled(MyFeature)=+k8s:required
    // +k8s:optional
    Config string `json:"config,omitempty"`
}
```

### Type
```go
// If "AlphaFeature" is enabled, this type is validated as an enum.
// +k8s:ifEnabled(AlphaFeature)=+k8s:enum
type FeatureEnum string

const (
    DefaultValue FeatureEnum = "Default"
    AlphaValue   FeatureEnum = "Alpha"
)
```
## Migrating from Handwritten Validation

Similar to `+k8s:ifDisabled`, the `+k8s:ifEnabled` tag simplifies conditional validation based on feature gates. It allows you to express that certain validation rules only apply when a feature is active, moving this logic from imperative Go code to declarative API definitions.

## Detailed Example: Conditionally Required Field

This example demonstrates how `+k8s:ifEnabled` can make a field conditionally required when a specific feature gate is enabled.

### 1. Define the Tag in `types.go`
Apply the `+k8s:ifEnabled` tag to a field, chaining it with `+k8s:required`.

**File:** `pkg/apis/example/v1/types.go`
```go
type MyResourceSpec struct {
    // This field is required only if "ExperimentalFeature" is enabled.
    // +k8s:ifEnabled(ExperimentalFeature)=+k8s:required
    // +k8s:optional
    ExperimentalConfig string `json:"experimentalConfig,omitempty"`
    // ...
}
```

### 2. Update Handwritten Validation (if applicable)
If there was prior handwritten validation that manually checked `ExperimentalFeature`'s status to make `ExperimentalConfig` required, that logic can now be removed or marked as covered.

**File:** `pkg/apis/example/validation/validation.go`
```go
func ValidateMyResourceSpec(spec *MyResourceSpec, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}

    // Original handwritten logic (example):
    // if utilfeature.DefaultFeatureGate.Enabled(features.ExperimentalFeature) && len(spec.ExperimentalConfig) == 0 {
    //     allErrs = append(allErrs, field.Required(fldPath.Child("experimentalConfig"), "must be specified when ExperimentalFeature is enabled"))
    // }

    // With +k8s:ifEnabled, the generated code handles this. If still present for backward compatibility:
    // if utilfeature.DefaultFeatureGate.Enabled(features.ExperimentalFeature) && len(spec.ExperimentalConfig) == 0 {
    //     allErrs = append(allErrs, field.Required(fldPath.Child("experimentalConfig"), "must be specified when ExperimentalFeature is enabled").MarkCoveredByDeclarative())
    // }

    return allErrs
}
```
The `+k8s:ifEnabled` tag makes the conditional requirement explicit in the API definition, reducing the need for imperative checks in handwritten code.

## Test Coverage

When using `+k8s:ifEnabled`, your declarative validation tests should cover both scenarios: when the specified feature gate is enabled and when it is disabled. This ensures that the conditional validation is applied correctly in both cases.

To control feature gates within a test, you can use the `featuregatetesting.SetFeatureGateDuringTest` helper function.

### Example: Conditional Enum Inclusion



Suppose you have an `AlphaValue` that is only a valid enum value when the `AlphaFeature` feature gate is enabled. You can achieve this by excluding it whenever the feature gate is *disabled*.



**File:** `pkg/apis/example/v1/types.go`

```go

// +k8s:enum

type FeatureEnum string



const (

    DefaultValue FeatureEnum = "Default"

    // +k8s:ifDisabled(AlphaFeature)=+k8s:enumExclude

    AlphaValue   FeatureEnum = "Alpha"

)



type MyResource struct {

    Feature FeatureEnum `json:"feature"`

}

```



Your `declarative_validation_test.go` should include tests for both states of the `AlphaFeature` feature gate.



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

    // Scenario 1: AlphaFeature feature gate is ENABLED

    //

    t.Run("AlphaFeature=true", func(t *testing.T) {

        featuregatetesting.SetFeatureGateDuringTest(t, utilfeature.DefaultFeatureGate, features.AlphaFeature, true)



        // Test that "Alpha" is an allowed value

        validObj := &example.MyResource{Feature: example.AlphaValue}

        if errs := validateObject(validObj); len(errs) > 0 {

            t.Errorf("expected no errors, but got: %v", errs)

        }

    })



    //

    // Scenario 2: AlphaFeature feature gate is DISABLED

    //

    t.Run("AlphaFeature=false", func(t *testing.T) {

        featuregatetesting.SetFeatureGateDuringTest(t, utilfeature.DefaultFeatureGate, features.AlphaFeature, false)



        // Test that "Alpha" is now a forbidden value

        invalidObj := &example.MyResource{Feature: example.AlphaValue}

        expectedErrs := field.ErrorList{

            field.NotSupported(field.NewPath("spec", "feature"), "Alpha", []string{"Default"}),

        }

        if errs := validateObject(invalidObj); !reflect.DeepEqual(errs, expectedErrs) {

            t.Errorf("expected errors %v, but got: %v", expectedErrs, errs)

        }

    })

}

```



In this example:

1.  We use `+k8s:ifDisabled(AlphaFeature)=+k8s:enumExclude` to make `AlphaValue` invalid when the feature gate is OFF.

2.  The tests verify this behavior by enabling and disabling the feature gate.
