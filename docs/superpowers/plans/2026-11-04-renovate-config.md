# Haven-BX/renovate-config Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship public `Haven-BX/renovate-config` as the fleet Renovate preset with CI validation, docs, and branch protection.

**Architecture:** Shareable Renovate presets (`default.json`, `apps.json`) consumed via `github>Haven-BX/renovate-config` / `:apps`. CI installs pinned `renovate` from `package-lock.json` and runs `renovate-config-validator --strict`; zizmor audits workflows via SHA-pinned action.

**Tech Stack:** Renovate presets (JSON), npm (devDependency for renovate CLI), GitHub Actions, zizmor-action.

## Global Constraints

- Org/casing: always `Haven-BX`
- Repo: **public** (intentional exception to private-only org default)
- Extends: no relative paths — use `github>Haven-BX/renovate-config` and `github>Haven-BX/renovate-config:apps`
- `platformAutomerge: false`; schedule `on monday`; lockFileMaintenance `automerge: false`
- Never `latest` for CI tooling; actions pinned to full SHA + version comment
- `permissions: contents: read` only; triggers `push: branches: [main]` + `pull_request`
- Dogfooding reads preset from default branch after merge — not a PR test gate
- Security PRs skip `prConcurrentLimit` by default (document; do not set a limit inside `vulnerabilityAlerts`)
- Security majors may still hit Dependency Dashboard approval — document as intentional
- Do not rest decision text on a specific Mend poll interval

---

### Task 1: Presets + package tooling

**Files:**
- Create: `default.json`, `apps.json`, `renovate.json`, `package.json`, `package-lock.json`, `.gitignore`
- Modify: `docs/superpowers/specs/2026-10-28-renovate-config-design.md` (extends syntax + four notes)

- [ ] **Step 1: Write `default.json`**

Verified options against Renovate docs. Astro is already grouped via `group:monorepos` → `monorepo:astro`; keep a custom group so `wrangler` joins Astro and minor/patch automerge. Explicit npm `minimumReleaseAge: 14 days` packageRule overrides `security:minimumReleaseAgeNpm`’s 3-day floor from `config:best-practices`.

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "description": "Haven BX default preset for client Astro sites",
  "extends": ["config:best-practices"],
  "timezone": "Europe/Helsinki",
  "schedule": ["on monday"],
  "minimumReleaseAge": "14 days",
  "internalChecksFilter": "strict",
  "prConcurrentLimit": 3,
  "rangeStrategy": "bump",
  "platformAutomerge": false,
  "lockFileMaintenance": {
    "enabled": true,
    "schedule": ["before 7am on the first day of the month"],
    "automerge": false
  },
  "packageRules": [
    {
      "description": "14-day wait for npm updates (overrides best-practices 3-day npm floor)",
      "matchDatasources": ["npm"],
      "minimumReleaseAge": "14 days",
      "internalChecksFilter": "strict"
    },
    {
      "description": "Patches, pins and digests merge themselves after CI passes",
      "matchUpdateTypes": ["patch", "pin", "digest"],
      "automerge": true
    },
    {
      "description": "Astro and Cloudflare tooling move together (adds wrangler beside monorepo:astro)",
      "matchPackageNames": ["astro", "@astrojs/**", "wrangler"],
      "matchUpdateTypes": ["minor", "patch"],
      "groupName": "Astro and Cloudflare",
      "automerge": true
    },
    {
      "description": "Majors wait for a human on the Dependency Dashboard",
      "matchUpdateTypes": ["major"],
      "automerge": false,
      "dependencyDashboardApproval": true,
      "labels": ["major", "needs-human"]
    }
  ],
  "vulnerabilityAlerts": {
    "schedule": ["at any time"],
    "minimumReleaseAge": "3 days",
    "labels": ["security"]
  }
}
```

- [ ] **Step 2: Write `apps.json`**

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "description": "Haven BX stricter preset for internal apps (Iris, Riva, consent log)",
  "extends": ["github>Haven-BX/renovate-config"],
  "labels": ["app"],
  "packageRules": [
    {
      "description": "Turn off inherited automerge for patches/pins/digests/minors",
      "matchUpdateTypes": ["patch", "pin", "digest", "minor"],
      "automerge": false
    },
    {
      "description": "Auto-merge only patch updates to devDependencies",
      "matchDepTypes": ["devDependencies"],
      "matchUpdateTypes": ["patch"],
      "automerge": true
    }
  ]
}
```

