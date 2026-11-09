# ScopeGuard AI -- Architecture Overview

## Design Principles

1. **Fail-closed by default.** If the tokenization engine cannot determine whether a value is PII, it tokenizes it. False positives are operationally cheap. False negatives are compliance events.

2. **Stateless request path, stateful vault.** Each API request is processed independently. The only shared state is the token vault, which maps tokens back to original values for detokenization. The vault is session-scoped and TTL-bound.

3. **Provider-agnostic.** ScopeGuard does not depend on any specific LLM provider's API. It operates at the HTTP layer. If your application can make an HTTP request to an LLM, ScopeGuard can sit in front of it.

4. **No model dependency.** ScopeGuard does not call an LLM to detect PII. Detection uses a local engine that does not require API tokens, runs offline, and adds sub-100ms latency.

## High-Level Flow

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│              │     │                  │     │              │
│  Your App    │────>│  ScopeGuard AI   │────>│  LLM API     │
│              │<────│                  │<────│              │
│              │     │                  │     │              │
└──────────────┘     └──────────────────┘     └──────────────┘
                            │
                     ┌──────┴──────┐
                     │ Token Vault │
                     │ + Audit Log │
                     └─────────────┘
```

ScopeGuard intercepts requests on the way to the LLM provider (tokenizing sensitive data) and responses on the way back (restoring original values). The LLM provider never sees real PII. The end user sees real values. The audit log records every substitution.

## Core Components

### Detection Engine

ScopeGuard uses a hybrid detection pipeline that combines pattern-based recognizers for structured data (SSNs, credit cards, phone numbers) with NER-based recognition for unstructured entities (names, locations, medical terms). Detection is tuned aggressively: the cost of a false positive (tokenizing a non-PII value) is low, while the cost of a false negative (leaking PII) is a compliance event.

### Scope Resolver

Each request specifies a scope that controls which entity types are tokenized and which are allowed through. Scopes are defined in configuration and can be changed without code modifications. The most restrictive scope is always the default.

### Token Vault

Tokens are reversible within a session: the same input consistently produces the same token, so the LLM sees stable references across multi-turn conversations. Detokenization requires the original session's vault. Vaults are isolated per session and expire after a configurable TTL.

### Audit Ledger

Every detection and tokenization event is logged with timestamp, session, entity type, action taken, and the active scope. The ledger never contains original PII values. It records that a substitution happened and what type of entity was involved, not the entity's content.

## Deployment Modes

### Standalone API

Run ScopeGuard as an independent service. Applications call tokenization and detokenization endpoints directly.

### Transparent Proxy

Run ScopeGuard as a proxy in front of an LLM provider. Applications point their base URL at ScopeGuard. Requests are tokenized, forwarded, and responses are detokenized transparently.

### Sidecar

Run ScopeGuard alongside an AI agent or chatbot. The agent's message bus routes through ScopeGuard before messages reach the agent and after the agent produces responses.

This mode is particularly effective for AI agents that use tools (MCP, function calling, etc.), where both the agent's reasoning and the tool results may contain PII.
