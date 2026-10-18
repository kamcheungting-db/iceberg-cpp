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
and manage persisted Iceberg SQL views. Engines supply schemas and SQL and handle
query resolution and execution. SQL parsing, materialized views, and additional
catalog backends are outside the first release.

## Proposed API

Reuse `Schema`, `Namespace`, `TableIdentifier`, and `Result`/`Status`. Keep view
metadata, versions, updates, and requirements separate from their table types.

| Surface | Operations |
| --- | --- |
| `Catalog` | `CreateView`, `LoadView`, `ListViews`, `ViewExists`, `ReplaceView`, `RenameView`, `DropView` |
| `View` | Owned metadata snapshots, current schema/version, history, properties, `SqlFor(dialect)`, `Refresh` |
| `View` mutations | `ReplaceVersion`, `UpdateProperties`, `UpdateLocation`; return a new `View` |

`ViewDefinition` contains schema, SQL representations, resolution context, and
summary. Creation also accepts properties and location. `Catalog::ReplaceView`
submits ordered updates with an explicit UUID assertion; convenience methods
build them from a loaded view.

Add optional `Catalog` methods defaulting to `NotSupported`, preserving source
compatibility and fitting `SessionCatalog`. New virtual methods change the ABI;
dependent binaries must rebuild. A separate Java-style `ViewCatalog` would need
additional capability discovery and session factories.

Snapshots own immutable metadata. Refresh checks the UUID and installs metadata,
revision, and session state together; failure preserves the old state. Callers
synchronize refresh. Retained snapshots remain usable independently.

## Metadata behavior

- Validate view-v1 IDs, references, required fields, and duplicate dialects without
  applying table-v1 schema restrictions. Parsing does not modify metadata.
- Match SQL dialects with ASCII case folding; return `NotFound` when the requested
  dialect is absent. Unlike Java's fallback lookup, never substitute another
  dialect. Preserve opaque unknown representations for round trips and property
  edits; reject definition changes involving them.
- Reuse schemas only when persisted content, including nested comments, matches.
  C++ `Schema::SameSchema` alone omits comments. Comment-only replacement must
  preserve the requested comments and leave old versions unchanged.
- Reuse equivalent retained versions; resolve REST `-1` placeholders in update
  order and use authoritative server-assigned IDs.
- Stage one history entry per builder: `v1 -> v2 -> v3` records only `v3`.
  Reactivations use fresh timestamps; property/location-only changes add no history.
  `Build()` finalizes once; publication requires a commit.
- Default `version.history.num-entries` to 10; require a positive value and
  protect current/new versions. Prevent dialect removal unless
  `replace.drop-dialect.allowed=true`.

## Commits and concurrency

Ordinary replacement refreshes its base, validates the definition, and submits
standard REST updates. The server commits atomically and returns authoritative
metadata. UUID assertions detect drop/recreation; two writers with the same UUID
can both succeed. Refresh does not close this race. Rename/drop have no UUID guard.

![Ordinary UUID checks allow competing replacements; a supported conditional backend rejects a stale revision.](diagrams/view-replacement-concurrency.svg)

Offer optional `ReplaceVersionIfUnchanged(definition, expected_revision)`, backed
by `Catalog::ReplaceViewIfUnchanged`. Callers capture metadata and an opaque
`ViewRevision` together through `snapshot()`. The revision binds UUID, catalog,
identifier, and protocol. The backend atomically checks the fixed expectation
with publication; every publication advances it. Conditional no-ops still require
a remote assertion.

Standard REST has no revision assertion. Return `NotSupported` until a documented
server profile provides atomic enforcement and token/error semantics. Never fall
back to an ordinary write or change the expectation. This is an optional extension.

Make at most one mutation attempt per call. Replacement conflicts return
`CommitFailed`; lost responses, mutation 5xx, or unusable commit responses return
`CommitStateUnknown`. Callers inspect state before deciding what to do next;
uncertain outcomes trigger no automatic replay or cleanup.

## REST integration

Honor advertised endpoints; unsupported operations return `NotSupported`. Follow
list pagination and fall back from unadvertised HEAD to advertised Load for
existence checks. Load inline metadata without requiring FileIO or storage access.
Reuse contextual authentication and `AuthManager::TableSession` for view sessions.
Distinguish absent response config from an empty map when refreshing overrides.

`RenameView` succeeds on HTTP 204 without an implicit load. A separate
`LoadView(to)` failure cannot change the completed rename result. Old handles
retain their identifier and snapshot. Confirmed missing-view drops succeed.

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
