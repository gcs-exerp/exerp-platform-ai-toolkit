# Commit Template and Rules
Commit messages must follow `DEV-XXXXX [type] Short title`.
- Start with the Jira ticket ID, followed by a bracketed type. Valid types are `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`, and `revert`.
- Use an imperative, clear, and concise title describing the change and impacted area; keep it under 50 characters where possible and do not end it with a period.
- Leave one empty line after the title and between body paragraphs. Wrap body lines at 72 characters.
- Use the body to explain why the change was made, not only what changed. Use bullets when they improve clarity.
- Suffix the commit message at the end with "+slack +docker" and ask the user if he wants to add other suffixes.
- Keep each commit focused on one logical change and use only ASCII characters (lower Latin-1); do not include markup.
- When squashing work by multiple developers, add one `Co-authored-by: Name <email>` trailer per contributor.
- Prefer focused commits per logical change; avoid mixing refactors and behaviour changes without reason.