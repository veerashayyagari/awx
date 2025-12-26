# Instances & Capacity

This document details AWX's instance management, capacity system, and how instances are created in different deployment scenarios.

## What to take away

- Instances are database records; pods/containers are the runtime backing them.
- Capacity is derived from CPU/memory calculations and is easy to misconfigure.
- Container Groups are special instance groups with no backing Instance rows.

## Key Files

| File | Purpose |
|------|---------|
| `awx/main/models/ha.py` | Instance, InstanceGroup, InstanceLink models |
| `awx/main/utils/common.py` | Capacity calculation functions |
| `awx/main/scheduler/task_manager_models.py` | TaskManagerInstance, TaskManagerInstanceGroups |
| `awx/main/scheduler/kubernetes.py` | PodManager for Container Groups |
| `awx/main/management/commands/provision_instance.py` | Instance registration |
| `awx/main/tasks/receptor.py` | Receptor mesh integration |
| `tools/docker-compose/bootstrap_development.sh` | Docker Compose instance setup |

## Understanding Instances: The Key Distinction

**Critical Concept**: An Instance is a **database record** that represents an AWX node. The actual container/pod/process is separate from the Instance record.

| Deployment | Instance Record | Actual Execution Environment |
|------------|-----------------|------------------------------|
| Docker Compose | Created via `provision_instance` at startup | Pre-existing containers defined in docker-compose.yml |
| K8s (static) | Created by AWX Operator | Pods in the AWX deployment |
| K8s (Container Group) | **No Instance record** | Ephemeral pods created per-job by Receptor |

## Quick Debug Checklist

If a job will not start or stays pending:
- Verify the job's `instance_group` and whether it has eligible instances.
- Check `Instance.enabled` and `Instance.capacity` values.
- If using K8s, confirm resource limits are set so capacity is accurate.
- For container groups, verify `pod_spec_override` and credentials are valid.

## Docker Compose: How Instances Work

### Container Lifecycle

Containers are **pre-created** by docker-compose before any Instance records exist:

**File**: `tools/docker-compose/ansible/roles/sources/templates/docker-compose.yml.j2`

```yaml
# Control plane nodes (awx_1, awx_2, etc.)
{% for i in range(control_plane_node_count|int) %}
  awx_{{ loop.index }}:
    image: "{{ awx_image }}:{{ awx_image_tag }}"
    hostname: awx_{{ loop.index }}
    command: launch_awx.sh
    environment:
      MAIN_NODE_TYPE: "${MAIN_NODE_TYPE:-hybrid}"
      EXECUTION_NODE_COUNT: {{ execution_node_count|int }}
    # ... volumes, ports, etc.
{% endfor %}

# Execution nodes (receptor-1, receptor-2, etc.) - only if EXECUTION_NODE_COUNT > 0
{% if execution_node_count|int > 0 %}
  receptor-hop:
    image: {{ receptor_image }}
    hostname: receptor-hop
    command: 'receptor --config /etc/receptor/receptor.conf'

  {% for i in range(execution_node_count|int) %}
  receptor-{{ loop.index }}:
    image: "{{ awx_image }}:{{ awx_image_tag }}"
    hostname: receptor-{{ loop.index }}
    command: 'receptor --config /etc/receptor/receptor.conf'
  {% endfor %}
{% endif %}
```

### Instance Registration at Bootstrap

**After** containers start, the bootstrap script registers them as Instance records:

**File**: `tools/docker-compose/bootstrap_development.sh`

```bash
# Register the main AWX node (runs inside awx_1 container)
awx-manage provision_instance --hostname="$(hostname)" --node_type="$MAIN_NODE_TYPE"

# Register instance groups
awx-manage register_queue --queuename=controlplane --instance_percent=100
awx-manage register_queue --queuename=default --instance_percent=100

# If execution nodes are configured, register them too
if [[ $EXECUTION_NODE_COUNT > 0 ]]; then
    awx-manage provision_instance --hostname="receptor-hop" --node_type="hop"
    awx-manage register_peers "receptor-hop" --peers "awx_1"

    for (( e=1; e<=$EXECUTION_NODE_COUNT; e++ )); do
        awx-manage provision_instance --hostname="receptor-$e" --node_type="execution"
        awx-manage register_peers "receptor-$e" --peers "receptor-hop"
    done
fi
```

