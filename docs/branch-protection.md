# Branch Protection Rollout

## Rulesets

Three ruleset specs live under `.github/rulesets/`. All are committed with
`enforcement: disabled` and activated in stages via the 3-PR rollout below.

### `main-protection`
Targets `~DEFAULT_BRANCH` (main). Enforces:
- No deletions or force pushes
- Required linear history (no merge commits)
- All changes via pull request (0 required approvals, stale-review dismissal, thread resolution required, squash/rebase only)
- 10 required status checks: lint, Node 22/24 matrix (ubuntu/mac/windows), Changeset Required, require-issue-link, pr-template-format

### `release-branches`
Targets `refs/heads/release/**` and `refs/heads/hotfix/**`. Same rules as
`main-protection` except `required_linear_history` is omitted (merge commits
are permitted on release/hotfix branches).

### `tag-immutability`
Targets all tags (`~ALL`). Blocks tag updates and deletions — tags are
immutable once created. Tag creation is unrestricted.

## 3-PR Rollout Plan

| PR | Branch | Action |
|----|--------|--------|
| PR-1 (this PR) | `chore/branch-protection-specs` | Check in spec files; `enforcement: disabled` — no effect on repo |
| PR-2 | `chore/branch-protection-evaluate` | Run `sync-rulesets.sh` with `ENFORCEMENT=evaluate`; 1-week dry-run via rule-suite logs |
| PR-3 | `chore/branch-protection-active` | Run `sync-rulesets.sh` with `ENFORCEMENT=active`; protection live |

## Running `sync-rulesets.sh`

**Prerequisites:** `gh` authenticated with repo-admin scope, `jq` installed.

```bash
# Dry-run (evaluate mode — logs violations, does not block)
REPO=GSD-redux/get-shit-done-redux ENFORCEMENT=evaluate bash scripts/sync-rulesets.sh

# Activate protection
REPO=GSD-redux/get-shit-done-redux ENFORCEMENT=active bash scripts/sync-rulesets.sh

# Roll back to disabled
REPO=GSD-redux/get-shit-done-redux ENFORCEMENT=disabled bash scripts/sync-rulesets.sh
```

The script is idempotent: running it twice with the same `ENFORCEMENT` value
is a no-op semantically (PUT with identical body).

## Reading evaluate-mode logs

After applying with `evaluate`, check which PRs/pushes would have been blocked:

```bash
REPO=GSD-redux/get-shit-done-redux
RULESET_ID=$(gh api repos/$REPO/rulesets --jq '.[] | select(.name=="main-protection") | .id')
gh api repos/$REPO/rulesets/$RULESET_ID/rule-suites
```

Each entry shows the actor, ref, result (`pass`/`fail`), and which rules
triggered. Use this to validate no legitimate workflows are broken before
flipping to `active` in PR-3.

## Phase-2 TODO

Enable the `required_signatures` rule (signed commits) once agent commits sign
uniformly. As of PR-1 this rule is intentionally omitted — unsigned agent
commits would be blocked by it. Track readiness in the issue linked to PR-3.
