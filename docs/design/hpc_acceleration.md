# Data Augmentation Framework  
**HPC Acceleration Design**  
`docs/design/hpc_acceleration.md`

---

## 1. Introduction & Scope

### 1.1 Purpose

This document explains **how** the **data-augmentation-framework** integrates with **HPC resources** (e.g., clusters, GPU nodes) to accelerate **large-scale** or **compute-intensive** transformations. It addresses:

- **Why** HPC is beneficial for data augmentation tasks.  
- **How** HPC jobs are orchestrated (submission, monitoring, retrieval).  
- **Design patterns** for chunking, concurrency, and integrating HPC output back into the framework.  
- **Performance benchmarks** guidelines (e.g., see `examples/` folder for real-world metrics).

### 1.2 Audience

- **Developers** extending HPC-based augmenters or concurrency managers.  
- **Ops/DevOps** engineers deploying HPC solutions.  
- **Researchers** wanting to understand HPC scaling for large data transformations.

### 1.3 Scope

- **In-Scope**: HPC job submission patterns, HPC-based concurrency, data flow, logging, security considerations.  
- **Out-of-Scope**: Low-level cluster configuration or specific HPC scheduler settings (e.g., Slurm vs. PBS vs. Kubernetes).  

---

## 2. HPC Use Cases & Motivation

1. **Large Datasets**: Genomics or finance data can easily reach millions of rows or multi-GB sizes.  
2. **Intensive Computations**: Some augmentations (e.g., complex transformations, gene expression modeling) may require GPU acceleration or parallel CPU nodes.  
3. **Scalability**: HPC allows distributing tasks across multiple nodes, reducing total run time.  
4. **Resource Management**: Instead of overloading a single machine, HPC clusters offer scheduling and job queuing to maximize throughput and resource utilization.

---

## 3. High-Level Architecture

Below is a simplified architecture showing how the data-augmentation-framework interacts with an HPC cluster or job manager.

```
 ┌─────────────────────────────────────────────────────────────┐
 |     TheraLab / Caller                                      |
 |  (e.g., multi-agent orchestrator or user commands)          |
 └─────────────────────────────────────────────────────────────┘
            |                  
            | (request HPC-based augmentation)
            v                  
 ┌─────────────────────────────────────────────────────────────┐
 |  Data Augmentation Framework                               |
 |  - HPCAugmenter or HPC concurrency manager                 |
 |  - Orchestrates job submission & retrieval                 |
 └─────────────────────────────────────────────────────────────┘
            |
            v
 ┌─────────────────────────────────────────────────────────────┐
 |     HPC Cluster / Scheduler (Slurm, PBS, Dask, etc.)        |
 |  - Receives batch jobs or distributed tasks                 |
 |  - Runs transformations (CPU, GPU, memory, etc.)            |
 |  - Saves output logs/data to shared storage                 |
 └─────────────────────────────────────────────────────────────┘

            ^
            | (results or logs)
            |
 ┌─────────────────────────────────────────────────────────────┐
 |  Context Framework & Data Stores                            |
 |  - Logs HPC transformations, job IDs, resources used        |
 |  - Merges HPC output back into final context-aware dataset  |
 └─────────────────────────────────────────────────────────────┘
```

---

## 4. HPC Integration Design

### 4.1 HPCAugmenter

The **HPCAugmenter** (or similarly named class) is responsible for:

1. **Job Submission**: Packaging data, code, or parameters into an HPC job.  
2. **Monitoring**: Checking job status (running, completed, failed).  
3. **Result Retrieval**: Locating output files or data post-run and merging them back into the original dataset.  
4. **Context Logging**: Recording HPC details (job ID, node usage, start/end times) in the **context-framework**.

**Key Methods** (example):

```python
class HPCAugmenter(BaseAugmenter):
    def augment_data(self, data, **kwargs):
        # 1. Submit HPC job (job_id = self._submit_job(data))
        # 2. Poll/wait for job completion
        # 3. Load HPC output into a local dataframe or context store
        # 4. Add HPC-related context logs
        return updated_data

    def _submit_job(self, data):
        # Convert data to a suitable format (CSV, parquet, etc.)
        # Possibly copy to shared filesystem
        # Use HPC command (e.g. `sbatch`, `qsub`, or library calls)
        # Return job_id
        pass
```

