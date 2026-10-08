# First publish of a new package

Problem: npm trusted publishing can only be configured on a package that already exists.
So the very first version of a brand-new package cannot come from the runner. Someone must create it,
and that someone must never be the agent.

Facts (npm CLI documentation):
- `npm trust`: package must exist; caller needs write access and 2FA on the account; GATs with 2FA bypass are not accepted.
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

1. Maintainer (human, own machine, 2FA) publishes `0.0.0-placeholder.0` with dist-tag `placeholder`, never `latest`, so nobody installs it by default. Content: README only. Scoped packages need `--access public`.
2. Maintainer deprecates it: `npm deprecate <pkg>@0.0.0-placeholder.0 "Placeholder; releases ship through CodeRoot"`.
3. Maintainer binds trust: `npm trust github <pkg> --repository coderoot-eth/coderoot-release-runner --file release.yml --environment <customer-env> --allow-publish` (or `--allow-stage-publish`, see stage-only decision).
4. Maintainer sets publishing access to "Require 2FA and disallow tokens" and removes any tokens they created.
5. First real version goes through the gate: submit, approve, runner publishes.

Why: the agent never touches npm, the placeholder carries no code, and the first real version is gated like every later one.
The placeholder reads `unregistered` in `verify`; that is correct and expected, and it is not on `latest`.

## Spec requirements
- Onboarding (D) lists these steps for new packages only.
- Runner refuses to publish if the package does not exist on npm (no implicit creation).
- Runner refuses if the version to publish is the placeholder version.

## To confirm
- On throwaway packages: placeholder published by hand, trust bound, then the runner publishes the next version.
