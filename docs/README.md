# Release runner documentation

**Parked:** this is the premium path, where CodeRoot holds a customer's npm publish path. It is built when a customer needs it. Until then, publishers release on npm themselves and submit the release to CodeRoot for attestation.

The release runner is the only identity allowed to publish a customer's npm package. It publishes exactly the bytes a human approved, under the approved name and version.

| Document | Contents |
|---|---|
| [spec.md](spec.md) | Specification: flow, states, isolation, hardening, tests, build checklist |
| [endpoints.md](endpoints.md) | Runner endpoints on the gate, with request and response examples |
| [onboarding.md](onboarding.md) | Steps a customer follows to let the runner publish their package |
| [first-publish.md](first-publish.md) | How a brand-new package gets its first version |
| [failure-modes.md](failure-modes.md) | What can go wrong at publish time and the rule for each case |
| [attack-tests.md](attack-tests.md) | Attacks to run against the finished runner |
