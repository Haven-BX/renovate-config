# Haven-BX/renovate-config

Shared Renovate presets for every Haven BX client site and internal apps. Changing this repo changes update behaviour fleet-wide on the next Renovate run.

## Client sites

Add only this file as `renovate.json`:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>Haven-BX/renovate-config"]
}
```

## Internal apps (Iris, Riva, consent log)

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>Haven-BX/renovate-config:apps"]
}
```

## Per-site overrides

Overrides are rare. Prefer fixing the shared preset. If a site truly needs an exception, add `packageRules` next to `extends` and explain why in a comment:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>Haven-BX/renovate-config"],
  "packageRules": [
    {
      // Pin stays until upstream X ships a fix for Y — see ticket Z
      "matchPackageNames": ["some-package"],
      "enabled": false
    }
  ]
}
```

## Change policy

1. Every change via pull request.
2. `validate` (and `zizmor`) must pass.
3. Describe the fleet-wide effect in the PR body.
4. Ask Robin before merging if the change weakens release age, automerge scope, or the major-approval gate.

## Warnings

- **Never enable Renovate on a repo before shared CI** (`Haven-BX/workflows` v1) is wired. Renovate treats “no checks” as passing and would auto-merge untested updates.
- **Dependabot alerts** must be enabled on a repo for Renovate `vulnerabilityAlerts` security PRs to appear.
- This repo’s own `renovate.json` extends the preset from the **default branch**. A PR here is not dogfooded until after merge; the validator and review are the real gate.

## Validate locally

```bash
npm ci --ignore-scripts
npm run validate
```
