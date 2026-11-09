# ScopeGuard AI -- Integration Guide

## API Endpoints

ScopeGuard exposes a REST API. All endpoints accept and return JSON.

### POST /tokenize

Submit text for PII/PHI detection and tokenization. Specify a scope to control which entity types are tokenized and which are allowed through. Returns the filtered text with sensitive entities replaced by reversible tokens, along with a summary of what was detected and what action was taken.

### POST /detokenize

Submit tokenized text to restore original values. Requires the session ID from the original tokenization call. Tokens are consistent within a session, so the same entity always resolves to the same original value.

### POST /analyze

Detect PII/PHI entities in text without performing tokenization. Returns entity types, confidence scores, and positions. Useful for auditing, dashboards, or pre-flight checks before sending data to an LLM.

### POST /proxy

Full transparent proxy mode. Submit a complete LLM API request and ScopeGuard handles the full round-trip: tokenize the request, forward it to the provider, detokenize the response, and return it. Your application code does not need to handle tokenization at all. Supports multiple LLM providers.

### GET /health

Health check with upstream dependency status.

### GET /scopes

List available scopes and their configurations.

### GET /vault/{session_id}

Inspect the token vault for a session (admin access required). Useful for debugging and auditing.

## Scopes

Scopes define per-workflow tokenization policies: which entity types to protect, which to allow through, and at what confidence level. Different workflows can apply different protection levels without code changes.

Examples of scope differentiation:

- A **clinical** scope might allow dates and locations through (needed for medical context) while blocking all patient identifiers
- A **billing** scope might allow person names and dates (needed for invoicing) while blocking SSNs and financial instruments
- A **security operations** scope might allow network indicators through (needed for triage) while blocking personal identity data
- A **maximum protection** scope blocks everything above a low confidence threshold

Scopes are defined in configuration, not code. Adding or modifying a scope requires no deployment.

## Deployment Modes

ScopeGuard supports three deployment patterns:

### Standalone API

Run ScopeGuard as an independent service. Your applications call the tokenization and detokenization endpoints directly, with full control over when and how tokenization happens in your pipeline.

### Transparent Proxy

Run ScopeGuard as a drop-in proxy in front of your LLM provider. Point your existing API base URL at ScopeGuard. Requests and responses are processed transparently. No application code changes required.

### Agent Sidecar

Run ScopeGuard alongside an AI agent or chatbot. Inbound messages are tokenized before the agent sees them. Outbound responses are detokenized before the user sees them. The agent reasons over tokens, never over real PII.

This mode is particularly effective for AI agents that use tools (MCP, function calling, etc.), where both the agent's reasoning and the tool results may contain sensitive data.

## Getting Started

ScopeGuard is commercially licensed. Contact [chris@griffincybersolutions.com](mailto:chris@griffincybersolutions.com) for evaluation access, integration support, and pricing.
