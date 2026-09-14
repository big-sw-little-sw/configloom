# Configloom
## Architecture and Implementation Specification

**Status:** Draft baseline for human-led, agent-assisted, or fully agentic implementation  
**Version:** 0.2  
**Date:** 2026-09-14

---

## 1. Purpose

Configloom is a multi-user, multi-project configuration service.

Teams may author configuration in:

- GitHub SaaS
- GitHub Enterprise
- GitLab Enterprise
- Configloom's own database-backed authoring system

Configloom resolves layered configuration into validated, immutable snapshots and serves those snapshots to applications, CLIs, SDKs, or local-file clients.

It also distributes client policy such as:

- minimum supported client versions
- deprecation warnings
- informational notices
- compatibility or safety blocks

Git is an optional source and workflow. Configloom must not require Git, pull requests, or CI/CD in order to use the service.

---

## 2. Design goals

- Self-service onboarding without operators manually wiring every project.
- Support both Git-backed and database-backed configuration authoring.
- Allow Git projects to choose their own publication workflow rather than imposing PRs.
- Support environment and label/branch-aware configuration with deterministic overrides.
- Preserve YAML, JSON, TOML, and INI as authoritative native documents.
- Allow explicit format conversion where feasible.
- Use pre-resolved immutable snapshots as the default production delivery model.
- Reuse one resolver in local tooling, CI, and service-side publication.
- Support both programmatic delivery and atomic local-file materialization.
- Provide project isolation, auditability, rollback, and explicit failure behavior.
- Give client policy an independent publication and freshness lifecycle.

---

## 3. Non-goals for the initial release

- Requiring Git or a GitOps workflow.
- Requiring PR review.
- Arbitrary executable commands delivered to clients.
- Guaranteed lossless conversion among incompatible formats.
- Request-specific fetch-time merging without a demonstrated requirement.
- A separate service or sidecar solely for client policy.
- Using client policy as a substitute for server-side authorization.

---

## 4. Core model

Authoring, resolution, publication, and delivery are deliberately separate.

| Concept       | Meaning                                                       | Key property                                               |
|---------------|---------------------------------------------------------------|------------------------------------------------------------|
| Project       | Ownership and authorization boundary                          | Multi-tenant boundary                                      |
| App           | The app for which this configuration is being managed         | Division within a project. Unit of onboarding / deboarding |
| Source        | Git-backed or database-backed authored documents              | Mutable authoring state                                    |
| Environment   | Deployment context such as dev/test/staging/prod              | Participates in resolution                                 |
| Label         | Named source/version selector; in Git often branch/tag/commit | Distinct from environment                                  |
| Revision      | Exact immutable inputs used for a build                       | Reproducibility                                            |
| Snapshot      | Complete resolved output for project/environment/label        | Immutable                                                  |
| Reference     | Mutable pointer such as `latest-approved`                     | Atomic advancement                                         |
| Client policy | Version constraints, warnings, and notices                    | Independent lifecycle                                      |

---

## 5. High-level architecture

Start as a **modular monolith**.

Keep logical modules behind explicit interfaces so they can be split later only when scale, security, or ownership makes that worthwhile.

Logical modules:

- **Management API + UI** — projects, onboarding, sources, previews, publishing, history, rollback, policy.
- **Source adapters** — GitHub SaaS, GitHub Enterprise, GitLab Enterprise, database.
- **Resolver** — parse, layer, merge, validate, generate provenance, render.
- **Snapshot builder** — resolve, validate, and persist immutable candidate snapshots.
- **Promotion manager** — evaluate/accept promotion requests and atomically advance references.
- **Delivery API** — serve snapshots/references with cache metadata.
- **Policy module** — independently publish and serve client policy.
- **CLI/SDKs** — fetch, cache, materialize files, preview resolution, evaluate policy.

---

## 6. Authoring, build, and promotion workflows

Configloom must support multiple workflows.

Git-backed configuration does **not** imply PR-first behavior.

Configloom separates two decisions:

- **Build trigger** — determines when a source revision should be resolved, validated, and stored as an immutable candidate snapshot.
- **Promotion policy** — determines when a candidate snapshot may advance a mutable reference such as `latest-approved`.

Conceptually:

```text
source event
    ↓
build candidate snapshot
    ↓
promotion decision
    ↓
advance reference
```

