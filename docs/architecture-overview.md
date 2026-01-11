# AWX High-Level Architecture

## 1. The 30-Second Mental Model

Think of AWX as a **web-based factory for running Ansible playbooks**:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              AWX ARCHITECTURE                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│    ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                │
│    │   Browser    │──────│    API       │──────│  Scheduler   │                │
│    │   (React)    │      │  (Django)    │      │(Task Manager)│                │
│    └──────────────┘      └──────────────┘      └──────┬───────┘                │
│          │                      │                      │                        │
│          │ REST                 │ ORM                  │ dispatch               │
│          ▼                      ▼                      ▼                        │
│    ┌─────────────────────────────────────────────────────────────────────┐     │
│    │                        PostgreSQL                                    │     │
│    │     (Jobs, Inventories, Credentials, Events, Settings, etc.)        │     │
│    └─────────────────────────────────────────────────────────────────────┘     │
│                                   │                                             │
│                                   │ NOTIFY/LISTEN                               │
│                                   ▼                                             │
│    ┌─────────────────────────────────────────────────────────────────────┐     │
│    │                         Dispatcher                                   │     │
│    │               (PostgreSQL pub/sub → worker pool)                    │     │
│    └──────────────────────────────┬──────────────────────────────────────┘     │
│                                   │                                             │
│                                   │ Receptor mesh                               │
│                                   ▼                                             │
│    ┌─────────────────────────────────────────────────────────────────────┐     │
│    │                      Execution Nodes                                 │     │
│    │            (ansible-runner inside Execution Environments)           │     │
│    └─────────────────────────────────────────────────────────────────────┘     │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 2. The Four Planes of AWX

AWX operates across **four logical layers**:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  PLANE 1: UI                                                                     │
│  ─────────────────────────────────────────────────────────────────────────────  │
│  │ React SPA │ → REST API calls → WebSocket for live updates                   │
│  │ awx/ui/   │                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  PLANE 2: API / CONTROL                                                          │
│  ─────────────────────────────────────────────────────────────────────────────  │
│  │ Django + DRF  │ → Models, Serializers, Views, Auth, RBAC                    │
│  │ awx/api/      │ → Business logic, validation, permissions                   │
│  │ awx/main/     │                                                              │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  PLANE 3: SCHEDULING / ORCHESTRATION                                             │
│  ─────────────────────────────────────────────────────────────────────────────  │
│  │ Task Manager    │ → Decides WHAT runs WHERE and WHEN                        │
│  │ Dispatcher      │ → Routes work to correct node via pg NOTIFY               │
│  │ awx/main/scheduler/ │                                                       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  PLANE 4: EXECUTION                                                              │
│  ─────────────────────────────────────────────────────────────────────────────  │
│  │ ansible-runner  │ → Actually runs Ansible playbooks                         │
│  │ Receptor        │ → Secure mesh for remote execution                        │
│  │ Execution Envs  │ → Container images with Ansible + dependencies            │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2a. Deep Dive: Task Manager (Plane 3)

The Task Manager is the **scheduler brain** of AWX. It answers three questions:

### WHAT runs? (It's not just "jobs"!)

AWX has **6 different types of runnable tasks**, not just playbook jobs:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         TASK TYPES IN AWX                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐               │
│  │      Job        │   │  ProjectUpdate  │   │ InventoryUpdate │               │
│  │                 │   │                 │   │                 │               │
│  │ Run a playbook  │   │ Sync from Git   │   │ Sync from AWS/  │               │
│  │ against hosts   │   │ or other SCM    │   │ VMware/etc.     │               │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘               │
│                                                                                  │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐               │
│  │  AdHocCommand   │   │    SystemJob    │   │   WorkflowJob   │               │
│  │                 │   │                 │   │                 │               │
│  │ One-off command │   │ AWX maintenance │   │ Orchestrate     │               │
│  │ (ping, shell)   │   │ (cleanup, etc.) │   │ multiple jobs   │               │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

