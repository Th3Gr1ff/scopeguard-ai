# ScopeGuard AI -- Architecture Overview

## Design Principles

1. **Fail-closed by default.** If the tokenization engine cannot determine whether a value is PII, it tokenizes it. False positives are operationally cheap. False negatives are compliance events.

2. **Stateless request path, stateful vault.** Each API request is processed independently. The only shared state is the token vault, which maps tokens back to original values for detokenization. The vault is session-scoped and TTL-bound.

3. **Provider-agnostic.** ScopeGuard does not depend on any specific LLM provider's API. It operates at the HTTP layer. If your application can make an HTTP request to an LLM, ScopeGuard can sit in front of it.

4. **No model dependency.** ScopeGuard does not call an LLM to detect PII. Detection uses deterministic pattern matching and a local NER model. This means detection latency is sub-100ms, detection does not cost API tokens, and detection works offline.

## Component Layout

```
                    ┌─────────────────────────────────────────┐
                    │            ScopeGuard Core               │
                    │                                         │
  Request ─────▶   │  ┌───────────┐   ┌──────────────────┐  │  ─────▶ LLM Provider
                    │  │  Analyzer │──▶│  Tokenizer       │  │
  Response ◀────   │  │  Engine   │   │  (replace/restore)│  │  ◀───── LLM Provider
                    │  └───────────┘   └──────────────────┘  │
                    │        │                │               │
                    │        ▼                ▼               │
                    │  ┌───────────┐   ┌──────────────────┐  │
                    │  │  Scope    │   │  Token Vault     │  │
                    │  │  Resolver │   │  (per-session)   │  │
                    │  └───────────┘   └──────────────────┘  │
                    │        │                │               │
                    │        ▼                ▼               │
                    │  ┌──────────────────────────────────┐  │
                    │  │         Audit Ledger              │  │
                    │  └──────────────────────────────────┘  │
                    └─────────────────────────────────────────┘
```

### Analyzer Engine

The detection pipeline runs in two stages:

1. **Regex recognizers** fire first, catching structured patterns (SSN formats, credit card numbers, phone numbers). These are fast and high-confidence.
2. **NER recognizers** fire second, catching unstructured entities (person names, locations, medical terms). These use a local spaCy model and produce confidence scores.

Results are merged, deduplicated, and filtered against the active scope's entity-type allowlist and confidence thresholds.

### Scope Resolver

Each request carries a scope identifier (defaulting to the most restrictive scope). The resolver loads the scope definition, which specifies:

- Which entity types are allowed to pass through untokenized
- The minimum confidence threshold for tokenization per entity type
- Any custom recognizer overrides

### Token Vault

The vault is a bidirectional map:

- `real_value -> token` (used during tokenization)
- `token -> real_value` (used during detokenization)

Tokens are formatted as `[ENTITY_TYPE_hash]` where the hash is a short deterministic digest. This preserves the entity type in the token (useful for downstream processing that needs to know "this was a person name" without knowing the name) while making the token irreversible without vault access.

### Audit Ledger

Append-only log. Every detection, tokenization, and detokenization event is recorded with:

- ISO 8601 timestamp
- Session identifier
- Entity type
- Action taken
- Scope that was active
- Confidence score (for NER-detected entities)

The ledger never contains original PII values. It records entity types and actions, not content.

## Deployment Modes

### Standalone API

Run ScopeGuard as an independent service. Applications call `/tokenize` and `/detokenize` endpoints directly.

### Transparent Proxy

Run ScopeGuard as a proxy in front of an LLM provider. Applications point their base URL at ScopeGuard. Requests are tokenized, forwarded, and responses are detokenized transparently.

### Sidecar

Run ScopeGuard alongside an AI agent or chatbot. The agent's message bus routes through ScopeGuard's filter endpoint before messages reach the agent and after the agent produces responses.

This mode is particularly effective for AI agents that use tools (MCP, function calling, etc.), where both the agent's reasoning and the tool results may contain PII.
