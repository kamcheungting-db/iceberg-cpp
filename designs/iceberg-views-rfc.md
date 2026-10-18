<!--
  Licensed to the Apache Software Foundation (ASF) under one
  or more contributor license agreements.  See the NOTICE file
  distributed with this work for additional information
  regarding copyright ownership.  The ASF licenses this file
  to you under the Apache License, Version 2.0 (the
  "License"); you may not use this file except in compliance
  with the License.  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing,
  software distributed under the License is distributed on an
  "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
  KIND, either express or implied.  See the License for the
  specific language governing permissions and limitations
  under the License.
-->

# RFC: Iceberg SQL views in C++

Status: proposed for discussion.

## Motivation and scope

Add view format v1 metadata and REST catalog support so C++ engines can inspect
and manage persisted Iceberg SQL views. For example, an engine can save an
`analytics.active_customers` view, and another compatible engine can load its
schema and SQL. Engines supply and validate schemas and SQL, resolve referenced
tables, and execute queries. The library stores definitions and manages their
lifecycle. SQL parsing, materialized views, and additional catalog backends are
outside the first release.

A **view version** records a definition: schema, SQL representations, resolution
context, and summary. Each SQL representation pairs a query with its engine
dialect, such as `spark`. A **metadata snapshot** is an owned, immutable copy of
the view's metadata, including retained versions, the current version, history,
and properties. It remains valid after later changes to the view.

## Proposed API

Reuse `Schema`, `Namespace`, `TableIdentifier`, and `Result`/`Status`. Keep view
metadata, versions, updates, and requirements separate from their table types.

| Surface | Operations |
| --- | --- |
| `Catalog` | `CreateView`, `LoadView`, `ListViews`, `ViewExists`, `ReplaceView`, `RenameView`, `DropView` |
| `View` | `snapshot()`, current schema/version, history, properties, `SqlFor(dialect)`, `Refresh` |
| `View` mutations | `ReplaceVersion`, `UpdateProperties`, `UpdateLocation`; return a new `View` |

`ViewDefinition` contains the definition fields above. `ViewCreation` wraps it
with optional location and properties. Create/load return a `View`;
rename/drop return `Status`. `Catalog::ReplaceView` is the low-level commit API:
it submits ordered updates with an explicit UUID assertion. The `View` mutation
helpers build these updates for callers.

Add optional `Catalog` methods defaulting to `NotSupported`, preserving source
compatibility and fitting `SessionCatalog`. New virtual methods change the ABI;
dependent binaries must rebuild. A separate Java-style `ViewCatalog` would need
additional capability discovery and session factories.

## Example: create, read, and replace

This illustrates the proposed API; view support is not implemented yet. Includes
are omitted, names are in namespace `iceberg`, and `catalog` is already configured.
The macros propagate `Result`/`Status` errors.

```cpp
Status ManageActiveCustomers(Catalog& catalog) {
  const TableIdentifier id{Namespace{{"analytics"}}, "active_customers"};
  ViewCreation creation;
  creation.definition.schema = std::make_shared<Schema>(
      std::vector<SchemaField>{
          SchemaField::MakeOptional(1, "customer_id", int64())});
  creation.definition.representations = {
      {.sql = "SELECT customer_id FROM customers WHERE active = true",
       .dialect = "spark"}};
  creation.definition.default_catalog = "prod";
  creation.definition.default_namespace = Namespace{{"crm"}};

  ICEBERG_RETURN_UNEXPECTED(catalog.CreateView(id, creation));
  ICEBERG_ASSIGN_OR_RAISE(auto view, catalog.LoadView(id));
  ICEBERG_ASSIGN_OR_RAISE(auto sql, view->SqlFor("SPARK"));
  // The engine can now plan and execute sql.sql using the saved context.

  const auto before = view->snapshot();
  auto next = creation.definition;
  next.representations[0].sql =
      "SELECT customer_id FROM customers WHERE active = true AND country = 'US'";
  ICEBERG_ASSIGN_OR_RAISE(auto updated, view->ReplaceVersion(next));
  // updated contains the new definition; view and before retain the old one.

  ICEBERG_RETURN_UNEXPECTED(view->Refresh());
  // view now has the latest metadata; before remains usable and unchanged.
  return {};
}
```

The engine resolves the unqualified `customers` reference as `prod.crm.customers`,
using the stored defaults. The view itself lives in `analytics`. Both queries
return the same `customer_id` schema, so replacement can reuse it. `SqlFor("SPARK")`
selects the saved Spark SQL; `SqlFor("trino")` returns `NotFound`.

`Refresh()` checks that the UUID still identifies the original view and installs
metadata, revision, and session state together. A failed refresh preserves the
old state. Callers synchronize refresh with other operations on the same handle.
Mutation helpers return a new handle and leave the original unchanged.

## Metadata behavior

- Validate view-v1 IDs, references, required fields, and duplicate dialects without
  applying table-v1 schema restrictions. Parsing does not modify metadata.
- Match SQL dialects with ASCII case folding; return `NotFound` when the requested
  dialect is absent. Unlike Java's fallback lookup, never substitute another
  dialect. Preserve opaque unknown representations for round trips and property
  edits; reject definition changes involving them.
- Reuse schemas only when persisted content, including nested comments, matches.
  For example, changing only the `customer_id` comment preserves that change in
  the replacement while older versions keep their original comments.
  C++ `Schema::SameSchema` alone omits comments and cannot implement this rule.
