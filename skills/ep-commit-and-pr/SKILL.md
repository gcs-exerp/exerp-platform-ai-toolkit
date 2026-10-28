---
name: ep-commit-and-pr
description: Reuse the ticket baseline established by ep-work, then commit ticket-scoped Git changes and open a pull request using the repository's configured templates. Use for Exerp Platform shipping requests.
metadata:
  short-description: Commit session changes and open a compliant PR
---

# EP commit and PR

Use this skill only for the Exerp Platform repository. It performs the commit-and-PR tail of a change; it does not implement or test product changes.

## Required safety boundary

Before any staging, commit, push, or PR mutation, reuse the ticket baseline established by `ep-work`. A baseline is a trusted snapshot of `HEAD`, the index, and the worktree file paths captured before implementation began. Use the verified Jira ticket as `<ticket>` and read `.git/ep-commit-and-pr-baselines/<ticket>.json`. The ticket must match `DEV-[0-9]+` and be the verified Jira ticket for this change. If the ticket or baseline is unavailable, stop and report: “That information is unavailable.” Never infer the baseline from file timestamps, the current diff, or the branch name.

Read and validate the JSON baseline before any mutation. Reject it if it is
malformed, its `ticketId` does not match the path or verified ticket, or its
repository root does not match the current checkout.

The baseline must have been created by `ep-work` before implementation began.
Reuse it without creating another baseline. The baseline must contain
`schema_version`, `repository_root`, `ticketId`, `branch`, `head`, the complete
session-start `git status --porcelain=v1` output, staged and unstaged
`git diff --cached --name-status` and `git diff --name-status` path/status lists,
and the session-start untracked path list from
`git ls-files --others --exclude-standard`.

The baseline must identify:

- the schema version and verified Jira ticket ID;
- the repository root and branch;
- the starting `HEAD` commit;
- paths already modified, staged, deleted, or untracked at session start;
- the index/worktree state needed to distinguish later edits.

Treat every path present in that baseline as out of scope, including a path that was later edited again. Do not stage, commit, push, or mention those paths in the PR. Review the final allowlist before staging and abort if it contains a baseline path or an unexpected path.

Never create or overwrite a baseline. Reuse the verified ticket baseline across
continuation sessions. Treat every path recorded in the baseline as
pre-existing and out of scope for shipping. Do not write the baseline into the
repository's commit.

## Configuration and templates

Read the repository's `.exerp-platform-ai-toolkit/config.yaml` and resolve `commit-template` and `pr-template` relative to the repository root. If an optional local override exists according to that configuration's conventions, apply it where applicable, but keep the tracked config's template paths as the fallback. Read both template files completely before drafting anything.

If configuration or either template is missing, unreadable, or invalid, stop before mutation and report the exact missing information. Do not invent template rules.

The current repository configuration normally resolves to:

- `.exerp-platform-ai-toolkit/templates/commit-template.md`
- `.exerp-platform-ai-toolkit/templates/pr-template.md`

These are fallbacks for discovery only; the config remains authoritative.

## Workflow

1. Confirm the repository root, verified Jira ticket, current branch, remote, and ticket baseline created by `ep-work`. Validate and reuse the baseline; if it is absent or invalid, stop before mutation. Inspect `git status`, the baseline allowlist, and the diff limited to paths outside the baseline.
2. Determine the Jira ticket, change type, concise imperative title, implementation summary, and verification evidence from the actual diff and conversation. If the ticket or required facts cannot be verified, stop and ask for them rather than inventing them.
3. Check that every path outside the baseline is intended, that no baseline path is included, and that no secrets or generated artifacts are being shipped. Preserve unrelated work.
4. Draft a commit message that exactly follows the configured commit template. Keep it focused on one logical change, use the required ticket/type/title format, include the why-oriented body and required suffix/trailers, and validate the character and line-wrapping rules from the template.
5. Stage only the reviewed allowlist of paths outside the baseline by explicit path. Never use a broad staging command. Re-check the staged diff and staged path list, then create the commit only after the user has authorized the requested commit-and-PR action.
6. Push the current branch to its configured remote. If the branch has no upstream, use the repository's normal branch name and set the upstream only as part of this requested workflow. Never amend any commit and never force-push anything to the PR.
7. Draft the PR title and body from the configured PR template. The body must contain the required short implementation details followed by the commits on separate lines, plus any other sections required by the template. Use verified facts only.
8. Query the remote for open PRs whose head is the current branch. If exactly one matching open PR exists, do not create another PR; use that PR and only commit and push the new commit to the configured remote so the existing PR reflects it. If more than one matching open PR exists, stop and report the ambiguity. If none exists, open one non-draft PR with the repository's GitHub CLI or equivalent configured integration, targeting the repository's normal base branch. The PR must remain open after creation. Never close or merge it; if the provider cannot create it open and non-draft, stop before creation. Capture and report the existing or newly created PR URL.
9. Afterward, verify the commit, pushed branch, PR URL, and that no baseline path entered the commit. Report the exact paths committed, commit ID, and PR URL.

## Hard constraints

- Never include paths recorded in the ticket baseline, even if they are changed again in a continuation session.
- Never create or overwrite the ticket baseline; baseline ownership belongs to `ep-work`.
- Never reset, checkout, clean, delete, amend any commit, force-push anything to the PR, or otherwise discard existing work.
- Never close, merge, or create the PR as a draft; it must be created open and non-draft.
- Never create a second PR when an open PR already exists for the current branch.
- Never stage with `git add -A`, `git add .`, or an equivalent broad operation.
- Keep commit and PR text ASCII when required by the configured templates; do not add markup where the commit template forbids it.
- If a required decision or authorization is missing, stop before external mutation and ask one concise question.
