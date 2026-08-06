# client-audit/1.0

**Extension name:** `sovereign-gen.client-audit/1.0`

**Status:** Draft — open RFC. Co-authored with the [Sovereign Context Protocol](https://arxiv.org/abs/2603.27094) team. Not yet authoritative; expect breaking changes in §5.

**Applies to:** SCP/1.0 servers and consumers.

## Problem

SCP/1.0 servers record what they served (`scp_access` rows). Consumers that
ingest served content into LLM context, then produce output citing it, sit
on the other half of an audit loop that the base protocol can't observe:

- Was the content the agent retrieved actually used in the model's context, or
  discarded after re-ranking?
- Did the produced output cite the source verbatim, paraphrase it, link to it,
  or omit it entirely?
- What query did the agent actually send to retrieval — the user's literal
  input, or a rewritten variant, and how did the rewrite change semantics?

A creator agreeing to license content to AI consumers via SCP can audit what
left their server. They cannot audit what the consumer did with it. This
extension closes that gap by letting the consumer report back, with the
reported events persisted alongside `scp_access` rows so the full loop —
request → retrieval → context use → citation — is queryable through the
standard `getAccessLog`.

## 1. Manifest declaration

A conforming server advertises this extension in the manifest's `extensions`
field:

```json
{
  "protocol": "SCP/1.0",
  "endpoint": "https://.../scp/v1",
  "extensions": ["sovereign-gen.client-audit/1.0"]
}
```

Consumers MUST verify the declaration before calling `recordEvents`.

Servers MAY ignore `recordEvents` calls (returning `404` / `400`) if they do
not implement this extension. Consumers SHOULD treat such responses as soft
failures — log and continue — rather than break the user's session over an
unaudited turn.

## 2. Method: `recordEvents`

**Wire path:** `POST {endpoint}/recordEvents`
**Authentication:** the server's standard SCP authentication (e.g.
`X-SCP-API-Key`).

### 2.1 Request

```json
{
  "events": [
    {
      "type": "context_use" | "citation" | "query_rewrite",
      "audit_log_id": "log-<uuid>",
      "license_id": "log-<uuid>",
      "content_id": "<canonical content URI; omitted for query_rewrite>",
      "meta": { /* type-specific keys, see §3 */ }
    }
  ]
}
```

- `audit_log_id` references the originating `scp_access` event written when
  the SCP server served the content this event relates to. It is the parent
  pointer that lets `getAccessLog` reconstruct the loop.
- `license_id` carries the same value as `audit_log_id` for consumer-emitted
  events under v1.0. This is **the recorded-against handle**, not the
  envelope-level license id (`lic-<uuid>`) from `getCreatorContent`. See §5.1.
- `content_id` is the canonical content URI from the originating SCP
  envelope's `attribution.content_ids`. Omitted only for `query_rewrite`,
  which is about a query rather than a single content item.
- `meta` is a freeform JSON object. Type-specific conventional keys are
  defined in §3; servers MUST preserve unknown keys.

The batch is **atomic**. A validation failure on any event in the batch
fails the entire request; no events are recorded.

### 2.2 Response

Standard SCP envelope wrapping:

```json
{
  "recorded": [
    { "event_id": "log-<uuid>", "license_status_at_record": "active" | "revoked" | "expired" }
  ],
  "n_events": 1
}
```

Servers SHOULD resolve the license referenced by `audit_log_id` at record
time and stamp the row's `meta` with `license_status_at_record`. This
preserves usage-against-revoked-license records rather than refusing them —
an essential property for the audit loop: violations must be capturable,
not erased.

The envelope's `audit_log_id` SHOULD reference the first recorded event in
the batch, giving the consumer a handle for the batch as a whole.

### 2.3 Errors

| Status | Cause |
|--------|-------|
| `400` | Validation failure (unknown event `type`, missing required field, empty batch, malformed meta). |
| `401` / `403` | Authentication / authorization failure. |
| `5xx` | Server-side failure. Consumers SHOULD retry with exponential backoff. |

## 3. Event types

### 3.1 `context_use`

Emitted when retrieved content actually enters the LLM's context — i.e.,
when a tool result is being handed back to the model, not merely fetched.

| field | requirement | meaning |
|---|---|---|
| `content_id` | required | canonical URI of the retrieved item entering context |
| `audit_log_id` | required | `scp_access` event for the originating retrieval |
| `meta.tool` | recommended | identifier of the consumer-side retrieval mechanism that produced the content (string; consumer-defined vocabulary) |
| `meta.link` | optional | resolved permalink, when distinct from `content_id` |

A single retrieval call (e.g. one `getCreatorContent` semantic search
returning N hits) emits one `context_use` per content_id the model receives.

### 3.2 `citation`

Emitted when the consumer commits a reference to source content into the
generated output. The defining moment is *output commitment* — the content
is now part of what the user sees, not merely considered during generation.

