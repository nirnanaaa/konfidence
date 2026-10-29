# API and UI

People, CI and bridges talk to the control plane through one protobuf schema.
This page owns every proto name and sketch (D-API-06), compatibility,
authentication, roles, the write and read paths, and the UI. Message
semantics belong to [03](03-bundles-and-transfer.md) (transfer) and
[04](04-trust-and-attestation.md) (enrollment, reports); CRD names to
[02](02-domain-model.md).

## One schema, two package trees, two listeners

All APIs live in one protobuf module served with Connect (D-API-01). Connect
handlers speak gRPC, gRPC-Web and the Connect protocol and pick one by
`Content-Type` ([Connect multi-protocol](https://connectrpc.com/docs/multi-protocol)).

| Package | Callers | Auth | Transport |
| --- | --- | --- | --- |
| `konfidence.bridge.v1` | bridges only | mTLS with the bridge identity key ([04](04-trust-and-attestation.md)) | gRPC over HTTP/2 |
| `konfidence.api.v1` | UI, CLI, machines | OIDC session; short-lived OIDC token for machines (M2+) | Connect JSON, also gRPC |
| `konfidence.authz.v1` | none; method options | n/a | n/a |

Every API replica serves both trees on two listeners:

- **Bridge listener.** HTTP/2 only, no CORS, no cookies, long idle timeouts
  on the load balancer. Stream lifetime is enforced in the `Connect` handler
  (D-TOPO-06), not by a listener setting.
- **Public listener.** HTTP/1.1 and HTTP/2, CORS through
  `connectrpc.com/cors`, session cookies, plain-HTTP download routes.

## `BridgeService` is the one bridge wire protocol

`konfidence.bridge.v1` has one service with five RPCs (D-API-05). Bridges
call; the control plane only answers (D-PROD-05). Isolated bridges use none
of it: their files go through `OfflineService` below.

```proto
service BridgeService {
  rpc Enroll(EnrollRequest) returns (EnrollResponse);                        // M1, token-based
  rpc Connect(stream BridgeMessage) returns (stream ServerMessage);          // M1, control stream
  rpc PlanTransfer(PlanTransferRequest) returns (PlanTransferResponse);      // M1
  rpc FetchBlob(FetchBlobRequest) returns (stream BlobChunk);                // M1, 1 MiB chunks
  rpc SubmitReport(SubmitReportRequest) returns (SubmitReportResponse);      // M1
}
message BridgeMessage { oneof msg { Hello hello = 1; Ack ack = 2; } }
message ServerMessage {
  oneof msg { Welcome welcome = 1; DeliveryAvailable delivery_available = 2; Reconnect reconnect = 3; }
}
message Hello {                            // first bridge message on every stream
  string landscape_id = 1;                 // lsc-…
  string bridge_version = 2;               // semver
  repeated uint32 bundle_formats = 3;      // readable format majors (D-BUN-03)
  repeated string capabilities = 4;        // "transfer.full", "tombstone", …
  uint64 last_applied_delivery_sequence = 5;
}
message Welcome {                          // first server message
  string min_bridge_version = 1;
  repeated string enabled_capabilities = 2;  // intersection of both sides
  uint64 desired_delivery_sequence = 3;
  uint64 delivery_floor = 4;                 // D-ATT-03
  uint64 accepted_report_sequence = 5;       // D-ATT-03
}
message DeliveryAvailable { uint64 delivery_sequence = 1; }
message Reconnect { string reason = 1; }   // bridge dials a fresh TCP connection
message Ack { uint64 delivery_sequence = 1; }
message PlanTransferRequest {
  uint64 delivery_sequence = 1;
  repeated string inventory = 2;           // full, sorted "sha256:<hex>"
  TransferProfile profile = 3;             // 03 owns the content
}
message PlanTransferResponse {
  bytes delivery_descriptor = 1;           // signed delivery
  repeated BlobRef missing = 2;
}
message BlobRef { string digest = 1; uint64 size = 2; }
message FetchBlobRequest { string digest = 1; uint64 offset = 2; }
message BlobChunk { bytes data = 1; }
message SubmitReportRequest { bytes envelope = 1; }   // DSSE, same bytes as a file import
message SubmitReportResponse { uint64 accepted_report_sequence = 1; }
```

[04](04-trust-and-attestation.md) specifies the enrollment messages; the
response is unsigned (D-TRUST-06). There is no unary handshake, because
stateless replicas cannot bind it to a later stream.

Bridges retry with exponential backoff and full jitter from 1 s to 5 min. On
`resource_exhausted` from `FetchBlob` they wait at least the returned delay
and resume from their offset. [01](01-control-plane.md) owns stream rules.

`SubmitReport` and `OfflineService.ImportStateRequest` call the same import
code. Import rejections are API errors, not CR reasons. The error detail
carries one of `EnrollmentUnconfirmed`, `KeyRevoked`, `SignatureInvalid`,
`LandscapeMismatch` or `ReportSequenceRegressed` (names in
[02](02-domain-model.md#reasons-form-one-vocabulary-for-condition-report-and-deployment)).

## Public services group by resource family

Resource names follow `projects/{project}/applications/{application}` paths.

| Service | RPCs and coverage | Milestone |
| --- | --- | --- |
| `AuthService` | `WhoAmI`, `Logout`, own permissions | M1 |
| `ProjectService` | `Project`, role bindings, trust rules | M1; trust rules M2+ |
| `LandscapeService` | `Landscape`, enrollment tokens, `ConfirmEnrollment`, `SkipEnrollmentCheck`, `RevokeLandscapeKey`, `LandscapeGrant` | M1; revoke M2+ |
| `CredentialService` | `GitCredential` metadata and status; key material write-only | M1 |
| `ApplicationService` | `Application`, sources, sync now | M1 |
| `ReleaseService` | list, get, `GetPlacement`, `GetPath` ([07](07-sources-and-releases.md#release-history-answers-diff-placement-and-path-queries)), diff (D-REL-03) | M1; diff M2+ |
| `PromotionService` | `PromotionConfig`, `Promotion`, `ApproveGate`, `RejectPromotion`, `OverrideGate`, `SetGateState`, `RollbackPromotion`, `RemoveApplication` | M1 up to `RejectPromotion`; rest M2+ |
| `DeploymentService` | `Deployment` list and get, per-resource status, history; the report query API for gate writers | M1 |
| `OfflineService` | `ExportCTF`, `ImportStateRequest`, `ImportAttestation`, `ImportEnrollmentRequest`, `ExportEnrollmentResponse` | M1 |
| `AuditService` | audit log query | M2+ |
| `WatchService` | change notices for the UI | M1 |

`OfflineService` holds every file a courier carries (D-XFER-05).
`ImportStateRequest` imports the file's reports, inserts `ctf.export` against
its inventory and returns the job reference. `ExportCTF` takes that reference
and returns the download once the CTF exists; without one it inserts
`ctf.export` for a full CTF. A state request file from a bridge below
`min_bridge_version` fails the export with `failed_precondition`.
`ImportAttestation` serves report-only trips. Gate writers poll
`DeploymentService`; konfidence pushes no events to them.

## Compatibility is additive-only because bridges upgrade by hand

The rules apply to all three package trees (D-API-02):

- CI runs `buf breaking` with the `WIRE_JSON` category, the minimum Buf
  documents for Connect JSON clients
  ([buf breaking rules](https://buf.build/docs/breaking/rules/)).
- Within `v1` only additive changes land. A field stays `deprecated = true`
  until the minimum bridge version no longer uses it, then becomes
  `reserved`. Enum values are never renumbered.
- Proto enums use the prefixed UPPER_SNAKE form and map 1:1 to the
  PascalCase CRD values in [02](02-domain-model.md): `HEALTH_DEGRADED` is
  `Degraded`.
- Capabilities are strings, not enums, so an old control plane keeps an
  unknown one. Each feature registers its string in one Go constants file
  next to the proto. Known: `transfer.full`, `transfer.thin`, `tombstone`,
  `attestation.v1`.

The control plane stores reported version, formats and capabilities in
`Landscape.status.bridge`; isolated bridges report them in the state request
file. Promotion validation rejects a target whose landscape lacks a needed
capability with `failed_precondition`. A bridge below `min_bridge_version`
still connects and reports, gets condition `BridgeOutdated`, and receives no
new delivery, so the operator sees why deployments stopped.

## Sessions for people, short-lived tokens for machines

The API is the only OIDC client; no IdP token reaches the browser (ADR-0030).

- **Browser (M1).** Authorization code flow with PKCE against one IdP per
  installation. The API stores the session in `sessions` and sets an
  `HttpOnly`, `Secure`, `SameSite=Lax` cookie with idle and absolute expiry.
- **CLI (M1).** Loopback PKCE flow; the session goes into the OS keychain.
- **Machines (M2+).** Short-lived OIDC tokens checked against trust rules on
  `Project` or `Landscape` (D-API-04). GitHub Actions, Kubernetes projected
  ServiceAccount tokens and transfer-station robots use the same rule type.
  `setBy` is `<issuer>#<sub>`. No long-lived API tokens.

Group claims map to roles through role bindings on `Project` and `Landscape`,
resolved per request from the informer cache. The public listener rejects
state-changing requests without the `Connect-Protocol-Version` header, which
forces a CORS preflight on foreign origins (unverified for connect-web).

## Roles exist at three scopes

Each role includes the ones above it in its scope.

| Scope | Role | May |
| --- | --- | --- |
| platform | `platform-admin` | create projects and every `Landscape`, assign their admins |
| project | `viewer` | read everything in the project |
| project | `developer` | edit `Application` and `PromotionConfig`, start promotions, trigger sync |
| project | `release-manager` | approve gates, reject promotions |
| project | `project-admin` | manage credentials and role bindings, override gates, request landscape access |
| landscape | `landscape-viewer` | read landscape status and all deployments on it |
| landscape | `landscape-operator` | export CTF, import state request and attestation files |
| landscape | `landscape-admin` | issue enrollment tokens, confirm or skip the enrollment code, approve grants, set link templates and `staleAfter` |

Rules across the table:

- **Override is not approve.** `OverrideGate` needs `project-admin` and a
  reason; the audit entry marks it (D-PROMO-01, ADR-0040).
- **Sharing needs both sides.** `project-admin` requests a `LandscapeGrant`,
  `landscape-admin` approves it. The `Boundary` is written on the landscape
  only; the UI shows the grant ID for the admin to copy (D-TEN-02).
- **Offline work needs a landscape role**, because a CTF covers all projects.
- **Skipping the enrollment code** (`SkipEnrollmentCheck`) needs a reason
  and is audited and shown in the UI (D-TRUST-02).

Every RPC declares its permission as a method option. One interceptor reads
it, resolves the scope from the resource name and checks the caller's roles.
A CI lint fails on any public RPC without it.

```proto
rpc ApproveGate(ApproveGateRequest) returns (ApproveGateResponse) {
  option (konfidence.authz.v1.rule) = { scope: SCOPE_PROJECT, permission: "promotion.gate.approve" };
}
```

## The API writes CRDs; controllers do the rest

The API is the only writer of user intent into CRDs (D-API-03):

1. Authenticate and authorize in the interceptor.
2. Validate with protovalidate, then domain checks: capabilities, reporting
   level against gates and completion (D-POL-05).
3. Insert an `audit_log` row with outcome `Pending`.
4. Create or patch the CRD with optimistic concurrency, with the audit ID in
   annotation `konfidence.cloud/audit-id` ([02](02-domain-model.md)).
5. Set the audit row to `Succeeded` or `Failed`.

A conflict returns `aborted`; the UI refetches and retries once. A sweeper
resolves `Pending` rows left by a crash after step 4. Gate calls patch
`Promotion.status.gates` (ADR-0039). Other work goes to River, and the UI
watches the returned job reference.

Browsers support server streaming only, so files bypass Connect streams
([connect-web](https://connectrpc.com/docs/web/getting-started)).
A finished `ctf.export` is served as a short-lived signed URL for a plain
`GET`. Uploads go as `bytes` in a unary call, capped at 16 MiB.

## Lists come from cache, history from Postgres, watch only invalidates

- **List and Get** read the informer cache; **history** reads Postgres with
  cursor pagination (`page_size` 50, at most 500).
- **Watch** is a server stream, `WatchService.Watch(project, kinds)`. It
  sends `{kind, name, resource_version}` notices, never objects. Authz runs
  on the watch scope and again on every refetch. Informer events and
  `konfidence_events` notices feed it. On reconnect the UI lists and watches
  again; there are no resume tokens.

## Status shows how we know, not only what we know

The UI never shows a bare "healthy" (D-UI-02). The API computes one
`Observation` per `Deployment`, so UI and CLI agree.

```proto
message Observation {
  Basis basis = 1;                          // BASIS_DISPATCHED | BASIS_ATTESTED | BASIS_LIVE
  google.protobuf.Timestamp observed_at = 2;
  Health health = 3;                        // HEALTH_UNKNOWN while dispatched
  string reason = 4;                        // reason from 02
}
```

| Basis | Meaning | Shown as |
| --- | --- | --- |
| `BASIS_DISPATCHED` | delivery acked or exported, no matching report yet | hollow chip, "dispatched 2 h ago" or "exported at T" |
| `BASIS_ATTESTED` | a signed report matches this `Deployment` (D-ATT-05) | health colour plus "as of T" |
| `BASIS_LIVE` | attached, stream up, last report younger than `staleAfter` | health colour, no age |

Isolated landscapes never reach `LIVE`; their report age always shows. Past
`Landscape.spec.staleAfter` (default 30 min; isolated landscapes set days)
the chip dims and reads "stale". The last report time comes from the
replica's in-memory map ([01](01-control-plane.md)), never from CR status. A
blocking reason or the phase `Unreachable` overrides the colour.

Deep links come only from `Landscape.spec.linkTemplates` (D-UI-03).

## The UI is three screens in M1

The UI uses Svelte, Skeleton and `konfidence-design-system` (D-UI-01) and
calls only `konfidence.api.v1` through connect-web. Navigation runs Fleet →
Landscape and Projects → Project → Application → Release, with Promotion →
Deployment under each project.

1. **M1 Fleet.** Mode, bridge version with `BridgeOutdated`, last contact,
   last report age, reporting level, deployment counts per basis.
2. **M1 Application.** Where-running matrix: landscapes as rows, cells with
   release label (`payments-api@42`) and observation chip.
3. **M1 Promotion.** Waves as columns, landscapes as cells, **Approve** and
   **Reject** gated by role.
4. **M2+** Deployment, Release with diff, Landscape with enrollment code,
   grant ID and offline actions, Audit. In M1 the CLI covers enrollment and
   offline files through the same public API.

## Path

1. **M1** Create the proto module with all three trees, `buf lint` and
   `buf breaking` against main with `WIRE_JSON` in CI.
2. **M1** Add the `konfidence.authz.v1.rule` option, the interceptor and the
   lint for public RPCs.
3. **M1** Implement `BridgeService`: `Enroll`, `Connect` with `Hello` and
   `Welcome`, `PlanTransfer`, `FetchBlob`, `SubmitReport`.
4. **M1** Implement OIDC login, `sessions`, `WhoAmI` and role resolution.
5. **M1** Implement read services from the informer cache plus
   `WatchService`, and the audited write path for `Project`, `GitCredential`
   key material, `Landscape` with enrollment tokens, `LandscapeGrant` request
   and approve with grant-ID minting, `ConfirmEnrollment`,
   `SkipEnrollmentCheck`, `Application`, `PromotionConfig`, `Promotion`,
   `ApproveGate` and `RejectPromotion`.
6. **M1** Implement `OfflineService` with the signed download route.
7. **M1** Build Fleet, Application and Promotion screens.
8. **M2+** Machine trust rules with `SetGateState`, `OverrideGate`,
   `RollbackPromotion`, `RemoveApplication`, `AuditService`, release diff and
   the remaining screens.

Later: custom roles, watch resume tokens if refetch load shows, and opt-in
four-eyes approval declared in [05](05-promotion-and-rollout.md).