### Docker Compose Topology

```
CONTROL_PLANE_NODE_COUNT=2 EXECUTION_NODE_COUNT=3 make docker-compose

┌──────────────┐                 ┌──────────────┐
│    awx_1     │◄───────────────►│    awx_2     │  (hybrid nodes)
│   (hybrid)   │                 │   (hybrid)   │
└──────┬───────┘                 └──────────────┘
       │
       │  Receptor mesh
       ▼
┌──────────────┐                 ┌──────────────┐
│ receptor-hop │◄───────────────►│  receptor-1  │
│    (hop)     │                 │ (execution)  │
└──────┬───────┘                 └──────────────┘
       │
       ├────────────────────────►┌──────────────┐
       │                         │  receptor-2  │
       │                         │ (execution)  │
       │                         └──────────────┘
       │
       └────────────────────────►┌──────────────┐
                                 │  receptor-3  │
                                 │ (execution)  │
                                 └──────────────┘
```

### Docker Compose Capacity Problem

**The containers have NO resource limits by default**, and ansible-runner reports **host** resources:

```python
# From ansible_runner/utils/capacity.py
def get_cpu_count():
    return multiprocessing.cpu_count()  # Returns HOST CPUs, not container!

def get_mem_in_bytes():
    with open('/proc/meminfo') as f:  # Returns HOST memory!
        ...
```

**Result**: A container on a 64-CPU, 256GB host reports capacity for 64 CPUs even if you wanted to limit it.

### Docker Compose Capacity Fix

Set environment variables in docker-compose or your shell:

```bash
# Option 1: Environment variables when starting
SYSTEM_TASK_ABS_CPU=4 SYSTEM_TASK_ABS_MEM=8Gi make docker-compose

# Option 2: Add to docker-compose.yml.j2
environment:
  SYSTEM_TASK_ABS_CPU: "4"
  SYSTEM_TASK_ABS_MEM: "8Gi"
```

## Kubernetes with AWX Operator: How Instances Work

### AWX Operator Pod Lifecycle

The AWX Operator creates **static pods** for AWX components:

1. **awx-task pod**: Runs scheduler, dispatcher, callback receiver
2. **awx-web pod**: Runs API and web UI
3. **awx-ee pod**: Control plane execution environment

These are created by the Operator when you apply an AWX CR:

```yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  replicas: 1
```

### Instance Auto-Registration in K8s

In Kubernetes, instances register **themselves** on startup:

**File**: `awx/main/management/commands/provision_instance.py:31-42`

```python
def _register_hostname(self, hostname, node_type, uuid):
    if not hostname:
        # K8s mode: auto-register from pod info
        if not settings.AWX_AUTO_DEPROVISION_INSTANCES:
            raise CommandError('...')

        # Register using pod IP and settings
        (changed, instance) = Instance.objects.register(
            ip_address=os.environ.get('MY_POD_IP'),
            node_type='control',
            uuid=settings.SYSTEM_UUID
        )

        # Also create default instance groups
        RegisterQueue(settings.DEFAULT_CONTROL_PLANE_QUEUE_NAME, ...).register()
        RegisterQueue(settings.DEFAULT_EXECUTION_QUEUE_NAME, ...,
                     is_container_group=True).register()  # Container Group!
```

### K8s Capacity Configuration

The AWX Operator automatically sets capacity based on **resource limits**:

**File**: `roles/installer/templates/configmaps/config.yaml.j2` (AWX Operator)

```yaml
{% if task_resource_requirements["limits"]["memory"] is defined %}
SYSTEM_TASK_ABS_MEM = '{{ task_resource_requirements["limits"]["memory"] }}'
{% endif %}

{% if task_resource_requirements["limits"]["cpu"] is defined %}
SYSTEM_TASK_ABS_CPU = '{{ task_resource_requirements["limits"]["cpu"] }}'
{% endif %}
```

**CRITICAL**: The default configuration only sets **requests**, not **limits**:

```yaml
# Default from roles/installer/defaults/main.yml
task_resource_requirements:
  requests:
    cpu: 100m
    memory: 128Mi
  # NO LIMITS! Capacity will be wrong!
```

