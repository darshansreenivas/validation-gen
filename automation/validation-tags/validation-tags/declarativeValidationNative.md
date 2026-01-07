# +k8s:declarativeValidationNative

## Description
Marker indicating that validation is purely declarative and has no handwritten equivalent. Affects error reporting.

## Scope
`Field`

## Supported Go Types
Any Go type for which other declarative validation tags are applied.

## Stability
**Stable**

## Usage

The `+k8s:declarativeValidationNative` tag is applied to a field. It must be used in conjunction with other stable validation tags.

### Field
```go
type MyStruct struct {
    // This field will be validated declaratively, and any errors will be reported directly.
    // +k8s:declarativeValidationNative
    // +k8s:required
    // +k8s:format=k8s-uuid
    ID string `json:"id"`
}
## Migrating from Handwritten Validation

The `+k8s:declarativeValidationNative` tag explicitly states that there is no handwritten validation for the field. If a field previously had handwritten validation, but is now solely validated declaratively, this tag should be added. This helps the validation system correctly attribute errors and ensures that the generated code does not attempt to mark non-existent handwritten errors as "covered."

## Detailed Example

Consider a scenario where an `ID` field has always been validated purely through declarative tags, without any corresponding handwritten Go validation.

### 1. Define the Tag in `types.go`
Apply the `+k8s:declarativeValidationNative` tag along with other declarative validation tags directly to the field.

**File:** `pkg/apis/example/v1/types.go`
```go
type MyResourceSpec struct {
    // This ID field is always validated declaratively, and has no handwritten equivalent.
    // +k8s:declarativeValidationNative
    // +k8s:required
    // +k8s:format=k8s-uuid
    ID string `json:"id"`
}
```

### 2. No Handwritten Validation Code
Because `+k8s:declarativeValidationNative` is present, there should be no corresponding handwritten validation function for this specific field (or its properties). Any errors will be directly reported by the declarative validation system.

## Test Coverage

When a field is marked with `+k8s:declarativeValidationNative`, it signals that there is no corresponding handwritten validation. Declarative validation tests should be added to ensure that the validation rules are enforced correctly.

### Example

In a `declarative_validation_test.go` file, add test cases that violate the declarative validation rules. The expected error should be constructed using the `field` package, and because there is no handwritten validation to compare against, the error should be marked with `.MarkDeclarativeNative()`.

Consider the `PodGroup` in the `scheduling.k8s.io` API, where the `name` field is validated using `+k8s:declarativeValidationNative` and `+k8s:format=k8s-short-name`.

**File:** `pkg/registry/scheduling/workload/declarative_validation_test.go`

```go
func testDeclarativeValidate(t *testing.T, apiVersion string) {
    // ...
    testCases := map[string]struct {
        input        scheduling.Workload
        expectedErrs field.ErrorList
    }{
        // ...
        "invalid podGroup name": {
            input: mkValidWorkload(func(obj *scheduling.Workload) {
                obj.Spec.PodGroups[0].Name = "Invalid_Name"
            }),
            expectedErrs: field.ErrorList{
                field.Invalid(field.NewPath("spec", "podGroups").Index(0).Child("name"), "Invalid_Name", "").WithOrigin("format=k8s-short-name").MarkDeclarativeNative(),
            },
        },
        // ...
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
1. An invalid `Workload` is created with a `podGroup` name that contains an underscore, violating the `k8s-short-name` format.
2. The expected error is a `field.Invalid` error, with the origin set to `format=k8s-short-name`.
3. Crucially, `.MarkDeclarativeNative()` is called on the error to signify that it originates purely from declarative validation.
