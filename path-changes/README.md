# path-changes

Report which path groups a push or pull request touched.

## Permissions

No special permissions are required for the usual path-detection use case.

## Inputs

### `paths` (required)

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

### `base-ref`

Default: empty string

Usually left empty. The action auto-detects the correct comparison base from the webhook payload:

- pull request → `event.pull_request.base.sha`
- push → `event.before`
- schedule / workflow_dispatch → all groups are treated as changed

You can override it only when you need a custom comparison target.

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