A build never implies promotion. Projects may configure automatic promotion where appropriate.

### 6.1 Supported workflow styles

#### A. PR / merge-gated publication

Typical flow:

```text
edit → pull/merge request → review → merge → build candidate → promotion decision
```

Good for:

- production configuration
- teams already using code review
- compliance-sensitive changes

This should be an option, not a requirement.

---

#### B. Push-to-branch publication

Typical flow:

```text
push to configured branch → build candidate → optional automatic promotion
```

Good for:

- low-friction internal projects
- development/test environments
- repositories where branch protection already provides sufficient governance

---

#### C. CI-driven publication

Typical flow:

```text
Git change → existing CI pipeline → Configloom build and/or promote API/CLI
```

The CI system may independently decide when Configloom should build a candidate and when a candidate should be promoted.

Good for organizations that already have:

- Jenkins
- GitHub Actions
- GitLab CI
- Buildkite
- CircleCI
- other internal CI systems

Configloom should expose a stable CLI/API rather than require a particular CI system.

---

#### D. Configloom webhook-triggered publication

Typical flow:

```text
Git provider event → Configloom webhook → build candidate → optional auto-promotion
```

Configloom may subscribe to push, merge, tag, or other provider events.

The project chooses:

- which event triggers a build
- which event may trigger promotion
- whether promotion is automatic or requires an explicit gate

---

#### E. Polling

Typical flow:

```text
Configloom periodically checks configured ref → detects new revision → build candidate
```

Use when:

- inbound webhooks cannot reach Configloom
- enterprise networking makes hooks difficult
- provider integration is intentionally minimal

Polling should be a fallback rather than the only mechanism.

---

#### F. Manual publication

A user may explicitly request:

```text
build revision X
promote snapshot Y to latest-approved
```

Useful for:

- testing
- unusual environments
- recovery
- teams that do not want automatic publication

---

#### G. Database-backed workflow

Typical flow:

```text
edit draft → preview → validate → build candidate → optional review/gate → promote
```

Review may be:

- disabled
- single approver
- multi-approver
- delegated to an external workflow later

The data model must not assume review always exists.


### 6.2 Promotion policies

Promotion is independent of the mechanism that triggered a build.

Initial policy options should include:

- **Automatic** — every successful candidate matching configured criteria advances the target reference.
- **Manual** — an authorized user explicitly promotes a candidate.
- **CI-controlled** — an external pipeline calls the promotion API after its own gates pass.
- **Source-event gated** — only candidates corresponding to an accepted source event, such as merge to a protected branch, may promote.
- **Approval gated** — one or more Configloom approvals are required before promotion. This may be deferred until there is a concrete need for native approval workflows.

Different environments may use different policies. For example:

```text
push feature branch → build only

push main → build → auto-promote dev

push main → build → CI gate → promote staging

same snapshot → manual/approval gate → promote prod
```

A snapshot can therefore be built once and promoted to one or more environment/reference targets where the configuration model permits it.


---

## 7. Git integration technologies

Treat authentication, event delivery, and repository access as separate choices.

### 7.1 GitHub SaaS / GitHub Enterprise

Preferred integration options, roughly in order:

#### Option 1 — GitHub App

Use a GitHub App installation for repository access and webhooks.

Advantages:

- repository-scoped installation
- fine-grained permissions
- short-lived installation tokens
- organization-level onboarding
- built-in webhook integration
- cleaner offboarding than personal credentials

Recommended default for managed GitHub integration.

#### Option 2 — Fine-grained personal access token

Advantages:

- simpler initial implementation
- useful for prototypes and restricted enterprise deployments

Tradeoffs:

- tied to a user
- lifecycle/rotation is less clean
- poorer organization-wide onboarding model

Suitable as a fallback.

#### Option 3 — Deploy key / SSH key

Advantages:

- simple read-only Git access
- provider-neutral at the Git transport layer

Tradeoffs:

- no identity/API integration
- no automatic webhook management
- separate key per repository is often required
- harder centralized lifecycle management

Useful for restricted environments.

#### Option 4 — Generic Git credentials

HTTPS username/token or SSH credentials.

Use only as an escape hatch for unusual enterprise setups.

---

### 7.2 GitLab Enterprise

