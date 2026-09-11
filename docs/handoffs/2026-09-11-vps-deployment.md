# Deploying Jarvis on a GPU-less VPS with a cloud LLM backend

**Date:** 2026-09-11
**Status:** Planned. Nothing built, nothing installed.
**Target:** A single always-on x86-64 VPS — 8 vCPU (AMD EPYC Rome / Zen 2, AVX2, no AVX-512), 16 GiB RAM, ~19 GB free disk, **no GPU**, Ubuntu 26.04.

> Host identifiers, addresses and firewall specifics are deliberately **not** in this file.
> They live in private infra notes (`claude-config: skills/personal-infra/CONTEXT.md`).
> This document is safe to read publicly.

---

## Goal

Move Jarvis from a laptop that sleeps and travels onto a VPS that is always on, and drive it
from a phone and a browser over a private WireGuard tunnel. Voice is **phone push-to-talk**,
not Raspberry Pi nodes.

This trades Core Principle 1 (no cloud dependencies) for Core Principle 2 (self-hostable with
optional cloud). That is a deliberate choice, not an accident — inference goes to a cloud API
because the box has no GPU and CPU inference is too slow (see below).

---

## Decision summary

| Decision | Choice | Why |
|---|---|---|
| Inference | **Cloud API, via llm-proxy REST backend** | No GPU. CPU inference measured too slow. |
| Provider | **OpenAI — not Anthropic** | Anthropic's native-tool-calling path is not correctly implemented. See Blockers. |
| `llm.interface` | **`ChatGPTOpenAI`** | Native tool calling; skips `<tool_call>` text parsing. |
| STT | **whisper.cpp `base.en`, 4 threads** | Measured ~1.5 s. `small.en` is 3–6 s, too slow. |
| TTS | **Piper**, built without Kokoro | Piper is CPU-fast and ~15 MB. |
| Pi Zero nodes | **Out of scope** | No nodes in this deployment. |
| Backups | **Hetzner Cloud Backups** | ~20% of instance cost, 7 daily retained. Enable *before* first migration. |

---

## What was verified

### whisper.cpp CPU benchmark (2026-09-11, on the target box)

Built from source, CPU-only. Runtime `system_info` confirmed `AVX = 1 | AVX2 = 1 | F16C = 1 |
FMA = 1 | BMI2 = 1 | OPENMP = 1 | REPACK = 1`, no AVX-512 — correct for Zen 2. These are honest
numbers from an optimised build.

Model stays resident in the service (`jarvis-whisper-api` pre-warms at startup), so
**processing-only** is the figure that matters. Load cost is paid once: `base.en` 130 ms,
`small.en` 272 ms.

| Model | Clip | Threads | Processing-only |
|---|---|--:|--:|
| base.en | 2 s | 8 | 915 ms |
| **base.en** | **4 s** | **8** | **970 ms** |
| base.en | 8 s | 8 | 1119 ms |
| **base.en** | **4 s** | **4** | **1481 ms** |
| small.en | 4 s | 8 | 3228 ms |
| small.en | 4 s | 4 | 5730 ms |

**Key structural finding — cost is nearly flat across utterance length** (915 / 970 / 1119 ms
for 2 s / 4 s / 8 s). whisper pads every clip to a fixed 30-second mel window, so the encoder
pass dominates and RTF is a misleading metric here. The floor is ~0.95 s per utterance
regardless of how short the command is.

**Greedy decoding does not help.** Tested `-bs 1` against the default 5-beam search:
1086 → 1048 ms at 8 threads, 1594 → 1546 ms at 4 threads. Both within noise, for the same
reason — a 4-second command has few tokens to decode, so the decoder is not the bill.
Do not spend time on this lever. If latency must come down later, the real options are a
quantized model (`base.en-q5_1`) or a streaming architecture, not decoder tuning.

### End-to-end budget

| Stage | Time |
|---|--:|
| Upload, phone → VPS over the tunnel | ~0.2–0.4 s |
| whisper `base.en`, 4 threads | **1.48 s** (measured) |
| OpenAI call, native tool calling, streaming | ~0.5–1.5 s (estimate) |
| Piper first chunk | ~0.1–0.3 s (estimate) |
| Audio return | ~0.2 s |
| **Total** | **~2.5–3.9 s** |

Inside `RULES.md`'s `<5 s` end-to-end target. Slower than the README's ~2.4 s fully-local
figure, which assumes a GPU and LAN-local nodes.

### Thread count is an operational decision, not a performance one

8 threads is 45% faster (970 ms vs 1481 ms) but saturates **all 8 vCPUs**. This box also runs
other long-lived workloads. **Use 4 threads.** It costs ~0.5 s, still clears the budget, and
leaves half the machine. Revisit only if turns feel slow in practice.

