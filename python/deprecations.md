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
    'error:.*fqn\.mypkg\.[\w\.]+ (is|was) deprecated:DeprecationWarning',
]
```

This is the same policy written as test configuration: every other warning is
an error, deprecations coming from dependencies are reported once and do not
fail the suite.

The last entry is the exception: using a symbol the project deprecated itself
is an error, so the project never releases code that still uses it, which
users would get as warnings they can do nothing about. It doesn't break
downstream projects, since it only matches the project's own package (here
`fqn.mypkg`), and it goes after the `once::` entries because later filters take
precedence. A filter can only tell the project's own deprecations apart by
their message, so this is a heuristic that relies on every message starting
with the fully qualified name of the deprecated symbol (see [the message
rules](#use-warningsdeprecated--typing_extensionsdeprecated-where-it-reaches)).
The [repository configuration
template](https://github.com/frequenz-floss/frequenz-repo-config-python)
generates this entry and its migration script adds it to existing projects.

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
  rendered `Deprecated:` admonition in the API documentation (see [Rendering
  deprecations in the API documentation](#rendering-deprecations-in-the-api-documentation)).
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
* Always start with the pattern `fqn.mypkg.OldThing is deprecated`, with the
  fully qualified name as plain text, neither in backticks nor as a
  cross-reference. The `pytest` filter in [A deprecation is never a breaking
  change](#a-deprecation-is-never-a-breaking-change) relies on it to make the
  project's own uses of the symbol an error, and misses any message that
  doesn't follow it.
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

## Use `frequenz.core` for enum members and moved symbols

The decorator cannot be applied to an enum member or to a module-level alias.
`frequenz-core` has a helper for each, and both warn at runtime:

* [`frequenz.core.enum.deprecated_member()`](https://frequenz-floss.github.io/frequenz-core-python/v1/reference/frequenz/core/enum/#frequenz.core.enum.deprecated_member)
  keeps an enum member usable, on an enum built from
  [`frequenz.core.enum.Enum`](https://frequenz-floss.github.io/frequenz-core-python/v1/reference/frequenz/core/enum/#frequenz.core.enum.Enum),
  and warns when code reaches it by name.
* [`frequenz.core.warnings.deprecated_aliases()`](https://frequenz-floss.github.io/frequenz-core-python/v1/reference/frequenz/core/warnings/#frequenz.core.warnings.deprecated_aliases)
  keeps an old import path working, see [Moving a symbol to another
  module](#moving-a-symbol-to-another-module).

```python
from frequenz.core.enum import Enum, deprecated_member


class TaskStatus(Enum):
    OPEN = 1
    PENDING = deprecated_member(
        1,
        "fqn.mypkg.TaskStatus.PENDING is deprecated since v1.2.0. "
        "Use [fqn.mypkg.TaskStatus.OPEN][] instead.",
    )
```

Their messages follow the same rules as the decorator's. The
`griffe_frequenz_core.deprecations` extension (see [Rendering deprecations in
the API documentation](#rendering-deprecations-in-the-api-documentation))
renders them as the same `Deprecated:` admonition, so don't write one by hand
for these either, unless you need to say more than the message does: the
extension leaves an existing `Deprecated:` admonition alone instead of adding a
second one.

## Write the admonition by hand where nothing else reaches

Neither the decorator nor the `frequenz-core` helpers reach an individual
function argument, a whole module, a constant, or an attribute. For those,
write a `Deprecated:` admonition in the docstring:

```python
def connect(*, payload: bytes, raw: bytes | None = None) -> None:
    """Connect to the service.

    Deprecated:
        The `raw` argument is deprecated since v1.2.0. Pass `payload` instead.
    """
```

* Write `Deprecated:`, not `Warning: Deprecated`. Only the former renders with
  the deprecation style, and it is what the extensions generate.
* Use no custom title. A title replaces the word "Deprecated" in the rendered
  output, so `Deprecated: v1.2.0` renders as just "v1.2.0". The version goes in
  the text.
* Put the admonition immediately after the summary line. The extensions put
  the generated ones at the very top, above the summary, but a docstring has to
  start with its summary, so right after it is as close as a hand-written one
  gets.
* Never hand-write an admonition for a symbol the decorator already marks. The
  `griffe-warnings-deprecated` extension adds its own regardless, so the page
  shows two.

Anything longer than "use X instead", such as renamed fields or changed
behavior, belongs in the docstring body as ordinary prose, not in a second
admonition.

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

So for everything in that category the rendered admonition, and the runtime
warning where there is one, are the only channels that reach the user, which is
why the admonition is not optional.

## Moving a symbol to another module

When a symbol keeps its identity and only changes location, a module
`__getattr__` can serve the old import path while returning the very same
object, so `isinstance` keeps working through both paths.
[`frequenz.core.warnings.deprecated_aliases()`](https://frequenz-floss.github.io/frequenz-core-python/v1/reference/frequenz/core/warnings/#frequenz.core.warnings.deprecated_aliases)
(available since `frequenz-core` v1.5.0) builds one:

```python
from typing import TYPE_CHECKING, TypeAlias

from frequenz.core.warnings import DeprecatedAlias, deprecated_aliases

