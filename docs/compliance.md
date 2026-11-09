# ScopeGuard AI -- Compliance Mapping

## The Problem

Organizations adopting AI tools face a structural compliance gap: LLM API calls transmit user-provided text to third-party infrastructure. When that text contains PII or PHI, the transmission may violate data protection regulations regardless of the provider's security posture.

The risk is not that the provider is insecure. The risk is that the data left your control at all.

## How ScopeGuard Addresses It

ScopeGuard tokenizes PII/PHI **before** the API call leaves your environment. The LLM provider receives tokens, not data. On the return path, ScopeGuard restores original values **after** the response enters your environment. At no point does real PII transit the external network.

## Regulatory Mapping

### HIPAA (Health Insurance Portability and Accountability Act)

| Requirement | ScopeGuard Control |
|-------------|-------------------|
| PHI must be de-identified before disclosure to non-covered entities (45 CFR 164.514) | ScopeGuard tokenizes PHI entity types (names, MRNs, SSNs, dates of birth) before they reach the LLM provider |
| Minimum Necessary standard (45 CFR 164.502(b)) | Scope configurations allow only the entity types needed for each workflow, blocking everything else |
| Audit controls (45 CFR 164.312(b)) | Audit ledger records every detection and tokenization event with timestamp, entity type, and action |

### PCI DSS (Payment Card Industry Data Security Standard)

| Requirement | ScopeGuard Control |
|-------------|-------------------|
| Protect stored cardholder data (Req. 3) | Credit card numbers are tokenized with irreversible tokens outside the vault boundary |
| Restrict access to cardholder data (Req. 7) | Vault access is session-scoped; detokenization requires the original session ID |
| Track and monitor all access (Req. 10) | Audit ledger captures every tokenization/detokenization event |

### GDPR (General Data Protection Regulation)

| Requirement | ScopeGuard Control |
|-------------|-------------------|
| Data minimization (Art. 5(1)(c)) | Scopes enforce minimum-necessary entity exposure per workflow |
| Technical safeguards for processing (Art. 32) | Tokenization is an appropriate technical measure for protecting personal data during AI processing |
| Records of processing activities (Art. 30) | Audit ledger provides per-event records of data handling |

### SOC 2

| Trust Services Criterion | ScopeGuard Control |
|-------------------------|-------------------|
| CC6.1 -- Logical access controls | Session-scoped vaults isolate token resolution |
| CC6.7 -- Restrict transmission of confidential information | Tokenization prevents PII from transiting external networks |
| CC7.2 -- Monitor system components for anomalies | Audit ledger provides detection and action records for monitoring |

## What ScopeGuard Does Not Replace

ScopeGuard is a technical control, not a compliance program. It does not replace:

- Data classification and inventory
- Privacy impact assessments
- Vendor risk assessments of LLM providers
- Employee training on data handling
- Incident response procedures
- Legal review of AI use policies

It provides one well-defined control (tokenize PII at the AI boundary) that auditors can evaluate, test, and verify.
