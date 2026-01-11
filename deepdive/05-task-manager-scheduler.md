# Task Manager & Scheduler

This document details how AWX schedules jobs across instances.

## What to take away

- Task Manager runs periodically and on-demand after job creation.
- It builds a unified task list, enforces dependencies, and assigns instances.
- Most "job stuck in pending" issues are capacity or dependency related.

## Key Files

| File | Purpose |
|------|---------|
| `awx/main/scheduler/task_manager.py` | Main scheduling logic |
| `awx/main/scheduler/task_manager_models.py` | Instance/InstanceGroup wrappers |
| `awx/main/scheduler/dag_workflow.py` | Workflow DAG processing |
| `awx/main/scheduler/dag_simple.py` | Base DAG implementation |
| `awx/main/scheduler/tasks.py` | Periodic task entry point |
| `awx/settings/defaults.py` | CELERYBEAT_SCHEDULE config |

## How Task Manager Is Triggered

### 1. Periodic Trigger (Every 20 seconds)

**File**: `awx/settings/defaults.py:437`

```python
CELERYBEAT_SCHEDULE = {
    'task_manager': {
        'task': 'awx.main.scheduler.tasks.run_task_manager',
        'schedule': timedelta(seconds=20),
        'options': {'expires': 20, 'queue': queue}
    },
    # ... other periodic tasks
}
```

### 2. On-Demand Trigger (After Job Creation)

**File**: `awx/main/utils/common.py:864-892`

```python
def schedule_task_manager():
    """Trigger task manager on next DB commit"""
    from awx.main.scheduler.tasks import run_task_manager

    def _schedule():
        run_task_manager.delay()

    # Use connection.on_commit to avoid race conditions
    # Only schedules if not already scheduled in this transaction
    connection.on_commit(_schedule)
```

Called from `signal_start()` after job transitions to `pending`.

## Common Scheduling Failure Modes

| Symptom | Likely Cause | Where to Look |
|---------|--------------|---------------|
| Job stays `pending` | Missing dependencies | `create_dependencies()` and `has_pending_dependencies()` |
| Job stays `pending` | No eligible instance group | `get_instance_group()` |
| Job stays `pending` | Capacity exhausted | `has_capacity()` and instance capacity values |
| Job stuck `waiting` | Dispatcher not running | `awx/main/management/commands/run_dispatcher.py` |

## TaskManager.schedule() Entry Point

**File**: `awx/main/scheduler/task_manager.py:708`

```python
def schedule(self):
    """Main entry point for task scheduling"""

    # 1. Acquire exclusive lock (prevents concurrent schedulers)
    with advisory_lock('task_manager_lock', wait=False) as acquired:
        if not acquired:
            logger.debug("Task manager already running, skipping")
            return

        # 2. Run the actual scheduling
        finished_wfjs = self._schedule()

        # 3. Record metrics
        self.record_aggregate_metrics()

    return finished_wfjs
```

## _schedule() Core Logic

**File**: `awx/main/scheduler/task_manager.py:650-684`

```python
def _schedule(self):
    # 1. Get ALL tasks (pending, waiting, running) across all job types
    all_sorted_tasks = self.get_tasks()

    # 2. Build structures for tracking capacity
    self.all_instances = TaskManagerInstances(all_sorted_tasks)
    self.all_ig = TaskManagerInstanceGroups(
        self.all_instances.instances_by_hostname
    )

    # 3. Handle running workflow jobs
    running_workflow_tasks = [
        t for t in all_sorted_tasks
        if isinstance(t, WorkflowJob) and t.status == 'running'
    ]

    # 4. Mark workflow nodes that shouldn't run
    for wf in running_workflow_tasks:
        dag = WorkflowDAG(wf)
        dag.mark_dnr_nodes()

    # 5. Spawn new jobs from workflow graphs
    self.spawn_workflow_graph_jobs(running_workflow_tasks)

    # 6. Handle approval node timeouts
    self.timeout_approval_node()

    # 7. Clean up orphaned jobs
    self.reap_jobs_from_orphaned_instances()

    # 8. Process all pending tasks
    self.process_tasks(all_sorted_tasks)

    return finished_wfjs
```

