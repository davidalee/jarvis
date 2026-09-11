# Repo findings, 2026-09-11

Three defects found while planning a GPU-less deployment. None are deployment-specific —
all three affect the upstream project. Recorded here because issues are disabled on this
fork (GitHub disables them on forks by default), so there is nowhere else to file them.

Each was established by reading the code, **not** by running it. Reproduce before fixing.

---

## 1. `jarvis-whisper-api` gets no Metal on macOS

**Severity: highest of the three.** It silently costs real latency on every macOS dev box.

`jarvis:43-58` declares every service `docker`. The Darwin override block at `jarvis:92-101`
rewrites exactly three to `local`:

```bash
jarvis-llm-proxy-api*) SERVICES[$i]="jarvis-llm-proxy-api|7704|2|local|/health" ;;
jarvis-tts*)           SERVICES[$i]="jarvis-tts|7707|3|local|/health" ;;
jarvis-ocr-service*)   SERVICES[$i]="jarvis-ocr-service|7031|3|local|/health" ;;
```

`jarvis-whisper-api` is **not** among them, so on macOS it runs inside the Docker Linux VM.
The comment three lines above the block states the consequence itself: *"Docker on Mac runs
inside a Linux VM without Metal passthrough."* So STT runs CPU-only on VM-allocated cores.

This matters more than it looks. STT is the one stage that cannot overlap with anything —
`jarvis-whisper-api/CLAUDE.md` states as an explicit non-goal: *"Not a: Streaming STT service.
Single shot WAV in, text out."* The node VADs, waits for end-of-utterance, then uploads. The
transcription is pure serialized dead time, and on macOS it is paying a CPU-only penalty on top.

Speaker recognition is affected too — `recognize_speaker` uses resemblyzer, whose CLAUDE.md
describes a CUDA path (*"~5-7s on CUDA cold start"*).

**Suggested fix:** add a fourth case to the same block. `jarvis-whisper-api`'s `run.sh` must
support `--setup` first, per the comment at `jarvis:90`.

**Not verified:** the actual latency delta on a macOS box. No measurement was taken.

---

## 2. `CLAUDE.md` understates the macOS override

`CLAUDE.md`, "Development model (mixed local/Docker)", says:

> The `jarvis` script overrides `mode=docker` → `mode=local` for `jarvis-llm-proxy-api` and
> `jarvis-ocr-service`.

The script overrides **three** services — it also rewrites `jarvis-tts` (`jarvis:98`). The doc
names two.

Worth correcting alongside finding 1, since anyone reading the doc to answer "which services
get Metal?" gets a wrong answer twice over.

---

## 3. `scripts/clone-repos.sh` omits two services the CLI expects

`scripts/clone-repos.sh` lists 22 repos. The `jarvis` CLI's `SERVICES` array (`jarvis:43-58`)
lists 15 services, two of which the clone script never fetches:

- `jarvis-web` (port 7722)
- `jarvis-notifications` (port 7712)

A fresh checkout following the documented clone step therefore cannot start the full stack.

(`jarvis-pantry` is in the `CLAUDE.md` service table but correctly absent from both the
`SERVICES` array and the clone script — it is the hosted package store, not a local service.
That one is not a bug.)

---

## Context

Found while assessing whether the stack could run on a GPU-less VPS. That assessment lives in
[`2026-09-11-vps-deployment.md`](./2026-09-11-vps-deployment.md), which also documents a fourth
defect specific to the cloud-backend path: native tool calling is not correctly wired for the
Anthropic provider in `jarvis-llm-proxy-api`. A work brief for that one is in
[`2026-09-11-anthropic-tool-calling-fix.md`](./2026-09-11-anthropic-tool-calling-fix.md).
