# localdex plugin for Claude Code

Use **localdex** — a Codex CLI fork pinned to a local model served by [LM Studio](https://lmstudio.ai) — from inside Claude Code for code reviews or to delegate tasks.

This is a fork of the official Codex plugin, rewired to invoke the `localdex` binary (no OpenAI account, API key, or network usage — everything runs against your local LM Studio server).

<video src="./docs/plugin-demo.webm" controls muted playsinline autoplay></video>

## Why this fork

The upstream plugin ([openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc)) shells out to the global `codex` binary, which by default talks to the OpenAI API and requires a ChatGPT login or API key. This fork targets [`localdex`](https://github.com/DCSdevelop/codexforlocal) instead — a Codex CLI build pinned to a local LM Studio server — so reviews and delegated tasks run entirely on your own machine, on your own model, with no account and no data leaving the box.

What changed relative to upstream:

- The invoked binary is a single constant, `CODEX_BIN` in `plugins/codex/scripts/lib/process.mjs` — default `localdex`, overridable with the `CODEX_COMPANION_BIN` env var.
- The plugin identity is `localdex@localcodex` (commands are `/localdex:*`), so it can be installed alongside the official `codex` plugin without conflicts.
- Setup and error messages point at the `install-localdex.sh` installer instead of `npm install -g @openai/codex`, and no login step is required: localdex's `lmstudio` provider reports ready without OpenAI authentication.
- Everything else — the app-server protocol, review gate, background jobs, session transfer — is unchanged from upstream, which keeps the fork easy to rebase.

## What You Get

- `/localdex:review` for a normal read-only Codex review
- `/localdex:adversarial-review` for a steerable challenge review
- `/localdex:rescue`, `/localdex:transfer`, `/localdex:status`, `/localdex:result`, and `/localdex:cancel` to delegate work, hand off sessions, and manage background jobs

## Requirements

- **The `localdex` binary** — built from the sibling `codexforlocal` repo: run `scripts/install-localdex.sh` there (needs a Rust toolchain). No OpenAI account or API key required.
- **LM Studio** with its server running (`lms server start`), your model downloaded, and **JIT model loading enabled** (the plugin's app-server path does not auto-load models). The server must support the OpenAI Responses API (`POST /v1/responses`).
- **Node.js 18.18 or later**

## Install

Install the `localdex` binary (from the codexforlocal repo checkout):

```bash
/Users/master/Documents/antigravity/068-CodexForLocal/codexforlocal/scripts/install-localdex.sh
```

Add this repo as a local marketplace in Claude Code (in-app, or `claude plugin marketplace add <path>` from a shell):

```bash
/plugin marketplace add /Users/master/Documents/antigravity/068-CodexForLocal/localcodex-plugin-cc
```

Install the plugin:

```bash
/plugin install localdex@localcodex
```

Reload plugins (or restart Claude Code):

```bash
/reload-plugins
```

Then run:

```bash
/localdex:setup
```

`/localdex:setup` will tell you whether localdex is ready — with LM Studio configured it reports ready with no login step. If localdex is missing, it can offer to run the install script for you (see `install-localdex.sh` above).

To update after editing this repo: `npm run bump-version`, then `claude plugin marketplace update localcodex && claude plugin update localdex`, and restart Claude Code.

Notes:
- The model comes from `~/.localdex/config.toml` — don't pass `--model` to the commands (the `spark` alias and other Codex cloud model names don't exist locally).
- To point at a non-default LM Studio address, set `CODEX_OSS_BASE_URL` before launching Claude Code.
- To make the plugin use a different binary name, set `CODEX_COMPANION_BIN` (defaults to `localdex`).

After install, you should see:

- the slash commands listed below
- the `localdex:codex-rescue` subagent in `/agents`

One simple first run is:

```bash
/localdex:review --background
/localdex:status
/localdex:result
```

## Usage

### `/localdex:review`

Runs a normal Codex review on your current work. It gives you the same quality of code review as running `/review` inside Codex directly.

> [!NOTE]
> Code review especially for multi-file changes might take a while. It's generally recommended to run it in the background.

Use it when you want:

- a review of your current uncommitted changes
- a review of your branch compared to a base branch like `main`

Use `--base <ref>` for branch review. It also supports `--wait` and `--background`. It is not steerable and does not take custom focus text. Use [`/localdex:adversarial-review`](#codexadversarial-review) when you want to challenge a specific decision or risk area.

Examples:

```bash
/localdex:review
/localdex:review --base main
/localdex:review --background
```

This command is read-only and will not perform any changes. When run in the background you can use [`/localdex:status`](#codexstatus) to check on the progress and [`/localdex:cancel`](#codexcancel) to cancel the ongoing task.

### `/localdex:adversarial-review`

Runs a **steerable** review that questions the chosen implementation and design.

It can be used to pressure-test assumptions, tradeoffs, failure modes, and whether a different approach would have been safer or simpler.

It uses the same review target selection as `/localdex:review`, including `--base <ref>` for branch review.
It also supports `--wait` and `--background`. Unlike `/localdex:review`, it can take extra focus text after the flags.

Use it when you want:

- a review before shipping that challenges the direction, not just the code details
- review focused on design choices, tradeoffs, hidden assumptions, and alternative approaches
- pressure-testing around specific risk areas like auth, data loss, rollback, race conditions, or reliability

Examples:

```bash
/localdex:adversarial-review
/localdex:adversarial-review --base main challenge whether this was the right caching and retry design
/localdex:adversarial-review --background look for race conditions and question the chosen approach
```

This command is read-only. It does not fix code.

### `/localdex:rescue`

Hands a task to Codex through the `localdex:codex-rescue` subagent.

Use it when you want Codex to:

- investigate a bug
- try a fix
- continue a previous Codex task
- take a faster or cheaper pass with a smaller model

> [!NOTE]
> Depending on the task and the model you choose these tasks might take a long time and it's generally recommended to force the task to be in the background or move the agent to the background.

It supports `--background`, `--wait`, `--resume`, and `--fresh`. If you omit `--resume` and `--fresh`, the plugin can offer to continue the latest rescue thread for this repo.

Examples:

```bash
/localdex:rescue investigate why the tests started failing
/localdex:rescue fix the failing test with the smallest safe patch
/localdex:rescue --resume apply the top fix from the last run
/localdex:rescue --effort medium investigate the flaky integration test
/localdex:rescue --background investigate the regression
```

You can also just ask for a task to be delegated to Codex:

```text
Ask Codex to redesign the database connection to be more resilient.
```

**Notes:**

- do not pass `--model` — the model comes from `~/.localdex/config.toml`, and Codex cloud model names (`spark`, `gpt-5.4-mini`, …) do not exist on your LM Studio server. `--effort` works normally.
- follow-up rescue requests can continue the latest Codex task in the repo

### `/localdex:transfer`

Creates a persistent Codex thread from the current Claude Code session and prints a `localdex resume <session-id>` command.

Use it when you started a debugging or implementation conversation in Claude Code and want to continue that same context directly in Codex.

Examples:

```bash
/localdex:transfer
/localdex:transfer --source ~/.claude/projects/-Users-me-repo/<session-id>.jsonl
```

The plugin's existing `SessionStart` hook supplies the current transcript path automatically; `--source` is available as a manual override. The transfer uses Codex's external-agent session importer, so it follows the same conversion rules as importing Claude history in the Codex App and creates visible turns that can be continued in the App or TUI. The source must be under `~/.claude/projects`, and older Codex versions that do not expose session import must be upgraded before using this command.

### `/localdex:status`

Shows running and recent Codex jobs for the current repository.

Examples:

```bash
/localdex:status
/localdex:status task-abc123
```

Use it to:

- check progress on background work
- see the latest completed job
- confirm whether a task is still running

### `/localdex:result`

Shows the final stored Codex output for a finished job.
When available, it also includes the Codex session ID so you can reopen that run directly in Codex with `localdex resume <session-id>`.

Examples:

```bash
/localdex:result
/localdex:result task-abc123
```

### `/localdex:cancel`

Cancels an active background Codex job.

Examples:

```bash
/localdex:cancel
/localdex:cancel task-abc123
```

### `/localdex:setup`

Checks whether Codex is installed and authenticated.
If Codex is missing and npm is available, it can offer to install Codex for you.

You can also use `/localdex:setup` to manage the optional review gate.

#### Enabling review gate

```bash
/localdex:setup --enable-review-gate
/localdex:setup --disable-review-gate
```

When the review gate is enabled, the plugin uses a `Stop` hook to run a targeted Codex review based on Claude's response. If that review finds issues, the stop is blocked so Claude can address them first.

> [!WARNING]
> The review gate can create a long-running Claude/Codex loop and may drain usage limits quickly. Only enable it when you plan to actively monitor the session.

## Typical Flows

### Review Before Shipping

```bash
/localdex:review
```

### Hand A Problem To Codex

```bash
/localdex:rescue investigate why the build is failing in CI
```

### Start Something Long-Running

```bash
/localdex:adversarial-review --background
/localdex:rescue --background investigate the flaky test
```

Then check in with:

```bash
/localdex:status
/localdex:result
```

## Codex Integration

The plugin wraps the [Codex app server](https://developers.openai.com/codex/app-server). It uses the `localdex` binary installed in your environment and applies localdex's own configuration.

### Common Configurations

If you want to change the default reasoning effort or the default model that gets used by the plugin, you can define that inside your user-level or project-level `config.toml`. For example to always use a specific LM Studio model on `high` for a specific project you can add the following to a `.codex/config.toml` file at the root of the directory you started Claude in (the model id must match `lms ls`):

```toml
model = "openai/gpt-oss-20b"
model_reasoning_effort = "high"
```

Your configuration will be picked up based on:

- user-level config in `~/.localdex/config.toml` (localdex uses an isolated `CODEX_HOME`; override with `LOCALDEX_HOME`)
- project-level overrides in `.codex/config.toml`
- project-level overrides only load when the [project is trusted](https://developers.openai.com/codex/config-advanced#project-config-files-codexconfigtoml)

Check out the Codex docs for more [configuration options](https://developers.openai.com/codex/config-reference).

### Moving The Work Over To Codex

Delegated tasks and any [stop gate](#what-does-the-review-gate-do) run can also be directly resumed inside Codex by running `localdex resume` either with the specific session ID you received from running `/localdex:result` or `/localdex:status` or by selecting it from the list.

This way you can review the Codex work or continue the work there.

## FAQ

### Do I need a Codex or OpenAI account for this plugin?

No. localdex talks to your local LM Studio server (`lmstudio` provider, no OpenAI authentication). Run `/localdex:setup` to check readiness — it should report the "LM Studio" provider with no login step. Just make sure the LM Studio server is running and your model is available.

### Does the plugin use a separate Codex runtime?

No. This plugin delegates through your local [Codex CLI](https://developers.openai.com/codex/cli/) and [Codex app server](https://developers.openai.com/codex/app-server/) on the same machine.

That means:

- it uses the same Codex install you would use directly
- it uses the same local authentication state
- it uses the same repository checkout and machine-local environment

### Will it use the same Codex config I already have?

It sees exactly the config the `localdex` binary itself uses: `~/.localdex/config.toml` plus trusted project-level `.codex/config.toml` overrides — see [configuration](#common-configurations). Note this is deliberately isolated from any stock Codex config in `~/.codex`, so an existing `codex` setup is neither used nor affected.

### Can I point it at a different LM Studio address?

Yes. The built-in `lmstudio` provider defaults to `http://localhost:1234/v1`; set `CODEX_OSS_BASE_URL` (or `CODEX_OSS_PORT`) in the environment before launching Claude Code to point elsewhere.
