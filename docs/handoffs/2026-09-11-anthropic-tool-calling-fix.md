# Work brief: fix native tool calling for the Anthropic provider

**Repo:** `jarvis-llm-proxy-api` — a **service** repo, separate from this meta-repo. It is not
cloned alongside this one. Clone to `~/src/` per the repo standard, and fork it separately if
you intend to open a PR; the `davidalee/jarvis` fork is the meta-repo and is unrelated to this file.

Read this repo's `CLAUDE.md` and `RULES.md` first, plus the service's own `CLAUDE.md`.
`RULES.md` mandates TDD (RED → GREEN → REFACTOR) and type hints on every parameter and return.
Imports at top of file.

---

## The defect

File: `backends/rest_backend.py` (~791 lines).

`_get_endpoint_for_provider` (L191–203) correctly routes `anthropic` to `/v1/messages`.
`_parse_response_for_provider` (L205–~230) correctly branches per provider — OpenAI and
lmstudio read `choices` (L209–210), Anthropic has its own branch (L215), Ollama at L223.
**So the plain, non-tool chat path works for Anthropic.**

`_chat_completion_with_tools` (L272–~322) does not. It:

- builds an OpenAI-shaped payload inline (L286–300), **never calling
  `_format_messages_for_provider`**
- resolves the endpoint at L303, so an OpenAI body is POSTed to `/v1/messages`
- calls `raise_for_status()` at L305
- reads `data.get("choices")`, `choices[0]["message"]`, `choices[0]["finish_reason"]` at L309–314

Its own docstring (L283–284) admits the assumption:

> Assumes an OpenAI-style ``/v1/chat/completions`` endpoint (provider=openai/lmstudio/generic).

The streaming path has the same defect — `urljoin` at ~L526, `chunk.get("choices")` at L576–579.
There is a `_merge_tool_call_deltas` helper at L19 that is OpenAI-delta-shaped.

**Expected failure mode:** Anthropic rejects the OpenAI body with a 400 and `raise_for_status()`
raises. This was established by **reading the source, not by running it.** Reproduce against a
live endpoint before fixing, and correct this description if the real behaviour differs.

---

## What the Anthropic Messages API actually needs

**Request** `/v1/messages`:

- headers `x-api-key` and `anthropic-version: 2023-06-01` — not `Authorization: Bearer`.
  Check whether `_setup_headers` (L143) already handles this.
- `max_tokens` is **required**; currently only sent when `params.max_tokens is not None`
- `system` is a top-level parameter, not a message role
- messages carry only `user` / `assistant` roles — no `system`, no `tool` role
- tools are `[{name, description, input_schema}]`, not OpenAI's
  `{type: "function", function: {name, description, parameters}}`
- `tool_choice` is `{type: "auto"|"any"|"tool", name?}`

**Response:**

- `content` is a **list of blocks**: `{type: "text", text}` and `{type: "tool_use", id, name, input}`
- `stop_reason` is `"end_turn"|"max_tokens"|"stop_sequence"|"tool_use"` — not `finish_reason`
- usage keys are `input_tokens`/`output_tokens`, not `prompt_tokens`/`completion_tokens`
- tool results go back as a user message containing `{type: "tool_result", tool_use_id, content}`

**Streaming** is SSE: `message_start`, `content_block_start`, `content_block_delta` (carrying
`text_delta` or `input_json_delta` for tool arguments), `content_block_stop`, `message_delta`,
`message_stop`.

---

## The constraint that matters most

**Normalize at the backend boundary.** `ChatResult.tool_calls` must keep the OpenAI shape
`[{id, type: "function", function: {name, arguments}}]`, because command-center consumes it that
way — see `app/core/tool_execution_engine.py`, where
`use_native_tools = self.prompt_provider.supports_native_tools`.

**Change nothing downstream of this file.** The only bundled prompt provider with native tools
enabled is `ChatGPTOpenAI`
(`app/core/prompt_providers/large/untrained/chatgpt_openai.py`); check whether it needs an
Anthropic sibling or whether it is provider-agnostic once the backend normalizes.

---

## Scope

**In:** `_chat_completion_with_tools`, the streaming tool path, header setup, and tests.

**Out:** the non-tool chat path (already works), anything in command-center, any other provider's
behaviour. **Do not regress** OpenAI / lmstudio / generic — they share this function.

## Deliverable

Tests first, covering both the Anthropic and the OpenAI paths through this function. Then the
implementation. Report which parts were verified against a live API versus reasoned about from
docs.

---

## Why this exists

Found while planning a GPU-less deployment that must use a cloud LLM — see
[`2026-09-11-vps-deployment.md`](./2026-09-11-vps-deployment.md). The practical consequence was
that **OpenAI is currently the only working cloud provider** for that deployment, which is why
the plan specifies it. Fixing this would remove that constraint.
