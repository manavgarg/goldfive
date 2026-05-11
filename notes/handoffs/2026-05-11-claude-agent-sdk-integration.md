# Handoff — claude-agent-sdk integration (2026-05-11)

Personal session log. Lives on the `personal/handoffs` branch of
`manavgarg/goldfive`, never merged. Paste into a fresh Claude Code
session as bootstrap context if you want to resume this work on a
different machine.

## Where we are

- **PR open**: <https://github.com/pedapudi/goldfive/pull/378> against
  `pedapudi/goldfive` `main`. Branch `feat/claude-agent-sdk-callable`
  on the `manavgarg/goldfive` fork.
- **Status**: Phase A merged-ready. Phase B/C in working-copy as
  unstaged changes on `feat/claude-agent-sdk-callable`.
- **Backups of the full BaseLlm + tool-translation code** also at
  `/tmp/goldfive_claude_sdk_full.py` and `/tmp/goldfive_agent_py_full.py`
  on the Mac where the session ran (volatile — `/tmp` clears on reboot).

## Project context

We're integrating `claude-agent-sdk` into goldfive so consumers can
run the orchestration LLM calls (planner, goal-deriver, judges) and
optionally the agent subagents through Anthropic's Max-billed `claude`
CLI login — no `ANTHROPIC_API_KEY` needed.

Working tree: `~/Desktop/claude/goldfive`. Adjacent harmonograf clone
at `~/Desktop/claude/harmonograf` (with goldfive + adk-python
submodules pulled into `third_party/`).

## What's in the PR

`goldfive/integrations/claude_sdk.py` with `make_call_llm` — a thin
`(system, prompt, model) -> str` callable around
`claude_agent_sdk.query`. Drop-in for:

- `LLMPlanner(call_llm=...)`
- `LLMGoalDeriver(call_llm=...)`
- `goldfive.wrap(call_llm=...)` (judge fallback)

Plus the `presentation_agent` example gains a
`GOLDFIVE_USE_CLAUDE_SDK=1` env-gated CLI branch that wires those
three call sites. Subagent model goes through ADK's native routing
(litellm / Gemini direct).

Also a related bug fix: `_build_app()` (adk-web factory) was only
attaching `HarmonografTelemetryPlugin`, never `HarmonografSink`. The
goldfive event stream (plans / drifts / task transitions) wasn't
reaching the harmonograf server from adk-web runs. Fixed by mirroring
the CLI `_run()` sink wiring.

## What's NOT in the PR (in working copy)

A full `ClaudeAgentSDKLlm` ADK `BaseLlm` adapter with tool translation
via claude-agent-sdk's MCP transport. Working — completes the
presentation example end-to-end on pure Claude — but it loses
goldfive observability of individual tool calls because tools execute
inside the adapter rather than via ADK's normal pipeline. That's a
goldfive-semantics regression (`CONFABULATION_RISK`,
`CAPABILITY_MISMATCH` drift detectors misfire), so we held it back.

The code is in working copy at:

- `goldfive/integrations/claude_sdk.py` — adds `ClaudeAgentSDKLlm`,
  `_extract_system_instruction`, `_flatten_contents_to_prompt`,
  `_build_input_schema_from_signature`, `_adk_tool_to_sdk_tool`,
  `make_claude_agent_sdk_llm_class` factory, lazy `__getattr__`.
- `examples/presentation_agent/agent.py` — the `use_claude_sdk` branch
  also wraps with `ClaudeAgentSDKLlm` when the model starts with
  `claude-`.

## Architecture insights worth keeping

### Goldfive's three LLM call sites

| Site | What sets the callable | What sets the model |
|---|---|---|
| Planner + Goal-deriver | `LLMPlanner(call_llm=…)`, `LLMGoalDeriver(call_llm=…)` | `model=` arg |
| Judges (goal-drift, reasoning) | Implicitly via `goldfive.wrap(call_llm=…)` fallback OR `JudgeConfig` | `GOLDFIVE_JUDGE_MODEL` env / explicit `JudgeConfig` |
| ADK subagents | `Agent(model=…)` — uses ADK's `BaseLlm` routing | model string passed to `Agent` constructor |

The `wrap(call_llm=)` precedence chain: **explicit > JudgeConfig >
detected**. Pass `call_llm=` and judges fall back to it for free.

### Plan / steering architecture

- Plan is a frozen `Plan` dataclass (`goldfive/types.py:447`). Every
  "edit" produces a new instance via `dataclasses.replace`. Live
  reference on `Session.plan` is atomically swapped.
- Plans are not append-only. Tasks within a plan can be replaced
  (REPLACE supersedes) or added as correction children (CORRECT
  supersedes). The event stream IS append-only.
- Agents see the plan via **dynamic instruction injection**
  (`goldfive/adapters/_adk_dynainst.py`). `install_dynamic_instructions`
  walks the tree, replaces each `LlmAgent`'s `instruction=` with a
  closure that resolves at turn-time: `<original prompt> + current
  task block + (optional) pending correction directive`. ADK's
  instruction-as-callable mechanism, NOT `before_model_callback`.
