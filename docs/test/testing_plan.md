# Data Augmentation Framework  
**Testing Plan**  
`test/testing_plan.md`

## 1. Introduction

This document outlines the **testing strategy** for the **data-augmentation-framework**. The framework includes various augmenters that fetch, transform, and enrich data from external sources (e.g., PubMed, finance APIs, HPC clusters), and integrates with the **context-framework** for provenance tracking. Given the complexity of external dependencies and concurrency features, our testing plan emphasizes **mocking**, **integration testing**, and **end-to-end** validation.

---

## 2. Scope & Objectives

- **Scope**:  
  - All modules within the data-augmentation-framework:  
    - **Augmenters** (PubMed, Gene, Finance, HPC, etc.)  
    - **Concurrency** (chunked parallel processing)  
    - **Context Integration** (interfacing with the context-framework)  
    - **Caching** & **Error/Retry** logic  
- **Objectives**:  
  - Validate each augmenter’s **business logic** (e.g., correct data transformation, correct calls to context).  
  - Ensure external calls are **mocked** in unit tests to avoid dependency on live services.  
  - Verify concurrency and HPC features function correctly in **integration** or specialized tests.  
  - Provide **end-to-end** scenarios that replicate real user workflows.

---

## 3. Test Categories & Strategies

We divide our tests into several **categories**:

1. **Unit Tests**  
   - Validate individual classes or functions in isolation.  
   - Typically mock external dependencies.  
2. **Integration Tests**  
   - Test interactions between augmenters, concurrency layers, or context-framework.  
   - May use local or sandbox versions of external APIs or HPC simulators if feasible.  
3. **End-to-End (E2E) Tests**  
   - Simulate real usage (e.g., orchestrator calling multiple augmenters on a dataset).  
   - Often span from “upload data” to “final augmented data with context.”  
4. **Performance / Concurrency Tests**  
   - Evaluate how the system behaves with large datasets or multiple concurrent augmentation tasks.  
5. **Security & Hardening Tests** (optional, if relevant)  
   - Check for secure handling of credentials, rate-limiting, etc.  

---

## 4. Unit Testing

### 4.1 Approach

- Use a Python testing framework such as **pytest** or **unittest**.  
- **Mock** all external calls (PubMed, finance APIs, HPC job submissions) so tests are deterministic and do not require network/hardware dependencies.  
- Keep each test method **small** and **focused** on a single function or class behavior.

### 4.2 Example: Testing `PubMedAugmenter`

```python
# test_augmenters/test_pubmed_augmenter.py

import pytest
from unittest.mock import patch
import pandas as pd
from data_augmentation_framework.augmenters_api import PubMedAugmenter

@patch("data_augmentation_framework.augmenters_api.pubmed_augmenter.requests.get")
def test_pubmed_augmenter_basic(mock_requests_get):
    # 1. Setup mock response
    mock_requests_get.return_value.ok = True
    mock_requests_get.return_value.json.return_value = {
        "esearchresult": {
            "idlist": ["12345", "67890"]
        }
    }

    # 2. Create sample data
    df = pd.DataFrame({"gene_symbol": ["BRCA1", "BRCA1"]})

    # 3. Instantiate augmenter
    augmenter = PubMedAugmenter(query_column="gene_symbol")

    # 4. Run augmentation
    augmented_df = augmenter.augment_data(df)

    # 5. Verify data was updated
    assert "pmids" in augmented_df.columns
    assert "abstract_snippets" in augmented_df.columns
    # Potentially check context logs as well
```

#### Key Points
- **Mocking**: `requests.get` is mocked to return a **fake** eSearch response with PMIDs.  
- **Assertion**: We confirm new columns (`pmids`, `abstract_snippets`) exist.

### 4.3 Additional Unit Tests

- **FinanceAugmenter**: Mock finance API responses.  
- **GeneAugmenter**: Mock gene database calls or local references.  
- **HPCAugmenter**: Mock HPC job submission and completion.

---

## 5. Mocking External Services

### 5.1 Rationale

- **Stability**: Tests won’t fail due to external downtime or rate-limit issues.  
- **Speed**: No network calls → faster test runs.  
- **Isolation**: True unit tests only validate the internal logic.

### 5.2 Techniques

- **`unittest.mock.patch`**: Replace functions like `requests.get` or HPC submission calls with local stubs.  
- **Dedicated Mock Classes**: Implement fake HPC managers or data fetchers that return static JSON or error responses to test retry logic.

### 5.3 Example HPC Mock

```python
# test_concurrency/test_hpc_augmenter.py

@patch("data_augmentation_framework.augmenters_api.hpc_augmenter.submit_hpc_job")
def test_hpc_augmenter_success(mock_submit):
    # Simulate HPC job returning a job ID
    mock_submit.return_value = "JOB_123"

    # Then simulate job completion or poll logic
    # ...
```

---

## 6. Integration Tests

### 6.1 Purpose

- Ensure multiple components (e.g., **context integration**, **augments**, **caching**, **concurrency**) work together as expected.  
- **Partial Mocks**: Might still stub out external APIs, but allow real concurrency or real context store usage.

### 6.2 Example Scenario

