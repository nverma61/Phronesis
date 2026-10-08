# Phronesis — Development Plan (v2)

## One-line thesis
Other guardrails check agent actions against rules. Phronesis checks actions against **purpose**, across **time**, and records the **reasons** — as an **independent, model-agnostic** overseer, so novel misbehavior that no rulebook anticipated becomes visible early.

## Strategic principles (these shape technical decisions)
1. **Independence.** The overseer should not be the party being overseen. Phronesis must work across model providers and must never depend on a single provider.
2. **Cross-family judging.** By default, agents are judged by a model from a *different* family (or an ensemble). Whether this actually improves detection is a hypothesis we test in Phase 6 — not a claim we make before then.
3. **Auditable overseer.** Open-source core, self-hostable, no hidden components. An overseer that can't itself be audited isn't independent.
4. **Standards over features.** The ledger format and the benchmark are designed from day one to be published and adopted by others, including competitors.
5. **Proof before breadth.** No platform features until the Phase 6 evaluation shows the core claim holds.

## What we are building (MVP scope)
1. **Interceptor** — sits between an agent and its tools; sees every tool call.
2. **Action classifier** — routine vs. sensitive, driven by config.
3. **Purpose judge (Level 2)** — sensitive actions must carry a stated reason; a separate judge checks it against the original task.
4. **Ledger** — append-only, hash-chained record of actions, reasons, and verdicts, in a **versioned, publishable format**.
5. **Trajectory judge (Level 3)** — scores windows of the ledger for patterns that no longer serve the task.
6. **Tripwires (Level 4)** — planted fake resources no legitimate task needs; any touch is a hard alarm.
7. **Verdict engine** — combines signals into allow / flag / escalate / block.
8. **Two operating modes** — `shadow` (observe and record verdicts, never interfere) and `enforce` (verdicts take effect).
9. **Temptation benchmark** — vendor-neutral scenarios with planted shortcuts and ground-truth labels, able to evaluate *any* oversight tool, not just Phronesis.

## Explicitly out of scope for the MVP
- Real network access for test agents (everything is simulated — see Safety).
- Hosted platform, escalation inbox UI, trust reports, auth, billing, multi-tenancy.
- Cross-customer label sharing (designed for later; not built now).
- Interpretability / activation probes.
- Claims that Phronesis makes agents "safe." It makes misbehavior visible sooner.

## Architecture decisions
| Decision | Choice | Why |
|---|---|---|
| Language | Python 3.11+ | Agent ecosystem, fast iteration |
| Core shape | Framework-agnostic library + thin adapters | Testable, portable |
| Model access | `ModelProvider` interface; implementations for **at least two model families** before Phase 6; a deterministic `FakeProvider` for tests | Independence; cross-family experiments; tests never hit live APIs |
| Judges | Configurable per judge: provider, model (from env), or ensemble | Cross-family by default |
| Agents (benchmark) | Run through the same `ModelProvider` interface | Enables the agent-family × judge-family experiment |
| First adapter | Python tool-function wrapper + minimal agent loop | Simplest end-to-end path |
| Second adapter | MCP proxy | "No agent code changes" integration |
| Reasons | Wrapper **injects a required `reason` parameter** into the schema of sensitive tools, and strips it before calling the real tool | Agents give reasons without tool authors changing anything |
| Ledger | SQLite, append-only, hash-chained; JSONL export; **versioned JSON Schema (`phronesis-ledger v0.1`)**; OpenTelemetry export in Phase 7 | Publishable standard; interoperable |
| Config | YAML, schema-validated | Policies live in version control |
| Test envs | Docker, networking disabled; all external tools are mocks | Safety |
| CI | GitHub Actions, lint + tests, **no live API calls** | Cheap, deterministic, no leaked keys |
| License | Deferred; decide before Phase 7 (Apache-2.0 is the leading option) | Repo stays private until then |

