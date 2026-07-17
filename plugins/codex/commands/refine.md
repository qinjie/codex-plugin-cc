---
description: Run one Codex review of a plan or spec file and fold the feedback into it
argument-hint: '<file> [--model <model|spark>] [--effort <none|minimal|low|medium|high|xhigh>]'
disable-model-invocation: true
allowed-tools: Read, Edit, Bash(node:*), AskUserQuestion
---

Run one Codex review pass over a plan or spec markdown file and fold Codex's feedback directly into that file.

Raw slash-command arguments:
`$ARGUMENTS`

Unlike `/codex:review` and `/codex:rescue`, this command is meant to act on Codex's output: it edits the target file. This is a single pass. Run it again if you want another round.

Target file:
- The first positional argument is the path to the plan or spec file to refine.
- If no file path is present, use `AskUserQuestion` exactly once to ask which file to refine, then stop if still unspecified.
- `--model` and `--effort` are runtime-selection flags. Preserve them for the Codex call, but do not treat them as part of the file path.
- If the user asks for `spark`, map it to `gpt-5.3-codex-spark`.
- Leave `--model` and `--effort` unset unless the user supplied them.

Path safety (do this before running Codex):
- Verify the target file exists by opening it with the `Read` tool, not with a shell command. This confirms the path is a real file rather than an injection payload.
- Refuse to run if the path contains shell metacharacters — quotes (`'` or `"`), `$`, backticks, `;`, `|`, `&`, `(`, `)`, `<`, `>`, newlines, or `\`. Tell the user to rename or move the file to a plain path instead. Never pass such a path into a shell command.

Review step (read-only Codex):
- Run Codex in the foreground through the shared `task` runtime. Do NOT pass `--write`: Codex must only read and critique the file, not edit it.
- Pass a prompt that names the file path and asks for specific, actionable revisions. Enclose the whole prompt in single quotes so the substituted path cannot be interpreted by the shell, and substitute the real path plus any `--model`/`--effort` flags the user gave:

```bash
node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task 'Read the plan/spec file at <file>. Review it as a planning document, not code. Return specific, actionable revisions: gaps, internal contradictions, unclear or ambiguous requirements, missing edge cases, and unstated assumptions. For each point, say what to change and why. Do not rewrite the whole document; do not praise. If the document is already sound, say so explicitly.'
```

- This runs foreground so the feedback is available to fold in this turn. Do not run it in the background.
- If the companion reports that Codex is missing or unauthenticated, stop and tell the user to run `/codex:setup`.
- If Codex was never successfully invoked, stop. Do not fabricate feedback or edit the file.

Fold-in step (Claude edits the file):
- Read the target file, then apply Codex's feedback to it with `Edit`.
- Incorporate each actionable point that genuinely improves the document. Preserve the file's existing structure, headings, and voice.
- If Codex reports no substantive issues, make no edits and say so.
- If you deliberately skip a piece of feedback, note it in the summary rather than silently dropping it.

Report:
- After editing, give a short summary of what changed and list any feedback you chose not to apply, with a one-line reason.
- Do not paste Codex's raw output verbatim; the point of this command is to act on it.