Support several models:

#### Project webhook

Best when onboarding is per repository/project.

#### Group webhook

Useful when a team wants one Configloom integration inherited across many repositories.

#### System hook

Potentially useful for centrally managed GitLab Self-Managed installations.

This requires administrator-level cooperation and should not be required for normal onboarding.

#### Project/group/access token

May be used for repository/API access depending on enterprise policy.

#### Deploy key / SSH key

Provider-neutral fallback for read-only repository access.

---

## 8. Change detection technologies

Configloom should abstract change detection behind a provider-neutral interface.

Possible implementations:

| Mechanism | Latency | Setup | Coupling | Recommended use |
|---|---:|---:|---:|---|
| Provider webhook | Low | Medium | Provider-specific | Preferred |
| GitHub App webhook | Low | Low after installation | GitHub-specific | Preferred for GitHub |
| GitLab project/group hook | Low | Medium | GitLab-specific | Preferred for GitLab |
| CI callback | CI-dependent | Low if CI exists | Low | Strong option |
| Polling provider API | Medium | Low | Provider-specific | Fallback |
| `git fetch` polling | Medium | Low | Provider-neutral | Restricted environments |
| Manual build/publish | User-driven | Minimal | None | Always retain |

Do not make any one trigger mandatory.

A project may enable more than one trigger, but publication must remain idempotent for the same source revision.

---

## 9. Snapshot and promotion model

A build produces a complete, already-merged immutable **candidate snapshot**:

```text
source revision
    ↓
resolve
    ↓
validate
    ↓
persist immutable candidate snapshot
```

A separate promotion operation may then advance a reference:

```text
candidate snapshot
    ↓
promotion policy / gate
    ↓
atomically advance reference
```

Normal retrieval does not merge fragments.

A failed build cannot change an active reference. A successful build also does not change an active reference unless its configured promotion policy permits that transition.

Rollback is another reference transition: it repoints a reference to a previous valid snapshot. Snapshots themselves never mutate.

Use the terms consistently:

- **build** — creates an immutable snapshot
- **promote** — advances a mutable reference to a snapshot
- **publish** — avoid as a core state-transition term because it ambiguously combines build and promotion

---

## 10. Resolution semantics

Resolution is storage-independent and deterministic.

The same:

- inputs
- resolver version
- configuration model

must produce the same logical output.

Initial merge contract:

- Resolve an explicit ordered layer list.
- Maps/objects merge recursively by key.
- Scalars replace earlier values.
- Lists replace by default.
- Element-wise list merging requires a future explicit feature.
- Deletion uses an explicit Configloom directive rather than overloading `null`.
- Type changes are accepted only if final validation permits them.
- Unsupported or ambiguous constructs fail with document/location diagnostics.
- Provenance explains the source/layer responsible for resolved values.

Example conceptual layering:

```text
base
  ↓
environment
  ↓
label-specific/environment-specific override
```

The exact repository/file convention is still an open decision.

---

## 11. Native formats and conversion

First-class formats:

- YAML
- JSON
- TOML
- INI

Preserve:

- original source text
- source format
- file/document name
- path
- metadata
- source revision

Database-backed configuration stores native documents rather than only flattened key/value pairs.

Rules:

- Resolved output defaults to a declared native/target format.
- Mixed input formats may be supported through a typed internal tree.
- Authoritative source remains unchanged.
- Conversion is an explicit export operation.
- Conversion does not rewrite source unless explicitly requested.
- Unsupported or lossy conversions must be reported.
- Lossy conversion requires explicit acknowledgement.
- Comments, ordering, YAML anchors, formatting, and format-specific constructs are not guaranteed to round-trip.

---

## 12. Git onboarding

Human login, Git-provider credentials, and runtime-client credentials are separate.

Typical onboarding:

```text
create/join project
→ choose Git provider
→ authenticate/authorize
→ select repository
→ select configuration root/path
→ choose refs/labels/environments
→ choose change-detection workflow
→ preview discovered configuration
→ validate
→ enable publication
```

Principles:

- Prefer provider Apps/integrations or narrowly scoped credentials.
- All providers implement one internal repository abstraction.
- Enterprise instances carry base URL and TLS/trust configuration.
- Stored credentials are encrypted and rotatable.
- Configloom never returns stored secrets after registration.
- Onboarding validates repository access, parsing, resolution, and publication configuration.

