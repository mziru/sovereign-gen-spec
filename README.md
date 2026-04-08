# sovereign-gen Protocol Specification

A streaming attribution protocol for LLM-generated content. Ensures that when an LLM generates a response using retrieved source material, every piece of output is classified by its source relationship, streamed with provenance metadata before content, and validated for both integrity and fidelity.

## Problem

Retrieval-augmented generation connects LLMs to external content at inference time. On the retrieval side, systems like the [Sovereign Context Protocol](https://arxiv.org/abs/2603.27094) (SCP) are defining attribution-aware data access layers that log, license, and audit every content access.

On the generation side, citation quality is an active research area. Post-hoc verification approaches—[TREC RAG 2025](https://arxiv.org/abs/2603.09891), [VeriCite](https://arxiv.org/abs/2510.11394) (SIGIR-AP 2025), [CiteFix](https://arxiv.org/abs/2504.15629) (ACL 2025)—evaluate or correct citations after generation completes. Retroactive source tracing methods like [WASA](https://arxiv.org/abs/2310.00646) (ACL 2025) embed watermarks in training data that surface in generated text.

These approaches address important problems, but none define a protocol for delivering attribution *incrementally during streaming generation*. Specifically, existing work does not address:

- How retrieval-layer provenance (source identity, license, content fingerprint) flows through generation into per-element user-facing output
- How to deliver that provenance to the client before content arrives, enabling incremental citation rendering
- How to distinguish verbatim quotation from paraphrase in generated output, and why that distinction requires different validation thresholds
- How to combine deterministic provenance integrity checks with nondeterministic fidelity scoring as complementary validation layers

This specification addresses these gaps. It is retrieval-layer agnostic—it works with any system that serves content alongside source metadata.

## Overview

sovereign-gen defines three components:

1. **A streaming attribution protocol** (SSE) that delivers provenance metadata before content, enabling clients to render citations incrementally as the LLM generates.

2. **A two-layer validation model** separating deterministic provenance verification from nondeterministic fidelity scoring, with citation-type-aware interpretation.

3. **An output element model** where every piece of generated content is classified by its relationship to source material, and that classification determines how validation applies.

## 1. Streaming Attribution Protocol

### Transport

Server-Sent Events (SSE) over HTTP. Each event has an `event` type and a `data` payload (JSON).

### Event Types

| Event | Purpose | When emitted |
|-------|---------|-------------|
| `stream_start` | Opens the stream | Once, before any elements |
| `element_init` | Declares a new element with metadata | Once per element, before any content |
| `element_append` | Delivers a content delta | Zero or more per element |
| `element_done` | Marks an element complete | Once per element, after all appends |
| `stream_end` | Closes the stream | Once, after all elements |

### Event Schemas

**`stream_start`**

```json
{
  "response_id": "string",
  "query": "string"
}
```

**`element_init`**

```json
{
  "element_id": "string",
  "kind": "sourced | synthesis",
  "citation_type": "quote | paraphrase | null",
  "provenance": {
    "source_id": "string",
    "content_id": "string",
    "license_id": "string",
    "content_fingerprint": "string"
  } | null
}
```

- `kind`: `"sourced"` if attributed to retrieved content, `"synthesis"` if original.
- `citation_type`: Present only when `kind` is `"sourced"`. `"quote"` if the text reproduces the source verbatim; `"paraphrase"` if restated.
- `provenance`: Present only when `kind` is `"sourced"`. Links the generated element back to the specific source, content, license, and integrity fingerprint from the retrieval layer. Field names are generic; see [Retrieval Layer Compatibility](#retrieval-layer-compatibility) for mappings to specific systems.

**`element_append`**

```json
{
  "element_id": "string",
  "delta": "string"
}
```

**`element_done`**

```json
{
  "element_id": "string"
}
```

**`stream_end`**

```json
{
  "response_id": "string",
  "element_count": "integer"
}
```

### Ordering Guarantees

1. `stream_start` is the first event. `stream_end` is the last.
2. For each element: exactly one `element_init`, followed by zero or more `element_append`, followed by exactly one `element_done`.
3. **Provenance arrives before content.** An `element_init` event (carrying full provenance) is always emitted before any `element_append` for that element. A client can render a citation card or attribution badge the moment `element_init` arrives, then fill in text as `element_append` deltas stream in.
4. Elements are sequential. The next element's `element_init` does not arrive until the previous element's `element_done` has been emitted.

### Example Stream

```
event: stream_start
data: {"response_id":"resp-abc","query":"best street food in Bangkok"}

event: element_init
data: {"element_id":"el-001","kind":"sourced","citation_type":"paraphrase",
       "provenance":{"source_id":"creator-003","content_id":"cnt-034",
                     "license_id":"lic-xyz","content_fingerprint":"sha256:a1b2..."}}

event: element_append
data: {"element_id":"el-001","delta":"Elena Torres describes Bangkok's "}

event: element_append
data: {"element_id":"el-001","delta":"street food scene as centered around..."}

event: element_done
data: {"element_id":"el-001"}

event: element_init
data: {"element_id":"el-002","kind":"sourced","citation_type":"quote",
       "provenance":{"source_id":"creator-003","content_id":"cnt-034",
                     "license_id":"lic-xyz","content_fingerprint":"sha256:a1b2..."}}

event: element_append
data: {"element_id":"el-002","delta":"\"The night markets of Yaowarat Road "}

event: element_append
data: {"element_id":"el-002","delta":"offer the most authentic experience.\""}

event: element_done
data: {"element_id":"el-002"}

event: element_init
data: {"element_id":"el-003","kind":"synthesis","citation_type":null,
       "provenance":null}

event: element_append
data: {"element_id":"el-003","delta":"Both sources suggest that the best..."}

event: element_done
data: {"element_id":"el-003"}

event: stream_end
data: {"response_id":"resp-abc","element_count":3}
```

## 2. Output Element Model

Every piece of generated content is one of two types:

**Sourced elements** are attributed to a specific piece of retrieved content. They carry:
- Provenance linking back to the source identity, content, license, and integrity fingerprint from the retrieval layer
- A citation type declaring whether the text is a verbatim quote or a paraphrase

**Synthesis elements** are original LLM-generated content not attributable to a single source—transitions, comparisons, cross-source conclusions. They carry a reasoning field explaining why the content couldn't be attributed.

This classification is not metadata applied after the fact. It is determined during generation and delivered in `element_init` before any content streams. The type determines how downstream validation interprets the fidelity score.

## 3. Validation Model

Validation has two layers that address fundamentally different questions.

### Layer 1: Provenance Verification (Deterministic)

Provenance fields on sourced elements (`source_id`, `content_id`, `license_id`, `content_fingerprint`) are not generated by the LLM. They are passed through from the retrieval layer, attached by the attribution pipeline based on which source the model referenced. The LLM never sees or produces these values.

Provenance verification is a system-level integrity check: it confirms that the pipeline's bookkeeping is consistent with what was actually served during retrieval. These are binary pass/fail checks:

| Check | What it verifies |
|-------|-----------------|
| **Fingerprint** | Recompute hash of the source content; compare to `content_fingerprint` in the element's provenance |
| **Source ID** | Was this `source_id` present in any served retrieval response? |
| **Content ID** | Was this `content_id` present in any served retrieval response? |
| **License status** | Is the referenced license still active? |
| **License expiry** | Has the license expiration passed? |
| **Audit trail** | Does a retrieval audit record exist for this content access? |

A sourced element passes provenance verification only if all checks pass. Any failure indicates a break in the attribution chain—the element claims a source relationship that cannot be verified against the retrieval record.

### Layer 2: Fidelity Scoring (Nondeterministic)

Evaluates whether the generated text faithfully represents the source content. Uses a 0–4 scale:

| Score | Level | Meaning |
|-------|-------|---------|
| 4 | VERBATIM | Near-direct reproduction of source text |
| 3 | FAITHFUL | All claims supported by source, meaning preserved, no added facts |
| 2 | PARTIAL | Some claims supported, but includes unsupported specifics or shifted meaning |
| 1 | TANGENTIAL | Shares topic but could be guessed without the source |
| 0 | NONE | Unrelated, contradicted, or fabricated |

The scale is the same regardless of citation type. What differs is the **interpretation**:

### Score Interpretation by Citation Type

| Score | Quote | Paraphrase |
|-------|-------|------------|
| 4 VERBATIM | Valid quote | Suspicious—should be marked as quote |
| 3 FAITHFUL | Near-quote with minor differences—may need correction | Valid paraphrase |
| 2 PARTIAL | Not a valid quote | Paraphrase drifted from source |
| 1 TANGENTIAL | Not a valid quote | Weak relationship to source |
| 0 NONE | Not a valid quote | No relationship to source |

**Acceptability threshold:**
- A **quote** must score VERBATIM (4) to be acceptable.
- A **paraphrase** must score FAITHFUL (3) or above to be acceptable.
- **PARTIAL (2) is a significant failure for either type**—it means the generated text has drifted from its source in ways that undermine the attribution claim.

This interpretation model means a single scoring system serves both citation types without requiring separate rubrics or separate scoring models. The citation type, declared in `element_init`, tells the consumer how to read the number.

### Validator Requirements

The fidelity scoring interface is model-agnostic. Implementations may use:
- Heuristic methods (token overlap, embedding similarity)
- LLM-as-judge with any chat model
- Fine-tuned specialist models optimized for citation fidelity scoring

The only requirement is that the validator accepts a generated text, a source text, and a citation type, and returns a score on the 0–4 scale with an explanation.

## Retrieval Layer Compatibility

sovereign-gen is retrieval-layer agnostic. The provenance fields in `element_init` map to any system that serves content with source metadata:

| sovereign-gen field | SCP envelope field | Generic RAG equivalent |
|---|---|---|
| `source_id` | `attribution.creator_ids[]` | Document/author identifier |
| `content_id` | `attribution.content_ids[]` | Chunk/passage identifier |
| `license_id` | `license.license_id` | Access policy reference |
| `content_fingerprint` | `license.content_fingerprint` | Content hash (SHA-256, etc.) |

**SCP** ([Sovereign Context Protocol](https://arxiv.org/abs/2603.27094); Panchigar, Rush & Canabarro, 2026) is a natural fit. SCP's atomic response envelope—bundling data, attribution, license, and audit reference—maps directly to sovereign-gen's provenance fields. SCP's blocking audit invariant ensures the retrieval record exists before content is served; sovereign-gen's provenance verification then checks that record from the generation side, closing the end-to-end loop.

But any retrieval system that provides source identity, content identity, a content hash, and an access policy reference is compatible.

## Status

This specification describes protocol version 0.1.

### Implementation Notes

A reference implementation exists but is not yet public. Key feasibility points:

- **Streaming structured output with provenance-before-content is achievable with current LLM provider APIs.** The ordering guarantee (metadata before text) is enforced by schema field ordering in the structured output, not by post-processing. The implementation streams real token-by-token output, not buffered-then-chunked simulation.
- **Provenance verification adds negligible latency.** All six checks are hash comparisons and dictionary lookups against data already in memory from the retrieval step.
- **The protocol has been validated against multiple LLM providers.** Provider support for streaming structured output varies—not all providers emit true token-level deltas for structured responses.
- **Fidelity scoring is decoupled from streaming.** It runs post-hoc against completed elements and does not block the stream. Preliminary work on fine-tuning a small open model (9B parameters) for citation fidelity scoring is underway, targeting cost-effective validation at scale.

### Not yet specified

- **Inline validation events**: Provenance checks and fidelity scores emitted as SSE events during streaming (e.g., a `provenance_check` event after `element_done`).
- **Multi-modal content**: Extension to images, audio, and video elements.
- **Batch mode**: Non-streaming request/response for batch processing pipelines.

## License

This specification is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
