# API and UI

People, CI and bridges talk to the control plane through one protobuf schema.
This page owns every proto name and the only proto sketch, compatibility,
authentication, roles, and the UI. Statement and enrollment semantics belong
to [04](04-trust-and-attestation.md), blob planning to
[03](03-bundles-and-transfer.md), and CRD and table names to
[02](02-domain-model.md).

## One schema serves bridges and people on two listeners

All APIs live in one protobuf module served with Connect (D-API-01). Connect
handlers speak gRPC, gRPC-Web and the Connect protocol and pick one by
`Content-Type` ([Connect multi-protocol](https://connectrpc.com/docs/multi-protocol)).

| Package | Callers | Auth | Listener |
| --- | --- | --- | --- |
| `konfidence.bridge.v1` | bridges only | mTLS with the bridge identity key ([04](04-trust-and-attestation.md)) | bridge: HTTP/2 only, no CORS, no cookies, idle timeout above the 4 min `Sync` hold |
| `konfidence.api.v1` | UI, CLI, machines | OIDC session; short-lived OIDC token for machines (M2+) | public: HTTP/1.1 and HTTP/2, CORS via `connectrpc.com/cors`, cookies, download routes |
| `konfidence.authz.v1` | none; method options | n/a | n/a |

## The bridge service moves one statement up and one answer down

`BridgeService` has three RPCs (D-API-05). Bridges call; the control plane
only answers (D-PROD-05).

```proto
service BridgeService {
  rpc Enroll(EnrollRequest) returns (EnrollResponse);           // token-based; stores statement 0 (04)
  rpc Sync(SignedStatement) returns (Answer);                   // held until the delivery changes or 4 min pass
  rpc FetchBlob(FetchBlobRequest) returns (stream BlobChunk);   // 1 MiB chunks, resumable by offset
}
message SignedStatement { bytes envelope = 1; }   // signed by the bridge key; same bytes as the statement file
message Statement {                               // payload inside the envelope; 04 owns the encoding
  string landscape_id = 1;                        // lsc-…
  uint64 statement_seq = 2;                       // reused until the content changes
  string bridge_version = 3;
  repeated string capabilities = 4;               // "bundle.v1", "statement.v1"
  uint64 verified_delivery_seq = 5;
  uint64 applied_delivery_seq = 6;
  oneof inventory { BlobList blobs = 7; string blobs_digest = 8; }   // digest when unchanged (D-XFER-03)
  repeated Entry entries = 9;                     // fields capped by each entry's reporting level
  repeated TransferProfile transfer_profile = 10; // per application; blob planning only (03)
  bool enrollment_check_skipped = 11;             // trust confirm --skip-check (04)
}
message Entry {
  string project_uid = 1; string application = 2; string entry_digest = 3; ReportingLevel level = 4;
  string reason = 5; Health health = 6; string policy_version = 7; string effective_values_digest = 8;
  repeated Resource resources = 9;                // level Resources only
  uint64 delivery_sequence = 10; string release_id = 11;   // delivery_sequence: entry matching (05)
}
message TransferProfile { string project_uid = 1; string application = 2; ImageTransfer images = 3; bool registry = 4; }
message Answer {
  bytes delivery = 1;                             // newest signed delivery; empty when the bridge has it
  repeated BlobRef missing = 2;                   // blob plan (03)
  string min_bridge_version = 3;
}
message BlobList { repeated string digests = 1; }  // sorted "sha256:<hex>"
message BlobRef { string digest = 1; uint64 size = 2; }
message FetchBlobRequest { string digest = 1; uint64 offset = 2; }
message BlobChunk { bytes data = 1; }
```

The bridge holds one `Sync` call at a time and cancels it to send a changed
statement. After an empty answer it calls again at once. A bridge below
`min_bridge_version` still has its statement imported, but its answer
carries no delivery, and the UI flags the bridge as outdated. Import
rejections are API errors with the detail `EnrollmentUnconfirmed`,
`KeyRevoked`, `SignatureInvalid` or `LandscapeMismatch`.

## The offline service takes a statement file and returns a CTF

`OfflineService` mirrors `BridgeService` with files (D-XFER-05).
`Enroll(file)` takes the enrollment request file and returns the response
file ([04](04-trust-and-attestation.md)). `Sync(landscape_id, statement)`
imports the statement through the same code as `BridgeService.Sync` and
returns `delivery_seq` and a short-lived signed `download_url`. Without a
statement it plans every blob. It holds while a `delivery.build` for the
landscape is unfinished. A statement file always carries the full inventory.
A plain `GET` on the URL streams the CTF from the blob store, so no job
writes it, and downloads bypass Connect streaming, which browsers lack
([connect-web](https://connectrpc.com/docs/web/getting-started)). Uploads
are unary `bytes`, capped at 16 MiB.

## Public services group by resource family

Resource names are paths such as `projects/{project}/applications/{application}`.

| Service | RPCs and coverage | Milestone |
| --- | --- | --- |
| `AuthService` | `WhoAmI`, `Logout`, own permissions | M1 |
| `ProjectService` | `Project`, role bindings, trust rules | M1; trust rules M2+ |
| `LandscapeService` | `Landscape`, `CreateEnrollmentToken`, `ConfirmEnrollment`, `SkipEnrollmentCheck`, `RevokeLandscapeKey`, `LandscapeGrant`, `GetLandscapeState` | M1; revoke M2+ |
| `CredentialService` | `GitCredential` metadata and status; key material write-only | M1 |
| `ApplicationService` | `Application`, `SyncApplication` | M1 |
| `ReleaseService` | list, get, `GetPlacement`, `GetPath` ([07](07-sources-and-releases.md)), `DiffReleases` (D-REL-03) | M1; diff M2+ |
| `PromotionService` | `PromotionConfig`, `CreatePromotion`, `GetPromotion`, `ListPromotions`, `ListRolloutTargets`, `SetGateState` | M1; removal and external writers M2+ |
| `OfflineService` | `Enroll`, `Sync`, the CTF download route | M1 |
| `AuditService` | audit log query | M2+ |

`CreatePromotion` takes a release, or none for a removal, and writes a
`promotions` row (D-TOPO-07). A rollback is `CreatePromotion` with an older
release. `GetLandscapeState` returns a landscape's slots and the entries of
its newest statement. Gate writers poll it and `ListRolloutTargets`.

`SetGateState(promotion, wave, type, state, reason)` is the only gate write
(D-PROMO-01). Approval sets `Satisfied`. Reject sets `Rejected` and ends the
promotion. An external writer sets any state. When the caller does not own
the gate type and holds `promotion.gate.override`, the handler records the
write as an audited override. The call inserts `promotion.advance`.

## Compatibility is additive-only because bridges upgrade by hand

- CI runs `buf breaking` with `WIRE_JSON`, the minimum for Connect JSON
  clients ([buf breaking rules](https://buf.build/docs/breaking/rules/)).
- Within `v1` only additive changes land (D-API-02). A field stays
  `deprecated = true` until the minimum bridge version no longer uses it,
  then becomes `reserved`. Enum values are never renumbered.
- Proto enums prefix the PascalCase values of [02](02-domain-model.md) in
  UPPER_SNAKE: `HEALTH_DEGRADED` is `Degraded`.
- Capabilities are strings, one Go constant each next to the proto, so an
  old control plane keeps an unknown one (D-BUN-03).

## Sessions for people, short-lived tokens for machines

The API is the only OIDC client; no IdP token reaches the browser (ADR-0030).
The browser (M1) uses the authorization code flow with PKCE against one IdP.
The API stores the session in `sessions` and sets an `HttpOnly`, `Secure`,
`SameSite=Lax` cookie with idle and absolute expiry. The CLI (M1) uses a
loopback PKCE flow and keeps the session in the OS keychain. Machines (M2+)
present short-lived OIDC tokens checked against trust rules on `Project` or
`Landscape` (D-API-04); `set_by` is `<issuer>#<sub>`.

Role bindings on `Project` and `Landscape` map group claims to roles per
request. The public listener rejects state-changing requests without the
`Connect-Protocol-Version` header, which forces a CORS preflight on foreign
origins (unverified for connect-web).

## Each role includes the roles above it in its scope

| Scope | Role | May |
| --- | --- | --- |
| platform | `platform-admin` | create projects and every `Landscape`, assign their admins |
| project | `viewer` | read everything in the project |
| project | `developer` | edit `Application` and `PromotionConfig`, create promotions, sync applications |
| project | `release-manager` | set approval gates to `Satisfied` or `Rejected` |
| project | `project-admin` | manage credentials, role bindings and grants; override gates with a reason (ADR-0040) |
| landscape | `landscape-viewer` | read landscape state and every rollout target on it |
| landscape | `landscape-operator` | call `OfflineService`; a CTF covers every project |
| landscape | `landscape-admin` | create enrollment tokens, confirm or skip the enrollment code with an audited reason, set link templates and `staleAfter` |

A grant needs both sides: `project-admin` creates the `LandscapeGrant`, and
the landscape admin writes a `Boundary` for the project UID (D-TEN-01).

Every RPC declares its permission in the `konfidence.authz.v1.rule` method
option, for example `promotion.gate.set` at `SCOPE_PROJECT`. One interceptor
checks it against the scope of the resource name; a CI lint fails on any
public RPC without it.

## The API writes intent to CRDs and facts to rows

After authorization, the API validates with protovalidate and domain checks
(D-API-03). It writes intent to the CRD with optimistic concurrency, or facts
to Postgres rows with any follow-up job in the same transaction. The
`audit_log` row commits with the row write, or right after the API server
accepts the CRD write. A conflict returns `aborted`; the UI refetches and
retries once.

Intent lists read the informer cache. Promotions, rollout targets, releases,
statements and history read Postgres with cursor pagination (`page_size` 50,
at most 500). The UI refetches the visible view every 10 s.

## Status shows how we know, not only what we know

The UI never shows a bare "healthy" (D-UI-02). The API computes one
`Observation` per rollout target, so UI and CLI agree. It carries `status`
and `reason` from the status function in
[02](02-domain-model.md#rollout-status-is-a-function-of-slot-and-latest-statement),
plus `basis`, `observed_at` and `health` (`HEALTH_UNKNOWN` while dispatched).

| Basis | Meaning | Shown as |
| --- | --- | --- |
| `BASIS_DISPATCHED` | a delivery lists the slot; no statement has verified it | hollow chip, "dispatched at T" |
| `BASIS_ATTESTED` | a statement covers the slot | health colour plus "as of T" |
| `BASIS_LIVE` | `mode: Attached` and the newest statement's `last_seen_at` is younger than `staleAfter` | health colour, no age |

Isolated landscapes never reach `LIVE`; their statement age always shows.
Past `staleAfter` the chip dims and reads "stale". A `Blocked`, `Refused` or
`Failed` status overrides the colour. Deep links come only from
`Landscape.spec.linkTemplates` (D-UI-03).

## The UI is three screens in M1

The UI uses Svelte, Skeleton and `konfidence-design-system` (D-UI-01) and
calls only `konfidence.api.v1` through connect-web.

1. **M1 Fleet.** Mode, bridge version with the outdated flag, statement age,
   reporting level, target counts per basis.
2. **M1 Application.** Where-running matrix: landscapes as rows, cells with
   release label (`payments-api@42`) and observation chip.
3. **M1 Promotion.** Waves as columns, landscapes as cells, **Approve** and
   **Reject** through `SetGateState`, gated by role.

M2+ adds Release with diff, Landscape with enrollment and offline actions,
and Audit. In M1 the CLI covers enrollment and offline files.

## Path

1. **M1** Create the proto module, `buf` checks in CI, the authz option, the
   interceptor and its lint.
2. **M1** Implement `BridgeService` (`Enroll`, held `Sync`, `FetchBlob`) and
   `OfflineService` with the signed download route.
3. **M1** Implement OIDC login, `sessions`, `WhoAmI` and role resolution.
4. **M1** Implement the read services and the audited writes for `Project`,
   `GitCredential`, `Landscape` with enrollment, `LandscapeGrant`,
   `Application`, `PromotionConfig`, `CreatePromotion` and `SetGateState`.
5. **M1** Build Fleet, Application and Promotion screens.
6. **M2+** Machine trust rules, external gate writers, removal,
   `AuditService`, release diff and the remaining screens. Later: custom
   roles, opt-in four-eyes approval ([05](05-promotion-and-rollout.md)).
