# RedS41ft/renovate-config

Shared Renovate presets for the whole org. Policy lives here and nowhere else.

- `default.json` — active repos. Patch/minor/pin/digest automerge behind CI; majors and core frameworks (FastAPI, SQLAlchemy, React, Vite) wait for review. Weekly, Monday pre-6am, Europe/London.
- `dormant.json` — long-tail repos. Inherits default, then automerges everything monthly.

Repos opt in with a one-line `renovate.json`:

```json
{ "extends": ["local>RedS41ft/renovate-config"] }
```

or for the dormant tier:

```json
{ "extends": ["local>RedS41ft/renovate-config:dormant"] }
```

## Rollout (one-off)

### 1. Create this repo and push the presets

```bash
gh repo create RedS41ft/renovate-config --private -y
# add default.json, dormant.json, this README; commit and push
```

### 2. Install the Mend Renovate app

https://github.com/apps/renovate — install on the RedS41ft org, all repositories.
Renovate opens an onboarding PR per repo. Because `RedS41ft/renovate-config`
exists, onboarding PRs will propose extending it automatically.

### 3. Enable GitHub auto-merge on every repo

Renovate uses platform automerge by default, which needs `allow_auto_merge`:

```bash
gh repo list RedS41ft --limit 100 --json name -q '.[].name' | \
  xargs -I{} gh api -X PATCH "repos/RedS41ft/{}" -f allow_auto_merge=true
```

### 4. Batch-merge the onboarding PRs

Review one or two to sanity-check, then:

```bash
for repo in $(gh repo list RedS41ft --limit 100 --json name -q '.[].name'); do
  pr=$(gh pr list -R "RedS41ft/$repo" --author 'app/renovate' \
       --search 'Configure Renovate' --json number -q '.[0].number')
  [ -n "$pr" ] && gh pr merge -R "RedS41ft/$repo" "$pr" --squash
done
```

For dormant repos, edit the onboarding PR (or the merged `renovate.json`)
to extend `:dormant` instead.

### 5. Retire Dependabot version updates

Delete `.github/dependabot.yml` from each repo (or empty its `updates:` list).
Keep Dependabot **security alerts** enabled in repo settings — they're free
and feed Renovate's `vulnerabilityAlerts` handling anyway.

### 6. Point the Monday digest at Renovate

In the `dependabot-digest` script, change the PR search author filter:

```
author:app/dependabot  →  author:app/renovate
```

(or include both during the transition). Everything else stays the same —
the digest will now list only the majors and framework bumps waiting on you.

## Safety notes

- **Renovate will not automerge a branch with zero passing checks.** By
  default it requires at least one green check, so no-CI repos silently fall
  back to normal PRs. To force automerge there anyway, set
  `"ignoreTests": true` in that repo's `renovate.json` — do this eyes-open,
  ideally only for lockfile maintenance.
- Policy changes go in this repo only. Repos pick them up on Renovate's next
  run — nothing to redeploy.
- First run against 26 repos will queue a backlog; `prConcurrentLimit` and
  `prHourlyLimit` in the presets make it trickle over a few days rather
  than flood.
