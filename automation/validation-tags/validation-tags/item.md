# +k8s:item

## Description
Validates a specific item in a `listType=map` list. The item is selected by matching values of the keys.

## Scope
`Field`, `ListVal`

## Supported Go Types
`[]struct{...}` (slices of structs), where the struct has fields corresponding to `listMapKey`.

## Arguments
`key=<value>` (Required): A key-value pair where `key` matches one of the `listMapKey` fields, and `value` is the expected value of that field.

## Payload
`+<validation-tag>`: The validation tag(s) to apply to the matching item.

## Stability
**Stable**

## Usage

### Field
```go
type CertificateSigningRequestStatus struct {
    // ...
    // +k8s:listType=map
    // +k8s:listMapKey=type
    // +k8s:item(type="Approved")=+k8s:zeroOrOneOfMember
    // +k8s:item(type="Denied")=+k8s:zeroOrOneOfMember
    Conditions []CertificateSigningRequestCondition `json:"conditions,omitempty"`
}
```
In this example, specific validation (`+k8s:zeroOrOneOfMember`) is applied only to list items where the `type` field is "Approved" or "Denied".

## Migrating from Handwritten Validation

The `+k8s:item` tag simplifies conditional validation of specific elements within a `listType=map` slice. This replaces handwritten logic that would iterate through the slice, check the value of a key field for each element, and then apply specific validation rules.

## Detailed Example: Conditionally Validating Certificate Conditions

This example demonstrates using `+k8s:item` to apply `+k8s:zeroOrOneOfMember` validation selectively to "Approved" and "Denied" conditions within the `Conditions` slice of `CertificateSigningRequestStatus`.

### 1. Define the Tag in `types.go`
Apply `+k8s:item` tags to the `Conditions` field, specifying which items to target based on their `type` field.

**File:** `staging/src/k8s.io/api/certificates/v1/types.go`
```go
type CertificateSigningRequestStatus {
    // ...
    // +listType=map
    // +listMapKey=type
    // +k8s:item(type="Approved")=+k8s:zeroOrOneOfMember
    // +k8s:item(type="Denied")=+k8s:zeroOrOneOfMember
    Conditions []CertificateSigningRequestCondition `json:"conditions,omitempty" protobuf:"bytes,1,rep,name=conditions"`
}
```

### 2. Update Handwritten Validation
Update the handwritten validation function to remove or mark as covered the explicit conditional checks that `+k8s:item` now handles.

**File:** `pkg/apis/certificates/validation/validation.go`
```go
func validateConditions(fldPath *field.Path, csr *certificates.CertificateSigningRequest, opts certificateValidationOptions) field.ErrorList {
    allErrs := field.ErrorList{}
    // ...

    for i, c := range csr.Status.Conditions {
        // Original handwritten logic (simplified example for zeroOrOneOfMember):
        // if c.Type == certificates.CertificateApproved || c.Type == certificates.CertificateDenied {
        //     // Apply zeroOrOneOfMember validation logic here
        //     // ...
        // }

        // With +k8s:item, the generated code handles this. If still present for backward compatibility:
        // if (c.Type == certificates.CertificateApproved || c.Type == certificates.CertificateDenied) && !opts.allowBothApprovedAndDenied {
        //     // The specific zeroOrOneOfMember error for Approved/Denied conditions would be marked covered.
        //     allErrs = append(allErrs, field.Invalid(fldPath, c.Type, "Approved and Denied conditions are mutually exclusive").WithOrigin("zeroOrOneOf").MarkCoveredByDeclarative())
        // }
    }
    return allErrs
}
```
The `+k8s:item` tag allows for more concise and explicit definitions of validation rules for specific elements within a list, reducing the complexity of handwritten validation code.

## Test Coverage

When using `+k8s:item`, your declarative validation tests should verify that the specified validation rule is correctly applied only to the items that match the given key-value pair.

### Example

Suppose you have a list of conditions where the `message` for the "Ready" type condition must have a maximum length of 10 characters.

**File:** `pkg/apis/example/v1/types.go`
```go
type Condition struct {
    // type of condition in CamelCase.
    Type string `json:"type" protobuf:"bytes,1,name=type"`
    // human-readable message indicating details about last transition.
    Message string `json:"message,omitempty" protobuf:"bytes,4,opt,name=message"`
}

type MyResourceStatus struct {
    // +k8s:listType=map
    // +k8s:listMapKey=type
    // +k8s:item(type=Ready)=+k8s:subfield(message)=+k8s:maxLength=10
    Conditions []Condition `json:"conditions,omitempty"`
}
```

Your `declarative_validation_test.go` should include tests to verify this conditional validation.

**File:** `pkg/apis/example/validation/declarative_validation_test.go` (hypothetical example)
```go
func TestDeclarativeValidateItem(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyResource
        expectedErrs field.ErrorList
    }{
        "valid Ready condition message length": {
            input: example.MyResource{
                Status: example.MyResourceStatus{
                    Conditions: []example.Condition{
                        {Type: "Ready", Message: "short"},
                    },
                },
            },
            expectedErrs: field.ErrorList{},
        },
        "invalid Ready condition message length": {
            input: example.MyResource{
                Status: example.MyResourceStatus{
                    Conditions: []example.Condition{
                        {Type: "Ready", Message: "this message is too long"},
                    },
                },
            },
            expectedErrs: field.ErrorList{
                field.TooLong(
                    field.NewPath("status", "conditions").Key("Ready").Child("message"),
                    "", 10,
                ).WithOrigin("maxLength"),
            },
        },
        "other condition type with long message is valid": {
            input: example.MyResource{
                Status: example.MyResourceStatus{
                    Conditions: []example.Condition{
                        {Type: "Progressing", Message: "this message is allowed to be long"},
                    },
                },
            },
            expectedErrs: field.ErrorList{},
        },
    }
    // ...
}
```

In this example:
1.  We test that a `Ready` condition with a short message is valid.
2.  We test that a `Ready` condition with a message exceeding the `maxLength` of 10 fails validation. Note that for `listType=map`, the path to the item is constructed using `.Key()` with the value of the `listMapKey` field (`type` in this case).
3.  We test that a condition of a different type (`Progressing`) is not affected by the `+k8s:item(type=Ready)` validation and can have a long message.