- Reuse equivalent retained versions; resolve REST `-1` placeholders in update
  order (`-1` means the last schema/version added in that request). Use the IDs
  returned by the server.
- Stage one history entry per builder. Selecting `v1 -> v2 -> v3` locally appends
  only `v3` to history; intermediate `v2` is recorded only if committed separately.
  Returning from `v1 -> v2 -> v1` locally records a fresh activation of `v1`.
  Property/location-only changes add no history. `Build()` finalizes once;
  publication requires a commit.
- Default `version.history.num-entries` to 10; require a positive value and
  protect current/new versions. Prevent dialect removal unless
  `replace.drop-dialect.allowed=true`.

## Commits and concurrency

Ordinary `ReplaceVersion` loads a fresh base without changing the original handle,
validates the definition, and submits standard REST updates. The server commits
atomically and returns authoritative metadata. UUID assertions detect
drop/recreation; two writers with the same UUID can both succeed. If both succeed,
the later commit determines the current version. Refresh does not close this race.
Rename/drop have no UUID guard.

![Ordinary UUID checks allow competing replacements; a supported conditional backend rejects a stale revision.](diagrams/view-replacement-concurrency.svg)

Callers requiring an unchanged-since-read guarantee use optional
`ReplaceVersionIfUnchanged(definition, expected_revision)`, backed by
`Catalog::ReplaceViewIfUnchanged`. Capture metadata and its optional `ViewRevision`
together through `snapshot()`, then derive the change from that captured metadata.
The **revision** is an opaque backend token bound to the UUID, catalog, identifier,
and protocol. Every metadata publication advances it: changing a property keeps
the current SQL version but invalidates an earlier revision.

The backend atomically checks the fixed revision when publishing. In the diagram,
A commits at `R1` and advances it to `R2`; B's conditional write using `R1` fails
without publishing changes. Conditional no-ops still require a remote assertion.

Standard REST has no revision assertion. Return `NotSupported` until a documented
server profile provides atomic enforcement and token/error semantics. Never fall
back to an ordinary write or change the expectation. This is an optional extension.

Make at most one mutation attempt per call:

| Situation | Result and caller action |
| --- | --- |
| Conditional revision is stale | `CommitFailed`; reload and reconsider the change before submitting another request. |
| Backend has only standard REST support | Conditional replacement returns `NotSupported`; the caller must explicitly choose whether an ordinary write is acceptable. |
| Commit response is lost, is HTTP 5xx, or cannot be used | `CommitStateUnknown`; the write may have succeeded. Inspect state before deciding what to do next. |

Other confirmed replacement conflicts also return `CommitFailed`. The library
never automatically replays a mutation or cleans up after an uncertain outcome.

## REST integration

Honor advertised endpoints; unsupported operations return `NotSupported`. Follow
list pagination and fall back from unadvertised HEAD to advertised Load for
existence checks. Load inline metadata without requiring FileIO or storage access.
Reuse contextual authentication and `AuthManager::TableSession` for view sessions.
Distinguish absent response config from an empty map when refreshing overrides.

`RenameView` succeeds on HTTP 204 without an implicit load. A separate
`LoadView(to)` failure cannot change the completed rename result. For example,
renaming `analytics.active_customers` to `analytics.current_customers` succeeds
even if a subsequent load is denied by the destination's read permissions.
Old handles retain their identifier and snapshot. Confirmed missing-view drops
succeed.

## Delivery and validation

| PR | Scope | Required evidence |
| --- | --- | --- |
| 1 | Metadata, JSON, builder, ordered updates | Pinned Java/Rust fixtures; malformed metadata, comment-only changes, IDs, history, retention, dialects |
| 2 | View inspection and REST read operations | Snapshot lifetime, UUID refresh checks, endpoint support, pagination, contextual/view sessions |
| 3 | Catalog mutations and optional conditional API | Reference-service lifecycle tests, conflicts, uncertain outcomes, rename/load separation, unsupported conditional behavior |

Run affected tests and pre-commit. Pin a view-capable REST service before PR 2.
Enable conditional writes only after testing atomicity with competing requests
against a real service. Ordinary REST support can ship independently.

## Feedback requested

1. Are optional `Catalog` methods and the proposed immutable snapshot API suitable?
2. Are explicit dialect lookup, opaque representation preservation, and single
   mutation attempts the right initial policies?
3. Which REST service should provide interoperability tests, and which documented
   profile could support conditional replacement?

## References

The format and REST specifications define the wire contract. Java guides lifecycle
behavior; Rust guides metadata and ordered updates. At the pinned Rust revision,
catalog view operations are absent.

- [View specification][spec] and [REST catalog specification][rest-spec].
- [Java view metadata builder][java-metadata] and [Rust view metadata builder][rust-builder].
- [Editable diagram source](diagrams/view-replacement-concurrency.dot).

[spec]: https://github.com/apache/iceberg/blob/524fe9ef1b4aa72be19623f667738b21a02b23b2/format/view-spec.md
[rest-spec]: https://github.com/apache/iceberg/blob/524fe9ef1b4aa72be19623f667738b21a02b23b2/open-api/rest-catalog-open-api.yaml
[java-metadata]: https://github.com/apache/iceberg/blob/524fe9ef1b4aa72be19623f667738b21a02b23b2/core/src/main/java/org/apache/iceberg/view/ViewMetadata.java
[rust-builder]: https://github.com/apache/iceberg-rust/blob/697abcee75b503fc106f445dae8bcfc401524dcc/crates/iceberg/src/spec/view_metadata_builder.rs
