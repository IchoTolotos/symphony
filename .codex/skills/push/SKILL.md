---
name: push
description:
  Push current branch changes to origin and create or update the corresponding
  pull request; use when asked to push, publish updates, or create pull request.
---

# Push

## Prerequisites

- `gh` CLI is installed and available in `PATH`.
- `gh auth status` succeeds for GitHub operations in this repo.

## Goals

- Push current branch changes to `origin` safely.
- Create a PR if none exists for the branch, otherwise update the existing PR.
- Target the PR at the issue's source branch instead of assuming `main`.
- Keep branch history clean when remote has moved.

## Related Skills

- `pull`: use this when push is rejected or sync is not clean (non-fast-forward,
  merge conflict risk, or stale branch).

## Steps

1. Identify current branch, determine the base branch, and confirm remote state.
   - Prefer the `Base branch (from branch-* label): ...` value from the issue
     context in `WORKFLOW.md`.
   - If a PR already exists, `gh pr view --json baseRefName -q .baseRefName`
     is an acceptable source of truth.
   - If neither source is available, stop and report that the base branch could
     not be determined safely.
2. Run local validation for SetScout before pushing.
   - Default gate: `npm run lint && npm run type-check`
   - If the ticket/workpad requires stronger or more targeted validation, run
     that too before every push.
3. Push branch to `origin` with upstream tracking if needed, using whatever
   remote URL is already configured.
4. If push is not clean/rejected:
   - If the failure is a non-fast-forward or sync problem, run the `pull`
     skill to merge `origin/<base_branch>`, resolve conflicts, and rerun
     validation.
   - Push again; use `--force-with-lease` only when history was rewritten.
   - If the failure is due to auth, permissions, or workflow restrictions on
     the configured remote, stop and surface the exact error instead of
     rewriting remotes or switching protocols as a workaround.

5. Ensure a PR exists for the branch:
   - If no PR exists, create one.
   - If a PR exists and is open, update it.
   - If branch is tied to a closed/merged PR, create a new branch + PR.
   - Write a proper PR title that clearly describes the change outcome
   - For branch updates, explicitly reconsider whether current PR title still
     matches the latest scope; update it if it no longer does.
6. Write/update a concise PR body without assuming a template:
   - Include `## Summary` with 1-3 bullets covering the full PR scope.
   - Include `## Test plan` with concrete commands/checks actually run.
   - If PR already exists, refresh title/body so they match the current total
     branch scope, not just the latest commit.
7. Reply with the PR URL from `gh pr view`.

## Commands

```sh
# Identify branch
branch=$(git branch --show-current)
base_branch="<value from issue.source_branch or existing PR base>"

# Minimal validation gate
npm run lint && npm run type-check

# Initial push: respect the current origin remote.
git push -u origin HEAD

# If that failed because the remote moved, use the pull skill. After
# pull-skill resolution and re-validation, retry the normal push:
git push -u origin HEAD

# If the configured remote rejects the push for auth, permissions, or workflow
# restrictions, stop and surface the exact error.

# Only if history was rewritten locally:
git push --force-with-lease origin HEAD

# Ensure a PR exists (create only if missing)
pr_state=$(gh pr view --json state -q .state 2>/dev/null || true)
if [ "$pr_state" = "MERGED" ] || [ "$pr_state" = "CLOSED" ]; then
  echo "Current branch is tied to a closed PR; create a new branch + PR." >&2
  exit 1
fi

# Write a clear, human-friendly title that summarizes the shipped change.
pr_title="<clear PR title written for this change>"
tmp_pr_body=$(mktemp)
cat > "$tmp_pr_body" <<'EOF'
## Summary
- <bullet covering the main change>

## Test plan
- `npm run lint`
- `npm run type-check`
EOF

if [ -z "$pr_state" ]; then
  gh pr create --base "$base_branch" --title "$pr_title" --body-file "$tmp_pr_body"
else
  # Reconsider title/body/base on every branch update; edit if scope shifted.
  gh pr edit --base "$base_branch" --title "$pr_title" --body-file "$tmp_pr_body"
fi

rm -f "$tmp_pr_body"

# Show PR URL for the reply
gh pr view --json url -q .url
```

## Notes

- Do not use `--force`; only use `--force-with-lease` as the last resort.
- Distinguish sync problems from remote auth/permission problems:
  - Use the `pull` skill for non-fast-forward or stale-branch issues.
  - Surface auth, permissions, or workflow restrictions directly instead of
    changing remotes or protocols.