#### 4.1.1 HPC-Specific Config
- **Queue Name** (e.g., “gpu”, “batch”).  
- **Required Resources**: CPU count, memory, GPU type, etc.  
- **Time Limits**: HPC job wall-time.  
- **Credentials**: HPC tokens or user credentials (see [`docs/security/data_source_auth.md`](../security/data_source_auth.md)).

---

### 4.2 Concurrency Manager & HPC

In some workflows, the concurrency manager might **split data** into smaller chunks, each chunk submitted as a separate HPC job. For example:

1. **Chunk** 1: Rows 0–99,999  
2. **Chunk** 2: Rows 100,000–199,999  
3. **Chunk** 3: … etc.

The HPC concurrency layer can handle **multiple job submissions** and wait until all are complete before merging results. This approach is especially beneficial for **massive** datasets.

---

## 5. HPC Data Flow

### 5.1 Typical Steps

1. **Data Preparation**: Export the relevant subset of data to a file (CSV, Parquet, HDF5) on a **shared filesystem** accessible by the HPC cluster.  
2. **Submission**: Issue a command to the HPC scheduler (e.g., `sbatch transform_script.sh`) with arguments referencing the data file location.  
3. **Cluster Execution**: HPC nodes run the transformation script or code, possibly using GPU libraries or multi-threaded CPU.  
4. **Output Storage**: The HPC job writes results to an output file or database.  
5. **Retrieval**: The HPCAugmenter or concurrency manager copies/reads the output back into a local DataFrame (or context store).  
6. **Context Logging**: Record HPC details (job ID, node usage, run time, etc.) in the context store.

---

## 6. Error Handling & Retry

### 6.1 Transient HPC Failures

- Node preemption, queue timeouts, or job scheduling issues can cause HPC jobs to fail.  
- The HPCAugmenter should **retry** job submission a configurable number of times (e.g., `max_retries=2`).  

### 6.2 Permanent Failures

- Missing dependencies on HPC nodes, invalid user credentials, or script errors might produce immediate or repeated failures.  
- HPCAugmenter logs these errors (including HPC job logs if retrievable) and raises an exception up the chain.

For more details, see [error & retry policy](../design/error_retry_policy.md).

---

## 7. Performance Benchmarking

To measure HPC acceleration, you can run **benchmark scripts** in an `examples/performance_benchmarks/` folder (or similarly named) that:

1. **Generate** large synthetic datasets (e.g., 1 million rows).  
2. **Invoke** HPC-based augmentation (e.g., a GPU-accelerated transform).  
3. **Track** metrics: total runtime, HPC queue wait time, HPC node usage, concurrency.  
4. **Log** or **plot** results to compare HPC vs. non-HPC performance.

**Recommended**: Keep your performance benchmarks outside core unit tests (which should remain quick and deterministic). Instead, run benchmarks periodically or in separate HPC test environments.

---

## 8. Example HPCAugmenter Implementation (Simplified)

