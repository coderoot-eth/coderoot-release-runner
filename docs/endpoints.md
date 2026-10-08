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

Gate then stores the dispatch record:

```json
{ "release_id": "rel_01J7B3C9V2N4P6R8T0W1X3Y5ZA", "org_id": "org_example",
  "environment": "org-example", "dispatched_at": "2026-10-09T10:00:00Z",
  "claimed_run_id": null, "claim_deadline": "2026-10-09T10:15:00Z" }
```

## 1. Authentication (every endpoint below)
`Authorization: Bearer <GitHub Actions OIDC token>` requested with `audience=secure.coderoot.app`.
The gate verifies the JWT signature against `https://token.actions.githubusercontent.com/.well-known/jwks`, `exp`/`nbf` with ≤ 60 s skew, then the claims:

| Claim | Rule | Endpoints |
|---|---|---|
| `iss` | `https://token.actions.githubusercontent.com` | all |
| `aud` | `secure.coderoot.app` | all |
| `repository` | `coderoot-eth/coderoot-release-runner` | all |
| `repository_id`, `repository_owner_id` | the values recorded when the repo was created (survive renames) | all |
| `job_workflow_ref` | `coderoot-eth/coderoot-release-runner/.github/workflows/release.yml@refs/heads/main` | all |
| `ref` | `refs/heads/main` | all |
| `event_name` | `workflow_dispatch` | all |
| `run_id` | equals `claimed_run_id` (after §2) | all except claim |
| `environment` | equals the release's `org-<org_id>` | artifact, state |

Failures: `401 runner_token_invalid` (signature, expiry, issuer, audience) or `403 runner_claim_mismatch` (any claim rule), with `detail.claim` naming the first failed claim. Never echo the token.

## 2. Claim
`POST /releases/{release_id}/claim` — first call of every run. Body: none.

Rules: release must be `approved` with `publish.state = dispatched`, before `claim_deadline`.
Binds `run_id` if unclaimed; idempotent for the same `run_id`.

`200`
```json
{ "release_id": "rel_01J7B3C9V2N4P6R8T0W1X3Y5ZA", "run_id": "18234567890",
  "environment": "org-example", "claimed_at": "2026-10-09T10:00:41Z" }
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
  "package_identity": "npm:@example/package@0.1.0",
  "hashes": ["sha256:<64 hex>", "sha512:<128 hex>"],
  "record": {
    "payload": "<RFC 8785 canonical JSON, as signed, base64>",
    "signature": "0x<65-byte secp256k1 r||s||v>",
    "signer": "0x8a1f3c5e7b9d0246a8c0e2f4b6d8a0c2e4f6b8d0",
    "signer_role": "Admin",
    "digest_alg": "sha256"
  }
}
```
The runner recomputes SHA-256 over the decoded `payload`, recovers the signer from `signature`, requires it to equal `signer`, and checks `signer` against `signers.json` ([spec.md](spec.md) §6) and §6 below. Fields the runner acts on (`org_id`, `package_identity`, `hashes`) are read **from the verified payload**, not from the unsigned wrapper; the wrapper copies exist for logging only and must match.

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
- Transitions allowed: `dispatched → running → published | failed`; `dispatched → failed`. Anything else `409 invalid_transition`.
- If the release is `revoked`: `200` with `{ "ignored": true, "state": "revoked" }`; nothing changes.
- For `published`: the gate fetches the npm packument itself and requires `dist.integrity` = `registry_integrity` = record SHA-512; otherwise it stores `failed` / `integrity_mismatch` and alerts.
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
The runner accepts a record only if its signer appears here **and** in the repo's `signers.json` with a role allowed to approve, and `signed_at` falls inside `valid_from`/`valid_to`. Mismatch: `failed` / `signer_unknown`.

## 7. Timeout (gate-internal)
If no claim by `claim_deadline` (dispatch + 15 min) or no terminal state within 30 min of claim, the gate reads npm: version present with matching integrity → `published`; otherwise `failed` / `runner_timeout`, and alerts.

## 8. Error codes
| HTTP | id | When |
|---|---|---|
| 401 | `runner_token_invalid` | bad signature, expired, wrong issuer or audience |
| 403 | `runner_claim_mismatch` | any claim rule in §1 |
| 404 | `release_not_found` | unknown `release_id` (also for other orgs) |
| 409 | `release_claimed` | another run holds the release |
| 409 | `release_not_publishable` | state not `approved`, or already published/failed/revoked |
| 409 | `invalid_transition` | state change not allowed |
| 410 | `claim_expired` | claim after deadline |
| 410 | `artifact_gone` | bytes no longer stored |

## 9. Rate and size
One run per release at a time (claim). Artifact ≤ 64 MiB. Endpoints are not exposed to org sessions or `crs_` tokens; runner identity only.
