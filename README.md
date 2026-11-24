# CS 593 Homework 3

This branch (`hw3-fix-2069`) fixes
[pallets/jinja#2069](https://github.com/pallets/jinja/issues/2069) on top of
Jinja 3.1.6. The original Jinja README follows this section.

## The bug

`meta.find_undeclared_variables()` reports a variable as undeclared even when
every branch of an `if` sets it before it is read:

```jinja
{% if a %}{% set output = 1 %}{% else %}{% set output = 2 %}{% endif %}
{{ output }}
```

Expected `{'a'}`, but 3.1.6 returns `{'a', 'output'}`. Rendering is correct; only
the reported names are wrong.

**Root cause:** When the branches of an `if` are merged, `Symbols.branch_update`
marks every name that any branch newly stores to be looked up in the context when
the frame is entered, without checking if all branches store it. That is safe for
rendering, because a branch that stores the name overwrites the value, but
`find_undeclared_variables()` takes each such lookup to mean that the template
needs the name from outside. All `elif` branches are also analyzed into one set
of symbols, so the merge can't tell whether each of them stores the name.

**Fix:** The analysis additionally records which names are stored on every path,
and which names may be read before that. Only `find_undeclared_variables()` uses
this. The generated code was identical to 3.1.6 for every template compared.

| File | Change |
| --- | --- |
| `src/jinja2/idtracking.py` | Record the additional information |
| `src/jinja2/meta.py` | Use it to skip names that can't be read from the context |
| `tests/test_api.py` | Tests only |

## Commits

| Commit | Description |
| --- | --- |
| `15206881` (tag `3.1.6`) | Original Jinja 3.1.6 |
| `d50b53aa` | Reproducing tests |
| `edbfeaf2` | Fix |
| `6ab69343` | Additional tests |

## Setup

The commands were run on Linux (WSL2) with Python 3.13 (uv) and 3.12 (venv).
First, clone only this branch:

```bash
git clone --branch hw3-fix-2069 --single-branch https://github.com/GalacticQuasar/jinja.git
cd jinja
```

Then create the environment in **one** of two ways. With
[uv](https://docs.astral.sh/uv/):

```bash
uv venv --python 3.13 .venv
uv pip install --python .venv/bin/python -r requirements/tests.txt -e .
```

Or with a plain virtual environment, using Python 3.10 or newer:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements/tests.txt -e .
```

Use `uv venv` rather than `uv run` or `uv sync`, which would create a `uv.lock`
that this project does not have.

## Reproduce the original failure

Check out the commit that has the reproducing tests but not the fix:

```bash
git checkout d50b53aa
.venv/bin/python -m pytest --tb=short tests/test_api.py -k "find_undeclared_variables_set_ or set_in_branch_render"
```

Expected: **5 failed, 8 passed** (34 deselected). Each failure has `'output'` as
the only extra item. The 8 tests that pass check behavior that must not change.

Git reports a detached HEAD here, which is expected. Return to the branch:

```bash
git checkout hw3-fix-2069
```

## Run the tests on the final code

Reproducing tests. Expected: **13 passed** (166 deselected).

```bash
.venv/bin/python -m pytest tests/test_api.py -k "find_undeclared_variables_set_ or set_in_branch_render"
```

Additional tests. Expected: **132 passed**.

```bash
.venv/bin/python -m pytest tests/test_api.py::TestMetaSetInBranches
```

Full suite. Expected: **1054 passed**.

```bash
.venv/bin/python -m pytest
```

## View only these changes

Relative to 3.1.6, leaving out this README section:

```bash
git diff 15206881 hw3-fix-2069 -- src tests
git diff --stat 15206881 hw3-fix-2069 -- src tests
git log --oneline 15206881..hw3-fix-2069
```

## Known limitations

- Only reads in the template's own source count. Reads by other templates
  (`include`, `import`, child blocks, `super()`) or by Python functions such as
  those using `@pass_context` do not cause a name to be reported.
- Some names are still reported although no path reads them from the context:
  - An inner scope before the `if` uses the same name for its own variable, such
    as `{% for x in seq %}{{ x }}{% endfor %}`.
  - A macro that reads the name is defined before the `if` but only called after.
  - A branch leaves the loop with `break` or `continue` instead of setting the
    name.
  - Inside a loop, a macro, call block, recursive loop, or loop filter reads the
    name. It could be kept and used in a later iteration.
  - The name is set at the top level and read in a `{% block %}`.
- Out of scope and unchanged from 3.1.6:
  - A name set in only some branches is reported even if it's never read unset.
  - A loop that reads a name before an unconditional `{% set %}`, such as
    `{% for i in seq %}{{ x }}{% endfor %}{% set x = 1 %}`, does not see the
    context value, and the name is not reported.
- The analysis runs for every compilation. On two benchmarks this added about 0-6%
  to compile time, except for one slower run.
- Not checked: Python 3.7, 3.9, and PyPy; rendering generated templates in async
  mode; third-party extensions.

---

# Jinja

Jinja is a fast, expressive, extensible templating engine. Special
placeholders in the template allow writing code similar to Python
syntax. Then the template is passed data to render the final document.

It includes:

-   Template inheritance and inclusion.
-   Define and import macros within templates.
-   HTML templates can use autoescaping to prevent XSS from untrusted
    user input.
-   A sandboxed environment can safely render untrusted templates.
-   AsyncIO support for generating templates and calling async
    functions.
-   I18N support with Babel.
-   Templates are compiled to optimized Python code just-in-time and
    cached, or can be compiled ahead-of-time.
-   Exceptions point to the correct line in templates to make debugging
    easier.
-   Extensible filters, tests, functions, and even syntax.

Jinja's philosophy is that while application logic belongs in Python if
possible, it shouldn't make the template designer's job difficult by
restricting functionality too much.


## In A Nutshell

```jinja
{% extends "base.html" %}
{% block title %}Members{% endblock %}
{% block content %}
  <ul>
  {% for user in users %}
    <li><a href="{{ user.url }}">{{ user.username }}</a></li>
  {% endfor %}
  </ul>
{% endblock %}
```

## Donate

The Pallets organization develops and supports Jinja and other popular
packages. In order to grow the community of contributors and users, and
allow the maintainers to devote more time to the projects, [please
donate today][].

[please donate today]: https://palletsprojects.com/donate

## Contributing

See our [detailed contributing documentation][contrib] for many ways to
contribute, including reporting issues, requesting features, asking or answering
questions, and making PRs.

[contrib]: https://palletsprojects.com/contributing/
