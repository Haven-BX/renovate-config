# AGENTS.md — Haven-BX/renovate-config

Standing rules for this repo. Copy lives in `CLAUDE.md`.

## What this repo is

The single source of truth for **how and when dependencies are updated** across every Haven BX
client site (and internal apps like Iris and Riva).

Every site repo contains only this `renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>Haven-BX/renovate-config"]
}
```

Apps use `"extends": ["github>Haven-BX/renovate-config:apps"]`.

Renovate reads `default.json` from this repo on every run. Changing this repo changes update
behaviour for the whole fleet on the next run. **A mistake here breaks updates everywhere at once**,
so every change is validated in CI before it lands.

### Fleet context

- Client sites: Astro 7, mostly static output, deployed to Cloudflare Workers (static assets),
  built by Cloudflare from Git. Content edited by clients in CloudCannon, which commits to the repo.
- Forms post to a central service (Iris). Client sites have no logins and few or no secrets.
- Package manager: npm 11.10+. Each site's `.npmrc` sets `min-release-age=3` (days) and
  `ignore-scripts=true`.
- Every site runs a shared CI workflow (from `Haven-BX/workflows`) on every PR and every push to
  `main`. Renovate must **never** be enabled on a repo before that CI exists, because Renovate
  treats "no checks" as passing and would auto-merge untested.
- Small team (2 people). Goal: humans only handle exceptions. ~10 min/week fleet-wide.

### Locked choices

- Org: `Haven-BX` (same casing everywhere)
- Visibility: public
- Runner: Mend Renovate GitHub App
- `platformAutomerge: false`; schedule `on monday`; lockFileMaintenance does not automerge
- Extends use `github>Haven-BX/renovate-config` / `:apps` — no relative paths
- This repo’s dogfood `renovate.json` reads the preset from the **default branch** after merge; it is not a PR test

## Standing rules for anyone (human or AI) editing this repo

- Never weaken `minimumReleaseAge`, auto-merge scope or the major-approval gate without a written
  decision in `decisions/`.
- Never add an option you have not confirmed in the current Renovate docs.
- Never use `latest` for tooling versions in CI. Pin Renovate in `package.json` / lockfile; pin Actions by SHA.
- Keep it small. If a rule only applies to one site, it belongs in that site's `renovate.json`, not here.
- When in doubt, ask Robin before merging. This repo affects every client site.

See `decisions/0001-update-policy.md` for why the rules are what they are.
