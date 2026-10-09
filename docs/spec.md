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
   - get a GitHub OIDC token for audience `secure.coderoot.app`; claim the release and fetch the record. If the claim is refused, the run exits without reporting: it is not bound to the release, and the release state is unchanged;
   - verify the signed record ([endpoints.md](endpoints.md) §3), else `record_invalid`; the signer must be in both signer lists for that org and role (§6), else `signer_unknown`;
   - refuse `record_state` unless state is `approved` and `publish.state` is not `published`;
   - output the environment name, equal to the verified record's org (e.g. `org_example`; how the signed record carries the org is V9), never from dispatch input.
   The OIDC token in this job cannot publish: npm trust requires the customer environment, which this job does not have.
4. Job `publish` (`environment: ${{ needs.resolve.outputs.env }}`, `contents: read`, `id-token: write`):
   - report `running`;
   - re-fetch and re-verify the record;
   - download the artifact; SHA-256 and SHA-512 must equal the record;
   - the tarball's size must equal `artifact.size_bytes` (`hash_mismatch`);
   - read `name` and `version` from the tarball's `package.json`; must equal the record's `artifact.name` and `artifact.version` (`identity_mismatch`);
   - refuse `publish_config` if the tarball's `package.json` has a `publishConfig` key other than `access`, `tag` or `provenance`. npm applies every `publishConfig` key not given on the command line, so a key such as `registry` would let the approved bytes redirect the publish;
   - check npm: package exists (`package_missing`); if the version is already published with the record's SHA-512, skip to the `published` report without publishing (G1); with other bytes, `integrity_mismatch` (G2);
   - re-read release state; stop if `revoked`;
   - `npm publish <tarball> --registry https://registry.npmjs.org/ --provenance --access public --tag <tag> --ignore-scripts` (tag rule: [failure-modes.md](failure-modes.md) G10), exact pinned npm;
   - read the registry's `dist.integrity` with backoff for up to 5 minutes, since a new version can take time to become visible: different from the record's SHA-512 → `integrity_mismatch`; still not visible → report `published` anyway and let the gate's own read decide (G16);
   - report `published` with `runner_run_id`, `registry_url`, `registry_integrity`, `provenance_uri`, `published_at`.
5. Any failure: report `failed` with a reason code; nothing published; no bundle.

## 4. State and failure rules
- States: `none` → `dispatched` → `running` → `published` | `failed`. `failed` → `dispatched` only by gate reset. `failed` (`runner_timeout`) → `published` only when the gate's own npm read shows the record's SHA-512.
- Gate ignores any runner update for a revoked release; a late update never undoes `revoked`.
- Gate marks `failed` / `runner_timeout` if no claim arrives within 15 minutes of dispatch, or no terminal state within 30 minutes of the claim, after checking npm ([failure-modes.md](failure-modes.md) G1).
- Retry after `failed`: gate reset only ([endpoints.md](endpoints.md) §7), never a re-run of a GitHub run: the claim binds `run_id` and `run_attempt`, and a re-run increments `run_attempt`. Retryable reasons: `npm_error`, `runner_timeout`, `artifact_missing`, `package_missing`. The gate checks npm first, then increments `attempt`, clears `claimed_run_id`, sets a new `claim_deadline`, sets `publish.state = dispatched` and dispatches again.
- Every other reason is final for that `release_id`. Publishing needs a new submission and a new approval.
- Ordering: the gate dispatches a release only when no other release of the same package is `dispatched` or `running`; the rest wait in the gate's queue in approval order and are dispatched as each one ends. The claim already stops two runs of one release. The workflow sets no `concurrency` group: GitHub keeps one pending run per group and cancels an older pending run when a new one arrives, and a group name would show on the public run page.
- Never wait on the chain.

Reason codes: `record_invalid`, `signer_unknown`, `record_state`, `artifact_missing`, `hash_mismatch`, `identity_mismatch`, `publish_config`, `package_missing`, `revoked`, `npm_error`, `integrity_mismatch`, `runner_timeout`.