---

## 13. Database-backed authoring

Database-backed projects expose the same logical source model but need first-class authoring UI.

Do not edit active configuration in place.

Use:

```text
draft revision → validation → publication → immutable revision/snapshot
```

UI capabilities:

- native-format editor
- syntax validation
- environment/label/document navigation
- resolved preview
- provenance
- diff against current publication
- diff against historical revisions
- diagnostics with source locations
- publish
- history
- rollback
- audit trail

Structured/form editing can come later.

---

## 14. Client delivery and cache

Expose a stable HTTP API first and keep SDKs thin.

Clients may request:

- a mutable reference such as `latest-approved`
- an immutable snapshot ID

Responses should include:

- snapshot ID
- content hash / ETag
- format
- publication metadata
- provenance metadata where requested

### Programmatic mode

SDK/API returns configuration directly.

### File mode

CLI or local agent materializes native files into a configured directory.

Requirements:

- cache immutable snapshots by ID/hash
- cache mutable references separately
- support conditional fetch via ETag/hash
- permit last-known-good configuration on network failure according to configured staleness policy
- never partially replace a file set
- download/validate to staging and atomically switch

Authenticated, project-scoped access is the default.

Anonymous access is allowed only for explicitly public projects.

---

## 15. Client policy and messaging

Client policy belongs in Configloom as a separate domain/module.

It should not initially be another service.

Policy is declarative, not executable.

Pinning a configuration snapshot must not pin operational policy.

Selectors may include:

- client/application
- version range
- environment
- platform

Actions may include:

- information
- warning
- deprecation
- minimum-version block
- effective date
- expiry date
- upgrade instructions/link

Never deliver arbitrary shell commands as policy.

Clients enforce policy locally.

Authorization for server operations remains server-side.

---

## 16. Offline policy behavior

Each policy class has:

- refresh interval
- maximum permitted age

Cached policy is the normal offline fallback.

| Policy class | Valid cached policy | Cache missing / too old |
|---|---|---|
| Informational/warning | Display if applicable; continue | Continue; optionally indicate policy status unknown |
| Minimum version | Enforce cached minimum, including existing block | Product-specific behavior |
| Critical compatibility/safety | Enforce cached rule | Block affected operations; retain help/version/update paths |

Missing-cache behavior must ship with the client because the client cannot retrieve that rule while offline.

---

## 17. Suggested API shape

Names are illustrative and should be finalized in a separate API-design pass.

```text
POST   /v1/projects
GET    /v1/projects/{id}
PATCH  /v1/projects/{id}

POST   /v1/projects/{id}/sources
POST   /v1/sources/{id}/validate

POST   /v1/projects/{id}/builds
GET    /v1/builds/{id}

GET    /v1/projects/{id}/snapshots/{snapshotId}

GET    /v1/projects/{id}/references/{environment}/{label}
POST   /v1/projects/{id}/references/{environment}/{label}:promote

GET    /v1/client-policy?client=...&version=...&environment=...
```

Database authoring requires additional resources for:

- drafts
- documents
- validation
- diffs
- publication

Prefer opaque identifiers and immutable snapshot URLs.

---

## 18. Security and tenancy

- Project is the primary authorization boundary.
- Every source, snapshot, reference, policy, and audit event belongs to a project/tenant.
- Suggested roles:
  - owner
  - publisher/maintainer
  - editor
  - reader
  - runtime client
- Runtime credentials are machine identities with independent rotation.
- Runtime credentials may be scoped by project/environment.
- Repository credentials belong in a secrets manager or KMS-backed encrypted store.
- Audit:
  - source registration/change
  - builds
  - publication
  - promotion/rollback
  - credential changes
  - policy publication
- Treat configuration as potentially secret-bearing.
- Do not log raw configuration bodies.

---

## 19. Failure and consistency rules

- Source unavailable → do not advance publication; continue serving last published snapshot.
- Parse/merge/validation error → build fails; active reference remains unchanged.
- Concurrent promotions → use optimistic concurrency / compare-and-swap.
- Build succeeds but promotion/reference update fails → snapshot remains valid and may be promoted later.
- Interrupted local-file update → existing materialized configuration remains intact.
- Git branch moves after build → provenance records exact commit SHA.
- Database document changes during build → build uses immutable draft revision.
- Duplicate webhook/CI events → build/publication operations must be idempotent by source revision and requested target.

