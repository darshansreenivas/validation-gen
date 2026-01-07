# Declarative Validation Native (DV Native)

"Declarative Validation Native" (DV Native) is a mode of operation for the declarative validation framework that allows net-new API fields and their validations to be defined exclusively using Go comment tags, without requiring a parallel hand-written implementation in Go.

This simplifies API development by making declarative tags the single source of truth for validation, reducing boilerplate code, and ensuring consistency between API definitions and enforcement logic.

## The `+k8s:declarativeValidationNative` Tag

To opt-in a field to DV Native mode, use the `+k8s:declarativeValidationNative` tag.

- **Tag**: `+k8s:declarativeValidationNative`
- **Scope**: `fields`
- **Purpose**: Asserts that the field's validation is fully and exclusively handled by the declarative validation framework using stable `+k8s:` validation tags on the same field.

### Declarative Validation Stability Levels
Declarative Validation has the concept of "Stability Levels" for each of the declarative validation tags that are part of the framework.  The Stability Levels include: Alpha, Beta, and Stable.

An up to date list of each tag and their associated Stability Level can be found at the kubernetes.io [Declarative Valdiation Tag Catalog](https://kubernetes.io/docs/reference/using-api/declarative-validation/#catalog) documentation (see the "Stability" column of the table there).

### `stable` Stability Level Enforcement For DV Native Fields
By default, fields marked with `+k8s:declarativeValidationNative` are only allowed to use Stability-Level: **stable** validation tags (e.g., `+k8s:required`, `+k8s:minimum`, `+k8s:maxLength`, `+k8s:format`, `+k8s:enum`).

The `validation-gen` tool will fail code generation if an Alpha or Beta tag is used on a native declarative field, providing compile-time safety. Currently, all validations used with native mode must be stable.

## Generator Behavior

When `validation-gen` encounters a field marked with `+k8s:declarativeValidationNative`, it generates validation code that automatically marks any resulting errors with `.MarkDeclarativeNative()`.

This programmatic marker allows the runtime validation logic to identify these errors as authoritative and always enforce them, regardless of the state of migration-related feature gates.

## Wiring Up Declarative Validation
If the kubernetes API object is not currently plumbed for declarative validation, additional plumbing will need to be done first before adding the DV Native validation. You can check if the plumbing is done by checking if there are any other current DV tags in the types.go file or by looking at the associated doc.go file and strategy.go file for the type to see if Declarative Validation has been plumbed there. See [MIGRATION_GUIDE.md](MIGRATION_GUIDE.md) for more information.

## Strategy Changes

To enable DV Native enforcement in your API strategy, you must pass the `rest.WithDeclarativeNative()` option to the declarative validation call in your `strategy.go`.

```go
func (myStrategy) Validate(ctx context.Context, obj runtime.Object) field.ErrorList {
    myObj := obj.(*myapi.MyResource)
    allErrs := validation.ValidateMyResource(myObj)
    return rest.ValidateDeclarativelyWithMigrationChecks(ctx, legacyscheme.Scheme, obj, nil, allErrs, operation.Create, rest.WithDeclarativeNative())
}
```

## Testing DV Native

When writing tests for DV Native fields, you must ensure that your test assertions account for the `DeclarativeNative` marking on the errors.

### Marking Expected Errors

In your test cases, use `.MarkDeclarativeNative()` on the expected errors for fields that are DV Native.

```go
{
    name: "missing native field",
    obj: &MyResource{...},
    expectedErrs: field.ErrorList{
        field.Required(field.NewPath("spec", "myNativeField"), "").MarkDeclarativeNative(),
    },
}
```

## Walkthrough: New API validation DV Native usage

This walkthrough demonstrates how to use DV Native for a new field, using the `Workload` API as an example.  

<!-- TODO: add link to k/k PR w/ these changes -->

### 1. Update API Types

Add the validation tags and the `+k8s:declarativeValidationNative` marker to your field in `types.go`.

```go
type PodGroup struct {
    // Name is a unique identifier for the PodGroup within the Workload.
    // +required
    // +k8s:required
    // +k8s:format=k8s-short-name
    // +k8s:declarativeValidationNative
    Name string `json:"name" protobuf:"bytes,1,opt,name=name"`
    ...
}
```


### 2. Update the Strategy

Ensure your strategy calls the declarative validation with the `rest.WithDeclarativeNative()` option.

```go
func (workloadStrategy) Validate(ctx context.Context, obj runtime.Object) field.ErrorList {
    workloadScheduling := obj.(*scheduling.Workload)
    allErrs := validation.ValidateWorkload(workloadScheduling)
    return rest.ValidateDeclarativelyWithMigrationChecks(ctx, legacyscheme.Scheme, obj, nil, allErrs, operation.Create, rest.WithDeclarativeNative())
}
```

### 3. Write Associated Tests

When writing tests for your validation logic, any expected errors being matched will need to be marked as native (as they will have this marked).

```go
"no pod group name": {
    obj: mkWorkload(func(w *scheduling.Workload) {
        w.Spec.PodGroups[0].Name = ""
    }),
    expectedErrs: field.ErrorList{
        field.Required(field.NewPath("spec", "podGroups").Index(0).Child("name"), "").MarkDeclarativeNative(),
    },
},
```

### 4. Regenerate and Verify

Run the code generator and your tests:

```bash
hack/update-codegen.sh validation
go test ./staging/src/k8s.io/code-generator/cmd/validation-gen/...
go test ./pkg/registry/<group>/<kind>/...
```
