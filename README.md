<a id="x-28PROJECT-DOCS-3A-40README-2040ANTS-DOC-2FLOCATIVES-3ASECTION-29"></a>

# GitHub Action to Setup Common Lisp for CI

This is a Github Action to setup Common Lisp, Roswell and Qlot.

It is useful to call it before running tests or building docs
for your Common Lisp libraries. Action encapsulates all steps
necessary to make available [Roswell][795a]
and [Qlot][e3ea] inside the Github `CI`.

<a id="x-28PROJECT-DOCS-3A-3A-40FEATURES-2040ANTS-DOC-2FLOCATIVES-3ASECTION-29"></a>

## What this action does for you?

* It installs Roswell and all it's dependencies, doing right thing depending on
  the operating system. It should work on Ubuntu, `OSX` and Windows.

* Upgrade `ASDF` to the latest version.

* Installs Qlot.

* Adds to `PATH` these directories: `~/.roswell/bin` and `.qlot/bin`

* Creates `.qlot` by running `qlot install`. How to override content of the
  `qlfile`, see "Overriding qlfile" section.

* And finally, it can install a specified `ASDF` system and all it's dependencies.
  But this step is optional.

<a id="x-28PROJECT-DOCS-3A-3A-40TYPICAL-USAGE-2040ANTS-DOC-2FLOCATIVES-3ASECTION-29"></a>

## A typical usage

Here is how a minimal GitHub Workflow might look like:

```yaml
name: 'CI'

on:
  push:
    branches:
      - 'main'
      - 'master'
  pull_request:

jobs:
  tests:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        lisp:
          - sbcl-bin
          - ccl-bin
          
    env:
      LISP: ${{ matrix.lisp }}

    steps:
      - uses: actions/checkout@v2
      - uses: 40ants/setup-lisp@v2
        with:
          asdf-system: cl-info
      - uses: 40ants/run-tests@v2
        with:
          asdf-system: cl-info
```
The part, corresponding to an action call is:

```yaml
- uses: 40ants/setup-lisp@v2
  with:
    asdf-system: cl-info
```
If you remove `with` part, then action will skip the `ASDF` system
installation.

Also, pay attention to the `env` section of the workflow. If you don't
set up a `LISP` env variable, action will set default lisp implementation
to `sbcl`:

```yaml
env:
  LISP: ${{ matrix.lisp }}
```
The last step in this workflow runs tests for the specified `ASDF`
system. It is documented [here][8469].

<a id="x-28PROJECT-DOCS-3A-3A-40ROSWELL-VERSION-2040ANTS-DOC-2FLOCATIVES-3ASECTION-29"></a>

## Overriding Roswell version

By default this action will install the latest version of Roswell known to be
working with this action. However, should you need to use a different version
instead, you can specify that via the `roswell-version` argument:

```
- uses: 40ants/setup-lisp@v2
  with:
    roswell-version: v21.10.14.111
```
<a id="x-28PROJECT-DOCS-3A-3A-40ASDF-VERSION-2040ANTS-DOC-2FLOCATIVES-3ASECTION-29"></a>

## Overriding ASDF version

By default this action will install the latest version of `ASDF` known to be
working with this action.  However, should you need to use a different version
instead, you can specify that via the `asdf-version` argument:

```
- uses: 40ants/setup-lisp@v2
  with:
    asdf-version: 3.3.5.3
```
<a id="x-28PROJECT-DOCS-3A-3A-40QLOT-VERSION-2040ANTS-DOC-2FLOCATIVES-3ASECTION-29"></a>

## Overriding Qlot version

By default this action will install the latest version of Qlot known to be
working with this action.  However, should you need to use a different version
instead, you can specify that via the `qlot-version` argument:

```
- uses: 40ants/setup-lisp@v2
  with:
    qlot-version: 0.11.5
```
<a id="x-28PROJECT-DOCS-3A-3A-40QL-FILE-2040ANTS-DOC-2FLOCATIVES-3ASECTION-29"></a>

## Overriding qlfile

Sometimes you might want to generate content of qlfile
depending on matrix parameters. For example with matrix like this one:

```yaml
matrix:
  os:
    - ubuntu-latest
    - macos-latest
    - windows-latest
  quicklisp-dist:
    - quicklisp
    - ultralisp
  lisp:
    - sbcl-bin
    - ccl-bin
    - ecl
```
you might want to add an [ultralisp][2a0d] source
to the qlfile. Here is how this can be archived:

```yaml
env:
  LISP: ${{ matrix.lisp }}
  OS: ${{ matrix.os }}
  QUICKLISP_DIST: ${{ matrix.quicklisp-dist }}

steps:
  - uses: actions/checkout@v2
  - uses: 40ants/setup-lisp@v2
    with:
      asdf-system: cl-info
      qlfile-template: |
        {% ifequal quicklisp_dist "ultralisp" %}
        dist ultralisp http://dist.ultralisp.org
        {% endifequal %}
```
Here we see a few important things.

1. We put into the env var the type of the quicklisp distribution we want to
   our library to test against.

2. We pass a multiline argument `qlfile-template` to the action.

3. Template refers to `quicklisp_dist` to conditionally include a line
   into `qlfile` when `quicklisp_dist == "ultralisp"`.

You can refer any environment variable inside the `qlfile` templater.
Also note, it is using [Djula][3dbd]
markup, similar to [Django][04b3]
and [Jinja2][dd23].

<a id="x-28PROJECT-DOCS-3A-3A-40CACHING-2040ANTS-DOC-2FLOCATIVES-3ASECTION-29"></a>

## Caching

Usually installing Roswell, a lisp implementation and dependencies
take from 2 to 10 minutes. Multiply this to the number of
matrix combinations and you'll get signifficant time.

To speed up build, you can use caching using a standad GitHub action `actions/cache@v2`.

To make caching work, add such sections into your workflow file:

```yaml
- name: Grant All Perms to Make Cache Restoring Possible
  run: |
    sudo mkdir -p /usr/local/etc/roswell
    sudo chown "${USER}" /usr/local/etc/roswell
    # Here the ros binary will be restored:
    sudo chown "${USER}" /usr/local/bin
- name: Get Current Month
  id: current-month
  run: |
    echo "::set-output name=value::$(date -u "+%Y-%m")"
- name: Cache Roswell Setup
  id: cache
  uses: actions/cache@v2
  env:
    cache-name: cache-roswell
  with:
    path: |
      /usr/local/bin/ros
      ~/.cache/common-lisp/
      ~/.roswell
      /usr/local/etc/roswell
      .qlot
    key: "${{ steps.current-month.outputs.value }}-${{ env.cache-name }}-${{ runner.os }}-${{ hashFiles('qlfile.lock') }}"
- name: Restore Path To Cached Files
  run: |
    echo $HOME/.roswell/bin >> $GITHUB_PATH
    echo .qlot/bin >> $GITHUB_PATH
  if: steps.cache.outputs.cache-hit == 'true'
- uses: 40ants/setup-lisp@v2
  if: steps.cache.outputs.cache-hit != 'true'
```
There are two important lines here.

* The last line `if: steps.cache.outputs.cache-hit != 'true'` skips
  running lisp installation, it it was take from the cache.

* The `key` value:

`
  key: "${{ steps.current-month.outputs.value }}-${{ env.cache-name }}-${{ runner.os }}-${{ hashFiles('qlfile.lock') }}"
`

It controls when your cache will be matched. If you are using `matrix`, put all it's components
  into the key.

I also added a current month there, to make sure cache will be renewed at least monthly.
  This way a new Roswell, Qlot and `ASDF` will be used in a build.

<a id="x-28PROJECT-DOCS-3A-3A-40ROADMAP-2040ANTS-DOC-2FLOCATIVES-3ASECTION-29"></a>

## Roadmap

* Support [CLPM][2c45].

* Vendor all dependencies, to make action more reliable and secure.

<a id="x-28PROJECT-DOCS-3A-3A-40CONTRIBUTION-2040ANTS-DOC-2FLOCATIVES-3ASECTION-29"></a>

## Contribution

If you want to contribute to this system, join development at GitHub:

[https://github.com/40ants/setup-lisp][cbff]


[8469]: https://40ants.com/run-tests
[04b3]: https://docs.djangoproject.com/en/3.1/topics/templates/
[cbff]: https://github.com/40ants/setup-lisp
[e3ea]: https://github.com/fukamachi/qlot
[3dbd]: https://github.com/mmontone/djula
[795a]: https://github.com/roswell/roswell
[2c45]: https://gitlab.common-lisp.net/clpm/clpm
[dd23]: https://jinja.palletsprojects.com/
[2a0d]: https://ultralisp.org

* * *
###### [generated by [40ANTS-DOC](https://40ants.com/doc/)]