| Task Type | Model Class | What It Does | Example |
|-----------|-------------|--------------|---------|
| **Job** | `Job` | Runs an Ansible playbook against inventory | "Deploy nginx to web servers" |
| **ProjectUpdate** | `ProjectUpdate` | Syncs playbooks from Git/SVN/Archive | "Pull latest from `main` branch" |
| **InventoryUpdate** | `InventoryUpdate` | Syncs hosts from cloud/CMDB sources | "Refresh EC2 instance list" |
| **AdHocCommand** | `AdHocCommand` | Runs a one-off Ansible module | "`ping` all hosts in inventory" |
| **SystemJob** | `SystemJob` | AWX internal maintenance tasks | "Cleanup jobs older than 30 days" |
| **WorkflowJob** | `WorkflowJob` | Orchestrates a graph of other jobs | "Sync → Deploy → Test → Notify" |

---

#### AdHocCommand: When to Use Instead of a Job

An **AdHocCommand** is a one-off Ansible module execution—no playbook required. It's the AWX equivalent of running `ansible` directly on the command line (vs `ansible-playbook`).

**When to use AdHocCommand vs Job:**

| Use AdHocCommand When... | Use Job When... |
|--------------------------|-----------------|
| Quick one-time task | Repeatable automation |
| No playbook exists yet | You have a playbook |
| Troubleshooting/debugging | Production workflows |
| Simple module call (`ping`, `shell`, `setup`) | Complex multi-task logic |
| Testing connectivity | Orchestrated deployments |

**Real-world examples:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         AD HOC COMMAND USE CASES                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  "Are my servers reachable?"                                                    │
│  ─────────────────────────────                                                  │
│  Module: ping                                                                   │
│  Args: (none)                                                                   │
│  → Quick connectivity check without writing a playbook                          │
│                                                                                  │
│  "What's the uptime on these hosts?"                                            │
│  ────────────────────────────────────                                           │
│  Module: command                                                                │
│  Args: uptime                                                                   │
│  → One-liner to gather info                                                     │
│                                                                                  │
│  "Restart nginx on web servers NOW"                                             │
│  ───────────────────────────────────                                            │
│  Module: service                                                                │
│  Args: name=nginx state=restarted                                               │
│  → Emergency fix without creating/modifying a playbook                          │
│                                                                                  │
│  "Gather facts from new hosts"                                                  │
│  ──────────────────────────────                                                 │
│  Module: setup                                                                  │
│  Args: (none)                                                                   │
│  → Populate AWX fact cache for new inventory members                            │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Allowed modules** (configurable via `AD_HOC_COMMANDS` setting):

```python
# From awx/settings/defaults.py
AD_HOC_COMMANDS = [
    'command', 'shell',           # Run arbitrary commands
    'ping', 'win_ping',           # Connectivity testing
    'setup',                      # Gather facts
    'yum', 'apt', 'apt_key', ...  # Package management
    'service', 'win_service',     # Service management
    'user', 'group',              # User/group management
    'mount', 'selinux',           # System configuration
]
```

**Key limitation**: AdHocCommands don't have templates—they're not reusable like Job Templates. If you find yourself running the same ad-hoc command repeatedly, that's a sign you should create a playbook and Job Template instead.

---

#### SystemJob: AWX's Housekeeping Tasks

**SystemJobs** are AWX's internal maintenance tasks. They clean up old data to keep the database healthy and prevent unbounded growth.

**The 4 SystemJob Types:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         SYSTEM JOB TYPES                                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  1. cleanup_jobs                                                                │
│     ────────────                                                                │
│     Removes old Job, ProjectUpdate, InventoryUpdate, AdHocCommand,             │
│     SystemJob, and WorkflowJob records from the database.                       │
│                                                                                  │
│     Default: 90 days                                                            │
│     Parameter: days=N (e.g., days=30)                                           │
│     What it deletes: Job records, their events, stdout, artifacts              │
│                                                                                  │
│  2. cleanup_activitystream                                                      │
│     ──────────────────────                                                      │
│     Removes old Activity Stream entries (audit log of who did what).           │
│                                                                                  │
│     Default: 90 days                                                            │
│     Parameter: days=N                                                           │
│     What it deletes: ActivityStream records                                     │
│                                                                                  │
│  3. cleanup_sessions                                                            │
│     ────────────────                                                            │
│     Removes expired browser session records from the database.                  │
│     (Users who logged into the web UI have sessions stored)                     │
│                                                                                  │
│     Parameter: (none, just removes expired ones)                                │
│                                                                                  │
│  4. cleanup_tokens                                                              │
│     ──────────────                                                              │
│     Removes expired OAuth 2 access tokens and refresh tokens.                   │
│     (API tokens that have passed their expiration date)                         │
│                                                                                  │
│     Parameter: (none, just removes expired ones)                                │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**How are SystemJobs triggered?**