## 5. Runner endpoints on the gate

Full contract with request/response examples and error codes: [endpoints.md](endpoints.md). Summary below.

### 5.1 Dispatch (gate → GitHub)
- GitHub App installed on `coderoot-eth/coderoot-release-runner` only, permission `actions: write` only. Its key lives in the gate's key vault.
- `workflow_dispatch` of `release.yml` on `ref: main` with `inputs: { release_id }`.
- Gate stores a dispatch record: `release_id`, `attempt`, `dispatched_at`, `claim_deadline`, expected `org_id` and environment, `claimed_run_id: null`, `claimed_run_attempt: null`, `previous_run_ids`.
- A dispatch alone cannot publish anything: the runner verifies the signed record itself.

### 5.2 Runner authentication (all runner → gate calls)
`Authorization: Bearer <GitHub OIDC token>`, audience `secure.coderoot.app`, verified against GitHub's JWKS. Required claims:

| Claim | Must equal |
|---|---|
| `iss` | `https://token.actions.githubusercontent.com` |
| `repository` | `coderoot-eth/coderoot-release-runner` |
| `workflow_ref` | `coderoot-eth/coderoot-release-runner/.github/workflows/release.yml@refs/heads/main` |
| `job_workflow_ref` | equal to `workflow_ref`, so the job cannot run a reusable workflow from elsewhere |
| `ref` | `refs/heads/main` |
| `event_name` | `workflow_dispatch` |
| `actor_id` | the gate GitHub App's bot account, for the claim; a run dispatched by anyone else is not bound |
| `run_id`, `run_attempt` | the run and attempt bound to this release |
| `environment` | the release's `org_id`, for §5.5 and for `running` / `published` in §5.6 (absent for `resolve`) |

Checking `ref` and `workflow_ref` closes the branch bypass on the gate side as well.

### 5.3 Claim: `POST /v1/runner/releases/{release_id}/claim`
First call from a run. Binds `run_id` and `run_attempt` to the release if the release is `dispatched`, unclaimed and before `claim_deadline`, and `run_id` is not in `previous_run_ids`; idempotent for the same `run_id` and `run_attempt`. Any other run gets `409 release_claimed`, including a re-run of the bound run (same `run_id`, higher `run_attempt`) and a run from an earlier gate attempt. Claims after the deadline get `410 claim_expired`.

### 5.4 Record: `GET /v1/runner/releases/{release_id}`
Returns the signed envelope (RFC 8785 bytes) and its secp256k1 signature, plus release state and publish state. Bound run only.

### 5.5 Artifact: `GET /v1/runner/releases/{release_id}/artifact`
Streams the bytes from storage. Bound run only, `environment` claim required and matching. Only while state is `approved` and `publish.state` is `dispatched` or `running`.

### 5.6 State: `POST /v1/runner/releases/{release_id}/state`
Body: `{ state: running|published|failed, reason?, runner_run_id, registry_url?, registry_integrity?, provenance_uri?, published_at? }`. Bound run only. `running` and `published` require the matching `environment` claim; `failed` is also accepted from `resolve`, which has none, so a refusal there is recorded.
Gate rules: ignore if revoked; for `published`, read npm and require `dist.integrity` = record SHA-512 before storing; a version not yet visible gets `202` and a re-read with backoff ([endpoints.md](endpoints.md) §5).

### 5.7 Signing keys: `GET /v1/runner/signing-keys?org_id=`
Lists authorised signer addresses with role, per org. Used as a cross-check only, see §6.

Error format: the gate API error shape. Codes: [endpoints.md](endpoints.md) §9.

## 6. Signer verification
Two independent sources must agree on every signer, so no single component can authorise one:
- `signers.json` in the runner repo: per `org_id`, address, role, `valid_from`, `valid_to`. Changes only through a reviewed PR approved by both code owners, @PabloReyes and @rafaljanicki (`.github/CODEOWNERS`). Schema: [endpoints.md](endpoints.md) §6.
- The gate's signer list (§5.7).

