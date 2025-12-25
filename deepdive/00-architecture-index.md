# AWX Architecture Map

This page orients new contributors to the control-plane flow and how the other deep-dive notes fit together. Use the diagrams as a road map, then jump into the linked files for code-level detail.

## High-level system map
```mermaid
graph TD
  subgraph UI
    FE[React SPA (awx/ui)]
  end
  subgraph API
    Django[awx/urls.py\nDRF views]
  end
  subgraph Control
    TM[TaskManager\nawx/main/scheduler/task_manager.py]
    Dispatch[Dispatcher\nawx/main/dispatch]
  end
  subgraph Execution
    Worker[awx/main/tasks/jobs.py\ncelery worker]
    Runner[ansible-runner / receptor]
  end
  subgraph Data & Events
    DB[(PostgreSQL)]
    WS[(Websocket broadcasts)]
  end

  FE -->|HTTP| Django
  Django --> TM
  TM --> Dispatch
  Dispatch --> Worker
  Worker --> Runner
  Runner -->|job events| WS
  WS --> FE
  Django --> DB
  TM --> DB
  Worker --> DB
```

### How to read the diagram
- **Boxes** represent concrete entry points in the repo (files or modules) instead of abstract components.
- **Arrows** are the primary runtime calls: HTTP into Django/DRF, scheduling through the task manager and dispatcher, job execution through celery workers and ansible-runner, and websocket notifications back to the UI.
- The **Data & Events** cluster shows which steps persist state or emit realtime updates.

## Flow-of-control drill-down
```mermaid
sequenceDiagram
  participant UI as React UI
  participant API as DRF views
  participant Model as UnifiedJob models
  participant TM as TaskManager
  participant Q as Dispatch queue (PG listen/notify)
  participant W as Celery worker
  participant R as Runner/receptor
  participant WS as Websocket

  UI->>API: POST /api/v2/job_templates/{id}/launch/
  API->>Model: create_unified_job() & signal_start()
  Model->>TM: schedule_task_manager (on_commit)
  TM->>Q: start_task() → apply_async()
  Q-->>W: LISTEN/NOTIFY payload
  W->>R: ansible-runner / receptor
  R-->>WS: job events & status
  WS-->>UI: live updates
```

- **Launch** begins in `awx/api/views/__init__.py::JobTemplateLaunch.post` and immediately writes a `UnifiedJob` row.
- **Signal dispatch** happens via `awx/main/utils/common.py::schedule_task_manager`, which uses `connection.on_commit` so scheduling only fires after the DB write is durable.
- **Scheduling** logic in `task_manager.schedule()` assigns the job to an instance group/instance and publishes to the dispatcher queue.
- **Execution** is performed by celery workers in `awx/main/tasks/jobs.py`, which call ansible-runner or receptor and stream callback events.
- **Notifications** are emitted from `UnifiedJob.websocket_emit_status` and callback handlers so the UI updates without polling.

## Where to go next
- [Job execution overview](./01-job-execution-overview.md) — annotated path from user click to websocket updates.
- [Frontend launch flow](./02-frontend-launch.md) — how the React UI gathers prompts and issues the launch API requests.
- [API layer](./03-api-layer.md) — DRF view/serializer details for launches and prompts.
- [Unified jobs model](./04-unified-jobs-model.md) — persistence and status transitions.
- [Scheduler](./05-task-manager-scheduler.md) — task_manager internals and dependency handling.
- [Dispatcher](./06-dispatcher.md) — queue publication and worker consumption.
- [Job execution](./07-job-execution.md) — runner integration, callback pipeline, event storage.
- [Instances & capacity](./08-instances-and-capacity.md) — instance/instance-group capacity models used by the scheduler.
