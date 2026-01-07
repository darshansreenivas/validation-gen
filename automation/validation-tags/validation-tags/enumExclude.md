# +k8s:enumExclude

## Description
Excludes a specific constant from the allowed enum values.

## Scope
`Const`

## Supported Go Types
Applies to a `const` value of a type that has been marked with `+k8s:enum`.

## Stability
**Alpha**

## Usage

The `+k8s:enumExclude` tag is applied to a constant declaration.

### Const
```go
// +k8s:enum
type Protocol string

const (
    TCP Protocol = "TCP"
    UDP Protocol = "UDP"

    // +k8s:enumExclude
    InternalProtocol Protocol = "Internal" // This value will ALWAYS be considered invalid
)
```

### Conditional Exclusion
You can conditionally exclude a constant from the enum based on a feature gate or option.

```go
// +k8s:enum
type Protocol string

const (
    TCP Protocol = "TCP"
    UDP Protocol = "UDP"

    // +k8s:ifDisabled(SCTPFeature)=+k8s:enumExclude
    SCTP Protocol = "SCTP" // SCTP is excluded if SCTPFeature is disabled (included if enabled)
    
    // +k8s:ifEnabled(DeprecatedLegacy)=+k8s:enumExclude
    Legacy Protocol = "Legacy" // Legacy is excluded if DeprecatedLegacy feature is enabled
)
```

The exclusion rule will be evaluated at runtime. If multiple conditional tags are used on the same constant, the value is excluded if ANY of the exclude conditions are met.

## Migrating from Handwritten Validation

The `+k8s:enumExclude` tag allows you to easily remove specific values from an enum's set of valid options without altering the underlying type or its handwritten validation logic (if any exists). This is particularly useful when certain enum values are intended for internal use only or become deprecated.

## Detailed Example: Excluding an Internal Enum Value

This example demonstrates how to use `+k8s:enumExclude` to prevent an internal or deprecated constant value from being considered a valid enum option.

### 1. Define the Enum Type and Constant
Define an enum type with `+k8s:enum` and declare a constant value that you wish to exclude. Apply `+k8s:enumExclude` directly to this constant.

**File:** `pkg/apis/example/v1/types.go`
```go
// +k8s:enum
type ConnectionState string

const (
	ConnectionStateConnected    ConnectionState = "Connected"
	ConnectionStateDisconnected ConnectionState = "Disconnected"
	// +k8s:enumExclude
	ConnectionStateInternalOnly ConnectionState = "InternalOnly" // This will be excluded from validation
)
```

### 2. Generated Validation Behavior
When validation code is generated, any field using `ConnectionState` will accept "Connected" and "Disconnected", but will reject "InternalOnly". Previously, you would have needed explicit handwritten checks to filter out "InternalOnly".

## Test Coverage

When using `+k8s:enumExclude`, you should add declarative validation tests to verify that the excluded constant is not accepted as a valid value.

### Example

Given the following enum definition where `ProtocolSCTP` is excluded:

**File:** `pkg/apis/example/v1/types.go`
```go
// +k8s:enum
type Protocol string

const (
    ProtocolTCP  Protocol = "TCP"
    ProtocolUDP  Protocol = "UDP"
    // +k8s:enumExclude
    ProtocolSCTP Protocol = "SCTP" // SCTP is not a supported protocol in this API
)

type MyResource struct {
    Protocol Protocol `json:"protocol"`
}
```

Your `declarative_validation_test.go` should include test cases to confirm that `SCTP` is rejected.

**File:** `pkg/apis/example/validation/declarative_validation_test.go` (hypothetical example)
```go
func TestDeclarativeValidateProtocol(t *testing.T) {
    // ...
    testCases := map[string]struct {
        input        example.MyResource
        expectedErrs field.ErrorList
    }{
        "valid protocol": {
            input: example.MyResource{
                Protocol: example.ProtocolTCP,
            },
            expectedErrs: field.ErrorList{},
        },
        "excluded protocol": {
            input: example.MyResource{
                Protocol: example.ProtocolSCTP,
            },
            expectedErrs: field.ErrorList{
                field.NotSupported(
                    field.NewPath("spec", "protocol"),
                    "SCTP",
                    []string{"TCP", "UDP"},
                ),
            },
        },
    }
    // ...
}
```

In this example:
1.  We test a valid enum value (`ProtocolTCP`) which should pass.
2.  We test the excluded value (`ProtocolSCTP`) and expect a `field.NotSupported` error. The list of supported values in the error message correctly omits `SCTP`.
