# +k8s:validation:zeroOrOneOfMember

## Description
The `+k8s:validation:zeroOrOneOfMember` tag is a validation rule that ensures exclusivity among a group of fields within a struct. When applied to multiple fields, it enforces that at most one of them can have a non-zero/non-nil value at any given time. This is useful for implementing unions or choices where only one option can be selected.

## Scope
`Field`

## Supported Go Types
Any Go type, including scalar types (like `int`, `string`), slices, maps, and pointers. The "zero" value is determined by the type's default (e.g., `0` for `int`, `""` for `string`, `nil` for pointers/slices/maps).

## Payload
A unique identifier string. All fields tagged with the same identifier belong to the same exclusivity group.

## Stability
**Alpha**

## Usage

### Field
```go
type Action struct {
    // Only one of the following actions can be specified.
    // +k8s:validation:zeroOrOneOfMember="action"
    Sleep *SleepAction `json:"sleep,omitempty"`
    // +k8s:validation:zeroOrOneOfMember="action"
    Http *HTTPAction `json:"http,omitempty"`
    // +k8s:validation:zeroOrOneOfMember="action"
    Exec *ExecAction `json:"exec,omitempty"`
}
```
In this example, `Sleep`, `Http`, and `Exec` are part of the "action" group. Validation will fail if more than one of these fields is set.

## Migrating from Handwritten Validation

The `+k8s:validation:zeroOrOneOfMember` tag replaces handwritten logic that checks for mutual exclusivity among fields. This is often done by counting how many of the fields are non-nil/non-zero and returning an error if the count is greater than one.

For example, old validation logic might look like this:
```go
func validateAction(action *Action, fldPath *field.Path) field.ErrorList {
    allErrs := field.ErrorList{}
    count := 0
    if action.Sleep != nil { count++ }
    if action.Http != nil { count++ }
    if action.Exec != nil { count++ }
    if count > 1 {
        allErrs = append(allErrs, field.Invalid(fldPath, action, "only one of sleep, http, or exec can be set"))
    }
    return allErrs
}
```
By using the `+k8s:validation:zeroOrOneOfMember` tag, this entire function can be replaced with declarative rules on the struct fields, simplifying the code and making the validation intent clearer.

## Test Coverage

To test the `+k8s:validation:zeroOrOneOfMember` tag, you should create test cases that cover all valid and invalid combinations of the fields in the exclusivity group.

### Example: Exclusive Action Types

**File:** `pkg/apis/example/v1/types.go`
```go
type Action struct {
    // +k8s:validation:zeroOrOneOfMember="action"
    Sleep *SleepAction `json:"sleep,omitempty"`

    // +k8s:validation:zeroOrOneOfMember="action"
    Http *HTTPAction `json:"http,omitempty"`

    // +k8s:validation:zeroOrOneOfMember="action"
    Exec *ExecAction `json:"exec,omitempty"`
}

type SleepAction struct {
    Seconds int `json:"seconds"`
}
type HTTPAction struct {
    URL string `json:"url"`
}
type ExecAction struct {
    Command []string `json:"command"`
}
```

**Test Cases:**
```go
// 1. All fields are nil (Valid)
validObj1 := &example.Action{}
// expected: no validation error

// 2. Only one field is set (Valid)
validObj2 := &example.Action{
    Sleep: &example.SleepAction{Seconds: 10},
}
// expected: no validation error

// 3. Two or more fields are set (Invalid)
invalidObj1 := &example.Action{
    Sleep: &example.SleepAction{Seconds: 10},
    Http:  &example.HTTPAction{URL: "http://example.com"},
}
// expected: field.Invalid(..., "exactly one of [exec, http, sleep] must be set")

// 4. All three fields are set (Invalid)
invalidObj2 := &example.Action{
    Sleep: &example.SleepAction{Seconds: 10},
    Http:  &example.HTTPAction{URL: "http://example.com"},
    Exec:  &example.ExecAction{Command: []string{"/bin/sh"}},
}
// expected: field.Invalid(..., "exactly one of [exec, http, sleep] must be set")
```
These test cases ensure that the `+k8s:validation:zeroOrOneOfMember` rule is correctly enforced, allowing zero or one, but not more than one, of the specified fields to be set.