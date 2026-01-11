# AWX Cluster Setup Guide

This guide explains how to set up AWX in two configurations and how job execution works in each:
1. **Single VM** (Hybrid Node) — simplest setup, everything on one machine
2. **Multi-VM** (Control + Execution Nodes) — scalable production setup

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Part 1: Single VM Setup (Hybrid Node)](#part-1-single-vm-setup-hybrid-node)
  - [Architecture](#architecture-single-vm)
  - [Installation](#installation-single-vm)
  - [Job Execution Flow](#job-execution-flow-single-vm)
- [Part 2: Multi-VM Setup (Control + Execution)](#part-2-multi-vm-setup-control--execution)
  - [Architecture](#architecture-multi-vm)
  - [Step 1: Set Up Control Node](#step-1-set-up-control-node)
  - [Step 2: Set Up Execution Node](#step-2-set-up-execution-node)
  - [Step 3: Configure Receptor TLS (Recommended)](#step-3-configure-receptor-tls-recommended)
  - [Step 4: Register Nodes in AWX](#step-4-register-nodes-in-awx)
  - [Job Execution Flow](#job-execution-flow-multi-vm)
- [Receptor Security Model](#receptor-security-model)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Component | Purpose | Required On |
|-----------|---------|-------------|
| **Python 3.9+** | AWX runtime | Control/Hybrid nodes |
| **PostgreSQL 13+** | Database | Control node (or external) |
| **Redis 6+** | Message broker, caching | Control node |
| **Receptor** | Mesh networking | All nodes |
| **ansible-runner** | Executes Ansible | Execution/Hybrid nodes |
| **Podman or Docker** | Run Execution Environments | Execution/Hybrid nodes |

---

## Part 1: Single VM Setup (Hybrid Node)

### Architecture (Single VM)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SINGLE VM (Hybrid Node)                      │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │   Web UI    │  │  REST API   │  │  Scheduler  │                 │
│  │   (React)   │  │   (Django)  │  │(TaskManager)│                 │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                 │
│         │                │                │                         │
│         └────────────────┴────────────────┘                         │
│                          │                                          │
│                          ▼                                          │
│                   ┌─────────────┐                                   │
│                   │ Dispatcher  │                                   │
│                   │  (Kombu)    │                                   │
│                   └──────┬──────┘                                   │
│                          │                                          │
│                          ▼                                          │
│                   ┌─────────────┐                                   │
│                   │  Receptor   │ ◄─── Unix Socket                  │
│                   │  (Go mesh)  │      /var/run/receptor/receptor.sock
│                   └──────┬──────┘                                   │
│                          │                                          │
│                          ▼                                          │
│                   ┌─────────────┐                                   │
│                   │ansible-runner                                   │
│                   │    + EE     │ ◄─── Execution Environment        │
│                   └─────────────┘      (Container Image)            │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐                                   │
│  │ PostgreSQL  │  │    Redis    │                                   │
│  └─────────────┘  └─────────────┘                                   │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Point**: In hybrid mode, the Receptor work unit stays **local** — no network hops.

### Installation (Single VM)

The easiest path is using the AWX Operator on a single-node Kubernetes (minikube/k3s) or Docker Compose for development:

#### Option A: Docker Compose (Development)

```bash
# Clone AWX
git clone https://github.com/ansible/awx.git
cd awx

# Build and run
make docker-compose-build
make docker-compose

# Access at https://localhost:8043
# Default credentials: admin / password
```

#### Option B: AWX Operator (Production-like)

```bash
# Install minikube or k3s first, then:
kubectl apply -k awx-operator/

# Create AWX instance
cat <<EOF | kubectl apply -f -
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
EOF
```

#### Register as Hybrid Node

```bash
# Inside the AWX container/pod:
awx-manage provision_instance --hostname=$(hostname) --node_type=hybrid
```

### Job Execution Flow (Single VM)

```
┌─────────────────────────────────────────────────────────────────────┐
│                   JOB EXECUTION: SINGLE VM                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. USER LAUNCHES JOB                                               │
│     ─────────────────                                               │
│     POST /api/v2/job_templates/5/launch/                            │
│           │                                                         │
│           ▼                                                         │
│  2. JOB CREATED (status=pending)                                    │
│     ─────────────────────────────                                   │
│     Django creates UnifiedJob record in PostgreSQL                  │
│     Sends NOTIFY via pg_notify('new_job')                           │
│           │                                                         │
│           ▼                                                         │
│  3. TASK MANAGER SCHEDULES                                          │
│     ─────────────────────────                                       │
│     TaskManager.schedule() runs (every 20s or on NOTIFY)            │
│     • Acquires advisory lock (pg_try_advisory_lock)                 │
│     • Finds pending job                                             │
│     • Checks: dependencies met? capacity available?                 │
│     • Assigns job to THIS node (we're hybrid)                       │
│     • status → waiting                                              │
│           │                                                         │
│           ▼                                                         │
│  4. DISPATCHER PICKS UP                                             │
│     ───────────────────                                             │
│     Kombu consumer receives task message                            │
│     Calls RunJob.run() in awx/main/tasks/jobs.py                    │
│           │                                                         │
│           ▼                                                         │
│  5. RECEPTOR WORK UNIT (LOCAL)                                      │
│     ──────────────────────────                                      │
│     Python → Unix socket → Receptor daemon                          │
│                                                                     │
│     receptor work submit \                                          │
│       --node awx_1 \              # Local node                      │
│       --worktype local \          # Run locally                     │
│       --signwork \                # Sign with private key           │
│       -- ansible-runner worker    # The actual command              │
│           │                                                         │
│           ▼                                                         │
│  6. ANSIBLE-RUNNER EXECUTES                                         │
│     ────────────────────────                                        │
│     Receptor spawns: ansible-runner worker                          │
│     • Pulls Execution Environment image (if needed)                 │
│     • Mounts project, inventory, credentials                        │
│     • Runs ansible-playbook inside container                        │
│     • Streams stdout to Receptor                                    │
│           │                                                         │
│           ▼                                                         │
│  7. OUTPUT STREAMS BACK                                             │
│     ─────────────────────                                           │
│     Receptor → Unix socket → Python (AWXReceptorJob.processor)      │
│     Job events parsed and queued                                    │
│           │                                                         │
│           ▼                                                         │
│  8. CALLBACK RECEIVER PERSISTS                                      │
│     ──────────────────────────                                      │
│     run_callback_receiver process reads event queue                 │
│     Bulk-inserts JobEvent records to PostgreSQL                     │
│           │                                                         │
│           ▼                                                         │
│  9. JOB COMPLETES                                                   │
│     ──────────────                                                  │
│     status → successful | failed | error                            │
│     WebSocket notification sent to UI                               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Timeline Example:**
```
T+0.0s   POST /launch/ received
T+0.1s   Job created, status=pending
T+0.2s   pg_notify('new_job')
T+0.3s   TaskManager wakes, acquires lock
T+0.5s   Job assigned, status=waiting
T+0.6s   Dispatcher receives task
T+0.8s   Receptor work submitted (local)
T+1.0s   ansible-runner starts
T+1.5s   Playbook execution begins
T+45.0s  Playbook completes
T+45.5s  Events flushed to DB
T+45.6s  status=successful, WebSocket update
```

---

## Part 2: Multi-VM Setup (Control + Execution)

### Architecture (Multi-VM)

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  ┌─────────────────────────────────┐    ┌─────────────────────────┐ │
│  │         CONTROL NODE            │    │     EXECUTION NODE      │ │
│  │         (VM #1)                 │    │        (VM #2)          │ │
│  │                                 │    │                         │ │
│  │  ┌─────────┐ ┌─────────┐        │    │                         │ │
│  │  │ Web UI  │ │  API    │        │    │                         │ │
│  │  └────┬────┘ └────┬────┘        │    │                         │ │
│  │       └─────┬─────┘             │    │                         │ │
│  │             ▼                   │    │                         │ │
│  │  ┌──────────────────┐           │    │                         │ │
│  │  │   Task Manager   │           │    │                         │ │
│  │  │   (Scheduler)    │           │    │                         │ │
│  │  └────────┬─────────┘           │    │                         │ │
│  │           ▼                     │    │                         │ │
│  │  ┌──────────────────┐           │    │                         │ │
│  │  │    Dispatcher    │           │    │                         │ │
│  │  └────────┬─────────┘           │    │                         │ │
│  │           ▼                     │    │                         │ │
│  │  ┌──────────────────┐           │    │  ┌──────────────────┐   │ │
│  │  │    Receptor      │◄──────────┼────┼──│    Receptor      │   │ │
│  │  │  (tcp-listener   │  TLS over │    │  │  (tcp-peer to    │   │ │
│  │  │   port 2222)     │  TCP      │    │  │   control:2222)  │   │ │
│  │  └──────────────────┘           │    │  └────────┬─────────┘   │ │
│  │                                 │    │           ▼             │ │
│  │  ┌──────────┐ ┌──────────┐      │    │  ┌──────────────────┐   │ │
│  │  │PostgreSQL│ │  Redis   │      │    │  │  ansible-runner  │   │ │
│  │  └──────────┘ └──────────┘      │    │  │   + EE Container │   │ │
│  │                                 │    │  └──────────────────┘   │ │
│  └─────────────────────────────────┘    └─────────────────────────┘ │
│                                                                     │
│         Network: Control ◄──── Executor connects OUT                │
│                  Port 2222 inbound open on Control                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 1: Set Up Control Node

#### 1.1 Install AWX (Control-only)

```bash
# Using AWX Operator or from source
# The node runs: web, API, scheduler, dispatcher, callback receiver
# It does NOT run jobs locally
```

#### 1.2 Install Receptor

```bash
# Install receptor binary
dnf install receptor   # RHEL/Fedora
# or download from https://github.com/ansible/receptor/releases
```

#### 1.3 Generate PKI (Certificates + Work Signing Keys)

```bash
# Create directory structure
mkdir -p /etc/receptor/certs /etc/receptor/work-signing

# ─────────────────────────────────────────────────────────────────
# STEP A: Create Private CA (do this ONCE, use for all nodes)
# ─────────────────────────────────────────────────────────────────
receptor --cert-init ca \
  --commonname "AWX Receptor CA" \
  --bits 2048 \
  --outcert /etc/receptor/certs/ca.crt \
  --outkey /etc/receptor/certs/ca.key

# ─────────────────────────────────────────────────────────────────
# STEP B: Generate Control Node Certificate
# ─────────────────────────────────────────────────────────────────
receptor --cert-makereq \
  --commonname "control-node" \
  --dnsname control.example.com \
  --outreq /etc/receptor/certs/control.req \
  --outkey /etc/receptor/certs/control.key

receptor --cert-signreq \
  --cacert /etc/receptor/certs/ca.crt \
  --cakey /etc/receptor/certs/ca.key \
  --req /etc/receptor/certs/control.req \
  --outcert /etc/receptor/certs/control.crt

# ─────────────────────────────────────────────────────────────────
# STEP C: Generate Work Signing Keys (do this ONCE)
# ─────────────────────────────────────────────────────────────────
openssl genrsa -out /etc/receptor/work-signing/work_private_key.pem 2048
openssl rsa -in /etc/receptor/work-signing/work_private_key.pem \
  -pubout -out /etc/receptor/work-signing/work_public_key.pem

# ─────────────────────────────────────────────────────────────────
# Files you now have:
# ─────────────────────────────────────────────────────────────────
# /etc/receptor/certs/
#   ├── ca.crt              ← Copy to ALL nodes
#   ├── ca.key              ← Keep secure, only for signing new certs
#   ├── control.crt         ← This node's certificate
#   └── control.key         ← This node's private key
#
# /etc/receptor/work-signing/
#   ├── work_private_key.pem ← CONTROL NODE ONLY (signs work)
#   └── work_public_key.pem  ← Copy to ALL execution nodes
```

#### 1.4 Configure Receptor (Control Node)

Create `/etc/receptor/receptor.conf`:

```yaml
---
# ─────────────────────────────────────────────────────────────────
# Node Identity
# ─────────────────────────────────────────────────────────────────
- node:
    id: control-node

# ─────────────────────────────────────────────────────────────────
# TLS Configuration (Server Side)
# ─────────────────────────────────────────────────────────────────
- tls-server:
    name: mesh-tls
    cert: /etc/receptor/certs/control.crt
    key: /etc/receptor/certs/control.key
    requireclientcert: true          # Mutual TLS - executor must authenticate
    clientcas: /etc/receptor/certs/ca.crt

# ─────────────────────────────────────────────────────────────────
# Network Listener (Accept Incoming Connections)
# ─────────────────────────────────────────────────────────────────
- tcp-listener:
    port: 2222
    tls: mesh-tls                    # Use TLS config defined above

# ─────────────────────────────────────────────────────────────────
# Work Signing (Control signs jobs with private key)
# ─────────────────────────────────────────────────────────────────
- work-signing:
    privatekey: /etc/receptor/work-signing/work_private_key.pem
    tokenexpiration: 1m

# Also verify (for local work, if any)
- work-verification:
    publickey: /etc/receptor/work-signing/work_public_key.pem

# ─────────────────────────────────────────────────────────────────
# Control Service (Unix socket for AWX Python ↔ Receptor)
# ─────────────────────────────────────────────────────────────────
- control-service:
    service: control
    filename: /var/run/receptor/receptor.sock

# ─────────────────────────────────────────────────────────────────
# Logging
# ─────────────────────────────────────────────────────────────────
- log-level: info
```

#### 1.5 Start Receptor and Register Node

```bash
# Start receptor
systemctl enable --now receptor

# Register in AWX database
awx-manage provision_instance --hostname=control-node --node_type=control
```

### Step 2: Set Up Execution Node

#### 2.1 Install Required Software

```bash
# Install receptor
dnf install receptor

# Install ansible-runner
pip install ansible-runner

# Install container runtime (for Execution Environments)
dnf install podman
# or: apt install docker.io
```

#### 2.2 Copy Certificates from Control Node

```bash
# Create directories
mkdir -p /etc/receptor/certs /etc/receptor/work-signing

# Copy these files FROM control node:
scp control-node:/etc/receptor/certs/ca.crt /etc/receptor/certs/
scp control-node:/etc/receptor/work-signing/work_public_key.pem /etc/receptor/work-signing/
```

#### 2.3 Generate Execution Node Certificate

```bash
# Generate cert request ON execution node
receptor --cert-makereq \
  --commonname "executor-node-1" \
  --dnsname executor1.example.com \
  --outreq /tmp/executor.req \
  --outkey /etc/receptor/certs/executor.key

# Copy request to control node, sign it there:
# (on control node)
receptor --cert-signreq \
  --cacert /etc/receptor/certs/ca.crt \
  --cakey /etc/receptor/certs/ca.key \
  --req /tmp/executor.req \
  --outcert /tmp/executor.crt

# Copy signed cert back to execution node
scp control-node:/tmp/executor.crt /etc/receptor/certs/
```

#### 2.4 Configure Receptor (Execution Node)

Create `/etc/receptor/receptor.conf`:

```yaml
---
# ─────────────────────────────────────────────────────────────────
# Node Identity
# ─────────────────────────────────────────────────────────────────
- node:
    id: executor-node-1

# ─────────────────────────────────────────────────────────────────
# TLS Configuration (Client Side)
# ─────────────────────────────────────────────────────────────────
- tls-client:
    name: mesh-tls
    cert: /etc/receptor/certs/executor.crt
    key: /etc/receptor/certs/executor.key
    rootcas: /etc/receptor/certs/ca.crt

# ─────────────────────────────────────────────────────────────────
# Connect to Control Node
# ─────────────────────────────────────────────────────────────────
- tcp-peer:
    address: control-node.example.com:2222
    tls: mesh-tls                    # Use TLS config defined above
    redial: true                     # Auto-reconnect if connection drops

# ─────────────────────────────────────────────────────────────────
# Work Verification (Verify jobs are signed by control)
# ─────────────────────────────────────────────────────────────────
- work-verification:
    publickey: /etc/receptor/work-signing/work_public_key.pem

# ─────────────────────────────────────────────────────────────────
# Work Command (How to execute jobs)
# ─────────────────────────────────────────────────────────────────
- work-command:
    worktype: local
    command: ansible-runner
    params: worker
    allowruntimeparams: true
    verifysignature: true            # IMPORTANT: Only run signed work

# ─────────────────────────────────────────────────────────────────
# Logging
# ─────────────────────────────────────────────────────────────────
- log-level: info
```

#### 2.5 Start Receptor

```bash
systemctl enable --now receptor
```

### Step 3: Configure Receptor TLS (Recommended)

Already covered in steps above. Here's the summary of what goes where:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     FILE DISTRIBUTION MATRIX                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  FILE                          CONTROL    EXECUTION   PURPOSE       │
│  ────────────────────────────  ─────────  ──────────  ────────────  │
│  ca.crt                        ✓          ✓           Trust anchor  │
│  ca.key                        ✓ (secure) ✗           Sign new certs│
│                                                                     │
│  control.crt                   ✓          ✗           Node identity │
│  control.key                   ✓          ✗           Node identity │
│                                                                     │
│  executor.crt                  ✗          ✓           Node identity │
│  executor.key                  ✗          ✓           Node identity │
│                                                                     │
│  work_private_key.pem          ✓          ✗           Sign jobs     │
│  work_public_key.pem           ✓          ✓           Verify jobs   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Step 4: Register Nodes in AWX

On the **control node**:

```bash
# Register the execution node in AWX
awx-manage provision_instance \
  --hostname=executor-node-1 \
  --node_type=execution

# Create an instance group (optional but recommended)
awx-manage create_instance_group \
  --name="production-executors"

# Add executor to the group
awx-manage add_instance_to_group \
  --instance=executor-node-1 \
  --group="production-executors"

# Register peers (tell AWX about the mesh topology)
awx-manage register_peers \
  --hostname=control-node \
  --peers executor-node-1
```

Verify the mesh:

```bash
# Check receptor status
receptorctl --socket /var/run/receptor/receptor.sock status

# Expected output:
# Node ID: control-node
# Connections:
#   executor-node-1: connected
```

### Job Execution Flow (Multi-VM)

```
┌─────────────────────────────────────────────────────────────────────┐
│                 JOB EXECUTION: MULTI-VM CLUSTER                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  CONTROL NODE (VM #1)              │  EXECUTION NODE (VM #2)        │
│  ─────────────────────             │  ──────────────────────        │
│                                    │                                │
│  1. POST /api/v2/.../launch/       │                                │
│     │                              │                                │
│     ▼                              │                                │
│  2. Job created (pending)          │                                │
│     PostgreSQL INSERT              │                                │
│     pg_notify('new_job')           │                                │
│     │                              │                                │
│     ▼                              │                                │
│  3. TaskManager.schedule()         │                                │
│     • Acquire advisory lock        │                                │
│     • Find pending job             │                                │
│     • Check capacity on nodes      │                                │
│     │                              │                                │
│     ▼                              │                                │
│  4. SELECT best node               │                                │
│     executor-node-1 chosen         │                                │
│     (has capacity, right group)    │                                │
│     status → waiting               │                                │
│     │                              │                                │
│     ▼                              │                                │
│  5. Dispatcher receives task       │                                │
│     RunJob.run() called            │                                │
│     │                              │                                │
│     ▼                              │                                │
│  6. AWXReceptorJob.run()           │                                │
│     │                              │                                │
│     ▼                              │                                │
│  7. receptorctl work submit        │                                │
│     --node executor-node-1 ────────┼───┐                            │
│     --signwork                     │   │                            │
│     │                              │   │ Receptor protocol          │
│     │                              │   │ over TLS (port 2222)       │
│     │                              │   │                            │
│     │                              │   ▼                            │
│     │                              │  8. Receptor receives work     │
│     │                              │     • Verify signature ✓       │
│     │                              │     • Spawn ansible-runner     │
│     │                              │     │                          │
│     │                              │     ▼                          │
│     │                              │  9. ansible-runner worker      │
│     │                              │     • Pull EE image            │
│     │                              │     • Mount artifacts          │
│     │                              │     • Execute playbook         │
│     │                              │     │                          │
│     │    stdout/events stream      │     │                          │
│     │ ◄────────────────────────────┼─────┘                          │
│     │                              │                                │
│     ▼                              │                                │
│  10. AWXReceptorJob.processor()    │                                │
│      Parse job events              │                                │
│      Queue to callback receiver    │                                │
│     │                              │                                │
│     ▼                              │                                │
│  11. Callback Receiver             │                                │
│      Bulk INSERT events to DB      │                                │
│     │                              │                                │
│     ▼                              │                                │
│  12. Job completes                 │                                │
│      status → successful           │                                │
│      WebSocket → UI                │                                │
│                                    │                                │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Differences from Single-VM:**

| Aspect | Single VM (Hybrid) | Multi-VM (Control + Execution) |
|--------|-------------------|-------------------------------|
| Network | Unix socket only | TCP/TLS over network |
| Where job runs | Same machine | Remote execution node |
| Receptor work target | `--node local` | `--node executor-node-1` |
| TLS required | No | Yes (recommended) |
| Scaling | Add RAM/CPU | Add more execution nodes |
| Fault isolation | None | Control survives executor failure |

---

## Receptor Security Model

AWX uses **two independent security layers** for Receptor communication:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TWO-LAYER SECURITY MODEL                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  LAYER 1: TLS (Transport Security)                                  │
│  ──────────────────────────────────                                 │
│  Purpose: Encrypt traffic, authenticate nodes                       │
│  How: Mutual TLS with certificates signed by private CA             │
│                                                                     │
│     Control Node                    Execution Node                  │
│     ┌────────────┐                  ┌────────────┐                  │
│     │ tls-server │◄─────────────────│ tls-client │                  │
│     │ control.crt│   TLS handshake  │executor.crt│                  │
│     │ control.key│   Both verify    │executor.key│                  │
│     │ ca.crt     │   against CA     │ ca.crt     │                  │
│     └────────────┘                  └────────────┘                  │
│                                                                     │
│  What it protects:                                                  │
│  ✓ Encryption of all traffic                                        │
│  ✓ Control node verifies executor identity                          │
│  ✓ Executor verifies control node identity                          │
│  ✓ Prevents man-in-the-middle attacks                               │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  LAYER 2: Work Signing (Job Authentication)                         │
│  ──────────────────────────────────────────                         │
│  Purpose: Prove job came from authorized control node               │
│  How: RSA signature on work unit payload                            │
│                                                                     │
│     Control Node                    Execution Node                  │
│     ┌────────────────┐              ┌────────────────┐              │
│     │ work-signing   │              │work-verification│             │
│     │ private_key.pem│──────────────│ public_key.pem │              │
│     └────────────────┘  signature   └────────────────┘              │
│                         attached                                    │
│                         to work                                     │
│                                                                     │
│  What it protects:                                                  │
│  ✓ Only control node can submit work (has private key)              │
│  ✓ Executor won't run unsigned/forged work                          │
│  ✓ Prevents rogue job submission even if TLS compromised            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Why Two Layers?**

| Scenario | TLS Only | Work Signing Only | Both ✓ |
|----------|----------|-------------------|--------|
| Traffic encrypted | ✓ | ✗ | ✓ |
| Nodes authenticated | ✓ | ✗ | ✓ |
| Jobs cryptographically verified | ✗ | ✓ | ✓ |
| Defense in depth | ✗ | ✗ | ✓ |

---

## Troubleshooting

### Verify Receptor Mesh Connectivity

```bash
# On control node:
receptorctl --socket /var/run/receptor/receptor.sock status

# Check if execution node is connected:
receptorctl --socket /var/run/receptor/receptor.sock ping executor-node-1
```

### Common Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `connection refused` | Port 2222 blocked | Open firewall on control node |
| `certificate verify failed` | CA mismatch | Same ca.crt on both nodes |
| `signature verification failed` | Wrong public key | Copy work_public_key.pem from control |
| `node not found` | Receptor not connected | Check `tcp-peer` address |
| Job stuck in `pending` | Instance not registered | Run `provision_instance` |
| Job stuck in `waiting` | No capacity | Check instance health in AWX UI |

### Check AWX Node Status

```bash
# List all registered instances
awx-manage list_instances

# Check instance health
awx-manage check_instance --hostname=executor-node-1
```

### Receptor Logs

```bash
# View receptor logs
journalctl -u receptor -f

# Increase verbosity (in receptor.conf)
- log-level: debug
```

### Test Work Submission Manually

```bash
# From control node, test receptor connectivity
receptorctl --socket /var/run/receptor/receptor.sock work submit \
  --node executor-node-1 \
  --worktype local \
  --payload "echo hello" \
  --follow

# Should see "hello" output if everything works
```

---

## Quick Reference: Commands Cheat Sheet

```bash
# ─────────────────────────────────────────────────────────────────
# AWX Management Commands
# ─────────────────────────────────────────────────────────────────
awx-manage provision_instance --hostname=X --node_type=control|execution|hybrid
awx-manage list_instances
awx-manage register_peers --hostname=X --peers Y,Z
awx-manage create_instance_group --name="my-group"
awx-manage add_instance_to_group --instance=X --group="my-group"

# ─────────────────────────────────────────────────────────────────
# Receptor Commands
# ─────────────────────────────────────────────────────────────────
receptorctl --socket /var/run/receptor/receptor.sock status
receptorctl --socket /var/run/receptor/receptor.sock ping <node>
receptorctl --socket /var/run/receptor/receptor.sock work list
receptorctl --socket /var/run/receptor/receptor.sock work submit --node X --worktype local ...

# ─────────────────────────────────────────────────────────────────
# Certificate Generation
# ─────────────────────────────────────────────────────────────────
receptor --cert-init ca --commonname "My CA" --outcert ca.crt --outkey ca.key
receptor --cert-makereq --commonname "node" --outreq node.req --outkey node.key
receptor --cert-signreq --cacert ca.crt --cakey ca.key --req node.req --outcert node.crt
```

---

## Next Steps

After your cluster is running:

1. **Create Instance Groups** — organize execution nodes by purpose (dev, prod, region)
2. **Configure Job Templates** — assign templates to specific instance groups
3. **Set up Execution Environments** — custom container images with your tools
4. **Enable Metrics** — Prometheus endpoint at `/api/v2/metrics/`
5. **Configure Logging** — external log aggregation for job output

See also:
- [Architecture Overview](architecture-overview.md)
- [Clustering Documentation](clustering.md)
- [Container Groups](container_groups.md)
- [Execution Environments](execution_environments.md)