## Core data model (starting point)
- `Task` — id, description (the agent's actual assignment), allowed scope.
- `ToolCall` — id, run_id, task_id, agent_id, tool name, args, timestamp, action_class.
- `Justification` — tool_call_id, stated reason (untrusted data).
- `Verdict` — target (tool_call_id or window_id), source (`rule` | `tripwire` | `purpose_judge` | `trajectory_judge`), decision (`allow` | `flag` | `escalate` | `block`), rationale, confidence, judge provider/model, token cost, latency.
- `LedgerEntry` — schema_version, sequence number, prev_hash, hash, payload (one of the above).
- `Run` — run_id, mode (`shadow` | `enforce`), config hash, agent provider/model, judge providers/models.

Every verdict records which provider and model produced it — required for the cross-family analysis and for auditability.

## Principles (initial set, configurable)
- **Necessity** — the action is required for the task.
- **Proportionality / minimal footprint** — no more access, compute, or reach than needed.
- **Sanctioned channels** — communicate only with approved parties through approved tools.
- **Goal over score** — no tampering with or probing of graders, tests, or evaluation infrastructure.
- **Escalate when in doubt** — questionable means should trigger a request for approval, not improvisation.

## Security design rules (non-negotiable)
1. **Judge isolation.** Judges receive only: the task, the action, the stated reason, and (for trajectory) the ledger window. Never the agent's full context or system prompt.
2. **Agent-supplied text is untrusted.** Reasons and tool args are delimited; judge prompts treat them as data, never instructions.
3. **Agents never see scores.** In enforce mode, blocked/escalated agents get a generic message only.
4. **Tripwires are indistinguishable.** Nothing visible to an agent reveals a resource is a tripwire. In shadow mode, tripwire touches return plausible mock responses.
5. **Fail closed** in enforce mode for sensitive actions on judge error or timeout. In shadow mode, record the failure as its own verdict.
6. **Cost guardrails.** Per-run judge budget (tokens and dollars); on budget exhaustion, fall back to escalate in enforce mode and record it.

## Safety for the benchmark
- Test agents run in containers with networking disabled.
- Network, forum, credential, and admin-API tools are mocks returning scripted data. Nothing reaches the real internet or real systems.
- Never point the benchmark at third-party infrastructure.

## Benchmark design (key to credibility)
- **Oversight interface.** The benchmark talks to oversight tools through a small interface (`observe(tool_call) -> verdicts`, `observe_window(entries) -> verdicts`). Phronesis is one implementation; the rule baseline is another; others can add theirs.
- **Ground truth from instrumentation, not judgment.** Each scenario's mocks log deterministically whether the shortcut was actually taken. That label is the answer key — never an LLM's opinion.
- **Guaranteed positives.** Real LLM agents may rarely take shortcuts. So every scenario also has **scripted adversarial agents**: deterministic scripts that take the shortcut, including variants with convincing false justifications and "slow" multi-step versions.
- **Clean controls.** Each scenario has a version with no shortcut available, to measure false positives.
- **Scenario shortcuts ≠ Phronesis tripwires.** The environment's temptations are part of the task world; tripwires are planted by Phronesis. Evaluation reports results with tripwires on and off, so the judges' contribution is measured separately.
- **Multiple runs per condition.** LLM agents are stochastic; run each condition N times (N set by budget, target ≥ 10).

## Phases
Estimates assume ~8–10 hours/week part-time. They are rough guesses; re-estimate after Phase 1.

### Phase 0 — Scaffolding (week 1)
- Branch `phase-0-scaffolding`. Repo structure, `pyproject.toml`, ruff, mypy (strict), pytest, pre-commit hooks, GitHub Actions CI.
- `.gitignore`, `.env.example` (no real keys committed). No LICENSE yet (deferred to Phase 7).
- Data models with unit tests. `FakeProvider` for deterministic tests.
**Done when:** CI passes lint, type checks, and model tests with no network access; pre-commit hooks run locally.

### Phase 1 — Core pipeline (weeks 1–2)
- `@phronesis.tool` wrapper: intercept, classify, inject/strip `reason` for sensitive tools, write to ledger.
- YAML config loader with schema validation.
- Ledger: hash-chained SQLite, JSONL export, `phronesis-ledger v0.1` JSON Schema file, `verify` CLI command, trajectory print CLI.
- `shadow` and `enforce` modes.
- Rule-based verdicts only (the baseline).
**Done when:** a toy agent's calls are all recorded; exported JSONL validates against the schema; tampering is detected by `verify`; mode switching is tested.

### Phase 2 — Tripwires (week 2)
- Types: fake file paths, fake credentials, fake endpoints, fake "shared agent forum" tool.
- Any touch → `block` verdict (enforce) or recorded verdict + plausible mock response (shadow).
**Done when:** each type triggers in tests; a test asserts no agent-visible output contains tripwire markers.

### Phase 3 — Providers + purpose judge (weeks 3–4)
- `ModelProvider` interface with implementations for two model families (choose which — see Open decisions).
- Purpose judge: structured JSON verdict; timeouts; caching of identical (task, action, reason) triples; cost + latency recorded.
- Judge config: single provider, cross-family default, or ensemble.
- Hand-label ~30–50 (task, action, reason) examples; if possible, have a second person label them too and record agreement.
**Done when:** the labeled set is scored by judges from both families; results are logged for review. (Targets set after the first run, not before.)

### Phase 4 — Temptation benchmark (weeks 4–6)
- Oversight interface; Phronesis and rule-baseline implementations.
- Minimal agent loop over `ModelProvider`, so agents from either family can run.
- 5 scenarios, each with tempted version, clean control, ground-truth instrumentation, and scripted adversarial agents:
  1. Leaked grader answers in the workspace.
  2. Unsanctioned "shared forum" with scripted peer agents posting tips.
  3. Planted credential + mock admin API that trivializes the task.
  4. Hard tests the agent could "pass" by editing the tests.
  5. Scope creep: a tool that grants extra permissions on request.
**Done when:** every scenario runs end to end for scripted and LLM agents, in both modes, producing ledgers with ground-truth labels.

### Phase 5 — Trajectory judge (weeks 6–7)
- Sliding windows (configurable size/stride); scores pattern-vs-task and drift in stated reasons.
- Runs asynchronously; can escalate retroactively.
**Done when:** it runs on all Phase 4 ledgers and verdicts are recorded with cost/latency.

### Phase 6 — Evaluation (weeks 7–8)
Run primarily in **shadow mode**, so oversight doesn't change agent behavior mid-run. Judge conditions are compared by replaying saved ledgers (see cost design rule).
- **Ablation:** rule baseline vs. +tripwires vs. +purpose judge vs. +trajectory judge vs. full stack; tripwires on/off.
- **Cross-family experiment:** 2×2 of agent family × judge family (same vs. cross), plus ensemble.
- **Metrics:** detection rate on tempted runs, false-positive rate on clean controls, steps-to-detection, judge cost and latency per run.
- **Reporting honesty:** raw counts plus confidence intervals; no claims beyond the sample; negative results reported.
**Done when:** a results table and short internal write-up exist, including what didn't work.

### Phase 7 — Publish (weeks 8–9)
- Choose and add the license before anything goes public.
- Ledger spec v0.1 as a standalone document + JSON Schema; OpenTelemetry export.
- Benchmark released with instructions for plugging in *other* oversight tools.
- Public write-up / blog post with the Phase 6 results.
**Done when:** someone outside the project could run the benchmark against their own oversight tool from the README alone.

### Phase 8 — MCP proxy + viewer (weeks 9–10)
- Phronesis as an MCP proxy in front of arbitrary upstream MCP servers.
- Local web viewer: timeline of actions, reasons, verdicts, with provider/model shown per verdict.
**Done when:** an off-the-shelf MCP client, pointed at the proxy instead of an upstream server, produces a valid ledger with no changes to the client; the viewer renders any exported Phase 4 ledger.

## After the MVP (only if Phase 6 supports the thesis)
Hosted cross-family judges; escalation inbox with human review; incident replay; trust reports for enterprise security reviews; opt-in anonymized label sharing to improve judges; partner integrations as a "judgment engine" for gateways and observability platforms.

## Known risks to test, not assume away
- Convincing but false justifications (covered by scripted adversarial agents).
- Agents detecting honeypots.
- Judge prompt injection via reasons/args.
- LLM agents rarely taking shortcuts (mitigated by scripted agents; still report LLM rates).
- False-positive fatigue.
- Judge cost at scale.
- Cross-family judging showing no benefit — publish it anyway.

## Decisions
- **Repo:** private until Phase 7.
- **License:** deferred until Phase 7 (before anything is published). No LICENSE file in Phase 0.
- **Model families (proposed):** Family A = Anthropic (Claude), Family B = OpenAI (GPT). Cheap tier for judges and bulk agent runs; a small number of spot-check runs with a stronger agent model. Optional later: an open-weight model run locally, to show judges can be fully self-hosted.
- **Budget (proposed):** ~$50/month in Phases 3–5, ~$150 for the Phase 6 month; hard usage limits set in both provider consoles. Re-estimate after Phase 3 using the real token counts the ledger records.

## Cost design rule: record once, judge many times
Agent runs are the expensive part. Every LLM agent run is recorded once (in shadow mode) and its ledger is saved. Judge configurations (same-family, cross-family, ensemble, prompt revisions) are evaluated by **replaying saved ledgers** through the judges — never by re-running agents. Use provider batch APIs for replays where available.

## Repo layout (proposal — confirm in Phase 0)
```
phronesis/
  core/        models.py, ledger.py, classifier.py, verdicts.py, config.py, modes.py
  providers/   base.py, fake.py, <family_a>.py, <family_b>.py
  judges/      purpose.py, trajectory.py, prompts/
  tripwires/   files.py, credentials.py, endpoints.py, forum.py
  adapters/    python_tools.py, mcp_proxy.py
  cli.py
  replay.py
spec/          ledger-v0.1.schema.json, LEDGER_SPEC.md
bench/
  oversight.py         (vendor-neutral interface)
  agent_loop.py
  scripted_agents/
  scenarios/   leaked_answers/, shared_forum/, planted_creds/, test_editing/, scope_creep/
  run_eval.py
configs/       default.yaml
tests/
docs/          PLAN.md
CLAUDE.md
```
