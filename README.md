# RedS41ft/renovate-config

Shared Renovate presets for the whole org. Policy lives here and nowhere else.

- `default.json` — active repos. Patch/minor/pin/digest automerge behind CI as soon as checks pass (`automergeSchedule: "at any time"`); majors and core frameworks (FastAPI, SQLAlchemy, React, Vite) wait for review. New branches open on a daily pre-6am check, Europe/London.
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

### 3. ~~Enable GitHub auto-merge on every repo~~ — not needed, and doesn't work here

`default.json` sets `platformAutomerge: false`, so Renovate merges PRs itself
via the API — it does not use GitHub's native "auto-merge" button, and this
repo's `allow_auto_merge` setting is irrelevant to it either way.

This also couldn't have worked as originally written: GitHub's native
auto-merge (and branch protection generally) requires GitHub Pro for private
repos on a personal account — the API silently no-ops the field on a private
repo (`gh api -X PATCH repos/{owner}/{repo} -f allow_auto_merge=true` returns
200 but leaves the setting `false`). It only really takes effect on public
repos.

What Renovate's self-merge actually needs is `automergeSchedule: ["at any
time"]` — without it, merging is bundled into the same run as branch
creation, so a PR that goes green mid-week just sits until the next
scheduled run instead of merging the moment CI passes. That's set in
`default.json`.

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