## get_tasks() - Fetching All Jobs

**File**: `awx/main/scheduler/task_manager.py:126`

```python
def get_tasks(self, status_list=('pending', 'waiting', 'running')):
    """
    Query ALL job types from the database.
    Returns a single sorted list of all jobs.
    """
    kv = {'status__in': status_list}

    # Query each job type separately
    jobs = Job.objects.filter(**kv)
    project_updates = ProjectUpdate.objects.filter(**kv)
    inventory_updates = InventoryUpdate.objects.filter(**kv)
    system_jobs = SystemJob.objects.filter(**kv)
    ad_hoc_commands = AdHocCommand.objects.filter(**kv)
    workflow_jobs = WorkflowJob.objects.filter(**kv)

    # Combine and sort by creation time
    all_tasks = list(chain(
        jobs,
        project_updates,
        inventory_updates,
        system_jobs,
        ad_hoc_commands,
        workflow_jobs
    ))

    # Sort by created time (oldest first = FIFO)
    return sorted(all_tasks, key=lambda t: t.created)
```

## process_tasks() - Main Scheduling Loop

**File**: `awx/main/scheduler/task_manager.py:485`

```python
def process_tasks(self, all_sorted_tasks):
    """Process pending tasks and start those that can run"""

    for task in all_sorted_tasks:
        if task.status != 'pending':
            continue  # Already waiting/running

        # 1. Check if task is blocked by dependencies
        if self.is_task_blocked(task):
            continue

        # 2. Create dependency jobs if needed
        # (e.g., sync project before running playbook)
        self.create_dependencies(task)
        if self.has_pending_dependencies(task):
            continue

        # 3. Find instance group for this task
        instance_group = self.get_instance_group(task)
        if not instance_group:
            continue

        # 4. Check if instance group has capacity
        if not self.has_capacity(task, instance_group):
            continue

        # 5. Find best instance in the group
        instance = self.select_instance(task, instance_group)
        if not instance:
            continue

        # 6. Start the task!
        self.start_task(task, instance_group, instance)
```

## start_task() - Launching a Job

**File**: `awx/main/scheduler/task_manager.py:259`

```python
def start_task(self, task, instance_group, instance):
    """
    Start a task on a specific instance.
    Transitions job from pending → waiting and dispatches to worker.
    """

    # 1. Update job status
    task.status = 'waiting'
    task.instance_group = instance_group

    # 2. Call pre_start hook (validation)
    if not task.pre_start():
        task.status = 'failed'
        task.save()
        return

    # 3. Generate unique task ID
    task.celery_task_id = str(uuid4())

    # 4. Assign nodes
    task.controller_node = self.get_controller_node(task, instance_group)
    task.execution_node = instance.hostname

    # 5. Save changes
    task.save(update_fields=[
        'status', 'celery_task_id', 'instance_group',
        'controller_node', 'execution_node'
    ])

    # 6. Emit WebSocket notification
    task.websocket_emit_status('waiting')

    # 7. Dispatch to worker (on DB commit)
    def dispatch_task():
        # Ensure event partition exists
        task.event_class.create_partition(task.pk)

        # Dispatch via PostgreSQL NOTIFY
        task.task_class.apply_async(
            [task.pk],
            opts,
            queue=queue_name
        )

    connection.on_commit(dispatch_task)
```

## Capacity-Based Instance Selection

**File**: `awx/main/scheduler/task_manager_models.py:93`

```python
def fit_task_to_most_remaining_capacity_instance(
    self,
    task,
    instance_group_name,
    impact=None,
    capacity_type=None,
    add_hybrid_control_cost=False
):
    """
    Find the instance with the most remaining capacity
    that can fit this task.
    """
    impact = impact or task.task_impact
    capacity_type = capacity_type or task.capacity_type
    instance_most_capacity = None
    most_remaining_capacity = -1

    instances = self.instance_groups[instance_group_name]['instances']

    for instance in instances:
        # Skip wrong node types
        if instance.node_type not in (capacity_type, 'hybrid'):
            continue

        # Calculate remaining capacity after this task
        would_be_remaining = instance.remaining_capacity - impact

        # Hybrid nodes always control their own tasks
        if add_hybrid_control_cost and instance.node_type == 'hybrid':
            would_be_remaining -= settings.AWX_CONTROL_NODE_TASK_IMPACT

        # Check if this is the best option so far
        if would_be_remaining >= 0:
            if would_be_remaining > most_remaining_capacity:
                instance_most_capacity = instance
                most_remaining_capacity = would_be_remaining

    return instance_most_capacity
```

