# Job Execution Overview

This document provides an end-to-end view of how a job executes in AWX, from the React launch button to websocket updates. Use it with the [architecture map](./00-architecture-index.md) to navigate the deeper drill-downs.

## Complete Flow Diagram

```mermaid
flowchart TB
  User([User click Launch]) --> FE
  subgraph Frontend
    FE[LaunchButton\nawx/ui/src/components/LaunchButton]
  end
  FE -->|POST /api/v2/job_templates/{id}/launch/| API
  subgraph API/Model
    API[JobTemplateLaunch\nawx/api/views/__init__.py]
    Model[UnifiedJob create + signal_start\nawx/main/models/unified_jobs.py]
  end
  API --> Model
  Model -->|schedule_task_manager (on commit)| Sched
  subgraph Scheduler
    Sched[task_manager.schedule\nawx/main/scheduler/task_manager.py]
  end
  Sched -->|start_task → apply_async| Dispatch
  subgraph Dispatcher
    Dispatch[publish.apply_async\nPostgreSQL LISTEN/NOTIFY]
  end
  Dispatch --> Worker
  subgraph Execution
    Worker[celery worker\nawx/main/tasks/jobs.py]
    Runner[ansible-runner / receptor]
  end
  Worker --> Runner
  Runner -->|events + status| WS
  subgraph Events
    WS[emit_channel_notification\nwebsocket broadcasts]
  end
  WS --> FE
```

### Code checkpoints
- **Launch entrypoint**: `JobTemplateLaunch.post()` validates prompts and permissions before calling `create_unified_job()` and returning the new job identifier.
- **Durable scheduling**: `schedule_task_manager()` uses `connection.on_commit` so the scheduler only runs after the new job row is committed.
- **Placement**: `task_manager.schedule()` locks, gathers runnable jobs, assigns an instance group/instance, then publishes via `start_task()` and `publish.apply_async()`.
- **Execution**: celery workers execute `RunJob` (or receptor variants) from `awx/main/tasks/jobs.py`, writing events as ansible-runner streams callbacks.
- **Realtime updates**: `UnifiedJob.websocket_emit_status()` and callback handlers emit `jobs-status_changed` notifications so the UI updates without polling.

## Job Status Transitions

```
new → pending → waiting → running → successful
                    │         │          │
                    │         │          └→ failed
                    │         │          └→ error
                    │         └→ canceled
                    └→ canceled
```

| Status | Meaning |
|--------|---------|
| `new` | Job record created, not yet submitted for scheduling |
| `pending` | Job submitted to task manager, waiting to be scheduled |
| `waiting` | Job assigned to a node, waiting for worker to pick it up |
| `running` | Ansible playbook is executing |
| `successful` | Playbook completed successfully |
| `failed` | Playbook failed (Ansible returned non-zero) |
| `error` | AWX-level error (e.g., can't connect to node) |
| `canceled` | User or system canceled the job |

## Detailed Component Documentation

- [Frontend Launch Flow](./02-frontend-launch.md)
- [API Layer](./03-api-layer.md)
- [Unified Jobs Model](./04-unified-jobs-model.md)
- [Task Manager & Scheduler](./05-task-manager-scheduler.md)
- [Dispatcher](./06-dispatcher.md)
- [Job Execution](./07-job-execution.md)
- [Instances & Capacity](./08-instances-and-capacity.md)
