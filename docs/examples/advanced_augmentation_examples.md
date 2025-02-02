## 2. `examples/advanced_augmentation_examples.md`

# Advanced Augmentation Examples

This document illustrates more **complex usage** scenarios for the **data-augmentation-framework**, including:

- **Concurrent (chunked) augmentation** for large datasets.  
- **HPC-based transformations**.  
- **Multiple augmenters** in a single workflow.  
- **Inspecting and merging context** across chunks.

---

## 1. Chunked Concurrency Example

### 1.1 Why Chunked Concurrency?

When dealing with **very large** DataFrames (millions of rows), calling an API or HPC transformation on every row sequentially can be slow. Instead, you can **split** data into chunks and run each chunk in parallel.

### 1.2 Sample Code

```python
import pandas as pd
from data_augmentation_framework.orchestrator import augment_data
from data_augmentation_framework.augmenters_api import PubMedAugmenter

# Assume we have a large dataset
data = pd.DataFrame({
    "gene_symbol": ["BRCA1"]*500000 + ["TP53"]*500000,  # 1 million rows
    "expression_value": [1.2]*500000 + [0.8]*500000
})

print("Data size:", len(data))

# Concurrency options:
chunk_size = 100000   # 100k rows per chunk
num_workers = 4       # 4 parallel processes or threads

# We call a high-level orchestrator function that handles chunking
augmented_data = augment_data(
    data=data,
    augmenter=PubMedAugmenter,
    augmenter_params={"query_column": "gene_symbol"},
    chunk_size=chunk_size,
    concurrency=num_workers
)

print("Augmentation complete. Checking context logs...")

adapter = getattr(augmented_data, "_context_adapter", None)
if adapter:
    # e.g., see how many context entries we have for BRCA1
    brca1_context_entries = adapter.get_context("BRCA1")
    print(f"BRCA1 has {len(brca1_context_entries)} context entries recorded.")
```

#### Explanation
1. **`augment_data`** is a convenience function (in some designs) that handles concurrency. It splits the DataFrame into 10 chunks (if 1,000,000 rows with chunk_size=100,000).  
2. Each chunk is fed to **PubMedAugmenter** in parallel.  
3. The orchestrator merges chunks and returns a single **context-aware** DataFrame.

---

## 2. HPC-Based Transformation Example

### 2.1 Scenario

You have a custom **HPCAugmenter** that offloads computations (e.g., GPU-based transformations) to an HPC cluster. Below is a simplified usage pattern.

```python
import pandas as pd
from data_augmentation_framework.augmenters_api import HPCAugmenter

# HPC-specific config
data = pd.DataFrame({
    "sample_id": [f"SAMPLE_{i}" for i in range(1000)],
    "raw_measurements": [i*0.1 for i in range(1000)]
})

augmenter = HPCAugmenter(
    job_name="HPC_ExpressionTransform",
    enable_cache=False,    # HPC job might not be beneficial to cache
    max_retries=2          # Retry HPC job submission if there's a transient error
)

augmented_data = augmenter.augment_data(data)

# After HPC job completes (the augment_data method typically blocks or polls),
# the results are merged back into `augmented_data`.

adapter = getattr(augmented_data, "_context_adapter", None)
if adapter:
    # HPC transformations typically log job IDs, HPC node usage, etc.
    job_context = adapter.get_context("global")
    print("HPC Job Context:", job_context)
```

#### Explanation
- **HPCAugmenter** might:
  1. Submit the data to a cluster job.  
  2. Wait for completion or poll the job queue.  
  3. Load the job’s output file(s) and merge them into your DataFrame.  
  4. Log HPC details (job ID, resources used) into the context store.

---

## 3. Multiple Augmenters in One Workflow

You can chain multiple augmenters on the **same** dataset, each logging separate context entries.

```python
import pandas as pd
from data_augmentation_framework.augmenters_api import (
    GeneAugmenter,
    PubMedAugmenter
)

df = pd.DataFrame({
    "gene_id": ["ENSG00000139618", "ENSG00000141510", "ENSG00000279457"],
    "expression_value": [2.3, 1.8, 3.1]
})

# 1. Normalize gene IDs
gene_augmenter = GeneAugmenter(gene_column="gene_id", source_db="Ensembl")
df_aug1 = gene_augmenter.augment_data(df)

# 2. Now query PubMed using the newly added "gene_symbol" column
pubmed_augmenter = PubMedAugmenter(query_column="gene_symbol")
df_aug2 = pubmed_augmenter.augment_data(df_aug1)

# Inspect the final DataFrame
print(df_aug2)
adapter = getattr(df_aug2, "_context_adapter", None)
if adapter:
    # Show all context for a specific row or symbol
    symbol_context = adapter.get_context("BRCA1")  # if GeneAugmenter set that
    print("Context logs for BRCA1:", symbol_context)
```

#### Explanation
1. **GeneAugmenter** converts raw gene IDs to official gene symbols (e.g., “BRCA1”) and logs each transformation.  
2. **PubMedAugmenter** then uses that symbol to fetch references from PubMed.  
3. Each augmenter writes separate entries to the context store, preserving the chain of transformations.

---

## 4. Inspecting and Merging Context Across Chunks

When using concurrency, each **chunk** might produce context entries. If you want to see all context together, typically the orchestrator merges them. In some advanced cases, you might manually retrieve context from each chunk and combine them.

**Pseudocode**:

```python
chunks = chunkify(data, chunk_size=50000)
chunk_results = []
for chunk in chunks:
    chunk_adapter = ensure_context_adapter(chunk)
    result_chunk = someAugmenter.augment_data(chunk)
    chunk_results.append(result_chunk)

# Merge data
final_df = pd.concat(chunk_results, ignore_index=True)

# Merge context (if each chunk has its own adapter/store, unify them)
final_adapter = unify_context_adapters([getattr(c, "_context_adapter", None) for c in chunk_results])
final_df._context_adapter = final_adapter
```

> Implementation details depend on how your concurrency framework is set up and how the context-framework’s store merges multiple partial logs.

---

## 5. Tips & Best Practices

1. **Log Enough Detail**: Always record key parameters (timestamp, HPC job ID, query terms) in context. This ensures traceability.  
2. **Handle Errors Gracefully**: Use the built-in retry logic or concurrency manager to skip or retry failing chunks without losing everything.  
3. **Parallel vs. HPC**: Concurrency is often local to your environment, while HPC calls might require job submission. Either way, the context logs remain consistent.  
4. **Use Shared Caches** (If Available): Repeated lookups for the same data can be expensive. A well-designed caching strategy can significantly reduce run time for large datasets.

---

## 6. Summary

- **Chunked Concurrency**: Speeds up augmentation for large datasets.  
- **HPC**: Offloads heavy transformations, logging details back into the context store.  
- **Multiple Augmenters**: You can chain transformations, each storing its own provenance.  
- **Merging Context**: Combine logs from multiple parallel tasks into a single data structure.  

These **advanced** examples illustrate how you can tailor the **data-augmentation-framework** to handle real-world complexity—large data volumes, HPC workflows, multi-step pipelines—while maintaining thorough **context** logging for every step.

---

**Document History**  
- **v1.0** – Initial tutorial and advanced examples for concurrency and HPC.  
- **v1.1** – Added multi-augmenter chaining and tips on merging context.

_End of Documents_