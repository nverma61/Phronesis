# Phronesis — project context for Claude Code

## What this project is
Phronesis is an independent, model-agnostic runtime oversight layer for AI agents. It intercepts agent tool calls, requires stated reasons for sensitive actions, judges those reasons against the agent's actual task (by default using a judge from a different model family than the agent), watches whole trajectories for patterns that stop serving the task, and uses planted tripwires as hard alarms. Full plan: `docs/PLAN.md`. Read it before starting any phase.

## How to work in this repo
- Work **one phase at a time** from `docs/PLAN.md`. Do not start a later phase unless asked.
- Before writing code for a phase, propose the files you will create/change and the key interfaces, then wait for approval.
- Write tests alongside code. A phase is not done until its "Done when" criteria pass.
- Keep commits small and scoped to one milestone; use clear commit messages.
- If a design decision in `docs/PLAN.md` looks wrong, say so and explain why instead of silently deviating.
- Ask before adding a new dependency.

## Tech conventions
- Python 3.11+, type hints everywhere, pydantic (or dataclasses) for models.
- ruff for lint/format, pytest for tests.
- All model access goes through the `ModelProvider` interface. Never call a provider SDK directly from judges, agents, or core code.
- Model names and API keys come from environment variables only; never hardcode or commit them.
- **Tests and CI never call live APIs.** Use `FakeProvider`. Live-API runs are separate scripts, run manually.
- Config in YAML under `configs/`, validated against a schema on load.
- Judges return structured JSON; parse defensively.
- The ledger format is versioned (`phronesis-ledger v0.1`). Any change to ledger fields requires updating the JSON Schema in `spec/` and bumping the version.
- Every verdict records its source, provider, model, token cost, and latency.
- Agent runs are recorded once; judge experiments replay saved ledgers (`phronesis/replay.py`). Never re-run agents just to test a judge change.
- The repo is private and has no license yet; do not add a LICENSE file or license headers.

## Security rules (never violate)
1. Judges never receive the agent's full context or system prompt — only task, action, stated reason, and (for trajectory) the ledger window.
2. Agent-supplied text (reasons, tool args) is untrusted: delimit it; judge prompts treat it as data, not instructions.
3. Agents never see judge scores or rationales — only a generic block/escalate message in enforce mode.
4. Nothing visible to an agent may reveal that a resource is a tripwire.
5. In enforce mode, sensitive actions fail closed on judge error, timeout, or budget exhaustion. In shadow mode, record the failure as a verdict.
6. The ledger is append-only and hash-chained.

## Safety rules for the benchmark (`bench/`)
- Test agents run in containers with networking disabled.
- All network, forum, credential, and admin-API tools are mocks returning scripted data.
- Never write code that sends benchmark traffic to real external systems.
- Ground-truth labels come from deterministic mock instrumentation, never from an LLM.
- The benchmark talks to oversight tools only through `bench/oversight.py`, so it stays vendor-neutral.

## Engineering quality bar
- Prefer the simplest design that meets the phase's "Done when" criteria. No speculative abstractions, plugin systems, or config options the plan doesn't call for.
- No dead code, commented-out code, TODOs without an issue reference, or print debugging left in.
- Comments explain *why*, never restate *what* the code does. Public functions and classes get concise docstrings.
- Type-check with mypy in strict mode for `phronesis/`; no `Any` or `# type: ignore` without a one-line justification.
- Tests assert behavior, not implementation details. Don't write tests that only check a mock was called.
- Use property-based tests (hypothesis) for invariants: ledger hash chaining, append-only behavior, config validation.
- Every security rule above has at least one test that would fail if the rule were broken.
- Errors are explicit: custom exception types in `phronesis/core/errors.py`; never swallow exceptions silently.
- Approved dev dependencies: pytest, pytest-cov, hypothesis, mypy, ruff, pre-commit. Ask before adding anything else.

## Git workflow
- Never commit directly to `main`. One branch per phase: `phase-N-short-name` (e.g. `phase-1-core-pipeline`).
- Commits are small and atomic: one logical change, and the test suite passes at every commit.
- Commit messages follow Conventional Commits: `type(scope): imperative summary` (≤ 72 chars), blank line, then a body explaining *why* and any tradeoffs. Types: feat, fix, test, refactor, docs, chore, ci, perf.
  - Good: `feat(ledger): hash-chain entries to make tampering detectable`
  - Bad: `update files`, `fix stuff`, `WIP`
- Tests ship in the same commit as the code they test (or the commit immediately before, for test-first work).
- Stage files explicitly; never `git add -A` or `git add .`. Run `git diff --staged` before committing.
- Do not commit until the user has reviewed the diff and approved. Propose the commit message; the user confirms.
- Never rewrite history on `main`, force-push, or amend commits that have been pushed.
- At the end of a phase, open a PR from the phase branch with: summary, design decisions and alternatives considered, how it was tested, and known limitations.
- Significant design decisions get a short ADR in `docs/adr/NNNN-title.md` (context, decision, consequences).

## Useful commands (fill in as they are created)
- Tests: `pytest`
- Coverage: `pytest --cov=phronesis --cov-report=term-missing`
- Lint: `ruff check . && ruff format --check .`
- Types: `mypy phronesis`
