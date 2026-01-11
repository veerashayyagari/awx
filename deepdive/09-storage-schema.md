# Storage Schema (PostgreSQL + Redis)

This document is a storage map for AWX. It focuses on how data is stored and how to discover the complete schema for your environment. It is intentionally a curated overview, not a full schema dump.

## What to take away

- PostgreSQL is the system of record; schema is defined by Django models and migrations.
- Redis is ephemeral and used for queues, websocket channels, metrics, and caching.
- For the authoritative, current column list use `psql` introspection or Django model definitions.

## PostgreSQL overview

AWX uses the `public` schema and the Django ORM. Table names follow the Django default:

- `main_*`: core AWX domain models (`awx/main/models/*`)
- `conf_*`: dynamic settings (`awx/conf/models.py`)
- `auth_*`, `django_*`, `contenttypes_*`, `sessions`: standard Django tables

Foreign keys are stored as `<field>_id` columns. Many-to-many relationships are stored in auto-generated join tables.

## Full schema inspection (recommended)

Use these commands against a running AWX database to list every table and column:

```bash
awx-manage dbshell
```

```sql
-- list all tables
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
ORDER BY table_name;

-- list all columns for all tables
SELECT table_name, column_name, data_type, is_nullable, column_default
FROM information_schema.columns
WHERE table_schema = 'public'
ORDER BY table_name, ordinal_position;

-- inspect a single table with comments and indexes
\d+ main_job
```

For model-level source of truth, check `awx/main/models/` and `awx/conf/models.py`. Many fields have `help_text` on the model that explains the column meaning.

## Core table map (by concern)

| Area | Tables | Notes |
|------|--------|-------|
| Jobs (templates + runs) | `main_unifiedjobtemplate`, `main_unifiedjob`, `main_jobtemplate`, `main_job` | Unified job types share a base schema |
| Job events | `main_jobevent` (+ per-job partitions if enabled) | High-volume event data from ansible-runner |
| Projects | `main_project`, `main_projectupdate` | SCM configuration and update runs |
| Inventory | `main_inventory`, `main_host`, `main_group`, `main_inventorysource`, `main_inventoryupdate` | Hosts, groups, and inventory syncs |
| Credentials | `main_credential`, `main_credentialtype` | Encrypted inputs and injectors |
| Scheduling | `main_schedule` | Periodic launches |
| Instances | `main_instance`, `main_instancegroup` | Capacity and node membership |
| Workflows | `main_workflowjobtemplate`, `main_workflowjob`, `main_workflowjobnode` | Workflow definitions and runs |
| Settings | `conf_setting` | Dynamic configuration stored in DB |

## Column guide (core tables)

This section lists the most important columns for each core table. Use `\d+` to see the full column list.

### `main_unifiedjob`

Core runtime job record for all job types.

| Column | Meaning |
|--------|---------|
| `id` | Primary key |
| `name` | Display name |
| `description` | Human description |
| `unified_job_template_id` | FK to the template used (if any) |
| `status` | `new`, `pending`, `waiting`, `running`, `successful`, `failed`, `error`, `canceled` |
| `launch_type` | `manual`, `scheduled`, `workflow`, `relaunch`, etc. |
| `created` | Row creation timestamp |
| `started` | When job started running |
| `finished` | When job ended |
| `canceled_on` | When cancel request was issued |
| `elapsed` | Seconds runtime |
| `execution_node` | Execution node hostname |
| `controller_node` | Control node hostname |
| `instance_group_id` | Instance group chosen by scheduler |
| `organization_id` | Org used for RBAC checks |
| `job_args` | CLI args for the runner process |
| `job_cwd` | Working directory |
| `job_env` | JSON env for execution |
| `start_args` | JSON-ish blob used for relaunch |
| `result_traceback` | Error traceback (if any) |
| `work_unit_id` | Receptor work unit ID (when applicable) |

### `main_job`

Concrete playbook execution row; extends `main_unifiedjob`.

| Column | Meaning |
|--------|---------|
| `job_template_id` | FK to `main_jobtemplate` |
| `inventory_id` | Inventory selected for the run |
| `project_id` | Project used for playbooks |
| `playbook` | Playbook path relative to project |
| `scm_revision` | Project revision used |
| `project_update_id` | FK to `main_projectupdate` used as dependency |
| `limit` | Host limit string |
| `job_tags` / `skip_tags` | Tag filters |
| `verbosity` | Ansible verbosity level |
| `forks` | Fork count |
| `execution_environment_id` | EE used for execution |
| `job_slice_number` / `job_slice_count` | Slice metadata for sliced jobs |

### `main_jobtemplate`

