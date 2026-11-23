# 3. Compute an explanation, not just reachability

| What? | Why? | Is it needed? | Where? | When? |
| --- | --- | --- | --- | --- |
| Immutable incoming adjacency index | Answer “who depends on this?” without scanning every edge at every hop. | Yes for the in-memory reference implementation. | Construct once per validated snapshot. | Before the first impact query. |
| Bounded node-simple path enumeration | Preserve multiple explanations without looping or exhausting resources. | Yes; a global visited set loses evidence. | Pure engine with caller cancellation. | For every query, even in tests. |

## Contract at the library boundary

Proposed shape: `AnalyzeImpact(snapshot, targetId, maxDepth, cancellationToken)`. The caller supplies a validated snapshot; the engine owns no filesystem or database connection. Check the target exists separately from checking for incoming edges. Depth counts **edges**, defaults to 4, and must be 1–6 inclusive; reject invalid values rather than clamp them.

Return dataset ID, target ID, effective depth, paths, distinct affected resources, both returned counts, and `resultLimitReached`. A path is expressed **dependent first, target last**, even though discovery proceeds backward. Alongside each adjacent pair return the metadata of that exact edge. Affected resources are the distinct starting nodes of returned paths, sorted by ordinal ID. Path order is not a public guarantee; do not present the first result as the shortest or most important.

## Reference algorithm

1. Build `byId` and `incoming[to]` for this snapshot only. Keep original edge records; reversing lookup direction must not reverse their meaning.
2. Start at the target with a path-local membership set containing that target. For each incoming `u → current`, skip `u` if it already appears **on this path**.
3. Push `u` and its edge. Emit a copy in dependent-to-target order; do not expose a mutable traversal buffer. Every nonempty prefix of the reverse walk is already a valid explanation.
4. Expand from `u` only if below the depth limit. Pop the edge and node before trying the next branch. Check cancellation/deadline during expansion, including branches that emit nothing.
5. Keep at most 100 paths. Only finding a **101st qualifying path** proves the cap was exceeded. Stop then, discard the sentinel, and derive the resource list and counts from the 100 retained paths.

Building the index takes O(V+E) space and time before optional sorting. Traversal work depends on the number of candidate path extensions, not just distinct nodes; a dense graph can generate many paths through the same vertices. The traversal stack needs only the current depth; copied result paths take space proportional to retained paths times depth. A result limit alone is not a deadline.

## Counterexamples to plausible implementations

| Fixture | Expected observation | Mistake it catches |
| --- | --- | --- |
| `a→b`, `a→c`, `b→t`, `c→t`; select `t`, depth 2 | Four paths, three affected resources; `a` has two explanations. | A global visited set or early deduplication suppresses one branch. |
| `a→b`, `b→a`; select `a`, depth 6 | Only `[b,a]`; no path contains the target twice. | Allowing repeated nodes until depth 6 produces fictitious explanations. |
| `a→b→t`; select `t`, depth 1 | `[b,t]` only, with metadata of `b→t`. | Off-by-one depth or using evidence from the wrong edge. |
| Isolated `t`; missing `x` | `t` succeeds with empty arrays; `x` returns missing-target outcome. | Returning an empty report for a misspelled ID. |
| Two datasets reuse `checkout` | Changing one map does not change the other's answer. | An index or cache keyed only by resource ID. |

**Cap fixture within the input limits:** create ten intermediate nodes each pointing to `t`. Add nine upstream nodes each pointing to all ten intermediates. At depth 2 there are 10 direct paths plus 90 indirect paths: exactly 100, cap flag false. Add one new upstream node with a single edge to one intermediate: exactly 101, cap flag true. Do not assert which explanation was dropped. Every returned path must still be valid and counts internally consistent.

**Failure is not truncation.** A deadline or cancellation before proving completion or the sentinel is not a successful report with a false cap flag. Surface a distinct failure/cancellation. Even a successful uncapped report means only “within this map and depth.” If a consumer later needs *all distinct affected resources* irrespective of explanation limits, design a separate reachability query; do not reinterpret this report’s resource count.
