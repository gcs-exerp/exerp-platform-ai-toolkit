# exerp-platform-ai-toolkit

## Codex session audit

Generate a local, standalone HTML report for one completed Codex session. The
report summarizes event order, available token and timing metrics, detected
skill references, and exact repeated tool-call candidates. Skill attribution
and repetition are review signals, not proof of invocation or waste.

The tool reads only the file you name. It does not scan the session archive,
open a browser, or contact a service. Prompt and instruction bodies, tool
arguments, and tool outputs are not included in the report.

Locate a rollout JSONL file under `$CODEX_HOME/sessions` (or
`~/.codex/sessions` when `CODEX_HOME` is unset), then run:

```sh
python3 tools/codex_session_audit.py /path/to/rollout-<session-id>.jsonl \
  --output /path/to/session-audit.html
```

The generated HTML file is self-contained and can be opened locally. The
rollout format is an internal format that may change; unsupported records are
skipped and reported as a coverage limitation. Metrics that are absent from
the selected trace are shown as unavailable.