---

## 20. Observability

Keep audit events separate from operational logs.

Metrics should include:

- build latency
- build failure rate
- source-provider errors
- webhook processing
- polling lag
- build rate
- promotion rate
- delivery latency/errors
- ETag/cache hit rate
- policy fetches
- stale-cache use

Trace build stages using project/build identifiers.

Never include raw secrets or configuration values in traces.

Expose snapshot provenance and resolver version for reproducibility.

---

## 21. Implementation phases

### Phase 0 — contracts and ADRs

Specify:

- merge semantics
- deletion semantics
- source model
- project/tenancy model
- snapshot/reference model
- API conventions
- policy freshness model
- build-trigger abstraction
- promotion-policy abstraction

Create conformance fixtures before production code.

### Phase 1 — resolver + local CLI

Implement:

- YAML/JSON/TOML/INI parsers
- typed internal tree
- merge/delete semantics
- validation hooks
- provenance
- deterministic rendering
- conversion diagnostics
- local resolve/preview

This engine should not depend on Git or the Configloom server.

### Phase 2 — service + minimal source workflow

Implement:

- projects
- database/source abstraction
- builds
- immutable candidate snapshots
- promotion policies
- references
- delivery API
- authentication/RBAC
- audit
- CLI fetch/cache/materialization

Add one Git provider integration, likely GitHub SaaS.

Do not require PR workflow.

### Phase 3 — Git workflow integrations

Add:

- GitHub App installation
- GitHub webhooks
- GitHub Enterprise
- GitLab Enterprise
- GitLab project/group hooks
- CI-triggered build and promotion APIs
- polling fallback
- manual build and promotion
- onboarding UI for independently choosing build triggers and promotion policy

### Phase 4 — database authoring

Add:

- drafts
- document persistence
- native editor
- preview/diff
- validation
- publish/history/rollback

### Phase 5 — client policy

Add:

- policy authoring
- policy publication
- evaluation contract
- SDK helpers
- cache/freshness behavior
- offline semantics

### Phase 6 — ecosystem

Consider:

- additional SDK languages
- structured/form editor
- format export/conversion
- promotion workflows
- more approval models
- fetch-time resolution only if a concrete requirement justifies it

---

## 22. Agent-friendly implementation contract

This specification, ADRs, and executable conformance tests are authoritative.

Agents and humans should follow these rules:

- Before each phase, produce a short plan mapping requirements to code, migrations, APIs, and tests.
- Do not infer product behavior from implementation accidents.
- Record deviations from accepted decisions rather than silently changing them.
- Build vertical slices with acceptance tests.
- Avoid speculative abstractions for providers or formats not yet implemented.
- Resolver behavior must have golden fixtures shared by server, CLI, and SDK conformance tests.
- Version public contracts.
- Keep migrations, API/schema changes, and generated files independently reviewable.
- Do not introduce fetch-time merging, executable policy, or silent lossy conversion without an explicit ADR.

---

## 23. MVP acceptance criteria

- A user can create a project without operator database edits.
- A project can be database-backed or Git-backed.
- Git onboarding does not require PRs.
- At least one Git workflow can trigger a build automatically.
- Building a snapshot does not implicitly advance an active reference.
- A project can configure promotion independently from its build trigger.
- Manual build and manual promotion are always available.
- Base plus environment overrides can be deterministically resolved.
- Publication produces an immutable snapshot tied to exact source revisions.
- Failed builds leave active references untouched.
- A runtime client can fetch `latest-approved`.
- A runtime client can cache and reuse its last valid snapshot offline.
- Local file materialization is atomic.
- YAML, JSON, TOML, and INI have resolver conformance fixtures.
- Native source is preserved.
- Users can inspect diagnostics, provenance, history, and rollback.
- Project A cannot access Project B without authorization.
- Audit records identify the human or machine identity responsible for publication/promotion.

---

## 24. Technology and implementation stack

The recommended baseline is **modern Java 25 or newer**. Configloom does **not** restrict itself to LTS Java releases. Newer Java releases may be adopted when useful and supported by the selected toolchain and dependencies.

