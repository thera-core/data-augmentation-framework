# Data Augmentation Framework  
**Data Source Authentication & Security**  
`docs/security/data_source_auth.md`

## 1. Introduction

This document outlines **how** and **where** the **data-augmentation-framework** should store and use **credentials** or **tokens** for external data sources (e.g., PubMed, Finance APIs, HPC job schedulers). The goal is to ensure **secure handling**, **minimal exposure** of secrets, and consistent **authentication** across augmenters.

### 1.1 Purpose

- Describe **recommended practices** for storing API keys/secrets.  
- Explain how each augmenter should **retrieve** and **use** those credentials without embedding them in source code.  
- Ensure compliance with typical **security standards** (no secrets in plain-text code repositories, environment variables usage, etc.).

### 1.2 Scope

- **In-Scope**:  
  - Credentials for external data APIs (PubMed, Alpha Vantage, HPC job accounts, etc.).  
  - Best practices for key rotation, environment variable usage, encryption, and logging.  
- **Out-of-Scope**:  
  - Data encryption at rest, intrusion detection, or large-scale enterprise security solutions (handled by the broader infrastructure or DevSecOps processes).

---

## 2. Security Model Overview

```
+------------------------------------------------------+
|  Data Augmentation Framework (Augmenters, etc.)      |
|  - Retrieves secrets at runtime from secure storage  |
|  - Connects to external APIs (PubMed, HPC, etc.)     |
+------------------------------------------------------+
          |                      
          |  (secure retrieval)          
          v                      
+------------------------------------------------------+
|   Secure Storage (env vars, Vault, AWS Secrets)      |
|   - Holds API keys, HPC tokens, etc.                 |
|   - Enforces access control / rotation policies       |
+------------------------------------------------------+
```

1. **Secrets** (API keys, tokens, HPC credentials) are **never** stored directly in code or version control.  
2. **Augmenters** or the concurrency layer **retrieve** these secrets at runtime, typically from environment variables or a secure secrets manager (e.g., HashiCorp Vault, AWS Secrets Manager).  
3. **Connections** to external APIs must be **TLS/HTTPS** secured, with robust error handling and minimal logging of sensitive info.

---

## 3. Storing Credentials

### 3.1 Environment Variables

A common and straightforward approach:

- **`PUBMED_API_KEY`**: For PubMed e-utilities if needed (NCBI typically has an API key mechanism for higher rate limits).  
- **`ALPHAVANTAGE_API_KEY`**: For finance data from Alpha Vantage or similar.  
- **`HPC_TOKEN`** or **`HPC_USER_PASS`**: HPC job submission credentials.

**Pros**:
- Simple to manage on local dev machines or in Docker containers.  
- Infrastructure (CI/CD pipelines, runtime environment) can inject environment variables securely.

**Cons**:
- Risk of accidental printing in logs if not carefully sanitized.  
- Rotation can require re-deploying environment variables unless you have a higher-level secrets manager.

### 3.2 Secrets Manager (Vault, AWS, GCP)

For more **robust** or enterprise-grade setups:

- **HashiCorp Vault**: The framework can call a Vault client library to retrieve secrets at runtime.  
- **AWS Secrets Manager** or **AWS Parameter Store**: Common in AWS-based systems.  
- **GCP Secret Manager**: For Google Cloud environments.

**Recommendation**:  
- Use a specialized secrets manager if you handle many keys or require **dynamic rotation**, **audit logs**, or **fine-grained access control**.

### 3.3 Guidelines

1. **No Hardcoded Keys** in code.  
2. **Limited Access**: Only the processes that run the data-augmentation-framework should be able to read these environment variables or secrets.  
3. **Rotation**: Periodically update API keys to reduce long-term exposure risks.  
4. **Encryption at Rest**: If you store secrets in a file, ensure it’s encrypted or handled by your secrets manager.  

---

## 4. Retrieving & Using Credentials in Augmenters

### 4.1 Example: PubMedAugmenter

```python
# pubmed_augmenter.py (simplified example)

import os
import requests

class PubMedAugmenter(BaseAugmenter):
    def __init__(self, query_column="gene_symbol", **kwargs):
        super().__init__(**kwargs)
        # Retrieve key from env or fallback to empty if not required
        self.api_key = os.getenv("PUBMED_API_KEY", "")

    def augment_data(self, data, **kwargs):
        # ... ensure context adapter, etc. ...
        queries = data[self.query_column].unique()

        for q in queries:
            # If an API key is used, add it to the request
            pmids = self._pubmed_search(q, api_key=self.api_key)
            # ...
        return data

    def _pubmed_search(self, query, api_key):
        # Example usage of an optional key
        params = {
            "term": query,
            "api_key": api_key,
            # other parameters...
        }
        response = requests.get("https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi", params=params)
        # ...
        return [12345, 67890]
```