| field | requirement | meaning |
|---|---|---|
| `content_id` | required | canonical URI of the referenced content |
| `audit_log_id` | required | `scp_access` event for the originating retrieval |
| `meta.citation_type` | recommended | consumer-supplied label classifying how the content was used. Conventional values include `"quote"` (verbatim reproduction) and `"link"` (directional reference); other values are permitted. Servers MUST NOT reject unrecognized values. |
| `meta.fingerprint` | recommended | content fingerprint captured at retrieval time, anchoring the citation to a verifiable source state |
| `meta.quote_text` | recommended when the consumer claims verbatim use | the literal text reproduced in output, enabling server-side substring verification against the fingerprinted source |
| `meta.link` | optional | resolved permalink |

This extension does not prescribe a consumer-side output model. Whether the
consumer represents its output as elements, sections, blocks, or any other
unit is out of scope; the audit event fires when *something* in that model
references source content.

### 3.3 `query_rewrite`

Emitted when the consumer transformed the user's input before sending it
to retrieval — for any reason (expansion, normalization, intent
restatement, controlled-vocabulary alignment, etc.). The mechanism is out
of scope; the event records that a transformation occurred and what the
inputs and outputs were.

| field | requirement | meaning |
|---|---|---|
| `content_id` | omitted | this event is about a query, not a content item |
| `audit_log_id` | required | the `scp_access` event for the search that the rewritten query produced |
| `meta.original_query` | required | the user's literal input |
| `meta.agent_query` | required | the string the agent actually sent to retrieval |

A `query_rewrite` event SHOULD be emitted even when the agent left the
query unchanged — the trivial case is also signal, and recording it
distinguishes "agent chose not to rewrite" from "agent never ran retrieval."

## 4. Read-back via `getAccessLog`

Recorded events MUST be returnable through the standard SCP `getAccessLog`
method, interleaved with `scp_access` rows in chronological order, so a
single read reconstructs the full audit loop. Storage layout is an
implementation concern; the requirement is on the read interface.

When `getAccessLog` returns rows from this extension, each row's `type`
field carries the literal extension event type (`context_use`, `citation`,
`query_rewrite`) without prefixing or transformation.

## 5. Open issues

These are flagged for joint resolution before v1.0 is locked. Implementers
who hit them in the meantime should document their choice.

### 5.1 `license_id` wire field naming

The wire field `license_id` on consumer-emitted events (§2.1) carries the
parent `audit_log_id` — a `log-…` value, not a license id (`lic-…`). The
field is named for backward compatibility with an earlier draft, and
conflicts with the envelope-level `license.license_id` returned by SCP/1.0
content methods.

**Proposed resolution:** rename the wire field to `audit_log_id` (matching
its semantic meaning) and reserve `license_id` strictly for `lic-…` values.
Breaking change; bumps the extension to `client-audit/1.1`.

### 5.2 Persisted-row schema

Servers that persist the parent reference SHOULD store it under a column
name aligned with the wire decision in §5.1. Reads (via `getAccessLog`)
return rows in the same shape they were written. Open until §5.1 lands.

### 5.3 Session scoping as a separate extension

Concurrent consumer sessions against a single SCP server produce
interleaved audit rows that downstream readers may need to demultiplex
(e.g. attribute events to a specific user interaction). One viable
approach:

1. Consumer sends an `X-Session-Id` header on every SCP method call.
2. Server writes the header value into `meta.session_id` on `scp_access` rows.
3. Consumer also stamps `meta.session_id` on every consumer-emitted event.
4. `getAccessLog` callers can filter on `meta.session_id`.

This is orthogonal to attribution and likely belongs in a sibling extension
(`sovereign-gen.session-scoping/1.0` or similar) rather than baked into
`client-audit/1.0`. **Decision needed.**

### 5.4 License status enum vocabulary

§2.2 defines `license_status_at_record ∈ {active, revoked, expired}`. Is
this the right vocabulary? Should `unknown` be included for events the
server records before the license can be resolved (clock skew, replication
lag)? Should `paused` be a separate state from `expired`?

### 5.5 Meta key formalization

§3 defines conventional `meta` keys but does not enforce them schemaically.
Stricter typing (e.g. JSON Schema for each event type's meta) would catch
malformed batches at the wire boundary; freeform meta preserves extensibility
without a spec bump. **Open: do we ship a meta schema in v1.0 or keep it
conventional?**

## 6. Versioning

Versions follow `client-audit/MAJOR.MINOR`. `client-audit/1.x` MUST be
wire-compatible: a server supporting `1.0` MUST accept events from a `1.1`
consumer (ignoring unknown fields) and vice versa. Breaking changes (e.g.
the field rename in §5.1) bump the major version.

Consumers and servers SHOULD declare their supported version range via the
manifest's `extensions` field. A server advertising
`sovereign-gen.client-audit/1.0` is making no claim about higher versions;
a consumer wanting `1.1` semantics SHOULD verify the server declares `1.1`.

---

## Changelog

- `1.0-draft.1` — initial RFC. Open issues §5.1–§5.5 deferred for joint
  resolution.
