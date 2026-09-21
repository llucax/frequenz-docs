# Deprecations

This document describes how Python projects at Frequenz deprecate things: what
a deprecation promises to users, how to mark a symbol so that the promise is
visible everywhere it can be, what to do when no marker reaches the thing being
deprecated, and what has to accompany every deprecation.

It applies to every project regardless of its version. The [0.x.x versioning
rules](semver-0.x.x.md) say *when* a deprecated symbol may be removed in
projects that are still below 1.0.0; this document says how to deprecate it in
the first place, which is the same work after 1.0.0.

In this document we use the words like MUST, MAY, etc. with the meaning defined
by [RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## A deprecation is never a breaking change

A deprecated symbol keeps working exactly as before. Code that uses it MUST
keep running, and the test suite of a downstream project MUST keep passing,
until the release that actually removes the symbol. The deprecation is a
message; the removal is the change.

Two things that are sometimes proposed break this promise and MUST NOT be done:

* Telling consumers to run with `-W error::DeprecationWarning`. That turns
  every deprecation into a crash at the point of use.
* Withholding a symbol from type checkers, for example by defining it only at
  runtime, so that `mypy` reports `attr-defined`. That turns every deprecation
  into a failing type check.

Frequenz projects configure `pytest` accordingly, in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
filterwarnings = [
    "error",
    "once::DeprecationWarning",
    "once::PendingDeprecationWarning",
]
```

This is the same policy written as test configuration: every other warning is
an error, deprecations are reported once and do not fail the suite.

## Removing a deprecated symbol

When possible the removal point follows downstream adoption.

Before removing a deprecated symbol, check whether downstream projects still
import it or still expose it in a public signature. Remove it once they have
migrated, or keep the compatibility symbol until they have. For clients built
on `frequenz-client-common`, this is the rule already stated in its
[deprecation and compatibility
guide](https://github.com/frequenz-floss/frequenz-client-common-python/blob/v0.x.x/docs/wrapping-guide/deprecation-and-compatibility.md).

When this is not possible, removal after a fixed number of releases or a
specific target release is fine.

In any case, when removing a deprecated symbol, **always** follow [semver
2.0.0](https://semver.org/spec/v2.0.0.html) (remove only in major releases) and
our own [semver-0.x.x.md](semver-0.x.x.md) (remove only in minor
releases) rules.

## Use `warnings.deprecated` / `typing_extensions.deprecated` where it reaches

On a class or a function, the
[`warnings.deprecated`](https://docs.python.org/3/library/warnings.html#warnings.deprecated)
decorator (or
[`typing_extensions.deprecated`](https://typing-extensions.readthedocs.io/en/stable/#typing_extensions.deprecated)
for Python <3.12) gives three things from one message:

* `mypy` reports every use of the symbol.
* The `griffe-warnings-deprecated` extension turns the message into the
  rendered `Deprecated:` admonition in the API documentation.
* The symbol warns at runtime.

```python
@deprecated(
    "fqn.mypkg.OldThing is deprecated since v1.2.0. "
    "Use [fqn.mypkg.NewThing][] instead."
)
class OldThing: ...
```

Five things that are easy to get wrong:

* The message is also the runtime warning text, so keep it to a sentence or
  two, and write it as implicit concatenation of single-line strings. A
  triple-quoted multi-line message keeps its indentation, which stops
  cross-references from resolving and prints an indented warning in the
  terminal.
* Always use the pattern `fqn.mypkg.OldThing is deprecated`, this allows to add
  warning filters that make using deprecations originated for an own library an
  error to make sure the library doesn't ship using deprecated symbols.
* Cross-references work in the message. Use the bare `[some.qualified.Name][]`
  form rather than wrapping the name in backticks: the backticks buy code font
  in the documentation at the cost of more noise in the console.
* Say which version deprecated the symbol, in the sentence itself, as in `"X is
  deprecated since v1.2.0. Use [Y][] instead."`. A separate "since" line cannot
  be expressed through the decorator, so the two would drift apart.
* On a class, the decorator warns on instantiation only, because it wraps
  `__new__`. A type that users receive rather than construct therefore emits no
  runtime warning at all, which is worth knowing when judging whether the
  runtime signal is doing any work for that symbol.

## Write the admonition by hand where the decorator cannot reach

The decorator cannot be applied to a module-level alias, an individual function
argument, an enum member, or a whole module. For those, write a `Deprecated:`
admonition in the docstring:

```python
def connect(*, payload: bytes, raw: bytes | None = None) -> None:
    """Connect to the service.

    Deprecated:
        The `raw` argument is deprecated since v1.2.0. Pass `payload` instead.
    """
```

* Use no custom title. A title replaces the word "Deprecated" in the rendered
  output, so `Deprecated: v1.2.0` renders as just "v1.2.0". The version goes in
  the text.
* Put the admonition immediately after the summary line, which is where the
  `griffe-warnings-deprecated` extension puts the generated ones.
* Never hand-write an admonition for a symbol the decorator already marks, or
  the page shows two.

Anything longer than "use X instead", such as renamed fields or changed
behavior, belongs in the docstring body as ordinary prose, not in a second
admonition.

Enum members are the one case with a dedicated helper:
[`frequenz.core.enum.deprecated_member`](https://frequenz-floss.github.io/frequenz-core-python/v1/reference/frequenz/core/enum/#frequenz.core.enum.deprecated_member)
keeps the old member usable and warns when code reaches it.

## What type checkers cannot do

The type checker can't report an exact alias, a re-exported name, a module and
a constant.

[PEP 702](https://peps.python.org/pep-0702/) covers classes, functions and
overloads only. It explicitly rejected deprecating whole modules, object
attributes and constants, and rejected a `Deprecated[T, message]` type
modifier. Accordingly, `mypy` accepts the decorator on `FuncDef`,
`OverloadedFuncDef` and `TypeInfo`, and nothing else.

A type alias also launders an existing deprecation. Given `Alias: TypeAlias =
DeprecatedClass`, only the alias definition itself is reported; uses of `Alias`
are not reported at all.

So for everything in that category the rendered admonition is the only channel
that reaches the user, which is why writing it is not optional.

## Moving a symbol to another module

When a symbol keeps its identity and only changes location, a module
`__getattr__` can serve the old import path while returning the very same
object, so `isinstance` keeps working through both paths.
[`frequenz.core.warnings.deprecated_aliases()`](https://frequenz-floss.github.io/frequenz-core-python/v1/reference/frequenz/core/warnings/#frequenz.core.warnings.deprecated_aliases)
(available since `frequenz-core` v1.5.0) builds one:

```python
from typing import TYPE_CHECKING, TypeAlias

from frequenz.core.warnings import deprecated_aliases

if TYPE_CHECKING:
    from newpkg.newmod import NewThing as _NewThing

    OldThing: TypeAlias = _NewThing
else:
    __getattr__ = deprecated_aliases(__name__, {"OldThing": "newpkg.newmod:NewThing"})
```

Two things MUST be observed when doing this:

* The `__getattr__` assignment belongs in the `else:` branch of an `if
  TYPE_CHECKING:` block. An unconditional module-level `__getattr__` makes
  `mypy` resolve every unknown name in that module to `Any`, silently, even
  under `--strict`, so a typo in a downstream import stops being an error.
* It aliases names, not modules. A package that moved wholesale needs a real
  `__init__.py` at the old path whose body holds the alias table, because the
  import system never consults a parent package's `__getattr__`.

## When an exact alias is not the right answer

An alias keeps the old and new names interchangeable, but it also removes the
type checker's ability to report anything, as described above. Which side wins
depends on the symbol:

* **Enums:** an enum with members cannot be subclassed, so an alias is the only
  option that keeps values comparable.
* **A type whose shape changed:** a separate deprecated class is correct, since
  the two are not interchangeable anyway.
* **A type users construct:** a real deprecated class is usually worth more
  than an alias, because identity, representation and the static signal matter
  more there than cross-compatibility.

## Every deprecation needs a test and a release note

Every deprecation MUST come with:

* An explicit assertion that the symbol warns, for example with
  [`pytest.deprecated_call()`](https://docs.pytest.org/en/stable/reference/reference.html#pytest.deprecated_call),
  checking the message.
* A migration bullet in the release notes saying what to use instead and what
  differs.

The test matters more than it looks. Because `once::DeprecationWarning` is
configured rather than `error`, a deprecation that silently stops firing does
not fail the suite. Only an explicit assertion catches it.
