# Sources, credentials and releases

The control plane turns git commits into releases. A release is its
build-input manifest, and the release ID is that manifest's digest. Names
come from [02](02-domain-model.md), job kinds from [01](01-control-plane.md),
bundle signing from [03](03-bundles-and-transfer.md), and promotion from
[05](05-promotion-and-rollout.md).

## One chain of jobs creates every release

Each job inserts the next one in the transaction that records its result
(D-TOPO-05). [01](01-control-plane.md#every-state-change-inserts-its-job-in-the-same-transaction)
defines the job kinds.

1. **`source.poll`** lists remote refs for one poll key (repo URL, ref,
   credential) and compares the head SHA with the one it saw last. A change
   inserts `release.build` for each application on that key.
2. **`release.build`** fetches each source at its SHA (depth 1), vendors
   chart dependencies and builds the manifest. If `releases` has no row with
   that release ID for the application, it inserts one with the next `seq`
   and state `Building`; an equal release ID creates nothing (D-REL-01). It
   runs the builders, assembles the OCM component
   `konfidence.cloud/p/<project>/<application>` at version `<seq>.0.0`,
   records `required_capabilities`, and inserts `bundle.sign`. A build error
   sets `Failed`.
3. **`bundle.sign`** re-hashes and signs the bundle
   ([03](03-bundles-and-transfer.md)), sets `bundle_digest` and `Ready`, and
   inserts `release.promote`.
4. **`release.promote`** starts a promotion for every `PromotionConfig` with
   `source: {application}` and `mode: Auto`.

## An application has one or more sources, each with its own credential

`Application.spec.sources[]` holds `name`, `url`, `ref` (`branch` or `tag`),
`credentialRef` and `builders`; `charts[]` names each chart's source and
`path` (D-SRC-06). Source names are DNS labels, unique within the
application. `credentialRef.project` names the owner of a granted credential.
M1 ships one source.

A new commit that leaves every chart path of a source unchanged keeps the
previous commit in the manifest (M2+). `release.build` compares git tree
hashes with the `trees` in the previous manifest. A source with builders
skips this check, because a builder sees the whole tree.

## Three credential types, one Secret each

`GitCredential` is project-scoped; `type` selects the keys its Secret holds.
Workers read key material at job time. It never appears in Postgres, River
job arguments, logs or bundles (D-SRC-03).

| `type` | Secret keys | Transport | Validity probe | Milestone |
| --- | --- | --- | --- | --- |
| `GitHubPAT` | `token` | HTTPS basic auth | `GET /repos/{owner}/{repo}` per granted repo | M1 |
| `SSHDeployKey` | `identity` | SSH | `ls-remote` | M2+ |
| `GitHubApp` | `privateKey`; `appID`, `installationID` in spec | HTTPS with minted token | mint a token | M2+ |

**SSH.** `spec.knownHosts` pins host keys and is required for `SSHDeployKey`
(D-SRC-04). It sits in spec because it is public and must be reviewable.
GitHub publishes its entries
([fingerprints](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)).
A host key mismatch fails the job.

**GitHub App.** The worker signs an RS256 JWT with `iss` set to the client ID
or app ID and `exp` at most 10 min ahead. It calls
`POST /app/installations/{installation_id}/access_tokens` with the job's
`repositories` and `permissions: {contents: read}`
([installation tokens](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app)).
Tokens last 1 h; the worker caches them per (credential, repo set) and drops
them 5 min early. Scoping by `repositories` enforces the grant, because one
installation often covers a whole organization.

## Grants scope a credential to projects and repositories

`spec.grants` lists `{project, repositories}` with `owner/repo` globs, the
owning project included. Only an admin of the owning project edits grants.
The API rejects an `Application` whose `credentialRef` has no matching
grant, and `release.build` re-checks it before it fetches. A revoked grant
stops fetches; existing releases stay valid.

## Expiry shows up as a condition before fetches fail

The credential check probes every `GitCredential` hourly and after each
Secret change, and writes its status:

- `Valid` turns `False` on a 401, a host key mismatch, or a suspended App
  installation, with the cause.
- `ExpiringSoon` turns `True` when a `GitHubPAT` expires within 14 days;
  `status.expiresAt` holds the date from the
  `github-authentication-token-expiration` response header. TODO(sources):
  verify that GitHub sends this header.

## Polling detects changes, webhooks only shorten the wait

`source.poll` runs once per poll key and interval, so applications on the
same repo and ref share one poll. The default interval is 2 min, the minimum
30 s, set by `spec.pollInterval`. [01](01-control-plane.md) defines the
ticker and the uniqueness.

For `GitHubApp` credentials (M2+), `POST /webhooks/github` validates
`X-Hub-Signature-256`
([validating deliveries](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries))
and inserts the matching `source.poll` jobs at once. The control plane
never takes a SHA from the payload, so a forged webhook causes at most an
extra poll (D-SRC-05).

Only workers reach git, and only github.com (D-SRC-01, D-SRC-02), plus the
registries that charts name as dependencies. Builders get no network.

## Builders turn a checked-out source into OCM resources

This page owns `builder/v1`, the contract the runtime team builds against
(D-PROD-03). The built-in `helm` builder runs in the worker (M1). Every other
builder is a container image registered in the control-plane configuration by
name, image digest and contract version (M2+). Applications name builders
only.

`release.build` runs an image builder as a Kubernetes Job with egress denied
and no ServiceAccount token:

| Path | Mode | Content |
| --- | --- | --- |
| `/in/request.json` | read-only | contract version, application, release ID, `seq`, source name, builder `config` |
| `/in/src/` | read-only | the source tree at its commit, no `.git` |
| `/in/vendor/` | read-only | vendored chart dependencies |
| `/out/result.json` | read-write | `contract`, `resources[]` (`name`, `type`, `version`, `path`, `labels`), `images[]` |

Each resource becomes an OCM local blob labelled with builder name and image
digest. Names are unique within the release, and images are pinned by digest.
A non-zero exit, a missing `result.json` or a run over 15 min sets the
release `Failed` and records the last 4 KiB of output as an event.
Only `helmChart` resources deploy (D-LAND-03); other types travel along. The
runtime team's vector builder wraps the `Stage` manifest from `vector.yaml`
in a chart (D-PROD-02).

The built-in Helm builder runs `helm dependency build` and `helm package`
per chart. It collects images from `helm template` with default values plus
the chart annotation `konfidence.cloud/images`. A failing `helm template`
fails the build, because a partial image list surfaces later as a missing
image on an isolated landscape.

## The release ID is the digest of the build-input manifest

The manifest is one JSON document in `releases.manifest`. The release ID is
`sha256:` plus the hex SHA-256 of its RFC 8785 canonical form (D-REL-01).

```json
{
  "format": "konfidence.cloud/release-manifest/v1",
  "sources": [{ "name": "app", "repo": "github.com/acme/payments-api",
                "commit": "9c1e…", "trees": { "deploy/chart": "4b82…" } }],
  "dependencies": [{ "source": "app", "chart": "deploy/chart", "name": "postgresql",
                     "version": "15.5.0", "digest": "sha256:aa12…" }],
  "application": {
    "charts": [{ "name": "api", "source": "app", "path": "deploy/chart",
                 "valuesFiles": ["deploy/values.yaml"], "dependsOn": [],
                 "crds": "CreateReplace", "overridable": ["/replicaCount", "/ingress/host"] }],
    "secretRequirements": [{ "name": "db", "keys": ["username", "password"] }],
    "builders": { "app": [] }
  }
}
```

- `format` versions the rules; a new format leaves old IDs valid.
- `sources` sort by `name`, `dependencies` by (`source`, `chart`, `name`),
  byte-wise.
- Repo URLs normalize to `host/owner/repo`: lowercase host, no scheme, user
  or `.git`, so SSH and HTTPS URLs hash the same.
- Paths are repo-relative with `/`, no leading `./` or trailing `/`; the root
  is `.`. Commits are 40-character lowercase hex.
- A dependency digest is the SHA-256 of the downloaded archive, or the
  manifest digest for `oci://`. Only the archive digest catches a re-pushed
  chart version; the commit already covers `Chart.lock`.
- `application` is the deploy-relevant `Application` spec. `pollInterval`,
  `credentialRef` and `ref` stay outside.

Equal release ID means equal deployable inputs. A changed builder config or
CRD policy makes a new release without a commit. Any RFC 8785 implementation
plus `sha256sum` recomputes an ID offline.

## Three identities answer three questions

| Identity | Example | Answers | Stable across rebuilds |
| --- | --- | --- | --- |
| release ID | `sha256:e3b0…` | which inputs | yes |
| bundle digest | `sha256:5d41…` | which bytes ship | no, depends on builder versions |
| release label | `payments-api@42` | what humans type | yes, one per release ID |

A release pins its bundle digest after the first successful build; a later
builder upgrade never rebuilds it (D-REL-02). Slots, statements and audit
records store release ID and bundle digest; the label is for display.

## Release queries read manifests, slots and statements

[08](08-api-and-ui.md) owns the RPC shapes (D-REL-03).

- **Diff** (M2+) is the diff of two manifests. Per source it adds the commits
  from `GET /repos/{owner}/{repo}/compare/{base}...{head}`, at most 250
  without pagination
  ([compare](https://docs.github.com/en/rest/commits/commits#compare-two-commits));
  `SSHDeployKey` shows the two commits only.
- **Placement:** the landscapes whose `desired` slot points at the release,
  with derived status, values digest and policy version from the latest
  statement (D-POL-06).
- **Path:** the promotions and waves the release passed, from `promotions`,
  `rollout_targets` and `gate_states`.

## Path

1. **M1** Manifest builder and hasher as a pure Go package with golden-file
   tests, including equal IDs for SSH and HTTPS URLs.
2. **M1** `GitHubPAT` with grants and the credential check.
3. **M1** `source.poll` and `release.build` with the Helm builder.
4. **M1** Placement and path queries.
5. **M2+** Multiple sources, the unchanged-path check, `SSHDeployKey`.
6. **M2+** `GitHubApp`, webhooks, the diff query, image builders for
   `builder/v1` with the vector builder first.

Later: semver-range refs, a shared fetch cache, GitHub Enterprise Server.
TODO(sources): verify GitHub's git-protocol limits against the 2 min default
at a few thousand applications.