---

## Target architecture

```
phone ──┐
        ├── private tunnel ──▶ VPS
browser ┘                      ├─ jarvis-web        /chat /inbox /devices /settings
                               ├─ jarvis-command-center ──▶ llm-proxy ──▶ OpenAI
                               ├─ jarvis-whisper-api  (CPU, base.en, 4 threads)
                               └─ jarvis-tts          (CPU, Piper)
```

Clients reach the box over the existing tunnel. **No firewall changes are required** and no
ports are exposed publicly — this preserves the existing tunnel-only posture.

### Containers

**Deploy:** `jarvis-config-service`, `jarvis-auth`, `jarvis-logs`, `jarvis-command-center`,
`jarvis-llm-proxy-api` (**both** the `llm-proxy-api` and `llm-proxy-model` containers),
`jarvis-whisper-api`, `jarvis-tts`, plus PostgreSQL. `jarvis-admin` optional but useful on a
headless box. `jarvis-web` if the browser client is wanted.

**Omit:** `jarvis-ocr-service` (no OCR need), `jarvis-node-setup` (no nodes),
`jarvis-settings-server` (deprecation candidate), `jarvis-mcp` (potentially deprecated),
`jarvis-recipes-server` (not in scope), `jarvis-notifications` (optional — CC degrades
gracefully without it), `llm-proxy-worker` (async queue; memory-extraction and deep-research
default to off). `jarvis-pantry` is not in the CLI's `SERVICES` array at all.

**Infra:** PostgreSQL required. Redis only needed with `llm-proxy-worker`. MinIO/S3 only stores
LoRA adapter blobs, and adapters are dormant — droppable. Mosquitto exists for server→node
push; with no nodes it is very likely droppable, but CC's startup was **not** confirmed to
tolerate its absence. Verify before removing.

---

## Configuration

### llm-proxy → OpenAI

Settings keys (DB-backed, with env fallbacks):

```
model.live.backend          = REST
model.live.rest_url         = https://api.openai.com
model.live.rest_model_name  = <model id>
model.background.backend    = REST          # background follows live
rest.provider               = openai
rest.request_format         = openai
rest.auth_type / rest.auth_token / rest.auth_header_name   # API key goes here
rest.timeout_seconds
```

**Do not** follow the `JARVIS_LIGHTWEIGHT_MODEL_BACKEND` / `model.main.*` examples in
`README.md` and `docs/REST_BACKEND_CONFIG.md`. The service's own `CLAUDE.md` invariant #5 says
the `full` / `lightweight` / `cloud` / `vision` aliases were removed and only `live` and
`background` exist. Those docs are stale.

### command-center

```
llm.interface = ChatGPTOpenAI
```

Default is `Qwen25MediumUntrained` and **must** be changed explicitly.
`ChatGPTOpenAI` sets `supports_native_tools = True` and `use_tool_classifier = False`, so tools
ride the native API `tools` parameter instead of being embedded in the system prompt and parsed
back out of `<tool_call>` tags. It is the only bundled provider with native tools enabled.

Worth doing after cutover: run `install-e2e/gpu/bench_corpus.cc.yaml` against this path to see
whether native tool calling beats the local models' measured 97–100% routing accuracy.

---

## Prerequisites, in order

1. **Enable Hetzner Cloud Backups.** ~20% of instance price, 7 daily retained, one toggle in the
   console. Do this *before* the first migration runs. This deployment introduces voice profiles,
   speaker embeddings, memories and transcripts — none of which regenerate. The box has no
   backup tooling, a single ext4 root, no LVM snapshots, and no provider snapshots today.
   Caveats: whole-server image, not file-level; and it lives at the same provider.
2. **Install Docker.** Genuinely absent on the target, not merely unpermitted.
3. **Reclaim disk** if needed. ~19 GB free against a ~5–7 GB footprint is comfortable, but the
   disk cannot grow without an irreversible one-way rescale.
4. **Clone the service repos.** `scripts/clone-repos.sh` lists 22 repos but **omits
   `jarvis-web` and `jarvis-notifications`**, which the `jarvis` CLI's `SERVICES` array expects.
   Fix the script first.
5. **Write a CPU compose file.** See Blockers — no usable one ships.
6. **Build images**, set configuration, run migrations, smoke-test the text path before audio.

---

## Blockers and gotchas

