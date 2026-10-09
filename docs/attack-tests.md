# Attack tests

Each attack runs against a working runner. The result is recorded as pass or fail against one rule: an agent cannot publish.

| # | Attack | Setup | Expected result |
|---|---|---|---|
| AT-1 | Publish without approval | Dispatch the runner with a `release_id` that has no signed record | `failed` / `record_invalid`, nothing on npm |
| AT-2 | Publish without approval, forged record | Dispatch with a record signed by a key that is not the org role key | `failed` / `signer_unknown` |
| AT-3 | Publish from outside the runner | Run a copy of `release.yml` on a non-main branch; separately, run another workflow file | Refused by GitHub (environment rule) or by npm (trust mismatch) |
| AT-4 | Publish with an npm token | Agent environment or maintainer token tries `npm publish` | Refused (2FA + disallow tokens) |
| AT-5 | Different artifact than approved | Change one byte of the stored artifact after approval | `failed` / `hash_mismatch` |
| AT-6 | Same bytes, different identity | Approved tarball whose `package.json` version differs from the record | `failed` / `identity_mismatch` |
| AT-7 | Cross-customer publish | Approved release for org A dispatched so it runs in org B's environment | Environment comes from the record, so this cannot be chosen; if forced, npm refuses |
| AT-8 | Replay a submission | Dispatch an already published release again | Ends `published` without a second publish; no duplicate |
| AT-9 | Replay after revoke | Dispatch a revoked release | `failed` / `revoked` |
| AT-10 | Input injection | `release_id` containing shell metacharacters | Rejected by format check before any use |
| AT-11 | Fake status | Post `published` to the gate without a run | Rejected: state endpoint accepts only the bound run with matching OIDC claims ([endpoints.md](endpoints.md)) |
| AT-12 | Run from a branch claims a release | Push a branch, dispatch there, call the claim endpoint | Gate refuses: `ref` / `job_workflow_ref` not main ([endpoints.md](endpoints.md)) |
| AT-13 | Second run steals a release | Two runs claim the same `release_id` | Second gets `409 release_claimed` |
| AT-14 | Placeholder release | Approved record for `0.0.0-placeholder.0` | `failed` / `placeholder_version`, no integrity alert, nothing on npm |
| AT-15 | Re-run an old run | After a gate reset, re-run the failed run from the Actions UI | `409 release_claimed`; only the new attempt's run proceeds |
| — | Forge a bundle | — | Out of runner scope (evidence bundle) |
| — | Revoke without authority | — | Out of runner scope (gate revoke API) |
