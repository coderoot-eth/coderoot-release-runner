# Release Runner: Specification

**Status:** review draft. Final after the checks in §17 pass and the gate owner signs.

## 1. Purpose and scope
The runner is the only identity allowed to publish a customer's npm package. It publishes exactly the bytes a human approved, under the approved name and version, and nothing else.

In scope: dispatch, runner endpoints on the gate, record and artifact verification, publish, state reporting, customer isolation, repo and workflow hardening.
Out of scope: approval UI, attestation writing, revoke, evidence bundle, non-npm registries.

## 2. Components
- **Gate** (`secure.coderoot.app`, gate service): signed record, artifact bytes in private object storage, release state, dispatch through a dedicated GitHub App, runner endpoints.
- **Runner repo** `coderoot-eth/coderoot-release-runner`: public, one workflow `release.yml`, one GitHub environment per customer, pinned signer list.
- **npm**: one trust configuration per package, bound to the runner repo + `release.yml` + that customer's environment.

## 3. Release flow
1. Human approves. Gate writes the role-key-signed record entry. `publish.state = dispatched`. Artifact bytes stay in storage until publication.
2. Gate dispatches `release.yml` on `main` through its GitHub App, with **one input: `release_id`**.
3. Job `resolve` (`contents: read`, `id-token: write`, **no environment**):
   - validate `release_id` format (`rel_` + 26-char ULID);
   - get a GitHub OIDC token for audience `secure.coderoot.app`; claim the release and fetch the record;
   - verify the role-key signature; signer must be in the pinned list for that org and role;
   - refuse unless state is `approved` and `publish.state` is not `published`;
   - output the environment name `org-<org_id>` from the record, never from dispatch input.
   The OIDC token in this job cannot publish: npm trust requires the customer environment, which this job does not have.
4. Job `publish` (`environment: ${{ needs.resolve.outputs.env }}`, `contents: read`, `id-token: write`):
   - report `running`;
   - re-fetch and re-verify the record;
   - download the artifact; SHA-256 and SHA-512 must equal the record;
   - read `name` and `version` from the tarball's `package.json`; must equal `package_identity`;
   - check npm: package exists; version not already published, or already published with the same integrity;
   - re-read release state; stop if `revoked`;
   - `npm publish <tarball> --provenance --access public --tag <tag>` (tag rule: [failure-modes.md](failure-modes.md) G10), exact pinned npm;
   - read the registry's `dist.integrity`; must equal the record's SHA-512;
   - report `published` with `runner_run_id`, `registry_url`, `registry_integrity`, `provenance_uri`, `published_at`.
5. Any failure: report `failed` with a reason code; nothing published; no bundle.

## 4. State and failure rules
- States: `none` → `dispatched` → `running` → `published` | `failed`.
- Gate ignores any runner update for a revoked release; a late update never undoes `revoked`.
- Gate marks `failed` / `runner_timeout` if no claim arrives within 15 minutes of dispatch, after checking npm ([failure-modes.md](failure-modes.md) G1).
- Re-dispatch after `failed`: same `release_id`, only if the version is not on npm or matches.
- Concurrency: `concurrency: publish-<package>`, no cancel-in-progress.
- Never wait on the chain.

Reason codes: `record_invalid`, `signer_unknown`, `record_state`, `artifact_missing`, `hash_mismatch`, `identity_mismatch`, `version_exists`, `package_missing`, `revoked`, `npm_error`, `integrity_mismatch`, `runner_timeout`.

## 5. Runner endpoints on the gate 

Full contract with request/response examples and error codes: [endpoints.md](endpoints.md). Summary below.

### 5.1 Dispatch (gate → GitHub)
- GitHub App installed on `coderoot-eth/coderoot-release-runner` only, permission `actions: write` only. Its key lives in the gate's key vault.
- `workflow_dispatch` of `release.yml` on `ref: main` with `inputs: { release_id }`.
- Gate stores a dispatch record: `release_id`, `dispatched_at`, expected `org_id` and environment, `claimed_run_id: null`.
- A dispatch alone cannot publish anything: the runner verifies the signed record itself.

### 5.2 Runner authentication (all runner → gate calls)
`Authorization: Bearer <GitHub OIDC token>`, audience `secure.coderoot.app`, verified against GitHub's JWKS. Required claims:

