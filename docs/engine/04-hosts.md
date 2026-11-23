# 4. Keep the engine portable; make storage replaceable

| What? | Why? | Is it needed? | Where? | When? |
| --- | --- | --- | --- | --- |
| .NET class library | A console test and an API should ask the same question and receive the same answer. | Yes; this is the shift from app to engine. | Proposed `Causyn.Engine`, referenced by hosts. | First vertical slice. |
| HTTP adapter | Other processes need a stable request and error contract. | Useful second host, unnecessary for proving traversal. | Existing `Causyn.Api`. | After pure fixtures pass. |
| Persistent catalog | A map should survive restart and concurrent import. | Needed for the persistent local pilot; Neo4j specifically is conditional. | Storage adapter outside the core. | After import identity is frozen. |

## Ownership and dependency direction

`Causyn.Api → Causyn.Engine`; the engine does not depend on controllers, configuration, network, or a driver. Move identifier rules into production code and make tests call it, not a test-only copy. Keep immutable map construction behind validation, including for typed callers that bypass JSON. Copy mutable input collections; “read-only” views of a caller-owned list can change mid-query. Keep a traversal’s buffers local to the request so two simultaneous queries cannot share a visited set.

Suggested split: transport reads bounded bytes; a map reader enforces JSON shape; domain construction checks graph invariants; analysis accepts only a valid snapshot; host translates outcomes. Allow the simplest in-process catalog for the HTTP slice. Introduce a store boundary around actual operations (import atomically, list, load by dataset ID), not a generic repository for every node. A database pushdown for traversal is an optimization to qualify separately, not a second definition of impact.

## Proposed HTTP surface

| Request | Successful behavior | Important distinction |
| --- | --- | --- |
| `POST /api/imports` | JSON map; 201 created or 200 unchanged | 409 for same ID with different content; malformed JSON and invalid map are separate 400 codes. |
| `GET /api/datasets` | List metadata by newest creation time, then ordinal ID | No datasets is 200 with an empty list. Pagination is a later shared-use requirement. |
| `GET /api/datasets/{id}/resources?query=...` | Search ID/name case-insensitively; trim query, max 80 characters; ordinal ID output, at most 100 resources | Blank query lists resources; no match is empty success, missing dataset is 404. Pin comparison semantics with fixtures. |
| `GET /api/datasets/{id}/impact/{target}?maxDepth=4` | Engine result, including limits and evidence | Invalid identifier/depth is 400; dataset or target not found is 404 with different codes. |
| `GET /api/datasets/{id}/export` | Download normalized import-format JSON | Never leak driver IDs or include server metadata as input. |
| `GET /health/live`, `GET /health/ready` | Liveness checks host; readiness checks essential storage | A storage outage should fail readiness with 503 without failing liveness. |

Use `application/problem+json` for application-handled errors: stable code, status, title, trace ID and bounded field errors when appropriate. Exercise 413 and 415 at the real HTTP boundary; the server may reject oversized input before application code can format the problem. Link request cancellation into traversal and storage. A database deadline (prototype target: 3 seconds of graph execution) maps to timeout, not “no dependents.” Treat unavailability separately from a timeout or a disconnected client.

## Why consider Neo4j, and when not to use it?

Labeled resources and directed edges fit the domain, but the v1 limits also make loading a small snapshot into memory plausible. Compare operational cost and measured latency before committing to database traversal. First persist and reload the validated map and run the reference engine on it. If query volume or map size later justifies pushing traversal down, compare the database results against the reference fixtures before exposing that path.

For a Neo4j prototype, enforce uniqueness of dataset ID and of `(datasetId, resourceId)`. Create the dataset, all resources, and edges in one transaction. Match both edge endpoints inside the dataset. In a race between imports, classify a genuine uniqueness conflict by reading the committed hash outside the failed transaction: same means unchanged; different means conflict. Do not classify an unavailable database as a duplicate. A failed edge write must roll the entire snapshot back.

Pass values as parameters. For database traversal, select among fixed depth-1-to-6 query shapes and constrain every node to the dataset, prevent repeated nodes, and fetch at most 101 qualifying paths. Apply a deadline independently of the row limit. Compare full path sets **below** the cap; **above** it compare validity, counts and cap semantics rather than requiring identical arbitrary subsets.

## Operating boundary

Keep the first pilot local and single-user. Dependency maps can reveal architecture and ownership. A shared deployment needs authenticated dataset access for search, impact, list and export; owner strings are not ACLs. Keep map contents out of default logs. Record outcome, elapsed time, sizes, and whether a cap was reached; avoid using unbounded IDs as metric labels. Verify restore by importing or restoring into an isolated instance and querying it. A successful export alone does not exercise operational recovery.