### 24.1 Recommended baseline

| Area | Recommended choice |
|---|---|
| Language | Java 25+ |
| Service | Spring Boot on the JVM using virtual threads |
| Resolver/core | Framework-light Java library |
| Database | PostgreSQL |
| CLI | Java 25+ compiled with GraalVM Native Image |
| CLI framework | Picocli or equivalent Graal-friendly library |
| Public API | HTTP/REST + OpenAPI |
| Other SDKs | Thin language-native and/or OpenAPI-generated clients |
| Git | Provider APIs plus Git transport where necessary |
| Background work | Simple database-backed jobs/workers initially |
| UI | TypeScript + mainstream web framework; exact choice deferred |

Conceptual modules:

```text
configloom-core
  parsing
  typed configuration model
  resolution / merge semantics
  validation
  provenance
  rendering / conversion

configloom-server
  Spring Boot + virtual threads
  management + delivery APIs
  tenancy / authorization
  source adapters
  build orchestration
  promotion
  database authoring
  client policy
  audit / observability

configloom-cli
  Java 25+
  configloom-core for local operations
  HTTP client for remote operations
  GraalVM-native distribution

other SDKs
  language-native HTTP clients
  generated from OpenAPI where useful
  idiomatic convenience layers where needed
```

### 24.2 Authoritative Java core

`configloom-core` should remain framework-light and avoid Spring dependencies unless a concrete need justifies them.

It owns the authoritative implementation of:

```text
parse
resolve
merge
delete
validate
explain provenance
render
convert
```

This gives the server and CLI identical resolver semantics, keeps tests fast, reduces framework coupling, simplifies native CLI compilation, and leaves the core easier to embed or extract later.

The server may use Spring Boot extensively. Use Java virtual threads as the default concurrency model for request handling and suitable blocking I/O workloads. Prefer straightforward blocking code over introducing reactive programming solely for scalability. This does not imply every task should run on a virtual thread: CPU-bound work, bounded external resources, and operations requiring explicit concurrency limits still need appropriate executors, limits, or backpressure.

The use of Spring Boot does not imply the resolver should depend on Spring.

### 24.3 GraalVM-native CLI

Design the CLI for GraalVM Native Image from the beginning.

Goals:

- no Java installation required for end users
- fast startup
- single-executable distribution
- local resolution and validation
- remote Configloom operations

Representative local commands:

```text
configloom resolve
configloom validate
configloom diff
configloom convert
```

Representative remote commands:

```text
configloom fetch
configloom build
configloom promote
```

Choose core/CLI libraries with native-image compatibility in mind. Reflection-heavy or runtime-code-generation-heavy dependencies should require justification when simpler alternatives exist.

The JVM is the normal server runtime initially. Native-image support for the server is desirable but not required unless measurements establish a benefit.

### 24.4 SDK strategy

Distinguish **delivery/management SDKs** from a **resolver SDK**.

Delivery/management SDKs perform operations such as:

```text
fetch()
getSnapshot()
build()
promote()
getPolicy()
```

They generally do not need configuration-resolution logic. Prefer OpenAPI-generated foundations where useful, with thin idiomatic wrappers for authentication, caching, materialization, and convenience.

Candidate SDK languages may include Java, Python, Go, TypeScript/JavaScript, Rust, and others according to actual consumers.

The resolver API performs:

```text
parse()
merge()
validate()
render()
```

Do **not** initially reimplement it independently in every language. Java `configloom-core` is authoritative. Non-JVM local/CI consumers can invoke the Graal-native CLI when they need local resolution.

### 24.5 Rust-core alternative considered

A Rust authoritative core was considered:

```text
Rust configloom-core
  ├── Rust CLI
  ├── JVM/JNI or C ABI
  ├── Python bindings
  ├── Node bindings
  └── potentially WASM
```

It offers excellent native distribution and could provide one resolver implementation across languages. However, it also introduces native artifacts, JNI/C ABI integration, memory/error translation, ABI/versioning concerns, and coordinated cross-language releases.

Most consumers should receive already-resolved snapshots and therefore need HTTP/authentication/cache/materialization behavior rather than an embedded resolver.

