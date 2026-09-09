# Hiero Gradle cache validation

This fork runs `assemble check` on fresh GitHub-hosted Ubuntu 24.04 runners
with Temurin 17.0.16 and Gradle 9.7.1. It measures native Gradle task-output
caching and separately declared wrapper/module archives through BoringCache.
GitHub OIDC enrollment passed in
[34252584636](https://github.com/boringcache/hiero-gradle-conventions/actions/runs/34252584636).

`.boringcache.toml` owns the cache plan. The validation init script selects
BoringCache for these jobs, with trusted cold/source-change writes and
read-only warm jobs. No local task-cache directory is restored. The upstream
source and its normal CI configuration remain intact.

The current Action is One v1.21.0 at
`90111526eb218a7f1e119ac2b29f765bd4d82734`, with CLI v1.21.0. Its
[read-only release check](https://github.com/boringcache/hiero-gradle-conventions/actions/runs/34315445436)
passed at validation commit `d7dbba4d6ba951ff71b0f4c264ba0d0bedcd5077`.
`assemble check` took 61.97 seconds and the full job took 84 seconds. Both
dependency archives restored. Gradle reported 164 actionable tasks: 109
executed, 49 from cache and six up-to-date. Kotlin compilation, test compilation
and the test task were restored from cache. Those tests did not execute again.

## Original comparison and five source changes

The original comparison used One v1.20.4 and CLI v1.20.5. All six jobs in the
[cold/warm comparison](https://github.com/boringcache/hiero-gradle-conventions/actions/runs/34253302124)
passed:

| Case | Command | Full job | Test behavior |
| --- | ---: | ---: | --- |
| Uncached | 453.80 s | 7m50s | 96 passed, one skipped |
| Native Gradle cold | 548.36 s | 9m24s | 96 passed, one skipped |
| Dependencies cold | 571.43 s | 10m02s | 96 passed, one skipped |
| Native Gradle warm | 95.63 s | 1m57s | Test task restored |
| Combined warm | 59.81 s | 1m36s | Test task restored |
| Dependencies warm | 520.18 s | 8m58s | 96 passed, one skipped |

Combined warm reuse shortened this job sample from 7m50s to 1m36s. Dependency
archives alone did not outperform the uncached sample. Restored test reports
describe prior results; they do not mean tests executed in a warm job. The
One v1.21.0 check above measures compatibility with these saved caches and is
reported separately from the original comparison.

The source sequence starts at `118b0f8e2735d97eb6dd90ee1e333ea61003d1c6`
and applies five actual first-parent changes through captured tip
`8d4c424b1998c6f09e8700495db68bf9fabeff44`. Each run passed 96 tests with one
skipped; the test task executed after each dependency change.

| Change | Original source | Full job | Run |
| --- | --- | ---: | --- |
| JUnit Jupiter 6.1.2 to 6.1.3 | `cb160fa1b5fd22781289524d802c99ee8fc96f07` | 8m28s | [34254683488](https://github.com/boringcache/hiero-gradle-conventions/actions/runs/34254683488) |
| Dependency analysis 3.16.0 to 3.18.0 | `03748df290850c7c20b7611a4086b2b9332e6954` | 8m52s | [34256150545](https://github.com/boringcache/hiero-gradle-conventions/actions/runs/34256150545) |
| Hiero conventions 0.7.10 to 0.7.11 | `4c309297a1d869efc9e81c35c25ed8274ff95ffb` | 9m05s | [34257087392](https://github.com/boringcache/hiero-gradle-conventions/actions/runs/34257087392) |
| Gradlex module dependencies 1.12.2 to 1.13.2 | `4b86b308bec9835a4a0815a05ed0dbd187eb52c7` | 9m39s | [34258096927](https://github.com/boringcache/hiero-gradle-conventions/actions/runs/34258096927) |
| Spotless 8.10.0 to 8.10.1 | `8d4c424b1998c6f09e8700495db68bf9fabeff44` | 10m15s | [34259067592](https://github.com/boringcache/hiero-gradle-conventions/actions/runs/34259067592) |

The final measured five-change head is
`406cf8426496f282af4d65628e9445a02b51e5d7`. Its upstream files and modes match
the captured tip. Later release-pin and documentation commits are distinct
from that measured head. These source-change runs had no paired uncached
control at each revision, so they establish reuse and invalidation without
establishing a speed improvement for every change.

The six retained module versions referenced approximately 462 MB of distinct
chunks versus approximately 2,052 MB summed by version. The wrapper retained
one 274 MB archive; the native Gradle inventory displayed 55 MB. These are
rounded UI observations, not exact physical or billing totals. No eviction
or storage-pressure improvement was measured.

## Coverage and limits

[Issue #559](https://github.com/hiero-ledger/hiero-gradle-conventions/issues/559)
asks for a GCP- or S3-backed remote build cache. This validation exercises the
managed BoringCache backend. It does not implement the named direct-bucket
architecture or establish the migration's storage/authentication requirements.

The baseline disables the upstream remote cache and uses hosted runners. It
does not compare performance with the existing private `hl-gc-gradle-lin`
runner or HTTP cache. Publishing and release workflows were not run. Each
timing is one sample, with runner and test-duration variation. Purchasing
intent and budget remain unverified.
