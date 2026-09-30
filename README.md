# Renovate: release notes stay compare-only once a lookup fails

Reproduction for a Renovate discussion about the release-notes package cache. A lookup that finds no release notes is cached as a compare-only entry under the key `<repository>:<version>`, and every later cache hit rewrites that entry with a fresh TTL instead of retrying. On a cache shared between repositories, such as the Mend hosted app, the entry never expires and every PR for that upstream version renders only a `[Compare Source]` link.

## Reproduction

`versions.yaml` pins `rook/rook` at `v1.20.7` twice, through the `github-releases` datasource, so no container registry is involved. Both entries resolve to the same upstream repository and the same release, `v1.20.8`, whose GitHub release has a body.

The only difference is in `renovate.json`: the second entry has `sourceDirectory: cache-key-escape`, which changes its release-notes cache key from `rook/rook:v1.20.8` to `rook/rook:cache-key-escape:v1.20.8`. The directory does not exist upstream, so Renovate finds no changelog file there and falls back to the GitHub release, exactly as it would without `sourceDirectory`.

Expected: both PRs render the release body.

Observed on the Mend hosted app (Renovate 44.112.0):

| Dependency | Cache key | Rendered |
|---|---|---|
| `rook-shared-cache-key` | `rook/rook:v1.20.8` | compare-only |
| `rook-fresh-cache-key` | `rook/rook:cache-key-escape:v1.20.8` | full release body |

Same repository, same upstream, same version, same run. The only variable is the cache key.

## Where it happens

`lib/workers/repository/update/pr/changelog/release-notes.ts`, `addReleaseNotes()`, per version:

1. `packageCache.get(namespace, '<repository>:<version>')`
2. on a miss: `getReleaseNotesMd()`, then `getReleaseNotes()`
3. if both return null: `{ url: v.compare.url, notesSourceUrl: '' }`
4. `packageCache.set(...)`, unconditionally, with `releaseNotesCacheMinutes(v.date)`
