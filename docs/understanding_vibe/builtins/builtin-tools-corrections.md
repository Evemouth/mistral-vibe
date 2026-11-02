# Corrections applied to `builtin-tools.md`

This document records the corrections made to [builtin-tools.md](builtin-tools.md) after checking it against the source code (`vibe/core/tools/`). Each entry shows the before, the after, and the code reference that justifies the correction.

## 1. Broken ADR links

- **Before**: `[ADR 0004](adr/0004-typed-permissioned-tools.md)` and `[ADR 0007](adr/0007-extension-mechanisms.md)` — relative paths invalid from `docs/understanding_vibe/builtins/`.
- **After**: `../../adr/0004-typed-permissioned-tools.md` and `../../adr/0007-extension-mechanisms.md` (links verified: both target files exist).
- **Why**: the document sits two levels below `docs/`; the ADRs live in `docs/adr/`.

## 2. Default permissions of the session-based shell family

- **Before**: the overview table listed `ASK` for `bash`, `bash_output`, `bash_stdin`, `bash_sessions`, `bash_log_file` (and likewise for `git_bash_*` / `powershell_*`).
- **After**: `bash` (managed) and `bash_log_file` remain `ASK`; `bash_output`, `bash_stdin`, `bash_sessions` are `ALWAYS`; same for the `git_bash_output/stdin/sessions` and `powershell_output/stdin/sessions` variants.
- **Code reference**: `vibe/core/tools/builtins/experimental_bash.py:1248-1275` — `BashOutputConfig`, `BashStdinConfig`, and `BashSessionsConfig` have `permission = ALWAYS`; only `BashLogFileConfig` has `ASK`. The Windows classes (`git_bash.py`, `windows_shell.py`) inherit these same configs. The `ToolManager` only overrides the default when an explicit override is configured (`vibe/core/tools/manager.py:754-759`).

## 3. `session_id` argument of session-based `bash`

- **Before**: the doc listed `session_id` (reuse a session) among `bash` arguments.
- **After**: each `bash` call starts a **new** session; the returned `session_id` is then used with the companion tools (`bash_output`, `bash_stdin`, `bash_sessions`). The actual arguments are listed: `command`, `timeout_seconds`, `background`, `hard_timeout`, `timeout`, `cwd`, `env`, `shell`.
- **Code reference**: `ExperimentalBashArgs` (`experimental_bash.py:1283-1308`) has no `session_id` field.

## 4. `bash_stdin`: missing `bytes_base64` argument

- **Before**: only `text` and `control` were documented.
- **After**: `session_id` plus exactly one of `text`, `control`, or `bytes_base64` (raw bytes, base64-encoded).
- **Code reference**: `BashStdinArgs` (`experimental_bash.py:1355-1382`), with the validator "exactly one of text, control, or bytes_base64".

## 5. `sensitive_patterns`: "private keys" was inaccurate

- **Before**: "e.g. `.env` files, private keys".
- **After**: the default list only covers dotenv files: `**/.env`, `**/.env.*`, `**/.env~`, `**/.envrc`, `**/.envrc.*`, `**/.envrc~`.
- **Code reference**: `DEFAULT_SENSITIVE_PATTERNS` in `vibe/core/tools/utils.py:36-43`.

## 6. `ask_user_question`: bounds presented as strict

- **Before**: "`questions` (array, required, 1–4 items)" and "`options` (array, required, 2–4 items)".
- **After**: strict minimum of at least 1 question / 2 options; the upper bounds (4 / 4) are recommendations; the automatic "Other" option is not counted in the 2–4.
- **Code reference**: `vibe/questions.py:28-55` — `min_length=1` on `questions`, `min_length=2` on `options`, no `max_length`; the descriptions say "1-4 recommended" / "2-4 recommended, not including 'Other'".
