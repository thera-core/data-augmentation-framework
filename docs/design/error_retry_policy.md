# Data Augmentation Framework  
**Error & Retry Policy**  
`docs/design/error_retry_policy.md`

## 1. Introduction

This document outlines the **error handling** and **retry strategies** within the **data-augmentation-framework**, covering how various augmenters, concurrency managers, and API integrations respond to exceptions and network failures. The goal is to ensure that the framework operates reliably under adverse conditions—such as transient network issues, rate limits, or HPC node failures—and can recover or fail gracefully.

### 1.1 Purpose

- Define a **consistent** approach for catching, logging, and retrying errors.  
- Provide guidelines for **configurable** retry behaviors based on error severity or frequency.  
- Clarify how each component (augmenters, concurrency, HPC manager) should implement or inherit these policies.

### 1.2 Scope

- **In-Scope**:  
  - Transient network failures, rate limit exceptions, HPC job errors, data parsing errors, and fallback mechanisms.  
  - How retries are configured, executed, and logged within the data-augmentation-framework.  
- **Out-of-Scope**:  
  - Detailed concurrency architecture (see [`architecture.md`](architecture.md) and concurrency modules).  
  - Security or credential-related errors (addressed in separate security policies).

---

## 2. Error Classification

Errors encountered in the **data-augmentation-framework** generally fall into three broad categories:

1. **Transient/Recoverable Errors**  
   - E.g., temporary network downtime, rate-limited (HTTP 429) responses, timeouts from external APIs.  
   - **Policy**: Attempt an **automatic retry** with exponential backoff (within configurable limits).

2. **Permanent/Non-Recoverable Errors**  
   - E.g., invalid user input (non-existent column, malformed gene ID), data format mismatch, “404 Not Found” for a resource that truly does not exist.  
   - **Policy**: **Do not retry**. Fail the current chunk or operation, log the error, and notify the caller or orchestrator.

3. **System/HPC Errors**  
   - E.g., HPC node crashes, job scheduling failures, concurrency manager meltdown.  
   - **Policy**: Depending on severity, attempt **one or more** retries or fallback to an alternative node or local processing. If all attempts fail, log an unrecoverable error.

---

## 3. Retry Strategies

### 3.1 Exponential Backoff

For **transient** errors (e.g., HTTP 500 or 502 from an external API, or HPC cluster timeouts), the recommended approach is **exponential backoff**:

1. **Initial Delay**: A short wait (e.g., 1 second).  
2. **Backoff Factor**: Multiply the delay by a factor (e.g., 2) each subsequent attempt.  
3. **Max Retries**: Stop after a certain number of retries (e.g., 3 or 5) or if the backoff interval exceeds a maximum threshold (e.g., 1 minute).

Example:

| Retry Attempt | Delay          |
|---------------|----------------|
| 1 (first)     | 1 second       |
| 2 (second)    | 2 seconds      |
| 3 (third)     | 4 seconds      |
| 4 (fourth)    | 8 seconds      |
| 5 (fifth)     | 16 seconds     |

If the error persists beyond the **fifth** attempt, the framework aborts the augmentation for that chunk or request.

### 3.2 Fixed Delay or Linear Backoff

Some APIs (e.g., certain HPC systems) may recommend a **fixed** or **linear** delay:

- **Fixed**: Always wait N seconds between retries (e.g., 5 seconds).  
- **Linear**: Wait increments by a constant each time (e.g., 2s, 4s, 6s, 8s…).

**Implementation**:  
- For HPC tasks that fail frequently due to resource contention, a short **fixed** delay (e.g., 30s) before resubmitting might be sufficient.  
- For external APIs known to have limited rate limits, exponential backoff is generally more robust.

### 3.3 Jitter

To avoid **thundering herd** problems (multiple concurrent tasks all retrying at once), the framework may inject a **random jitter** (+/- 0.2 * delay) into backoff intervals. This helps stagger retry attempts.

---

## 4. Implementation Guidelines

### 4.1 Augmenters

Each augmenter (e.g., `PubMedAugmenter`, `FinanceAugmenter`) should:

1. **Catch** known transient exceptions (e.g., `requests.exceptions.Timeout`, `HTTP 429`) and apply a retry loop.  
2. **Distinguish** between errors that are truly permanent (e.g., invalid query parameters) vs. network or rate-limit errors.  
3. **Log** each retry attempt with the relevant context (timestamp, attempt number, error message).

> **Recommended**: Use a shared utility function or decorator (e.g., `@retry_on_exception(...)`) in `base_augmenter.py` to unify logic across different augmenters.

### 4.2 Concurrency Manager

When dealing with chunked or parallel tasks:

- **Chunk-Level Retries**: If a chunk fails due to a transient error, only **re-try** that chunk. Do not abort the entire augmentation.  
- **Maximum Failures**: If a certain percentage (e.g., >50%) of chunks fail repeatedly, consider stopping the entire job.  
- **HPC-Specific**: For HPC tasks, the concurrency manager might have logic to **resubmit** a job to a different node or a smaller/larger job queue after a failure.

### 4.3 HPC Manager

For HPC-based transformations:

1. **Transient HPC Errors**:  
   - E.g., node preemption, queue timeouts.  
   - Retry the job submission a few times. Possibly increase resources requested.  
