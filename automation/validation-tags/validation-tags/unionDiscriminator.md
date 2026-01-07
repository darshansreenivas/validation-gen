# +k8s:unionDiscriminator

## Description
Designates the field that determines which union member is active. This tag is used on a field whose value dictates which of the `+k8s:unionMember` fields in the same struct are permitted to be set.

## Scope
`Field`

## Supported Go Types
`string`, `*string` (and any alias of these types) for the discriminator field. The types of the union member fields can be any Go type.

## Arguments
`union=<name>` (optional): Specifies the name of the union if a struct contains multiple unions. This is typically omitted for simple cases.

## Stability
**Stable**

## Usage

### Field
```go
type MyUnionStruct struct {
    // This field acts as the discriminator. Its value determines which of M1 or M2 can be set.
    // +k8s:unionDiscriminator
    D MyDiscriminatorType `json:"d"`

    // These are the union member fields. Only one can be set based on the value of D.
    // +k8s:unionMember
    // +k8s:optional
    M1 *MemberType1 `json:"m1"`

    // +k8s:unionMember
    // +k8s:optional
    M2 *MemberType2 `json:"m2"`
}

type MyDiscriminatorType string

const (
    DiscriminatorM1 MyDiscriminatorType = "M1"
    DiscriminatorM2 MyDiscriminatorType = "M2"
)

type MemberType1 struct { /* ... */ }
type MemberType2 struct { /* ... */ }
```
In this example, if `D` is `M1`, then `M1` can be set and `M2` must be unset. If `D` is `M2`, then `M2` can be set and `M1` must be unset.

## Migrating from Handwritten Validation

The `+k8s:unionDiscriminator` tag, in conjunction with `+k8s:unionMember`, declaratively defines a discriminated union. This replaces complex handwritten validation logic that would typically involve:
1.  Checking the value of a "type" or "kind" field (the discriminator).
2.  Based on the discriminator's value, asserting that only one of a set of other fields (the union members) is set, and others are unset.
3.  Generating appropriate `field.Required`, `field.Forbidden`, or `field.Invalid` errors for violations.

By using these tags, the Kubernetes API machinery automatically handles these checks, making the API definition self-documenting and reducing the boilerplate code in validation functions.

## Detailed Example: Discriminated Union for Resource Source

This example uses a custom struct to illustrate how `+k8s:unionDiscriminator` works with `+k8s:unionMember` to ensure that only one resource source field is specified based on a `SourceType` discriminator.

### 1. Define the Tags in `types.go`
Define a struct with a discriminator field (`SourceType`) and multiple union member fields (`LocalSource`, `RemoteSource`), each marked appropriately.

**File:** `pkg/apis/example/v1/types.go`
```go
type ResourceSource struct {
    // This field determines which of the other source fields is active.
    // +k8s:unionDiscriminator
    SourceType string `json:"sourceType"`

    // These fields are mutually exclusive, controlled by sourceType.
    // +k8s:unionMember
    // +k8s:optional
    LocalSource *LocalResource `json:"local,omitempty"`

    // +k8s:unionMember
    // +k8s:optional
    RemoteSource *RemoteResource `json:"remote,omitempty"`
}

type LocalResource struct {
    Path string `json:"path"`
}

type RemoteResource struct {
    URL string `json:"url"`
}
```

### 2. Define Discriminator Values
Constants matching the discriminator field's type are used to map to the respective union members.

**File:** `pkg/apis/example/v1/types.go`
```go
const (
    SourceTypeLocal  string = "Local"
    SourceTypeRemote string = "Remote"
)
```

### 3. Update Handwritten Validation
If there was prior handwritten validation for this union, it can now be removed or simplified. The generated validation will ensure that:
*   `SourceType` has a valid value (e.g., "Local" or "Remote").
*   If `SourceType` is "Local", `LocalSource` is set and `RemoteSource` is unset.
*   If `SourceType` is "Remote", `RemoteSource` is set and `LocalSource` is unset.

