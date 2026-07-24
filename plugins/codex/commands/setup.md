---
description: Check whether the local Codex CLI is ready and optionally toggle the stop-time review gate
argument-hint: '[--enable-review-gate|--disable-review-gate]'
allowed-tools: Bash(node:*), Bash(npm:*), AskUserQuestion
---

Run:

```bash
node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" setup --json $ARGUMENTS
```

If the result says localdex is unavailable:
- Use `AskUserQuestion` exactly once to ask whether Claude should install localdex now.
- Put the install option first and suffix it with `(Recommended)`.
- Use these two options:
  - `Install localdex (Recommended)`
  - `Skip for now`
- If the user chooses install, locate their codexforlocal checkout (clone https://github.com/DCSdevelop/codexforlocal if they don't have one) and run its install script (requires a Rust toolchain):

```bash
<path-to-codexforlocal>/scripts/install-localdex.sh
```

- Then rerun:

```bash
node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" setup --json $ARGUMENTS
```

If Codex is already installed or npm is unavailable:
- Do not ask about installation.

Output rules:
- Present the final setup output to the user.
- If installation was skipped, present the original setup output.
- If Codex is installed but not authenticated, preserve the guidance to run `!localdex login`.