2. **Fatal HPC Errors**:  
   - Compilation errors, missing dependencies on HPC node.  
   - Abort and log the failure.  
3. **Context Logging**:  
   - Each HPC retry attempt must be logged with the job ID, node ID, or resource changes.

### 4.4 Caching Layer

- **Failed Queries**: If an external API returns an error, **do not cache** the response.  
- **Partial Success**: If an API call returns partial data but some items fail, store the partial data but mark failed items for re-query next time.

---

## 5. Configuration & Customization

To accommodate different operational environments, the framework allows **configurable retry policies** via:

1. **Environment Variables**:  
   - `DEFAULT_MAX_RETRIES`: Maximum attempts (default=3).  
   - `DEFAULT_BACKOFF_FACTOR`: Exponential backoff factor (default=2).  
   - `ENABLE_JITTER`: Boolean toggle for random jitter.
2. **Augmenter-Specific Parameters**:  
   - Some augmenters may override the default policy if their target API or HPC environment has unique constraints.  
   - E.g., `PubMedAugmenter(backoff_factor=1.5, max_retries=5)`.
3. **Global Config Files**:  
   - A YAML or JSON file (`retry_config.yaml`) can store more advanced per-augmenter or per-environment settings.

---

## 6. Logging & Monitoring

Each retry attempt or error event should be **logged** with sufficient detail for troubleshooting:

- **Log Fields**:  
  - Timestamp, augmenter name, error type, attempt number, delay until next attempt, partial stack trace (if available).  
  - HPC or concurrency logs: job ID, node info, chunk ID, etc.
- **Log Levels**:  
  - **Transient Retried Errors**: `WARN` or `INFO`, depending on severity.  
  - **Permanent Failures**: `ERROR`.  
  - **Uncaught Exceptions**: `CRITICAL`.

> **Monitoring**:  
> Integration with application performance monitoring (APM) tools or log aggregation (e.g., ELK, Splunk) can provide real-time alerts if repeated failures occur.

---

## 7. Example Pseudocode

```python
# base_augmenter.py

import time
import random
import logging

DEFAULT_MAX_RETRIES = 3
DEFAULT_BACKOFF_FACTOR = 2
ENABLE_JITTER = True

def retry_on_exception(func):
    """Decorator to automatically retry augmentations on transient errors."""
    def wrapper(*args, **kwargs):
        max_retries = kwargs.pop('max_retries', DEFAULT_MAX_RETRIES)
        backoff_factor = kwargs.pop('backoff_factor', DEFAULT_BACKOFF_FACTOR)
        
        attempt = 0
        delay = 1
        while True:
            try:
                return func(*args, **kwargs)
            except TransientError as e:
                attempt += 1
                if attempt > max_retries:
                    logging.error(f"[{func.__name__}] Exceeded max retries. Error: {e}")
                    raise  # escalate up
                # apply backoff
                sleep_time = delay * (backoff_factor ** (attempt - 1))
                if ENABLE_JITTER:
                    jitter = random.uniform(0, 0.2 * sleep_time)
                    sleep_time += jitter
                logging.warning(f"[{func.__name__}] TransientError encountered. "
                                f"Retrying attempt {attempt}/{max_retries} in {sleep_time:.2f}s. Error: {e}")
                time.sleep(sleep_time)
            except PermanentError as e:
                logging.error(f"[{func.__name__}] Permanent error, no retry. Error: {e}")
                raise
    return wrapper
```

- **`TransientError`** and **`PermanentError`** would be custom exceptions or mapped from typical exceptions (HTTP 429 → `TransientError`, HTTP 404 → `PermanentError`, etc.).  
- Augmenters can decorate their main data-fetching functions with `@retry_on_exception` for standardized retry logic.

---

## 8. Edge Cases

1. **Repeated Permanent Errors**: If the input data is invalid, repeated calls will keep failing. The framework should **fail fast** rather than repeatedly retrying.  
2. **Partial Success**: Some chunked tasks might succeed while others fail. The concurrency manager should handle these partial outcomes gracefully.  
3. **Backpressure**: If multiple tasks all fail simultaneously, be mindful not to create large retry storms. The concurrency manager should throttle overall attempts.  
4. **User Cancellation**: If the user (e.g., TheraLab session) cancels midway, any ongoing retries should be stopped.

---

## 9. Conclusion

A robust **Error & Retry Policy** is crucial for ensuring that data augmentation tasks gracefully handle transient errors (network issues, rate limits, HPC node failures) while avoiding unnecessary retries on permanent failures. By adhering to these **best practices**—including **exponential backoff**, **error classification**, and **comprehensive logging**—the **data-augmentation-framework** can provide a **stable** and **predictable** experience for end users and downstream systems.

For more details, see:

- [Sequence Diagrams](sequence_diagrams.md) for how error handling is visualized.  
- [Data Source Integration Specs](data_source_integration.md) for API-specific errors and rate limits.  
- [Concurrency & Architecture Overview](architecture.md) for how chunked parallel tasks manage retries individually.

---

**Document History**  
- **v1.0** – Initial publication of retry logic and error classification strategies.  
- **v1.1** – Added HPC considerations and advanced configuration examples.

_End of Document_