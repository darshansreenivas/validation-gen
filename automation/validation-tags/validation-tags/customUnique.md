# +k8s:customUnique

## Description
Disables generated uniqueness validation, deferring to custom handwritten validation logic.

## Scope
`Field`, `Type`

## Supported Go Types
`[]any`, `*[]any` (and any alias of these types)

## Stability
**Alpha**

## Usage

The `+k8s:customUnique` tag is applied to a field of type slice, and must be used in conjunction with `+k8s:listType`.

### Field
```go
type CertificateSigningRequestStatus struct {
    // +k8s:listType=map
    // +k8s:listMapKey=type
    // +k8s:customUnique
    Conditions []CertificateSigningRequestCondition `json:"conditions,omitempty"`
}
```

### Type
```go
// +k8s:listType=set
// +k8s:customUnique
type MyCustomUniqueList []string

type MyStruct struct {
    List MyCustomUniqueList `json:"list,omitempty"`
}
```

## Migrating from Handwritten Validation

The `+k8s:customUnique` tag explicitly disables generated uniqueness validation, signaling that custom handwritten logic will handle uniqueness checks. When migrating, ensure your handwritten validation correctly implements the desired uniqueness rules and, if necessary, marks errors as covered using `.MarkCoveredByDeclarative()`.

## Detailed Example: Custom Unique Conditions

This example demonstrates how `+k8s:customUnique` is used on a slice field, where uniqueness is handled by custom validation logic.

### 1. Define the Tag in `types.go`
Add the `+k8s:customUnique` tag to the `Conditions` field in the `CertificateSigningRequestStatus` struct definition, along with `+k8s:listType=map` and `+k8s:listMapKey=type`.

**File:** `staging/src/k8s.io/api/certificates/v1/types.go`
```go
type CertificateSigningRequestStatus struct {
    // ...
    // +k8s:listType=map
    // +k8s:listMapKey=type
    // +k8s:customUnique
    Conditions []CertificateSigningRequestCondition `json:"conditions,omitempty" protobuf:"bytes,1,rep,name=conditions"`
}
```

### 2. Implement Handwritten Uniqueness Validation
Implement or update the handwritten validation function to check for unique condition types. This function will be responsible for reporting duplicate errors.

**File:** `pkg/apis/certificates/validation/validation.go`

```go
func validateConditions(fldPath *field.Path, csr *certificates.CertificateSigningRequest, opts certificateValidationOptions) field.ErrorList {
    allErrs := field.ErrorList{}
    seenTypes := map[certificates.RequestConditionType]bool{}

    for i, c := range csr.Status.Conditions {
        // ... other validations for condition fields ...

        // This custom logic checks for duplicate condition types
        // The +k8s:customUnique tag ensures that the code generator
        // does not add its own uniqueness check for this field.
        if !opts.allowDuplicateConditionTypes { // This option is for backward compatibility
            if seenTypes[c.Type] {
                allErrs = append(allErrs, field.Duplicate(fldPath.Index(i).Child("type"), c.Type))
            }
            seenTypes[c.Type] = true
        }
    }
    return allErrs
}
```
In this example, the `+k8s:customUnique` tag ensures that the generated code skips uniqueness validation for the `Conditions` field, allowing the `validateConditions` function to manage it entirely.

## Test Coverage

Since `+k8s:customUnique` disables generated uniqueness validation in favor of handwritten logic, declarative validation tests should not expect uniqueness errors with an origin like `unique` or `duplicate`. Instead, the responsibility for testing uniqueness lies with the tests for the handwritten validation logic.

However, it is still important to have declarative validation tests for the field to cover any other validation tags. For example, if a field with `+k8s:customUnique` also uses `+k8s:item`, the declarative validation tests should cover the `+k8s:item` validation.

### Example

In the case of `CertificateSigningRequestStatus.Conditions`, which uses `+k8s:customUnique`, the declarative validation tests in `pkg/registry/certificates/certificates/declarative_validation_test.go` focus on the `+k8s:zeroOrOneOfMember` validation applied with `+k8s:item`, not on uniqueness.

A test case might look like this:

```go
testCases := map[string]struct {
    input        api.CertificateSigningRequest
    expectedErrs field.ErrorList
}{
    "status.conditions: Approved+Denied = invalid": {
        input: makeValidCSR(withApprovedCondition(), withDeniedCondition()),
        expectedErrs: field.ErrorList{
            field.Invalid(field.NewPath("status", "conditions"), nil, "").WithOrigin("zeroOrOneOf"),
        },
    },
}
```

This test verifies the `zeroOrOneOfMember` constraint, while the uniqueness of conditions is tested separately in the handwritten validation tests. Additionally, a "ratcheting" test case can be added to ensure that existing objects with duplicate items are not invalidated on update.

```go
"ratcheting: allow existing duplicate types - valid": {
    old:          makeValidCSR(withApprovedCondition(), withApprovedCondition(), withDeniedCondition(), withDeniedCondition()),
    update:       makeValidCSR(withDeniedCondition(), withDeniedCondition(), withApprovedCondition(), withApprovedCondition()),
    subresources: []string{"/status"},
},
```
