<!--
SPDX-FileCopyrightText: Copyright (c) 2024-2025 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Planner

The planner is a component that monitors the state of the system and makes adjustments to the number of workers to ensure that the system is running efficiently. It can dynamically scale prefill/decode workers up and down based on a variety of KV metrics. You can find documentation and benchmarking examples in the [planner docs](../../docs/planner.md).

## Usage

After you've deployed a dynamo graph, you can start the planner with the following command:

```bash
python components/planner.py --namespace <namespace>
```

## Backends

1. `local` - uses circus to start/stop worker subprocesses
2. `kubernetes` - uses the kubernetes API to adjust replicas of each component's resource definition. This is a work in progress and not currently available

## Local Backend (LocalPlanner)

The LocalPlanner is built on top of circus, which is what we use to manage component subprocesses when running dynamo serve. LocalPlanner allows the planner component to scale workers up and down based on system metrics.

**Current limitations**
1. Single node only
2. Workers must be using only a single GPU
3. Your initial deployment must be replicas=1 for both prefill and decode

We are working on addressing these as fast as possible.

### Under the Hood

Circus has a concept of an arbiter and a watcher:
- **Arbiter**: The supervisor process that manages all watchers
- **Watcher**: A process that encodes environment variables, command, name, and other information needed to run a component

When a service is started, each worker process is spun up as a watcher. For example, when starting a VllmWorker, a watcher is created that looks like:

```json
{
  "dynamo_VllmWorker": {
    "watcher_name": "dynamo_VllmWorker",
    "cmd": "/opt/dynamo/venv/bin/python3 -m dynamo.sdk.cli.serve_dynamo graphs.agg_router:Frontend --service-name VllmWorker --worker-id $(CIRCUS.WID) --worker-env [{\"CUDA_VISIBLE_DEVICES\": \"0\"}]",
    "resources": {
      "allocated_gpus": [
        0
      ]
    },
    "lease": 7587886183172559418
  }
}
```

The arbiter exposes an endpoint allowing messages to add/remove/change watchers. The LocalPlanner leverages this functionality to dynamically adjust worker counts.

### Implementation

The planner architecture is designed to be simple and extensible:
- An abstract class supports basic add/remove component operations
- This is implemented in `local_connector.py`
- Circus interaction logic is in `circusd.py`, which reads the statefile, connects to the endpoint, and provides add/remove functionality
- Planner starts an instance of `LocalConnector` and uses it to modify the deployment topology

### Statefile

The statefile maintains the current state of all running workers and is used by the LocalPlanner to track and modify the deployment. It's stored at `~/.dynamo/state/{namespace}.json` (or in the directory specified by `DYN_LOCAL_STATE_DIR`). The statefile is automatically created when you run dynamo serve and is cleaned up when the arbiter terminates. Each worker is identified as `{namespace}_{component_name}` with an optional numeric suffix for additional instances.

#### Example: Adding and Removing Workers

Starting with a single decode worker:
```json
{
  "dynamo_VllmWorker": {..., "resources":{...}}
}
```

After adding a worker:
```json
{
  "dynamo_VllmWorker": {..., "resources":{...}},
  "dynamo_VllmWorker_1": {..., "resources":{...}}
}
```

After removing a worker (removes the highest suffix):
```json
{
  "dynamo_VllmWorker": {..., "resources":{...}}
}
```

If scaled to zero, the initial entry is kept without resources to maintain configuration information:
```json
{
  "dynamo_VllmWorker": {...}
}
```

### Looking forward

- Support for a multinode LocalPlanner
- Storing the statefile (and initial configurations) in ETCD using the the new `EtcdKvCache`.

### Testing

For manual testing, you can use the controller_test.py file to add/remove components after you've run a serve command with `--enable-local-planner`.

## Kubernetes Backend

[Coming soon]


