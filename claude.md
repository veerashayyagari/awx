# AWX - Ansible Automation Platform

## Project Overview

AWX is the open-source upstream project for Red Hat Ansible Automation Platform. It provides:

- Web-based UI for managing Ansible automation
- REST API for programmatic access
- Task execution engine for running Ansible playbooks at scale
- Job scheduling, credential management, and role-based access control (RBAC)

## Technology Stack

| Component         | Technology                | Version |
| ----------------- | ------------------------- | ------- |
| Backend           | Django                    | 3.2.13  |
| API               | Django REST Framework     | 3.13.1  |
| Frontend          | React                     | 17.0.2  |
| UI Library        | PatternFly                | 4.x     |
| Database          | PostgreSQL                | 12+     |
| Cache/Broker      | Redis                     | Latest  |
| WebSocket         | Django Channels           | 2.4.0   |
| Task Dispatch     | PostgreSQL LISTEN/NOTIFY  | -       |
| Execution         | ansible-runner + Receptor | Latest  |
| Container Runtime | Podman/Docker             | -       |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    React UI (PatternFly)                     │
│                      awx/ui/src/                             │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP/WebSocket
┌──────────────────────────▼──────────────────────────────────┐
│              Django REST Framework API                       │
│                    awx/api/                                  │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                 Core Application Layer                       │
│                    awx/main/                                 │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│              Job Execution Infrastructure                    │
│         Receptor mesh + ansible-runner + EE containers       │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│           PostgreSQL + Redis Data Layer                      │
└─────────────────────────────────────────────────────────────┘
```

## Directory Structure

```
awx/
├── awx/                          # Main Django application
│   ├── api/                      # REST API (DRF)
│   │   ├── serializers.py        # Model serializers (228KB)
│   │   ├── views/                # API endpoints
│   │   └── urls/                 # URL routing
│   ├── main/                     # Core application
│   │   ├── models/               # Django ORM models (22 files)
│   │   ├── tasks/                # Job execution
│   │   │   ├── jobs.py           # Main job runner (87KB)
│   │   │   ├── receptor.py       # Distributed execution
│   │   │   └── callback.py       # Event processing
│   │   ├── scheduler/            # Task orchestration
│   │   │   └── task_manager.py   # Job scheduling
│   │   ├── dispatch/             # Async task dispatch
│   │   ├── access.py             # RBAC logic (112KB)
│   │   └── migrations/           # 180+ DB migrations
│   ├── conf/                     # Configuration app
│   ├── sso/                      # SSO backends (LDAP, SAML, OAuth)
│   ├── ui/                       # React frontend
│   │   └── src/
│   │       ├── components/       # 71 reusable components
│   │       ├── screens/          # 25+ page components
│   │       └── api/              # API client (51 models)
│   └── settings/                 # Django settings
├── awxkit/                       # Python CLI client
├── awx_collection/               # Ansible collection
├── tools/docker-compose/         # Docker dev environment
└── requirements/                 # Python dependencies
```

## Deep Dive Documentation

For detailed analysis of each component, see the `code_analysis/` folder:

| Document                                                                   | Description                                     |
| -------------------------------------------------------------------------- | ----------------------------------------------- |
| [01-job-execution-overview.md](code_analysis/01-job-execution-overview.md) | High-level flow diagram and status transitions  |
| [02-frontend-launch.md](code_analysis/02-frontend-launch.md)               | React UI launch flow, LaunchButton component    |
| [03-api-layer.md](code_analysis/03-api-layer.md)                           | Django REST API, JobTemplateLaunch view         |
| [04-unified-jobs-model.md](code_analysis/04-unified-jobs-model.md)         | Polymorphic job hierarchy, create_unified_job   |
| [05-task-manager-scheduler.md](code_analysis/05-task-manager-scheduler.md) | Scheduling, capacity, WorkflowDAG               |
| [06-dispatcher.md](code_analysis/06-dispatcher.md)                         | PostgreSQL LISTEN/NOTIFY, @task decorator       |
| [07-job-execution.md](code_analysis/07-job-execution.md)                   | RunJob, BaseTask, Receptor, callbacks           |
| [08-instances-and-capacity.md](code_analysis/08-instances-and-capacity.md) | Instances, InstanceGroups, capacity calculation |

## Key Concepts

### Job Execution Flow (Summary)

```
UI Launch → API → create_unified_job() → signal_start()
         → TaskManager.schedule() → start_task()
         → PostgreSQL NOTIFY → Worker receives
         → RunJob.run() → Receptor → ansible-runner
         → Callbacks → WebSocket → UI updates