if TYPE_CHECKING:
    from newpkg.newmod import NewThing as _NewThing

    OldThing: TypeAlias = _NewThing
    """A thing, now called `NewThing`."""
else:
    __getattr__ = deprecated_aliases(
        __name__,
        DeprecatedAlias(
            "OldThing",
            new_module="newpkg.newmod",
            new_name="NewThing",
            since="v1.2.0",
        ),
    )
```

Each alias is a `DeprecatedAlias` naming the old symbol and where it is now:
`new_module`, the module it lives in now, and `new_name`, its new name when it
was renamed too. Without `new_module`, the alias points at a symbol renamed in
its own module. Give `since`, the version the alias is deprecated in, and it
warns with this guide's standard wording, `{old} is deprecated since {since}.
Use {new} instead.`; since every alias gives its own, each can say a different
version. Give `message` instead for a custom template taking only `{old}` and
`{new}`, the fully qualified old and new names, when the standard wording is not
enough. The `griffe_frequenz_core.deprecations` extension renders the same
warning in the documentation of each alias, with `{new}` linked to the target.

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

## Using a deprecated symbol from the library's own code

A library often has to keep using a symbol it deprecated itself, for example
to convert to or from the old type. The user was already warned when they used
the deprecated symbol, so warning them again from the library's internals is
noise. Wrap the call that reaches the deprecated symbol, and only that call, in
[`frequenz.core.warnings.ignoring_deprecations()`](https://frequenz-floss.github.io/frequenz-core-python/v1/reference/frequenz/core/warnings/#frequenz.core.warnings.ignoring_deprecations):

```python
from frequenz.core.warnings import ignoring_deprecations


def from_wire(raw: str) -> OldThing:
    with ignoring_deprecations():
        return OldThing(raw)
```

Don't use `warnings.catch_warnings()` for this. Entering and leaving it resets
the warnings deduplication history of the whole program, so every warning
already shown is shown again, on every call
([python/cpython#73858](https://github.com/python/cpython/issues/73858)).
[`ignoring_warnings()`](https://frequenz-floss.github.io/frequenz-core-python/v1/reference/frequenz/core/warnings/#frequenz.core.warnings.ignoring_warnings)
does the same for other warning categories.

Both only add a filter while the block runs, and that filter is normally shared
by every thread and asyncio task, so keep the block short and don't hold it
across an `await`.

## Every deprecation needs a test and a release note

Every deprecation MUST come with:

* An explicit assertion that the symbol warns, for example with
  [`pytest.deprecated_call()`](https://docs.pytest.org/en/stable/reference/reference.html#pytest.deprecated_call),
  checking the message.
* A migration bullet in the release notes saying what to use instead and what
  differs.

The test matters more than it looks. A deprecation that silently stops firing
fails nothing, whatever the warning filters say, since there is no warning left
to turn into an error. Only an explicit assertion catches it.

`pytest.deprecated_call()` also records the warning instead of letting the
filter for the project's own deprecations turn it into an error, so any test
that uses one of the project's deprecated symbols on purpose needs it. Where
checking the warning doesn't make sense, mark the test with
`@pytest.mark.filterwarnings("once::DeprecationWarning")` instead.

That filter also catches the opposite mistake, a replacement that still goes
through the deprecated symbol internally, but only for messages it matches, and
a deprecation coming from a dependency is still only reported once. To test
that the replacement doesn't warn at all, wrap it in
[`frequenz.core.warnings.asserting_no_deprecations()`](https://frequenz-floss.github.io/frequenz-core-python/v1/reference/frequenz/core/warnings/#frequenz.core.warnings.asserting_no_deprecations),
or
[`asserting_no_warnings()`](https://frequenz-floss.github.io/frequenz-core-python/v1/reference/frequenz/core/warnings/#frequenz.core.warnings.asserting_no_warnings)
for any category:

```python
from frequenz.core.warnings import asserting_no_deprecations


def test_new_thing_does_not_warn() -> None:
    with asserting_no_deprecations():
        NewThing("raw")
```

It fails with an `AssertionError` listing each deprecation raised inside the
block. It resets the deduplication history too, so keep it to tests.

## Rendering deprecations in the API documentation

The `Deprecated:` admonitions are rendered by two griffe extensions, enabled in
the mkdocstrings Python handler in `mkdocs.yml`:

* [`griffe-warnings-deprecated`](https://mkdocstrings.github.io/griffe-warnings-deprecated/)
  for the `deprecated` decorator.
* [`griffe-frequenz-core`](https://github.com/frequenz-floss/griffe-frequenz-core)
  (`griffe_frequenz_core.deprecations`, v1.0.0 or later) for
  `deprecated_aliases()` and `deprecated_member()`.

```yaml
plugins:
  - mkdocstrings:
      handlers:
        python:
          options:
            extensions:
              - griffe_warnings_deprecated:
                  kind: deprecated
                  title: Deprecated
              - griffe_frequenz_core.deprecations:
                  kind: deprecated
                  title: Deprecated
```

Both packages go in the `dev-mkdocs` dependencies. The `deprecated` kind is
styled in `docs/_css/mkdocstrings.css`, which, like the first extension, comes
with the [repository configuration
template](https://github.com/frequenz-floss/frequenz-repo-config-python); a
hand-written `Deprecated:` admonition uses the same kind, so all of them look
alike.
