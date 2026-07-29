# Agent Failure Recovery — Analysis of `arepa-ai/agents-service`

**Date:** 2026-07-29 · **Scope:** deep-agent loop, tool-calling failure semantics, recovery subsystems, user-facing failure surface.

> **Validation + fix status (2026-07-29):** every P0 finding was re-validated against the code; fixes shipped in [agents-service#457](https://github.com/arepa-ai/agents-service/pull/457) (provider-error classification, loud model-call-limit exhaustion, `kind="error"` stream event, reconcile HTTP routes, `RecoverySkipped` classification parity, docs corrections) — all LLM-cost-neutral.
>
> **One correction to this report:** finding #1 in §2 (hub `recursion_limit` unset → `GraphRecursionError` at ~12 model calls) is **wrong for platform-served runs** — langgraph-api's `DEFAULT_RECURSION_LIMIT` is **10011**, not LangGraph's OSS default of 25, so the hub is effectively unbounded by recursion. The real hub bound is `ModelCallLimitMiddleware(run_limit=60)`, whose *silent* `exit_behavior="end"` exhaustion (finding #3) was the actual gap — now fixed. Direct invocations outside the platform (scripts, tests) do get the OSS default 25 and must set their own.

## TL;DR

The agent loop (deepagents 0.6.x on LangGraph 1.2, Postgres checkpointing) **recovers well from the common in-loop failures** — malformed tool args, empty completions, and a subagent finishing without emitting its artifact all have live recovery machinery. But **several whole-run failure classes bypass every one of those layers and leave the user hanging with no message, no `failed` status, and a progress UI frozen at "generating"** — most notably an unset hub recursion limit, uncaught provider errors on the hub's own primary model, a silent model-call-limit exit, no `error` event kind in the streaming contract, and a stale-run reaper that exists but has no caller.

---

## 1. What happens when the LLM can't call a tool

Three recovery layers exist inside a subagent run, in order:

### Layer 1 — malformed tool arguments (in-loop self-correction)
No `handle_tool_errors` is configured anywhere; the code relies on LangGraph `ToolNode`'s default: pydantic arg-validation failures (`ToolInvocationError`) become `ToolMessage(status="error")` fed back to the model, which self-corrects. Tools like `emit_action_phases` additionally self-validate and return corrective ToolMessages instead of raising (`action/tools/emit_action_phases.py:101-139`), and `TypedPhaseSchemaMiddleware` (`orchestration/middleware/typed_phase_schema.py:194`) rewrites the schema per-run so the model can't hallucinate phase keys.

**Gap:** correction rounds are unbounded — `_count_emit_correction_rounds` (`action/middleware/core.py:125`) is telemetry only, never enforced. A model stuck in a validation loop burns the whole model-call budget (12 for action).

**Gap:** the default handler converts **only** `ToolInvocationError` to a ToolMessage — any other exception raised *inside* a tool body kills the graph turn. The only catch-all boundary is around subagent dispatch (`orchestrator/subagents/registry.py:576`); ordinary hub tools (`classify_intent`, `refine`, `finalize_workflow`, `stage_gate`, …) have none.

### Layer 2 — empty completion / no tool call (model-call guard)
`orchestration/middleware/model_call_guard.py:219` (`guard_model_call`), mounted on the hub and every leaf: if a turn has no content and no tool call → append a reframe nudge and retry the same model → override to Claude Haiku → **if still empty, return the empty response anyway** (`:265-269`). A plain-text no-tool-call turn at the hub is treated as a legitimate end-of-turn, by design (`orchestrator/prompts/rules.md:32`).

**Gap:** a refusal ("I can't help with that") is non-empty content, so the guard treats it as success. There is zero refusal/content-filter detection anywhere; the transcript-replay recovery then re-sends the same refusing transcript.

### Layer 3 — run ended without emitting its terminal artifact (post-hoc synthesis)
Every leaf's `aafter_agent` checks for its contract artifact (`/action_phases.json`, `/strategy.json`, `/website_blueprint.json`, plan sections) and, if missing, re-synthesizes it from the stripped transcript via a structured-output escalation ladder (`orchestration/llm_recovery.py:146` → `core/llm.py:658`). On exhaustion → `SubagentDispatchError(kind="empty_result")` → error ToolMessage back to the hub → hub may re-dispatch (attempt counter persisted in `/_dispatch_attempts.json`, thread ID re-salted per attempt so retries never resume a poisoned checkpoint) → **dispatch circuit breaker at 3 consecutive failures** (`subagents/retry.py:48`) → `_terminal_failure_response` writes `projects.status='failed'` and a founder-visible message.

All four recovery modules are live and wired. Planning additionally has per-section wave recovery (ADR-0130) with graceful degradation to placeholder sections.

**So: inside the loop, yes, it mostly recovers.** The problem is everything *around* the loop.

---

## 2. Where the user is left hanging (no message, no status, stuck UI)

| # | Path | Evidence | User experience |
|---|------|----------|-----------------|
| 1 | **Hub `recursion_limit` unset** — LangGraph default 25 super-steps vs a 60-model-call budget; `GraphRecursionError` fires ~12 calls in and is caught nowhere at the hub | `subagent_cfg.py:73` gives subagents 200; nothing sets it for the orchestrator run | Run aborts; no `failed` status; no message; **the founder's own turn is erased from `chat_history`** (`shared/memory/nodes.py:253-256` — flush only exists for `GraphInterrupt`) |
| 2 | **Uncaught provider errors on the hub's primary model** (Anthropic 429/529/timeout on `claude-haiku-4-5`; OpenAI RateLimitError) | `model_call_guard.py:176` catches only two Google classes; `orchestrator/middleware/core.py:1538` adds only `OpenAIAuthenticationError` — whose own comment documents this exact "aborts before `aafter_agent`, no terminal status" incident mechanism | Same silent death as #1 |
| 3 | **`ModelCallLimitMiddleware(run_limit=60, exit_behavior="end")` at the hub** | `orchestrator/agent.py:242` | Run "succeeds" silently having produced nothing |
| 4 | **No `error`/`failed` `EventKind`** — 12 event kinds, all success/progress; dispatch failure emits neither `progress` nor `narrate` | `events/envelope.py:26-40`; `registry.py:541-591` | Stage pill stuck at "generating" forever on the typed channel; error visibility depends on the FE reading the raw SDK `tools` channel |
| 5 | **Stale-run reaper is dead code** — `reconcile_stale_projects` flips stale `in_progress` → `stalled` + founder message, but has no in-repo caller (no cron, no scheduler, nothing reads `STALE_RUN_THRESHOLD_MINUTES`) | `shared/services/staleness_reconciler.py:65`; `core/config.py:240` | Worker death / OOM / crash ⇒ project stuck `in_progress` **forever** |
| 6 | **Website promotion reconciler also uncalled** — a stuck promotion row holds the UNIQUE partial index and blocks all future promotions for that project | `website/promotion/reconciler.py:10-12` | Permanent wedge, no signal |
| 7 | **No wall-clock run timeout** — `docs/architecture.md:97` claims `execution_timeout_seconds=900`; no such key exists in `langgraph.json` | verified by grep | A wedged run has no bound except per-subagent timeouts |
| 8 | **Partial-output surfacing is LLM-optional** — `request_choice(stage="pipeline_partial_output")` depends on the model choosing to call it | `orchestrator/tools.py:142`; `prompts/rules.md:92` | A crash bypasses it; completed artifacts sit unassembled |

Secondary defects: `RecoverySkipped` uncaught in marketing/website `aafter_agent` (misclassified `kind="tool"` instead of `empty_result`); marketing's `coerce_strategy` silently rejects dict payloads; marketing has no semantic-emptiness gate (a schema-valid strategy with zero channels ships); the documented Flash→Pro→GPT-4o recovery ladder collapses to ~1.5 rungs when `GLM52_SPIKE_ENABLED` (`core/llm.py:723`); `asyncio.gather` without `return_exceptions` in `planning/compose.py:279`; `strip_noisy_messages` over-strips any message containing "limit"+"exceeded"; `SKILL.md` documents unknown-subagent behavior the code doesn't implement.

---

## 3. Recommendations (prioritized)

### P0 — stop silent deaths (small diffs, big UX)
1. **Set the hub's `recursion_limit`** (e.g. 200, matching subagents) via run config or `langgraph.json`. One line; today's ceiling of ~12 model calls contradicts the 60-call budget.
2. **Terminal-failure safety net at the hub boundary**: extend `awrap_model_call`'s classification beyond `OpenAIAuthenticationError` to all provider transient/fatal classes (anthropic/openai rate-limit, timeout, 5xx) → bounded backoff/fallback, then `_terminal_failure_response`. Add the missing chat-history flush on the exception path (mirror the `GraphInterrupt` flush at `core.py:1561`).
3. **Add `error` (and `stalled`) to `EventKind`** and emit it from the `except` blocks in `registry.py` and from `_terminal_failure_response`, with a `retryable: bool` hint so the FE can render a Retry button instead of a frozen pill.
4. **Wire the staleness reconciler** — it's written and tested; it needs only a scheduler (LangGraph cron / Cloud Scheduler hitting a thin invoker). Same for the promotion reconciler.
5. **Make hub model-call-limit exhaustion loud**: on the synthetic limit message, run the same terminal-failure path (founder message + status write) instead of `exit_behavior="end"` silence.

### P1 — harden the loop
6. Error boundary for hub-level tools (wrapper converting exceptions to `ToolMessage(status="error")`, as dispatch already does).
7. Catch `RecoverySkipped` in marketing/website; accept dicts in `coerce_strategy`; add a semantic-emptiness gate to marketing.
8. Refusal detection in the model-call guard (treat refusal-shaped, tool-less terminal turns as recoverable → reframe/fallback rather than success).
9. Enforce the emit-correction-round cap (it's already counted); `return_exceptions=True` in planning's gathers.
10. Configure a per-run wall-clock timeout, and fix `docs/architecture.md` to match reality.

### P2 — make failure a first-class UX
11. **Deterministic partial-result delivery**: on terminal failure, assemble whatever artifacts completed (the warm-start bridge in `orchestrator/backend.py` already persists them) and tell the founder what finished, what didn't, and offer resume — don't leave it to the LLM to volunteer `pipeline_partial_output`.
12. **Teach the concierge about `failed`/`stalled`** so "what happened to my site?" gets a real answer and a retry affordance.
13. Recovery-rate telemetry per failure class (the `telemetry_key`s exist) to watch these regressions.

---

## Answer to the core question

> *"If the LLM is unable to call a tool, does the loop recover or do we leave the user hanging?"*

**Both, depending on where it fails.** Malformed calls, empty turns, and missing artifacts recover through three live layers, ending in a circuit breaker that marks the project failed with a founder message. But a `GraphRecursionError` at the hub, a rate-limit on the hub's own model, a silent budget exhaustion, a worker crash, or any exception from a non-dispatch tool skips every layer: no status write, no event, no message — and in the crash case the founder's own message disappears from history. The single highest-leverage fix is a hub-level terminal-failure boundary (items 1–3 above) plus actually scheduling the reconciler that was built for exactly this (item 4).