## Workflow DAG Processing

**File**: `awx/main/scheduler/dag_workflow.py`

For WorkflowJobs, the task manager uses a DAG (Directed Acyclic Graph) to:
1. Track dependencies between workflow nodes
2. Determine which nodes are ready to run
3. Spawn jobs for ready nodes

```python
def spawn_workflow_graph_jobs(self, workflow_jobs):
    """Spawn jobs for workflow nodes that are ready"""

    for workflow_job in workflow_jobs:
        if workflow_job.cancel_flag:
            continue

        # Build DAG from workflow structure
        dag = WorkflowDAG(workflow_job)

        # Find nodes ready to run (BFS traversal)
        spawn_nodes = dag.bfs_nodes_to_run()

        for spawn_node in spawn_nodes:
            if spawn_node.unified_job_template is None:
                continue

            # Create job from the node's template
            kv = spawn_node.get_job_kwargs()
            job = spawn_node.unified_job_template.create_unified_job(**kv)

            # Link job to workflow node
            spawn_node.job = job
            spawn_node.save()

            # Signal the job to start
            job.signal_start()
```

## WorkflowDAG.bfs_nodes_to_run()

```python
def bfs_nodes_to_run(self):
    """
    Breadth-first search to find nodes ready for execution.
    A node is ready when all its parent dependencies are satisfied.
    """
    nodes = self.get_root_nodes()
    nodes_found = []

    for node in nodes:
        obj = node['node_object']

        if obj.do_not_run:
            continue

        if obj.job:
            # Node already has a job
            if obj.job.status == 'successful':
                # Add success children
                nodes.extend(self.get_children(obj, 'success_nodes'))
                nodes.extend(self.get_children(obj, 'always_nodes'))
            elif obj.job.status in ['failed', 'error', 'canceled']:
                # Add failure children
                nodes.extend(self.get_children(obj, 'failure_nodes'))
                nodes.extend(self.get_children(obj, 'always_nodes'))
        else:
            # Node needs a job - check if parents are done
            if self._are_relevant_parents_finished(node):
                if obj.all_parents_must_converge:
                    if self._all_parents_met_convergence_criteria(node):
                        nodes_found.append(node)
                else:
                    nodes_found.append(node)

    return [n['node_object'] for n in nodes_found]
```

## Advisory Lock

The task manager uses a PostgreSQL advisory lock to prevent concurrent runs:

```python
from awx.main.utils.common import advisory_lock

with advisory_lock('task_manager_lock', wait=False) as acquired:
    if not acquired:
        # Another task manager is running
        return
    # Do scheduling...
```

This ensures only one task manager instance is scheduling at a time, even in a multi-node cluster.

## Task Manager Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Task Manager Trigger                          │
│         (Periodic every 20s OR on-demand from signal_start)     │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Acquire Advisory Lock                           │
│              (Only one scheduler runs at a time)                 │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                      get_tasks()                                 │
│   Query all pending/waiting/running jobs from all job types     │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                 Build Capacity Tracking                          │
│   TaskManagerInstances: track consumed capacity per instance    │
│   TaskManagerInstanceGroups: group instances                    │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│              spawn_workflow_graph_jobs()                         │
│   Process running workflows, spawn jobs for ready nodes         │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    process_tasks()                               │
│   For each pending task:                                        │
│   1. Check dependencies                                         │
│   2. Find instance group                                        │
│   3. Check capacity                                             │
│   4. Select best instance                                       │
│   5. start_task() → status=waiting → apply_async()              │
└─────────────────────────────────────────────────────────────────┘
```