The runner accepts a record only if the signer is in both for the record's `org_id`, with the same role, that role may approve, and the payload's `approved_at_ms` lies inside the validity window in both. Anything else fails `signer_unknown`.
- Role-key addresses are already public as EAS attesters, so listing them adds no exposure.
- Rotation: add the new key to both with `valid_from`; set `valid_to` on the old key in both. Records signed inside the old window stay valid.

## 7. Customer isolation
- One GitHub environment per customer, named exactly the customer's **`org_id`** (e.g. `org_example`). No prefix, no transformation; runner and gate compare it verbatim. Permanent, like the repo name.
- `org_id` is an opaque identifier, never a customer name: environment names are public (§8.1).
- Every environment: deployment branches `main` only, no secrets, no variables.
- Each customer's package trusts only its own environment.
- With more than one customer, the environment is the isolation boundary.

## 8. Repo and workflow hardening
Repo: public, branch protection on `main` (two reviews, code-owner review on `.github/workflows/` and `signers.json`, no force push), minimal admins and writers. Code owners for both paths are @PabloReyes and @rafaljanicki, and both approvals are required. CODEOWNERS alone accepts any one owner's approval, so a required status check confirms that both have approved.
Workflow: dispatch trigger only; per-job permissions as in §3; actions pinned to commit SHAs; cache off; npm pinned to an exact version (12.2.0, npm's current release); no `setup-node` `registry-url`; no install, build or pack in either job; inputs only through `env`; job timeouts of 5 minutes for `resolve` and 15 minutes for `publish`, so a run ends before the gate's 30-minute stall timeout; GitHub-hosted runners only. Every call to the gate uses a freshly requested OIDC token.

### 8.1 Public by design
The repo is public, so anyone can read its run list, run logs, workflow inputs, environment names and `signers.json`. Rules:
- Logs carry only `release_id`, `attempt`, reason codes and digests. Never the package name or version before `npm publish`, the record payload, gate responses, the tarball's contents, or a token.
- Failure detail (expected and actual values) goes to the gate's state endpoint, not to the log.
- No workflow artifacts and no job summaries with release data.
- npm's own `npm publish` output is allowed: it runs only after every check has passed, for a version about to be public.
- `signers.json` holds only opaque `org_id`s and role-key addresses, which are already public as EAS attesters.

## 9. Provenance
On. The statement names `coderoot-eth/coderoot-release-runner` and `release.yml` as the build source. The customer's source repo and commit stay in the record and the bundle.

## 10. Stage-only option (decision: gate owner)
npm can restrict trust to staging (`--allow-stage-publish`); the runner stages and a package maintainer approves with 2FA.
Gain: a hijacked runner can stage but not ship. Cost: a human step per release by the customer's maintainer.
Proposal: per-customer choice at onboarding, default direct publish; exactly one permission set.

## 11. Hosting
GitHub-hosted runners: fresh VM per job, nothing persists. Self-hosted only if a customer objects to bytes passing through GitHub; then ephemeral and single-use. Not in v1.

## 12. Customer onboarding (summary; detail in [onboarding.md](onboarding.md))
1. New package only: maintainer publishes a README-only placeholder by hand with 2FA, dist-tag `placeholder`, deprecated ([first-publish.md](first-publish.md)).
2. Check existing trust; npm allows one configuration per package, so the customer's own CI publisher is replaced.
3. `npm trust github <pkg> --repository coderoot-eth/coderoot-release-runner --file release.yml --environment <org_id>` with exactly one of `--allow-publish` / `--allow-stage-publish`.
4. Publishing access: "Require 2FA and disallow tokens".
5. Revoke old npm tokens; remove npm credentials from the customer's CI.
6. Review maintainers; all keep 2FA.
7. Explain the provenance change.
8. First release through the gate.

CodeRoot side, before step 3: create environment `<org_id>` (main only), add the org's role-key addresses to `signers.json` by reviewed PR.

## 13. First publish of a new package (summary; detail in [first-publish.md](first-publish.md))
Trusted publishing needs the package to exist. The customer's maintainer creates it with a README-only placeholder; the agent never touches npm. The runner refuses to create packages (`package_missing`). The placeholder's version is whatever the maintainer chose, so nothing depends on its format: the gate refuses any submission whose version is already on npm (§14), which covers the placeholder. The placeholder may sit on `latest` until the first real release takes it.

## 14. Go-live failure modes (summary; detail in [failure-modes.md](failure-modes.md))
| Case | Rule |
|---|---|
| Version already on npm, same bytes | Report `published`, no second publish |
| Version already on npm at submit or approval | Gate refuses it (`version_exists`); never approved, never dispatched. Covers the placeholder whatever its version |
| Version already on npm, different bytes, at publish | It appeared after approval: `integrity_mismatch`, alert: bypass signal |
| Two runs for one release | Claim endpoint lets only one through |
| Re-run of a claimed run | `run_attempt` differs: refused; retries go through the gate reset |
| Several releases of one package | Gate queue dispatches one at a time |
| Retry after failure | Gate reset, retryable reasons only, after the npm check; old run refused |
| Revoke during publish | Re-check before publish; never undo `revoked`; npm version stays if already out |
| npm or GitHub outage | Fail closed; the gate reset starts with the npm check |
| Run never starts or stalls | Gate timeout → `runner_timeout` after npm check |
| Prerelease / lower version | Fixed tag rule, not chosen by the submitter |
| New version not yet visible on npm | Runner and gate read with backoff; not visible is never `integrity_mismatch` |
| `publishConfig` in the tarball | `publish_config`; the registry is always passed on the command line |
| `published` report after `runner_timeout` | Accepted if the gate's npm read matches |

## 15. Tests (DoD)
| Requirement | Test |
|---|---|
| Approved version on npm, hash matches | runner publish; registry `dist.integrity` = record SHA-512 |
| Tampering refused | change one byte in storage after approval → `hash_mismatch` |
| Unapproved release cannot trigger | dispatch a `release_id` that is unknown or not `approved` → claim refused (`404` / `409 release_not_publishable`), run exits, nothing on npm; a record whose signature does not verify → `record_invalid` |
| Agent holds no npm token | Action credential guard (exit 4); no npm secret in runner repo or environments |
| Forged signer | signer not in `signers.json` → `signer_unknown` |
| Signer only on the gate | signer in the gate list but not in `signers.json` → `signer_unknown` |
| `publishConfig` redirect | tarball with `publishConfig.registry` → `publish_config`, nothing published anywhere |
| Registry lag | `published` report before the version is visible → gate keeps `running`, then stores `published` |
| Dispatch by a repo writer | run not dispatched by the gate's App → claim refused `403 runner_claim_mismatch` |
| Identity mismatch | tarball `package.json` version differs → `identity_mismatch` |
| Customer isolation | run in org A's environment cannot publish an org B package |
| Version already on npm | submit a version that is already on npm, e.g. the placeholder → refused `version_exists` at submit; a version published between submit and approval → refused at approval. No dispatch, no integrity alert |
| Retry | reset after `npm_error`: new run claims; re-run of the old run → `409 release_claimed` |
| Re-run before failure is reported | bound run crashes, then is re-run from the Actions UI → `409 release_claimed` / `403 runner_claim_mismatch` (`run_attempt`); release waits for the gate timeout and a reset |
| Ordering | approve two releases of one package → the second is dispatched only after the first ends |
| Branch bypass | non-main branch run refused by GitHub and by the gate |
| Revoke | revoke after dispatch: before the claim → claim refused; after it → run stops `revoked` before publish. Release stays `revoked`, nothing on npm |
| Old token | publish with an old npm token refused |
| Second run | `409 release_claimed` |
Attack cases for attack test suite: [attack-tests.md](attack-tests.md).

## 16. Build checklist
Who builds is decided after this spec. Split by side:

**Runner side (`coderoot-eth/coderoot-release-runner`)**
- [ ] Branch protection on `main`: two reviews, CODEOWNERS on `.github/workflows/` and `signers.json`, no force push
- [ ] `release.yml`: `resolve` + `publish` jobs as §3, permissions per job, pinned actions, exact npm, no cache, no install/build/pack
- [ ] `signers.json` per org, reviewed
- [ ] Record verification as [endpoints.md](endpoints.md) §3: canonical envelope bytes, SHA-256, secp256k1 recovery equal to `role_key_address`, act only on the verified payload
- [ ] `.github/CODEOWNERS` and the required check that both owners approved
- [ ] Artifact checks: both hashes, `package.json` identity, `publishConfig` allowlist, npm existence/version check, revoke re-check
- [ ] Publish command with explicit `--registry` and `--ignore-scripts`; registry read with backoff
- [ ] Log rules of §8.1
- [ ] State reporting and reason codes
- [ ] Tests from §15 that run without the gate (fixtures + throwaway packages)

**Gate side**
- [ ] GitHub App: runner repo only, `actions: write` only, key in the key vault
- [ ] Dispatch + dispatch record
- [ ] OIDC verification with the claim table ([endpoints.md](endpoints.md) §1)
- [ ] Endpoints: claim, record, artifact, state, signing-keys
- [ ] State rules: transitions, revoked ignores updates, npm integrity re-check on `published` with backoff, late `published` after `runner_timeout`
- [ ] `actor_id` check on claim; claim binds `run_id` + `run_attempt`
- [ ] Per-package dispatch queue in approval order
- [ ] Submit and approval refuse a version already on npm (`version_exists`)
- [ ] Reset for retryable reasons: npm check, `attempt`, `previous_run_ids`, new deadline
- [ ] Timeout job and alerts to the operations alert channel

**Per customer**
- [ ] Environment named `<org_id>` (main only, no secrets)
- [ ] Onboarding §12

## 17. To confirm before final
| # | Assumption | Status |
|---|---|---|
| V1 | npm refuses a run from the wrong environment | Confirmed: a package trusting another environment refused the publish (`ENEEDAUTH`, no token from the exchange) |
| V2 | A non-main branch run cannot publish | Confirmed: GitHub refused the branch run before its first step (environment branch rule); a run with no environment was refused by npm (`ENEEDAUTH`) |
| V3 | "Disallow tokens" refuses an old token while trusted publishing works | Trusted publishing with "Require 2FA and disallow tokens" confirmed. Old-token refusal: open, npm check run 5 |
| V4 | Stage-only refuses direct publish; staged + 2FA approve works; provenance on staged versions | Direct publish refused (`E403 OIDC permission denied for this action`). Staging works and npm signs the provenance statement at stage time. Open: approval with 2FA, and the attestation on the approved version |
| V5 | Record format matches the gate's implementation | Fields and canonical bytes confirmed against an unsigned test vector ([endpoints.md](endpoints.md) §3). Open: a signed vector, for the signature check |
| V6 | Endpoints §5 and pinned signers §6 accepted | Open: gate owner review |
| V7 | Where `latest` points after the placeholder publish | Confirmed: a placeholder published with `--tag placeholder` also becomes `latest` |
| V8 | Gate submit and approval refuse a version already on npm (`version_exists`) | Open: gate owner. This changes the gate's submit API contract, which needs the same addition |
| V9 | The signed payload identifies the org | Open: the payload has no `org_id`. Either the gate adds `org_id` to the payload, or the runner takes the org from `role_key_address` through `signers.json`, which requires every address to belong to one org only. Gate owner decides |
| V10 | A restricted (private) scoped package publishes through trusted publishing with provenance off | Open: npm check run 8. If it works, provenance is on for public packages and off for private ones (§9, G11); if not, private packages are out of v1 (§1) |

## 18. Sign-off
Countersigned by: ____________ Date: ________