| Claim | Must equal |
|---|---|
| `iss` | `https://token.actions.githubusercontent.com` |
| `repository` | `coderoot-eth/coderoot-release-runner` |
| `job_workflow_ref` | `coderoot-eth/coderoot-release-runner/.github/workflows/release.yml@refs/heads/main` |
| `ref` | `refs/heads/main` |
| `event_name` | `workflow_dispatch` |
| `run_id` | the run bound to this release |
| `environment` | `org-<org_id>` of the release, for §5.5 and §5.6 (absent for `resolve`) |

Checking `ref` and `job_workflow_ref` closes the branch bypass on the gate side as well.

### 5.3 Claim: `POST /v1/runner/releases/{release_id}/claim`
First call from a run. Binds `run_id` to the release if the release is `dispatched` and unclaimed, or already bound to the same `run_id`. Any other run gets `409 release_claimed`. Late claims after `runner_timeout` get `409`.

### 5.4 Record: `GET /v1/runner/releases/{release_id}`
Returns the signed envelope (RFC 8785 payload + signature), `org_id`, `package_identity`, both hashes, release state, publish state. Bound run only.

### 5.5 Artifact: `GET /v1/runner/releases/{release_id}/artifact`
Streams the bytes from storage. Bound run only, `environment` claim required and matching. Only while state is `approved` and publish is not `published`.

### 5.6 State: `POST /v1/runner/releases/{release_id}/state`
Body: `{ state: running|published|failed, reason?, runner_run_id, registry_url?, registry_integrity?, provenance_uri?, published_at? }`. Bound run only.
Gate rules: ignore if revoked; for `published`, re-read npm and require `dist.integrity` = record SHA-512 before storing.

### 5.7 Signing keys: `GET /v1/runner/signing-keys?org_id=`
Lists authorised signer addresses with role, per org. Used as a cross-check only, see §6.

Error format: the gate API error shape. Codes: `401 runner_token_invalid`, `403 runner_claim_mismatch`, `409 release_claimed`, `409 release_not_publishable`, `404 release_not_found`.

## 6. Signer verification
Gate design: the runner fetches the authorised signers from the Worker.
Problem: the same Worker dispatches the runner and serves the record. If it were compromised, it could serve a forged record and a matching forged key list.
Rule:
- The runner repo holds `signers.json`: per `org_id`, the role-key addresses allowed to approve. Changes only through a reviewed PR (two approvals, CODEOWNERS).
- The runner accepts a record only if the signer is in `signers.json` **and** in the Worker list. Any mismatch fails `signer_unknown`.
- Role-key addresses are already public as EAS attesters, so publishing them adds no exposure.
- Rotation: new key added to both before use; old key removed after.

## 7. Customer isolation
- One GitHub environment per customer, named **`org-<org_id>`**. Permanent, like the repo name.
- Every environment: deployment branches `main` only, no secrets, no variables.
- Each customer's package trusts only its own environment.
- With more than one customer, the environment is the isolation boundary.

## 8. Repo and workflow hardening
Repo: public, branch protection on `main` (two reviews, CODEOWNERS on `.github/workflows/` and `signers.json`, no force push), minimal admins and writers.
Workflow: dispatch trigger only; per-job permissions as in §3; actions pinned to commit SHAs; cache off; npm pinned to an exact version (proposed 12.2.0); no `setup-node` `registry-url`; no install, build or pack in either job; inputs only through `env`; job timeouts; GitHub-hosted runners only.

## 9. Provenance
On. The statement names `coderoot-eth/coderoot-release-runner` and `release.yml` as the build source. The customer's source repo and commit stay in the record and the bundle.

## 10. Stage-only option (decision: gate owner)
npm can restrict trust to staging (`--allow-stage-publish`); the runner stages and a package maintainer approves with 2FA.
Gain: a hijacked runner can stage but not ship. Cost: a human step per release by the customer's maintainer.
Proposal: per-customer choice at onboarding, default direct publish; exactly one permission set.

## 11. Hosting
GitHub-hosted runners: fresh VM per job, nothing persists. Self-hosted only if a customer objects to bytes passing through GitHub; then ephemeral and single-use. Not in v1.

## 12. Customer onboarding (summary; detail in [onboarding.md](onboarding.md))
1. New package only: maintainer publishes a placeholder by hand with 2FA, non-`latest` tag, deprecated ([first-publish.md](first-publish.md)).
2. Check existing trust; npm allows one configuration per package, so the customer's own CI publisher is replaced.
3. `npm trust github <pkg> --repository coderoot-eth/coderoot-release-runner --file release.yml --environment org-<org_id>` with exactly one of `--allow-publish` / `--allow-stage-publish`.
4. Publishing access: "Require 2FA and disallow tokens".
5. Revoke old npm tokens; remove npm credentials from the customer's CI.
6. Review maintainers; all keep 2FA.
7. Explain the provenance change.
8. First release through the gate.