- The only `before_model_callback` use is for **Stream C cooperative
  cancel** on CRITICAL drift — short-circuits the next LLM dispatch.

### Drift detection

- **Synchronous detectors** (pattern-match on event content): fire
  every event, no LLM call, microseconds. E.g. `CONFABULATION_RISK`.
- **Async LLM judges** (`judge_goal_drift`, `reasoning_judge`): fire
  in the background concurrently with task execution. Two trigger
  paths: turn counter (every N agent turns) and task boundary
  (`mark_task_completed/failed/cancelled`, rate-limited to 10s
  minimum spacing). Spawn-and-detach pattern via `_background_judges`
  set, drained at shutdown. **They don't block task execution.**
- In short fast runs (mock-mode subagents), judges may get cancelled
  before they complete — that's where the "unparseable verdict /
  CancelledError" we saw came from.

### Force-drive vs Observe execution modes

- **Force-drive** (`SequentialExecutor(max_task_invocations=8)` in the
  CLI driver): goldfive directly invokes `task.assignee_agent_id` for
  each task. Coordinator's LLM never decides. Headless-friendly,
  deterministic. Constrains agent tree agency at the top level
  (within each invocation the agent is still free).
- **Observe** (default ADK executor in `adk web`): coordinator's LLM
  decides what to call. Goldfive plugin watches via callbacks and
  infers `current_task_id` via a "pin ladder"
  (`_adk_plugin.py:_pin_current_task_id_for_agent`):
  1. Per-invocation pinned task_id
  2. Latest `delegation_observed` task_id
  3. `OrchestrationStore.pin_current_task()` live pin
  4. `session.state["goldfive.current_task_id"]`
  5. Agent's last `report_task_started/completed` call

### claude-agent-sdk = Claude Code = agent runtime, not thin API

The bundled `claude` CLI runs a full agent loop by default. To coerce
into thin LLM behaviour:

1. `setting_sources=[]` — SDK isolation mode. No filesystem settings,
   no CLAUDE.md auto-detection. Documented at
   `claude_agent_sdk/types.py:1793-1803`.
2. `tools=[allowlist]` — **NOT** `allowed_tools=[]`. The former is the
   base-set allowlist; the latter only controls permission prompts.
   With `tools=["mcp__adk__<name>", ...]`, Claude Code's built-in
   tools (`TodoWrite`, `Task`, `Read`, `Bash`, …) are excluded → the
   internal agent loop dies.

Without those, agent invocations yielded 40+ messages of internal
`TaskStarted` / `TaskProgress` noise and 30-60s per call. With them,
5 messages, ~10s, plain text in/out.

For `make_call_llm` (planner/judges), neither knob is needed because
short structured prompts converge in 1-2 internal turns regardless.

### Why the BaseLlm adapter is hard to do "right"

ADK's `BaseLlm` contract is **transcript-driven**: every
`generate_content_async` call delivers the full conversation
including past `function_call` / `function_response` pairs. The
adapter is supposed to return the model's next `function_call` (not
execute it); ADK runs the tool and re-invokes with
`function_response`.

`claude-agent-sdk` has no "given this transcript, return next message"
API. Both `query()` and `ClaudeSDKClient` only accept user-message
streams. The two shapes don't compose without either:

- A stateful `ClaudeSDKClient` per ADK conversation, kept alive
  across `generate_content_async` calls, fed user/tool_result
  messages. Needs a registry keyed on a stable per-conversation id.
  Estimated 200-300 lines.
- Hacky text-encoded conversation replay each call.
- Switch to `anthropic` SDK direct — gives
  `messages.create(messages=[...])`, maps 1:1 to ADK pattern in ~50
  lines. Loses Max billing for these calls only (API credits ~$0.01/run on Haiku).

## Next steps (what to do when resuming)

