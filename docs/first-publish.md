# First publish of a new package

npm trusted publishing can only be configured on a package that already exists.
The first version of a brand-new package therefore cannot come from the runner. A person creates it, never the agent.

Facts (npm CLI documentation):
- `npm trust`: package must exist; caller needs write access and 2FA on the account; granular access tokens with 2FA bypass are not accepted.
- The registry supports one trust configuration per package. A new one requires revoking the old one.
- Staged publishing also requires the package to exist, so stage-only does not solve this.

Applies only to brand-new packages. A customer onboarding an existing package skips this.

## Options

| # | Option | Agent holds a token? | Problems |
|---|---|---|---|
| 1 | Customer maintainer publishes a placeholder by hand, with 2FA, then binds trust to the runner | No | One unattested version on npm |
| 2 | Gate approves the real first release; maintainer publishes those exact approved bytes by hand, then binds trust | No | First real version published by a person, not the runner; needs a download-and-verify step |
| 3 | CodeRoot account publishes the placeholder in the customer's scope | No | CodeRoot becomes a maintainer of customer packages. Custody risk, rejected |

## Recommendation: option 1

1. Maintainer (human, own machine, 2FA) publishes `0.0.0-placeholder.0` with `--tag placeholder`. Content: README only. Scoped packages need `--access public`. As the only version, it may still land on `latest` (checked with `npm view <pkg> dist-tags` in npm check run 1). Steps 2 and 5 cover this: it is deprecated, and the first real release takes `latest` (failure-modes G10).
2. Maintainer deprecates it: `npm deprecate <pkg>@0.0.0-placeholder.0 "Placeholder; releases ship through CodeRoot"`.
3. Maintainer binds trust: `npm trust github <pkg> --repository coderoot-eth/coderoot-release-runner --file release.yml --environment <org_id> --allow-publish` (or `--allow-stage-publish`, see stage-only decision).
4. Maintainer sets publishing access to "Require 2FA and disallow tokens" and removes any tokens they created.
5. First real version goes through the gate: submit, approve, runner publishes.

Why: the agent never touches npm, the placeholder carries no code, and the first real version is gated like every later one.
The placeholder reads `unregistered` in `verify`; that is correct and expected. Until the first real release it may be `latest`: deprecated, README only.

## Spec requirements
- [onboarding.md](onboarding.md) lists these steps for new packages only.
- Runner refuses to publish if the package does not exist on npm (no implicit creation).
- Runner refuses the placeholder version with `placeholder_version` in the `resolve` job, before any npm or artifact check. Otherwise it would find the version on npm with different bytes and raise a false `integrity_mismatch` bypass alert.

## To confirm
- On throwaway packages: placeholder published by hand, trust bound, then the runner publishes the next version.
- `npm view <pkg> dist-tags` right after the placeholder publish: whether `latest` points at it.
