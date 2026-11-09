# ScopeGuard AI -- Integration Guide

## API Endpoints

ScopeGuard exposes a REST API with the following endpoints:

### POST /tokenize

Detect and replace PII/PHI entities in text.

**Request:**
```json
{
  "text": "Patient John Smith, SSN 123-45-6789, was admitted on 2026-03-15.",
  "scope": "clinical_summary",
  "session_id": "session-abc-123"
}
```

**Response:**
```json
{
  "filtered_text": "Patient [PERSON_a1b2c3d4], SSN [US_SSN_5b87b2b5], was admitted on 2026-03-15.",
  "entities_found": [
    {
      "entity_type": "PERSON",
      "score": 0.92,
      "action": "tokenized"
    },
    {
      "entity_type": "US_SSN",
      "score": 1.0,
      "action": "tokenized"
    },
    {
      "entity_type": "DATE_TIME",
      "score": 0.85,
      "action": "allowed_by_scope"
    }
  ],
  "session_id": "session-abc-123"
}
```

Note: The date was detected but allowed through because the `clinical_summary` scope permits `DATE_TIME` entities.

### POST /detokenize

Restore original values from tokens.

**Request:**
```json
{
  "text": "Patient [PERSON_a1b2c3d4] has been discharged. Please notify [PERSON_a1b2c3d4] at their home address.",
  "session_id": "session-abc-123"
}
```

**Response:**
```json
{
  "restored_text": "Patient John Smith has been discharged. Please notify John Smith at their home address.",
  "tokens_resolved": 2
}
```

Note: The same token resolves to the same value throughout the session, so the model's references stay consistent.

### POST /analyze

Detect PII/PHI entities without tokenizing. Useful for auditing or building dashboards.

**Request:**
```json
{
  "text": "Contact Jane Doe at jane@example.com or 555-0123.",
  "scope": "default"
}
```

**Response:**
```json
{
  "entities": [
    {"entity_type": "PERSON", "score": 0.95, "start": 8, "end": 16},
    {"entity_type": "EMAIL_ADDRESS", "score": 1.0, "start": 20, "end": 36},
    {"entity_type": "PHONE_NUMBER", "score": 0.88, "start": 40, "end": 48}
  ]
}
```

### POST /proxy

Full transparent proxy mode. Send a complete LLM API request; ScopeGuard tokenizes it, forwards to the provider, detokenizes the response, and returns it.

**Request:**
```json
{
  "provider": "anthropic",
  "scope": "default",
  "session_id": "session-abc-123",
  "request_body": {
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "Summarize the medical history for patient John Smith, SSN 123-45-6789."}
    ]
  }
}
```

The model receives the tokenized version. The response is detokenized before reaching your application. Your code does not need to handle tokenization at all.

### GET /health

Health check with upstream dependency status.

### GET /scopes

List available scopes and their configurations.

### GET /vault/{session_id}

Inspect the token vault for a session (admin access required). Useful for debugging and auditing.

## Scope Configuration

Scopes are defined in a JSON file. Example:

```json
{
  "clinical_summary": {
    "description": "Allow dates and locations for clinical context",
    "default_action": "tokenize",
    "default_threshold": 0.7,
    "allow": ["DATE_TIME", "LOCATION"],
    "overrides": {
      "US_SSN": {"threshold": 0.2},
      "CREDIT_CARD": {"threshold": 0.3}
    }
  }
}
```

- `default_action`: What to do with detected entities (`tokenize` or `allow`)
- `default_threshold`: Minimum NER confidence score to trigger tokenization
- `allow`: Entity types that pass through untokenized in this scope
- `overrides`: Per-entity-type threshold overrides (lower = more aggressive detection)

## Integration Patterns

### Python (requests)

```python
import requests

SCOPEGUARD_URL = "http://localhost:8990"

def tokenize(text, scope="default", session_id=None):
    resp = requests.post(f"{SCOPEGUARD_URL}/tokenize", json={
        "text": text,
        "scope": scope,
        "session_id": session_id,
    })
    return resp.json()

def detokenize(text, session_id):
    resp = requests.post(f"{SCOPEGUARD_URL}/detokenize", json={
        "text": text,
        "session_id": session_id,
    })
    return resp.json()
```

### curl

```bash
# Tokenize
curl -X POST http://localhost:8990/tokenize \
  -H "Content-Type: application/json" \
  -d '{"text": "SSN: 123-45-6789", "scope": "default"}'

# Detokenize
curl -X POST http://localhost:8990/detokenize \
  -H "Content-Type: application/json" \
  -d '{"text": "[US_SSN_5b87b2b5]", "session_id": "your-session-id"}'
```

### Agent Sidecar Pattern

For AI agents (chatbots, copilots, autonomous agents), deploy ScopeGuard as a sidecar that intercepts the message bus:

```
User message ──▶ ScopeGuard /filter ──▶ Agent receives tokenized text
                                              │
Agent response ◀── ScopeGuard /send ◀── Agent produces response
```

The agent never sees real PII. The user sees real values. The audit log captures every substitution.
