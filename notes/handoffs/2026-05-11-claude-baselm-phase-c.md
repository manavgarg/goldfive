# Handoff — claude-agent-sdk BaseLlm Phase C (2026-05-11)

Sequel to the 2026-05-11 handoff for PR #378. This entry covers the
Phase C BaseLlm work — the `ClaudeAgentSDKLlm` adapter that lets
goldfive subagents run on Claude via Max while preserving full
goldfive observability.

## Where we are

- **PR open**: <https://github.com/pedapudi/goldfive/pull/383> against
  `pedapudi/goldfive` `main`. Branch
  `feat/claude-agent-sdk-baselm` on the `manavgarg/goldfive` fork,
  stacked on `feat/claude-agent-sdk-callable` (PR #378).
- **End-to-end validated**: `GOLDFIVE_USE_CLAUDE_SDK=1
  GOLDFIVE_EXAMPLE_MODEL=claude-haiku-4-5` against the
  `presentation_agent` example produced `success=True 3/3` with all
  three LLM call-sites on Claude Haiku via Max. Real
  `judge_goal_drift` verdicts. Real `delegation_observed` events.

## Architecture: text-encoded replay + PreToolUse defer

The adapter implements ADK's `BaseLlm` contract. Per
`generate_content_async` call:

1. Render the full ADK conversation (incl. prior `function_call` /
   `function_response` pairs) as a descriptive text transcript
   (`_render_contents_as_transcript`).
2. Spawn a fresh `ClaudeSDKClient` with ADK tool schemas registered as
   MCP tools (`mcp__adk__<name>`). Schemas extracted via
   `_extract_input_schema_from_adk_tool` which prefers
   `tool._get_declaration()` → works for `FunctionTool` and
   `AgentTool` alike.
3. `PreToolUse` hook returns `permissionDecision: "defer"` for ADK
   tools, `"deny"` for everything else.
4. Iterate the message stream, capture first deferred tool call, stop
   at `stop_reason=tool_deferred`.
5. Return `LlmResponse` with `function_call` Part (or text-only Part
   if no tool call). ADK runs the tool through its normal pipeline →
   goldfive plugin observes → next ADK invocation re-enters this
   method with the `function_response` appended to `contents`.

## Why text-encoded replay, not stateful client reuse

Original design intent was a **stateful `ClaudeSDKClient` per ADK
conversation** kept alive across `generate_content_async` calls.
Three SDK constraints made that impractical:

1. **No transcript-replay API**: both `query()` and `ClaudeSDKClient`
   only accept user-message streams. There's no way to inject prior
   assistant `tool_use` blocks for replay.
2. **Same-client resume after defer doesn't continue cleanly**: tested
   in probe 2/6 — after `permissionDecision: "defer"` lands a
   `ResultMessage` with `stop_reason="tool_deferred"`, sending the
   `tool_result` via the same client's `query()` returns *"It looks
   like your message came through empty"* — Claude's conversation
   state has moved past the deferred call.
3. **`resume=<session_id>` spawns a fresh subprocess** anyway. The
   "stateful win" of carrying KV cache across turns evaporates.

So text-replay accepts quadratic token cost (resending the full
transcript each turn) in exchange for predictable behaviour and a
simpler implementation. About 250 lines vs an estimated 200-300 for
stateful + lifecycle management.

## Coercion knobs — same as the planner-callable from PR #378

| Knob | Effect |
|---|---|
| `setting_sources=[]` | SDK isolation mode (`types.py:1793-1803`). No filesystem settings loaded, no CLAUDE.md auto-detection, no project-init paths. |
| `tools=[mcp__adk__<name>, …]` | **Visibility allowlist** — only ADK tools surfaced to Claude. Built-ins (`TodoWrite`, `Task`, `Read`, `Bash`, …) excluded → internal agent loop stays dormant. |
| `allowed_tools=[mcp__adk__<name>, …]` | Auto-approve our tools (no permission prompts). NOT the same as `tools` — `allowed_tools=[]` only controls permission prompts, doesn't restrict visibility. Verified empirically — getting these two confused costs hours. |
| `hooks={PreToolUse: [defer]}` | The capture mechanism. Defers ADK tools, denies everything else. |
| `max_turns=5` | Bounded budget for the SDK's internal turn count. Tool-use loops + Claude's pre-defer apology text can consume a few turns. |

## Issues found and fixed during implementation

### 1. AgentTool schema extraction returned empty schema

Symptom: `Tool error: 'request'` (`KeyError`) when coordinator tried
to delegate to research_agent via `AgentTool(research_agent)`.

Root cause: prior helper `_build_input_schema_from_signature(tool.func)`
worked for `FunctionTool` (which has `tool.func`) but returned an
empty schema for `AgentTool` (no `.func` attribute). Claude saw a
tool with no parameters, called it with `{}`, and ADK's
`AgentTool.run_async` raised `KeyError('request')`.

Fix: `_extract_input_schema_from_adk_tool` prefers
`tool._get_declaration()`. Works uniformly for `FunctionTool`,
`AgentTool`, and any subclass conforming to `BaseTool`. Converts the
returned `FunctionDeclaration.parameters` (a `google.genai.types.Schema`)
to a JSON-Schema dict via `_genai_schema_to_json_schema`.

Verification: `_extract_input_schema_from_adk_tool(AgentTool(...))`
now returns `{"type": "object", "properties": {"request": {"type":
"string"}}, "required": ["request"]}` correctly.

### 2. User-account MCP servers leak through `setting_sources=[]`

Symptom: Claude tried to call `mcp__claude_ai_Google_Drive__create_file`
inside research_agent's invocation. ADK doesn't have that tool, threw
"Tool not found."

Root cause: Claude Code's bundled CLI surfaces **cloud-account-bound
MCP servers** (Gmail, Drive, Calendar) to the model. These are loaded
from the user's Claude.ai profile, NOT from local `~/.claude/settings.json`,
so `setting_sources=[]` doesn't disable them. The `tools=[...]` allowlist
controls execution but not visibility — the model still sees them in
its context.

Fix: `PreToolUse` hook denies any tool whose name isn't in our
`mcp__adk__*` allowlist with a clear message. Claude sees the deny,
retries with a real ADK tool within the same Claude run.

## Code map

`goldfive/integrations/claude_sdk.py` additions:

- `_extract_system_instruction(config)` — ADK `GenerateContentConfig.system_instruction` → string
- `_render_contents_as_transcript(contents)` — ADK contents (incl. function_call/response) → role-tagged text transcript
- `_genai_schema_to_json_schema(schema)` — `google.genai.types.Schema` → JSON Schema dict
- `_build_input_schema_from_signature(func)` — fallback path from Python sig
- `_extract_input_schema_from_adk_tool(adk_tool)` — declaration-preferring schema extractor (fixes AgentTool bug)
- `_adk_tool_to_sdk_tool_schema(adk_name, adk_tool)` — wrap ADK tool as SDK MCP tool with stub handler (defer prevents execution)
- `make_claude_agent_sdk_llm_class()` — factory returning `ClaudeAgentSDKLlm(BaseLlm)`
- `__getattr__` for lazy `ClaudeAgentSDKLlm` import

## Limits, known issues, future work

### Agent semantic quality on Haiku

In the validation run, the coordinator misdelegated `draft_presentation`
to `research_agent` instead of `web_developer_agent`, so no slide files
got written. The adapter plumbing is correct (`success=True 3/3`) but
the example's coordinator prompt assumes stronger delegation reasoning
than Haiku reliably provides. Either:
- Bump `GOLDFIVE_EXAMPLE_MODEL=claude-sonnet-4-6` (subagents go to
  Sonnet, planner/judges stay Haiku since hardcoded)
- Or stronger coordinator prompt with explicit "use web_developer_agent
  for any task with 'draft'/'slide'/'create'/'build' in the title"

Not adapter-related — example-tuning issue.

### Goldfive intermediate steering modes

Conversation explored what's possible between strict force-drive and
pure observe. Today:
- Strict force-drive: `SequentialExecutor(max_task_invocations=N)` in
  CLI — but in our example this still invokes the *coordinator*, not
  the assignee directly. Coordinator's LLM still chooses delegations.
- Observe (adk web): coordinator's LLM drives everything per user
  message. Goldfive observes + steers between turns via prompt
  injection / refine / cooperative cancel.

A truly strict mode ("plan is law, dispatch directly to
assignee_agent_id, bypass coordinator's LLM") would be a small new
executor — not implemented today.

### Stateful client retry

If someone wants to revisit stateful `ClaudeSDKClient` per ADK
conversation (the "ideal" architecture I drew up before learning about
the SDK constraints), the open questions are:

- Find a stable conversation-key in `LlmRequest` (NOT `previous_interaction_id` — that's Gemini-specific). Probably hash of `contents[0]` or look for an ADK context-var with invocation_id.
- Solve the "send tool_result and resume" problem (probes 2 + 6 showed same-client `query()` post-defer doesn't continue cleanly). `resume=<session_id>` MIGHT work but probe 7 was inconclusive.
- Lifecycle: when to close clients? `BasePlugin.after_agent_callback` exists in ADK; goldfive could register a small cleanup plugin.

Worth ~200-300 more lines if the per-turn subprocess cost matters.
Doesn't help today since each fresh subprocess is unavoidable for the
defer-replay pattern.

### `anthropic` SDK direct fallback

The cleanest BaseLlm shape uses `anthropic.messages.create(messages=[...])`
which maps 1:1 to ADK's `generate_content_async`. Maybe 50 lines vs
this PR's 480. Requires `ANTHROPIC_API_KEY` (loses Max billing for
subagent calls; API credits instead). Planner/judges can stay on
claude-agent-sdk for Max via PR #378's `make_call_llm`.

If at some point you want to ship goldfive integration as part of a
commercial product where end users hit it programmatically at scale,
this is probably the right path (Max plan terms are for individual /
interactive use).

## Discovery probe trail

For future-me / next-Claude wanting to verify SDK behaviour:

| Probe | Question | Finding |
|---|---|---|
| 1 (`select:tools` field, `previous_interaction_id` source) | Is `previous_interaction_id` a viable registry key? | No — Gemini-specific |
| 2 (`ClaudeSDKClient` + tool execution) | Does breaking iteration on `ToolUseBlock` prevent handler execution? | No — handler runs in background |
| 4 (`PreToolUse` hook denying) | Does the hook fire reliably + prevent handler? | Yes — when iteration continues past the tool_use; if we break too early the hook doesn't even fire |
| 5 (full deny iteration) | What does the SDK do with a denied tool? | Emits synthetic `ToolResultBlock(is_error=True, content="reason")` and Claude apologizes to user |
| 6 (`defer` + same-client resume) | Can we feed tool_result via same client after defer? | No — Claude responds "your message came through empty" |
| 7 (`defer` + new client with `resume=<session_id>`) | Can we resume via session id in a new client? | Inconclusive — probe went silent, likely viable but adds subprocess cost |

## Files changed in this PR

- `goldfive/integrations/claude_sdk.py` — adds `ClaudeAgentSDKLlm` + helpers
- `examples/presentation_agent/agent.py` — wraps with `ClaudeAgentSDKLlm`
  in the `use_claude_sdk` branch when model name starts with `claude-`

## How to resume this work on another machine

```bash
git clone https://github.com/manavgarg/goldfive.git
cd goldfive
git fetch origin personal/handoffs:personal/handoffs
git fetch origin feat/claude-agent-sdk-baselm:feat/claude-agent-sdk-baselm
git checkout feat/claude-agent-sdk-baselm

# Same env setup as previous handoff:
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
uv sync --extra adk
uv pip install claude-agent-sdk openai
# claude /login if not already authenticated to Max

# Run:
GOLDFIVE_USE_CLAUDE_SDK=1 \
GOLDFIVE_EXAMPLE_MODEL=claude-haiku-4-5 \
HARMONOGRAF_SERVER=127.0.0.1:7531 \
uv run --extra adk python examples/presentation_agent/agent.py --topic octopus --verbose
```

Without harmonograf: drop the `HARMONOGRAF_SERVER` env. Without
verbose: drop `--verbose` (loses the goldfive event-stream
JSON output but the run is much quieter).
