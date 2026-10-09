# Customer onboarding

Who: the customer's npm package maintainer, with 2FA on their npm account.
Before: CodeRoot creates the customer's GitHub environment, named exactly the customer's `org_id`, in `coderoot-eth/coderoot-release-runner` (main only, no secrets).

## Steps
1. **Package exists?** If brand new, follow [first-publish.md](first-publish.md) (placeholder) first.
2. **Check existing trust.** `npm trust list <pkg>`. npm allows one trust configuration per package. If the customer's own CI is the trusted publisher today, it is replaced; their own CI can no longer publish. Confirm they accept this.
3. **Bind trust to CodeRoot.** `npm trust github <pkg> --repository coderoot-eth/coderoot-release-runner --file release.yml --environment <org_id>` plus exactly one of `--allow-publish` or `--allow-stage-publish` (stage-only choice, [spec.md](spec.md) §10).
4. **Disallow tokens.** Package settings → publishing access → "Require two-factor authentication and disallow tokens".
5. **Remove old tokens.** Revoke npm tokens that could publish this package; remove `NPM_TOKEN` / `NODE_AUTH_TOKEN` and `.npmrc` auth lines from the customer's CI secrets.
6. **Review maintainers.** List maintainers; remove anyone who should not be able to publish by hand. Every remaining maintainer has 2FA.
7. **Provenance change.** Tell the customer: npm provenance will name CodeRoot's runner as the build source. Their source repo and commit stay in the CodeRoot record and bundle.
8. **Verify.** First release through the gate; check npm shows the version published by the trusted publisher with provenance.

## Checks CodeRoot runs after onboarding
- `npm trust list <pkg>` shows exactly one configuration pointing at the runner and the right environment.
- A publish attempt with an old token is refused.
- For stage-only packages: a direct publish from the runner is refused.

## Open until the npm checks
- Whether a wrong-environment run is refused (isolation).
- Whether "disallow tokens" refuses a fresh granular token on this setup.
- How staged versions show provenance, if at all.
