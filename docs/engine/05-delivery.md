# 5. Deliver in slices with evidence

| What? | Why? | Is it needed? | Where? | When? |
| --- | --- | --- | --- | --- |
| Behavioral fixtures | Preserve meaning while implementation changes. | Yes; they serve library, HTTP, and storage. | Versioned synthetic test data. | Before the corresponding feature. |
| Milestone gates | Make it clear what can be demonstrated today. | Yes; a roadmap is not implementation evidence. | Pull requests and release notes. | At each handoff. |
| Pilot and operational checks | Discover whether people trust and use the answer. | Required before adoption claims or shared deployment. | Local pilot first. | After correctness and restart tests. |

## Ordered implementation backlog

| Slice | Deliverable and dependencies | Demonstration required to close |
| --- | --- | --- |
| 0 — Agree on the contract | Ratify ID grammar, timestamp encoding and field-length unit; write one accepted and one rejected example for each disputed rule. | A maintainer can predict acceptance without reading a regex or serializer source. Document any break with existing tests. |
| 1 — Domain and reader | Add engine project to the solution; production ID rule, validated immutable snapshot, strict v1 reader. No database prerequisite. | A plain caller loads the payments fixture, rejects a missing endpoint by field path, and cannot mutate the snapshot through its original arrays. |
| 2 — In-memory impact | Incoming index, depth handling, path evidence, cap detection and cancellation. | Canonical, diamond, cycle, isolated, exactly-100 and 101-path fixtures pass; an independent caller uses the engine without launching ASP.NET. |
| 3 — Import semantics | Canonical writer, golden bytes/hash, export, in-process catalog. | Retry unchanged, changed content conflicts, independent snapshots with colliding resource IDs stay isolated. |
| 4 — HTTP | Bind body/query, translate outcomes, report health. | Integration tests show the same answers as direct calls and distinguish invalid, absent, unavailable and timed-out cases. |
| 5 — Persistence | Select and pin a store/driver, document local startup and teardown, add transaction and readiness checks. | Restart preserves map; interrupted import leaves no partial data; simultaneous same-ID requests converge on one immutable snapshot. |
| 6 — Local pilot | Provide a runnable sample, export, operating limits, and release notes. | A maintainer traces a real change, records a useful or missing explanation, and can recover the sample in a clean environment. |

Keep the first PR small: move the ID rule and its tests across the assembly boundary, resolve the grammar deliberately, and verify `dotnet test Causyn.slnx`. Avoid bundling the first schema reader with a driver, UI, or public package release. If time runs short after slice 2, demonstrate the core as a library and label persistence and HTTP as pending.

## Verification strategy

| Risk | Evidence to collect | Why this test belongs here |
| --- | --- | --- |
| Reader hides ambiguous input | Duplicate JSON property, missing boolean, invalid offset, unknown field, null item, duplicate edge, invalid endpoint. Assert stable field location where applicable. | Constructing a typed object alone cannot exercise wire-format ambiguity. |
| Hash drift across versions | Store canonical UTF-8 and expected digest for a fixture; reorder and reformat equivalent input; change one meaningful edge. | Comparing two hashes calculated by the same buggy writer can pass despite a compatibility break. |
| Wrong paths with plausible counts | Assert exact paths **and aligned evidence** for the diamond. For small generated maps compare against an independent oracle that enumerates forward paths from each candidate to the target. | The oracle should not reuse the production incoming-index implementation. |
| Dense queries monopolize work | Exercise maximum allowed counts, depths 1 and 6, disconnected and dense fixtures, already-cancelled and mid-query cancellation. Record runtime and allocations on named hardware. | A 101-row cap measures output, not worst-case expansion or cancellation latency. |
| Storage races | Submit concurrent equal and conflicting imports; inspect final content after restart; simulate failure before commit. | A mock cannot demonstrate actual constraints, rollback, or durability. |
| Misleading service errors | Test transport rejection, missing dataset vs target, unavailable store, elapsed deadline, and request abort. | Users must be able to distinguish retryable infrastructure failure from invalid input and a successful empty answer. |

For performance reports, identify map shape, node/edge counts, depth, retained paths, warmup, number of samples, runtime and store versions. Separate snapshot load time from analysis and HTTP overhead. Report latency distribution rather than the fastest run. Agree on numeric pilot budgets with the first consumer before treating measurements as pass/fail. No performance target is a measured result in this blueprint.

## Release, compatibility, and stop conditions

- **Before merge:** build and test on the declared .NET SDK; inspect contract changes independently of implementation refactors. Run persistent integration checks when touching transactions, query shape or driver version. Publish only synthetic fixtures.
- **Before sharing a package:** consume the packed artifact from a clean .NET project, confirm no required web host or database dependency, document cancellation and error semantics, and settle license and package naming. Project branding does not establish NuGet name availability.
- **Version separately:** package version describes the .NET API; `schemaVersion` describes import semantics; the HTTP representation is a third compatibility surface. Strict readers make even additive JSON fields potentially breaking. Never reinterpret persisted hashes silently after changing the writer.
- **Stop or narrow:** if map quality prevents useful answers, address collection and provenance first. If one real user needs only direct dependents, test the value of indirect paths before expanding graph features. If database pushdown adds no measured benefit, keep the simpler in-memory reference query.
- **Defer until justified:** snapshot diff needs stable cross-snapshot identity; shortest paths need a tie policy; environment-aware edges need deployment semantics; multi-user hosting needs authorization and recovery evidence. Each changes the question CAUSYN can answer and deserves its own fixture before becoming a feature.