**Concurrent PubMedAugmenter** test:

1. Provide a **large** DataFrame.  
2. Invoke the concurrency manager to chunk the data (e.g., 10k rows each).  
3. Confirm that **each chunk** is processed in parallel, the results are merged, and context logs exist for each chunk.  
4. **Mock** external API but keep concurrency logic real.

---

## 7. End-to-End (E2E) Tests

### 7.1 Goals

- Replicate a **real user workflow** (like a TheraLab orchestrator calling multiple augmenters in sequence).  
- Use **live** or **sandbox** external APIs if feasible, or high-level mocks that simulate the full request/response cycle.

### 7.2 Example E2E Flow

1. **Upload** a dataset (e.g., gene symbols).  
2. **Run** `augment_data(...)` with two augmenters: `GeneAugmenter` then `PubMedAugmenter`.  
3. **Check** final DataFrame columns, confirm context logs for each transformation.  
4. **Optionally**: measure performance or concurrency usage.

**Sample Pseudocode**:

```python
def test_end_to_end_gene_and_pubmed():
    df = pd.DataFrame({"gene_id": ["ENSG00000139618", "ENSG00000141510"]})

    # 1. GeneAugmenter normalizes to official symbols
    gene_aug = GeneAugmenter(gene_column="gene_id")
    df_aug1 = gene_aug.augment_data(df)

    # 2. PubMedAugmenter fetches references for that symbol
    pubmed_aug = PubMedAugmenter(query_column="gene_symbol")
    df_aug2 = pubmed_aug.augment_data(df_aug1)

    # 3. Validate final results
    assert "gene_symbol" in df_aug2.columns  # from GeneAugmenter
    assert "pmids" in df_aug2.columns       # from PubMedAugmenter
    # Check context logs
    adapter = getattr(df_aug2, "_context_adapter", None)
    assert adapter is not None
    # Possibly verify that 2 context entries exist for each row (Gene + PubMed).
```

---

## 8. Performance & Concurrency Testing

### 8.1 Large Datasets

- Test the framework with **large** (100k+ rows) or multi-GB data to see if concurrency logic, chunking, and memory usage behave as expected.  
- **Measure** runtime, memory footprint, CPU usage.

### 8.2 HPC & Parallelism

- If HPC is part of your environment, run **stress tests** submitting many HPC jobs in parallel.  
- Validate that **retry** or error handling works if HPC nodes are busy or fail mid-job.

---

## 9. Test Data & Configuration

- **Small Synthetic Datasets**: For quick unit tests (1-100 rows).  
- **Medium Realistic Datasets** (1k-10k rows): For integration tests.  
- **Large Scale** (100k+ rows): For performance/concurrency tests (run in CI or separate environment if resources are limited).

> Store these test files either in `test/data/` or generate them on-the-fly.

---

## 10. CI/CD Integration

1. **Automated Test Runs**:  
   - All unit and integration tests run on each pull request (e.g., GitHub Actions, Jenkins).  
   - Possibly separate an HPC test suite if HPC resources are not always available.
2. **Code Coverage**:  
   - Use a tool like **coverage.py** to measure test coverage.  
   - Aim for **80%+** coverage, especially for critical augmenters and concurrency logic.
3. **Test Reporting**:  
   - Generate HTML or XML reports to quickly identify failing tests.  
   - Track performance regression if relevant (e.g., comparing test run times over commits).

---

## 11. Edge Cases & Special Testing Considerations

1. **Partial Failures**:  
   - Some chunk fails mid-augmentation, but others succeed. Ensure the concurrency manager logs partial success or final error state.  
2. **Rate Limits** & **429 Errors**:  
   - Test the framework’s **retry logic** for external APIs.  
3. **Caching**:  
   - Verify that repeated queries retrieve data from cache, not from external APIs, and that the context logs reflect the usage of cached data.  
4. **User Annotation Conflicts**:  
   - If user-provided context conflicts with new augmentations, ensure the logs capture both and no crash occurs.

---

## 12. Maintenance & Future Updates

- Keep tests **organized** by module (e.g., `test_augmenters/`, `test_concurrency/`, `test_integration/`).  
- As new augmenters or HPC logic are added, create or update corresponding tests.  
- Revisit **mock** configurations whenever external API endpoints or HPC submission commands change.

---

## 13. Summary

A **robust testing plan** is essential to ensure that the **data-augmentation-framework** remains **reliable**, **scalable**, and **maintainable**. By combining **unit**, **integration**, and **end-to-end** tests—plus careful **mocking** of external services—we can confidently develop new augmenters, concurrency features, and HPC integrations without introducing regressions or instability.

**Key Takeaways**:
- **Mock external APIs** in unit tests for stability and isolation.  
- **Integration tests** check interoperability among augmenters, concurrency logic, and context-framework.  
- **End-to-end tests** replicate real user workflows and validate the entire pipeline.  
- **Performance tests** with large data ensure concurrency and HPC features scale properly.

---

**Document History**  
- **v1.0** – Initial testing plan, including unit, integration, and E2E strategies.  
- **v1.1** – Added concurrency/performance testing guidelines and HPC-specific scenarios.

_End of Document_