# Semantic version rules and guidelines for 0.x.x releases

We follow [semantic versioning](https://semver.org/), but we also have many
projects at 0.x.x, which means no rules at all in semver. So far the convention
was to keep patch release as bug fix releases (so 0.X.y), and minor releases
could include new features and/or breaking changes.

As we have bigger projects with many dependencies, it is becoming more
challenging to update dependencies when we don't have any guarantees for 0.x.x
versions. As we don't want to prematurely declare a library 1.0.0 (because that
carries some expectations about completeness, not only about breaking changes),
we need to agree on some guarantees for 0.x.x releases.

The goal is to allow developers to move their libraries that are under heavy
development fast, and yet make the life of users of those libraries easier,
without ending in [dependency
hell](https://en.wikipedia.org/wiki/Dependency_hell).

This document covers *when* symbols may be deprecated and removed in 0.x.x
projects. How to actually mark a symbol as deprecated is covered in
[Deprecations](deprecations.md), which applies to every project, including
those past 1.0.0.

In this document we use the words like MUST, MAY, etc. with the meaning defined
by [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Patch versions guarantees

Patch versions MUST provide the same guarantees as minor versions after 1.0.0,
paraphrasing semver on [minor versions](https://semver.org/#spec-item-7), patch
version Z (0.y.Z):

* MUST be incremented if new, backward compatible functionality is introduced
  to the public API.
* MUST be incremented if any public API functionality is marked as deprecated.
* MAY be incremented if substantial new functionality or improvements are
  introduced within the private code.
* MAY include *patch level changes*. *Patch level changes* are [defined by
  semver](https://semver.org/#spec-item-6) as:

  - MUST be incremented if only backward compatible bug fixes are introduced.
  - A bug fix is defined as an internal change that fixes incorrect behavior.

Additionally:

* Deprecated symbols MAY NOT be maintained, it is recommended to switch to
  non-deprecated symbols as soon as possible.

## Minor versions guarantees

Minor versions MUST provide the same guarantees as major versions after 1.0.0,
paraphrasing semver on [major versions](https://semver.org/#spec-item-8), minor
version Y (0.Y.z):

* MUST be incremented if any backward incompatible changes are introduced to
  the public API.
* MAY also include patch level changes.
* The patch version MUST be reset to 0 when minor version is incremented.

Additionally:

* Code changes SHOULD not be required if upgrading from the last patch version
  of the previous minor and no deprecated symbols are used.

## Library Developer Guidelines

When libraries are still under heavy development but have many users, it is
highly recommended prioritizing patch releases. Introducing a breaking (minor)
release is best reserved for situations where it is unfeasible to move the
project forward in a backward-compatible way.

### Patch releases

To be able to do a patch release instead of a minor release (make the release
backwards compatible):

* Avoid removing symbols, deprecate them instead.

* Mark any unmaintained symbol as `deprecated` as soon as possible to encourage
  users to update as soon as possible too.

* Changing behavior:

  - Instead of making a breaking change to the existing symbol, create a new
    one.
  - Copy the old symbol adding a numeric suffix to the name (for example
    `MyClass` -> `MyClass2`, `my_method(int)` -> `my_method2(str)`).
  - Do the breaking changes to the copy.
  - Deprecate the old symbol instead of removing it, giving clear
    instructions on how to replace the deprecated symbol.

#### Tightening an invariant without a hard break

When tightening a validation rule or invariant on existing code:

* Keep currently valid behavior unchanged in patch releases.
* Emit a runtime warning only for usages that will become invalid under the
  stricter rules in a future release.
* Enforce the stricter validation rule as a hard error only in the next minor
  release.
* Where feasible, offer callers an opt-in mechanism (such as an optional
  parameter or flag) to preview or enforce the future behavior early.
* Note that emitting a runtime warning requires a runtime interception point,
  such as a constructor, function call, or property accessor. Where no
  interception point exists (for example, a plain data attribute or mutable
  field), use documented or static deprecation notices instead.

#### Deprecating a single member or attribute

When deprecating individual parts of an existing structure rather than an
entire type or function (see [Deprecations](deprecations.md) for how to mark
each of these):

* **Methods and properties:** Mark them as `@deprecated`.
* **Enum members:** Use `frequenz.core.enum.Enum` to mark members as deprecated.
* **Plain data attributes:** Plain data attributes and public fields typically
  lack runtime interception points. Mark them as deprecated in documentation
  and static type annotations, and defer structural removal or renaming to a
  minor release.

### Minor releases

* Remove all old deprecated symbols.
* Ideally minor releases should not require any code changes if coming from the
  last patch version of the previous minor and no deprecated symbols are used.
  To achieve this:

  - If there is more than one version of a symbol (numeric suffixes were
    added) and the version without a numeric suffix is still present, then
    keep the symbol with the numeric suffix.
 
    For example: Releasing 0.2.0, 0.1.5 is the latest patch and has
    `@deprecated MyClass`, `@deprecated MyClass2`, `MyClass3`. Remove all
    deprecated, keep `MyClass3`.

  - If there is more than one version of a symbol (numeric suffixes were
    added) and the version without a numeric suffix was already removed, then
    rename it to the symbol without the suffix and create a deprecated alias
    for the symbol with the suffix.

    For example: Releasing 0.3.0, 0.2.3 is the latest patch and has
    `@deprecated MyClass3`, `@deprecated MyClass4`, `MyClass5`.

    * Remove all deprecated.
    * Rename `MyClass5` to `MyClass`.
    * Add a deprecated alias for `MyClass5`: `@deprecated MyClass5: TypeAlias
      = MyClass`

  When using this scheme, jumping between consecutive minor versions should
  not require any code changes, as long as no deprecated symbols are used, so
  minor releases are only soft-breaking changes, as a non-breaking path is
  still offered.

## Examples

### Introducing breaking changes to a class or function

If we want to rename an argument for function `foo`:

```python
def foo(wrong_arg: int) -> None: ...
```

We can create a new function `foo2` with the new name and deprecate the old
one:

```python
@deprecated("foo is deprecated, please use `foo2` instead")
def foo(wrong_arg: int) -> None: ...

def foo2(right_arg: str) -> None: ...
```