SystemJobs can be run in **three ways**:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    HOW TO RUN SYSTEM JOBS                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  METHOD 1: Manual (via UI or API)                                               │
│  ─────────────────────────────────                                              │
│  Navigate to: Administration → Management Jobs → Launch                         │
│  API: POST /api/v2/system_job_templates/{id}/launch/                            │
│       Body: {"extra_vars": {"days": 30}}                                        │
│                                                                                  │
│  METHOD 2: Scheduled (recommended for production)                               │
│  ────────────────────────────────────────────────                               │
│  Each SystemJobTemplate can have Schedules attached.                            │
│  Navigate to: Administration → Management Jobs → Schedules                      │
│  Example: Run cleanup_jobs every Sunday at 2 AM                                 │
│                                                                                  │
│  API: POST /api/v2/system_job_templates/{id}/schedules/                         │
│       Body: {                                                                   │
│         "name": "Weekly cleanup",                                               │
│         "rrule": "DTSTART:20240101T020000Z RRULE:FREQ=WEEKLY;BYDAY=SU",        │
│         "extra_data": {"days": 30}                                              │
│       }                                                                         │
│                                                                                  │
│  METHOD 3: Never (not recommended)                                              │
│  ─────────────────────────────────                                              │
│  If you never run cleanup jobs, your database will grow forever!               │
│  Job events alone can be millions of rows per month in busy systems.           │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Important**: AWX does NOT run these automatically out of the box! You must set up schedules or run them manually. In production, you should schedule `cleanup_jobs` and `cleanup_activitystream` to run regularly (e.g., weekly).

**Where to find them in the UI:**

Administration → Management Jobs

---

#### WorkflowJob: Orchestrating Multiple Jobs

A **WorkflowJob** is a directed acyclic graph (DAG) that orchestrates multiple jobs with conditional logic. Think of it as a "meta-job" that runs other jobs in a defined order with success/failure branching.

**Visual Example:**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         WORKFLOW EXAMPLE: Deploy Application                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│                            ┌─────────────────┐                                  │
│                            │  Sync Project   │                                  │
│                            │ (ProjectUpdate) │                                  │
│                            └────────┬────────┘                                  │
│                                     │                                           │
│                                     │ on success                                │
│                                     ▼                                           │
│                            ┌─────────────────┐                                  │
│                            │  Deploy to Dev  │                                  │
│                            │     (Job)       │                                  │
│                            └────────┬────────┘                                  │
│                                     │                                           │
│                      ┌──────────────┴──────────────┐                            │
│                      │                             │                            │
│                      │ on success                  │ on failure                 │
│                      ▼                             ▼                            │
│             ┌─────────────────┐           ┌─────────────────┐                   │
│             │  Run Tests      │           │  Notify: Failed │                   │
│             │     (Job)       │           │     (Job)       │                   │
│             └────────┬────────┘           └─────────────────┘                   │
│                      │                                                          │
│           ┌──────────┴──────────┐                                               │
│           │                     │                                               │
│           │ on success          │ on failure                                    │
│           ▼                     ▼                                               │
│  ┌─────────────────┐   ┌─────────────────┐                                      │
│  │ Deploy to Prod  │   │ Rollback Dev    │                                      │
│  │     (Job)       │   │     (Job)       │                                      │
│  └────────┬────────┘   └────────┬────────┘                                      │
│           │                     │                                               │
│           │ always              │ always                                        │
│           ▼                     ▼                                               │
│  ┌─────────────────┐   ┌─────────────────┐                                      │
│  │ Notify: Success │   │ Notify: Failed  │                                      │
│  │     (Job)       │   │     (Job)       │                                      │
│  └─────────────────┘   └─────────────────┘                                      │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Key Workflow Concepts:**

| Concept | Description |
|---------|-------------|
| **Node** | A step in the workflow (wraps a Job Template, Project, Inventory Source, or another Workflow) |
| **success_nodes** | Nodes to run if this node's job succeeds |
| **failure_nodes** | Nodes to run if this node's job fails |
| **always_nodes** | Nodes to run regardless of success/failure |
| **Convergence** | Option to wait for ALL parent nodes before running (vs any parent) |
| **Approval Node** | Special node that pauses workflow until a human approves |

