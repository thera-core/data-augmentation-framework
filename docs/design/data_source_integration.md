# Data Augmentation Framework  
**Data Source Integration Specs**  
`docs/design/data_source_integration.md`

## 1. Introduction

This document outlines how various external **data sources** can be integrated into the **Data Augmentation Framework**. It covers typical integration patterns, authentication, data retrieval flows, caching strategies, and best practices for maintaining a robust and scalable augmentation pipeline.

### 1.1 Purpose

- Provide **technical guidelines** for implementing new augmenters that fetch or transform data from external sources.  
- Document **existing integrations** (e.g., PubMed, Finance APIs) and **common patterns** for new integrations.  
- Ensure **consistent** logging, error handling, and context tracking across all data sources.

### 1.2 Scope

- **In-Scope**: PubMed, Finance APIs, potential HPC-based data retrieval.  
- **Out-of-Scope**: Detailed HPC concurrency design (see [`architecture.md`](architecture.md) and concurrency modules), as well as internal data transformations not involving external services.

---

## 2. General Integration Patterns

Although each data source has unique endpoints and data formats, most integrations follow a similar **four-step** pattern:

1. **Authentication** or Setup  
   - Retrieve API keys or tokens (if needed).  
   - Handle environment configuration (URLs, request headers, etc.).
2. **Data Request**  
   - Query external endpoints, possibly in **batch** or **chunked** mode for large data sets.  
   - Use caching to avoid repeated requests.
3. **Data Parsing & Transformation**  
   - Convert retrieved data into a format suitable for the augmentation pipeline.  
   - Optionally handle partial or streaming responses.
4. **Context Logging**  
   - Invoke the **Context-Tracking Framework** to record provenance (e.g., “Data fetched from PubMed query: … at time …”).

### 2.1 Recommended File/Module Structure

```
data-augmentation-framework/
└─ augmenters/
   ├─ pubmed_augmenter.py
   ├─ finance_augmenter.py
   ├─ ...
   └─ base_augmenter.py
```

- **`base_augmenter.py`**: Shared logic, error handling wrappers, context logging utilities.  
- **`pubmed_augmenter.py`**: Integration with PubMed.  
- **`finance_augmenter.py`**: Integration with one or more finance data providers.  
- Additional data source integrations can follow a similar naming convention.

---

## 3. PubMed Integration

