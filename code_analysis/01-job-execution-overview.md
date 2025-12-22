# Job Execution Overview

This document provides a high-level overview of how a job executes in AWX, from user click to completion.

## Complete Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              USER ACTION                                      │
│                         Click "Launch" Button                                 │
└─────────────────────────────────────────┬────────────────────────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  FRONTEND (React)                                                             │
│  awx/ui/src/components/LaunchButton/LaunchButton.js                          │
│                                                                               │
│  handleLaunch() → JobTemplatesAPI.launch(id, params)                         │
│                   POST /api/v2/job_templates/{id}/launch/                    │
└─────────────────────────────────────────┬────────────────────────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  API LAYER (Django REST Framework)                                           │
│  awx/api/views/__init__.py:2354 - JobTemplateLaunch                          │
│                                                                               │
│  post() → validate → create_unified_job() → signal_start()                   │
│         → Return 201 {job_id, status, ...}                                   │
└─────────────────────────────────────────┬────────────────────────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  MODEL LAYER                                                                  │
│  awx/main/models/unified_jobs.py                                             │
│                                                                               │
│  create_unified_job():334 → Copy template → Save Job (status=new)            │
│  signal_start():1342 → status=pending → websocket_emit → schedule_task_mgr   │
└─────────────────────────────────────────┬────────────────────────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  SCHEDULER                                                                    │
│  awx/main/scheduler/task_manager.py                                          │
│                                                                               │
│  schedule():708 → Lock → process_pending_tasks() → start_task()              │
│  start_task():259 → status=waiting → assign node → apply_async()             │
└─────────────────────────────────────────┬────────────────────────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  DISPATCHER                                                                   │
│  awx/main/dispatch/publish.py                                                │
│                                                                               │
│  apply_async() → pg_bus_conn.notify(queue, message)                          │
│               → PostgreSQL LISTEN/NOTIFY                                     │
│               → Worker pool receives message                                 │
└─────────────────────────────────────────┬────────────────────────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  EXECUTION                                                                    │
│  awx/main/tasks/jobs.py - RunJob.run()                                       │
│                                                                               │
│  run():397 → status=running → build context → inject credentials             │
│           → AWXReceptorJob.run() OR ansible_runner.run()                     │
│           → Ansible playbook executes                                        │
│           → status=successful/failed → cleanup                               │
└─────────────────────────────────────────┬────────────────────────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  CALLBACKS                                                                    │
│  awx/main/tasks/callback.py - RunnerCallback                                 │
│                                                                               │
│  For each Ansible event:                                                     │
│  event_handler():70 → process → dispatcher.dispatch() → Save JobEvent       │
│                     → WebSocket emit (rate-limited)                          │
└─────────────────────────────────────────┬────────────────────────────────────┘
                                          │
                                          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  WEBSOCKET                                                                    │
│  awx/main/models/unified_jobs.py:1255                                        │
│                                                                               │
│  websocket_emit_status() → emit_channel_notification('jobs-status_changed')  │
│                          → Frontend receives real-time updates               │
└──────────────────────────────────────────────────────────────────────────────┘
```

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