**What can be inside a Workflow Node?**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    WORKFLOW NODE TYPES                                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────┐   A regular playbook job                                   │
│  │  Job Template   │   Most common node type                                    │
│  └─────────────────┘                                                            │
│                                                                                  │
│  ┌─────────────────┐   Sync a project before running jobs                       │
│  │    Project      │   Ensures latest playbooks                                 │
│  └─────────────────┘                                                            │
│                                                                                  │
│  ┌─────────────────┐   Sync inventory before running jobs                       │
│  │InventorySource  │   Ensures latest host list                                │
│  └─────────────────┘                                                            │
│                                                                                  │
│  ┌─────────────────┐   Nested workflow (workflow-in-workflow)                   │
│  │WorkflowJobTempl │   For reusable sub-processes                               │
│  └─────────────────┘                                                            │
│                                                                                  │
│  ┌─────────────────┐   Pause until human approves                               │
│  │   Approval      │   For change management gates                              │
│  └─────────────────┘                                                            │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**When to use Workflows vs a single Job:**

| Use Workflow When... | Use Single Job When... |
|---------------------|------------------------|
| Multiple playbooks need to run in sequence | One playbook does everything |
| Different playbooks for different failure paths | Simple success/fail is enough |
| You need human approval gates | Fully automated |
| Combining different inventory sources | Single inventory |
| Reusing existing Job Templates in new combinations | New automation |

**Artifact Passing:**

Workflows can pass data between nodes using **artifacts**. If a job sets a fact using `set_stats`, that data is available to downstream nodes:

```yaml
# In playbook for Node A
- name: Save deployment version
  set_stats:
    data:
      deployed_version: "1.2.3"

# In playbook for Node B (downstream)
# Access via: {{ deployed_version }}
```

The Task Manager queries **ALL** of these together and processes them in a single sorted list:

```python
# From awx/main/scheduler/task_manager.py:get_tasks()
all_tasks = sorted(
    jobs + project_updates + inventory_updates + 
    system_jobs + ad_hoc_commands + workflow_jobs,
    key=lambda task: task.created  # Oldest first (FIFO)
)
```

### WHERE does it run? (Nodes are NOT all the same!)

Different tasks require different node types, and nodes have varying capacity:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         NODE SELECTION FACTORS                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  FACTOR 1: Node Type Compatibility                                              │
│  ────────────────────────────────────                                           │
│                                                                                  │
│  Task Type          │ Can Run On                                                │
│  ───────────────────┼───────────────────────────────────                        │
│  Job (playbook)     │ Execution node, Hybrid node                               │
│  ProjectUpdate      │ Control node, Hybrid node (needs API access)              │
│  InventoryUpdate    │ Control node, Hybrid node (writes to DB)                  │
│  SystemJob          │ Control node, Hybrid node (maintenance tasks)             │
│                                                                                  │
│  FACTOR 2: Capacity                                                             │
│  ──────────────────                                                             │
│                                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                    │
│  │   Node A     │     │   Node B     │     │   Node C     │                    │
│  │              │     │              │     │              │                    │
│  │ Capacity: 40 │     │ Capacity: 20 │     │ Capacity: 8  │                    │
│  │ Running: 35  │     │ Running: 5   │     │ Running: 0   │                    │
│  │ ─────────    │     │ ─────────    │     │ ─────────    │                    │
│  │ Free: 5  ❌  │     │ Free: 15 ✓   │     │ Free: 8  ✓   │                    │
│  └──────────────┘     └──────────────┘     └──────────────┘                    │
│                                                                                  │
│  Job needs capacity=10 → Node A can't fit, Node B or C can                     │
│  Task Manager picks Node B (most remaining capacity)                            │
│                                                                                  │
│  FACTOR 3: Instance Groups                                                      │
│  ─────────────────────────                                                      │
│                                                                                  │
│  ┌─────────────────────────────┐     ┌─────────────────────────────┐           │
│  │  Instance Group: "prod"     │     │  Instance Group: "dev"      │           │
│  │                             │     │                             │           │
│  │  ┌───────┐    ┌───────┐    │     │  ┌───────┐                  │           │
│  │  │Node A │    │Node B │    │     │  │Node C │                  │           │
│  │  └───────┘    └───────┘    │     │  └───────┘                  │           │
│  └─────────────────────────────┘     └─────────────────────────────┘           │
│                                                                                  │
│  Job Template assigned to "prod" → Only runs on Node A or B                    │
│                                                                                  │
│  FACTOR 4: Container Groups (special case)                                      │
│  ─────────────────────────────────────────                                      │
│                                                                                  │
│  Some Instance Groups have is_container_group=True                              │
│  These don't have static nodes — they spawn ephemeral K8s pods per job         │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### WHEN can it run? (Jobs can be blocked!)

