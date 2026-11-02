# Built-in Tools

This document describes the tools that ship with Mistral Vibe — the **builtin tools** — what each one does, the arguments it accepts, and the permission it requires by default.

Builtin tools live in `vibe/core/tools/builtins/`. Each tool is a Python class implementing `BaseTool` (`vibe/core/tools/base.py`); its tool name is the snake_case form of the class name (e.g. `ReadFile` → `read_file`), unless the class overrides `get_name()`. The description shown to the model is loaded from a matching prompt file in `vibe/core/tools/builtins/prompts/<tool_name>.md`.

## Permission model

Every tool has a default permission level (`ToolPermission`, see `vibe/core/tools/models.py`):

| Level | Meaning |
|---|---|
| `ALWAYS` | The tool runs without asking the user. |
| `ASK` | The user is asked for approval before each call (or per session, depending on the approval flow). |
| `NEVER` | The tool is disabled. |

The default can be changed per tool in the Vibe configuration. File-based tools (`read_file`, `grep`, `edit`, `write_file`) additionally declare `sensitive_patterns`: file patterns that trigger an `ASK` even when the tool is configured as `ALWAYS`. The defaults (`DEFAULT_SENSITIVE_PATTERNS` in `vibe/core/tools/utils.py`) cover dotenv files: `**/.env`, `**/.env.*`, `**/.env~`, `**/.envrc`, `**/.envrc.*`, `**/.envrc~`.

## Tool overview

| Tool | Default permission | Availability | Purpose |
|---|---|---|---|
| `read_file` | `ALWAYS` | Always | Read a file, with paging |
| `grep` | `ALWAYS` | Always | Regex search in file contents |
| `write_file` | `ASK` | Always | Create a new file |
| `edit` | `ASK` | Always | Exact string replacement in a file |
| `bash` | `ASK` | Non-Windows / any OS with the simple shell | Run a one-off shell command (stateless) |
| `bash` (managed shell), `bash_log_file` | `ASK` | Non-Windows with managed-shell support | Session-based shell (PTY, background, interactive input) — replaces `bash` when available |
| `bash_output`, `bash_stdin`, `bash_sessions` | `ALWAYS` | Non-Windows with managed-shell support | Companion tools to poll output, send input, and manage managed-shell sessions |
| `git_bash`, `git_bash_log_file` | `ASK` | Windows with Git Bash available | Git Bash shell on native Windows |
| `git_bash_output`, `git_bash_stdin`, `git_bash_sessions` | `ALWAYS` | Windows with Git Bash available | Git Bash session companions |
| `powershell`, `powershell_log_file` | `ASK` | Windows with PowerShell available | PowerShell shell on native Windows |
| `powershell_output`, `powershell_stdin`, `powershell_sessions` | `ALWAYS` | Windows with PowerShell available | PowerShell session companions |
| `web_search` | `ASK` | Requires a Mistral API key | Search the web, returns cited answers |
| `web_fetch` | `ASK` | Always | Fetch a URL and return its content as markdown |
| `ask_user_question` | `ALWAYS` | Always | Ask the user structured multiple-choice questions |
| `skill` | `ALWAYS` | Always | Load a skill by name |
| `task` | `ASK` | Always | Launch a subagent for a complex task |
| `todo` | `ALWAYS` | Always | Read or replace the session todo list |
| `exit_plan_mode` | `ALWAYS` | Plan mode only | Signal that the plan is ready for implementation |

## File tools

### `read_file`

Read a file. Use absolute paths. For large files, page through with `offset`/`limit` instead of reading everything; prefer `grep` to locate content. Binary and model-weight files (`.bin`, `.safetensors`, `.pt`, `.gguf`) must not be read.

Arguments:

- `file_path` (string, required): absolute path to the file.
- `offset` (integer, optional, 1-indexed): line number to start reading from.
- `limit` (integer, default `2000`): maximum number of lines to read.

Output is capped by `max_read_bytes` (config). Line numbers are prefixed in the output — never include them when copying text into `edit`.

### `grep`

Search file contents with a regex (ripgrep-backed). Returns matching file paths, line numbers, and lines.

Arguments:

- `pattern` (string, required): the regex pattern.
- `path` (string, default `.`): file or directory to search in; defaults to the current working directory.
- `max_matches` (integer, optional): override the default maximum number of matches.
- `use_default_ignore` (boolean, default `true`): respect `.gitignore` and `.ignore` files.

Output is capped by `max_output_bytes` (config).

### `write_file`

Create a new file. **Errors if the file already exists** — use `edit` to modify existing files. Prefer editing existing files over creating new ones, and do not proactively create documentation or README files.

Arguments:

- `file_path` (string, required): absolute path of the file to write.
- `content` (string, required): the content to write.

### `edit`

Exact string replacement in a file. You must `read_file` the file first. `old_string` must match exactly (including whitespace) and occur exactly once, unless `replace_all` is set — otherwise provide more context to make it unique. If an edit fails, re-read the file before retrying.

Arguments:

- `file_path` (string, required): absolute path of the file to modify.
- `old_string` (string, required): the text to replace.
- `new_string` (string, required): the replacement text (must differ from `old_string`).
- `replace_all` (boolean, default `false`): replace all occurrences of `old_string`.

## Shell tools

### `bash` (simple, stateless)

Execute a shell command and return its output. Each call starts a **fresh shell** from the user's profile — working directory, environment variables, and shell functions do **not** persist between calls, so prefer absolute paths. There is no background execution; avoid long-running or blocking processes.

Arguments:

- `command` (string, required): the shell command to execute.
- `timeout` (integer, optional, seconds): override the default timeout (300 s, configurable via `default_timeout`). The command is killed when the timeout expires.

Returns `command`, `shell`, `exit_code`, `stdout`, `stderr` (separate streams, capped by `max_output_bytes`).

The command runs under a permission model (allow / deny / ask) with allow/deny lists configured per tool (`allowlist`, `denylist`); a denied command means the user declined it — adjust your approach rather than retrying verbatim. Git guidance: interactive flags (`-i`) are unsupported, use the `gh` CLI for GitHub operations, commit or push only when the user asks, and do not append a co-authoring footer (the signature is added automatically).

### Session-based shells (managed shell)

On platforms where the managed-shell backend is supported, the simple `bash` is replaced by a **stateful session shell** with the same `bash` name, plus four companion tools. Sessions run in a PTY and keep a durable log file under `~/.vibe/shell-tool/sessions/`.

- **`bash`** — run a command in a session. Each call starts a **new** session; the returned `session_id` is then used with the companion tools below (`bash_output`, `bash_stdin`, `bash_sessions`). Key arguments: `command`, `timeout_seconds`, `background` (keep long-running processes alive, e.g. dev servers), `hard_timeout` (kill the process group on timeout instead of returning a live session), plus `timeout`, `cwd`, `env`, `shell`. A soft foreground timeout returns a live session you can poll later.
- **`bash_output`** — long-poll a session's output: `session_id`, `cursor` (byte offset into the log; pass the `next_cursor` from the previous call to resume without re-reading), `wait_seconds`, `max_bytes`. Returns when the process exits, the output cap is hit, the session is killed/reset, or the wait window expires.
- **`bash_stdin`** — send input to a session: `session_id` plus exactly one of `text` (e.g. `"y\n"` to answer a prompt), `control` (e.g. `["ctrl_c"]` to interrupt), or `bytes_base64` (raw bytes, base64-encoded).
- **`bash_sessions`** — manage sessions: `action` = `list` | `inspect` | `kill` | `reset` (with `session_id` for `inspect`/`kill`; `reset(clear_logs=true)` also deletes stored logs).
- **`bash_log_file`** — read (`action="read"`) a session's full output file; `write`/`append` annotate it once the session has exited.

The same family exists under Windows-specific names, resolved automatically:

- **`git_bash`**, `git_bash_output`, `git_bash_stdin`, `git_bash_sessions`, `git_bash_log_file` — Git Bash (POSIX syntax) on native Windows, when a usable `bash.exe` is found. The simple fallback variant runs each command in a fresh stateless Git Bash process.
- **`powershell`, `powershell_output`, `powershell_stdin`, `powershell_sessions`, `powershell_log_file` — PowerShell (`pwsh.exe`, then `powershell.exe`) on native Windows; `cmd.exe` is never used.

