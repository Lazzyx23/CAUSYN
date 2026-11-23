# 2. Define a map that can be replayed

| What? | Why? | Is it needed? | Where? | When? |
| --- | --- | --- | --- | --- |
| Strict schema-v1 input and semantic validation | Reject ambiguous or broken graphs before analysis. | Yes, even with no database. | JSON reader plus shared domain constructor. | Before creating the first snapshot. |
| Canonical bytes and import identity | Distinguish retries from attempted replacements. | Needed for repeatable import, not for walking a graph. | Snapshot serialization and catalog boundary. | Freeze before storing real snapshots. |

## Smallest useful specimen

```json
{
  "schemaVersion": 1,
  "datasetId": "demo-v1",
  "resources": [
    {"id":"checkout","name":"Checkout","kind":"service","owner":"commerce"},
    {"id":"payments","name":"Payments","kind":"external-api","owner":"commerce"}
  ],
  "dependencies": [
    {"from":"checkout","to":"payments","required":false,"source":"manual:owner-review","lastVerifiedAt":"2026-09-30T10:00:00+02:00"}
  ]
}
```

Selecting `payments` should return `checkout`, including `required:false`. Dropping that property is invalid rather than shorthand for false. A caller may supply a one-resource map with an empty dependency array; a disconnected node is valid data.

## Input decisions

| Field or invariant | Proposed v1 rule | Reason and diagnostic |
| --- | --- | --- |
| Schema | Integer `1`, known properties only at every level | Unknown fields may encode semantics the reader cannot honor. Reject duplicate JSON keys before binding; a dictionary may already have lost them. |
| Identity | Recommend ASCII lowercase letter first, then lowercase letters, digits or `-`, up to 64 characters; ordinal equality, no trimming or case folding | This matches the original map design. Today's test helper permits capitals and `_`, and a test explicitly accepts `order_db`. **Ratify before freezing v1**: either narrow the helper and update tests deliberately, or publish the wider grammar for both IDs and routes. Never silently rewrite IDs. |
| Resources | 1–100 entries; unique ID; kind exactly `application`, `service`, `database`, or `external-api` | Array positions are not identity. Two resources may share a display name. |
| Text | Trim name (1–100), owner (1–80), source (1–200); `unassigned` is a valid owner | Reject whitespace-only values. Specify the length unit before publishing non-ASCII fixtures; proposed .NET v1 unit: UTF-16 code units after trimming. |
| Edges | 0–300 entries; both endpoints in this map; one edge per ordered pair; no self-edge | `A→B` and `B→A` are distinct and allowed. Do not reject a real mutual dependency or resolve a missing endpoint from another dataset. |
| Evidence | Explicit boolean; timestamp with `Z` or explicit offset, normalized to UTC | Do not infer the host time zone. Future dates are claims supplied by the author, not proof of freshness. Source is inert text. |

Bound JSON input to 1 MiB at the import boundary, including when a sender omits a content length. Reject malformed types, null items, duplicate fields and unsupported versions before building adjacency. Then collect bounded semantic errors with locations such as `dependencies[0].to`. Resolve endpoint membership after collecting all resource IDs; valid input must not depend on array order. A failed validation yields errors and **no usable partial snapshot**.

## Canonicalization is a compatibility decision

Validate, trim only the documented text fields, normalize timestamps, sort resources by ordinal ID, sort edges by ordinal `(from,to)`, and serialize with fixed property order and UTF-8 settings. Include schema version and dataset ID; exclude creation time and storage keys. Compute SHA-256 over those bytes. Before implementation is accepted, pin timestamp precision (proposal: seven fractional digits and literal `Z`), escaping, whitespace, and absence of BOM with a byte-for-byte golden fixture. Do not rely on the current serializer defaults surviving upgrades.

For the specimen, `10:00:00+02:00` and `08:00:00Z` describe one instant. Swapping resource order or changing JSON indentation must preserve the hash. Flipping `required` must change it. Changing only `datasetId` also changes it: this fingerprint identifies import content, not topology across versions. It provides neither confidentiality nor proof of authorship.

| Existing ID | Incoming map | Result |
| --- | --- | --- |
| None | Valid | Create once with server creation metadata. |
| Present | Same normalized content hash | Unchanged; preserve original creation time. |
| Present | Different hash | Conflict; preserve the complete original. |
| Any | Invalid | Reject before a write. |

Export just the normalized input fields. Reimporting that export should be unchanged locally and create an equivalent snapshot in a fresh catalog. This is a portability check, not a backup of permissions, operational metadata, or the whole database.