Not every pending job can start immediately:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         BLOCKING RULES                                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  RULE 1: Dependencies Must Complete First                                       │
│  ─────────────────────────────────────────                                      │
│                                                                                  │
│  If Job Template has "Update on Launch" enabled for Project:                    │
│                                                                                  │
│       ┌───────────────┐         ┌───────────────┐                              │
│       │ ProjectUpdate │ ──────► │     Job       │                              │
│       │   (pending)   │  blocks │   (pending)   │                              │
│       └───────────────┘         └───────────────┘                              │
│                                                                                  │
│  The Job won't start until the ProjectUpdate completes successfully            │
│                                                                                  │
│  RULE 2: Same-Template Serialization                                            │
│  ────────────────────────────────────                                           │
│                                                                                  │
│  By default, only ONE job per Job Template can run at a time:                   │
│                                                                                  │
│       Job Template: "Deploy App"                                                │
│       ┌───────────────┐         ┌───────────────┐                              │
│       │   Job #42     │ ──────► │   Job #43     │                              │
│       │  (running)    │  blocks │   (pending)   │                              │
│       └───────────────┘         └───────────────┘                              │
│                                                                                  │
│  (Can be disabled with allow_simultaneous=True on Job Template)                 │
│                                                                                  │
│  RULE 3: Inventory Source Lock                                                  │
│  ─────────────────────────────                                                  │
│                                                                                  │
│  Only ONE InventoryUpdate per InventorySource can run at a time                │
│                                                                                  │
│  RULE 4: Capacity Exhausted                                                     │
│  ──────────────────────────                                                     │
│                                                                                  │
│  If all nodes in the job's Instance Group are at capacity:                      │
│  Job stays "pending" with explanation: "not enough available capacity"         │
│                                                                                  │
│  RULE 5: Workflow Ordering                                                      │
│  ─────────────────────────                                                      │
│                                                                                  │
│       ┌─────────┐     ┌─────────┐     ┌─────────┐                              │
│       │ Node A  │────►│ Node B  │────►│ Node C  │                              │
│       │(running)│     │(waiting)│     │(waiting)│                              │
│       └─────────┘     └─────────┘     └─────────┘                              │
│                                                                                  │
│  Node B's job won't spawn until Node A's job succeeds/fails                    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Task Manager Algorithm (Simplified)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      TASK MANAGER LOOP (every 20 seconds)                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  1. ACQUIRE LOCK (only one Task Manager runs at a time across the cluster)     │
│     └─► If lock taken by another node → exit immediately                       │
│                                                                                  │
│  2. FETCH ALL TASKS (pending + waiting + running across all 6 types)           │
│     └─► Sort by created timestamp (oldest first = FIFO)                        │
│                                                                                  │
│  3. BUILD CAPACITY MAP                                                          │
│     └─► Calculate remaining capacity on each node                              │
│     └─► Account for already-running and waiting jobs                           │
│                                                                                  │
│  4. PROCESS WORKFLOWS                                                           │
│     └─► For each running WorkflowJob:                                          │
│         └─► Check which nodes are ready (dependencies met)                     │
│         └─► Spawn child jobs for ready nodes                                   │
│                                                                                  │
│  5. GENERATE DEPENDENCIES                                                       │
│     └─► For Jobs with update_on_launch → create ProjectUpdate/InventoryUpdate  │
│     └─► Link them as dependencies                                              │
│                                                                                  │
│  6. FOR EACH PENDING TASK (in creation order):                                  │
│     │                                                                           │
│     ├─► Is it blocked by a dependency? → SKIP                                  │
│     │                                                                           │
│     ├─► Is there a control node with capacity? → NO → SKIP                     │
│     │                                                                           │
│     ├─► Find preferred Instance Groups for this task                           │
│     │   └─► For each group:                                                    │
│     │       ├─► Container Group? → START (no capacity check, creates pod)      │
│     │       └─► Find node with most remaining capacity that fits this task     │
│     │           ├─► Found node → START on that node                            │
│     │           └─► No node fits → try next group                              │
│     │                                                                           │
│     └─► No group could run it → leave pending, update job_explanation          │
│                                                                                  │
│  7. RELEASE LOCK                                                                │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Key Code Locations

