# Building CAUSYN: the engine blueprint

**Status: design for the first implementation, not a shipped SDK.** CAUSYN (“Causality + Synapse”) helps someone deciding what to check before changing a resource. Its unit of explanation is a recorded dependency path. Its unit of history is an immutable map. Its first consumer should be a plain .NET caller, even though the repository started as an API.

| Topic | What decision does it settle? | Why / is this needed? | Where does it belong? | When do we need it? |
| --- | --- | --- | --- | --- |
| [Product and evidence](01-product.md) | What counts as a useful answer? | A list of neighbors alone cannot explain impact. | Consumer workflow | Before designing public types |
| [Snapshot contract](02-snapshots.md) | Which inputs represent the same map? | Replay and repeat imports depend on identity. | Core rules and JSON boundary | Before writing any store |
| [Impact engine](03-traversal.md) | Which paths qualify and when is an answer incomplete? | Traversing the wrong direction still returns plausible results. | In-memory engine | First executable milestone |
| [Hosts and storage](04-hosts.md) | Who owns parsing, queries, and persistence? | The HTTP service should reuse the engine without defining it. | API and storage adapter | After pure behavior is proven |
| [Verification and delivery](05-delivery.md) | What evidence warrants moving forward? | Passing a demo on one happy path is not a release criterion. | Tests, CI, local pilot | From first fixture to pilot |

**What exists:** [`Causyn.slnx`](../../Causyn.slnx) contains an ASP.NET Core host and xUnit tests. [`Program.cs`](../../Causyn.Api/Program.cs) registers controllers and development OpenAPI. The [identifier rule](../../Causyn.Tests/Domain/ResourceRules.cs) lives in the test assembly. There is no impact-analysis library yet. Names of new types, commands beyond those in the root README, and routes below describe proposals.

**How to use this plan:** implement one vertical slice at a time. Start with the decision example in chapter 1; carry its expected results through the library, HTTP host, and persistent store. Each chapter distinguishes a required behavior from a proposed implementation or a decision still to ratify. If two implementations disagree, settle the observable contract with a small fixture before changing either implementation.
