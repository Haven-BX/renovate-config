# Design: Haven-BX/renovate-config

Date: 2026-10-28  
Status: approved for planning  
Repo: public `Haven-BX/renovate-config`  
Renovate runner: Mend Renovate GitHub App  
GitHub plan: Team (org); this repo uses public-repo branch protection regardless

## Goal

Single source of truth for how and when dependencies update across every Haven BX client site and internal apps (Iris, Riva, consent log). Site repos extend this preset only; mistakes here affect the whole fleet on the next Renovate run.

## Decisions locked

| Topic | Choice |
| --- | --- |
| Org | `Haven-BX` (same casing everywhere) |
| Visibility | Public |
| Runner | Mend Renovate GitHub App |
| Approach | Brief-literal: `default.json` + `apps.json` + dogfood |

## Repo layout

```
renovate-config/
├── AGENTS.md                      # standing rules (org name Haven-BX)
├── CLAUDE.md                      # copy of AGENTS.md
├── package.json                   # renovate (+ zizmor if npm-installed) as devDependencies
├── package-lock.json
├── default.json                   # client-site preset
├── apps.json                      # internal-app preset
├── renovate.json                  # dogfood → extends ./default.json
├── README.md
├── decisions/0001-update-policy.md
└── .github/workflows/validate.yml
```

### Consumer configs

Sites:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>Haven-BX/renovate-config"]
}
```

Apps:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>Haven-BX/renovate-config:apps"]
}
```

Extends use shareable-preset syntax only — no relative paths (invalid in Renovate).
`apps.json` and this repo’s `renovate.json` use `github>Haven-BX/renovate-config`
(loads `default.json`). Consumers of the apps preset use `github>Haven-BX/renovate-config:apps`.

## Preset behaviour

### `default.json`

- Extends `config:best-practices`
- `"timezone": "Europe/Helsinki"`
- `"schedule": ["on monday"]` — full Monday (Helsinki), not a narrow morning window
- `"minimumReleaseAge": "14 days"` plus an npm `packageRule` at 14 days (overrides `security:minimumReleaseAgeNpm`’s 3-day floor from best-practices)
- `"internalChecksFilter": "strict"`
- `"prConcurrentLimit": 3` — vulnerability alert PRs skip this limit by default unless a limit is set inside `vulnerabilityAlerts` (we leave that unset)
- `"rangeStrategy": "bump"`
- `"platformAutomerge": false` — Renovate default is `true`; set false so Renovate merges after green checks without relying on platform required-status-checks
- `lockFileMaintenance`: enabled; schedule first day of month before 7am (Helsinki); **`automerge: false`** (explicit). Renovate’s own `minimumReleaseAge` does not apply to lockfile maintenance; sites’ npm `min-release-age=3` is the intended floor if npm reads `.npmrc`. Monthly human review.
- Package rules:
  1. npm 14-day release age (see above)
  2. patch / pin / digest → `automerge: true`
  3. Astro + Cloudflare: `astro`, `@astrojs/**`, `wrangler`; minor + patch → group + `automerge: true` (`group:monorepos` already groups Astro; this adds `wrangler` and automerge)
  4. major → `automerge: false`, `dependencyDashboardApproval: true`, labels `major`, `needs-human` (a security fix that is also a major may still wait here — intentional)
- `vulnerabilityAlerts`: schedule any time; `minimumReleaseAge: "3 days"`; label `security`. README notes these PRs only appear if Dependabot **alerts** are on for the repo.
- No required-review assumptions in config.

### `apps.json`

- Extends `github>Haven-BX/renovate-config` (loads `default.json` from this repo’s default branch)
- Label everything `app`
- Because later `packageRules` override earlier ones, and extending default brings patch + Astro automerge, rules **must** be ordered:
  1. `matchUpdateTypes: ["patch", "pin", "digest", "minor"]` → `automerge: false`
  2. `matchDepTypes: ["devDependencies"]`, `matchUpdateTypes: ["patch"]` → `automerge: true`
- Same 14-day / 3-day security timing via inheritance
- Proof: validator; apps onboarding PR description lists what will/won’t automerge (Iris enablement only after its CI exists)

### `renovate.json` (this repo)

Extends `github>Haven-BX/renovate-config`. Dogfooding applies only **after merge** to the default branch — not a PR test. Validator + review are the gate. Renovate can still bump pinned tooling behind the 14-day wait once live.

## CI

### Triggers

- `push: branches: [main]`
- `pull_request`
- Not unrestricted `push` (avoids double CI on every PR branch)

### Jobs

Shared constraints: top-level `permissions: contents: read` only; `timeout-minutes` on every job; actions pinned to full commit SHAs with version comments; `actions/checkout` with `persist-credentials: false`; current Node LTS via SHA-pinned `actions/setup-node`.

1. **validate** — `npm ci --ignore-scripts`, then run `renovate-config-validator --strict` on `default.json`, `apps.json`, `renovate.json` from `node_modules` (not `npx renovate@…` inline).
2. **zizmor** — blocking static audit of `.github/workflows/`. Prefer SHA-pinned action; if installed via npm, same commit-lockfile pattern as Renovate.

Acceptance: intentionally break a preset once to prove CI fails, then revert.

### Branch protection (this repo)

Public repo → require PR + green `validate` before merging to `main`. Human review required: valid-but-wrong configs (e.g. auto-merging majors) pass the validator and still hit every site.

## Docs

- **README:** what the repo is (2 sentences); site + apps extend snippets; rare per-site override example + “explain in a comment” rule; change policy (PR, validator green, describe fleet-wide effect); never enable Renovate before shared CI; Dependabot alerts note for security PRs.
- **`decisions/0001-update-policy.md`:** why table including `platformAutomerge: false`, Monday full-day schedule, no lockfile automerge, apps override order.
- **AGENTS.md / CLAUDE.md:** standing brief with `Haven-BX` substituted.

## Out of scope / pilot order

Build of this repo does **not** enable Renovate on any consumer.

1. Ship this repo (validated + protected `main`)
2. Shared `Haven-BX/workflows` repo has a `v1` and is wired on the pilot site
3. Enable Renovate on **one** pilot site; inspect onboarding PR for schedule, release age, grouping
4. Apps/Iris: if CI not live, inspect apps behaviour from onboarding PR description only; enable Renovate on Iris only once its CI exists

## Standing rules (carry into AGENTS.md)

- Never weaken `minimumReleaseAge`, auto-merge scope, or the major-approval gate without a written decision in `decisions/`
- Never add an option not confirmed in current Renovate docs
- Never use `latest` for CI tooling versions
- Keep fleet rules here; one-site rules stay in that site’s `renovate.json`
- When in doubt, ask Robin before merging