**File:** `pkg/apis/example/validation/validation.go`
```go
func ValidateResourceSource(resourceSource *ResourceSource, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}

    // Original handwritten validation logic (example):
    // switch resourceSource.SourceType {
    // case SourceTypeLocal:
    //     if resourceSource.LocalSource == nil {
    //         allErrs = append(allErrs, field.Required(fldPath.Child("local"), "must be specified for local source type"))
    //     }
    //     if resourceSource.RemoteSource != nil {
    //         allErrs = append(allErrs, field.Forbidden(fldPath.Child("remote"), "must not be specified for local source type"))
    //     }
    // case SourceTypeRemote:
    //     if resourceSource.RemoteSource == nil {
    //         allErrs = append(allErrs, field.Required(fldPath.Child("remote"), "must be specified for remote source type"))
    //     }
    //     if resourceSource.LocalSource != nil {
    //         allErrs = append(allErrs, field.Forbidden(fldPath.Child("local"), "must not be specified for remote source type"))
    //     }
    // default:
    //     allErrs = append(allErrs, field.Invalid(fldPath.Child("sourceType"), resourceSource.SourceType, "unknown source type"))
    // }

    // With +k8s:unionDiscriminator and +k8s:unionMember, this logic is handled automatically.
    // No explicit handwritten validation is needed here for the union's structure.

    // You might still have other validations for the content of LocalSource or RemoteSource if needed.
    if resourceSource.LocalSource != nil {
        // Validate fields within LocalSource, potentially marking errors covered by declarative tags on those sub-fields.
    }
    if resourceSource.RemoteSource != nil {
        // Validate fields within RemoteSource.
    }

    return allErrs
}
```
The `+k8s:unionDiscriminator` and `+k8s:unionMember` tags establish a clear and enforced contract for discriminated unions within the Kubernetes API.

## Test Coverage

When using `+k8s:union` with `+k8s:unionDiscriminator`, your declarative validation tests should verify that:
1.  Exactly one of the union member fields is set.
2.  The discriminator field is set to the correct value corresponding to the chosen member.

### Example

Suppose you have a union type for specifying a protocol, where `type` is the discriminator.

**File:** `pkg/apis/example/v1/types.go`
```go
// +k8s:union
// +k8s:unionDiscriminator=type
type ProtocolConfig struct {
    // +k8s:unionMember
    Type string `json:"type"`

    // +k8s:unionMember(discriminator="TCP")
    TCP *TCPConfig `json:"tcp,omitempty"`

    // +k8s:unionMember(discriminator="UDP")
    UDP *UDPConfig `json:"udp,omitempty"`
}

type TCPConfig struct {
    Timeout int `json:"timeout"`
}
type UDPConfig struct {
    BufferSize int `json:"bufferSize"`
}
```

Your `declarative_validation_test.go` should test the mutual exclusivity and discriminator correctness.

**File:** `pkg/apis/example/validation/declarative_validation_test.go` (hypothetical example)
```go
func TestDeclarativeValidateUnion(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.ProtocolConfig
        expectedErrs field.ErrorList
    }{
        "valid tcp config": {
            input: example.ProtocolConfig{
                Type: "TCP",
                TCP: &example.TCPConfig{Timeout: 100},
            },
            expectedErrs: field.ErrorList{},
        },
        "multiple members set": {
            input: example.ProtocolConfig{
                Type: "TCP",
                TCP:  &example.TCPConfig{Timeout: 100},
                UDP:  &example.UDPConfig{BufferSize: 2048},
            },
            expectedErrs: field.ErrorList{
                field.Invalid(field.NewPath("udp"), "set", "must not be set when tcp is set"),
            },
        },
        "discriminator does not match member": {
            input: example.ProtocolConfig{
                Type: "UDP", // Mismatch
                TCP:  &example.TCPConfig{Timeout: 100},
            },
            expectedErrs: field.ErrorList{
                field.Invalid(field.NewPath("type"), "UDP", "must be 'TCP' when 'tcp' is set"),
            },
        },
        "no member set": {
            input: example.ProtocolConfig{
                Type: "TCP", // Discriminator is set, but member is not
            },
            expectedErrs: field.ErrorList{
                field.Required(field.NewPath("tcp"), "must be set when 'type' is 'TCP'"),
            },
        },
    }
    // ...
}
```

In this example:
1.  We test a valid configuration where the `type` discriminator matches the set member (`TCP`).
2.  We test three invalid cases: multiple members set, a mismatched discriminator, and a missing member for a set discriminator.
3.  The validation system generates appropriate `field.Invalid` and `field.Required` errors to enforce the union rules.