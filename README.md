# Tracekit extension for Gemini CLI

> **Experimental:** Tracekit's Gemini CLI hooks are tested against recorded Gemini CLI hook payloads and a real
> Tracekit signer, not yet in live Gemini CLI sessions ([support](https://github.com/Cygnux-Labs/Tracekit/wiki/Support)).

Each tool call Gemini CLI makes goes to a local [Tracekit](https://github.com/Cygnux-Labs/Tracekit) signer before it
runs. The signer's policy allows it, blocks it (Gemini tells the model why), or holds it until you approve it from
another terminal (`tracekit approvals list`, then `tracekit approvals approve <id>`). The signer writes the call, the
decision, the result, your prompts and the session's start and end as signed, hash-chained records, which export as a
bundle anyone can verify offline: a tamper-evident record, with the limits [below](#limits).

The extension only carries the hooks. The signer, the policy and the verifier are the `tracekit-ai` Python package.

## Install

```bash
pip install tracekit-ai               # 1.4.2 or later
tracekit init --dev --v2 --no-hooks   # same-user dev signer (it starts on the first hook); the hooks come from here
gemini extensions install https://github.com/Cygnux-Labs/tracekit-gemini-extension
```

Restart Gemini CLI. After a session, `tracekit last` exports its run and verifies it (`--open` for a run that hasn't
ended); `tracekit view --dev` shows every run.

`--no-hooks` stops init from writing hooks for any agent. It still creates the dev signer's data directory, which tells
the extension that Tracekit is set up (see [when a hook can't run](#when-a-hook-cant-run)). Don't add
`--agent gemini`: without `--no-hooks` that writes init's own Gemini hooks (next section), and with it init creates
nothing at all.

System mode, where the signer runs as its own user (Linux; macOS needs `--experimental-macos`):
`sudo /usr/bin/python3 -m tracekit init --v2 --user <agent-user> --agent gemini --no-hooks`, then install the
extension as the agent's user. This combination is untested.

## The extension or `tracekit init --agent gemini`, not both

`tracekit init --dev --v2 --agent gemini` writes the same five hooks into `~/.gemini/settings.json`. Gemini CLI runs
those and the extension's side by side (it drops a hook only when its name and command repeat another's, and the
commands differ). Tested with both installed: every prompt, call and result is recorded twice, a denied call is denied
twice, and a held call asks for two approvals. Approving one isn't enough: the call runs only once both are approved,
and if one isn't, that hook blocks the call when it stops waiting (after 9 minutes). `tracekit doctor` doesn't notice.

If you ran init with `--agent gemini`, remove its hooks and keep the extension's:

```bash
tracekit uninstall --agent gemini   # removes only Tracekit's hooks from ~/.gemini/settings.json; records stay
```

## What is recorded

- Every tool call that goes through Gemini CLI's tool hooks: the tool (`run_shell_command` as `Bash`, `read_file` and
  `read_many_files` as `Read`, `write_file` as `Write`, `replace` as `Edit`, any other tool under its own name), the
  command, file path or URL in clear, file contents and outputs as hashes, the policy's decision and the result.
- Your prompt at each turn, and the session's start and end.
- What the model said before each hook: Gemini CLI's thought summaries and text from its chat recording.

In dev mode the signer also keeps each call's redacted arguments and result, encrypted, for 7 days, so
`tracekit view --dev` can show them; `TRACEKIT_DEV_CONTENT=0`, set before the dev signer starts, keeps none. Exported
bundles hold only the hashes. The hook talks only to the local signer and sends nothing off the machine. Field by
field: [privacy](https://github.com/Cygnux-Labs/Tracekit/wiki/Privacy).

Not recorded: the model's requests and responses themselves, and what a tool does beyond its arguments and its result
(a shell command's child processes, an MCP server's own work).

## What the hook script trusts

The hooks run `scripts/tracekit-hook`, a POSIX shell script, which runs Tracekit's own Gemini CLI hook
(`tracekit.integrations.harness_hooks`) for the event, as `tracekit init --agent gemini` does.

- **Which Python.** In system mode (`/etc/tracekit/client.json` exists) only the root-owned `/opt/tracekit/bin/python`.
  Otherwise the first that imports `tracekit` of: the Python the `tracekit` command on your `PATH` runs with (a pipx or
  venv install), `python3`, `python`. User site-packages are added back for `pip install --user`.
- **Python runs isolated** (`-I`): the project directory and `PYTHON*` variables can't put their code on its path.
- **Which signer.** Gemini CLI loads a project's `.env` into the environment its hooks get (it doesn't override
  variables already set). So the script drops `TRACEKIT_PYTHON`, `TRACEKIT_SIGNER` (and its `_TOKEN`, `_TOKEN_FILE`,
  `_CERT`, `_KEY`, `_CA`) and `TRACEKIT_RUNTIME_DIR`: the signer is the one the root-owned system config names, else your
  own dev signer.
- **What it can't control.** `PATH` is the one you started Gemini CLI with: a `.env` can't change it, but a relative or
  project directory on it would be searched. Other variables a project's `.env` sets still reach the hook: on Linux
  `XDG_DATA_HOME` moves where the dev signer keeps its keys and records, and a `BASH_ENV` file runs before every hook
  Gemini starts. Gemini's folder trust (`security.folderTrust.enabled`) stops untrusted folders' `.env` files from loading.

## When a hook can't run

- **A tool call** (BeforeTool) is blocked when its hook starts but can't finish, when the installed tracekit is older
  than 1.4.2 or its hook won't import, and when no Python has tracekit though Tracekit is set up (the dev signer's data
  directory exists). Before you set Tracekit up, every event shows a notice and runs, unrecorded.
- **Any other event** (a prompt, a tool's result, a session's start or end) is never blocked: its error shows as a
  warning and the event isn't recorded. The script drops the hook's output on such a failure, as tracekit 1.4.2 writes a
  deny there and Gemini CLI would act on it: discard your prompt, or hide the tool's result from the model.
- **System mode:** if `/opt/tracekit/bin/python` can't run the hook, tool calls are blocked (with `fail_mode: open` in
  `/etc/tracekit/client.json`, they run unrecorded, with a warning).
- **Timeouts:** Gemini CLI runs a call whose hook passes its timeout as if the hook had said nothing. BeforeTool's is
  600 s (Gemini counts milliseconds: `600000`), and Tracekit answers a held call within 540 s, so it blocks first.

## Limits

- macOS and Linux (WSL too). Gemini CLI runs hooks through PowerShell on Windows, which can't run this script: don't
  install the extension there; `tracekit init --dev --v2 --agent gemini` writes hooks Windows can run.
- Gemini CLI gives a tool call no id, so Tracekit pairs each BeforeTool with its AfterTool by tool and arguments, first
  in first out.
- Gemini CLI's whole-CLI sandbox (`-s`, `GEMINI_SANDBOX`, `tools.sandbox`) runs Gemini and its hooks inside the sandbox,
  where Tracekit's hooks don't work; use its tool sandbox (`security.toolSandboxing: true`) instead
  ([Gemini CLI's sandbox](https://github.com/Cygnux-Labs/Tracekit/wiki/Integrations-Gemini-Sandbox)).
- In dev mode the signer runs as your user, so an agent running as you could stop it, remove the extension, or rewrite
  the log and its keys (`tracekit doctor` says so: `D-PROCESS-BOUNDARY`). System mode puts the signer under its own
  user.
- A signer named by `TRACEKIT_SIGNER` (a remote one) isn't used: the script drops that variable (see above).
- `tracekit doctor` reads Claude Code's settings for hooks: with only this extension it warns `D-HOOKS-PRESENT` about
  `~/.claude/settings.json`, which you can ignore here, and it doesn't see this extension's hooks.

All of Tracekit's limits, and what to do about each: [limits](https://github.com/Cygnux-Labs/Tracekit/wiki/Limits).
How the Gemini CLI hooks work: [coding agents](https://github.com/Cygnux-Labs/Tracekit/wiki/Coding-Agents).

## Uninstall

`gemini extensions uninstall tracekit`. The signer and its records stay; `tracekit down` stops the dev signer.

## License

MIT, as Tracekit: [github.com/Cygnux-Labs/Tracekit](https://github.com/Cygnux-Labs/Tracekit).
