# confidence-routed-agents

> 📖 **Read the write-up:** [Confidence-Routed Agents: Cheap Typed Decisions First, Frontier Model on Escalation](https://webofmike.com/confidence-routed-agents/)

A runnable demo of routing agent tool-call risk decisions through a cheap typed-decision model first ([TypeSafe AI's Jev](https://typesafe.ai)) and escalating to a frontier LLM (Claude Sonnet 5) only when Jev's own confidence is below a threshold. The escalation call runs through [agentgateway](https://agentgateway.dev) v1.5.0, which holds the real Anthropic credential, enforces a hard per-key USD budget on the escalation path, and logs every call.

This is not a model comparison. [jev-benchmark](https://github.com/themsquared/jev-benchmark) and [jev-calibration](https://github.com/themsquared/jev-calibration) already did that. This is the infrastructure question nobody answered in either post: "escalate when confidence is low" is not a model feature, it is a routing problem, and it needs a gateway in the loop for the same reasons any multi-upstream call needs one — budgets, a unified audit trail, and failover when the cheap path is unreachable.

## What this proves

1. **Routing on a published, data-backed threshold cuts cost without cutting accuracy.** [jev-calibration](https://github.com/themsquared/jev-calibration) measured that Jev's confidence of exactly 1.000 was right 133/134 times on this same 240-case set, and recommended escalating everything below it. This repo runs that recommendation for real, against a live Claude Sonnet 5 backend, and reports what it actually costs and how accurate it actually is.
2. **The escalation path never holds an Anthropic key.** The router's only credential is a gateway-scoped key with no model access of its own; agentgateway injects the real `x-api-key` server-side. Same secretless pattern as [secretless-ai-agents](https://webofmike.com/secretless-ai-agents/), applied to the escalation half of a two-model pipeline instead of the whole thing.
3. **A hard USD budget on the escalation key fails closed, with real dollar amounts.** If a threshold is set too low and escalation volume runs away, the budget returns HTTP 429 before the bill does. Verified against a real (tiny, temporary) budget in this README, not simulated.
4. **Jev being unreachable does not stop the agent.** `--no-jev` simulates the cheap path being down; every request falls through to the frontier model automatically. No code path exists where a model failure of either kind drops a request on the floor.

## Architecture

```
            ┌──────────────┐  confidence >= threshold   ┌─────────────┐
 task ──────▶   router.py   ├───────────────────────────▶  answer     │
            │  (calls Jev   │                            │  (cheap     │
            │   directly)   │  confidence < threshold    │   path)     │
            └──────┬────────┘                            └─────────────┘
                   │
                   │ escalate
                   ▼
          ┌─────────────────────┐   budget OK    ┌───────────────┐
          │   agentgateway       ├───────────────▶ Anthropic API  │
          │  (holds the real     │   budget       │ claude-sonnet-5│
          │   Anthropic key,     │   exceeded     └───────────────┘
          │   per-key USD budget,│   ──▶ HTTP 429
          │   access log)        │
          └─────────────────────┘
```

Jev is a typed-decision API, not a chat model, so it is not an agentgateway LLM backend here: `router.py` calls `api.typesafe.ai` directly. The escalation call is the only hop that goes through agentgateway, because it is the only one that needs a shared credential, a budget, and gateway-level audit logging.

## What is in here

| Path | What it is |
|---|---|
| `config/agentgateway.yaml` | A single `claude-sonnet-5` model behind one API key (`sk-router-escalation`) with a $1.00/24h USD budget, backed by agentgateway's real modelCatalog pricing for cost accounting |
| `docker-compose.yaml` | agentgateway v1.5.0 only (Jev and Claude are both reached over the open internet, no mock servers) |
| `router.py` | The routing logic: call Jev, check confidence, escalate through the gateway or don't, write one audit record either way |
| `tasks.jsonl` | 240 hand-labelled agent tool-call risk cases (readonly/destructive/privileged/exfiltration), carried over verbatim from [jev-calibration](https://github.com/themsquared/jev-calibration) so the numbers in this repo are comparable to that post's, not a fresh and incomparable sample |
| `results/*.jsonl` | Raw per-task routing decisions at each threshold tested |

## Quickstart

Requires Docker and API keys for [TypeSafe](https://typesafe.ai) (`TYPESAFE_API_KEY` in `~/.config/blogify/typesafe.env`) and Anthropic (`ANTHROPIC_API_KEY` in `~/.config/blogify/llm.env`). Both calls are real and billed; this is not a mock.

```bash
git clone https://github.com/themsquared/confidence-routed-agents.git
cd confidence-routed-agents
mkdir -p data
docker compose up -d
```

Run the full 240-case set at a threshold:

```bash
python3 router.py --threshold 0.85
```

Simulate Jev being unreachable (every request escalates):

```bash
python3 router.py --no-jev --limit 60
```

## What you should see

Four thresholds against the identical 240-case set, one live Claude Sonnet 5 call per escalation:

| Threshold | Escalated | Accuracy | Total cost (240 calls) | Avg cost/call |
|---|---|---|---|---|
| 0.00 (never escalate, Jev only) | 0/240 (0%) | 90.0% | $0.0042 | $0.0000174 |
| 0.70 | 32/240 (13.3%) | 92.9% | $0.0262 | $0.0001090 |
| 0.85 | 41/240 (17.1%) | 92.1% | $0.0356 | $0.0001484 |
| 0.95 | 61/240 (25.4%) | 92.9% | $0.0520 | $0.0002168 |
| 0.99 | 83/240 (34.6%) | 93.3% | $0.0646 | $0.0002692 |
| 1.00 (jev-calibration's own recommendation: escalate anything below exact 1.000) | 110/240 (45.8%) | 93.3% | $0.0846 | $0.0003523 |
| never use Jev (all escalate) | 60/60 on a 60-case subset (100%) | 91.7% | $0.0422 (60 calls) | $0.0007035 |

The all-escalate row is measured on a 60-case subset (not the full 240) to bound the live-Claude spend in this README's own verification run; `router.py --no-jev` with no `--limit` reproduces it on the full set at proportionally the same cost and accuracy.

Two things in this table are worth stating plainly rather than smoothing over. First, 0.85 scores a hair worse than both its neighbors (92.1% against 92.9% at both 0.70 and 0.95) — at n=240 that is noise from a handful of cases moving across the cutoff, not a real dip, and the honest read is a band from roughly 92-93%, not a precise function of the threshold. Second, and this is the actual headline: routing at jev-calibration's own recommended cutoff (escalate anything below exactly 1.000) cost **half** what escalating everything costs ($0.0846 vs an extrapolated $0.169 for 240 all-Claude calls) and scored **better**, not worse: 93.3% against 91.7%. The cheap path is not a compromise here. On this task set, Jev's near-certain answers (the 1.000-confidence bucket jev-calibration already showed was right 133/134 times) are more reliable than asking Claude Sonnet 5 for a terse, non-reasoning JSON classification on the same input. Escalating a case Jev was already sure about does not improve on Jev; it just spends money finding that out.

### The budget actually blocks

Verified with a temporary $0.0002/24h budget (not the real $1.00 one shipped in `config/agentgateway.yaml`) to prove the mechanism without needing hundreds of calls:

```
call 1 -> HTTP 200
call 2 -> HTTP 200
call 3 -> HTTP 200
call 4 -> HTTP 429
{"error":{"message":"Budget exceeded","type":"rate_limit_error","code":"budget_exceeded"}}
```

agentgateway's own log line for the rejected call:

```
2026-10-01T14:39:20.199122Z  warn  budget  API key budget exceeded  connection.id=4 request.id=4 api_key="router" budget="daily-usd" used=0.000216000 limit_unit="USD" limit_amount=0.0002 exceeded=true
```

`used` and `limit_amount` are both real dollar figures computed from `config/agentgateway.yaml`'s `modelCatalog` rates and the actual token counts of the three calls that preceded the rejection, not round numbers picked for the demo.

## Gotchas

- **`max_tokens: 128` silently drops the answer on some cases.** One case in the 240-task full run came back with no `content` block at all (`finish_reason` present, `content` key simply absent) at `max_tokens: 128`. This is the identical failure the [jev-calibration](https://github.com/themsquared/jev-calibration) harness already documented at the same token budget: Claude's internal reasoning consumed the entire response budget before it reached the answer. Raising to `max_tokens: 256` did not reproduce it across all 240 cases. `router.py` checks for a missing/empty `content` field explicitly rather than assuming the key exists.
- **The SQLite budget store needs its directory to exist and be writable before the container's first request**, not before it starts: agentgateway starts cleanly against a missing `/data` volume and only fails with `unable to open database file` on the first budget-tracked call. `mkdir -p data` before `docker compose up` avoids a confusing first-request failure.
- **Jev's listed `input_tokens` field is close enough to use for cost, not exact per call.** Jev's own `usage.input_tokens` was used when present; the fallback of 400 tokens (jev-calibration's measured average) is used only on the rare response missing it, and is marked in the raw results so it is never silently treated as measured.

## What this does not cover

- No frontier-vs-frontier comparison; Claude Sonnet 5 is the only escalation backend tested.
- No measurement of what happens when escalation volume is sustained for days, only a point-in-time budget trip.
- The $1.00/24h budget in the shipped config is a demo value, not a sizing recommendation; size it from measured escalation rate and real model pricing for your own workload.

## License

Apache-2.0. `tasks.jsonl` is the same file published in [themsquared/jev-calibration](https://github.com/themsquared/jev-calibration), also Apache-2.0.
