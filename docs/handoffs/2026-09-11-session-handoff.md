# Handoff — Jarvis on a GPU-less VPS

**Date:** 2026-09-11 · **Tier:** standalone (no session marker, no `docs/plans/`)

Everything substantive is already committed. This document is the map, the pending actions,
and the traps — not a restatement.

> **No host identifiers here.** This file is committed to a public fork. Addresses, firewall
> detail and host specifics live in the private `claude-config` repo at
> `skills/personal-infra/CONTEXT.md`, reachable via the `personal-infra` skill.

---

## Read these first, in this order

| Document | What it holds |
|---|---|
| `docs/handoffs/2026-09-11-vps-deployment.md` | The plan. Decisions, measured benchmark, container set, exact config keys, 10 blockers, open decisions. |
| `docs/handoffs/2026-09-11-repo-findings.md` | Three defects: whisper-api missing from the macOS Darwin override, a stale `CLAUDE.md` claim, two repos missing from `clone-repos.sh`. |
| `docs/handoffs/2026-09-11-anthropic-tool-calling-fix.md` | Work brief for the fourth defect, in a **different repo** (`jarvis-llm-proxy-api`). |
| `skills/personal-infra/CONTEXT.md` (private repo) | The host. Read via the `personal-infra` skill, not by path. |

Do not re-derive any of the above. It was verified once and written down.

---

## Repo topology — get this right before touching anything

Three separate repos are in play and two share a name:

| | |
|---|---|
| `alexberardi/jarvis` | Upstream. **READ only.** Public. Never push, never PR here. |
| `davidalee/jarvis` | The fork. **ADMIN.** Public — forks of public repos cannot be made private. |
| `davidalee/claude-config` | Personal config. **ADMIN.** **Private.** Home for anything naming a host. |

In the local checkout, `origin` → the fork, `upstream` → `alexberardi`. Issues are **disabled**
on the fork (GitHub default), which is why findings are committed as docs rather than filed.

**Name collision, read twice:** the VPS is *named* `jarvis`, and the project is *called* Jarvis.
They were unrelated until this session. Any sentence about "jarvis" must say which.

---

## Decisions already made — do not reopen

| Decision | Why |
|---|---|
| **Cloud LLM, not local** | No GPU. CPU inference measured too slow. |
| **OpenAI, not Anthropic** | Anthropic's native-tool-calling path is broken — see the fix brief. |
| **Pi Zero nodes are out of scope** | Voice is phone push-to-talk. This is what keeps the deployment tunnel-only with no firewall change. |
| **`base.en` at 4 threads, not 8** | 8 saturates all vCPUs; the box shares CPU with a long-lived agent runtime. Costs ~0.5 s, worth it. |
| **Do not strip `requirements-base.txt`** | Fix `Dockerfile.cpu` line 41 instead. Recorded in the plan's Open Decisions. |

---

## State of play

**Open, both draft, both `MERGEABLE`, no CI configured:**

- `davidalee/jarvis#1` — the deployment plan, 2 commits
- `davidalee/claude-config#7` — restores 3 `Decisions` entries dropped from `main`

**Local only, not yet pushed:** branch `docs/session-findings` @ `ad4f796` in
`~/src/jarvis/.worktrees/session-findings` — carries the findings and the Anthropic brief.
This box has no backups; pushing is the first thing to do.

**Nothing is installed.** Docker is still absent. No service repo is cloned. No container built.

**Pending, awaiting the user:** push `docs/session-findings`; delete `~/whisper-bench` (888 MB,
results already captured); fast-forward local `main` (30 behind).

---

## Traps this session hit — do not repeat

1. **A stale branch checkout served stale facts.** `~/.claude/skills` symlinks into the
   `claude-config` working tree. The checkout was parked on an old feature branch, so the
   `personal-infra` skill returned superseded content and a disk figure that disagreed with
   direct measurement for an hour. **Check what branch `claude-config` is on before trusting
   the skill.** It is now on `main`.
2. **`gh pr create` defaults to the parent repo on a fork.** Always pass
   `--repo davidalee/jarvis`. Verify with `isCrossRepository: false` afterwards.
3. **`gh pr edit --body` can fail on a deprecated Projects-classic GraphQL query** while
   reporting partial success. Use `gh api -X PATCH repos/.../pulls/N` instead, and verify.
4. **`/tmp` is tmpfs on this box.** Never leave a work product there.
5. **Two things were asserted before being checked, and both were wrong.** The Anthropic failure
   was described as a silent empty response when `raise_for_status()` makes it a loud 400; and a
   disk discrepancy was called unexplained when the explanation was in `main`'s `CONTEXT.md`.
   Both are corrected in the committed docs. **Read the code path to its end before characterising
   a failure mode.**

---

## Next actions, in order

1. Push `docs/session-findings` and open its draft PR.
2. **Enable Hetzner Cloud Backups** before any migration runs. This is the one prerequisite that
   cannot be retrofitted onto data already lost, and it has been an open gap in `CONTEXT.md`
   since 2026-09-08.
3. Install Docker, then write a CPU compose file — **none ships**; every existing one hardcodes
   `nvidia` device reservations via a shared YAML anchor.
4. Follow the plan's Prerequisites section from step 4.

After `claude-config#7` merges, `work/worktree-layout-standard` can be deleted — it has no
upstream and holds nothing else unique.

---

## Suggested skills

Call these with the Skill tool:

- **`personal-infra`** — **before any work touching the VPS.** It is the only documented source
  for host addresses, firewall posture, and the `Decisions` log. Do not read `CONTEXT.md` by path
  and do not ask the user to restate what it contains.
- **`mattpocock-skills:tdd`** — if you take on the Anthropic fix. `RULES.md` mandates
  RED → GREEN → REFACTOR, and the brief specifies tests first covering both the Anthropic and
  OpenAI paths.
- **`mattpocock-skills:diagnosing-bugs`** — to reproduce the Anthropic 400 against a live endpoint
  before changing anything. The defect was established by reading source, never by running it.
- **`run`** — once containers exist, to launch the stack and confirm the text path before audio.

**Do not call `claude-api`.** Its own trigger says to skip when another provider is named, and
this deployment is deliberately OpenAI-bound.

`/loose-ends` already ran; its report is in the conversation, and every finding from it is
committed or listed above. There is no session marker, so `/end-session` does not apply.
