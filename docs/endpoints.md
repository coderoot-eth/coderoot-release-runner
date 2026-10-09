# Runner endpoints on `secure.coderoot.app`

Draft for review. Same conventions as the gate API: JSON, UTF-8, tagged lowercase hashes, the gate API error shape, ULID ids.
Base: `https://secure.coderoot.app/v1/runner`. All examples use `release_id = rel_01J7B3C9V2N4P6R8T0W1X3Y5ZA`.

## 0. Dispatch (gate → GitHub, not an endpoint of ours)
After the signed record exists, the gate calls GitHub with its App installation token
(App installed on `coderoot-eth/coderoot-release-runner` only, permission `actions: write` only):

```
POST https://api.github.com/repos/coderoot-eth/coderoot-release-runner/actions/workflows/release.yml/dispatches
{ "ref": "main", "inputs": { "release_id": "rel_01J7B3C9V2N4P6R8T0W1X3Y5ZA" } }
```

Gate then stores the dispatch record. The environment name is the `org_id`, verbatim:

```json
{ "release_id": "rel_01J7B3C9V2N4P6R8T0W1X3Y5ZA", "org_id": "org_example",
  "environment": "org_example", "attempt": 1, "dispatched_at": "2026-10-09T10:00:00Z",
  "claimed_run_id": null, "claimed_run_attempt": null, "previous_run_ids": [],
  "claim_deadline": "2026-10-09T10:15:00Z" }
```

## 1. Authentication (every endpoint below)
`Authorization: Bearer <GitHub Actions OIDC token>` requested with `audience=secure.coderoot.app`. The runner requests a fresh token for every call; GitHub's tokens are short-lived.
The gate verifies the JWT signature against `https://token.actions.githubusercontent.com/.well-known/jwks`, `exp`/`nbf` with ≤ 60 s skew, then the claims:

| Claim | Rule | Endpoints |
|---|---|---|
| `iss` | `https://token.actions.githubusercontent.com` | all |
| `aud` | `secure.coderoot.app` | all |
| `repository` | `coderoot-eth/coderoot-release-runner` | all |
| `repository_id`, `repository_owner_id` | the values recorded when the repo was created (survive renames) | all |
| `workflow_ref` | `coderoot-eth/coderoot-release-runner/.github/workflows/release.yml@refs/heads/main` | all |
| `job_workflow_ref` | equals `workflow_ref` (a top-level workflow carries both, with the same value) | all |
| `ref` | `refs/heads/main` | all |
| `event_name` | `workflow_dispatch` | all |
| `actor_id` | equals the gate GitHub App's bot account id | claim |
| `run_id`, `run_attempt` | equal `claimed_run_id`, `claimed_run_attempt` (after §2) | all except claim |
| `environment` | equals the release's `org_id` | artifact; state for `running` and `published` |

Measured on a token issued to a job running a top-level workflow in an environment: it carries both `workflow_ref` and `job_workflow_ref`, with the same value, and `run_attempt` and `actor_id` are strings (`"1"`). The gate compares them as strings.

Failures: `401 runner_token_invalid` (signature, expiry, issuer, audience) or `403 runner_claim_mismatch` (any claim rule), with `detail.claim` naming the first failed claim. Never echo the token.

## 2. Claim
`POST /releases/{release_id}/claim` — first call of every run. Body: none.

Rules: release must be `approved` with `publish.state = dispatched`, before `claim_deadline`, and the token's `run_id` must not be in `previous_run_ids`.
Binds `run_id` and `run_attempt` if unclaimed; idempotent for the same pair. A re-run of the bound run keeps `run_id` but increments `run_attempt`, so it gets `409 release_claimed`, as does a run from an earlier gate attempt. Retries go through the gate reset (§7) only.

`200`
```json
{ "release_id": "rel_01J7B3C9V2N4P6R8T0W1X3Y5ZA", "run_id": "18234567890", "run_attempt": 1,
  "environment": "org_example", "attempt": 1, "claimed_at": "2026-10-09T10:00:41Z" }
```
Errors: `404 release_not_found`; `409 release_claimed` (`detail.run_id_claimed` is not returned, only that it is taken); `409 release_not_publishable` (`detail.state`, `detail.publish_state`); `410 claim_expired`.

## 3. Record
`GET /releases/{release_id}`

`200`
```json
{
  "release_id": "rel_01J7B3C9V2N4P6R8T0W1X3Y5ZA",
  "org_id": "org_example",
  "state": "approved",
  "publish": { "state": "dispatched", "runner_run_id": "18234567890" },
  "record": {
    "envelope": "<base64 of the signed envelope's exact RFC 8785 UTF-8 bytes>",
    "signature": "0x<65-byte secp256k1 r||s||v>"
  }
}
```

