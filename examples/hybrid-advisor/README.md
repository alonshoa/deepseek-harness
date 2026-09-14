# Hybrid Advisor MVP

This example composes existing DeepSeek Harness subagent primitives into a practical hybrid-model workflow:

- a long-running parent agent is the implementation worker (intended for a local model such as Qwen through Ollama);
- a Codex one-shot subagent is exposed to that worker as the synchronous `ask_advisor` tool;
- external-model use is explicitly gated by the user before every advisor call;
- both operate from the same parent-session working directory;
- the parent remains the only intended workspace owner and performs final verification.

The important property is synchronization: `ask_advisor` is configured with background execution disabled. A call therefore blocks the parent tool loop until Codex returns its final answer. The answer becomes a normal tool result in the existing parent session, and the local worker continues from the same repository state.

## What this MVP reuses

No custom coordinator or session-transfer layer is required. The composition uses:

1. `@deepseek-ai/dsh-subagent-codex` as the strong-model provider.
2. `@deepseek-ai/dsh-tool-subagent` as the model-facing delegation tool.
3. The parent session and workspace as the continuing worker state.
4. `AGENTS.md` as the escalation, approval, and handoff policy.

Install the two optional packages into the Profile if they are not already present, apply the `cordis.patch.yml` rows, and configure the parent model/provider normally.

## Expected end-to-end flow

1. The local worker receives the user's coding task.
2. It inspects and edits the repository using its normal tools.
3. It runs tests/build/render/other verification.
4. When the escalation policy in `AGENTS.md` is met, it stops and proposes external review instead of calling Codex immediately.
5. The user approves or declines the proposed external review.
6. If declined, the worker continues locally. If approved, the worker calls `ask_advisor` with a standalone problem packet.
7. Codex starts a fresh ephemeral one-shot run in the same cwd.
8. The worker waits for the final Codex response.
9. The response is returned as a tool result in the worker's existing session.
10. The local worker evaluates the advice, implements the change, and verifies it.
11. Only the local worker declares the original task complete.

Approval is intentionally one-shot: one explicit approval authorizes one advisor call. A second escalation requires another approval unless the user explicitly grants broader permission.

## Escalation decision policy

The worker should continue locally by default and propose external review only when its expected value is high. Initial triggers are deliberately simple and inspectable:

- two materially different fixes for the same blocker failed;
- the same verification failure repeats and the next attempt would mostly be guessing;
- reasonable investigation did not produce a plausible root cause;
- a consequential architectural decision needs stronger review;
- an unexpected regression cannot be explained from the current diff;
- the user explicitly requests strong-model review.

This first version uses model-visible policy rather than a new orchestration service. Later versions can add deterministic counters for repeated failure signatures and failed-fix attempts without changing the approval or advisor interfaces.

## Current safety limitation

This MVP intentionally does not claim hard read-only enforcement for the Codex advisor. The current `subagent-codex` provider's `permissionMode: never` maps to `approvalPolicy: never` while leaving the native sandbox field unspecified. The worker policy instructs Codex not to modify files, but that instruction is not an authority boundary.

Before treating the advisor as safely read-only for arbitrary repositories, add and test an explicit Codex provider mode that maps to the native `sandbox: read-only` thread setting (assuming the pinned Codex app-server version supports that value). Until then, use this MVP for controlled development/testing workspaces and inspect the working tree around advisor calls when side effects matter.

## MVP acceptance test

A useful first test is a small repository with a deliberately non-trivial failing test:

- start the parent with the local worker model;
- ask it to solve the task end-to-end;
- after two failed materially different fixes, confirm it proposes external review but does not invoke Codex yet;
- approve the review in the next user turn;
- confirm it then invokes `ask_advisor` exactly once;
- confirm the Codex answer returns to the same parent conversation;
- confirm the parent applies or rejects the recommendation itself;
- confirm the parent reruns verification and completes the original task.

The next iteration should add hard read-only Codex sandbox enforcement and automated tests for the advisor composition before adding automatic escalation or multi-writer takeover.