### Recommended K8s Configuration

```yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  task_resource_requirements:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m      # Sets SYSTEM_TASK_ABS_CPU
      memory: 4Gi     # Sets SYSTEM_TASK_ABS_MEM

  ee_resource_requirements:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi
```

## Container Groups: Dynamic Pod Creation

Container Groups are a special type of InstanceGroup that creates **ephemeral pods per job**.

### How Container Groups Differ

| Regular InstanceGroup | Container Group |
|----------------------|-----------------|
| Has static Instance members | `is_container_group=True`, no instances |
| Jobs run on existing nodes | Pods created on-demand for each job |
| Capacity tracked per instance | No capacity tracking (unlimited by default) |
| Receptor routes to execution node | Receptor creates pod via K8s API |

### Container Group Job Flow

**File**: `awx/main/scheduler/task_manager.py:543-547`

```python
for instance_group in preferred_instance_groups:
    if instance_group.is_container_group:
        # No instance selection! Just start the task
        self.dependency_graph.add_job(task)
        self.start_task(task, instance_group, task.get_jobs_fail_chain(), None)  # None = no instance
        found_acceptable_queue = True
        break
```

### Pod Creation via Receptor

When a job runs on a Container Group, Receptor creates the pod:

**File**: `awx/main/tasks/receptor.py:408-441`

```python
@property
def work_type(self):
    if self.task.instance.is_container_group_task:
        if self.credential:
            return 'kubernetes-runtime-auth'  # Use provided K8s credential
        return 'kubernetes-incluster-auth'    # Use in-cluster service account
    # ... regular execution node handling

@property
def receptor_params(self):
    if self.task.instance.is_container_group_task:
        spec_yaml = yaml.dump(self.pod_definition, explicit_start=True)
        receptor_params = {
            "secret_kube_pod": spec_yaml,  # Pod spec for Receptor to create
            "pod_pending_timeout": "5m",
        }
        if self.credential:
            kubeconfig_yaml = yaml.dump(self.kube_config, explicit_start=True)
            receptor_params["secret_kube_config"] = kubeconfig_yaml
        return receptor_params
```

### Default Pod Spec for Container Groups

**File**: `awx/main/utils/execution_environments.py:24-45`

```python
def get_default_pod_spec():
    ee = get_default_execution_environment()
    return {
        "apiVersion": "v1",
        "kind": "Pod",
        "metadata": {"namespace": settings.AWX_CONTAINER_GROUP_DEFAULT_NAMESPACE},
        "spec": {
            "serviceAccountName": "default",
            "automountServiceAccountToken": False,
            "containers": [{
                "image": ee.image,
                "name": 'worker',
                "args": ['ansible-runner', 'worker', '--private-data-dir=/runner'],
                "resources": {"requests": {"cpu": "250m", "memory": "100Mi"}},
            }],
        },
    }
```

### Customizing Container Group Pod Spec

You can override the pod spec via `pod_spec_override` on the InstanceGroup:

```yaml
# Stored in InstanceGroup.pod_spec_override
spec:
  containers:
    - resources:
        requests:
          cpu: "1"
          memory: "2Gi"
        limits:
          cpu: "2"
          memory: "4Gi"
```

**File**: `awx/main/tasks/receptor.py:461-468`

```python
@property
def pod_definition(self):
    default_pod_spec = get_default_pod_spec()

    pod_spec_override = {}
    if self.task and self.task.instance.instance_group.pod_spec_override:
        pod_spec_override = parse_yaml_or_json(self.task.instance.instance_group.pod_spec_override)

    # Override merges with defaults
    pod_spec = deepmerge(default_pod_spec, pod_spec_override)
```

## Node Types

| Type | Description | Can Control | Can Execute |
|------|-------------|-------------|-------------|
| `control` | Runs API, scheduler, web UI | Yes | No |
| `execution` | Runs Ansible playbooks | No | Yes |
| `hybrid` | Does both (default for dev) | Yes | Yes |
| `hop` | Receptor relay only | No | No |

## Capacity Calculation

### CPU Capacity Formula

**File**: `awx/main/utils/common.py:741-756`

