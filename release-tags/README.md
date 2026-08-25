# release-tags

Publish the immutable tag for a semantic version, and with `floating_tags` the
two floating major/minor tags next to it.

## Required checkout settings

Use a full checkout because the action needs the tag refs to exist locally and must push new tags back to the repo:

```yaml
steps:
  - uses: actions/checkout@v7
    with:
      fetch-depth: 0
```

## Required permissions

This action writes and pushes git tags, so it needs repository write access:

```yaml
permissions:
  contents: write # required: release-tags pushes/moves git tags in the repo
```

Without this permission, the tag push fails.

## Inputs

### `version` (required)

The semantic version to publish, for example `1.4.2`.

```yaml
with:
  version: "1.4.2"
```

### `tag_prefix`

Default: `v`

Prefix added before the version. `v` yields tags like `v1.4.2`, `v1.4`, and `v1`.

Example:

```yaml
with:
  tag_prefix: "my-action-"
```

This would create tags like `my-action-1.4.2`, `my-action-1.4`, and `my-action-1`.

### `floating_tags`

Default: `false`

Publish the floating `<prefix><major>` and `<prefix><major>.<minor>` tags next
to the immutable full one:

```yaml
with:
  floating_tags: "true" # v1.4.2, v1.4 and v1
```

Off by default, because most releases are deployed by exact version and a
repository releasing several components would otherwise carry two moving tags
per component. Turn it on where consumers pin a moving line - the way the
actions in this repository are consumed as `@v1`.

While off, the `minor` and `major` outputs are empty, so nothing can pin a tag
that was never published. `next-version` is unaffected either way: it looks
for tags with three dot-separated parts and therefore never reads the floating
ones.

### `dry_run`

Default: `false`

If `true`, the action prints what it would publish but does not create or push any tags.

Example:

```yaml
with:
  dry_run: "true"
```

## Outputs

### `full`

The immutable full tag, for example `v1.4.2`.

### `minor`

The floating minor tag, for example `v1.4`. Empty when `floating_tags` is off.

### `major`

The floating major tag, for example `v1`. Empty when `floating_tags` is off.

## Minimal example

```yaml
permissions:
  contents: write

steps:
  - name: Checkout repository
    uses: actions/checkout@v7
    with:
      fetch-depth: 0

  - name: Get next version
    id: get-version
    uses: florian-hild/actions/next-version@v1
    with:
      bump_version: "true"

  - name: Release tags
    uses: florian-hild/actions/release-tags@v1
    with:
      version: ${{ steps.get-version.outputs.version }}
```