For that reason, **Rust is not the recommended authoritative core initially**. A Rust delivery SDK or CLI remains possible later. Any independent resolver implementation must pass the same resolver conformance suite and requires an explicit ADR.

### 24.6 Infrastructure restraint

Do not introduce infrastructure simply because it is common in configuration platforms.

The initial architecture should not require:

- Kafka
- Redis
- Temporal
- Kubernetes operators
- a service mesh
- multiple independently deployed microservices

PostgreSQL may initially hold service metadata, database-authored documents, build/job state, promotion/reference coordination, and audit metadata.

Add queues, caches, workflow engines, or service decomposition only when measured load, reliability requirements, or operational constraints justify them.

### 24.7 Decision status

These are recommended baseline decisions rather than equal-weight unresolved options:

- Java **25+**, with no LTS-only restriction
- Spring Boot for the service, using Java virtual threads as the default request/I/O concurrency model
- framework-light Java for the authoritative resolver/core
- PostgreSQL as primary relational storage
- GraalVM Native Image for primary CLI distribution
- HTTP/REST + OpenAPI for public contracts
- thin/generated language-native delivery SDKs
- one authoritative resolver implementation initially

Exact libraries for parsing, validation, Git access, authentication, persistence, migrations, UI, and background jobs should be chosen during implementation planning and captured in ADRs when architecturally significant.

---

## 25. Decisions still open

- Exact source-file naming/layout convention.
- Ordered layer model.
- Whether projects may freely mix source formats.
- Target format declaration model.
- Validation model:
  - JSON Schema
  - project-provided validator
  - plugin
  - combination
- Human SSO/identity mechanism.
- Runtime authentication mechanism.
- Exact GitHub/GitLab credential options allowed in each deployment.
- Default build trigger when onboarding a Git repository.
- Default promotion policy for each environment.
- Whether auto-promotion should ever be the onboarding default.
- Client-policy cache age defaults.
- Fail-open/fail-closed behavior for minimum-version policy when no valid cache exists.
- Secret-management boundary.
- Snapshot storage and retention.
- Initial SDK languages.
- Whether `label` remains the user-facing term or becomes a more general channel/release selector.

---

## 26. Explicitly deferred architecture

### Fetch-time merging

Fetch-time merging remains an architectural option, not an MVP requirement.

Introduce it only for a concrete use case that cannot reasonably be served by publication-time snapshots, for example genuinely request-specific overlays.

If introduced, it must use the same resolver and conformance suite as publication-time resolution.

### Distributed service decomposition

Do not split Configloom into many services merely because logical modules exist.

Start modular. Split only for demonstrated requirements such as:

- independent scaling
- stronger security isolation
- separate ownership
- substantially different availability requirements

---

## 27. Recommended repository artifacts

```text
docs/
  architecture.md
  config-model.md
  security.md
  onboarding/
  adr/

openapi/
  configloom.yaml

schemas/
  client-policy/
  metadata/

conformance/
  resolver/
  conversion/

IMPLEMENTATION.md
```

### `docs/architecture.md`

This specification.

### `docs/adr/`

One short ADR per settled architectural decision. The initial ADR set should include the Java 25+/Spring Boot/GraalVM baseline and the authoritative-resolver decision.

### `docs/config-model.md`

Precise rules for:

- layers
- merging
- deletion
- provenance
- type handling
- format behavior

### `openapi/configloom.yaml`

Public management and delivery contracts.

### `conformance/resolver/`

Golden test cases containing:

- input sources/layers
- expected resolved output
- expected provenance
- expected invalid cases

### `conformance/conversion/`

Lossless and lossy conversion cases.

### `docs/onboarding/`

Git-provider and database-backed onboarding flows.

### `IMPLEMENTATION.md`

Living state for human/agent implementation:

- current phase
- completed work
- next work
- pending decisions
- deviations from plan

---

## 28. First implementation checkpoint

Do not begin with the web UI or provider integrations.

First make the configuration model executable:

1. settle the initial layer convention
2. settle merge/delete semantics
3. create cross-format golden fixtures
4. implement a local resolver CLI
5. prove deterministic output and provenance

Once these tests define what Configloom means by a resolved configuration, the server, Git integrations, CI hooks, database UI, and SDKs can all depend on the same contract. The next workflow contract to implement is the separation between snapshot builds and reference promotion.