| Component | File | Key Method |
|-----------|------|------------|
| Entry point | `awx/main/scheduler/task_manager.py` | `schedule()` |
| Fetch all tasks | `awx/main/scheduler/task_manager.py` | `get_tasks()` |
| Blocking logic | `awx/main/scheduler/task_manager.py` | `job_blocked_by()` |
| Instance selection | `awx/main/scheduler/task_manager_models.py` | `fit_task_to_most_remaining_capacity_instance()` |
| Start a task | `awx/main/scheduler/task_manager.py` | `start_task()` |
| Workflow graph | `awx/main/scheduler/dag_workflow.py` | `WorkflowDAG` |
| Dependency creation | `awx/main/scheduler/task_manager.py` | `generate_dependencies()` |

### .NET Analogy

If you're coming from .NET, think of the Task Manager like:

| AWX Concept | .NET Equivalent |
|-------------|-----------------|
| Task Manager | Hangfire/Quartz Scheduler + custom dispatcher |
| `schedule()` loop | Background Service polling for work |
| Advisory Lock | Distributed lock (Redis/SQL-based) |
| Instance capacity | Worker pool with concurrency limits |
| Dependencies | Job continuations / parent-child jobs |
| Instance Groups | Named queues (like Hangfire queues) |

## 3. Job Execution Flow (The Heart of AWX)

This is the end-to-end flow when you click **"Launch"**:

```
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │                         USER: Click "Launch" Button                          │
 └───────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ 1. FRONTEND                                                                  │
 │    ────────                                                                  │
 │    LaunchButton.js → POST /api/v2/job_templates/{id}/launch/                │
 │    Creates job record in database with status = "pending"                   │
 └───────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ 2. API LAYER (Django REST Framework)                                         │
 │    ─────────────────────────────────                                         │
 │    JobTemplateLaunch.post() → create_unified_job() → signal_start()         │
 │    Returns: { "id": 42, "status": "pending", ... }                          │
 └───────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ 3. TASK MANAGER (Scheduler)                                                  │
 │    ────────────────────────                                                  │
 │    Runs periodically + on events                                            │
 │    schedule() → Acquires lock → Finds pending jobs → Checks capacity        │
 │    Picks best node → status = "waiting" → Dispatches work                   │
 └───────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ 4. DISPATCHER                                                                │
 │    ──────────                                                                │
 │    pg_notify(queue, job_message) → PostgreSQL LISTEN/NOTIFY                 │
 │    Worker on target node receives message                                   │
 └───────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ 5. JOB RUNNER                                                                │
 │    ──────────                                                                │
 │    RunJob.run() → status = "running"                                        │
 │    Build private_data_dir → Inject credentials → ansible-runner.run()      │
 │    OR: AWXReceptorJob for remote execution nodes                            │
 └───────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ 6. ANSIBLE EXECUTION                                                         │
 │    ─────────────────                                                         │
 │    ansible-playbook runs inside Execution Environment (container)           │
 │    Each Ansible event → callback → JobEvent record → WebSocket              │
 └───────────────────────────────────┬─────────────────────────────────────────┘
                                     │
                                     ▼
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │ 7. COMPLETION                                                                │
 │    ──────────                                                                │
 │    status = "successful" | "failed" | "error"                               │
 │    WebSocket emits final status → UI updates                                │
 └─────────────────────────────────────────────────────────────────────────────┘
```

## 4. Job Status State Machine