Reusable launch configuration for jobs.

| Column | Meaning |
|--------|---------|
| `project_id` | Project that provides playbooks |
| `inventory_id` | Default inventory |
| `job_type` | `run` or `check` |
| `playbook` | Playbook path |
| `ask_*_on_launch` | Prompt flags (inventory, credentials, vars, tags, etc.) |
| `job_slice_count` | Desired slice count for large inventories |

### `main_jobevent`

Event stream records from ansible-runner.

| Column | Meaning |
|--------|---------|
| `id` | Primary key |
| `job_id` | FK to `main_job` |
| `created` | Event timestamp |
| `event` | Event type (e.g., `runner_on_ok`) |
| `counter` | Monotonic event counter |
| `uuid` | Event UUID |
| `parent_uuid` | Parent event UUID |
| `host_id` / `host_name` | Host identity |
| `event_data` | JSON event payload |
| `stdout` | Rendered stdout for the event |

### `main_project`

Project metadata and SCM settings.

| Column | Meaning |
|--------|---------|
| `organization_id` | Owning org |
| `scm_type` | `git`, `svn`, etc. |
| `scm_url` | SCM repository URL |
| `scm_branch` | Branch/ref (default) |
| `scm_revision` | Last synced revision |
| `scm_update_on_launch` | Auto-update on launch |
| `allow_override` | Allow JT to override SCM ref |
| `default_environment_id` | Default execution environment |

### `main_inventory`

Inventory container for hosts and groups.

| Column | Meaning |
|--------|---------|
| `organization_id` | Owning org |
| `kind` | `''` (normal) or `smart` |
| `host_filter` | Smart inventory host filter |
| `variables` | JSON/YAML inventory vars |

### `main_host`

Managed host in an inventory.

| Column | Meaning |
|--------|---------|
| `inventory_id` | Owning inventory |
| `enabled` | Host is available for runs |
| `instance_id` | External inventory unique ID |
| `variables` | JSON/YAML host vars |
| `ansible_facts` | Latest collected facts |
| `last_job_id` | Most recent job touching host |

### `main_instance`

Runtime node record used by the scheduler.

| Column | Meaning |
|--------|---------|
| `uuid` | Instance UUID |
| `hostname` | Node hostname (matches `CLUSTER_HOST_ID`) |
| `ip_address` | Node IP |
| `node_type` | `control`, `execution`, `hybrid`, `hop` |
| `capacity` | Effective capacity |
| `cpu_capacity` / `mem_capacity` | Raw capacity values |
| `enabled` | Scheduler eligibility |
| `last_seen` | Last heartbeat |

### `main_instancegroup`

Execution group; contains instances or container group metadata.

| Column | Meaning |
|--------|---------|
| `name` | Queue/group name |
| `is_container_group` | True for container groups |
| `credential_id` | K8s credential for container group |
| `pod_spec_override` | YAML/JSON pod spec override |
| `policy_instance_*` | Auto-membership policy fields |

### `conf_setting`

Dynamic settings stored in the DB.

| Column | Meaning |
|--------|---------|
| `key` | Setting name |
| `value` | JSON value |
| `created` / `modified` | Audit timestamps |

## Redis usage

Redis is used as a message bus, cache, and metrics store. Keys are generally ephemeral.

### Callback queue (job events)

- `callback_tasks` (list): Event payloads pushed by `CallbackQueueDispatcher` (`awx/main/queue.py`).
- Consumers pop with `BLPOP` in the callback receiver (`awx/main/dispatch/worker/callback.py`).

### Dispatcher and callback receiver stats

- `awx_dispatcher_statistics` (string): Worker pool debug output.
- `awx_callback_receiver_statistics_<pid>` (string): Callback receiver worker stats.

### Subsystem metrics

- `awx_metrics` (hash): Prometheus-compatible metric fields.
- `awx_metrics_instance_<hostname>` (string): Per-instance JSON metrics blob.

See `awx/main/analytics/subsystem_metrics.py` and `docs/subsystem_metrics.md`.

### Websocket channels (channels_redis)

AWX uses Django Channels with Redis as the backend:

- `asgi:group:<group>` (ZSET): Websocket group membership.
- `asgi:channel:<channel>` (list): Per-channel message queue.
- `asgispecific.*` keys: Internal channels_redis structures.

Keys are managed by `channels_redis`; do not write to them directly. See `docs/debugging/debugging_job_event_performance.md`.

### Django cache

The Django cache is configured with Redis DB 1:

- `CACHES['default']` in `awx/settings/defaults.py`
- Used for settings cache invalidation and other short-lived data.

Cache keys are namespaced and managed by `django-redis`.
