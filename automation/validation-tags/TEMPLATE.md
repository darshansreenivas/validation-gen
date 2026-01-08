# +k8s:tagName

## Description
Short description of what the validation tag does.

## Scope
`Field`, `Type` (Specify valid scopes)

## Supported Go Types
List of supported Go types (e.g., `string`, `int`, `[]string`, `map[string]string`).

## Stability
**Alpha** | **Beta** | **Stable**

## Usage

### Field
```go
type MyStruct struct {
    // Description of usage
    // +k8s:tagName
    Field string `json:"field"`
}
```

## Migrating from Handwritten Validation

Explain how to replace existing handwritten validation code with this declarative tag. Provide context on what logic this tag replaces.

## Detailed Example: <Scenario Name>

Provide a realistic, detailed example of how to use this tag, preferably from a real or realistic API.

### 1. Define the Tag in `types.go`
Show the struct definition with the tag applied.

```go
type MyResourceSpec struct {
    // +k8s:tagName
    Field string `json:"field"`
}
```

### 2. Update Handwritten Validation
Show how to update the validation function (e.g., removing old checks or marking them as covered).

```go
func ValidateMyResourceSpec(spec *MyResourceSpec, fldPath *field.Path) field.ErrorList {
    // ...
}
```

## Test Coverage

Explain how to write declarative validation tests for this tag.

### Example

Provide a test case example.

```go
func TestDeclarativeValidateTagName(t *testing.T) {
    // ...
}
```