1. **Strategy 1 spike** — 2-hour budget. Implement stateful
   `ClaudeSDKClient` per ADK conversation in `ClaudeAgentSDKLlm`:
   - Module-level dict keyed on (something stable from `LlmRequest`
     identifying the conversation — `previous_interaction_id`,
     conversation hash, or first content's id)
   - First call: instantiate `ClaudeSDKClient`, send initial user
     prompt
   - Subsequent calls: extract NEW `function_response` parts from
     `LlmRequest.contents`, translate to `tool_result` blocks, send
     via `client.query(...)` 
   - On each call, iterate the message stream and capture the FIRST
     `ToolUseBlock` in an `AssistantMessage`
   - Return that tool_use as ADK `function_call` Part in
     `LlmResponse` — ADK runs the tool via its normal pipeline
     (goldfive observes), then comes back with `function_response`
   - Lifecycle: close clients when ADK session ends. Look for
     hooks; might need a finalizer.
   - Caveat noted in client.py:58-64 — can't cross async runtime
     contexts. ADK runs the adapter on a single asyncio loop, so
     this should be fine, but verify.

2. **If the spike doesn't converge cleanly** → fall back to
   `anthropic` SDK direct for the BaseLlm adapter. Trivial
   implementation. Planner / judges stay on `claude-agent-sdk` →
   Max via this PR's `make_call_llm`.

3. **Either way, the BaseLlm adapter PR** would be a follow-up to
   PR #378 — different branch, separate review.

## Key dead-ends to avoid relearning

- `permission_mode='plan'` triggers Claude Code's project-init
  pathway. Produces a CLAUDE.md template instead of answering. Don't
  use it for thin-LLM coercion.
- `disallowed_tools=['Task', 'TodoWrite', …]` only partially
  suppresses the agent loop. The actual disable is `tools=[]` /
  `tools=[allowlist]`.
- `claude-agent-sdk` spawns a fresh `claude` CLI subprocess per
  query. With `setting_sources=[]` these sessions don't appear in
  your interactive `claude` history — good for library use.
- gRPC fork warnings (`fork_posix.cc:71: Other threads are currently
  calling into gRPC, skipping fork() handlers`) are noise from
  harmonograf-client's gRPC channel. Looks scary but doesn't break
  anything.
- Gemini free tier: `gemini-2.5-flash` is 5 RPM, `gemini-2.5-flash-lite`
  is 10-20 RPM (per-min metric + per-day metric). Goldfive's
  drift→refine recovers from 429s but burns retry budget. Throttle
  if doing repeated runs.

## Env setup recipe (new machine)

```bash
# Clone
git clone https://github.com/manavgarg/goldfive.git
cd goldfive
git fetch origin personal/handoffs:personal/handoffs   # this file
git fetch origin feat/claude-agent-sdk-callable:feat/claude-agent-sdk-callable

# Install uv if missing
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"

# Install Python deps including ADK extras
uv sync --extra adk
uv pip install claude-agent-sdk openai   # both needed by the example

# Verify claude CLI is logged into Max
claude --version
# If not logged in: `claude /login`

# (Optional) Harmonograf stack for live observability
cd ..
git clone https://github.com/pedapudi/harmonograf.git
cd harmonograf
git submodule update --init --recursive
git clone --depth 1 https://github.com/google/adk-python.git third_party/adk-python
npm install -g pnpm@9   # frontend needs pnpm
# Need Node 22+; on macOS: brew install node@22; PATH=/opt/homebrew/opt/node@22/bin:$PATH
cd frontend && pnpm install && cd ..   # pnpm install (not frozen-lockfile, optional darwin-arm64 binding)
make install
make demo &   # server :7531 + UI :5173 + own adk web :8080
# or: cd server && uv run python -m harmonograf_server --store sqlite --data-dir ../data --port 7531

# Install harmonograf_client into goldfive's venv (so HarmonografSink imports)
cd ../goldfive
uv pip install -e ../harmonograf/client
uv pip install -e .   # re-pin goldfive editable from working copy (otherwise uv swaps to harmonograf's submodule)
```

## How to resume the PR work

```bash
cd ~/path/to/goldfive
git checkout feat/claude-agent-sdk-callable
# The committed PR is what's on this branch. To get back the
# Phase B/C BaseLlm work-in-progress, you'd need to re-create it from
# the prior session OR retrieve from the original Mac's /tmp/
# backups before they expire.
```

If `/tmp/` is gone, the diff between the PR's `claude_sdk.py` (~80
lines) and the full Phase B/C version (~330 lines) was:

- Added `_extract_system_instruction` (ADK config → string)
- Added `_flatten_contents_to_prompt` (ADK contents → role-tagged
  prompt, text parts only)
- Added `_PY_TYPE_TO_JSON` + `_build_input_schema_from_signature`
  (Python sig → JSON schema for SDK tool decl)
- Added `_adk_tool_to_sdk_tool` (ADK `BaseTool` → claude-agent-sdk
  `SdkMcpTool`, handler invokes `tool.func(**args)` or
  `run_async(args=, tool_context=None)`)
- Added `make_claude_agent_sdk_llm_class` factory returning
  `ClaudeAgentSDKLlm(BaseLlm)` with `generate_content_async` that
  builds a per-call MCP server from `LlmRequest.tools_dict`, sets
  `tools=[mcp__adk__<name>, ...]`, `setting_sources=[]`, calls
  `query()`, accumulates text from `AssistantMessage` content blocks
- Added lazy `__getattr__` so `from goldfive.integrations.claude_sdk
  import ClaudeAgentSDKLlm` works without forcing ADK import on
  module load

The agent.py `use_claude_sdk` branch also added a `claude-*` check
that wrapped the model with `ClaudeAgentSDKLlm` instead of passing the
string through.

## Bookkeeping note for future handoffs

Append new handoffs to `notes/handoffs/` as
`YYYY-MM-DD-<slug>.md`. Keep this branch (`personal/handoffs`)
single-purpose — never merge to `main` or any PR branch. Push to fork
only:

```
git push origin personal/handoffs
```

To skim recent handoffs across machines:

```
git log --oneline personal/handoffs -- notes/handoffs/
```