The signed envelope, decoded:
```json
{ "format": "coderoot-signed-release-record-v1",
  "payload": {
    "approved_at_ms": 1791028800000,
    "approver_role": "Admin",
    "artifact": { "name": "@example/package", "registry": "npm",
                  "sha256": "sha256:<64 hex>", "sha512": "sha512:<128 hex>",
                  "size_bytes": 279, "version": "0.1.0" },
    "commitment_key_version": 1,
    "entry_id": "rec_…",
    "format": "coderoot-release-record-v1",
    "policy": { "grade": "human-required", "sha256": null },
    "release_id": "rel_01J7B3C9V2N4P6R8T0W1X3Y5ZA",
    "review_context_commitment": "0x<64 hex>",
    "role_key_address": "0x8a1f3c5e7b9d0246a8c0e2f4b6d8a0c2e4f6b8d0",
    "source_commit": "<40 hex>",
    "submission_id": "sub_…" },
  "record_commitment": "0x<64 hex>" }
```

The runner verifies, in order, and fails `record_invalid` on any step unless noted:
1. Decode `envelope`. The bytes must be RFC 8785 canonical: re-serialising the parsed JSON gives the same bytes.
2. `digest = SHA-256(envelope bytes)`. Recover the address from `signature` over the raw 32-byte digest (no Ethereum message prefix). It must equal `payload.role_key_address`.
3. `format` is `coderoot-signed-release-record-v1` and `payload.format` is `coderoot-release-record-v1`.
4. `payload.release_id` equals the `release_id` in the path.
5. `payload.artifact.registry` is `npm`; `artifact.version` is an exact semver version, never a range or tag.
6. Signer check ([spec.md](spec.md) §6 and §6 below), using `role_key_address`, `approver_role` and `approved_at_ms`. Failure: `signer_unknown`.

Everything the runner acts on comes from the verified payload: `artifact.name`, `artifact.version`, `artifact.sha256`, `artifact.sha512`, `artifact.size_bytes`, `approved_at_ms`. The unsigned wrapper fields exist for logging only and must match.

`record_commitment` is an HMAC-SHA256 of the payload under the org's commitment key. The runner cannot recompute it, since the key is secret; it is covered by the signature like the rest of the envelope.

The payload carries no `org_id`. The runner needs a signed org to choose the environment and the signer list ([spec.md](spec.md) §17, V9).

Errors: `404 release_not_found`; `403 runner_claim_mismatch`.

## 4. Artifact
`GET /releases/{release_id}/artifact`

`200` `Content-Type: application/octet-stream`, body = exact tarball bytes from storage.
Headers: `X-CodeRoot-SHA256`, `X-CodeRoot-SHA512` (informational; the runner hashes the body itself), `Content-Length`.
Allowed only while `state = approved` and `publish.state ∈ {dispatched, running}`.

Errors: `403 runner_claim_mismatch` (wrong environment); `409 release_not_publishable`; `410 artifact_gone`.

## 5. State
`POST /releases/{release_id}/state`

Running:
```json
{ "state": "running", "runner_run_id": "18234567890" }
```
Published:
```json
{ "state": "published", "runner_run_id": "18234567890",
  "registry_url": "https://registry.npmjs.org/@example/package/0.1.0",
  "registry_integrity": "sha512-<base64>",
  "provenance_uri": "https://search.sigstore.dev/?logIndex=<n>",
  "published_at": "2026-10-09T10:01:12Z" }
```
Failed:
```json
{ "state": "failed", "runner_run_id": "18234567890", "reason": "hash_mismatch",
  "detail": { "expected": "sha256:<hex>", "actual": "sha256:<hex>" } }
```

Gate rules:
- `failed` is accepted from the bound run with or without an `environment` claim, so the `resolve` job can report its refusals (`record_invalid`, `signer_unknown`, `record_state`). If the claim is present it must equal the release's `org_id`.
- Transitions allowed: `dispatched → running → published | failed`; `dispatched → failed`. Anything else `409 invalid_transition`.
- `failed → dispatched` happens only through the gate reset (§7), never through this endpoint.
- `failed` (`runner_timeout`) → `published` is accepted from the run that held the claim, if the gate's npm read below matches. A slow run that published after the timeout is recorded, not lost.
- If the release is `revoked`: `200` with `{ "ignored": true, "state": "revoked" }`; nothing changes.
- For `published`: the gate reads the version from npm itself.
  - Present, `dist.integrity` = record SHA-512 (and = `registry_integrity` when sent): store `published`.
  - Present with other integrity: store `failed` / `integrity_mismatch` and alert.
  - Not visible yet: answer `202` with `{ "publish_state": "running", "registry_pending": true }`, keep `running` and re-read with backoff. If still not visible 30 minutes after the claim, the timeout rule (§8) applies.
