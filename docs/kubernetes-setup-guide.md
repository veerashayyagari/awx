# AWX on Kubernetes Setup Guide

This guide explains how AWX works when deployed on Kubernetes and covers:
1. **AWX Operator Deployment** — how AWX itself runs on K8s
2. **Container Groups** — ephemeral K8s pods for job execution
3. **Hybrid Setup** — K8s control plane + external execution nodes
4. **Job Execution Flow** — step-by-step walkthrough

---

## Table of Contents

- [Overview: AWX on Kubernetes](#overview-awx-on-kubernetes)
- [Part 1: AWX Operator Deployment](#part-1-awx-operator-deployment)
  - [Architecture](#architecture-awx-operator)
  - [Installation](#installation)
  - [What Gets Created](#what-gets-created)
- [Part 2: Container Groups (K8s-Native Execution)](#part-2-container-groups-k8s-native-execution)
  - [How Container Groups Work](#how-container-groups-work)
  - [Creating a Container Group](#creating-a-container-group)
  - [Pod Customization](#pod-customization)
  - [Job Execution Flow (Container Group)](#job-execution-flow-container-group)
- [Part 3: Hybrid Setup (K8s Control + External Execution)](#part-3-hybrid-setup-k8s-control--external-execution)
  - [Architecture](#architecture-hybrid-k8s)
  - [Connecting External Execution Nodes](#connecting-external-execution-nodes)
- [Key Differences: K8s vs VM Deployment](#key-differences-k8s-vs-vm-deployment)
- [Configuration Reference](#configuration-reference)
- [Troubleshooting](#troubleshooting)

---

## Overview: AWX on Kubernetes

When AWX runs on Kubernetes, it leverages K8s-native features:

| Aspect | VM Deployment | Kubernetes Deployment |
|--------|--------------|----------------------|
| AWX deployment | Systemd services on VMs | Pods managed by Operator |
| Database | External PostgreSQL | StatefulSet or external |
| Scaling control plane | Add VMs manually | Adjust replica count |
| Job execution (default) | Fixed execution nodes | Ephemeral pods (Container Groups) |
| Receptor mesh | TCP between VMs | In-cluster networking or external |
| Certificates | Manual PKI setup | Managed by Operator (Secrets) |

**Key Insight**: In K8s, AWX can run jobs in **ephemeral pods** that exist only for the duration of the job — this is the "Container Group" model.

---

## Part 1: AWX Operator Deployment

### Architecture (AWX Operator)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        KUBERNETES CLUSTER                                   │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         awx NAMESPACE                                 │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    AWX Operator (Deployment)                    │  │  │
│  │  │                                                                 │  │  │
│  │  │  Watches: AWX Custom Resource (CR)                              │  │  │
│  │  │  Creates: Deployments, Services, Secrets, ConfigMaps            │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                        │  │
│  │                              ▼ creates                                │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    awx-web (Deployment)                         │  │  │
│  │  │                                                                 │  │  │
│  │  │  Containers:                                                    │  │  │
│  │  │  ├─ awx-web      (nginx + uwsgi, serves UI + API)               │  │  │
│  │  │  └─ awx-rsyslog  (log aggregation)                              │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    awx-task (Deployment)                        │  │  │
│  │  │                                                                 │  │  │
│  │  │  Containers:                                                    │  │  │
│  │  │  ├─ awx-task     (dispatcher, scheduler, callback receiver)     │  │  │
│  │  │  ├─ awx-ee       (receptor daemon for job execution)            │  │  │
│  │  │  └─ awx-rsyslog  (log aggregation)                              │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    awx-postgres (StatefulSet)                   │  │  │
│  │  │                    OR external PostgreSQL                       │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                       │  │
│  │  ┌─────────────────────┐  ┌─────────────────────────────────────────┐ │  │
│  │  │   awx-redis (Pod)   │  │       Services                         │ │  │
│  │  │   (message broker)  │  │  ├─ awx-service (ClusterIP/NodePort)   │ │  │
│  │  └─────────────────────┘  │  └─ awx-postgres-13 (ClusterIP)        │ │  │
│  │                           └─────────────────────────────────────────┘ │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                        Secrets                                  │  │  │
│  │  │  ├─ awx-admin-password                                          │  │  │
│  │  │  ├─ awx-postgres-configuration                                  │  │  │
│  │  │  ├─ awx-secret-key                                              │  │  │
│  │  │  ├─ awx-receptor-ca     (Receptor TLS CA)                       │  │  │
│  │  │  ├─ awx-receptor-work-signing (work signing keys)               │  │  │
│  │  │  └─ awx-broadcast-websocket                                     │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                 awx-execution NAMESPACE (for Container Groups)        │  │
│  │                                                                       │  │
│  │   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐             │  │
│  │   │automation-job-│  │automation-job-│  │automation-job-│             │  │
│  │   │    123        │  │    124        │  │    125        │  ...        │  │
│  │   │  (ephemeral)  │  │  (ephemeral)  │  │  (ephemeral)  │             │  │
│  │   └───────────────┘  └───────────────┘  └───────────────┘             │  │
│  │                                                                       │  │
│  │   Each pod: Runs ansible-runner for one job, then terminates         │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Installation

#### Prerequisites

```bash
# Kubernetes cluster (minikube, k3s, EKS, GKE, AKS, OpenShift, etc.)
kubectl version

# Cluster admin access
kubectl auth can-i create customresourcedefinitions
```

#### Step 1: Deploy AWX Operator

```bash
# Clone the operator repo
git clone https://github.com/ansible/awx-operator.git
cd awx-operator

# Deploy the operator
export NAMESPACE=awx
make deploy
```

#### Step 2: Create AWX Instance

```yaml
# awx-instance.yaml
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  # Service configuration
  service_type: ClusterIP   # or NodePort, LoadBalancer
  
  # Replicas (control plane scaling)
  replicas: 1
  web_replicas: 1
  task_replicas: 1
  
  # Resource limits
  web_resource_requirements:
    requests:
      cpu: 500m
      memory: 1Gi
  task_resource_requirements:
    requests:
      cpu: 500m
      memory: 2Gi
  
  # PostgreSQL (use built-in or external)
  postgres_configuration_secret: awx-postgres-configuration  # for external
  
  # Container Group default namespace
  container_group_default_namespace: awx-execution
```

```bash
kubectl apply -f awx-instance.yaml

# Watch the deployment
kubectl -n awx get pods -w
```

#### Step 3: Access AWX

```bash
# Get the admin password
kubectl -n awx get secret awx-admin-password -o jsonpath='{.data.password}' | base64 -d

# Port-forward to access UI
kubectl -n awx port-forward svc/awx-service 8080:80

# Access at http://localhost:8080
```

### What Gets Created

| Resource | Name | Purpose |
|----------|------|---------|
| Deployment | `awx-web` | UI + API (nginx + uwsgi) |
| Deployment | `awx-task` | Scheduler, dispatcher, callback receiver, receptor |
| StatefulSet | `awx-postgres-13` | PostgreSQL database (if not external) |
| Pod | `awx-redis` | Redis for caching/messaging |
| Service | `awx-service` | External access to web UI |
| Secret | `awx-receptor-ca` | Receptor TLS certificates |
| Secret | `awx-receptor-work-signing` | Work signing keys |
| ConfigMap | `awx-<name>-config` | AWX settings |

---

## Part 2: Container Groups (K8s-Native Execution)

### How Container Groups Work

**Container Groups** are the K8s-native way to run jobs. Instead of fixed execution nodes:

1. AWX creates an **ephemeral pod** for each job
2. Pod runs the Execution Environment image
3. `ansible-runner` executes inside the pod
4. Output streams back via Receptor
5. Pod is deleted when job completes

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       CONTAINER GROUP EXECUTION                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  TRADITIONAL (VM)                      CONTAINER GROUP (K8s)                │
│  ─────────────────                     ─────────────────────                │
│                                                                             │
│  ┌────────────┐                        ┌────────────┐                       │
│  │ Execution  │  Fixed, always-on      │  awx-task  │  Control plane only   │
│  │   Node     │  runs multiple jobs    │    Pod     │                       │
│  └────────────┘                        └─────┬──────┘                       │
│                                              │                              │
│                                              │ K8s API: create pod          │
│                                              ▼                              │
│                                        ┌────────────┐                       │
│                                        │ Job Pod    │  Created per job      │
│                                        │ automation-│  Runs one playbook    │
│                                        │ job-123    │  Deleted after        │
│                                        └────────────┘                       │
│                                                                             │
│  Pros:                                 Pros:                                │
│  • Faster startup (no pod creation)   • Clean environment every run        │
│  • Consistent capacity                • Automatic scaling                   │
│  • Lower K8s API load                 • No dedicated infrastructure         │
│                                        • Resource limits per job            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Creating a Container Group

#### Option A: In-Cluster (Same K8s Cluster as AWX)

AWX uses its pod's service account to create job pods in the same cluster:

```bash
# Via AWX API
curl -X POST https://awx.example.com/api/v2/instance_groups/ \
  -u admin:password \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "k8s-executors",
    "is_container_group": true
  }'
```

**Note**: No credential needed — AWX uses `kubernetes-incluster-auth` (service account).

#### Option B: External K8s Cluster

To run jobs on a different K8s cluster:

1. **Create a Kubernetes Credential**:

```bash
# Create credential for external cluster
curl -X POST https://awx.example.com/api/v2/credentials/ \
  -u admin:password \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "external-k8s",
    "credential_type": 17,
    "inputs": {
      "host": "https://external-cluster.example.com:6443",
      "bearer_token": "<service-account-token>",
      "verify_ssl": true,
      "ssl_ca_cert": "<CA-certificate-PEM>"
    }
  }'
```

2. **Create Container Group with Credential**:

```bash
curl -X POST https://awx.example.com/api/v2/instance_groups/ \
  -u admin:password \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "external-k8s-executors",
    "is_container_group": true,
    "credential": <credential-id>
  }'
```

### Pod Customization

You can customize the pod spec for Container Groups:

```yaml
# pod_spec_override (applied via API or UI)
apiVersion: v1
kind: Pod
metadata:
  namespace: my-jobs-namespace
spec:
  serviceAccountName: ansible-runner
  automountServiceAccountToken: true
  containers:
    - name: worker
      image: quay.io/ansible/awx-ee:latest
      resources:
        requests:
          cpu: "1"
          memory: 2Gi
        limits:
          cpu: "2"
          memory: 4Gi
      env:
        - name: MY_VAR
          value: "my-value"
  nodeSelector:
    workload: ansible
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "ansible"
      effect: "NoSchedule"
```

Apply via API:

```bash
curl -X PATCH https://awx.example.com/api/v2/instance_groups/<id>/ \
  -u admin:password \
  -H 'Content-Type: application/json' \
  -d '{
    "pod_spec_override": "apiVersion: v1\nkind: Pod\nspec:\n  containers:\n    - name: worker\n      resources:\n        limits:\n          memory: 4Gi"
  }'
```

### Job Execution Flow (Container Group)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   JOB EXECUTION: CONTAINER GROUP                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  awx NAMESPACE                              awx-execution NAMESPACE         │
│  ──────────────                             ─────────────────────           │
│                                                                             │
│  1. POST /api/v2/.../launch/                                                │
│     │                                                                       │
│     ▼                                                                       │
│  2. Job created (pending)                                                   │
│     Instance Group = "k8s-executors" (Container Group)                      │
│     │                                                                       │
│     ▼                                                                       │
│  3. TaskManager.schedule()                                                  │
│     • Sees is_container_group=True                                          │
│     • NO capacity check (pods are elastic)                                  │
│     • status → waiting                                                      │
│     │                                                                       │
│     ▼                                                                       │
│  4. Dispatcher receives task                                                │
│     RunJob.run() → AWXReceptorJob                                           │
│     │                                                                       │
│     ▼                                                                       │
│  5. Determine work_type                                                     │
│     │                                                                       │
│     ├─ has credential? → 'kubernetes-runtime-auth'                          │
│     │                    (use provided K8s credential)                      │
│     │                                                                       │
│     └─ no credential?  → 'kubernetes-incluster-auth'                        │
│                          (use pod's service account)                        │
│     │                                                                       │
│     ▼                                                                       │
│  6. PodManager.create() via K8s API ───────────┐                            │
│     │                                          │                            │
│     │                                          ▼                            │
│     │                              ┌───────────────────────┐                │
│     │                              │  automation-job-123   │                │
│     │                              │  ─────────────────────│                │
│     │                              │  Image: awx-ee:latest │                │
│     │                              │  Command:             │                │
│     │                              │    ansible-runner     │                │
│     │                              │    worker             │                │
│     │                              │    --private-data-dir │                │
│     │                              │    /runner            │                │
│     │                              └───────────┬───────────┘                │
│     │                                          │                            │
│     │          stdout via Receptor             │                            │
│     │ ◄────────────────────────────────────────┘                            │
│     │                                                                       │
│     ▼                                                                       │
│  7. AWXReceptorJob.processor()                                              │
│     Parse job events, queue to callback receiver                            │
│     │                                                                       │
│     ▼                                                                       │
│  8. Callback Receiver persists to PostgreSQL                                │
│     │                                                                       │
│     ▼                                                                       │
│  9. Job completes, pod deleted ────────────────┐                            │
│     status → successful                        │                            │
│                                                ▼                            │
│                                     Pod deleted automatically               │
│                                     (or by cleanup job)                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Code Path** (from [receptor.py](../awx/main/tasks/receptor.py)):

```python
@property
def work_type(self):
    if self.task.instance.is_container_group_task:
        if self.credential:
            return 'kubernetes-runtime-auth'   # External cluster
        return 'kubernetes-incluster-auth'     # Same cluster
    # ... non-container-group logic
```

---

## Part 3: Hybrid Setup (K8s Control + External Execution)

You can run AWX control plane on Kubernetes but execute jobs on external VMs.

### Architecture (Hybrid K8s)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ┌───────────────────────────────────┐       ┌───────────────────────────┐  │
│  │      KUBERNETES CLUSTER           │       │    EXTERNAL VMs           │  │
│  │      (Control Plane)              │       │    (Execution Nodes)      │  │
│  │                                   │       │                           │  │
│  │  ┌─────────────────────────────┐  │       │  ┌─────────────────────┐  │  │
│  │  │         awx-web             │  │       │  │  execution-node-1   │  │  │
│  │  │  (UI + API)                 │  │       │  │                     │  │  │
│  │  └─────────────────────────────┘  │       │  │  ┌───────────────┐  │  │  │
│  │                                   │       │  │  │   Receptor    │  │  │  │
│  │  ┌─────────────────────────────┐  │       │  │  │  (tcp-peer    │  │  │  │
│  │  │         awx-task            │  │       │  │  │   to K8s)     │  │  │  │
│  │  │  • Scheduler                │  │       │  │  └───────┬───────┘  │  │  │
│  │  │  • Dispatcher               │  │       │  │          │         │  │  │
│  │  │  • Callback Receiver        │  │       │  │  ┌───────▼───────┐  │  │  │
│  │  │                             │  │       │  │  │ansible-runner │  │  │  │
│  │  │  ┌───────────────────────┐  │  │       │  │  │    + EE       │  │  │  │
│  │  │  │      Receptor         │  │  │       │  │  └───────────────┘  │  │  │
│  │  │  │  (tcp-listener:2222)  │◄─┼──┼───────┼──┤                     │  │  │
│  │  │  └───────────────────────┘  │  │       │  └─────────────────────┘  │  │
│  │  └─────────────────────────────┘  │       │                           │  │
│  │                                   │       │  ┌─────────────────────┐  │  │
│  │  ┌─────────────────────────────┐  │       │  │  execution-node-2   │  │  │
│  │  │     awx-postgres + redis    │  │       │  │        ...          │  │  │
│  │  └─────────────────────────────┘  │       │  └─────────────────────┘  │  │
│  │                                   │       │                           │  │
│  │  K8s Service: awx-receptor-svc   │       │                           │  │
│  │  (LoadBalancer or NodePort)       │       │                           │  │
│  │  exposes port 2222                │       │                           │  │
│  │                                   │       │                           │  │
│  └───────────────────────────────────┘       └───────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Connecting External Execution Nodes

#### Step 1: Expose Receptor from K8s

The AWX Operator can expose the Receptor service:

```yaml
# AWX CR with external Receptor exposure
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  # ... other config ...
  
  # Expose receptor for external execution nodes
  receptor_log_level: info
```

Or manually create a service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: awx-receptor-external
  namespace: awx
spec:
  type: LoadBalancer  # or NodePort
  selector:
    app.kubernetes.io/component: awx-task
  ports:
    - port: 2222
      targetPort: 2222
      name: receptor
```

#### Step 2: Extract Certificates from K8s Secrets

```bash
# Get the CA certificate
kubectl -n awx get secret awx-receptor-ca -o jsonpath='{.data.tls\.crt}' | base64 -d > ca.crt

# Get work signing public key
kubectl -n awx get secret awx-receptor-work-signing -o jsonpath='{.data.work-public-key\.pem}' | base64 -d > work_public_key.pem
```

#### Step 3: Generate Execution Node Certificate

```bash
# Generate cert request on execution node
receptor --cert-makereq \
  --commonname "executor-node-1" \
  --outreq executor.req \
  --outkey /etc/receptor/certs/executor.key

# Sign with CA from K8s (need ca.key - may need to extract or have operator sign)
# Or use the operator to issue certificates
```

#### Step 4: Configure External Execution Node

```yaml
# /etc/receptor/receptor.conf on execution node
---
- node:
    id: executor-node-1

- tls-client:
    name: mesh-tls
    cert: /etc/receptor/certs/executor.crt
    key: /etc/receptor/certs/executor.key
    rootcas: /etc/receptor/certs/ca.crt

- tcp-peer:
    address: awx-receptor.example.com:2222   # K8s service external IP
    tls: mesh-tls
    redial: true

- work-verification:
    publickey: /etc/receptor/work-signing/work_public_key.pem

- work-command:
    worktype: local
    command: ansible-runner
    params: worker
    allowruntimeparams: true
    verifysignature: true

- log-level: info
```

#### Step 5: Register in AWX

```bash
# Via AWX API or UI, register the execution node
curl -X POST https://awx.example.com/api/v2/instances/ \
  -u admin:password \
  -H 'Content-Type: application/json' \
  -d '{
    "hostname": "executor-node-1",
    "node_type": "execution"
  }'
```

---

## Key Differences: K8s vs VM Deployment

| Aspect | VM Deployment | Kubernetes |
|--------|---------------|------------|
| **Control plane HA** | Multiple VMs + load balancer | Multiple replicas + K8s Service |
| **Database HA** | External PostgreSQL cluster | StatefulSet or external |
| **Execution default** | Fixed execution nodes | Container Groups (ephemeral pods) |
| **Scaling execution** | Add VMs, register with AWX | Automatic (pods created on demand) |
| **Receptor TLS** | Manual PKI setup | Operator manages Secrets |
| **Work signing** | Manual key distribution | Operator manages Secrets |
| **Capacity tracking** | Per-node capacity units | No capacity check for Container Groups |
| **Job isolation** | Execution Environments | Pods (better isolation) |
| **Startup latency** | Low (EE already running) | Higher (pod creation + image pull) |
| **Resource cleanup** | Manual or scheduled | Automatic (pod deleted) |

### When to Use Container Groups vs External Execution Nodes

| Use Case | Recommendation |
|----------|---------------|
| Cloud-native, elastic workloads | Container Groups |
| Need consistent, low-latency execution | External execution nodes |
| Air-gapped environments | External execution nodes |
| Burst capacity for peak loads | Container Groups |
| Specialized hardware (GPU, etc.) | External nodes with specific hardware |
| Multi-tenant isolation | Container Groups (namespace per tenant) |

---

## Configuration Reference

### AWX Settings for K8s

| Setting | Default | Description |
|---------|---------|-------------|
| `IS_K8S` | `True` (when on K8s) | Detected automatically, changes behavior |
| `AWX_CONTAINER_GROUP_DEFAULT_NAMESPACE` | `default` | Namespace for Container Group pods |
| `DEFAULT_EXECUTION_QUEUE_NAME` | `default` | Default instance group name |
| `DEFAULT_CONTROL_PLANE_QUEUE_NAME` | `controlplane` | Instance group for control tasks |

### Work Types in K8s

| Work Type | When Used | Description |
|-----------|-----------|-------------|
| `kubernetes-incluster-auth` | Container Group, no credential | Use pod's service account |
| `kubernetes-runtime-auth` | Container Group with credential | Use provided K8s credential |
| `local` | Hybrid node in K8s | Execute in same pod |
| `ansible-runner` | External execution node | Send to remote receptor |

### Default Pod Spec

From [execution_environments.py](../awx/main/utils/execution_environments.py):

```python
def get_default_pod_spec():
    return {
        "apiVersion": "v1",
        "kind": "Pod",
        "metadata": {
            "namespace": settings.AWX_CONTAINER_GROUP_DEFAULT_NAMESPACE
        },
        "spec": {
            "serviceAccountName": "default",
            "automountServiceAccountToken": False,
            "containers": [{
                "image": "<execution-environment-image>",
                "name": "worker",
                "args": ["ansible-runner", "worker", "--private-data-dir=/runner"],
                "resources": {
                    "requests": {"cpu": "250m", "memory": "100Mi"}
                }
            }]
        }
    }
```

---

## Troubleshooting

### Pod Creation Issues

```bash
# Check if job pod was created
kubectl -n awx-execution get pods -l ansible-awx-job-id=<job-id>

# Check pod events
kubectl -n awx-execution describe pod automation-job-<id>

# Common issues:
# - ImagePullBackOff: Can't pull EE image
# - Pending: No nodes with enough resources
# - CreateContainerError: Security context issues
```

### Container Group Not Working

```bash
# Verify instance group is container group
curl -s https://awx/api/v2/instance_groups/<id>/ | jq '.is_container_group'

# Check if AWX can reach K8s API
kubectl -n awx logs deployment/awx-task -c awx-task | grep -i kubernetes
```

### External Execution Node Not Connecting

```bash
# On AWX K8s pod, check receptor status
kubectl -n awx exec -it deployment/awx-task -c awx-ee -- \
  receptorctl --socket /var/run/receptor/receptor.sock status

# Check if external node is visible
kubectl -n awx exec -it deployment/awx-task -c awx-ee -- \
  receptorctl --socket /var/run/receptor/receptor.sock ping executor-node-1
```

### View Receptor Logs in K8s

```bash
# Receptor logs from awx-task pod
kubectl -n awx logs deployment/awx-task -c awx-ee -f
```

### Job Stuck in Pending/Waiting

```bash
# Check scheduler logs
kubectl -n awx logs deployment/awx-task -c awx-task | grep -i "task_manager\|schedule"

# Verify instance group assignment
curl -s https://awx/api/v2/jobs/<id>/ | jq '.instance_group, .execution_node'
```

---

## Quick Reference

```bash
# ─────────────────────────────────────────────────────────────────
# AWX Operator Commands
# ─────────────────────────────────────────────────────────────────
kubectl apply -k awx-operator/                    # Deploy operator
kubectl -n awx get awx                            # List AWX instances
kubectl -n awx describe awx awx                   # Instance details

# ─────────────────────────────────────────────────────────────────
# Pods and Logs
# ─────────────────────────────────────────────────────────────────
kubectl -n awx get pods                           # AWX pods
kubectl -n awx-execution get pods                 # Job pods
kubectl -n awx logs deploy/awx-task -c awx-task   # Task logs
kubectl -n awx logs deploy/awx-task -c awx-ee     # Receptor logs

# ─────────────────────────────────────────────────────────────────
# Secrets (Certificates)
# ─────────────────────────────────────────────────────────────────
kubectl -n awx get secrets | grep receptor
kubectl -n awx get secret awx-receptor-ca -o yaml
kubectl -n awx get secret awx-receptor-work-signing -o yaml

# ─────────────────────────────────────────────────────────────────
# Debug
# ─────────────────────────────────────────────────────────────────
kubectl -n awx exec -it deploy/awx-task -c awx-task -- awx-manage shell
kubectl -n awx exec -it deploy/awx-task -c awx-ee -- receptorctl status
```

---

## See Also

- [Cluster Setup Guide](cluster-setup-guide.md) — VM-based deployment
- [Architecture Overview](architecture-overview.md) — AWX internals
- [Container Groups](container_groups.md) — detailed Container Group docs
- [Execution Environments](execution_environments.md) — custom EE images
