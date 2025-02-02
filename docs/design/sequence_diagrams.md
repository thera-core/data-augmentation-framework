Below is an example of a **Sequence Diagrams** documentation file for the **data-augmentation-framework**. It illustrates how various components—such as augmenters, concurrency layers, and the context-tracking framework—interact during data augmentation workflows. You can adapt and expand these diagrams to match your specific use cases, naming conventions, and technologies.

---

# Data Augmentation Framework  
**Sequence Diagrams**  
`docs/design/sequence_diagrams.md`

## 1. Introduction

This document presents **sequence diagrams** that show how requests flow through the **Data Augmentation Framework**. The diagrams highlight the interactions between:

- **Clients** (e.g., TheraLab or other caller)  
- **The Orchestrator** (the high-level entry point in the augmentation framework)  
- **Augmenters** (e.g., PubMedAugmenter, FinanceAugmenter)  
- **Concurrency Manager** (handles chunking, parallelism, or HPC job scheduling)  
- **Context-Tracking Framework** (logs and retrieves context/provenance)

By visualizing these interactions, developers and stakeholders can better understand the **lifecycle** of an augmentation task, from the initial request to the final returned data and context logging.

---

## 2. Basic Augmentation Request

The following diagram shows a **simple** (non-chunked, single-threaded) augmentation request initiated by the caller, passing through the framework, and updating the context store.

```mermaid
sequenceDiagram
    participant Caller as TheraLab / Other Client
    participant Orchestrator as Orchestrator
    participant Augmenter as <Augmenter> (e.g., PubMedAugmenter)
    participant CF as Context Framework

    Caller->>Orchestrator: augment_data(data, augmenter=PubMedAugmenter)
    note right of Orchestrator: Validate input, set defaults<br/> (e.g., concurrency = 1)
    Orchestrator->>CF: Ensure data is wrapped with ContextAwareAdapter
    note right of CF: If data is not already context-aware,<br/> create or attach an adapter

    Orchestrator->>Augmenter: call augmenter with data
    Augmenter->>Augmenter: fetch external data (API, HPC job, etc.)
    Augmenter->>CF: add_context(item_id, source="Augmenter", details=...)
    note right of CF: Store transformation info <br/> (timestamp, parameters, etc.)

    Augmenter->>Orchestrator: return augmented data
    Orchestrator->>Caller: return final augmented data
```

### Explanation
1. **Caller** invokes `augment_data(...)` on the Orchestrator, specifying which augmenter to use.  
2. **Orchestrator** checks if the incoming data is already context-aware. If not, it wraps the data with an appropriate **Context Adapter** from the `context-framework`.  
3. The **Augmenter** fetches or computes any external data needed (e.g., PubMed results).  
4. The **Augmenter** logs its operations via the **Context Framework** (e.g., storing provenance).  
5. Finally, the **Orchestrator** returns the augmented (and context-enriched) data to the caller.

---

## 3. Chunked Concurrency Flow

For **large datasets**, the Data Augmentation Framework may employ **chunked** or **parallel** execution. The sequence diagram below illustrates how an orchestrator might break the data into chunks, assign them to augmenters, and then merge the results.

```mermaid
sequenceDiagram
    participant Caller as TheraLab / Other Client
    participant Orchestrator as Orchestrator
    participant ConcurrencyMgr as Concurrency Manager
    participant Augmenter as <Augmenter>
    participant CF as Context Framework

    Caller->>Orchestrator: augment_data(large_data, augmenter=PubMedAugmenter, chunk_size=10k, concurrency=4)
    note right of Orchestrator: Validate input, parse concurrency settings
    Orchestrator->>CF: Ensure data is context-aware

    Orchestrator->>ConcurrencyMgr: split data into chunks
    ConcurrencyMgr->>ConcurrencyMgr: create thread/process pool

    loop for each chunk
        ConcurrencyMgr->>Augmenter: run augmentation on chunk
        Augmenter->>Augmenter: fetch external data for chunk
        Augmenter->>CF: add_context(...) for each row/col
        Augmenter->>ConcurrencyMgr: return augmented chunk
    end

    ConcurrencyMgr->>Orchestrator: combined augmented data
    Orchestrator->>Caller: return final augmented dataset
```

### Key Points
- **Concurrency Manager**: Responsible for splitting the input data into fixed-size chunks (e.g., 10k rows each).  
- **Parallel Execution**: Each chunk runs in a separate thread or process, enabling faster augmentation.  
- **Context Logging**: Occurs at the row/column level for each chunk.  
- **Merging Results**: The concurrency manager merges the chunks back into one combined context-aware data structure.

---

## 4. PubMedAugmenter Detailed Sequence

The next diagram dives deeper into a **PubMedAugmenter** scenario, where the augmenter queries PubMed’s E-utilities and logs results. It shows the external API interaction explicitly.

```mermaid
sequenceDiagram
    participant Caller as TheraLab
    participant Orchestrator as Orchestrator
    participant Augmenter as PubMedAugmenter
    participant PubMedAPI as PubMed E-Utilities
    participant CF as Context Framework

    Caller->>Orchestrator: augment_data(data, PubMedAugmenter)
    Orchestrator->>Augmenter: call PubMedAugmenter.augment(...)
    Augmenter->>PubMedAPI: eSearch query (e.g., BRCA1[Gene])
    note right of PubMedAPI: Return list of PMIDs
    PubMedAPI-->>Augmenter: 200 OK, PMIDs = [12345, 67890, ...]
    
    Augmenter->>PubMedAPI: eFetch PMIDs
    note right of PubMedAPI: Return article abstracts, metadata
    PubMedAPI-->>Augmenter: 200 OK, articles in XML/JSON

    Augmenter->>Augmenter: parse abstracts, map to relevant rows
    Augmenter->>CF: adapter.add_context(row_id, {pmid, snippet, ...})
    Augmenter->>Orchestrator: return updated data
    Orchestrator->>Caller: final augmented data
```