```

### Job Status Transitions

```
new → pending → waiting → running → successful/failed/error/canceled
```

### Unified Job Hierarchy

AWX uses polymorphic models:

- `UnifiedJobTemplate` → `UnifiedJob`
  - `JobTemplate` → `Job` (playbook execution)
  - `Project` → `ProjectUpdate` (git sync)
  - `InventorySource` → `InventoryUpdate` (inventory refresh)
  - `WorkflowJobTemplate` → `WorkflowJob` (orchestration)

### Instance Types

| Type      | Control | Execute | Description            |
| --------- | ------- | ------- | ---------------------- |
| control   | Yes     | No      | API, scheduler, web UI |
| execution | No      | Yes     | Run playbooks          |
| hybrid    | Yes     | Yes     | Both (default for dev) |
| hop       | No      | No      | Receptor relay         |

### Task Dispatch

- Uses PostgreSQL LISTEN/NOTIFY (not Celery broker)
- Task Manager runs every 20 seconds OR on-demand
- Capacity-based instance selection

## Development Environment

### Prerequisites

- Docker or Podman
- Make
- Node.js 16.13+ (for UI development)
- Python 3.9+

### Setup Commands

```bash
# Build and start development environment
make docker-compose-build
make docker-compose

# Access the UI
# http://localhost:8013 (HTTP)
# https://localhost:8043 (HTTPS)

# Default credentials in: tools/docker-compose/_sources/secrets/
```

### Common Make Targets

```bash
make docker-compose-build    # Build dev image
make docker-compose          # Start environment
make docker-compose-test     # Interactive shell
make migrate                 # Run migrations
make ui-devel               # Build UI (dev mode)
make test                   # Run all tests
make test_unit              # Unit tests only
make linters                # Run linters
```

## Testing

```bash
# All tests
make test

# Specific test file
py.test awx/main/tests/functional/api/test_job.py

# With markers
py.test -m "ac"              # Access control tests

# UI tests
make ui-test-screens
```

## Code Style

### Python

```bash
make black    # Format code
make linters  # Run all linters
```

### JavaScript

```bash
cd awx/ui && npm run lint
cd awx/ui && npm run prettier
```

## Debugging

### Django Shell

```bash
make docker-compose-test
awx-manage shell_plus
```

### Database Access

```bash
awx-manage dbshell
```

### Common Issues

```bash
# Check migration status
awx-manage showmigrations

# Run pending migrations
awx-manage migrate

# Rebuild UI after changes
make ui-devel
```

## Key Files Reference

### Backend Core

- `awx/main/models/unified_jobs.py` - Polymorphic job hierarchy
- `awx/main/models/jobs.py` - JobTemplate and Job models
- `awx/main/models/ha.py` - Instance, InstanceGroup
- `awx/main/access.py` - Permission checking (112KB)

### Job Execution

- `awx/main/tasks/jobs.py` - BaseTask and job runners (87KB)
- `awx/main/tasks/receptor.py` - Receptor integration
- `awx/main/scheduler/task_manager.py` - Job scheduling
- `awx/main/dispatch/publish.py` - Task publishing

### API

- `awx/api/serializers.py` - All serializers (228KB)
- `awx/api/views/__init__.py:2354` - JobTemplateLaunch

### Frontend

- `awx/ui/src/components/LaunchButton/LaunchButton.js` - Launch flow
- `awx/ui/src/api/models/JobTemplates.js` - API client

## API Endpoints

Base URL: `/api/v2/`

Key endpoints:

- `/api/v2/job_templates/{id}/launch/` - Launch a job
- `/api/v2/jobs/{id}/` - Job details
- `/api/v2/jobs/{id}/stdout/` - Job output
- `/api/v2/jobs/{id}/cancel/` - Cancel job

## References

- [AWX GitHub](https://github.com/ansible/awx)
- [AWX Operator](https://github.com/ansible/awx-operator)
- [Ansible Runner](https://github.com/ansible/ansible-runner)
- [Receptor](https://github.com/ansible/receptor)
