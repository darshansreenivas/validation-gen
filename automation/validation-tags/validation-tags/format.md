# +k8s:format

## Description
Validates that a string conforms to a specific format.

## Scope
`Field`, `Type`, `ListVal`, `MapKey`, `MapVal`

## Supported Go Types
`string`, `*string` (and any alias of these types)

## Payload
`<format-string>`

## Supported Formats

| Format | Description | Constraints |
| :--- | :--- | :--- |
| `k8s-ip` | IPv4 or IPv6 address. | IPv4: octets may have leading zeros. |
| `k8s-cidr` | IPv4 or IPv6 CIDR. | Valid IP address with a prefix length (e.g., `192.168.1.0/24`). |
| `k8s-uuid` | RFC 4122 UUID. | Canonical string representation. |
| `k8s-label-key` | Kubernetes label key. | max 253 chars, optional DNS prefix + `/` + name segment. |
| `k8s-label-value` | Kubernetes label value. | max 63 chars, alphanumeric + `_`, `-`, `.`. |
| `k8s-short-name` | DNS label. | max 63 chars, alphanumeric + `-`, start/end with alphanumeric. |
| `k8s-long-name` | DNS subdomain. | max 253 chars, dot-separated DNS labels. |
| `k8s-long-name-caseless` | **Deprecated**: Case-insensitive DNS subdomain. | Same as `k8s-long-name` but allows uppercase. (Use `k8s-long-name` instead). |
| `k8s-path-segment-name` | Safe for use as a URL path segment. | Cannot be `.` or `..` or contain `/` or `%`. |
| `k8s-resource-fully-qualified-name` | Qualified resource identifier. | `prefix/name` where prefix is a DNS subdomain and name is a C identifier (max 32 chars, regex `[A-Za-z_][A-Za-z0-9_]*`). |
| `k8s-resource-pool-name` | Hierarchical pool name. | Slash-separated DNS subdomains (max 253 chars). |
| `k8s-extended-resource-name` | Extended resource name. | Domain-prefixed name (no `kubernetes.io`), valid label key when `requests.` prepended. |

## Stability
**Stable**

## Usage
### Field
```go
type NodeSpec struct {
    // +k8s:format=k8s-uuid
    ProviderID string `json:"providerID,omitempty"`
}
```

### Type
```go
// +k8s:format=k8s-uuid
type UUID string

type NodeSpec struct {
    ProviderID UUID `json:"providerID,omitempty"`
}
```

### Map & Slice
To validate items in a map or slice, compose with `+k8s:eachVal` or `+k8s:eachKey`.

```go
type ResourceList struct {
    // Validates that map values are valid extended resource names
    // +k8s:eachVal=+k8s:format=k8s-extended-resource-name
    Resources map[string]string `json:"resources,omitempty"`
}
```

## When to use

### Identifiers & Names
*   **`k8s-short-name`**: Use for simple, local identifiers that must be valid DNS labels (e.g., container names, port names).
*   **`k8s-long-name`**: Use for globally unique identifiers or names that need to be DNS subdomains (e.g., hostnames, driver names like `example.com/driver`).
*   **`k8s-path-segment-name`**: Use for names that will definitely be used as part of a REST URL path (e.g., `metadata.name` for resources).

### Labels & Metadata
*   **`k8s-label-key`**: Use for fields that store keys for labels, annotations, or taints.
*   **`k8s-label-value`**: Use for fields that store values for labels.

### Resources & Hardware
*   **`k8s-extended-resource-name`**: Use for fields referencing compute resources (e.g., `nvidia.com/gpu`).
*   **`k8s-resource-fully-qualified-name`**: Use for specific qualified identifiers often used in device plugins or low-level resource management.
*   **`k8s-resource-pool-name`**: Use for hierarchical names of resource pools.

### System
*   **`k8s-uuid`**: Use for system-assigned unique identifiers (UIDs).
*   **`k8s-ip`**: Use for network address fields.
*   **`k8s-cidr`**: Use for network range fields (e.g., `podCIDR`).

## Migrating from Handwritten Validation

When adding `+k8s:format` to a field that already has handwritten validation, you should not immediately delete the handwritten code. Instead, follow this pattern to ensure compatibility and correct error reporting:

1.  **Add the Tag**: Add the `+k8s:format=...` tag to the struct field.
2.  **Mark Errors**: Update the handwritten validation logic to mark the returned errors as covered by declarative validation. This helps in de-duplicating errors and verifying compliance.

## Detailed Example: Validating a UUID

This example demonstrates how to apply `k8s-uuid` validation to a `ShareID` field.

### 1. Define the Tag in `types.go`
Add the `+k8s:format=k8s-uuid` tag to the field definition in the API struct.

