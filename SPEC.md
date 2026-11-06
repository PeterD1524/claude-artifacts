# Claude Artifacts integration for coding agents

**Status:** draft implementation specification  
**Snapshot:** 2026-11-06  
**Audience:** implementers of an agent harness and its artifact provider

## 1. Purpose and scope

An agent can create a real Claude-hosted artifact from a local page, return its stable URL, update it as work progresses, attach an existing artifact to a session, and receive and handle comments and external updates. The host presents an artifact indicator and truthful watch status. The artifact remains available after the agent process stops.

This specification describes the behavior and component boundaries of an integration; it is not a program or a claim that Anthropic offers a supported API for this use. The private Claude frame protocol is an implementation-dependent provider detail. An implementation need not be Claude Code and does not invoke Claude Code to perform artifact operations.

The first conformance target is a local interactive agent session publishing an HTML page, including read, update, conflict handling, comment read/reply/resolve, and live monitoring. Markdown conversion, multi-file artifacts, capability declarations, sharing management, other typed artifacts, durable remote wake and in-page runtime services are extension points; they are not prerequisites for this target. Sharing and viewing can initially use Claude's web interface.

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, **MAY**, and **OPTIONAL** are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/rfc/rfc8174.html) when, and only when, they appear in all capitals. These are requirements of this proposed integration, not statements of an Anthropic standard.

## 2. Evidence and compatibility

