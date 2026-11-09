# ScopeGuard AI

**Real-time PII/PHI tokenization gateway for AI workloads.**

ScopeGuard AI sits between your applications and LLM providers, intercepting sensitive data before it leaves your environment and restoring it on the way back. No model retraining. No prompt engineering. Drop it in front of any API call and PII stops leaking.

Developed by [Griffin Cyber Solutions](https://griffincybersolutions.com).

---

## What It Does

ScopeGuard performs bidirectional tokenization at the API gateway layer:

- **Inbound (to LLM):** Detects and replaces PII/PHI entities with reversible tokens before the request reaches the model
- **Outbound (from LLM):** Restores original values in the response so the end user sees real data, but the model never did

The model receives `[US_SSN_5b87b2b5]` instead of `123-45-6789`. The user sees the original SSN in the response. The audit log records every substitution.

## Detection Engine

ScopeGuard uses a hybrid detection pipeline combining:

- **Pattern matching** for structured PII (SSNs, credit cards, phone numbers, IBANs, routing numbers)
- **Named Entity Recognition (NER)** for unstructured PII (names, addresses, medical terms, dates in context)
- **Configurable confidence thresholds** per entity type and per scope

Detection is deliberately aggressive. False positives are cheap (a tokenized non-PII value still works). False negatives are expensive (leaked PII is a compliance event). The engine is tuned accordingly.

## Supported Entity Types

| Category | Entities |
|----------|----------|
| **Identity** | Person names, email addresses, phone numbers, dates of birth |
| **Financial** | SSNs (all formats), credit card numbers, IBANs, bank routing numbers |
| **Medical** | Medical record numbers, NPI numbers, clinical terms in context |
| **Network** | IP addresses, URLs, MAC addresses |
| **Location** | Physical addresses, GPS coordinates |
| **Temporal** | Dates, timestamps (contextual -- tokenized when adjacent to identifying information) |

## Scopes

ScopeGuard supports configurable **scopes** that define which entity types to tokenize and at what confidence threshold. This lets different workflows apply different protection levels:

- A **clinical summary** scope might allow dates and locations through (needed for medical context) while blocking all patient identifiers
- A **billing** scope might allow person names and dates (needed for invoicing) while blocking SSNs and financial data
- A **security operations** scope might allow IPs, URLs, and email addresses through (needed for incident triage) while blocking everything else

Scopes are defined in a simple JSON configuration. No code changes required to add or modify scopes.

## Architecture

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│              │     │                  │     │              │
│  Your App    │────▶│  ScopeGuard AI   │────▶│  LLM API     │
│              │◀────│  (tokenize/      │◀────│  (Anthropic,  │
│              │     │   detokenize)    │     │   OpenAI,     │
│              │     │                  │     │   etc.)       │
└──────────────┘     └──────────────────┘     └──────────────┘
                            │
                     ┌──────┴──────┐
                     │ Token Vault │
                     │ + Audit Log │
                     └─────────────┘
```

ScopeGuard operates as a **transparent proxy** or as a **sidecar API**:

- **Proxy mode:** Point your LLM API calls at ScopeGuard instead of the provider. It tokenizes the request, forwards to the real provider, detokenizes the response, and returns it. Your application code doesn't change.
- **API mode:** Call `/tokenize` and `/detokenize` endpoints directly for fine-grained control. Useful for pipelines, batch processing, or integration with existing gateways.

## Token Vault

Every tokenization operation creates a reversible mapping stored in a per-session vault:

- Original value to token (for tokenization)
- Token to original value (for detokenization)
- Session isolation (one session's tokens cannot be resolved by another)
- Configurable TTL (vaults expire after a defined period)

Tokens are deterministic within a session: the same input produces the same token, so the model sees consistent references across a multi-turn conversation.

## Audit Trail

Every tokenization and detokenization event is logged:

- Timestamp
- Session ID
- Entity type detected
- Confidence score
- Scope applied
- Action taken (tokenized, allowed, blocked)

No original PII values are written to the audit log. The log records *that* a substitution happened and *what type* of entity was involved, not the entity's value.

## Integration Examples

ScopeGuard integrates with any application that makes LLM API calls. Some tested configurations:

- **Direct API proxy** for Anthropic Claude and OpenAI GPT models
- **Sidecar for AI agent frameworks** (intercept inbound/outbound messages at the agent boundary)
- **Browser extension companion** for web-based AI chat interfaces
- **MCP (Model Context Protocol) gateway** for tool-using AI agents

## Licensing

ScopeGuard AI is commercially licensed software. Licenses are cryptographically signed (Ed25519) and validated at runtime.

License tiers:
- **Evaluation:** Full functionality, 30-day window, no production use
- **Standard:** Single-instance deployment, annual renewal
- **Enterprise:** Multi-instance, custom scopes, priority support, SLA

For licensing inquiries: [chris@griffincybersolutions.com](mailto:chris@griffincybersolutions.com)

## Compliance Relevance

ScopeGuard addresses requirements in:

- **HIPAA** -- PHI must not be transmitted to third-party AI services without de-identification
- **PCI DSS** -- Cardholder data must be masked/tokenized outside the cardholder data environment
- **GDPR** -- Personal data sent to AI processors requires appropriate technical safeguards
- **SOC 2** -- Sensitive data handling controls are a core trust services criterion
- **State privacy laws (CCPA, CPRA, etc.)** -- Consumer PII transmitted to AI services falls under data protection obligations

ScopeGuard does not replace a compliance program. It provides a technical control that auditors can point to.

## Relationship to IntentGuard

ScopeGuard is the **data-governance** complement to [IntentGuard](https://github.com/Th3Gr1ff/intentguard-enterprise), Griffin Cyber Solutions' **action-governance** gateway for AI agents.

| | IntentGuard | ScopeGuard |
|---|---|---|
| **Controls** | What the AI agent is allowed to *do* | What data the AI agent is allowed to *see* |
| **Mechanism** | Policy-based action approval (OPA) | PII/PHI tokenization at the API boundary |
| **Threat model** | Unauthorized tool use, prompt injection leading to action | Data leakage, PII exposure to third-party models |

Together they form a complete governance layer: IntentGuard gates the actions, ScopeGuard gates the data.

## About Griffin Cyber Solutions

Griffin Cyber Solutions provides security testing, AI governance tooling, and OSSTMM-based security assessments. Founded by Chris Griffin, longtime ISECOM colleague and OSSTMM practitioner.

- Web: [griffincybersolutions.com](https://griffincybersolutions.com)
- Email: [chris@griffincybersolutions.com](mailto:chris@griffincybersolutions.com)
- GitHub: [github.com/Th3Gr1ff](https://github.com/Th3Gr1ff)
