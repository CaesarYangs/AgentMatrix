# Initial local daemon Pi build

> Issue draft: GitHub Issues are currently disabled for this repository. Enable Issues in repository settings before publishing this tracker.

**Proposed issue title:** [feature]: Build local-daemon Pi workflow with persistent Issue conversations

## Summary

Build the first end-to-end AgentMatrix workflow: pair a user-owned computer/server with a local daemon, select its registered workspace and Pi agent from an Issue, execute locally, and persist/visualize the conversation on the platform.

This is the umbrella tracking issue for **M0–M5** in the [dev README](https://github.com/CaesarYangs/AgentMatrix/blob/dev/README.md). Implement through independently testable increments on `dev` or task branches merged into `dev`; validated milestones later merge into `main`.

## Why

Users should use their own installed agents, repositories, tools, and provider credentials. AgentMatrix should focus on task organization, durable dispatch, conversation history, visibility, and human acceptance. The platform should not host agent processes or require model API keys for this workflow.

Local execution means that agents and tools run on the connected machine; a local agent can still call a cloud model.

## Architecture and boundaries

- **Platform:** Django API and PostgreSQL own machine registration, authorization, durable commands, Issue/Session/Run records, normalized events, and selected artifacts. Existing Issue, permissions, and UI foundations are reused.
- **Daemon:** independently installable Node.js/TypeScript application owns machine identity, local workspace registration, process supervision, native agent sessions, durable command receipts, and an event upload journal.
- **Adapter:** start with a pinned, tested Pi subprocess using its structured JSONL RPC interface. Other agent adapters are future work.
- **Transport:** daemon-initiated HTTPS polling plus batched event uploads; browser incremental reads. No inbound daemon port, SSH requirement, or direct machine access to the platform database.
- **Persistence:** native agent context stays local; normalized conversations persist centrally. A displayed transcript is not sufficient to reconstruct native context.
- **Dispatch:** durable PostgreSQL commands are sufficient for the new path; Celery does not host or invoke agents.

## Initial scope

One machine owner, one selected machine/workspace per Session, one Pi adapter, one primary Session per Issue, and one active Run per Session. Work starts explicitly and human acceptance completes the Issue.

The machine owner must register local workspaces and agent executables. Requests reference registered IDs rather than arbitrary executables or paths. Workspace selection is not a sandbox: Pi runs with its process permissions, and isolation must be enforced locally.

Machine credentials are scoped and revocable, separate from user API tokens. Provider credentials remain local and are never baked into images or committed. Synchronized prompts/tool output may contain project content; only selected artifacts are uploaded.

## M0 — Protocol and two-container harness

- [ ] Define a versioned wire contract for pairing, machine heartbeat/capabilities, registered workspaces, command delivery/receipts, event batches/acknowledgements, reconciliation, and cancellation.
- [ ] Define stable command/event IDs, sequence and ownership rules, payload validation, and compatibility behavior for Python and TypeScript.
- [ ] Define a fixed fixture task that changes a Git repository and produces a reviewable result.
- [ ] Build exactly two runtime environment containers: `platform` and `local-machine`.
- [ ] Build `platform` from current source, bundling required database/cache/queue/storage services under a supervisor. The existing AIO image still needs external services and cannot be used unchanged.
- [ ] Put daemon, pinned Pi, local tools, and fixture repository in `local-machine`; use independent persistent volumes and publish only the platform browser port.
- [ ] Keep repositories and native agent state out of platform mounts. Run tests from the host or existing containers without a third runtime service.
- [ ] Add health checks and reproducible boot/shutdown instructions.

**Acceptance:** both source-built environments boot; the machine reaches the platform through its outbound API path; platform and machine have independent storage.

## M1 — Machine pairing and presence

- [ ] Implement short-lived, single-use pairing with explicit browser confirmation of machine and owning workspace.
- [ ] Issue/store a revocable machine-scoped credential; implement separate machine authentication and authorization.
- [ ] Report heartbeat, installed/supported agent versions, capabilities, and locally registered workspace IDs.
- [ ] Add machine listing/detail and online/unreachable/revoked UI states.
- [ ] Reject expired/reused pairing, cross-owner/workspace access, unregistered workspace IDs, and commands/events not assigned to the authenticated machine.

**Acceptance:** an owner pairs and sees an eligible machine/workspace; unauthorized callers cannot dispatch or ingest events, and revoked credentials cannot access the API.

## M2 — Durable commands and local Pi execution

- [ ] Add Machine, MachineWorkspace, AgentSession, Run, MachineCommand, AgentEvent, and Artifact records with migrations and API registration.
- [ ] Add daemon command polling, durable receipts, session/run ownership, and single-active-Run enforcement.
- [ ] Implement the Pi adapter: start/open persistent native sessions, submit prompts, read structured events, query state, and cancel.
- [ ] Persist the native session mapping and local event journal before relying on platform connectivity.
- [ ] Distinguish prompt acceptance from final execution outcome; validate Pi completion/error/abort semantics against the pinned version.
- [ ] Account for queued follow-up work during cancellation. Do not implement stop as an unconditional process kill.
- [ ] Keep diagnostics separate from protocol output; handle process startup errors, unexpected exits, and bounded output buffering.

**Acceptance:** an Issue starts real Pi inside the simulated machine, changes the fixture repository, and returns a persistent conversation and reviewable result. Duplicate delivery does not create a second agent execution.

## M3 — Issue conversation and controls

- [ ] Select a machine, registered workspace, and eligible agent from an Issue.
- [ ] Add an Issue conversation panel and Kanban execution summary using existing UI conventions.
- [ ] Display persistent user/assistant messages, incremental output, collapsible tool calls/results, errors, and artifact links.
- [ ] Reconcile streamed deltas with authoritative completed messages without duplicate text.
- [ ] Add follow-up execution in the same native session, cancellation requested/confirmed states, and explicit human acceptance.
- [ ] Expose only supported adapter controls; keep steering/interactive extensions out of the initial UI unless explicitly supported and tested.
- [ ] Restore history and current status after browser refresh.

**Acceptance:** an owner runs a task, inspects tool activity/result, sends a second instruction in the same native session, and accepts the Issue without leaving the platform.

## M4 — Reconnect, restart, and synchronization correctness

- [ ] Persist event batches before upload and acknowledge only committed platform events.
- [ ] Replay unacknowledged data with stable identity/ordering and deduplicate retried uploads.
- [ ] Reconcile local receipts/native state with platform command ownership after network loss or daemon restart.
- [ ] Protect terminal Run states from stale events and resolve cancel/completion races.
- [ ] Show unreachable/uncertain execution explicitly. Never infer failure or redispatch to another machine merely because heartbeat/lease expired.
- [ ] Record ambiguous process/tool outcomes for review rather than silently repeating side effects.
- [ ] Confirm revocation blocks future API access without claiming it immediately stops an offline local process.
- [ ] Exercise independent platform/machine restarts and bounded behavior for large output and disk/network errors.

**Acceptance:** a machine can continue locally during a connection interruption and upload complete history on reconnect; retries do not duplicate work or messages. A crashed process is reconciled or marked uncertain, never silently relaunched.

## M5 — Package and release the initial workflow

- [ ] Package the daemon with configuration, help, graceful shutdown, persistent data locations, and supported-version checks.
- [ ] Document installation, pairing, workspace registration, local provider setup, startup, revocation, recovery, and troubleshooting.
- [ ] Add deterministic protocol/integration tests using a simulated adapter.
- [ ] Add a real-Pi smoke scenario with credentials supplied at runtime; keep its result distinct from simulated test results.
- [ ] Document supported platforms and verified Pi/Node versions; record reproducible commands and validation evidence.
- [ ] Update the README with actual setup commands only once implemented.

**Acceptance:** the complete install → pair → run → follow up → disconnect/reconnect → review flow is repeatable, and both containers preserve their own state across restarts.

## Definition of done

- [ ] Browser-accessible platform and simulated local machine run as exactly two environment containers.
- [ ] Agent execution, repository changes, provider credentials, and native sessions stay on the machine.
- [ ] A real Pi task modifies the fixture repository and produces a result visible from its Issue.
- [ ] A second instruction continues the same native conversation.
- [ ] Messages, tool activity, artifacts, and execution history persist across browser refresh and platform restart.
- [ ] Command redelivery, duplicate/out-of-order events, acknowledgement loss, and reconnection are tested.
- [ ] Cancellation acknowledgement, agent crash, machine revocation, and uncertain execution are tested.
- [ ] Pairing and owner/workspace/assignment boundaries are covered.
- [ ] Checks appropriate to each changed package pass; real-agent smoke evidence and remaining limitations are recorded.

## Proposed code boundaries

- `apps/daemon/`: CLI/service, transport, supervisor, local persistence, and synchronization.
- `packages/agent-protocol/`: versioned schemas and cross-language conformance fixtures.
- `packages/agent-adapters/`: local adapter interface and Pi subprocess implementation.
- `apps/api/plane/db/models/agent.py`, migrations, `plane/agent/`, views/serializers/URLs: platform records and services.
- `packages/types/src/agent/`, `packages/services/src/agent/`: browser contracts/clients.
- `apps/web/store/agent/`, `apps/web/components/agent/`, existing Issue panels/cards: machine and conversation experience.
- `deployments/development/` and a dedicated Compose file: source-built two-container harness and fixtures.

## Outside this initial build

Codex/Claude Code adapters, shared-machine grants, automatic routing, multi-agent orchestration, strategy editors, cross-machine session migration, live attachment to independently running terminal sessions, and automatic execution on card drag.

Pi references: [RPC](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/rpc.md), [sessions](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/sessions.md), and [events](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/json.md).
