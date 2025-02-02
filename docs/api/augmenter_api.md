Below is a sample **API/Class Reference** document for the **data-augmentation-framework**, specifically focusing on **augmenters** like `GeneAugmenter`, `PubMedAugmenter`, or other domain-specific augmentation classes. Adapt the specifics (class names, method signatures, parameters) to match your actual codebase and requirements.

---

# Data Augmentation Framework  
**Augmenters API / Class Reference**  
`docs/api/augmenters_api.md`

## 1. Introduction

This document provides **class-level reference** for the **augmenters** in the **Data Augmentation Framework**. Each augmenter is designed to **retrieve**, **transform**, or **enrich** data from external sources (e.g., PubMed, finance APIs, HPC pipelines). By implementing a **common interface** or inheriting from a shared **`BaseAugmenter`** class, these classes maintain consistent patterns for **logging**, **context integration**, and **error handling**.

---

## 2. Common Base Class: `BaseAugmenter`

```python
# base_augmenter.py
class BaseAugmenter:
    """
    Abstract base class for all augmenters in the Data Augmentation Framework.

    Responsibilities:
    - Provide a standard method signature for augmenting data.
    - Handle context logging integration if needed (or allow child classes to do so).
    - Optionally provide shared error retry logic, caching hooks, etc.
    """

    def __init__(self, enable_cache: bool = True, max_retries: int = 3, **kwargs):
        """
        Initializes the BaseAugmenter with default or user-specified configurations.

        :param enable_cache: Whether to use the framework's caching layer (default = True)
        :param max_retries: Maximum number of retries for transient errors (default = 3)
        :param kwargs: Additional optional parameters for subclass augmenters
        """
        self.enable_cache = enable_cache
        self.max_retries = max_retries
        # Additional config or placeholders for child classes
        self.config = kwargs

    def augment_data(self, data, **kwargs):
        """
        Abstract method to be overridden by child classes.

        :param data: The input dataset (usually a DataFrame or ContextAwareDataStructure).
        :param kwargs: Additional parameters needed for augmentation.
        :return: Augmented dataset with context logs (if applicable).
        """
        raise NotImplementedError("Subclasses must implement `augment_data`.")
```

### Key Points
- **Inheritance**: All concrete augmenters (e.g., `GeneAugmenter`, `PubMedAugmenter`) should inherit from this class.  
- **Common Parameters**:
  - `enable_cache` controls whether repeated lookups (e.g., same gene ID) can be cached.  
  - `max_retries` used in conjunction with the framework’s [error & retry policy](../design/error_retry_policy.md).  
- **`augment_data`** must be **overridden** by subclasses to implement the specific logic of the augmentation.

---

## 3. GeneAugmenter

```python
class GeneAugmenter(BaseAugmenter):
    """
    Augments a dataset by normalizing or enriching gene identifiers
    with metadata (e.g., official gene symbols, descriptions).
    Inherits from `BaseAugmenter`.
    """

    def __init__(
        self,
        source_db: str = "NCBI",
        gene_column: str = "gene_id",
        **kwargs
    ):
        """
        :param source_db: Data source (e.g. "NCBI", "Ensembl", "UCSC") to fetch gene info.
        :param gene_column: Column name in the input data containing gene identifiers.
        :param kwargs: Additional arguments (passed to BaseAugmenter).
        """
        super().__init__(**kwargs)
        self.source_db = source_db
        self.gene_column = gene_column

    def augment_data(self, data, **kwargs):
        """
        Normalizes gene IDs in the specified column, fetching additional metadata
        from the configured source_db.

        :param data: The input data, typically a pandas DataFrame or a
                     context-aware structure with an attached adapter.
        :param kwargs: Extra parameters (e.g., concurrency settings).
        :return: The augmented dataset, including context logs with gene info.
        """
        # 1. Ensure data is context-aware (wrap if needed)
        adapter = ensure_context_adapter(data)

        # 2. Extract gene IDs and process in chunks or singly
        gene_ids = data[self.gene_column].unique()
        for gene_id in gene_ids:
            # Optionally use caching or concurrency as needed
            gene_info = self._fetch_gene_info(gene_id)  # calls external DB or local references
            # 3. Insert gene_info into data rows
            data.loc[data[self.gene_column] == gene_id, "gene_symbol"] = gene_info.get("symbol")
            data.loc[data[self.gene_column] == gene_id, "gene_description"] = gene_info.get("description")

            # 4. Log context
            adapter.add_context(
                item_id=gene_id,
                source="GeneAugmenter",
                details={
                    "source_db": self.source_db,
                    "normalized_symbol": gene_info.get("symbol"),
                }
            )

        return data

    def _fetch_gene_info(self, gene_id):
        """
        Internal helper to retrieve gene metadata from an external source or local cache.

        :param gene_id: The gene identifier to look up.
        :return: A dictionary containing gene information (symbol, description, etc.)
        """
        # Placeholder logic:
        if self.enable_cache:
            # check cache first
            pass

        # Example pseudo-call to external DB
        # gene_data = requests.get(f"https://api.ncbi.nlm.nih.gov/gene/{gene_id}").json()
        gene_data = {
            "symbol": f"Symbol_of_{gene_id}",
            "description": f"Description_of_{gene_id}"
        }
        return gene_data
```

