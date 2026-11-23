# CAUSYN

**Causality + Synapse** · *“Nothing changes alone.”*

If the payment provider changes, which parts of your system should you check first? CAUSYN is a dependency-impact engine in development for .NET. Its goal is to answer that question from a supplied dependency map and show the path behind each answer.

## A change, traced

Each arrow means **depends on**. Read `Checkout → Payment provider` as “Checkout depends on the payment provider.”

```text
Web app    → Checkout → Payment provider
Mobile app → Checkout → Orders database
Refunds               → Payment provider
```

Select **Payment provider** and trace the arrows backward:

| Distance | Potentially affected | Why |
| --- | --- | --- |
| One dependency away | Checkout, Refunds | Both use the provider directly. |
| Two dependencies away | Web app, Mobile app | Both rely on Checkout, which uses the provider. |

**Orders database is absent for a reason.** Checkout uses the database; the database does not depend on the payment provider. Simply listing everything connected to Checkout would give the wrong answer.

## What an answer should tell you

- **Which map was analyzed.** A map is an immutable snapshot. Importing equivalent content under the same dataset ID should leave it unchanged; conflicting content should preserve the original and report a conflict.
- **Why each resource appears.** Results should include dependency paths and available edge details, such as whether a dependency is optional, its source, and when it was last verified. Two paths can explain one affected resource.
- **Where the search stopped.** The planned contract supports depths of 1–6 edges and returns up to 100 paths. Cycles are allowed, but a path cannot revisit a resource. The result should say when additional paths were omitted.

These are explanations of **potential impact within the recorded map**. A path is a reason to investigate, not a prediction that a service will fail.

## Current state and direction

Read the [CAUSYN engine blueprint](docs/engine/README.md) for the proposed data contract, traversal algorithm, host boundaries, and milestone acceptance tests.

The engine is the next milestone. Today the repository contains a [.NET 10 API starter](Causyn.Api/Program.cs) and [resource identifier tests](Causyn.Tests/ResourceIdRulesTests.cs). The identifier rule currently lives in the test project. There is no callable impact engine, import endpoint, database integration, or browser UI yet.

1. **Prove the core in memory.** Validate one map and reproduce the payment-provider example from a plain .NET caller. Check cycles, optional edges, missing targets, and result limits.
2. **Expose the same behavior over HTTP.** The API should translate requests and results; it should not implement a second set of traversal rules.
3. **Persist immutable snapshots.** Verify repeat imports, conflicts, and export against persistent storage. Neo4j is the proposed store, not a prerequisite for testing the engine.

## Run the current project

From the repository root with the **.NET 10 SDK** installed:

```sh
dotnet test Causyn.slnx
dotnet run --project Causyn.Api/Causyn.Api.csproj --launch-profile https
```

The first command builds and runs the identifier tests. The second starts the API host; in Development, its OpenAPI document is available at `https://localhost:7025/openapi/v1.json`. The dependency example above describes the intended engine behavior, not a live endpoint.

*“No change stands alone; every choice reshapes what follows.”*
