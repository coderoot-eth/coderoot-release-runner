# Attack tests

Each attack runs against a working runner. The result is recorded as pass or fail against one rule: an agent cannot publish.

| # | Attack | Setup | Expected result |
|---|---|---|---|
| AT-1 | Publish without approval | Dispatch the runner with a `release_id` that is unknown or not approved | Claim refused (`404 release_not_found` / `409 release_not_publishable`), run exits, nothing on npm |
| AT-2 | Publish without approval, forged record | Dispatch with a record signed by a key that is not the org role key | `failed` / `signer_unknown` |
| AT-3 | Publish from outside the runner | Run a copy of `release.yml` on a non-main branch; separately, run another workflow file | Refused by GitHub (environment rule) or by npm (trust mismatch) |
| AT-4 | Publish with an npm token | Agent environment or maintainer token tries `npm publish` | Refused (2FA + disallow tokens) |
| AT-5 | Different artifact than approved | Change one byte of the stored artifact after approval | `failed` / `hash_mismatch` |
| AT-6 | Same bytes, different identity | Approved tarball whose `package.json` version differs from the record | `failed` / `identity_mismatch` |
| AT-7 | Cross-customer publish | Approved release for org A dispatched so it runs in org B's environment | Environment comes from the record, so this cannot be chosen; if forced, the gate refuses the artifact (`403 runner_claim_mismatch`) and npm refuses the publish |
| AT-8 | Replay a submission | Dispatch an already published release again | Claim refused (`409 release_not_publishable`), run exits; release stays `published`, no second publish |
| AT-9 | Replay after revoke | Dispatch a revoked release | Claim refused (`409 release_not_publishable`), run exits; release stays `revoked`, nothing on npm |
| AT-10 | Input injection | `release_id` containing shell metacharacters | Rejected by format check before any use |
| AT-11 | Fake status | Post `published` to the gate without a run | Rejected: state endpoint accepts only the bound run with matching OIDC claims ([endpoints.md](endpoints.md)) |
| AT-12 | Run from a branch claims a release | Push a branch, dispatch there, call the claim endpoint | Gate refuses: `ref` / `job_workflow_ref` not main ([endpoints.md](endpoints.md)) |
| AT-13 | Second run steals a release | Two runs claim the same `release_id` | Second gets `409 release_claimed` |
| AT-14 | Placeholder release | Approved record for `0.0.0-placeholder.0` | `failed` / `placeholder_version`, no integrity alert, nothing on npm |
| AT-15 | Re-run an old run | After a gate reset, re-run the failed run from the Actions UI | `409 release_claimed`; only the new attempt's run proceeds |
| AT-16 | Gate serves a forged signer | Record signed by a key added to the gate's signer list but not to `signers.json` | `failed` / `signer_unknown`, nothing on npm |
| AT-17 | Approved bytes redirect the publish | Approved tarball whose `package.json` sets `publishConfig.registry` to another registry | `failed` / `publish_config`; nothing published to either registry |
| AT-18 | Repo writer dispatches the workflow | A writer dispatches `release.yml` on `main` for a `dispatched` release before the gate's run claims it | Claim refused `403 runner_claim_mismatch` (`actor_id`); the gate's own run proceeds |
| — | Forge a bundle | — | Out of runner scope (evidence bundle) |
| — | Revoke without authority | — | Out of runner scope (gate revoke API) |
