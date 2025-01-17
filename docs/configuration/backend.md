The same Ludwig config / Python code that runs on your local machine can be executed remotely in a distributed manner
with zero code changes. This distributed execution includes preprocessing, training, and batch prediction.

In most cases, Ludwig will be able to automatically detect if you're running in an environment that supports distributed
execution, but you can also make this explicit on the command line with the `--backend` arg or by providing a `backend`
section to the Ludwig config YAML:

```yaml
backend:
  type: ray
  cache_dir: s3://my_bucket/cache
  cache_credentials: /home/user/.credentials.json
  processor:
    type: dask
  trainer:
    strategy: ddp
  loader: {}
```

Parameters:

- `type`: How the job will be distributed, one of `local`, `ray`, `deepspeed`, `horovod`.
- `cache_dir`: Where the preprocessed data will be written on disk, defaults to the location of the input dataset. See [Cloud Storage](../user_guide/cloud_storage.md#remote-dataset-cache) for more details
- `cache_credentials`: Optional dictionary of credentials (or path to credential JSON file) used to write to the cache. See [Cloud Storage](../user_guide/cloud_storage.md#using-different-cache-and-dataset-filesystems) for more details
- `processor`: (Ray only) parameters to configure execution of distributed data processing.
- `trainer`: (Ray only) parameters to configure execution of distributed training.
- `loader`: (Ray only) parameters to configure data loading from processed data to training batches.

# Processor

The `processor` section configures distributed data processing. The `local` backend uses the Pandas dataframe library, which runs in a single
process with the entire datasets in memory. To make the data processing scalable to large datasets, we support two distributed dataframe libraries
with the `ray` backend:

- `dask`: (default) a lazily executed version of distributed Pandas.
- `modin`: an eagerly executed version of distributed Pandas.

## Dask

[Dask](https://dask.org/) is the default distributed data processing library when using Ludwig on Ray. It executes distributed Pandas operations
on partitions of the data in parallel. One beneficial property of Dask is that it is executed lazily, which allows it to stream very large datasets
without needing to hold the entire dataset in distributed memory at once.

One downside to Dask is that it can require some tuning to get the best performance. There are two knobs we expose in Ludwig for tuning Dask:

- `parallelism`: the number of partitions to divide the dataset into (defaults to letting Dask figure this out automatically).
- `persist`: whether intermediate stages of preprocessing should be cached in distributed memory (default: `true`).

Increasing `parallelism` can reduce memory pressure during preprocessing for large datasets and increase parallelism (horizontal scaling). The downside to
too much parallelism is that there is some overhead for each partition-level operation (serialization and deserialization), which can dominate the runtime
if set too high.

Setting `persist` to `false` can be useful if the dataset is too large for all the memory and disk of the entire Ray cluster. Only set this to `false` if you're
seeing issues running out of memory or disk space.

Example:

```yaml
backend:
  type: ray
  processor:
    type: dask
    parallelism: 100
    persist: true
```

## Modin

[Modin](https://github.com/modin-project/modin) is an eagerly-executed distributed dataframe library that closely mirrors the behavior of
Pandas. Because it behaves almost identically to Pandas but is able to distribute the dataset
across the Ray cluster, there are fewer things to configure to optimize its performance.

Support for Modin is currently experimental.

Example:

```yaml
backend:
  type: ray
  processor:
    type: modin
```

# Trainer

The `trainer` section configures distributed training strategy. Currently we support the following strategies (described
in detail below) with more coming in the future:

- Distributed Data Parallel (DDP)
- DeepSpeed
- Horovod

The following parameters can be configured for any distributed strategy:

- `strategy`: one of `horovod`, `ddp`, or `fsdp`.
- `use_gpu`: whether to use GPUs for training (defaults to `true` when the cluster has at least one GPU).
- `num_workers`: how many Horovod workers to use for training (defaults to the number of GPUs, or 1 if no GPUs are found).
- `resources_per_worker`: the Ray resources to assign to each Horovod worker (defaults to 1 CPU and 1 GPU if available).
- `logdir`: path to the file directory where logs should be persisted.
- `max_retries`: number of retries when Ray actors fail (defaults to 3).

See the [Ray Train API](https://docs.ray.io/en/latest/train/api.html#trainer) for more details on these parameters.

Example:

```yaml
backend:
  type: ray
  trainer:
    strategy: ddp
    use_gpu: true
    num_workers: 4
    resources_per_worker:
        CPU: 2
        GPU: 1
```

In most cases, you shouldn't need to set these values explicitly, as Ludwig will set them on your behalf to maximize
the resources available in the cluster. There are two cases in which it makes sense to set these values explicitly, however:

- Your Ray cluster makes use of autoscaling (in which case Ludwig will not know how many GPUs are available with scaling).
- Your Ray cluster is running multiple workloads at once (in which case you may wish to reserve resources for other jobs).

## Distributed Data Parallel (DDP)

[Distributed Data Parallel](https://pytorch.org/tutorials/intermediate/ddp_tutorial.html) or DDP is PyTorch's native data-parallel
library that does not require installing any additional packages to use. It is the default strategy when using the `ray` backend.

```yaml
backend:
  type: ray
  trainer:
    strategy: ddp
```

## DeepSpeed

[DeepSpeed](https://github.com/microsoft/DeepSpeed) is a data + model parallel framework for training very large models whose parameters
are then spread across multiple GPUs during training.

The primary scenario to use DeepSpeed is when the model you're training is too large to fit into a single GPU
(e.g., fine-tuning a large language model like Llama-2).
When the model is small enough to fit in a single GPU, however, benchmarking has shown it's generally better 
to use a data parallel framework like DDP or Horovod.

```yaml
backend:
  type: ray
  trainer:
    strategy: deepspeed
```

DeepSpeed comes with a number of [configuration options](https://www.deepspeed.ai/docs/config-json/), most of which can be provided in
the Ludwig config as shown in the DeepSpeed documentation.

The currently supported DeepSpeed config section in Ludwig include:

- `zero_optimization`, controls the optimization stage used to reduce GPU memory pressure during training.
- `fp16`, whether to train with float16 precision.
- `bf16`, whether to train with bfloat16 precision on supported hardware (e.g., Ampere architecture GPUs and above).

Other parameters like batch size, gradient accumulation, and optimizer params are specified as part of the normal Ludwig `trainer`
section of the config. When DeepSpeed is used as the backend, Ludwig will defer the implementation of these settings to DeepSpeed automatically.

Example DeepSpeed config using ZeRO stage 3, optimizer offload, and bfloat16:

```yaml
backend:
  type: ray
  trainer:
    use_gpu: true
    strategy:
      type: deepspeed
      zero_optimization:
        stage: 3
        offload_optimizer:
          device: cpu
          pin_memory: true
      bf16:
        enabled: true
```

## Horovod

[Horovod](https://horovod.ai/) is a distributed data-parallel framework that is optimized for bandwidth-constrained computing
environments. It makes use of Nvidia's NCCL for fast GPU-to-GPU communication.

In benchmarks, we found DDP and Horovod to perform near identically, so if you're not already using Horovod, DDP is the easiest
way to get started with distributed training in Ludwig.

Example:

```yaml
backend:
  type: ray
  trainer:
    strategy: horovod
```

# Loader

The `loader` section configures the "last mile" data ingest from processed data (typically cached in the Parquet
format) to tensor batches used for training the model.

When training a deep learning model at scale, the chain is only as strong as its weakest link -- in other words, the data loading pipeline needs to be at least as fast as the GPU forward / backward passes, otherwise the whole process will be bottlenecked by the data loader.

In most cases, Ludwig's defaults will be sufficient to get good data loading performance, but if you notice GPU utilization dropping even after
scaling the batch size, or going long periods at 0% utilization before spiking up again, you may need to tune the `loader` parameters below to
improve throughput:

- `fully_executed`: Force full evaluation of the preprocessed dataset by loading all blocks into cluster memory / storage (defaults to `true`). Disable this if the dataset is much larger than the total amount of cluster memory allocated to the Ray object store and you notice that object spilling is occurring frequently during training.
- `window_size_bytes`: Load and shuffle the preprocessed dataset in discrete windows of this size (defaults to `null`, meaning data will not be windowed). Try configuring this is if shuffling is taking a very long time, indicated by every epoch of training taking many minutes to start. In general, larger window sizes result in more uniform shuffling (which can lead to better model performance in some cases), while smaller window sizes will be faster to load. This setting is particularly useful when running hyperopt over a large dataset.

Example:

```yaml
backend:
  type: ray
  loader:
    fully_executed: false
    window_size_bytes: 500000000
```