1. **Anthropic + native tool calling is broken.** In `backends/rest_backend.py`,
   `_parse_response_for_provider` (line 205) correctly branches per provider — OpenAI reads
   `choices`, Anthropic has its own branch at line 215. But `_chat_completion_with_tools`
   (line 272) does not. It resolves the right endpoint at line 303, then unconditionally reads
   `data.get("choices")`, `choices[0]["message"]`, `choices[0]["finish_reason"]` at lines
   309–314. Its own docstring (283–284) says it *"Assumes an OpenAI-style
   `/v1/chat/completions` endpoint."* The streaming path has the same shape (line 576).

   It also builds the payload inline and **never calls `_format_messages_for_provider`**, so an
   OpenAI-shaped body is POSTed to Anthropic's `/v1/messages`. Anthropic rejects it — `max_tokens`
   is required there but only sent conditionally; the `tools` schema is
   `{name, description, input_schema}` rather than OpenAI's nested `function` object; and `system`
   must be a top-level parameter, not a message role. `raise_for_status()` (line 305) turns that
   into an `HTTPStatusError`, so the **failure is loud, not silent** — the `choices` parse would
   only yield empty content if a 200 ever came back, which it will not against a real Anthropic
   endpoint.

   **Result: `rest.provider=anthropic` with native tools does not work.** Use OpenAI, or an
   OpenAI-wire-format proxy. Fixing this upstream is a worthwhile contribution.

2. **No usable compose file for a GPU-less host.** `deploy/llm-proxy/docker-compose.yaml` and
   llm-proxy's own dev/prod compose files all hardcode
   `deploy.resources.reservations.devices: [nvidia]` through a shared YAML anchor and point at
   the CUDA Dockerfile. A `Dockerfile.cpu` exists; **no CPU compose file does.** One must be
   written.

3. **`llm-proxy-model` cannot be omitted on a REST backend.** It is the process making the
   outbound call. Without `MODEL_SERVICE_URL` the API returns 500 "passthrough-only".

4. **Image bloat: `torch` pulls CUDA wheels by default.** `requirements-base.txt` installs
   `torch>=2.13.0` on every platform. From default PyPI on Linux this drags in the NVIDIA CUDA
   wheels — several GB — on a box with no GPU. Install from
   `--index-url https://download.pytorch.org/whl/cpu` instead (~200 MB, same functionality).
   **This is the single biggest disk lever.** With it, total footprint is ~5–7 GB; without it,
   comfortably double.

5. **The adapter-training stack is dead weight here.** `requirements-base.txt` also carries
   `peft`, `datasets`, `trl`, `accelerate`, `bitsandbytes` for adapter training — and
   `jarvis-command-center/CLAUDE.md` invariant #2 says adapters are *"Dormant… Treat as
   code-frozen."* Note `sentence-transformers` (and therefore `torch`) **is** genuinely needed
   for CC memory recall's vector search. See Open decisions.

6. **`/health` on llm-proxy assumes a model-loading lifecycle** regardless of backend — 200
   `initializing` for up to 15 minutes, 503 on failure. A bad API key or wrong provider
   manifests as a stuck container, not an auth error. Check `docker compose logs
   llm-proxy-model` first.

7. **whisper-api fails fatally if the model path is wrong** — by design, fail-loud. Get the
   model download right before wiring health checks around it.

8. **`jarvis-tts` bakes Kokoro into the image by default** even though Piper is the active
   provider. Build with `--build-arg INSTALL_EXTRAS=""`.

9. **`web_search.enabled` defaults to false and is fail-closed.** A cloud model is exactly the
   kind you'd want doing tool-based web search — enable it deliberately per household.

10. **`JARVIS_ADAPTER_CALLBACK_TOKEN` is fail-closed.** Irrelevant while async jobs are off, but
    callbacks silently 503 if it is unset and they are later enabled.

---

## Open decisions

1. **Strip `requirements-base.txt`?** Removing the dormant training stack means carrying a fork
   of a file upstream will keep changing. Keeping it costs ~3 GB. Both fit in ~19 GB.
   *Undecided.*
2. **Upstream the Anthropic tool-calling fix?** It is a real bug in the parent project, not a
   local workaround. *Undecided.*

---

## Not done, not verified

- Nothing has been installed, built, or configured. Docker is still absent.
- LLM and TTS timings in the budget table are **estimates**. Only whisper was measured.
- Whether CC starts cleanly without Mosquitto is **not confirmed**.
- Whether the `jarvis` CLI's `doctor` / `init` flow assumes a local GGUF model is
  **not confirmed** — verify interactively rather than assuming.
- `base.en` is the smallest English model; household proper nouns will suffer. Not a regression
  — `jarvis-whisper-api`'s Dockerfile already defaults to `base.en`.
- A benchmark scratch directory (~888 MB) was left on the target at `~/whisper-bench/`. It is
  throwaway; the deployment builds its own whisper in Docker.