- The constructor looks up the environment variable **`PUBMED_API_KEY`**.  
- The method `_pubmed_search` includes that API key in the request **only if** the key is present.

### 4.2 FinanceAugmenter

```python
class FinanceAugmenter(BaseAugmenter):
    def __init__(self, ticker_column="ticker", api_source="AlphaVantage", **kwargs):
        super().__init__(**kwargs)
        self.api_key = os.getenv("ALPHAVANTAGE_API_KEY", "")
        # ...
```

### 4.3 HPC Integration

- HPC job schedulers might require a **username/password** or **token**.  
- For instance, store HPC credentials in environment variables (`HPC_USER`, `HPC_PASS`) or an OAuth token in a secure location.

---

## 5. Logging & Auditing

### 5.1 Logging

- **Never** log full credentials.  
- If an API request includes an API key, ensure logs do not print the key if an error occurs (scrub or redact sensitive fields).  
- Secure logs with appropriate access controls.

### 5.2 Auditing

- If using a secrets manager, rely on that system’s **audit logs** to track who accessed each secret.  
- If environment variables are used, keep them out of any debug dumps or trace logs in production.

---

## 6. Network Security

1. **Use HTTPS** for all external API calls to avoid sending tokens in plaintext over the network.  
2. Some HPC setups might rely on SSH or other secure protocols. Ensure HPC credentials are not sent in cleartext.  
3. If tokens must be refreshed (like OAuth tokens), ensure the refresh flow is **secured** and tested.

---

## 7. Key Rotation & Expiration

- **Rotation**: Periodically replace API keys or HPC tokens (e.g., every 90 days) to reduce long-term exposure.  
- **Handling Expiration**: If a key expires mid-run, the framework or concurrency manager should detect and prompt retrieval of a new token (if feasible).  
- **Fallback**: If the key is invalid or missing, the augmenter can raise an exception that indicates the credential issue.

---

## 8. Concurrency & HPC Considerations

### 8.1 Shared Access

When the concurrency manager spawns multiple threads/processes:

- The same environment variables (keys) can be shared.  
- If each worker reads from a secrets manager, ensure **rate limiting** or caching to avoid repeated secret retrieval.

### 8.2 HPC Access

- HPC credentials must be set in a way HPC jobs can run. Possibly pass ephemeral tokens to HPC job submission commands.  
- **Do not** store HPC user credentials in plain text in job scripts or logs. Use short-lived tokens or environment variables injected at job submission time.

---

## 9. Example Config Patterns

### 9.1 Docker & K8s

- **Docker**: Pass environment variables to your container (`-e PUBMED_API_KEY=...`).  
- **Kubernetes**: Use **Secrets** objects, then mount them as environment variables or files in the container.  

### 9.2 Local Development

- **`.env` Files**: A developer might keep a local `.env` with credentials. Ensure `.gitignore` excludes these files.  
- **Vault** or **Parameter Store**: For advanced local dev, you can still retrieve secrets from a centralized system.

---

## 10. Compliance & Policy

Depending on your organization’s policy or industry regulations:

- Enforce **strong policies** on secret handling (no secrets in code, mandatory rotation, etc.).  
- Conduct **periodic scans** of repositories to ensure no secrets are accidentally committed.

---

## 11. Summary & Recommendations

1. **Use environment variables** or a **secrets manager**—**never** hardcode credentials in code.  
2. **Limit** secret exposure in logs, messages, or stack traces.  
3. **Protect** HPC credentials and ensure HPC job scripts do not contain plaintext secrets.  
4. **Rotate** keys regularly and handle expiration gracefully.  
5. Always use **TLS/HTTPS** for external calls.

By adhering to these **best practices**, the **data-augmentation-framework** maintains **secure** communication with external data sources and HPC systems while protecting sensitive credentials from accidental disclosure or unauthorized access.

---

**Document History**  
- **v1.0** – Initial guidelines for external API and HPC authentication security.  
- **v1.1** – Added HPC job submission considerations and environment variable usage details.

_End of Document_