In all shell variants, prefer the dedicated tools over shell utilities: `read_file` instead of `cat`/`head`/`tail`, `grep` instead of `grep`/`find`/`rg`, `edit`/`write_file` instead of `sed`/redirection.

## Web tools

### `web_search`

Search the web for current information. Returns answers with cited sources. Use for recent events, updated docs, and version checks.

Arguments:

- `query` (string, required): the search query.

Configuration: `timeout` (default 120 s) and the Mistral model used for search. Only available when a Mistral API key is configured (otherwise the tool is hidden).

### `web_fetch`

Fetch content from a URL, returned as markdown. Read-only.

Arguments:

- `url` (string, required): the URL to fetch.
- `timeout` (integer, optional, seconds, max 120): override the default timeout (30 s, configurable via `default_timeout`).

Returned content is capped by `max_content_bytes` (default 120 000 bytes).

## Interaction tools

### `ask_user_question`

Ask the user one or more structured questions with options. An "Other" free-text option is added automatically to each question.

Arguments:

- `questions` (array, required, at least 1 item; 1–4 recommended). Each question has:
  - `question` (string, required): the question text.
  - `header` (string, optional, max 20 chars): short chip/tag label.
  - `options` (array, required, at least 2 items; 2–4 recommended, not counting the automatic "Other"): each with a `label` and an optional `description`.
  - `multi_select` (boolean, default `false`): allow selecting multiple options.
  - `hide_other` (boolean, default `false`): hide the free-text "Other" option.
- `footer_note` (string, optional): subtle note displayed at the bottom of the question widget.

Returns the list of answers (each with the original question, the answer, and whether it was a custom "Other" answer).

### `skill`

Load a specialized skill by name (from `available_skills`) and follow the returned instructions step by step.

Arguments:

- `name` (string, required): the name of the skill.

Returns the skill name, the full skill content block, and optionally the absolute path of the skill directory.

### `task`

Launch a subagent for complex multi-step work. Provide a self-contained description; the subagent runs read-only and returns a final message — summarize it to the user yourself.

Arguments:

- `task` (string, required): the task for the subagent to perform.
- `agent` (string, default `"explore"`): the type of specialized subagent to use.

Returns the subagent's final message and execution metadata (turns used, whether it completed).

### `todo`

Track tasks for the current session. Use for multi-step work: write the list as soon as the steps are known, then keep it current as work progresses — after each step finishes, mark it completed and the next one `in_progress`. `action: "write"` replaces the **full** list, so include every item you want to keep.

Arguments:

- `action` (string, required): `read` to view the current list, or `write` to replace it.
- `todos` (array, required when `action="write"`): items with `id` (stable unique identifier, reused across updates), `content`, `status` (`pending` | `in_progress` | `completed` | `cancelled`), and `priority` (`low` | `medium` | `high`).

The list is capped at `max_todos` (default 100).

### `exit_plan_mode`

Signal that your plan is complete and ready for implementation. Call only after finalizing the plan — not before, and not if the user wants to keep planning. When the user chooses to proceed, they are offered options to auto-approve or keep requesting approval for edits.

Arguments: none.

## Related documentation

- [ADR 0004 — Typed Permissioned Tools](../../adr/0004-typed-permissioned-tools.md) — tool contracts, permissions, and output.
- [ADR 0007 — Extension Mechanisms](../../adr/0007-extension-mechanisms.md) — skills, subagents, and custom tools alongside the builtins.
- Tool source code: `vibe/core/tools/builtins/`; model-facing descriptions: `vibe/core/tools/builtins/prompts/`.
- [Interactive explorer](interactive/index.html) — a single-file interactive site (searchable tool cards, quiz, permission game) built from this document. It is a single static file with no dependencies: either open `docs/understanding_vibe/builtins/interactive/index.html` directly in a browser (double-click or `file://` URL), or serve the repo root with `uv run --no-project python -m http.server 8000` and open `http://localhost:8000/docs/understanding_vibe/builtins/interactive/`. Note: plain `uv run python -m http.server` first builds the whole project and can hang for minutes before serving anything — for a static file server, always use `--no-project`.