### Usage Example

```python
from data_augmentation_framework.augmenters_api import GeneAugmenter

augmenter = GeneAugmenter(source_db="NCBI", gene_column="gene_id", enable_cache=True)
augmented_df = augmenter.augment_data(df)
```

---

## 4. PubMedAugmenter

```python
class PubMedAugmenter(BaseAugmenter):
    """
    Augments the dataset by retrieving PubMed references (PMIDs, abstracts)
    for specified keywords or gene symbols. Inherits from BaseAugmenter.
    """

    def __init__(self, query_column: str = "gene_symbol", **kwargs):
        """
        :param query_column: Column name in the input data used to form PubMed queries.
        :param kwargs: Additional parameters (passed to BaseAugmenter).
        """
        super().__init__(**kwargs)
        self.query_column = query_column

    def augment_data(self, data, **kwargs):
        """
        Fetches PubMed PMIDs and optionally abstracts for each value in `query_column`.
        Logs results in the context store.
        """
        adapter = ensure_context_adapter(data)
        unique_queries = data[self.query_column].unique()

        for q in unique_queries:
            pmid_list = self._search_pubmed(q)   # eSearch
            abstracts = self._fetch_abstracts(pmid_list)  # eFetch
            # Update data with some snippet or metadata
            data.loc[data[self.query_column] == q, "pmids"] = [pmid_list]*sum(data[self.query_column] == q)
            data.loc[data[self.query_column] == q, "abstract_snippets"] = [abstracts]*sum(data[self.query_column] == q)

            # Log context
            adapter.add_context(
                item_id=q,
                source="PubMedAugmenter",
                details={
                    "pmids": pmid_list,
                    "abstract_snippets": [a[:100] for a in abstracts],  # store partial snippet
                }
            )

        return data

    def _search_pubmed(self, query: str):
        """
        Calls PubMed eSearch for a given query, returns a list of PMIDs.
        Can include caching or retry logic for transient errors.
        """
        # Example pseudo-code
        pmids = []
        # result = requests.get("https://eutils.ncbi.nlm.nih.gov/...&term=" + query).json()
        # parse result and append to pmids
        pmids = [12345, 67890]  # placeholder
        return pmids

    def _fetch_abstracts(self, pmids):
        """
        Calls PubMed eFetch to retrieve abstracts for the given PMIDs.
        Returns a list of abstract texts.
        """
        # result = requests.post("https://eutils.ncbi.nlm.nih.gov/...efetch", data={"id": ",".join(pmids)}).json()
        abstracts = ["Abstract text for 12345", "Abstract text for 67890"]
        return abstracts
```

### Usage Example

```python
from data_augmentation_framework.augmenters_api import PubMedAugmenter

pubmed_augmenter = PubMedAugmenter(query_column="gene_symbol", enable_cache=True)
augmented_df = pubmed_augmenter.augment_data(df)
```

---

## 5. Example: FinanceAugmenter

```python
class FinanceAugmenter(BaseAugmenter):
    """
    Fetches stock or financial data for a specified ticker column,
    integrating daily prices or fundamental metrics into the dataset.
    """

    def __init__(self, ticker_column: str = "ticker", api_source: str = "AlphaVantage", **kwargs):
        """
        :param ticker_column: The column in `data` containing stock tickers.
        :param api_source: Which finance API to use (AlphaVantage, Yahoo, etc.)
        :param kwargs: Additional config (e.g., concurrency, caching).
        """
        super().__init__(**kwargs)
        self.ticker_column = ticker_column
        self.api_source = api_source

    def augment_data(self, data, **kwargs):
        adapter = ensure_context_adapter(data)
        tickers = data[self.ticker_column].unique()

        for t in tickers:
            finance_data = self._fetch_finance_data(t)
            # Insert the retrieved data into relevant rows
            data.loc[data[self.ticker_column] == t, "latest_price"] = finance_data.get("latest_price")
            data.loc[data[self.ticker_column] == t, "market_cap"] = finance_data.get("market_cap")

            # Log context
            adapter.add_context(
                item_id=t,
                source="FinanceAugmenter",
                details={
                    "api_source": self.api_source,
                    "latest_price": finance_data.get("latest_price"),
                    "market_cap": finance_data.get("market_cap"),
                }
            )

        return data

    def _fetch_finance_data(self, ticker):
        """
        Calls the finance API to get data for the specified ticker.
        Returns a dict with relevant fields (price, market cap, etc.).
        """
        # Example pseudo-call:
        # response = requests.get(f"https://api.alpha_vantage.co/query?function=GLOBAL_QUOTE&symbol={ticker}&apikey=XYZ").json()
        # parse response
        finance_data = {
            "latest_price": 150.00,
            "market_cap": 2000000000
        }
        return finance_data
```

### Usage Example

