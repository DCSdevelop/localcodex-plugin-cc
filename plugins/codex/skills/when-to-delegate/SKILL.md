---
name: when-to-delegate
description: Decide whether to hand a task to localdex (the free, private, local LM Studio model) instead of doing it in Claude. Consult BEFORE delegating anything to localdex, and whenever a task is small, self-contained, token-heavy-but-simple, privacy-sensitive, or the user asks to save tokens / work offline / "use localdex" without a specific task.
user-invocable: false
---

# When to delegate to localdex

localdex runs a local coding model (LM Studio, no cloud, no cost per token). Delegating to it
spends *its* tokens instead of Claude's — but it is far slower and weaker than Claude.
Measured on this machine (M5 Pro, qwen3-coder-30b): ~26–29 tok/s generation,
~1.4k tok/s prompt ingestion, and every localdex turn re-ingests a large system prompt
(expect several seconds of latency per turn before the first token).

## Decision rubric

Delegate to localdex when ALL of these hold:
- The task is **self-contained**: one clear instruction, no back-and-forth, no tight feedback loop.
- The answer does not need Claude-level reasoning: summaries, explanations, boilerplate,
  config drafts, docstrings, commit-message drafts, small standalone scripts, second opinions.
- Claude's own tools would NOT answer it near-instantly (see anti-patterns).
- Latency is acceptable: a localdex turn takes tens of seconds to minutes.

Strong extra reasons to delegate even borderline tasks:
- The content is **privacy-sensitive** and should not leave the machine.
- The user wants to **save Claude tokens/quota**, or is rate-limited, or offline.
- The output would be **long but mechanical** (e.g. "summarize each file in this folder",
  "write exhaustive boilerplate") — big token spend, little reasoning.
- The user explicitly asks for localdex/Codex.

## Anti-patterns — do NOT delegate these

- **Literal `grep`/`find`/file-lookup work.** Claude's Grep/Glob/Read tools answer in
  milliseconds and cost almost nothing; routing a text search through a 27 tok/s LLM turns a
  1-second answer into minutes. Delegate only the *interpretation* layer ("read these matches
  and summarize the pattern"), never the search itself.
- Multi-step edits that need review between steps, or anything Claude must verify anyway.
- Tasks requiring large repeated context ingestion (whole-repo exploration is the slow path:
  every tool call adds thousands of tokens the local model must re-read).
- Anything where correctness is critical and hard to check — the local model drifts on
  long contexts; keep its tasks short and its prompts pointed.

## How to delegate

Follow the `/localdex:rescue` flow: invoke the `localdex:codex-rescue` subagent with the raw
request (it calls `codex-companion.mjs task`). Read-only is the default; `--write` only when
the task must edit files. Prefer `--background` for anything expected to run over a minute,
then collect with `/localdex:status` and `/localdex:result`. Keep prompts short, pointed, and
self-contained — name exact files and the exact deliverable, and say
"do not explore the repository" when knowledge alone suffices.