```python
# hpc_augmenter.py (within data_augmentation_framework)

import subprocess
import time
from context_framework.adapters import ensure_context_adapter

class HPCAugmenter(BaseAugmenter):
    def __init__(self, job_name="HPC_Augment", queue="batch", cpus=4, gpus=0, max_retries=2, **kwargs):
        super().__init__(max_retries=max_retries, **kwargs)
        self.job_name = job_name
        self.queue = queue
        self.cpus = cpus
        self.gpus = gpus

    def augment_data(self, data, **kwargs):
        adapter = ensure_context_adapter(data)

        # 1. Export data to shared location
        input_path = self._write_input_to_shared(data)

        # 2. Submit HPC job
        job_id = None
        attempt = 0
        while attempt < self.max_retries:
            try:
                job_id = self._submit_job(input_path)
                break  # success
            except TemporaryHPCError as e:
                attempt += 1
                if attempt == self.max_retries:
                    raise e
                time.sleep(30)  # backoff
        if not job_id:
            raise RuntimeError("HPC job submission failed.")

        # 3. Wait for job completion
        self._wait_for_job(job_id)

        # 4. Retrieve HPC output
        output_df = self._load_output_from_shared(input_path)
        # Combine output_df into original data if needed
        data = self._merge_results(data, output_df)

        # 5. Log context
        adapter.add_context(
            item_id="global",
            source="HPCAugmenter",
            details={
                "job_name": self.job_name,
                "job_id": job_id,
                "queue": self.queue,
                "cpus": self.cpus,
                "gpus": self.gpus,
                "timestamp": time.time()
            }
        )
        return data

    def _write_input_to_shared(self, df):
        # Writes df to a shared file system path, e.g. /mnt/shared/...
        shared_path = "/mnt/shared/hpc_temp_input.csv"
        df.to_csv(shared_path, index=False)
        return shared_path

    def _submit_job(self, input_path):
        # Example: Slurm sbatch
        cmd = f"sbatch --job-name={self.job_name} --cpus-per-task={self.cpus}"
        if self.gpus > 0:
            cmd += f" --gres=gpu:{self.gpus}"
        cmd += f" transform_script.sh {input_path}"
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True)
        if result.returncode != 0:
            # parse error, handle transient or permanent
            raise TemporaryHPCError(f"Slurm submission failed: {result.stderr}")
        # parse job id from output, e.g. "Submitted batch job 12345"
        job_id = self._parse_job_id(result.stdout)
        return job_id

    def _wait_for_job(self, job_id):
        # Poll HPC scheduler (squeue or equivalent) until job completes
        while True:
            # e.g., parse squeue
            if self._job_finished(job_id):
                break
            time.sleep(10)

    def _load_output_from_shared(self, input_path):
        # HPC script might produce "output.csv" next to input
        output_path = input_path.replace("input.csv", "output.csv")
        import pandas as pd
        return pd.read_csv(output_path)

    def _merge_results(self, original_df, output_df):
        # domain-specific logic: merge HPC-transformed columns back
        return output_df

    def _parse_job_id(self, stdout_str):
        # parse job ID from "Submitted batch job <job_id>"
        return stdout_str.strip().split()[-1]

    def _job_finished(self, job_id):
        # e.g., run "squeue -j job_id"
        # if not found, assume finished
        return True
```

> **Note**: This example is **simplified**. Real HPC usage might involve environment modules, specialized scripts, more robust error handling, or HPC resource validation.

---

## 9. Security & Credentials

- HPC schedulers often require user credentials or tokens.  
- Follow the guidelines in [`docs/security/data_source_auth.md`](../security/data_source_auth.md) to store HPC credentials (e.g., environment variables, secrets manager).  
- **Do not** log tokens or passwords in HPC job scripts.

---

## 10. Context Logging

Every HPC job invocation should record:

1. **Job ID**  
2. **Queue / Partition**  
3. **Requested resources** (CPUs, GPUs, memory, time limit)  
4. **Start/End Times** (if available)  
5. **Output location** (path to HPC results)

Use the same `adapter.add_context(...)` approach to ensure all HPC operations are traceable.

---

## 11. Testing HPC Features

- **Unit Tests**: Mock HPC calls (subprocess, cluster submission) for quick local testing.  
- **Integration Tests**: Possibly run small HPC jobs in a test cluster or container environment if available.  
- **Performance / Stress Tests**: Launch multiple HPC augmentations on large data to confirm scaling.  
- **Examples**: Provide scripts in `examples/hpc_performance_benchmarks/` for real HPC usage scenarios.

---

## 12. Future Extensions

1. **Distributed HPC**: Expand to Dask or Spark for distributed data frames or real-time streaming.  
2. **Adaptive Scheduling**: Adjust HPC resource requests dynamically based on dataset size or HPC load.  
3. **Cloud HPC**: Integrate with AWS Batch, GCP Dataproc, or Azure Batch for on-demand HPC resources.  
4. **Advanced Retry / Checkpointing**: If HPC tasks fail mid-way, store partial progress and resume.

---

## 13. Conclusion

The **HPC Acceleration** design allows the **data-augmentation-framework** to offload **resource-intensive** transformations to cluster or GPU environments, ensuring **scalability** for large datasets. By leveraging job schedulers, chunk-based concurrency, and robust context logging, the framework provides a **traceable**, **fault-tolerant** HPC workflow that can be extended to a variety of scientific, financial, or big-data use cases.

For further details, see:

- [Testing Plan](../../test/testing_plan.md) for HPC test strategies.  
- [Security & Auth](../../security/data_source_auth.md) for storing HPC credentials.  
- [Error & Retry Policy](error_retry_policy.md) for HPC transient failures.  
- [Performance Benchmarks (Examples)](../../examples/) for real-world HPC usage demonstrations.

---

**Document History**  
- **v1.0** – Initial HPC acceleration design, detailing HPCAugmenter patterns.  
- **v1.1** – Added concurrency manager references, advanced HPC usage scenarios, and performance benchmark notes.

_End of Document_