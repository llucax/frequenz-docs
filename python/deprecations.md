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
with the fully qualified name of the deprecated symbol (see [the rules for
warnings](#the-warning-and-the-notice)).
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

## The warning and the notice

Every deprecation reaches users through two texts, written for different
readers, so they follow different rules:

* The **warning**, emitted at runtime and, for decorated symbols, also reported
  by `mypy`. It is read in a terminal or a log, far from the deprecated symbol:
  it points at the line that used it, not at the symbol itself.
* The **notice**, the `Deprecated:` admonition in the API documentation. It is
  read right where the deprecated symbol is documented.

The warning MUST:

* Start with the pattern `fqn.mypkg.OldThing is deprecated`, with the fully
  qualified name as plain text. The `pytest` filter in [A deprecation is never
  a breaking change](#a-deprecation-is-never-a-breaking-change) relies on it to
  make the project's own uses of the symbol an error, and misses any message
  that doesn't follow it.
* Say what to use instead, by its fully qualified name too, as in
  `fqn.mypkg.OldThing is deprecated. Use fqn.mypkg.NewThing instead.`
* Be plain text, without backticks or cross-references, which are only noise in
  a terminal.
* Be a sentence or two, written as implicit concatenation of single-line
  strings. A triple-quoted multi-line message keeps its indentation and prints
  an indented warning.

It SHOULD NOT say since which version the symbol is deprecated. What matters
to whoever runs the code is that it is deprecated now; the version belongs in
the notice, and leaving it out keeps the warning short.

The notice MUST:

* Say since which version the symbol is deprecated and what to use instead,
  linking the replacement in code font, as in ``Deprecated since v1.2.0. Use
  [`NewThing`][fqn.mypkg.NewThing] instead.``
* Not repeat the name of the deprecated symbol, which is right above it.

For the link text, use the fully qualified name when the replacement lives in
another module, so readers see where it went, as in
``[`fqn.newpkg.NewThing`][]``, and just its name when the context makes it
obvious, such as another member of the same enum, as in
``[`OPEN`][fqn.mypkg.TaskStatus.OPEN]``.

Anything longer than "use X instead", such as renamed fields or changed
behavior, belongs in the docstring body as ordinary prose, not in the notice.

When many symbols share the same long explanation, such as every member of a
deprecated enum, explain it once, in the notice of the symbol they belong to,
and have the others link it: ``Deprecated since v1.2.0. See
[`OldEnum`][fqn.mypkg.OldEnum] for what to use instead.`` A link is easy to
follow in the documentation, but not in a terminal, so each warning still
explains the replacement in full.

The tools below generate both texts wherever they have what they need, which
[What the documentation tooling does](#what-the-documentation-tooling-does)
sums up; everywhere else the notice is [written by
hand](#writing-the-notice-by-hand).

## Use `warnings.deprecated` / `typing_extensions.deprecated` where it reaches

On a class or a function, the
[`warnings.deprecated`](https://docs.python.org/3/library/warnings.html#warnings.deprecated)
decorator (or
[`typing_extensions.deprecated`](https://typing-extensions.readthedocs.io/en/stable/#typing_extensions.deprecated)
for Python <3.12) gives three things:

* `mypy` reports every use of the symbol.
* The symbol warns at runtime.
* The symbol is marked as deprecated in the API documentation.

The decorator only takes the warning, so the notice is written by hand in the
docstring:

```python
@deprecated("fqn.mypkg.OldThing is deprecated. Use fqn.mypkg.NewThing instead.")
class OldThing:
    """A thing, now called `NewThing`.

    Deprecated:
        Deprecated since v1.2.0. Use [`NewThing`][fqn.mypkg.NewThing] instead.
    """
```

On a class, the decorator warns on instantiation only, because it wraps
`__new__`. A type that users receive rather than construct therefore emits no
runtime warning at all, which is worth knowing when judging whether the runtime
signal is doing any work for that symbol.

## Use `frequenz.core` for enum members and moved symbols

The decorator cannot be applied to an enum member or to a module-level alias.
`frequenz-core` has a helper for each, both warn at runtime, and both take the
deprecation as structured arguments, so the warning and the notice are both
generated from them:

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
    PENDING = deprecated_member(1, new_name="OPEN", since="v1.2.0")
```

`new_name` is the member to use instead, in the same enum, and `since` the
version it is deprecated in. Reaching `TaskStatus.PENDING` warns
`fqn.mypkg.TaskStatus.PENDING is deprecated. Use fqn.mypkg.TaskStatus.OPEN
instead.`, and its notice says ``Deprecated since v1.2.0. Use
[`OPEN`][fqn.mypkg.TaskStatus.OPEN] instead.``

When there is no member to point at, or the standard wording is not enough,
pass a `message`, written by the rules for warnings above. It replaces the
warning only, so write the notice by hand too:

```python
class TaskStatus(Enum):
    UNSPECIFIED = deprecated_member(
        0,
        "fqn.mypkg.TaskStatus.UNSPECIFIED is deprecated. "
        "Use the integer 0 instead if you need that low-level value.",
        since="v1.2.0",
    )
    """The status is unspecified.

    Deprecated:
        Deprecated since v1.2.0. Use the integer `0` instead if you need that
        low-level value.
    """
```

The documentation is built without running the code, so write these arguments
as string literals right in the call. What can't be read that way is documented
as well as possible, with a warning, see [What the documentation tooling
does](#what-the-documentation-tooling-does).

## Writing the notice by hand

Write a `Deprecated:` admonition in the docstring:

* For what nothing marks: an individual function argument, a whole module, a
  constant, or an attribute.
* For a symbol deprecated with the decorator, which only carries the warning.
* To say more than a generated notice does, for example for an alias or enum
  member with its own `message`. A hand-written notice always replaces the
  generated one, and it is the only way to change what the documentation says:
  `message` only changes the warning.

```python
def connect(*, payload: bytes, raw: bytes | None = None) -> None:
    """Connect to the service.

    Deprecated:
        The `raw` argument is deprecated since v1.2.0. Use `payload` instead.
    """
```

* Follow the rules for notices in [The warning and the
  notice](#the-warning-and-the-notice). A notice about part of a symbol, such
  as one argument, has to say which part.
* Write `Deprecated:`, not `Warning: Deprecated`. Only the former renders with
  the deprecation style, and it is what the tooling generates.
* Use no custom title. A title replaces the word "Deprecated" in the rendered
  output, so `Deprecated: v1.2.0` renders as just "v1.2.0". The version goes in
  the text.
* Put it immediately after the summary line, since a docstring has to start
  with its summary. On a symbol the tooling marks as deprecated, a decorated
  class or function, an enum member or an alias, the notice is moved to the
  top, where generated ones go, above the summary. Anywhere else, such as on a
  module or for an argument, it stays where it is written.
* Write one notice per symbol. Anything longer than "use X instead" goes in the
  docstring body as ordinary prose.

## What the documentation tooling does

| Deprecated with                        | Warning                       | Notice generated from      | Write the notice by hand                |
| -------------------------------------- | ----------------------------- | -------------------------- | --------------------------------------- |
| `@deprecated(...)`                     | the decorator's message       | nothing                    | always                                  |
| `deprecated_member(..., new_name=...)` | generated, or `message`       | `since` and `new_name`     | to say more, or with a custom `message` |
| `deprecated_member(..., message)`      | `message`                     | nothing                    | always                                  |
| `DeprecatedAlias(...)`                 | generated, or `message`       | `since` and the new path   | to say more, or with a custom `message` |
| nothing (module, constant, argument)   | whatever the code emits       | nothing                    | always                                  |

A deprecated symbol the tooling marks always gets a notice, even when it can't
be written as this guide asks: if there is no hand-written notice and nothing to
generate one from, the documentation shows the warning, or as much as could be
read, such as ``Deprecated. Use [`NewThing`][fqn.mypkg.NewThing] instead.``
when `since` is missing. Each such case logs a warning, which fails the strict
build, so it doesn't go unnoticed; the fix is to give the missing arguments as
string literals, or to write the notice by hand.

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
its own module. `since` is the version the alias is deprecated in; since every
alias gives its own, each can say a different version.

From those, the alias warns with the standard wording, `{old} is deprecated.
Use {new} instead.`, with fully qualified names, and its notice says
``Deprecated since {since}. Use [`{new}`][] instead.``, showing just the new
name for a rename within the same module.

When the standard warning is not enough, give `message` too, a template taking
only `{old}` and `{new}`, the fully qualified old and new names, and written by
the rules for warnings. It replaces the warning only: to say more in the
documentation, write the notice by hand in the docstring of the alias declared
for type checkers, which replaces the generated one.

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

The notices are rendered by the
[`griffe-frequenz-core`](https://github.com/frequenz-floss/griffe-frequenz-core)
extension (`griffe_frequenz_core.deprecations`), enabled in the mkdocstrings
Python handler in `mkdocs.yml`. It handles the `deprecated` decorator,
`deprecated_member()` and `deprecated_aliases()` alike, as [What the
documentation tooling does](#what-the-documentation-tooling-does) describes:

```yaml
plugins:
  - mkdocstrings:
      handlers:
        python:
          options:
            extensions:
              - griffe_frequenz_core.deprecations:
                  kind: deprecated
                  title: Deprecated
```

The package goes in the `dev-mkdocs` dependencies. The `deprecated` kind is
styled in `docs/_css/mkdocstrings.css`, and both come with the [repository
configuration
template](https://github.com/frequenz-floss/frequenz-repo-config-python); a
hand-written `Deprecated:` admonition uses the same kind, so all of them look
alike.

Frequenz projects build their documentation in strict mode, so warnings fail
the build. `griffe-frequenz-core` warns whenever it can't document a
deprecation as this guide asks, and a link to a replacement that can't be
resolved warns too. For a symbol that moved to another project, add that
project's `objects.inv` to the `inventories` of the Python handler, so the link
resolves.
