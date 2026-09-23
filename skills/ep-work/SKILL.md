---
name: ep-work
description: "Implement an Exerp Platform change from a plan and, by default, commit it and open a pull request."
argument-hint: "[implementation plan path or plan content] [commit-and-pr:off]"
---

# EP Work

Implement an Exerp Platform change from the supplied implementation plan. This
skill is an execution wrapper around `compound-engineering:ce-work` followed by
`ep-commit-and-pr` unless shipping is explicitly disabled.

## Workflow

1. Resolve the input as either an implementation-plan path or plan content. If
   neither is supplied, ask the user for the plan and stop.
2. Resolve and validate the Jira ticket and current session ID before any
   implementation change. The ticket must match `DEV-[0-9]+` and come from
   verified plan or task context; the session ID must come from the current
   Codex session metadata. Never infer either value from a timestamp, branch
   name, filename, or generated random value. If either value is unavailable,
   stop and report that the information is unavailable.
3. Establish the repository root and create the JSON baseline at
   `.git/ep-commit-and-pr-baselines/<ticket>/<session-id>.json`. If that exact
   path already exists, stop and ask the user how to handle it; never overwrite
   an existing baseline silently. The JSON must contain:
   - `schema_version`, `repository_root`, `ticket`, `session_id`, `branch`,
     and starting `head`;
   - the complete session-start `git status --porcelain=v1` output;
   - staged and unstaged path/status lists from
     `git diff --cached --name-status` and `git diff --name-status`;
   - the session-start untracked path list from
     `git ls-files --others --exclude-standard`.

   Run all snapshot commands before writing the file, write valid JSON
   atomically, and verify that the stored ticket and session ID match the path.
   Treat every path recorded in it as pre-existing and out of scope for any
   later shipping workflow. Do not add the baseline file to the product change
   or commit it.
4. Read the complete `compound-engineering:ce-work` skill instructions and
   follow only its implementation and local-verification workflow for the
   supplied plan.
5. Invoke that workflow with `babysit:off` in implementation-only/
   return-to-caller mode. `ce-work` must not create commits, amend commits,
   push changes, or open, update, or raise a pull request. Do not let it enter
   its standalone shipping workflow; those actions belong exclusively to
   `ep-commit-and-pr` below.
6. Keep the work limited to the plan's scope and preserve the repository's
   existing instructions, including any module-specific instructions.
7. After implementation and local verification, inspect the input for the
   exact switch `commit-and-pr:off`.
8. If the switch is absent, read and invoke the complete
   `ep-commit-and-pr` skill using the baseline created in step 3. Follow its
   configuration, authorization, staging, commit, push, and open non-draft PR
   rules exactly. Do not bypass its safety checks or create a second baseline.
9. If `commit-and-pr:off` is present, do not invoke `ep-commit-and-pr`; stop
   after local verification and report changed files, verification performed,
   and any blockers.

## Delivery boundary

- The default outcome includes the `ep-commit-and-pr` workflow and its commit
  and open non-draft PR actions.
- `commit-and-pr:off` is the only switch that disables that workflow.
- `ce-work` is never the shipping owner. It may implement and locally verify
  only; `ep-commit-and-pr` owns all commit, push, and PR actions.
- When disabled, never create a commit, amend a commit, push changes, or open,
  update, or raise a pull request.

If the switch is malformed or ambiguous, treat it as absent and follow the
default commit-and-PR path.
