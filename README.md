# Renovate: release notes stay compare-only once a lookup fails

Reproduction for a Renovate discussion about the release-notes package cache. A lookup that finds no release notes is cached as a compare-only entry under the key `<repository>:<version>`, and every later cache hit rewrites that entry with a fresh TTL instead of retrying. On a cache shared between repositories, such as the Mend hosted app, the entry never expires and every PR for that upstream version renders only a `[Compare Source]` link.

## Reproduction

`versions.yaml` pins the `rook-ceph` Helm chart from `https://charts.rook.io/release` twice, at `v1.20.7`, under two dependency names. Both resolve to `github.com/rook/rook` through the chart index, and both update to `v1.20.8`, whose GitHub release has a body.

`renovate.json` treats `rook-ceph-own-key` differently in exactly one way: `sourceDirectory: deploy/charts/rook-ceph`, the directory the chart is actually published from. It holds no changelog file, so Renovate falls back to the GitHub release as usual, but the release-notes cache key becomes `rook/rook:deploy/charts/rook-ceph:v1.20.8` instead of `rook/rook:v1.20.8`.

Expected: both PRs render the release body.

Observed on the Mend hosted app (Renovate 44.112.0):

| Dependency | Cache key | Rendered |
|---|---|---|
| `rook-ceph-shared-key` | `rook/rook:v1.20.8` | compare-only |
| `rook-ceph-own-key` | `rook/rook:deploy/charts/rook-ceph:v1.20.8` | full release body |

Same repository, same dependency, same upstream release, same run. The only variable is the cache key.

Note on datasources: the key also appends the release `gitRef` when the datasource provides one. `github-releases` does, so a pin through that datasource uses `rook/rook:v1.20.8:v1.20.8` and does not share the entry with Helm or Docker consumers of the same release. The Helm datasource is used here to hit the same key as a chart consumer.

## Where it happens

`lib/workers/repository/update/pr/changelog/release-notes.ts`, `addReleaseNotes()`, per version:

1. `packageCache.get(namespace, '<repository>:<version>')`
2. on a miss: `getReleaseNotesMd()`, then `getReleaseNotes()`
3. if both return null: `{ url: v.compare.url, notesSourceUrl: '' }`
4. `packageCache.set(...)`, unconditionally, with `releaseNotesCacheMinutes(v.date)`