- [ ] **Step 3: Write `renovate.json` (dogfood; reads default branch after merge)**

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>Haven-BX/renovate-config"]
}
```

- [ ] **Step 4: Add `package.json`, `.gitignore`, install lockfile**

```json
{
  "name": "renovate-config",
  "private": true,
  "description": "Haven BX shared Renovate presets",
  "engines": {
    "node": ">=22"
  },
  "devDependencies": {
    "renovate": "44.212.0"
  },
  "scripts": {
    "validate": "renovate-config-validator --strict default.json apps.json renovate.json"
  }
}
```

`.gitignore`: `node_modules/`

Run: `npm install`  
Expected: `package-lock.json` created; `node_modules/renovate` present.

- [ ] **Step 5: Validate presets locally**

Run: `npm run validate`  
Expected: exit 0 (may warn that remote preset `github>Haven-BX/renovate-config` is not fetchable until the repo exists — if validator requires the remote, create the empty public repo first then re-run; or temporarily validate `default.json` alone, then full set after push).

- [ ] **Step 6: Commit**

```bash
git add default.json apps.json renovate.json package.json package-lock.json .gitignore
git commit -m "Add Renovate presets and pinned validator tooling"
```

---

### Task 2: Docs (README, decisions, AGENTS)

**Files:**
- Create: `README.md`, `decisions/0001-update-policy.md`, `CLAUDE.md`
- Modify: `AGENTS.md` (replace haven-bx → Haven-BX; incorporate standing refinements)

- [ ] **Step 1: Write README** (what/why, site + apps snippets, override example + comment rule, change policy, never enable before CI, Dependabot alerts note, dogfood caveat)

- [ ] **Step 2: Write `decisions/0001-update-policy.md`** including: platformAutomerge false; Monday full day (no claimed poll interval); no lockfile automerge; apps override order; security PRs skip concurrent limit by default; security majors may need dashboard approval intentionally; dogfood is post-merge only

- [ ] **Step 3: Update AGENTS.md / copy CLAUDE.md** with Haven-BX and refinements

- [ ] **Step 4: Commit**

```bash
git add README.md decisions/0001-update-policy.md AGENTS.md CLAUDE.md
git commit -m "Document preset usage, change policy, and update decisions"
```

---

### Task 3: CI workflow

**Files:**
- Create: `.github/workflows/validate.yml`

Pinned (looked up 2026-11-04):
- `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1` # v7.0.1
- `actions/setup-node@949feb2413d6458794dcd2491c4babbbce0c15c1` # v7.1.0
- `zizmorcore/zizmor-action@45d7276c9ca790aabd705e31ca99c6185c3e76dd` # v0.8.1
- Node: `26` (current LTS)

- [ ] **Step 1: Write validate.yml** with validate + zizmor jobs as specified

- [ ] **Step 2: Run validator again; run zizmor locally if available (`gh`/`docker`) else rely on CI**

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/validate.yml
git commit -m "Add SHA-pinned validate and zizmor workflow"
```

---

### Task 4: Create public GitHub repo, push, protect main, deliberate-break test

- [ ] **Step 1:** `gh repo create Haven-BX/renovate-config --public --source=. --remote=origin --description "Shared Renovate presets for Haven BX sites and apps"`
- [ ] **Step 2:** Push `main`; confirm remote public
- [ ] **Step 3:** Branch protection: require PR + status check `validate` (and `zizmor` if separate check name); require approving review if API allows
- [ ] **Step 4:** On a throwaway branch, break a preset, open PR, confirm validate fails; revert/close
- [ ] **Step 5:** Re-run `npm run validate` now that remote preset exists

---

### Task 5: Spec/plan sync

- [ ] **Step 1:** Update design spec with extends syntax, dogfood caveat, security edge cases, remove hard-coded 4-hour claim from decision rationale
- [ ] **Step 2:** Commit docs if changed
