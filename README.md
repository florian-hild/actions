# GitHub Actions

| Action | Purpose |
|---|---|
| [`next-version`](next-version/) | Derive the next semantic version from git tags and commit subjects |
| [`release-tags`](release-tags/) | Publish one immutable and two floating tags for a version |
| [`path-changes`](path-changes/) | Report which path groups a push or pull request touched |

## Versioning

One version covers the whole repository.

Every push to `main` that touches an action publishes three tags:

| Tag | Moves | Use it for |
|---|---|---|
| `v1.4.2` | never | exact reproducibility |
| `v1.4` | with every patch | fixes only, no new behaviour |
| `v1` | with every compatible release | the normal choice |

**Pin the major.** You get fixes and new features, never a break.

The size of the bump comes from the commit **subject** lines:

| Marker | Result |
|---|---|
| `(MAJOR)` | `1.4.2` → `2.0.0` |
| `(MINOR)` | `1.4.2` → `1.5.0` |
| neither | `1.4.2` → `1.4.3` |

Subjects only, not bodies: a commit whose body says *"reverts the (MINOR) bump
from #12"* must not move the minor.

A `(MAJOR)` release leaves `v1` where it is, so existing consumers keep working
until they move to `v2` themselves.

Because one version covers the repository, a breaking change in one action
raises the major for all of them. That is the trade-off for a single, readable
`@v1` on the consumer side.