**File:** `staging/src/k8s.io/api/resource/v1beta1/types.go`
```go
type DeviceRequestAllocationResult struct {
    // ...
    // +k8s:optional
    // +k8s:format=k8s-uuid
    ShareID *types.UID `json:"shareID,omitempty" protobuf:"bytes,9,opt,name=shareID"`
}
```

### 2. Update Handwritten Validation
Update the imperative validation logic to match the declarative tag. You must:
1.  Attach the specific origin string to the error using `.WithOrigin("format=<tag-value>")`.
2.  Mark the error call in the main validation function as covered using `.MarkCoveredByDeclarative()`.

**File:** `pkg/apis/resource/validation/validation.go`

```go
// 1. Helper function sets the Origin
func validateUID(uid string, fldPath *field.Path) field.ErrorList {
    var allErrs field.ErrorList
    if !isValidUUID(uid) {
        allErrs = append(allErrs, field.Invalid(fldPath, uid, "uid must be in RFC 4122 normalized form..."))
    }
    // Return errors with the specific origin "format=k8s-uuid"
    return allErrs.WithOrigin("format=k8s-uuid")
}

// 2. Main validation function marks it as covered
func validateDeviceRequestAllocationResult(result resource.DeviceRequestAllocationResult, fldPath *field.Path) field.ErrorList {
    var allErrs field.ErrorList
    // ... other checks ...
    if result.ShareID != nil {
        // Call the helper and mark the result as covered
        allErrs = append(allErrs, validateUID(string(*result.ShareID), fldPath.Child("shareID")).MarkCoveredByDeclarative()...)
    }
    return allErrs
}
```

### 3. Verify with Declarative Validation Tests
Add a test case to `declarative_validation_test.go` (or similar) to ensure the system correctly identifies and reports the declarative validation error. This test verifies that the `format=k8s-uuid` tag is working.

**File:** `pkg/registry/resource/resourceclaim/declarative_validation_test.go`

```go
func TestValidateStatusUpdateForDeclarative(t *testing.T) {
    // ... setup ...
    testCases := map[string]struct {
        update       resource.ResourceClaim
        expectedErrs field.ErrorList
    }{
        "invalid status.Allocation.Devices.Results[].ShareID": {
            old:    mkValidResourceClaim(),
            update: mkResourceClaimWithStatus(tweakStatusDeviceRequestAllocationResultShareID("invalid-uid")),
            expectedErrs: field.ErrorList{
                // Expect the error with the correct Origin
                field.Invalid(
                    field.NewPath("status", "allocation", "devices", "results").Index(0).Child("shareID"), 
                    "invalid-uid", 
                    "" // Detail message may vary or be ignored by fuzzy matcher
                ).WithOrigin("format=k8s-uuid"),
            },
        },
        // ...
    }
    // ... run tests using VerifyUpdateValidationEquivalence ...
}
```

## Test Coverage

When using `+k8s:format`, you should add declarative validation tests to verify that the string field conforms to the specified format.

### Example

Suppose you have a field that must be a valid DNS-1123 subdomain:

**File:** `pkg/apis/example/v1/types.go`
```go
type MyResourceSpec struct {
    // Validates that the field is a DNS-1123 subdomain.
    // +k8s:format=k8s-dns-1123
    Subdomain string `json:"subdomain,omitempty"`
}
```

Your `declarative_validation_test.go` should include test cases for both valid and invalid formats.

**File:** `pkg/apis/example/validation/declarative_validation_test.go` (hypothetical example)
```go
func TestDeclarativeValidateFormat(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyResource
        expectedErrs field.ErrorList
    }{
        "valid subdomain": {
            input: example.MyResource{
                Spec: example.MyResourceSpec{
                    Subdomain: "my-valid-subdomain",
                },
            },
            expectedErrs: field.ErrorList{},
        },
        "invalid subdomain": {
            input: example.MyResource{
                Spec: example.MyResourceSpec{
                    Subdomain: "Invalid_Subdomain!",
                },
            },
            expectedErrs: field.ErrorList{
                field.Invalid(
                    field.NewPath("spec", "subdomain"),
                    "Invalid_Subdomain!",
                    "a DNS-1123 subdomain must consist of lower case alphanumeric characters, '-' or '.', and must start and end with an alphanumeric character (e.g. 'example.com', regex used for validation is '[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*')",
                ),
            },
        },
    }
    // ...
}
```

In this example:
1.  We test a valid DNS-1123 subdomain, which should pass.
2.  We test an invalid subdomain and expect a `field.Invalid` error. The error message provides details about the expected format.
3.  The error does not need to be marked with `.MarkCoveredByDeclarative()` unless you are migrating from handwritten validation that performs the same check. If this is a new validation, it is purely declarative.