CodeRoot side, before step 3: create environment `org-<org_id>` (main only), add the org's role-key addresses to `signers.json` by reviewed PR.

## 13. First publish of a new package (summary; detail in [first-publish.md](first-publish.md))
Trusted publishing needs the package to exist. The customer's maintainer creates it with a README-only placeholder; the agent never touches npm. The runner refuses to create packages (`package_missing`) and refuses the placeholder version.

## 14. Go-live failure modes (summary; detail in [failure-modes.md](failure-modes.md))
| Case | Rule |
|---|---|
| Version already on npm, same bytes | Report `published`, no second publish |
| Version already on npm, different bytes | `integrity_mismatch`, alert: bypass signal |
| Two runs for one release | Claim endpoint lets only one through |
| Re-dispatch after failure | Same release only, after the npm check |
| Revoke during publish | Re-check before publish; never undo `revoked`; npm version stays if already out |
| npm or GitHub outage | Fail closed; re-dispatch starts with the npm check |
| Run never starts or stalls | Gate timeout → `runner_timeout` after npm check |
| Prerelease / lower version | Fixed tag rule, not chosen by the submitter |

## 15. Tests (DoD)
| Requirement | Test |
|---|---|
| Approved version on npm, hash matches | runner publish; registry `dist.integrity` = record SHA-512 |
| Tampering refused | change one byte in storage after approval → `hash_mismatch` |
| Unapproved release cannot trigger | `release_id` with no signed record → `record_invalid` |
| Agent holds no npm token | Action credential guard (exit 4); no npm secret in runner repo or environments |
| Forged signer | signer not in `signers.json` → `signer_unknown` |
| Identity mismatch | tarball `package.json` version differs → `identity_mismatch` |
| Customer isolation | run in `org-a` cannot publish an `org-b` package |
| Branch bypass | non-main branch run refused by GitHub and by the gate |
| Revoke | revoke after dispatch → `revoked` |
| Old token | publish with an old npm token refused |
| Second run | `409 release_claimed` |
Attack cases for attack test suite: [attack-tests.md](attack-tests.md).

## 16. Build checklist
Who builds is decided after this spec. Split by side:

**Runner side (`coderoot-eth/coderoot-release-runner`)**
- [ ] Branch protection on `main`: two reviews, CODEOWNERS on `.github/workflows/` and `signers.json`, no force push
- [ ] `release.yml`: `resolve` + `publish` jobs as §3, permissions per job, pinned actions, exact npm, no cache, no install/build/pack
- [ ] `signers.json` per org, reviewed
- [ ] Record verification: RFC 8785 payload, SHA-256, secp256k1 recovery, act only on verified payload
- [ ] Artifact checks: both hashes, `package.json` identity, npm existence/version check, revoke re-check
- [ ] State reporting and reason codes
- [ ] Tests from §15 that run without the gate (fixtures + throwaway packages)

**Gate side**
- [ ] GitHub App: runner repo only, `actions: write` only, key in the key vault
- [ ] Dispatch + dispatch record
- [ ] OIDC verification with the claim table ([endpoints.md](endpoints.md) §1)
- [ ] Endpoints: claim, record, artifact, state, signing-keys
- [ ] State rules: transitions, revoked ignores updates, npm integrity re-check on `published`
- [ ] Timeout job and alerts to the operations alert channel

**Per customer**
- [ ] Environment `org-<org_id>` (main only, no secrets)
- [ ] Onboarding §12

## 17. To confirm before final
| # | Assumption | How | Waiting on |
|---|---|---|---|
| V1 | npm refuses a run from the wrong environment | npm check run 2 | environments + test packages |
| V2 | A non-main branch run cannot publish | npm check runs 3–4 | same |
| V3 | "Disallow tokens" refuses an old token while trusted publishing works | npm check run 5 | same (earlier spike evidence points yes) |
| V4 | Stage-only refuses direct publish; staged + 2FA approve works; provenance on staged versions | npm check runs 6–7 | same |
| V5 | Record format matches gate owner's implementation | test vectors | package owner |
| V6 | Endpoints §5 and pinned signers §6 accepted | review | gate owner |

## 18. Sign-off
Countersigned by: ____________ Date: ________
