# Ansible + AWX Terminology (with Example)

This document defines the core concepts you will see when running playbooks in AWX and ties them to the AWX UI and database.

## What to take away

- A **project** provides playbooks, an **inventory** provides hosts, and a **job template** combines them.
- A **job** is a single execution of a job template.
- AWX schedules jobs to **instances** grouped by **instance groups** (including container groups).

## Core terms (from ansible to AWX)

| Term | Meaning in AWX |
|------|----------------|
| **Project** | A source of playbooks (usually a git repo). |
| **Playbook** | A YAML file inside the project (e.g., `site.yml`). |
| **Inventory** | A collection of hosts. |
| **Host** | A managed node (machine, VM, container, etc.). |
| **Group** (Inventory Group) | A named set of hosts inside an inventory. |
| **Inventory Source** | External inventory integration (cloud, SCM, script, plugin). |
| **Inventory Update** | A job that syncs an inventory source. |
| **Credential** | Secret inputs for auth (SSH, cloud API, vault, etc.). |
| **Credential Type** | Defines inputs and how they are injected. |
| **Execution Environment (EE)** | A container image with ansible-runner + collections. |
| **Job Template (JT)** | Saved launch config (project + inventory + playbook + prompts). |
| **Job** | A single run of a job template. |
| **Workflow** | A chain of jobs with success/failure logic. |
| **Schedule** | Time-based trigger for a job template or workflow. |
| **Instance** | A node that can execute jobs (control/execution/hybrid). |
| **Instance Group** | A scheduling target (queue of instances). |
| **Container Group** | Instance group that spawns ephemeral K8s pods per job. |
| **Control Plane** | API, scheduler, dispatcher, websocket services. |
| **Execution Plane** | Where ansible-runner executes jobs. |

## AWX-specific terms you will see in code

| Term | Where it shows up |
|------|--------------------|
| **Task Manager** | Scheduler that assigns jobs to instances. |
| **Dispatcher** | Publishes tasks via Postgres NOTIFY. |
| **Callback Receiver** | Persists ansible-runner events into `JobEvent` tables. |
| **Job Events** | Structured records of playbook output and event data. |
| **Survey** | User prompts (extra vars) attached to a job template. |

## Example: "Deploy a Web App"

### 1) Project

- Git repo: `https://example.com/org/webapp-ops.git`
- Playbooks: `deploy.yml`, `rollback.yml`

In AWX: create a **Project** pointing to that repo.

### 2) Inventory and Groups

- Inventory: `Prod`
- Hosts: `web-1`, `web-2`
- Group: `web`

In AWX: create an **Inventory**, add **Hosts**, then group them under `web`.

### 3) Credentials

- SSH key to access the hosts
- Optional vault credential for secrets

In AWX: create **Credential** objects and associate them with the job template.

### 4) Job Template

Create a **Job Template**:

- Project: `webapp-ops`
- Inventory: `Prod`
- Playbook: `deploy.yml`
- Credentials: `Prod SSH`, `Vault`
- Prompts: allow `limit` and `extra_vars` on launch

### 5) Launch a Job

When you click Launch:

1. UI calls `GET /api/v2/job_templates/{id}/launch/` to gather prompt data.
2. UI posts `POST /api/v2/job_templates/{id}/launch/`.
3. A **Job** row is created (`main_job` + `main_unifiedjob`).
4. Task manager assigns an **Instance Group**, then an **Instance**.
5. Dispatcher sends a task; the worker runs `ansible-runner`.
6. Events are stored as `JobEvent` rows and streamed to the UI.

### 6) Outputs You See

- Job status: `pending -> waiting -> running -> successful`
- Live output in the Job Output view (via websocket)
- Event details per task and host

## Quick mapping: UI to tables

| UI Concept | Primary Table(s) |
|------------|------------------|
| Project | `main_project` |
| Inventory | `main_inventory` |
| Host | `main_host` |
| Group | `main_group` |
| Job Template | `main_jobtemplate` |
| Job | `main_job`, `main_unifiedjob` |
| Job Events | `main_jobevent` |
| Instance | `main_instance` |
| Instance Group | `main_instancegroup` |

## Next steps

- Walk the end-to-end flow: `deepdive/01-job-execution-overview.md`.
- See storage details: `deepdive/09-storage-schema.md`.