```python
from data_augmentation_framework.augmenters_api import FinanceAugmenter

finance_aug = FinanceAugmenter(ticker_column="Ticker", api_source="AlphaVantage", enable_cache=True)
augmented_df = finance_aug.augment_data(df)
```

---

## 6. HPC or Advanced Transform Augmenters

```python
class HPCAugmenter(BaseAugmenter):
    """
    Offloads large-scale transformations to an HPC cluster or GPU nodes.
    After the job completes, it merges the results back into the dataset.
    """

    def __init__(self, job_name: str = "HPC_Job", **kwargs):
        """
        :param job_name: Identifier for the HPC job submission.
        :param kwargs: Additional HPC parameters (e.g., queue, resource requests).
        """
        super().__init__(**kwargs)
        self.job_name = job_name

    def augment_data(self, data, **kwargs):
        adapter = ensure_context_adapter(data)
        # 1. Submit job to HPC
        job_id = self._submit_hpc_job(data)

        # 2. Wait for job to complete (or poll)
        result_path = self._wait_for_job(job_id)

        # 3. Load HPC output and integrate with data
        data = self._merge_hpc_results(data, result_path)

        # 4. Log context
        adapter.add_context(
            item_id="global",
            source="HPCAugmenter",
            details={
                "job_name": self.job_name,
                "job_id": job_id,
                "result_path": result_path
            }
        )

        return data

    def _submit_hpc_job(self, data):
        """
        Submits the data to the HPC cluster (e.g., Slurm job submission).
        Returns the HPC job ID.
        """
        # Pseudocode for HPC submission
        job_id = "123456"
        return job_id

    def _wait_for_job(self, job_id):
        """
        Polls the HPC queue until the job finishes, then returns the path to results.
        """
        # Pseudocode
        result_path = "/path/to/hpc/results"
        return result_path

    def _merge_hpc_results(self, data, result_path):
        """
        Reads HPC output from result_path and merges it back into data.
        """
        # e.g., load results from a file, integrate into data columns
        return data
```

### Usage Example

```python
hpc_augmenter = HPCAugmenter(job_name="GeneTransformJob", enable_cache=False)
augmented_df = hpc_augmenter.augment_data(df)
```

---

## 7. Utility Function: `ensure_context_adapter`

Throughout the examples, we reference a helper like:

```python
def ensure_context_adapter(data):
    """
    If 'data' is not already context-aware, wrap it with the appropriate adapter.
    Returns an adapter object for logging context.
    """
    # Example logic:
    if not hasattr(data, "_context_adapter"):
        from context_framework.adapters import PandasContextAdapter
        data._context_adapter = PandasContextAdapter(data)
    return data._context_adapter
```

**Purpose**:  
- Ensures that the dataset can log transformations via the **Context-Tracking Framework**.  
- Usually resides in a shared utility module or is part of the orchestrator logic.

---

## 8. Additional Notes & Best Practices

1. **Caching**  
   - Each augmenter can leverage the framework’s caching system to reduce repeated calls for identical queries.  
   - Typically controlled via `enable_cache` or additional parameters in the constructor.  

2. **Error Handling / Retry**  
   - All external calls (PubMed, finance APIs, HPC submissions) should incorporate the standard [error & retry policy](../design/error_retry_policy.md) to handle transient or rate-limit errors gracefully.  

3. **Context Logging**  
   - Each augmenter should log **who/what/when** in the context store for transparency. This ensures that downstream processes or audits can see the provenance of each transformation.  

4. **Chunking / Concurrency**  
   - When dealing with large data sets, the orchestrator or concurrency manager may chunk the data and invoke these augmenters in parallel.  
   - Each augmenter is typically **chunk-agnostic**, meaning it processes whatever subset is provided.  

5. **Testing**  
   - Provide **unit tests** for each augmenter, mocking external API calls.  
   - Provide **integration tests** with real or sandbox API endpoints (when possible) to validate end-to-end workflows.

---

## 9. Summary

The **augmenters** in the Data Augmentation Framework are specialized classes that **add context** or **transform** data by referencing external sources (APIs, HPC jobs, domain-specific databases). By extending the common **`BaseAugmenter`** and adhering to standard practices (logging context, handling errors, optional caching), developers can create consistent, powerful enrichment tools for a wide array of use cases.

**Main Classes**:

- **`BaseAugmenter`**: Abstract parent for consistent structure.  
- **`GeneAugmenter`**: Normalizes gene IDs, fetches official symbols/descriptions.  
- **`PubMedAugmenter`**: Retrieves PMIDs/abstracts based on gene symbols or keywords.  
- **`FinanceAugmenter`**: Integrates financial data (e.g., stock prices, fundamentals).  
- **`HPCAugmenter`**: Offloads large-scale transformations to an HPC cluster.

Each **augmenter** ensures **traceable**, **auditable** transformations through the **Context-Tracking Framework** and supports **retry** or **caching** according to the user’s configuration.

---

**Document History**  
- **v1.0** – Initial compilation of commonly used augmenters and their APIs.  
- **v1.1** – Added HPC example and context utility references.  

_End of Document_