```python
def get_cpu_effective_capacity(cpu_count):
    cpu_count = get_corrected_cpu(cpu_count)  # Apply SYSTEM_TASK_ABS_CPU override

    forkcpu = os.getenv('SYSTEM_TASK_FORKS_CPU') or settings.SYSTEM_TASK_FORKS_CPU or 4

    return max(1, int(cpu_count * forkcpu))
```

Example: `2 CPUs × 4 forks/CPU = 8 CPU capacity`

### Memory Capacity Formula

**File**: `awx/main/utils/common.py:815-844`

```python
def get_mem_effective_capacity(mem_bytes):
    mem_bytes = get_corrected_memory(mem_bytes)  # Apply SYSTEM_TASK_ABS_MEM override

    mem_mb_per_fork = os.getenv('SYSTEM_TASK_FORKS_MEM') or settings.SYSTEM_TASK_FORKS_MEM or 100

    # Deduct 2GB for system processes (non-K8s only)
    memory_penalty_bytes = 2147483648  # 2GB
    if settings.IS_K8S:
        memory_penalty_bytes = 0  # K8s containers have dedicated memory

    mem_mb = (mem_bytes - memory_penalty_bytes) // (1024 * 1024)
    return max(1, mem_mb // mem_mb_per_fork)
```

Example (K8s): `4096 MB / 100 MB per fork = 40 memory capacity`

### Final Capacity

**File**: `awx/main/models/ha.py:225-232`

```python
def set_capacity_value(self):
    if self.enabled and self.node_type != 'hop':
        lower_cap = min(self.mem_capacity, self.cpu_capacity)
        higher_cap = max(self.mem_capacity, self.cpu_capacity)
        self.capacity = lower_cap + (higher_cap - lower_cap) * self.capacity_adjustment
    else:
        self.capacity = 0
```

With `capacity_adjustment=1.0` (default):
- CPU capacity: 8
- Memory capacity: 40
- Final: `8 + (40 - 8) × 1.0 = 40`

## Configuration Reference

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `SYSTEM_TASK_ABS_CPU` | auto-detect | Override detected CPU count (e.g., "2", "500m") |
| `SYSTEM_TASK_ABS_MEM` | auto-detect | Override detected memory (e.g., "4Gi", "4096Mi") |
| `SYSTEM_TASK_FORKS_CPU` | 4 | Forks per CPU core |
| `SYSTEM_TASK_FORKS_MEM` | 100 | MB per fork |

### Kubernetes Resource Formats

CPU values:
- `"2"` = 2 cores
- `"1.5"` = 1.5 cores
- `"500m"` = 0.5 cores (500 millicores)

Memory values:
- `"4Gi"` = 4 gibibytes (4 × 2^30 bytes)
- `"4G"` = 4 gigabytes (4 × 10^9 bytes)
- `"4096Mi"` = 4096 mebibytes

### AWX Operator Sample Configuration

```yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  task_resource_requirements:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi

  web_resource_requirements:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 1000m
      memory: 2Gi

  ee_resource_requirements:
    requests:
      cpu: 500m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi

  redis_resource_requirements:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 500m
      memory: 256Mi
```

## Summary: Instance Creation by Deployment Type

| Scenario | Who Creates Containers/Pods | Who Creates Instance Records | Capacity Source |
|----------|----------------------------|------------------------------|-----------------|
| Docker Compose | docker-compose.yml (pre-created) | `bootstrap_development.sh` | Host resources (wrong without override) |
| K8s (Operator) | AWX Operator (static pods) | AWX pod on startup | Resource limits from AWX CR |
| Container Group | Receptor (per-job ephemeral pods) | **No Instance created** | Pod spec (per-job) |

## Documentation References

- [AWX Operator - Container Resource Requirements](https://github.com/ansible/awx-operator/blob/devel/docs/user-guide/advanced-configuration/containers-resource-requirements.md)
- [AWX Operator - Exporting Environment Variables](https://github.com/ansible/awx-operator/blob/devel/docs/user-guide/advanced-configuration/exporting-environment-variables-to-containers.md)
- [AWX Capacity Documentation](https://github.com/ansible/awx/blob/devel/docs/capacity.md)
- [PR #11725 - K8s resource format support](https://github.com/ansible/awx/pull/11725)