- Repeating the same body is harmless (`200`, same result).

`200`
```json
{ "release_id": "rel_01J7B3C9V2N4P6R8T0W1X3Y5ZA", "publish_state": "published", "accepted": true }
```

## 6. Signing keys
`GET /signing-keys?org_id=org_example`

`200`
```json
{ "org_id": "org_example",
  "signers": [ { "address": "0x8a1f3c5e7b9d0246a8c0e2f4b6d8a0c2e4f6b8d0", "role": "Admin", "valid_from": "2026-09-22T00:00:00Z", "valid_to": null } ],
  "as_of": "2026-10-09T10:00:41Z" }
```

`signers.json` in the runner repo uses the same entry shape:
```json
{ "schema": 1,
  "orgs": {
    "org_example": [
      { "address": "0x8a1f3c5e7b9d0246a8c0e2f4b6d8a0c2e4f6b8d0", "role": "Admin",
        "valid_from": "2026-09-22T00:00:00Z", "valid_to": null } ] } }
```
Addresses are lowercase hex. The window is `valid_from` ≤ t < `valid_to`; `valid_to: null` means open.

`approved_at_ms` is the approval time inside the signed payload, in milliseconds since the epoch, read only after the signature verifies. A payload without it fails `record_invalid`.

The runner accepts a record only if its signer appears here **and** in `signers.json` for the record's `org_id`, with the same role, that role may approve, and `approved_at_ms` lies inside the window in both. Otherwise: `failed` / `signer_unknown`.

## 7. Reset (gate-internal, not a runner endpoint)
The only way to retry a failed release. Allowed only if the release is still `approved` and the failure reason is `npm_error`, `runner_timeout`, `artifact_missing` or `package_missing`. Every other reason is final for the `release_id`; publishing needs a new submission and a new approval.

1. Read npm for the version. Present with `dist.integrity` = record SHA-512 → `published` (failure-modes G1). Present with other integrity → stays `failed` / `integrity_mismatch`, alert. Absent → continue.
2. For `artifact_missing`: the bytes must be in storage again, otherwise stop.
3. Update the dispatch record: append `claimed_run_id` (if any) to `previous_run_ids`, set `claimed_run_id` and `claimed_run_attempt` to `null`, `attempt + 1`, new `dispatched_at` and `claim_deadline`; set `publish.state = dispatched`.
4. Dispatch again (§0), subject to the per-package queue ([spec.md](spec.md) §4).

```json
{ "release_id": "rel_01J7B3C9V2N4P6R8T0W1X3Y5ZA", "org_id": "org_example",
  "environment": "org_example", "attempt": 2, "dispatched_at": "2026-10-09T11:00:00Z",
  "claimed_run_id": null, "claimed_run_attempt": null, "previous_run_ids": ["18234567890"],
  "claim_deadline": "2026-10-09T11:15:00Z" }
```
A run that never claimed in the earlier attempt may still claim the new one; it verifies the same signed record, and the claim lets only one run through.

## 8. Timeout (gate-internal)
If no claim by `claim_deadline` (dispatch + 15 min) or no terminal state within 30 min of claim, the gate reads npm: version present with matching integrity → `published`; present with other integrity → `failed` / `integrity_mismatch`; absent → `failed` / `runner_timeout`. Each `failed` alerts.

## 9. Error codes
| HTTP | id | When |
|---|---|---|
| 401 | `runner_token_invalid` | bad signature, expired, wrong issuer or audience |
| 403 | `runner_claim_mismatch` | any claim rule in §1, including a run not dispatched by the gate's App |
| 404 | `release_not_found` | unknown `release_id` (also for other orgs) |
| 409 | `release_claimed` | another run holds the release, a re-run of the bound run (`run_attempt`), or a run from an earlier gate attempt |
| 409 | `release_not_publishable` | state not `approved`, or already published/failed/revoked |
| 409 | `invalid_transition` | state change not allowed |
| 410 | `claim_expired` | claim after deadline |
| 410 | `artifact_gone` | bytes no longer stored |

## 10. Rate and size
One run per release at a time (claim). Artifact ≤ 64 MiB. Endpoints are not exposed to org sessions or `crs_` tokens; runner identity only.
