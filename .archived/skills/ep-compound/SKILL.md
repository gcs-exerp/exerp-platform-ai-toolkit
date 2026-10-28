---
name: ep-compound
description: "Capture one verified Exerp Platform Jira change using local research and implementation plans; optionally inspect and update only kickoff notes in the Jira description."
argument-hint: "[DEV-XXXXX] [kickoff-notes:off] [mode:non-interactive] [depth:lightweight|full]"
---

# EP Compound

Use this wrapper when a solved Exerp Platform Jira item should be documented through
`compound-engineering:ce-compound`. It supplies the Jira ID and local plan context,
then adds the Jira-specific kickoff-note handoff when the caller supplies enough
description context.

## Required context

1. Resolve the Jira key from the invocation arguments. If no Jira key is available in
   the current session, ask the user for it. Accept `DEV-<digits>`; do not infer a
   key from unrelated text. If the key cannot be verified, stop.
2. Read `.exerp-platform-ai-toolkit/config.yaml` from the repository root. Do not use a
   local override for these settings. Resolve `docs_root` using the same rules as
   `ce-compound`; if it is unset, use its documented default. Read `kickoff-template`
   from this file only when kickoff notes are enabled. A missing or unusable template
   blocks only the kickoff-update portion; do not invent a template.
3. Discover existing research and implementation-plan documents only below the
   configured artifact root, especially its `research/` and `plans/` areas. Select
   documents whose verified content or filename relates to the Jira key; preserve
   exact paths and report when no matching document is found.
4. Before invoking `ce-compound`, check only the filenames under
   `<docs_root>/solutions/` for an existing solution document whose filename starts
   with the exact Jira key followed by `-`. If one exists, report its path and skip
   solution generation and solution verification entirely. Do not open, inspect,
   validate, update, or replace the existing solution document.
5. Do not call Jira MCP read/search operations before `ce-compound` runs. Do not
   provide Jira issue fields, comments, links, or description content to
   `ce-compound`. Do not inspect the current branch for pull requests and do not
   look for open pull requests through any connector or CLI. The only Jira read
   permitted is the post-compound description check described below, and it is
   exclusively for kickoff-note detection.

## Delegate to ce-compound

When no matching solution file exists, internally run
`compound-engineering:ce-compound` with one explicit context brief containing:

- the Jira key supplied by the caller;
- matching local research and implementation-plan paths and relevant verified
  contents; and
- any stated limitations.

Apply the delegated skill's durable-learning bar, artifact-root rules, validation,
and report format. Do not lower the bar or fabricate a solved problem. Run one
learning per invocation.

When generating a solution, its filename must always start with the exact supplied Jira key,
followed by a hyphen and descriptive slug, such as
`DEV-66114-refund-revocation.md`. Pass this naming requirement into the delegated
skill and verify it after writing. If the delegated workflow cannot produce a
Jira-prefixed filename, stop and report the nonconforming result rather than
silently accepting it.

### Mandatory wrapper override: no concepts file

The delegated skill's vocabulary-capture phase is disabled for this wrapper. Do not
create, update, seed, reconcile, scrub, or otherwise touch any `CONCEPTS.md`, whether
it already exists or not. Report the vocabulary phase as skipped by `ep-compound`'s
policy. This override applies in every mode.

No other repository files may be changed by this wrapper beyond the qualifying
learning artifact written by the delegated skill.

## Kickoff notes

Unless the invocation contains `kickoff-notes:off` or repository config explicitly
sets `kickoff-notes` to `off`, run this after the solution phase completes or is
skipped because a matching solution already exists:

1. Call `mcp__jira__jira_get_issue_field_value` with exactly
   `{ "issue_key": "<jira-key>", "field_id": "description" }` only for kickoff
   detection. Do not provide the Jira description to `ce-compound` or use it as
   solution-generation input.
2. Safely inspect the returned description. Accept a direct ADF object or exactly
   one text content block whose `text` value parses as JSON. Parse only with a
   strict JSON parser, and continue only when the parsed value has `type: "doc"`, a
   numeric `version`, and an array `content`. Reject malformed, partial, mixed, or
   non-ADF responses.
3. Detect an existing kickoff-notes section using the configured template heading,
   a stable kickoff marker, or the known kickoff heading. If it is already present,
   report that kickoff notes already exist and stop this kickoff flow. Do not render
   replacement notes, ask to update them, or call an update tool.
4. If kickoff notes are absent, read the file configured by `kickoff-template` in
   `.exerp-platform-ai-toolkit/config.yaml`, render it using only the Jira key, local
   research/plan documents, and any newly generated compound result, and preserve
   the template's structure. Do not invent missing values.
5. Pass the generated Markdown directly to
   `mcp__jira__jira_append_markdown_to_issue_description` with only:
   `{ "issue_key": "<jira-key>", "markdown": "<generated markdown>" }`.
   This operation preserves the existing rich description and converts only the
   appended Markdown into ADF. Do not call `jira_update_issue`, replace the full
   description, send ADF JSON, stringify JSON, or update any other Jira field.
6. If the description read or Markdown append fails, report the failure and leave
   Jira unchanged. If kickoff notes are disabled, do not read the template and do
   not call any Jira update operation.

## Boundaries

- Use only the Jira key and verified local research, plan, and repository data.
- Do not implement code, read Jira MCP data during solution generation, or look for
  open pull requests. The sole Jira read exception is the post-compound description
  check for kickoff notes.
- Never replace or rewrite the existing Jira description; the kickoff flow may only
  append newly generated Markdown when no kickoff notes are present.
- If required information is unavailable, say so explicitly rather than guessing.
