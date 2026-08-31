# path-changes

Report which path groups a push or pull request touched.

Groups can be listed explicitly (`paths`), or discovered from the git index
(`marker` or `depth`) so that new projects under a root are picked up without
touching the calling workflow.

## Permissions

No special permissions are required for the usual path-detection use case.

## Inputs

### `paths`

A JSON object that maps group names to glob patterns.

Example:

```yaml
with:
  paths: |
    {
      "app01": ["app01/**", ".github/workflows/build-app01.yml"],
      "app02": ["app02/**"]
    }
```

The action evaluates each path group and writes a boolean result for it.
Leave `paths` empty to use discovery mode instead (`marker` or `depth`).

### `root`

Default: `.`

The directory to search for projects in discovery mode. The root itself is
never a project. Ignored when `paths` is set.

### `marker`

A glob matched against the path relative to a candidate directory; the
directory is a project when at least one tracked file under it matches.

Examples: `pyproject.toml`, `Makefile`, `tests/**`, `Containerfile`.

Untracked files are never considered, so local junk (a `.venv`, build output)
cannot become a project. Mutually exclusive with `depth` and with `paths`.

### `depth`

Depth mode: every tracked file under `root` contributes its first `depth`
slash-separated segments as a project; a shallower file contributes its whole
path. Example: `root: bash`, `depth: 3` yields `bash/bin/<name>` and
`bash/lib/<name>`. Mutually exclusive with `marker` and with `paths`.

### `include`

Default: empty

Space-separated glob patterns appended to every group, so a change to one of
them marks every group changed. Example: `.editorconfig` (the shfmt rules for
all shell scripts).

### `capabilities`

A JSON object mapping a capability name to a list of globs matched against the
path relative to a project directory; the project has the capability when at
least one tracked file matches. Discovery mode only. Capability names must be
usable as GitHub expression property keys (letters, digits, underscore, first
character not a digit).

Example:

```yaml
with:
  capabilities: '{"test": ["tests/**"], "container": ["Containerfile"]}'
```

### `base-ref`

Default: empty string

Usually left empty. The action auto-detects the correct comparison base from
the webhook payload:

- pull request → `event.pull_request.base.sha`
- push → `event.before`
- schedule / workflow_dispatch → all groups are treated as changed

You can override it only when you need a custom comparison target.

The base commit is not part of a default depth-1 checkout, so the action
fetches it on demand (`git fetch origin <sha> --depth=1`). That fetch needs
the checkout's credentials to persist: on a private repository with
`persist-credentials: false` the fetch is anonymous and fails — either leave
the credentials in place or check out with `fetch-depth: 0`.

Example:

```yaml
with:
  base-ref: "HEAD~1"
```

## Outputs

### `results`

A JSON object with one boolean per group, for example:

```json
{"app01": true, "app02": false}
```

You can read a single group either as:

```yaml
steps.path-changes.outputs.app01
```

or:

```yaml
fromJSON(steps.path-changes.outputs.results).app01
```

In discovery mode the groups are named `p0` … `pN`; read the project-level
outputs instead.

### `all`

A JSON array of all discovered project directories, sorted. Discovery mode
only; `[]` when the groups were given explicitly.

```yaml
fromJSON(steps.path-changes.outputs.all)
```

### `changed`

A JSON array of the discovered projects whose files changed — ready for a
matrix:

```yaml
strategy:
  matrix:
    project: ${{ fromJSON(steps.path-changes.outputs.changed) }}
```

An empty array makes GitHub skip the job, so no `if:` guard is needed.

### `changed_dirs`

The changed project directories, space-joined, ready to pass to an input that
takes space-separated paths:

```yaml
with:
  paths: ${{ steps.path-changes.outputs.changed_dirs }}
```

### `capabilities_changed`

A JSON object mapping each declared capability to the JSON array of changed
projects that have it. Every declared capability is present, with an empty
array when no changed project has it:

```yaml
matrix:
  app: ${{ fromJSON(steps.path-changes.outputs.capabilities_changed).container }}
```

## Minimal example

```yaml
steps:
  - name: Checkout repository
    uses: actions/checkout@v7

  - name: Detect path changes
    id: path-changes
    uses: florian-hild/actions/path-changes@v1
    with:
      paths: |
        {
          "app01": ["app01/**", ".github/workflows/build-app01.yml"],
          "app02": ["app02/**"]
        }

  - run: echo "app01 changed? ${{ fromJSON(steps.path-changes.outputs.results).app01 }}"
```

## Discovery examples

Lint every uv project under `python/`, but only the ones this pull request
actually touched:

```yaml
steps:
  - uses: actions/checkout@v7
    with:
      persist-credentials: false

  - id: pc
    uses: florian-hild/actions/path-changes@v2
    with:
      root: python
      marker: pyproject.toml

jobs: # (the lint job)
  lint:
    needs: changes
    strategy:
      fail-fast: false
      matrix:
        project: ${{ fromJSON(needs.changes.outputs.changed) }}
    uses: florian-hild/reusable-workflows/.github/workflows/python-lint.yml@v2
    with:
      working_directory: ${{ matrix.project }}
```

Bash projects identified by directory layout, with a shared config file that
invalidates everything:

```yaml
- id: pc
  uses: florian-hild/actions/path-changes@v2
  with:
    root: bash
    depth: 3
    include: .editorconfig

- uses: florian-hild/reusable-workflows/.github/workflows/shell-lint.yml@v2
  with:
    paths: ${{ steps.pc.outputs.changed_dirs }}
```

A monorepo with heterogeneous apps — discovery once, one matrix per
capability, jobs without the capability skip themselves:

```yaml
- id: pc
  uses: florian-hild/actions/path-changes@v2
  with:
    root: apps
    marker: Makefile
    capabilities: '{"python": ["pyproject.toml"], "test": ["tests/**"], "container": ["Containerfile"]}'

# lint:   matrix over fromJSON(steps.pc.outputs.capabilities_changed).python
# test:   matrix over fromJSON(steps.pc.outputs.capabilities_changed).test
# build:  matrix over fromJSON(steps.pc.outputs.capabilities_changed).container
```

A capability with no changed project yields an empty array, so the job that
consumes it is skipped — a job that depends on a skipped job still runs.