```
                    ┌──────────────────────────────────────────────┐
                    │                                              │
                    ▼                                              │
    ┌───────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐       │
    │  new  │───►│ pending │───►│ waiting │───►│ running │───────┤
    └───────┘    └────┬────┘    └────┬────┘    └────┬────┘       │
                      │              │              │             │
                      │              │              ├───► successful
                      │              │              │
                      │              │              ├───► failed
                      │              │              │
                      │              │              └───► error
                      │              │
                      └──────┬───────┘
                             │
                             ▼
                        canceled
```

| Status | What It Means |
|--------|---------------|
| `new` | Job created, not yet submitted to scheduler |
| `pending` | In scheduler queue, waiting for capacity/dependencies |
| `waiting` | Assigned to node, waiting for worker to pick up |
| `running` | Ansible is executing |
| `successful` | Playbook finished with exit code 0 |
| `failed` | Playbook finished with non-zero exit |
| `error` | AWX system error (couldn't start, node unreachable, etc.) |
| `canceled` | User or system canceled the job |

## 5. Node Types and Clustering

AWX supports **horizontal scaling** with different node types:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              AWX CLUSTER                                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   CONTROL PLANE                           EXECUTION PLANE                       │
│   ─────────────                           ───────────────                       │
│                                                                                  │
│   ┌─────────────┐    ┌─────────────┐      ┌─────────────┐    ┌─────────────┐   │
│   │  Control    │    │  Control    │      │  Execution  │    │  Execution  │   │
│   │   Node 1    │◄──►│   Node 2    │      │   Node 1    │    │   Node 2    │   │
│   │             │    │             │      │             │    │             │   │
│   │ • API       │    │ • API       │      │ • ansible-  │    │ • ansible-  │   │
│   │ • Scheduler │    │ • Scheduler │      │   runner    │    │   runner    │   │
│   │ • Web UI    │    │ • Web UI    │      │ • Receptor  │    │ • Receptor  │   │
│   │ • Callbacks │    │ • Callbacks │      │             │    │             │   │
│   └──────┬──────┘    └─────────────┘      └──────▲──────┘    └─────────────┘   │
│          │                                        │                             │
│          │         ┌─────────────┐               │                             │
│          └────────►│  Hop Node   │───────────────┘                             │
│                    │  (Relay)    │     Receptor Mesh                           │
│                    └─────────────┘     (secure comms)                          │
│                                                                                  │
│   ┌─────────────┐                                                               │
│   │   Hybrid    │  ← Can do BOTH control + execution                           │
│   │    Node     │    (common in dev, small deployments)                        │
│   └─────────────┘                                                               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

| Node Type | Role | Runs What |
|-----------|------|-----------|
| **Control** | Brain | API, Scheduler, Web UI, Callback receiver |
| **Execution** | Muscle | ansible-runner, Receptor |
| **Hybrid** | Both | Everything (dev default) |
| **Hop** | Relay | Just Receptor (routes traffic) |

## 6. Key Services (Background Processes)

These are the long-running processes inside AWX containers:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         AWX BACKGROUND SERVICES                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌───────────────────┐                                                          │
│  │   uwsgi / daphne  │  HTTP server (API + WebSocket)                          │
│  │                   │  Entry: awx/wsgi.py, awx/asgi.py                        │
│  └───────────────────┘                                                          │
│                                                                                  │
│  ┌───────────────────┐                                                          │
│  │   run_dispatcher  │  Background task worker pool                            │
│  │                   │  Entry: awx/main/management/commands/run_dispatcher.py  │
│  └───────────────────┘                                                          │
│                                                                                  │
│  ┌───────────────────┐                                                          │
│  │ run_callback_     │  Processes Ansible events from jobs                     │
│  │     receiver      │  Entry: awx/main/management/commands/                   │
│  └───────────────────┘         run_callback_receiver.py                        │
│                                                                                  │
│  ┌───────────────────┐                                                          │
│  │   run_wsrelay     │  WebSocket message relay between nodes                  │
│  │                   │  Entry: awx/main/management/commands/run_wsrelay.py     │
│  └───────────────────┘                                                          │
│                                                                                  │
│  ┌───────────────────┐                                                          │
│  │  run_ws_heartbeat │  Node health monitoring                                 │
│  │                   │  Entry: awx/main/management/commands/                   │
│  └───────────────────┘         run_ws_heartbeat.py                             │
│                                                                                  │
│  ┌───────────────────┐                                                          │
│  │     receptor      │  Secure mesh networking to execution nodes              │
│  │                   │  External binary (golang)                               │
│  └───────────────────┘                                                          │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 7. Data Model (Core Entities)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              AWX DATA MODEL                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│                            ┌──────────────┐                                     │
│                            │ Organization │  ← Top-level container              │
│                            └──────┬───────┘                                     │
│                 ┌─────────────────┼─────────────────┐                           │
│                 │                 │                 │                           │
│                 ▼                 ▼                 ▼                           │
│          ┌───────────┐    ┌───────────┐     ┌────────────┐                     │
│          │  Project  │    │ Inventory │     │ Credential │                     │
│          │           │    │           │     │            │                     │
│          │ • SCM URL │    │ • Hosts   │     │ • SSH keys │                     │
│          │ • Branch  │    │ • Groups  │     │ • Passwords│                     │
│          │ • Playbks │    │ • Vars    │     │ • Tokens   │                     │
│          └─────┬─────┘    └─────┬─────┘     └──────┬─────┘                     │
│                │                │                  │                            │
│                └────────────────┼──────────────────┘                            │
│                                 │                                               │
│                                 ▼                                               │
│                         ┌──────────────┐                                        │
│                         │ Job Template │  ← Reusable job definition            │
│                         │              │                                        │
│                         │ • Project    │                                        │
│                         │ • Inventory  │                                        │
│                         │ • Playbook   │                                        │
│                         │ • Credentials│                                        │
│                         │ • Extra vars │                                        │
│                         └──────┬───────┘                                        │
│                                │                                                │
│                         Launch │                                                │
│                                ▼                                                │
│                         ┌──────────────┐                                        │
│                         │     Job      │  ← Single execution instance          │
│                         │              │                                        │
│                         │ • Status     │                                        │
│                         │ • Output     │                                        │
│                         │ • Events     │──────► ┌────────────┐                  │
│                         │ • Artifacts  │        │ Job Events │                  │
│                         └──────────────┘        │ (1000s per │                  │
│                                                 │    job)    │                  │
│                                                 └────────────┘                  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 8. Directory → Component Mapping

| Directory | Layer | .NET Analogy |
|-----------|-------|--------------|
| `awx/ui/` | React frontend | Blazor / Angular frontend |
| `awx/api/` | REST API (DRF views, serializers) | ASP.NET Web API controllers |
| `awx/main/models/` | Domain models (ORM) | Entity Framework DbContext + entities |
| `awx/main/scheduler/` | Task Manager | Hangfire / Quartz scheduler |
| `awx/main/tasks/` | Background jobs | Hosted services / workers |
| `awx/main/dispatch/` | Message queue | MassTransit / Azure Service Bus |
| `awx/conf/` | Dynamic settings | IConfiguration + IOptions |
| `awx/settings/` | Static settings | appsettings.json |

## 9. Quick Reference: Where To Look

| "I want to understand..." | Start Here |
|---------------------------|------------|
| How jobs are launched | `awx/api/views/__init__.py` → `JobTemplateLaunch` |
| How scheduling works | `awx/main/scheduler/task_manager.py` |
| The Job model | `awx/main/models/jobs.py` |
| Background task execution | `awx/main/tasks/jobs.py` → `RunJob` |
| WebSocket updates | `awx/main/wsrelay.py` |
| API authentication | `awx/api/authentication.py` |
| RBAC / permissions | `awx/main/access.py` |
| Capacity calculation | `awx/main/utils/common.py` |

## 10. Related Documentation

- [Job Execution Deep Dive](../deepdive/01-job-execution-overview.md)
- [Task Manager System](task_manager_system.md)
- [Clustering](clustering.md)
- [Capacity](capacity.md)
- [WebSockets](websockets.md)

---

## TODO: Sections to Enhance

- [x] ~~Add Task Manager deep dive (what/where/when decisions)~~
- [ ] Add Dispatcher deep dive (how tasks are routed to workers)
- [ ] Add detailed API layer patterns (serializers, views, permissions)
- [ ] Add database schema diagrams
- [ ] Add Receptor mesh topology details
- [ ] Add Execution Environment deep dive
- [ ] Add authentication flow diagrams
- [ ] Add RBAC model explanation
