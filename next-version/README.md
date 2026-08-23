# next-version

Derive the next semantic version from git tags and commit subjects.

## Required checkout settings

Use a full checkout because the action reads tag history and recent commit subjects:

```yaml
steps:
  - uses: actions/checkout@v7
    with:
      fetch-depth: 0
```

Without full history, the action cannot reliably find the latest release tag and may incorrectly restart from `0.0.1`.

## Permissions

No special repository write permission is needed for this action itself.

## Inputs

### `change_path`

Default: `.`

Restricts the diff check to a specific path. The action looks at files changed under that path and skips bumping when nothing in it changed.

Example:

```yaml
with:
  change_path: "src/"
```

### `tag_pattern`

Default: `v`

Prefix used when discovering the latest tag. A value of `v` matches tags like `v1.2.3`.

Example:

```yaml
with:
  tag_pattern: "release-"
```

### `bump_version`

Default: `false`

If set to `true`, the action always computes a new version even when no matching files changed in `change_path`.

This is useful when you intentionally want a release on a branch even without a code change in the monitored path.

Example:

```yaml
with:
  bump_version: "true"
```

### `version_line`

Default: empty string

Pins releases to a specific major line, for example `2` to keep a maintenance branch on `2.x` instead of jumping to a newer major.

If set, the action only considers tags like `<tag_pattern><major>.*.*` for that line and rejects a `(MAJOR)` bump on that branch.

Example:

```yaml
with:
  version_line: "2"
```

## Outputs

### `version`

The next semantic version, for example `1.4.3`.

### `major`, `minor`, `patch`

Integer components of the computed version.

### `changed`

`true` or `false`, depending on whether the target path changed or whether `bump_version` forced a release.

## Minimal example

```yaml
steps:
  - name: Checkout repository
    uses: actions/checkout@v7
    with:
      fetch-depth: 0

  - name: Get next version
    id: get-version
    uses: florian-hild/actions/next-version@v1

  - run: echo "next version is ${{ steps.get-version.outputs.version }}"
```
