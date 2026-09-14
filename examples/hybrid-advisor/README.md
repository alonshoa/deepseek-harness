# Hybrid Advisor MVP

This example composes existing DeepSeek Harness subagent primitives into a practical hybrid-model workflow:

- a long-running parent agent is the implementation worker (intended for a local model such as Qwen through Ollama);
- a Codex one-shot subagent is exposed to that worker as the synchronous `ask_advisor` tool;
- both operate from the same parent-session working directory;
- the parent remains the only intended workspace owner and performs final verification.

The important property is synchronization: `ask_advisor` is configured with background execution disabled. A call therefore blocks the parent tool loop until Codex returns its final answer. The answer becomes a normal tool result in the existing parent session, and the local worker continues from the same repository state.

## What this MVP reuses

No custom coordinator or session-transfer layer is required. The composition uses:

1. `@deepseek-ai/dsh-subagent-codex` as the strong-model provider.
2. `@deepseek-ai/dsh-tool-subagent` as the model-facing delegation tool.
3. The parent session and workspace as the continuing worker state.
4. `AGENTS.md` as the escalation and handoff policy.

Install the two optional packages into the Profile if they are not already present, apply the `cordis.patch.yml` rows, and configure the parent model/provider normally.

## Expected end-to-end flow

1. The local worker receives the user's coding task.
2. It inspects and edits the repository using its normal tools.
3. It runs tests/build/render/other verification.
4. When the escalation policy in `AGENTS.md` is met, it calls `ask_advisor` with a standalone problem packet.
5. Codex starts a fresh ephemeral one-shot run in the same cwd.
6. The worker waits for the final Codex response.
7. The response is returned as a tool result in the worker's existing session.
8. The local worker evaluates the advice, implements the change, and verifies it.
9. Only the local worker declares the original task complete.

## Current safety limitation

This MVP intentionally does not claim hard read-only enforcement for the Codex advisor. The current `subagent-codex` provider's `permissionMode: never` maps to `approvalPolicy: never` while leaving the native sandbox field unspecified. The worker policy instructs Codex not to modify files, but that instruction is not an authority boundary.

Before treating the advisor as safely read-only for arbitrary repositories, add and test an explicit Codex provider mode that maps to the native `sandbox: read-only` thread setting (assuming the pinned Codex app-server version supports that value). Until then, use this MVP for controlled development/testing workspaces and inspect the working tree around advisor calls when side effects matter.

## MVP acceptance test

A useful first test is a small repository with a deliberately non-trivial failing test:

- start the parent with the local worker model;
- ask it to solve the task end-to-end;
- after two failed materially different fixes, confirm it invokes `ask_advisor`;
- confirm the Codex answer returns to the same parent conversation;
- confirm the parent applies or rejects the recommendation itself;
- confirm the parent reruns verification and completes the original task.

The next iteration should add hard read-only Codex sandbox enforcement and automated tests for the advisor composition before adding automatic escalation or multi-writer takeover.
