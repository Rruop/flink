---
title: Elastic Scaling
weight: 5
type: docs

---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Elastic Scaling

Apache Flink allows you to rescale your jobs. You can do this manually by stopping the job and restarting from the savepoint created during shutdown with a different parallelism.

This page describes options where Flink automatically adjusts the parallelism instead.

## Reactive Mode

{{< hint info >}}
Reactive mode is an MVP ("minimum viable product") feature. The Flink community is actively looking for feedback by users through our mailing lists. Please check the limitations listed on this page.
{{< /hint >}}

Reactive Mode configures a job so that it always uses all resources available in the cluster. Adding a TaskManager will scale up your job, removing resources will scale it down. Flink will manage the parallelism of the job, always setting it to the highest possible values.

Reactive Mode restarts a job on a rescaling event, restoring it from the latest completed checkpoint. This means that there is no overhead of creating a savepoint (which is needed for manually rescaling a job). Also, the amount of data that is reprocessed after rescaling depends on the checkpointing interval, and the restore time depends on the state size. 

The Reactive Mode allows Flink users to implement a powerful autoscaling mechanism, by having an external service monitor certain metrics, such as consumer lag, aggregate CPU utilization, throughput or latency. As soon as these metrics are above or below a certain threshold, additional TaskManagers can be added or removed from the Flink cluster. This could be implemented through changing the [replica factor](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#replicas) of a Kubernetes deployment, or an [autoscaling group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html) on AWS. This external service only needs to handle the resource allocation and deallocation. Flink will take care of keeping the job running with the resources available.
 
### Getting started

If you just want to try out Reactive Mode, follow these instructions. They assume that you are deploying Flink on a single machine.

```bash

# these instructions assume you are in the root directory of a Flink distribution.

# Put Job into lib/ directory
cp ./examples/streaming/TopSpeedWindowing.jar lib/
# Submit Job in Reactive Mode
./bin/standalone-job.sh start -Dscheduler-mode=reactive -Dexecution.checkpointing.interval="10s" -j org.apache.flink.streaming.examples.windowing.TopSpeedWindowing
# Start first TaskManager
./bin/taskmanager.sh start
```

Let's quickly examine the used submission command:
- `./bin/standalone-job.sh start` deploys Flink in [Application Mode]({{< ref "docs/deployment/overview" >}}#application-mode)
- `-Dscheduler-mode=reactive` enables Reactive Mode.
- `-Dexecution.checkpointing.interval="10s"` configure checkpointing and restart strategy.
- the last argument is passing the Job's main class name.

You have now started a Flink job in Reactive Mode. The [web interface](http://localhost:8081) shows that the job is running on one TaskManager. If you want to scale up the job, simply add another TaskManager to the cluster:
```bash
# Start additional TaskManager
./bin/taskmanager.sh start
```

To scale down, remove a TaskManager instance.
```bash
# Remove a TaskManager
./bin/taskmanager.sh stop
```

### Usage

#### Configuration

To enable Reactive Mode, you need to configure `scheduler-mode` to `reactive`.

The **parallelism of individual operators in a job will be determined by the scheduler**. It is not configurable
and will be ignored if explicitly set, either on individual operators or the entire job.

The only way of influencing the parallelism is by setting a max parallelism for an operator
(which will be respected by the scheduler). The maxParallelism is bounded by 2^15 (32768).
If you do not set a max parallelism for individual operators or the entire job, the
[default parallelism rules]({{< ref "docs/dev/datastream/execution/parallel" >}}#setting-the-maximum-parallelism) will be applied,
potentially applying lower bounds than the max possible value. As with the default scheduling mode, please take
the [best practices for parallelism]({{< ref "docs/ops/production_ready" >}}#set-an-explicit-max-parallelism) into consideration.

Note that such a high max parallelism might affect performance of the job, since more internal structures are needed to maintain [some internal structures](https://flink.apache.org/features/2017/07/04/flink-rescalable-state.html) of Flink.

When enabling Reactive Mode, the [`jobmanager.adaptive-scheduler.resource-wait-timeout`]({{< ref "docs/deployment/config">}}#jobmanager-adaptive-scheduler-resource-wait-timeout) configuration key will default to `-1`. This means that the JobManager will run forever waiting for sufficient resources.
If you want the JobManager to stop after a certain time without enough TaskManagers to run the job, configure `jobmanager.adaptive-scheduler.resource-wait-timeout`.

With Reactive Mode enabled, the [`jobmanager.adaptive-scheduler.resource-stabilization-timeout`]({{< ref "docs/deployment/config">}}#jobmanager-adaptive-scheduler-resource-stabilization-timeout) configuration key will default to `0`: Flink will start running the job, as soon as there are sufficient resources available.
In scenarios where TaskManagers are not connecting at the same time, but slowly one after another, this behavior leads to a job restart whenever a TaskManager connects. Increase this configuration value if you want to wait for the resources to stabilize before scheduling the job.
Additionally, one can configure [`jobmanager.adaptive-scheduler.min-parallelism-increase`]({{< ref "docs/deployment/config">}}#jobmanager-adaptive-scheduler-min-parallelism-increase): This configuration option specifics the minimum amount of additional, aggregate parallelism increase before triggering a scale-up. For example if you have a job with a source (parallelism=2) and a sink (parallelism=2), the aggregate parallelism is 4. By default, the configuration key is set to 1, so any increase in the aggregate parallelism will trigger a restart.

#### Recommendations

- **Configure periodic checkpointing for stateful jobs**: Reactive mode restores from the latest completed checkpoint on a rescale event. If no periodic checkpointing is enabled, your program will lose its state. Checkpointing also configures a **restart strategy**. Reactive Mode will respect the configured restarting strategy: If no restarting strategy is configured, reactive mode will fail your job, instead of scaling it.

- Downscaling in Reactive Mode might take longer if the TaskManager is not properly shutdown (i.e., if a SIGKILL signal is used instead of a SIGTERM signal). In this case, Flink waits for the heartbeat between JobManager and the stopped TaskManager(s) to time out. You will see that your Flink job is stuck for roughly 50 seconds before redeploying your job with a lower parallelism.

  The default timeout is configured to 50 seconds. Adjust the [`heartbeat.timeout`]({{< ref "docs/deployment/config">}}#heartbeat-timeout) configuration to a lower value, if your infrastructure permits this. Setting a low heartbeat timeout can lead to failures if a TaskManager fails to respond to a heartbeat, for example due to a network congestion or a long garbage collection pause. Note that the [`heartbeat.interval`]({{< ref "docs/deployment/config">}}#heartbeat-interval) always needs to be lower than the timeout.


### Limitations

Since Reactive Mode is a new, experimental feature, not all features supported by the default scheduler are also available with Reactive Mode (and its adaptive scheduler). The Flink community is working on addressing these limitations.

- **Deployment is only supported as a standalone application deployment**. Active resource providers (such as native Kubernetes, YARN) are explicitly not supported. Standalone session clusters are not supported either. The application deployment is limited to single job applications. 

  The only supported deployment options are [Standalone in Application Mode]({{< ref "docs/deployment/resource-providers/standalone/overview" >}}#application-mode) ([described](#getting-started) on this page), [Docker in Application Mode]({{< ref "docs/deployment/resource-providers/standalone/docker" >}}#application-mode-on-docker) and [Standalone Kubernetes Application Cluster]({{< ref "docs/deployment/resource-providers/standalone/kubernetes" >}}#deploy-application-cluster).

The [limitations of Adaptive Scheduler](#limitations-1) also apply to Reactive Mode.


## Adaptive Scheduler

{{< hint warning >}}
Using Adaptive Scheduler directly (not through Reactive Mode) is only advised for advanced users because slot allocation on a session cluster with multiple jobs is not defined.
{{< /hint >}}

The Adaptive Scheduler can adjust the parallelism of a job based on available slots. It will automatically reduce the parallelism if not enough slots are available to run the job with the originally configured parallelism; be it due to not enough resources being available at the time of submission, or TaskManager outages during the job execution. If new slots become available the job will be scaled up again, up to the configured parallelism.
In Reactive Mode (see above) the configured parallelism is ignored and treated as if it was set to infinity, letting the job always use as many resources as possible.
You can also use Adaptive Scheduler without Reactive Mode, but there are some practical limitations:
- If you are using Adaptive Scheduler on a session cluster, there are no guarantees regarding the distribution of slots between multiple running jobs in the same session.

One benefit of the Adaptive Scheduler over the default scheduler is that it can handle TaskManager losses gracefully, since it would just scale down in these cases.

### Usage

The following configuration parameters need to be set:

- `jobmanager.scheduler: adaptive`: Change from the default scheduler to adaptive scheduler
- `cluster.declarative-resource-management.enabled` Declarative resource management must be enabled (enabled by default).

The behavior of Adaptive Scheduler is configured by [all configuration options containing `adaptive-scheduler`]({{< ref "docs/deployment/config">}}#advanced-scheduling-options) in their name.

### Limitations

- **Streaming jobs only**: The first version of Adaptive Scheduler runs with streaming jobs only. When submitting a batch job, we will automatically fall back to the default scheduler.
- **No support for [local recovery]({{< ref "docs/ops/state/large_state_tuning">}}#task-local-recovery)**: Local recovery is a feature that schedules tasks to machines so that the state on that machine gets re-used if possible. The lack of this feature means that Adaptive Scheduler will always need to download the entire state from the checkpoint storage.
- **No support for partial failover**: Partial failover means that the scheduler is able to restart parts ("regions" in Flink's internals) of a failed job, instead of the entire job. This limitation impacts only recovery time of embarrassingly parallel jobs: Flink's default scheduler can restart failed parts, while Adaptive Scheduler will restart the entire job.
- **Unused slots**: If the max parallelism for slot sharing groups is not equal, slots offered to Adaptive Scheduler might be unused.
- Scaling events trigger job and task restarts, which will increase the number of Task attempts.

## Adaptive Batch Scheduler

The Adaptive Batch Scheduler is a batch job scheduler that can automatically adjust the execution plan. It currently supports automatically deciding parallelisms of operators for batch jobs. If an operator is not set with a parallelism, the scheduler will decide parallelism for it according to the size of its consumed datasets. This can bring many benefits:
- Batch job users can be relieved from parallelism tuning
- Automatically tuned parallelisms can better fit consumed datasets which have a varying volume size every day
- Operators from SQL batch jobs can be assigned with different parallelisms which are automatically tuned

At present, the Adaptive Batch Scheduler is the default scheduler for Flink batch jobs. No additional configuration is required unless other schedulers are explicitly configured, e.g. `jobmanager.scheduler: default`. Note that you need to
leave the [`execution.batch-shuffle-mode`]({{< ref "docs/deployment/config" >}}#execution-batch-shuffle-mode) unset or explicitly set it to `ALL_EXCHANGES_BLOCKING` (default value) or `ALL_EXCHANGES_HYBRID_FULL` or `ALL_EXCHANGES_HYBRID_SELECTIVE` due to ["BLOCKING or HYBRID jobs only"](#limitations-2).

### Automatically decide parallelisms for operators

#### Usage

To automatically decide parallelisms for operators with Adaptive Batch Scheduler, you need to:
- Toggle the feature on: 
  
    Adaptive Batch Scheduler enables automatic parallelism derivation by default. You can configure [`execution.batch.adaptive.auto-parallelism.enabled`]({{< ref "docs/deployment/config" >}}#execution-batch-adaptive-auto-parallelism-enabled) to toggle this feature. 
In addition, there are several related configuration options that may need adjustment when using Adaptive Batch Scheduler to automatically decide parallelisms for operators:
  - [`execution.batch.adaptive.auto-parallelism.min-parallelism`]({{< ref "docs/deployment/config" >}}#execution-batch-adaptive-auto-parallelism-min-parallelism): The lower bound of allowed parallelism to set adaptively.
  - [`execution.batch.adaptive.auto-parallelism.max-parallelism`]({{< ref "docs/deployment/config" >}}#execution-batch-adaptive-auto-parallelism-max-parallelism): The upper bound of allowed parallelism to set adaptively. The default parallelism set via [`parallelism.default`]({{< ref "docs/deployment/config" >}}) or `StreamExecutionEnvironment#setParallelism()` will be used as upper bound of allowed parallelism if this configuration is not configured.
  - [`execution.batch.adaptive.auto-parallelism.avg-data-volume-per-task`]({{< ref "docs/deployment/config" >}}#execution-batch-adaptive-auto-parallelism-avg-data-volume-per-ta): The average size of data volume to expect each task instance to process. Note that when data skew occurs, or the decided parallelism reaches the max parallelism (due to too much data), the data actually processed by some tasks may far exceed this value.
  - [`execution.batch.adaptive.auto-parallelism.default-source-parallelism`]({{< ref "docs/deployment/config" >}}#execution-batch-adaptive-auto-parallelism-default-source-paralle): The default parallelism of data source.

- Avoid setting the parallelism of operators:

    The Adaptive Batch Scheduler only decides the parallelism for operators which do not have a parallelism set. So if you want the parallelism of an operator to be automatically decided, you need to avoid setting the parallelism for the operator through the 'setParallelism()' method.

    In addition, the following configurations are required for DataSet jobs:
  - Set `parallelism.default: -1`.
  - Don't call `setParallelism()` on `ExecutionEnvironment`.

#### Performance tuning

1. It's recommended to use [Sort Shuffle](https://flink.apache.org/2021/10/26/sort-shuffle-part1.html) and set [`taskmanager.network.memory.buffers-per-channel`]({{< ref "docs/deployment/config" >}}#taskmanager-network-memory-buffers-per-channel) to `0`. This can decouple the required network memory from parallelism, so that for large scale jobs, the "Insufficient number of network buffers" errors are less likely to happen.
2. It's recommended to set [`execution.batch.adaptive.auto-parallelism.max-parallelism`]({{< ref "docs/deployment/config" >}}#execution-batch-adaptive-auto-parallelism-max-parallelism) to the parallelism you expect to need in the worst case. Values larger than that are not recommended, because excessive value may affect the performance. This option can affect the number of subpartitions produced by upstream tasks, large number of subpartitions may degrade the performance of hash shuffle and the performance of network transmission due to small packets.
                                                                                                                                                                                                                                                                                      
### Limitations

- **Batch jobs only**: Adaptive Batch Scheduler only supports batch jobs. Exception will be thrown if a streaming job is submitted.
- **BLOCKING or HYBRID jobs only**: At the moment, Adaptive Batch Scheduler only supports jobs whose [shuffle mode]({{< ref "docs/deployment/config" >}}#execution-batch-shuffle-mode) is `ALL_EXCHANGES_BLOCKING / ALL_EXCHANGES_HYBRID_FULL / ALL_EXCHANGES_HYBRID_SELECTIVE`.
- **FileInputFormat sources are not supported**: FileInputFormat sources are not supported, including `StreamExecutionEnvironment#readFile(...)` `StreamExecutionEnvironment#readTextFile(...)` and `StreamExecutionEnvironment#createInput(FileInputFormat, ...)`. Users should use the new sources([FileSystem DataStream Connector]({{< ref "docs/connectors/datastream/filesystem.md" >}}) or [FileSystem SQL Connector]({{< ref "docs/connectors/table/filesystem.md" >}})) to read files when using the Adaptive Batch Scheduler.
- **Inconsistent broadcast results metrics on WebUI**: When use Adaptive Batch Scheduler to automatically decide parallelisms for operators, for broadcast results, the number of bytes/records sent by the upstream task counted by metric is not equal to the number of bytes/records received by the downstream task, which may confuse users when displayed on the Web UI. See [FLIP-187](https://cwiki.apache.org/confluence/display/FLINK/FLIP-187%3A+Adaptive+Batch+Job+Scheduler) for details.

{{< top >}}
