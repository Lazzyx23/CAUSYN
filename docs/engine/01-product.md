# 1. The decision CAUSYN supports

| What? | Why? | Is it needed? | Where? | When? |
| --- | --- | --- | --- | --- |
| Explain potential dependents of a selected resource. | A change owner needs to know whom to involve and what to test. | Yes if indirect dependencies change that decision; validate this with users. | During change planning, using a named snapshot. | Before rollout; the same map can later help investigation. |

## One change, two different questions

A team is moving its payments integration to a new provider endpoint. “What does Checkout need?” asks for outgoing edges. “Who might need to test this provider change?” asks for incoming edges, continued transitively. Both questions are useful; version one answers the second. Calling both “related resources” would erase the distinction that makes the answer actionable.

Use six resources and five assertions: `web-app → checkout`, `mobile-app → checkout`, `checkout → payment-provider`, `refunds → payment-provider`, `checkout → orders-db`. Every arrow means **the left resource depends on the right resource**.

| Selected target | Expected result | What the owner learns |
| --- | --- | --- |
| `payment-provider`, depth 1 | Checkout and Refunds | Contact the direct integration owners. |
| Same target, depth 2 | Also Web app and Mobile app, each through Checkout | Include end-to-end checkout journeys in testing. |
| `orders-db`, depth 2 | Checkout, Web app, Mobile app; no Refunds | The same Checkout service participates in two different change plans. |
| `web-app`, any supported depth | Existing target; no recorded dependents | The map supplies no upstream consumers, not a guarantee of safety. |

## Evidence, inference, and action

- **Recorded fact about the input:** a specific snapshot contains a directed edge, its asserted source, a verification date, and whether it is required. The importer need not have independently verified any of those claims.
- **Inference:** a sequence of those edges provides a reason to investigate the starting resource. Optional dependencies still matter: analytics might degrade while checkout continues. A path is not a probability, outage forecast, or assertion that every change to the target matters.
- **Human decision:** an owner determines compatibility, failover, test coverage, and who needs notification. The engine returns context; it does not page teams or block releases.

## Validate the need before extending the scope

Ask a maintainer for a recent change, the map they had *at the time*, and their original review list. Have them write their list before seeing CAUSYN’s answer. Inspect one newly suggested dependent, one correctly excluded neighbor, and any missing known consumer. Record whether the discrepancy was a missing edge, stale evidence, misunderstood direction, or traversal error. These require different fixes.

Run the same exercise with a second change. If owners cannot maintain even a small representative map, prioritize the evidence collection problem over a richer visualization. If a simple inventory already yields the same review decisions, do not justify a graph database by counting nodes. The pilot succeeds when someone can explain a changed test or review decision by pointing to a defensible path.

## Boundaries worth defending

Start with one relation and one snapshot per query. Runtime calls, package imports, deployment ordering, and team ownership are not interchangeable arrows: blindly composing them can invent a causal chain. Owner is descriptive metadata, not a traversable edge or an authorization grant. Environments, versions, discovery, historical diffs, and confidence scores each need an explicit meaning and a consumer before expanding v1. An LLM reviewing a diff is a separate use case; the impact example requires no model, credentials, or provider.