### Explanation
1. **Augmenter** performs two main steps: **eSearch** to get PMIDs, then **eFetch** to get full abstracts.  
2. After receiving the raw data, the augmenter **parses** the articles and updates the relevant rows.  
3. **Context** is logged for each row or gene ID with the PMIDs and partial article info.  
4. The Orchestrator returns the enriched DataFrame (or context-aware structure) back to the caller.

---

## 5. FinanceAugmenter with Caching

This diagram focuses on a **FinanceAugmenter** retrieving stock prices from a finance API and using a **local cache** to avoid repeated requests.

```mermaid
sequenceDiagram
    participant Caller as TheraLab
    participant Orchestrator as Orchestrator
    participant FinanceAugmenter as FinanceAugmenter
    participant Cache as Cache Manager
    participant FinanceAPI as Finance API (Alpha Vantage)
    participant CF as Context Framework

    Caller->>Orchestrator: augment_data(tickers_df, FinanceAugmenter, enable_cache=True)
    Orchestrator->>FinanceAugmenter: call augment for each symbol

    alt Cache Hit
        FinanceAugmenter->>Cache: check cache("AAPL, 2021-01-01->2021-12-31")
        Cache-->>FinanceAugmenter: data found
        note right of FinanceAugmenter: skip API call
    else Cache Miss
        FinanceAugmenter->>FinanceAPI: request daily prices for "AAPL"
        FinanceAPI-->>FinanceAugmenter: returns JSON data
        FinanceAugmenter->>Cache: store data in cache
    end

    FinanceAugmenter->>CF: add_context(row_id, details={...symbol info...})
    FinanceAugmenter->>Orchestrator: return augmented chunk

    Orchestrator->>Caller: combined augmented DataFrame
```

### Notable Steps
- **Check Cache**: The augmenter first checks the local or distributed cache for existing data.  
- **API Call**: Only if the cache misses, the augmenter calls the finance API.  
- **Context Logging**: Each row or symbol is updated with the retrieved financial data’s provenance.  
- **Return**: The orchestrator merges augmented chunks and returns final data.

---

## 6. HPC-Based Transformation Example

For HPC or GPU-based transformations, the sequence might involve job submission and waiting for completion:

```mermaid
sequenceDiagram
    participant Caller as TheraLab
    participant Orchestrator as Orchestrator
    participant HPCAugmenter as HPCAugmenter
    participant HPCMgr as HPC Manager
    participant HPCCluster as HPC Cluster
    participant CF as Context Framework

    Caller->>Orchestrator: augment_data(data, HPCAugmenter)
    Orchestrator->>HPCAugmenter: HPCAugmenter.augment(data)
    HPCAugmenter->>HPCMgr: submit job(s) for HPC transformation
    note right of HPCMgr: e.g., Slurm job submission

    HPCMgr->>HPCCluster: run HPC job
    note right of HPCCluster: HPC node(s) process data
    HPCCluster-->>HPCMgr: job complete, results saved
    HPCMgr-->>HPCAugmenter: path to output or processed data

    HPCAugmenter->>HPCAugmenter: load HPC output, integrate with dataset
    HPCAugmenter->>CF: add_context for HPC job info (job_id, node, resources)
    HPCAugmenter->>Orchestrator: return augmented data
    Orchestrator->>Caller: final HPC-processed dataset
```

### Commentary
- **HPCAugmenter** delegates intensive tasks to an HPC cluster, possibly for GPU-accelerated transformations.  
- **HPCMgr** handles job submission, monitoring, and retrieving results.  
- **Context Logging** includes HPC job IDs, resource usage, etc.

---

## 7. Error Handling & Retry Sequence

Below is a simplified sequence for how the Data Augmentation Framework handles transient errors or rate limits when calling an external API.

```mermaid
sequenceDiagram
    participant Caller as TheraLab
    participant Orchestrator as Orchestrator
    participant Augmenter as SomeAugmenter
    participant ExternalAPI as External Data Source
    participant CF as Context Framework

    Caller->>Orchestrator: augment_data(data, SomeAugmenter)
    Orchestrator->>Augmenter: call SomeAugmenter.augment(...)

    Augmenter->>ExternalAPI: fetch/transform request
    ExternalAPI-->>Augmenter: HTTP 429 (Rate Limit Exceeded)
    note right of Augmenter: detect rate limit<br/> start backoff

    Augmenter->>Augmenter: wait for backoff interval
    Augmenter->>ExternalAPI: retry request
    ExternalAPI-->>Augmenter: 200 OK, data returned

    Augmenter->>CF: add_context with transformation details
    Augmenter->>Orchestrator: return augmented data
    Orchestrator->>Caller: success
```

### Notes
- The **Augmenter** includes logic for **retry with exponential backoff** or a similar strategy.  
- Each attempt or failure is **logged** (including the final success or failure) in the context store or general logs.

---

## 8. Summary & Conclusion

These **sequence diagrams** illustrate how requests progress through the **Data Augmentation Framework**, covering:

1. **Simple augmentations**  
2. **Chunked concurrency**  
3. **PubMed** and **Finance** integrations  
4. **HPC** job-based transformations  
5. **Error and retry flows**  

By standardizing these flows, developers can maintain **consistent patterns** for context logging, error handling, concurrency, and caching across all augmenters. When adding **new data sources** or **advanced HPC** logic, use the diagrams above as a reference to ensure **clear, traceable** interactions and robust system behavior.

---

**Document History**  
- **v1.0** – Initial set of sequence diagrams for major use cases.  
- **v1.1** – Added HPC transformation example and error handling sequence.

_End of Document_