The **PubMedAugmenter** (or similarly named class) provides an interface to the [NCBI E-utilities](https://www.ncbi.nlm.nih.gov/books/NBK25500/) or alternative PubMed APIs.

### 3.1 Use Cases

- **Augmenting gene-based datasets** with related literature.  
- **Adding references** or abstracts to rows containing PubMed IDs or biological keywords.

### 3.2 Key Endpoints & Flows

1. **E-Search**  
   - URL: `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi`  
   - Purpose: Search for PubMed IDs given keywords or gene symbols.  
   - Typical Flow:  
     1. Construct query string (e.g., “BRCA1[Gene] AND human[Organism]”).  
     2. Retrieve a list of **PMIDs** in JSON or XML.  
     3. Handle pagination for large result sets.
2. **E-Fetch**  
   - URL: `https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi`  
   - Purpose: Retrieve full abstracts or metadata given a list of PMIDs.  
   - Typical Flow:  
     1. Pass the PMIDs to eFetch.  
     2. Parse the returned abstracts, titles, MeSH terms, etc.  
     3. Map the data back to your internal structure (columns, cells, or context metadata).

### 3.3 Authentication & Rate Limits

- **API Keys**:  
  - NCBI provides an API key mechanism for increased rate limits.  
  - Store keys in environment variables or secure config (no hard-coding).  
- **Rate Limits**:  
  - Standard usage typically allows several queries per second.  
  - If surpassing free-tier limits, implement request throttling or caching.

### 3.4 Caching Strategy

- **Short-Term Caching**:  
  - Cache search results for specific gene queries (e.g., “BRCA1[Gene]” → list of PMIDs).  
  - TTL-based (e.g., 24 hours) to avoid stale references.  
- **Abstract Caching**:  
  - Full abstracts may be cached to reduce repeated calls, especially for recurring queries.

### 3.5 Data Parsing & Context Logging

- **Parsing**:  
  - Convert XML/JSON to a structured format (title, authors, abstract text).  
- **Context Logging**:  
  - For each row or gene ID, store:  
    - PMID(s) retrieved.  
    - Timestamp of fetch.  
    - API endpoint or query used.  
  - Example snippet:
    ```python
    adapter.add_context(
        item_id=row_index,
        source="PubMedAugmenter",
        details={
            "pmid": pmid,
            "query": "BRCA1[Gene]",
            "abstract_snippet": abstract_text[:200]
        }
    )
    ```

---

## 4. Finance API Integration

While PubMed focuses on biomedical data, the **FinanceAugmenter** might integrate with one or more finance APIs such as **Yahoo Finance**, **Alpha Vantage**, or **Finnhub**.

### 4.1 Use Cases

- **Augmenting stock tickers**: For datasets that contain financial symbols (e.g., AAPL, TSLA, MSFT).  
- **Historical Pricing**: Merging daily or intraday price data into the context structure.  
- **Fundamental Data**: Retrieving P/E ratios, market cap, or other key metrics.

### 4.2 Example: Alpha Vantage Integration

1. **Endpoint**: `https://www.alphavantage.co/query?function=TIME_SERIES_DAILY&symbol=XYZ&apikey=...`  
2. **Response Handling**:  
   - Parse JSON time-series data.  
   - For each date, store open/high/low/close, volume, etc.  
3. **Batch Processing**:  
   - If the dataset has multiple symbols (100+), chunk them to avoid overloading the API or hitting rate limits.  
4. **Caching**:  
   - Caching at symbol + date range level can drastically reduce repeated queries.  
   - E.g., `("AAPL", "2021-01-01 to 2021-12-31") -> [prices]`.

### 4.3 Authentication & Rate Limits

- **API Key**:  
  - Typically one key per user or per project.  
  - Must be kept secure in environment variables/config.  
- **Rate Limiting**:  
  - Commonly ~5 calls per minute for free tiers; paid tiers may allow more.  
  - Throttle or queue requests in concurrency manager.

### 4.4 Context Logging

- For each ticker or row:  
  - Record the API used, date range requested, any errors or missing data.  
  - Example snippet:
    ```python
    adapter.add_context(
        item_id=row_index,
        source="FinanceAugmenter",
        details={
            "symbol": "AAPL",
            "date_range": "2021-01-01 to 2021-12-31",
            "data_points": len(prices_returned)
        }
    )
    ```

---

## 5. Other External Sources & HPC Transforms

The framework should be flexible to include additional data sources or HPC-based transformations. Some potential areas:

### 5.1 Genomic Databases

- **Ensembl**, **NCBI Gene**, **UCSC Genome Browser**.  
- Often used for advanced gene or variant annotation tasks.

### 5.2 HPC Job-Generated Data

- **Flow**:  
  1. Submit a job to an HPC cluster (e.g., Slurm) that produces an output file or structured data.  
  2. Once the job completes, read the output into the augmentation pipeline.  
- **Context Logging**:  
  - Record job ID, HPC node used, CPU/GPU hours, etc.  
  - Potentially store HPC logs for debugging.

### 5.3 Offline or Proprietary Datasets

- **Local Databases** or **internal data warehouses** might require custom connectors.  
- Must follow similar caching, logging, and concurrency best practices.

---

## 6. Error Handling & Retry Logic

1. **Network/Connection Errors**  
   - Implement automatic **retry with backoff** (e.g., exponential backoff) for transient issues.  
   - If consistently failing, log the error in context and skip or fail gracefully.
2. **Invalid or Missing Data**  
   - Some queries (symbols, gene IDs) might return 404 or empty results.  
   - Mark them in context as “not found” without aborting the entire process.
3. **API Changes**  
   - APIs can evolve (new response formats, deprecated endpoints).  
   - Keep each augmenter’s integration logic decoupled so updates can be applied without affecting others.

---

## 7. Security Considerations

1. **Credentials**  
   - Store all API keys/tokens in environment variables or a secure vault system.  
   - Never commit credentials to source control.
2. **Request Encryption**  
   - Use HTTPS endpoints only.  
   - Validate SSL certificates where possible.
3. **Access Control**  
   - If the augmentation framework runs in a multi-tenant environment, ensure each user or agent has the correct permissions for external calls.

---

## 8. Best Practices & Recommendations

1. **Use Shared Utilities**  
   - Common code for HTTP requests, JSON/XML parsing, caching logic should reside in a shared module or `base_augmenter.py`.
2. **Enable Caching**  
   - Especially critical for repeated queries to the same endpoints.  
   - Consider time-based (TTL) or size-based (LRU) caches.
3. **Follow RESTful/HTTP Conventions**  
   - Use clear GET/POST requests, handle query parameters, handle pagination.  
   - Keep external requests idempotent where possible.
4. **Log Verbosely**  
   - Each query or HPC job submission should have an entry in the logs.  
   - This improves observability and troubleshooting.
5. **Graceful Degradation**  
   - If a data source is offline or rate-limited, the augmenter should skip or provide partial results without crashing the entire pipeline.

---

## 9. Example Integration Workflow

**Scenario**: A user in TheraLab requests to augment a list of gene symbols with both PubMed abstracts and daily stock prices for each gene’s “related company.”  

1. **User**: Issues command “/augment gene data with PubMed and finance info.”  
2. **TheraLab**: Calls `data_augmentation_framework.orchestrator.augment_data()` with two augmenters: `PubMedAugmenter` and `FinanceAugmenter`.  
3. **PubMedAugmenter**:  
   - For each gene, constructs a search query.  
   - Fetches PMIDs, eFetches abstracts.  
   - Logs details (PMIDs, query) to context.  
4. **FinanceAugmenter**:  
   - For each gene → find related ticker symbol (optional lookup).  
   - Pulls daily close prices from a finance API.  
   - Logs retrieval date range, symbol, response length.  
5. **Orchestrator**:  
   - Merges augmented data.  
   - Returns final context-aware DataFrame to TheraLab.  
6. **TheraLab**:  
   - Presents summarized results, or saves them to a digital journal.  

---

## 10. Future Directions

1. **Plugin System**  
   - Auto-discover augmenters for new data sources without modifying the core code.  
2. **Advanced HPC Integration**  
   - Incorporate Slurm or Dask for fully distributed retrieval or large-scale transformations.  
3. **Unified Configuration**  
   - Provide a single configuration file (e.g., `datasources.yaml`) to store all credentials, endpoints, and rate limit settings.  
4. **Schema Validation**  
   - Rigorously define expected input/output fields for each augmenter, possibly using JSON schemas.

---

## 11. Conclusion

Integrating external data sources in the **Data Augmentation Framework** hinges on **clear, modular** patterns for authentication, retrieval, caching, and context logging. By adhering to the guidelines above, developers can **quickly add** new data sources (PubMed, Finance APIs, HPC outputs, etc.) while maintaining **robustness**, **scalability**, and **traceability**.

For more detailed technical references, see:

- [Architecture Overview](architecture.md)  
- [Functional & Nonfunctional Requirements](../requirements/functional_requirements.md)  
- [Context-Tracking Framework Docs](../../context-framework/...)  

---

**Document History**  
- **v1.0** – Initial publication covering PubMed and basic finance APIs.  
- **v1.1** – Expanded HPC references and added best practices for plugin architecture.

_End of Document_