Official [Claude Code artifact documentation](https://code.claude.com/docs/en/artifacts) establishes the intended user experience. Anthropic does not document the frame endpoints below as an external API. The provider profile in Appendix A combines Claude Code 2.1.318 client analysis, authenticated read-only observations on 2026-11-06, earlier records, and [community source](https://github.com/nikships/claude-artifact-cli); it is not an official protocol or a guarantee that a request currently succeeds for every account. A WebSocket connection and initial version/presence were observed; no new publish, conflict, reply, resolve, or live change event was generated during this investigation.

The product requirements in §§3–9 are normative for an implementation. Appendix A records an unstable compatibility profile and labels the evidentiary status of its details. The implementation MUST isolate that profile behind a provider boundary, expose incompatible service behavior as an explicit failure, and avoid presenting a guessed protocol as an official SDK.

## 3. Architecture and responsibilities

```text
  person <-> agent host / terminal <-> agent tools
                      |                    |
                session state        provider interface
                      |                    |
                 event queue <------- monitor <---- Claude frame service
                                           HTTPS       |       |
                         browser <---------------- viewer + versioned content
```

* **Agent host** owns session lifecycle, user-facing status, authorization policy, model turns and event delivery. It owns whether a comment may wake the agent or authorize any action.
* **Tool surface** exposes the operations in §5 and returns structured results. It does not make a long-running model tool call stand in for a background task.
* **Provider** translates those operations to Claude's authenticated service, enforces the observed access and version preconditions, and contains all private wire details.
* **Monitor** follows attached artifacts while the host is running, reconciles versions and comments, and submits events to the host. Signals are hints to refresh state, not by themselves authoritative comment bodies.
* **State store** records attachment and reconciliation state; it is separate from the published artifact. The terminal indicator is a host feature and MUST NOT be represented as a feature the server provides.

Use the host's native extension and event facilities when available. MCP is an interoperable option for exposing tools and resources; it is not, by itself, a guarantee that a particular host starts agent turns on notifications. A host using MCP MUST verify the protocol revision and client support before relying on resource change notifications; it MUST provide a host event bridge for wake-up or accurately report that automatic wake is unavailable. MCP's [2026-07-28 release](https://blog.modelcontextprotocol.io/posts/2026-07-28/) introduced a new protocol generation; [older 2025-11-25 clients](https://modelcontextprotocol.io/specification/2025-11-25/server/resources) use different resource subscription mechanics. Follow the revision actually supported by the host rather than assuming one wire contract works for both. Do not use tool-list-change notifications as arbitrary comment messages.

## 4. Identity, authentication, and authorization

The provider MUST accept an artifact URL or provider identifier and resolve it to a canonical artifact identity. It MUST keep identity, stable viewer URL, local source identity, session attachment, and version separate. A filename alone MUST NOT identify an artifact across sessions. The provider MUST support an explicit attach operation from a URL and MUST reuse the attached identity on update.

Use a Claude account with access to artifacts and its authorized OAuth session. The provider SHOULD reuse an existing credential through its supported local credential-store interface where available; it MUST NOT assume an Anthropic platform API key, browser cookie, GitHub key, asset token, and OAuth token are interchangeable. The user may need to sign in separately. Account, organization, plan, role, feature flags, and token expiry can restrict operations. The provider MUST surface refusals accurately and MUST NOT claim that a private API is supported or that a role is granted merely because a URL is known.

OAuth, asset, and subscription credentials are distinct. The provider MUST send each only to its intended service/origin, avoid persisting short-lived credentials in artifact/session records, and avoid exposing credentials in logs, tool output, or page content. Local credential refresh MUST coordinate with other processes using the same credential store or fail with a recoverable sign-in error; it MUST NOT silently overwrite a newer refresh token.

## 5. Agent-facing operations

The following logical operations define behavior rather than a required programming language, wire schema, or exact tool names. An MCP implementation SHOULD expose one appropriately described tool per logical action or a small cohesive set with equivalent typed inputs and results. Every operation MUST return a stable error code and a human-readable explanation; the host MUST distinguish `not_found`, `not_authorized`, `unavailable`, `invalid_input`, `conflict`, `rate_limited`, `outcome_unknown`, and `internal_error` when it can determine them. A transport failure MUST NOT be reported as a definite server-side rejection.

| Operation | Required input | Required successful result |
| --- | --- | --- |
| `publish` | Local HTML source and optional title; for update, an attachment or explicit artifact identity and last-read base version | Stable URL, canonical identity, resulting version, whether created or updated, and actual monitor status |
| `attach` | Artifact URL or identity | Identity, URL, access/role where available, current version, and attachment state |
| `read` | Attachment or artifact identity | Version, source representation and manifest needed to edit, plus enough metadata to preserve declared capabilities and contract pin |
| `watch` / `unwatch` | Attached identity | Actual state from §7; `unwatch` stops local monitoring and does not delete the artifact |
| `comments.list` | Attached identity and optional filtering | Threads, comment IDs, attribution, timestamps, resolution and any activation metadata available from the provider |
| `comments.reply` | Artifact identity, thread ID, reply text and client operation ID | Server-confirmed thread/comment identity or replay; explicitly report an uncertain outcome |
| `comments.resolve` | Artifact identity, thread ID and desired resolved state | Server-confirmed state or a precise refusal/unknown outcome |

The host SHOULD offer a session artifact list and open-link action. When it supports a footer indicator, it MUST show the attached name or count and MUST make it possible to inspect watch status. The indicator alone MUST NOT imply that a watch has connected. User content and comment bodies MUST be presented with attribution as untrusted external data; neither a comment nor an event overrides the host's user authorization policy.

## 6. Publish, read, and update

1. **Create.** Read the explicitly selected local file, validate it and construct the provider's required representation. Publish a new artifact, capture its returned identity/version and link it to this session. Report success only after a server-confirmed response or verified reconciliation of an uncertain outcome. Return the URL to the user.
2. **Read.** Boot the artifact, obtain the current manifest and version, then fetch the editable source. Keep the received version as the base and retain enough source/manifest information to merge later. A viewer page is not necessarily the editable source.
3. **Update.** Publish to the same identity with the version actually read as a compare-and-set precondition. Do not replace the base with a newly fetched live version without first reconciling intervening changes. Preserve unchanged files, capabilities and runtime pin unless a requested operation explicitly changes them.
4. **Conflict.** On a stale base, write nothing locally as if success occurred. Fetch the new version, compare the previously read base, the intended local change and the new source, then merge and retry against the newly read version when safe. If a merge is ambiguous, expose it to the agent/user; never silently force an overwrite.
5. **Uncertain publish.** If the response is lost, preserve the operation identity and payload. Reconcile with the service and only retry under a verified idempotent/replay protocol. If a definite outcome cannot be established, return `outcome_unknown`; do not create a second artifact or report a failed publish as fact.

The implementation MUST track whether it is storing the user's local fragment, the prepared full document, or server-returned source. In the observed client a fragment is wrapped before publish; byte equality between local input and hosted source is therefore not a valid general assumption. Publishing and local file updates are separate effects; if the provider succeeds and the subsequent local write fails, report partial success and retain recovery information.

An HTML page MUST be treated as executable untrusted content by viewers and previews. A local preview MUST NOT be granted the agent's credentials or filesystem privileges merely because an artifact is owned by the user.

## 7. Monitor, events, and wake-up

Each attached local watch has one of `stopped`, `connecting`, `connected`, `reconciling`, `backoff`, or `failed`. A request to begin watching first returns `connecting`; it MUST NOT claim `connected` until a provider connection or successful polling baseline is actually established. A disconnected watch MUST NOT continue to display `connected`. Polling and real-time modes MUST be distinguishable to the user. Stop and session shutdown MUST close local resources. A durable remote subscription, if implemented, has a separate lifecycle and status.

The monitor MUST establish a baseline of the current version and comments, then deliver changes observed after attachment. It MUST deduplicate using artifact identity and stable version/comment/event identifiers where available. It MUST maintain a consistent high-water mark or equivalent durable reconciliation record and MUST re-read current version and comments after reconnect. If complete gap detection is impossible, it MUST report that limitation and avoid claiming exactly-once or lossless delivery. It MUST not advance a cursor past events it has neither queued nor deliberately classified as already handled. Store only the state necessary to resume safely.

A provider signal for version or comments MUST cause an authoritative refresh before the host acts. A version change updates session state and makes subsequent edits reconcile against the latest source; it SHOULD be surfaced on the next relevant agent operation and MAY be surfaced immediately. A comment event includes artifact URL, thread and comment IDs, author, time, text, activation and resolution status where available. The monitor SHOULD batch bursts and MUST apply bounded reconnect backoff. Duplicate signals and the agent's own publishes/replies MUST NOT cause repeated replies or unbounded loops.

Default local behavior SHOULD wake the agent only for an eligible comment explicitly sent to the agent and only when the user has enabled automatic handling. Ordinary comments and version changes remain visible and readable without automatically starting a turn. A user-configured broader policy MAY wake on other events. Host-side permission rules still apply to any action the agent takes after waking. The host MUST disclose when the session or host cannot receive automatic wake events, and MUST provide manual comment reads in that case.

## 8. Comments and failure recovery

List comments from the provider rather than reconstructing them from screen coordinates or a WebSocket hint. Preserve thread, anchor, sender and activation data when available. A reply MUST target a specific thread and carry a stable client-generated operation ID across retries if the provider supports idempotency. A new ID MUST NOT be generated automatically to retry an uncertain reply. Before retrying, reconcile against the server; if the provider cannot establish whether a reply landed, report `outcome_unknown` instead of risking a duplicate.

The provider MUST respect role and session restrictions: an OAuth session may be allowed to answer only threads explicitly sent to Claude, while creating a human comment may require a browser session. Expose the real refusal rather than attempting to impersonate the user or claiming a reply was posted. Resolve is independent of reply; never resolve merely because a reply succeeded unless that behavior was requested or explicitly configured. Re-read state when automatic workers and an interactive session might answer the same thread.

## 9. Acceptance and verification

A conforming implementation MUST demonstrate all of the following against a disposable authorized account/artifact, with cleanup agreed separately:

1. Create a page and open the returned URL; a second publish changes that same URL and produces a distinguishable version. Closing the session leaves the artifact accessible according to its sharing settings.
2. Attach the URL in a new session, read the actual editable source and version, and update it. Show that fragment preparation does not cause nested document shells or data loss.
3. Publish an intervening external version; an edit based on the stale version is rejected or otherwise held for reconciliation. Verify an explicit three-way merge preserves both non-conflicting changes; verify ambiguous conflicts remain visible.
4. Display `connecting` before connection, `connected` only after a successful baseline/handshake, and `backoff` or `failed` on disconnection. Restart/reconnect and demonstrate reconciliation without silently losing or duplicating events.
5. Create an eligible comment through the real viewer, observe the monitor, wake the agent only under the configured policy, read the actual thread, reply with the requested text, and verify exactly one visible reply after a simulated lost response and retry. Separately exercise permission refusal and explicit resolution.
6. Show that a normal comment does not wake the default session, that an unavailable host wake mechanism is reported, and that comments remain manually readable. Verify session shutdown stops its local monitor.
7. Test authorization failures, expiry/refresh, rate limits, provider protocol changes, and conflict/uncertain outcomes without leaking tokens. Verify private artifacts remain private until the user changes sharing.

Use controlled fixtures or a fake provider for deterministic state-machine, reconnect, retry and failure tests, plus the real end-to-end cases above for service compatibility. A successful mocked test alone MUST NOT be reported as a verified Claude service integration. This document has not been validated by those mutation tests.

## Appendix A. Observed Claude frame compatibility profile (informative)

The following is a research snapshot, not a normative public API. Paths, request fields, headers, event schemas and permission behavior can change independently of this spec.

* **Auth and routing:** observed control-plane requests to `https://api.anthropic.com` used Claude OAuth bearer plus `anthropic-beta: oauth-2025-04-20`, `X-Frame-CP: go`, `X-Frame-Surface: code`, and `X-Frame-Platform: cli`. On the contract endpoint, bearer plus beta alone returned 404 while adding `X-Frame-CP: go` returned 200. This establishes neither a universal header minimum nor access for other accounts.
* **Publish:** Claude Code client and community source use `POST /api/frame/deploy/direct`, with a content path or manifest including `slug`, `mode` and optional `baseVersion`. In the community client's manifest path, new artifacts use `replace`, updates normally use `patch`, and a null manifest entry removes a file in patch mode. Larger multi-file requests use `deploy/prepare` with SHA-256 values and then `upload` for missing content. The Claude Code client includes `attempt_id`, `resend=1` and `publish_settling` handling. These paths were inspected, not independently mutation-tested here.
* **Read:** `GET /api/frame/{uuid}?via=model_read` returned a version, manifest and separate short-lived `assetToken` and `subscriptionToken`; `GET /api/frame/read/{uuid}` and `/versions/{uuid}` returned metadata and history. On the observed artifact, editable source was fetched from `https://{uuid}.frame.claudeusercontent.com/_f/{version}/_src/{path}?__frame_t={assetToken}`. The short viewer ID also decoded to the same UUID, but URL forms and identifiers should be validated by the provider, not generalized to legacy artifact types.
* **Live watch:** connecting to `wss://claude.ai/edge-api/frame-live/{uuid}/ws` with subprotocols `frame-live.v1` and the subscription token succeeded and emitted an initial `{\"ver\": ...}` and a presence message. The Claude Code client's parser also handles comment signals; a separate `frame-sync.v1` route exists. No live change or comment message was induced during this research. A session-owned webhook-based durable wake path is distinct from a local socket.
* **Comments:** observed `GET /api/frame/comments/{uuid}` returned an empty thread list for the example. Client code and earlier records show reply through `POST /api/frame/comments/{uuid}/{thread}`, and resolution paths including `/session-resolve`; the community client describes an idempotency ID and activation restrictions. Its reply body is `{\"id\": \"<client-uuid>\", \"text\": \"...\", \"answers_summon\": true}` when answering a summon, and it expects a `comment_id` in the result; its resolve body is `{\"resolved\": true}`. No comment mutation or retry was performed in the live research.
* **Runtime capabilities:** the user-visible page runtime (`claude.use(...)`) is a separate contract from the agent/provider API. The observed server supplied runtime contract `0.2.96`; a stored pin and existing declarations matter on redeploy. The included historical skill copy references `0.2.91` and is not a substitute for fetching the current contract.

The binary examined for this snapshot was Claude Code 2.1.318, SHA-256 `708c77b204db30086ff6f1ed34577bda03e1be74ced80557cbf7b54249ebf0bd`. The following manifest is a **schematic** create request derived from client code, not a tested request or a stable schema:

```json
{
  "title": "Review notes",
  "mode": "replace",
  "manifest": {
    "index.html": {"content": "<!doctype html>…", "contentType": "text/html"}
  }
}
```

For an update, the community client's manifest path adds the existing `slug` and `baseVersion` and uses `patch` by default. The content may require the client's page transformation, and server feature gates or contract versions may change what is accepted. The examples must not be treated as proof of a successful fresh create, update, or retry.

## Appendix B. Sources and related work

* [Official Claude Code artifacts documentation](https://code.claude.com/docs/en/artifacts) — documented experience, access and page behavior.
* [MCP specification, revision 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28) and [2025-11-25 resources](https://modelcontextprotocol.io/specification/2025-11-25/server/resources) — standards-based tools and resource notifications, subject to host support and protocol revision.
* [BCP 14 / RFC 8174](https://www.rfc-editor.org/rfc/rfc8174.html) — requirements language.
* [Community CLI](https://github.com/nikships/claude-artifact-cli) — private API client including comments and polling watch; its existence does not establish push integration with an agent host.
* [Community MCP implementation](https://github.com/kamilio/claude-artifacts) — publishing-oriented tool interface; inspected implementation did not include a comments/watch path.
* [Community runtime analysis](https://github.com/tarqd/claude-artifact-harness/blob/main/docs/analysis/shell.md) — independent reverse engineering of viewer runtime and transports; not an Anthropic-hosted publishing client.
* [Skill provenance](skills/README.md) — preserved Claude skill copies and limitations.
