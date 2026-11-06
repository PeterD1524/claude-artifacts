# Claude Artifacts API reference

This is an **unofficial, partial description of Anthropic's existing private API**, not a proposal for a new integration API and not an Anthropic-maintained specification. It documents the Claude Code frame path, not every legacy Claude chat artifact or every typed artifact. Behavior depends on account, authorization, product surface, feature flags and service version.

* [OpenAPI 3.2.1 description](openapi.yaml) — HTTP control plane and versioned source read.
* [AsyncAPI 3.1.0 description](asyncapi.yaml) — WebSocket messages from the **agent client's perspective** (`receive` means the client receives a message).
* [Integration design](../SPEC.md) — a separate, proposed client design. Its requirements do not define Anthropic's API.

The formats follow the [OpenAPI Specification](https://spec.openapis.org/oas/v3.2.1.html) and the [AsyncAPI Specification](https://www.asyncapi.com/docs/reference/specification/v3.1.0), including its [WebSocket binding](https://www.asyncapi.com/docs/reference/bindings/websockets). Schemas intentionally leave unverified fields optional and allow unknown properties. They are an evolving map, not proof of complete valid payloads or forward compatibility.

## Evidence and limits

`x-evidence` on each operation or event distinguishes:

| Value | Meaning |
| --- | --- |
| `observed-read` | A read-only request or the initial socket message succeeded against one artifact/account on 2026-11-06. |
| `client-code` | The behavior is present in the Claude Code 2.1.318 binary; it may be gated or unused in a given session. |
| `community-client` | An independent [implementation](https://github.com/nikships/claude-artifact-cli/tree/5cecd0c8b4fcfb94575b6d581b17b50f6d13441d) uses or describes the operation. We did not independently verify it. |
| `client-code-and-community-client` | Both describe it; it was not newly mutation-tested. |

The inspected Claude Code binary's SHA-256 was `708c77b204db30086ff6f1ed34577bda03e1be74ced80557cbf7b54249ebf0bd`. Earlier records also show successful publish and comment handling. No new create, update, conflict, comment write, sharing change, or delete was performed for this research. The WebSocket probe established a connection and received an initial version and presence; it did **not** observe a real change or comment event. Validation of these description files checks their format, not whether Anthropic implements them.

## Origins, credentials and identifiers

The control plane used `https://api.anthropic.com` and a Claude account OAuth bearer, with `anthropic-beta: oauth-2025-04-20`. The inspected clients also send `X-Frame-CP: go`, `X-Frame-Surface: code`, `X-Frame-Platform: cli`, `X-Frame-Client-Version`, and `Accept: application/json, application/vnd.ant.frame-refusal+json`. On `/api/frame/contract/latest` specifically, bearer plus beta yielded 404; adding only `X-Frame-CP: go` yielded 200. That is evidence for this endpoint and account, not proof of which headers every other route requires. An Anthropic platform API key was not established to authenticate these routes.

Boot returns distinct short-lived credentials: an `assetToken` for `https://<uuid>.frame.claudeusercontent.com` and a `subscriptionToken` for the observed live socket. Do not send the OAuth bearer to the content host or treat those tokens as interchangeable. Do not publish token-bearing response examples, query strings or WebSocket headers. A missing or expired token is not itself evidence that an artifact was deleted.

The URL form `https://claude.ai/code/artifact/<uuid>` contains the canonical UUID. In the observed artifact, `https://claude.ai/artifact/<short-id>` encoded that same UUID as a base-58 integer using the Bitcoin alphabet; the community client implements that conversion. This does not establish that legacy `/public/artifacts/...` routes use the same contract.

## Create, read and update

1. `POST /api/frame/deploy/direct` can create a frame when no existing slug is supplied, or publish a new version when a slug is supplied. The binary has a single-content path; the community client uses a manifest path. They are separate request forms, and the full schema is unknown.
2. `GET /api/frame/{slug}?via=model_read` booted the observed artifact and returned its current `ver`, `files`, role and short-lived tokens. `GET /api/frame/read/{slug}` returned metadata and `GET /api/frame/versions/{slug}` returned history.
3. The observed editable source came from `https://{slug}.frame.claudeusercontent.com/_f/{ver}/_src/{path}?__frame_t={assetToken}`. The local source was an HTML fragment; the published source contained that fragment inside a generated HTML shell. Consequently, published source bytes need not equal the original local input bytes. The served view may include injected runtime material; it is not necessarily the stored source.
4. Community code describes manifest `replace` as replacing the set, `patch` as overlaying changes on `baseVersion`, and `null` as a patch removal. A stale base can return 409. The client may stage large supporting files using `/deploy/prepare` and `/upload` before the final deploy. Its direct path and binary retry code are documented in the OpenAPI operation descriptions; no fresh request was tested here.

The binary contains attempt IDs, resend handling and `publish_settling`. A lost response is not proof of failure. The exact idempotency guarantees are unresolved; do not assume a newly generated attempt ID safely retries the same create.

Capability declaration and runtime contract pin are distinct from file content. The community client reports that omission preserves the stored declaration, `{}` clears it and a nonempty declaration replaces the set. Verify this against the active service and client before changing capability declarations.

## Comments and live notifications

`GET /api/frame/comments/{slug}` returned `threads: []` for the observed artifact. The fields for nonempty threads, reply body, idempotent client ID and `/session-resolve` behavior in OpenAPI come from the Claude Code client or a community implementation. An OAuth reply can be restricted to a thread a person explicitly sent to Claude. The community client reports a 403 for non-activated threads and says that creating a human thread and ordinary browser-side resolution require a claude.ai session.

The browser shell and page capabilities also have comment-create and resolve operations. Independent [shell analysis](https://github.com/tarqd/claude-artifact-harness/blob/main/docs/analysis/shell.md) describes `POST /api/frame/comments/{slug}` and `POST /api/frame/comments/{slug}/{thread}/resolve` with capability/session context. Their browser authentication and complete request contracts have not been independently verified, so the OAuth-oriented OpenAPI file does not pretend these are unauthenticated agent operations. The page's `claude.use("comments")` is a separate in-frame API.

For `frame-live.v1`, the observed boot token was offered as the second WebSocket subprotocol to `wss://claude.ai/edge-api/frame-live/{slug}/ws`; the first was `frame-live.v1`. The server selected `frame-live.v1` and sent a version, then presence. Static client code recognizes `kind: "comment"` and version-bearing messages. A signal prompts a fresh HTTP read; it is not proven to contain the full comment. The binary also contains a separate `frame-sync.v1` transport with `ctrl:comment`, `ctrl:invalidate` and token refresh controls. Durable server-side wake subscriptions are distinct from either local connection.

## Errors and unmodeled operations

The client distinguishes unauthorized and forbidden responses, 404, 409 conflicts and rate limits, but the same HTTP status can have different causes. In particular, a 409 deploy can mean a stale version or `publish_settling`; a 404 can reflect routing or authorization. Error body shapes and every status code are not fully established, so generic responses in OpenAPI are deliberately permissive.

The files do not enumerate every internal browser, live-document, thumbnail, relay, share, database or typed-artifact method. Their omission does not imply nonexistence. The [runtime contracts](../runtime/README.md) describe capabilities granted to an artifact page, not a complete server API. The [skill reference](../skills/README.md) provides page-authoring context.

## Validation

Use the official AsyncAPI CLI and an OpenAPI 3.2-capable validator, for example:

```sh
uvx --from openapi-spec-validator openapi-spec-validator api/openapi.yaml
TEST=1 npx --yes @asyncapi/cli@5.0.7 validate api/asyncapi.yaml
```

Run from the repository root. These validate document structure and references, not remote compatibility or write behavior.
