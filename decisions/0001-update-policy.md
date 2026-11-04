# 0001 — Update policy

## Context

Haven BX needs one shared Renovate policy so humans only handle exceptions (~10 min/week). Client sites are static Astro apps with shared CI; internal apps (Iris, Riva) need a stricter automerge scope.

## Decision

Ship `default.json` for sites and `apps.json` for internal apps. Key rules and why:

| Rule | Reason |
| --- | --- |
| `config:best-practices` | Renovate maintainers’ recommended base. Pins GitHub Actions to commit SHAs and flags abandoned packages. |
| `minimumReleaseAge: 14 days` (npm packageRule) | Renovate maintainers recommend ~14 days before auto-merging third-party updates. Most hijacked npm versions are pulled within hours or days. Explicit packageRule overrides the 3-day floor from `security:minimumReleaseAgeNpm` inside best-practices. |
| Security fixes at 3 days | Faster than normal updates, but not shorter than sites’ npm `min-release-age=3`, or Renovate’s own lockfile update can fail. |
| `schedule: ["on monday"]` | One predictable batch day per week. A full Monday window (Europe/Helsinki) is wide enough that a hosted-app run is likely to land inside it; a narrow “before 7am” window can miss and slip a week. |
| `prConcurrentLimit: 3` | Keeps PR noise low per site. |
| `platformAutomerge: false` | Renovate defaults this to `true`. With platform automerge and no required status checks, GitHub can merge before CI finishes. Renovate’s own automerge waits for green checks. |
| Patches / pins / digests auto-merge | Low risk, high volume; CI is the gate. |
| Astro + Cloudflare group | `group:monorepos` already groups Astro packages; this rule adds `wrangler` and automerges minor/patch together. |
| Majors need Dependency Dashboard approval | Breaking changes are done deliberately, starter first, rarely. |
| `internalChecksFilter: strict` | Renovate does not create a PR until internal checks (like release age) are met. |
| Lockfile maintenance: no automerge | Renovate’s release-age check does not apply to lockfile maintenance; npm’s `.npmrc` `min-release-age` is the intended floor if npm reads it. One PR per site per month for human review. |
| Apps: disable inherited automerge, then allow only devDependency patches | Extending `default.json` would otherwise auto-merge runtime patches and Astro minors. Later `packageRules` override earlier ones. |
| Vulnerability alerts skip the concurrent PR limit by default | Renovate docs: security PRs ignore `prConcurrentLimit` unless you set a limit *inside* `vulnerabilityAlerts`. We leave that unset so security fixes are not queued behind three open update PRs. |
| Security majors may still need dashboard approval | If a security fix is also a major, the major-approval rule may hold it on the Dependency Dashboard. That is acceptable: a security major should get human eyes. |

## Consequences

- A mistake in this repo affects every consumer on the next Renovate run → CI validator, branch protection, and human review on `main`.
- Dogfooding via this repo’s `renovate.json` only reflects the preset **after merge** to the default branch; it is not a PR test.
- Security PRs require Dependabot **alerts** enabled on each consumer repo.
- Haven-BX org Actions policy is currently `local_only`, so marketplace actions (`actions/checkout`, `zizmorcore/zizmor-action`, etc.) cannot run. CI uses `run:`-only checkout + pinned zizmor binary download until the org allows selected/github-owned actions; then prefer SHA-pinned `uses:`.
