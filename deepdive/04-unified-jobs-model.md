# Unified Jobs Model

This document details AWX's polymorphic job model system.

## What to take away

- UnifiedJob/UnifiedJobTemplate are abstract bases; concrete job types inherit and override `task_class`.
- The launch path copies template fields into a new job row, then signals scheduling.
- Relaunch behavior is stored in `JobLaunchConfig` plus `start_args`.

## Key Files

| File | Purpose |
|------|---------|
| `awx/main/models/unified_jobs.py` | Core UnifiedJob and UnifiedJobTemplate models |
| `awx/main/models/jobs.py` | Job and JobTemplate (playbook execution) |
| `awx/main/models/projects.py` | Project and ProjectUpdate (SCM sync) |
| `awx/main/models/inventory.py` | InventorySource and InventoryUpdate |
| `awx/main/models/ad_hoc_commands.py` | AdHocCommand |
| `awx/main/models/workflow.py` | WorkflowJob and WorkflowJobTemplate |
| `awx/main/models/system.py` | SystemJob (cleanup, etc.) |

## Polymorphic Model Hierarchy

AWX uses Django's model inheritance for a unified job system:

```
UnifiedJobTemplate (abstract base)
├── JobTemplate          → creates → Job
├── Project              → creates → ProjectUpdate
├── InventorySource      → creates → InventoryUpdate
├── WorkflowJobTemplate  → creates → WorkflowJob
└── SystemJobTemplate    → creates → SystemJob

UnifiedJob (abstract base)
├── Job                  (playbook execution)
├── ProjectUpdate        (git clone/pull)
├── InventoryUpdate      (dynamic inventory sync)
├── WorkflowJob          (workflow orchestration)
├── SystemJob            (cleanup tasks)
└── AdHocCommand         (one-off commands)
```

## UnifiedJobTemplate Base

**File**: `awx/main/models/unified_jobs.py`

```python
class UnifiedJobTemplate(PolymorphicModel, ...):
    """Base model for things that can create jobs"""

    name = models.CharField(max_length=512)
    description = models.TextField(blank=True, default='')
    organization = models.ForeignKey(Organization, ...)

    # What type of job does this create?
    @property
    def unified_job_class(self):
        raise NotImplementedError  # Subclasses override

    def create_unified_job(self, **kwargs):
        """Create a job instance from this template"""
        unified_job_class = self.unified_job_class
        fields = self._get_unified_job_field_names()

        # Copy fields from template to job
        for field_name in fields:
            if field_name not in kwargs:
                kwargs[field_name] = getattr(self, field_name)

        # Create the job
        new_job = unified_job_class(**kwargs)
        new_job.save()

        # Copy M2M relationships (credentials, labels, etc.)
        self._copy_m2m_relationships(new_job)

        return new_job
```

### Why Polymorphic Models Matter

`UnifiedJob` lets the scheduler and APIs work with a single "job" shape while still supporting specialized types. That is why many queries are against `UnifiedJob`/`UnifiedJobTemplate`, even if the concrete class is `Job` or `WorkflowJob`.

## UnifiedJob Base

```python
class UnifiedJob(PolymorphicModel, ...):
    """Base model for all job types"""

    # Template reference
    unified_job_template = models.ForeignKey(
        UnifiedJobTemplate,
        null=True,
        on_delete=models.SET_NULL
    )

    # Status tracking
    status = models.CharField(
        max_length=20,
        choices=JOB_STATUS_CHOICES,
        default='new'
    )
    started = models.DateTimeField(null=True)
    finished = models.DateTimeField(null=True)
    elapsed = models.DecimalField(...)

    # Node assignment
    execution_node = models.TextField(blank=True, default='')
    controller_node = models.TextField(blank=True, default='')

    # Capacity
    task_impact = models.PositiveIntegerField(default=0)

    # What task class runs this job?
    @property
    def task_class(self):
        raise NotImplementedError  # Subclasses override
```

### Template vs Job Fields

Two fields commonly confused:
- `unified_job_template` lives on every job type and points back to its template.
- `job_template` is a concrete field only on `Job` (playbook execution).

## create_unified_job() Deep Dive

**File**: `awx/main/models/unified_jobs.py:334`

```python
def create_unified_job(self, **kwargs):
    """
    Creates a job from this template.
    Called by the API when user launches a job.
    """

    # 1. Get the concrete job class (Job, ProjectUpdate, etc.)
    unified_job_class = self.unified_job_class

    # 2. Determine which fields to copy
    fields = self._get_unified_job_field_names()

    # 3. Copy template fields to new job
    unprocessed_kwargs = {}
    for field_name in fields:
        if field_name in kwargs:
            continue  # User provided this value
        # Copy from template
        kwargs[field_name] = getattr(self, field_name)

    # 4. Handle survey password encryption
    if 'extra_vars' in kwargs:
        kwargs['extra_vars'] = self._encrypt_survey_passwords(
            kwargs['extra_vars']
        )

    # 5. Create and save the job
    new_job = unified_job_class(**kwargs)
    new_job.unified_job_template = self
    new_job.save()

    # 6. Copy M2M relationships
    self._copy_m2m_relationships(new_job)
    # Copies: credentials, labels, instance_groups

    # 7. Create JobLaunchConfig for relaunch
    JobLaunchConfig.objects.create(job=new_job, ...)

    # 8. Activity stream entry
    self.emit_activity_stream_entry('launch', new_job)

    return new_job
```

### Relaunch Data

`JobLaunchConfig` and `start_args` are persisted so relaunch endpoints can replay launch parameters without re-prompting users.

## signal_start() Deep Dive

**File**: `awx/main/models/unified_jobs.py:1342`

```python
def signal_start(self, **kwargs):
    """
    Called after job creation to start execution.
    Transitions job from 'new' to 'pending' and triggers scheduler.
    """

    # 1. Validate job can start
    if not self.can_start:
        return False

    # 2. Check passwords
    needed_passwords = self.passwords_needed_to_start
    for pw in needed_passwords:
        if pw not in kwargs:
            return False

    # 3. Save start arguments (for relaunch)
    self.start_args = json.dumps(kwargs)

    # 4. Transition to pending
    self.status = 'pending'
    self.save(update_fields=['status', 'start_args'])

    # 5. Emit WebSocket notification
    self.websocket_emit_status('pending')

    # 6. Trigger the task manager
    from awx.main.utils.common import schedule_task_manager
    schedule_task_manager()

    return True
```

## Job Status Choices

```python
JOB_STATUS_CHOICES = [
    ('new', 'New'),                  # Just created
    ('pending', 'Pending'),          # Waiting for scheduler
    ('waiting', 'Waiting'),          # Assigned to node, waiting for worker
    ('running', 'Running'),          # Executing
    ('successful', 'Successful'),    # Completed OK
    ('failed', 'Failed'),            # Ansible failed
    ('error', 'Error'),              # AWX error
    ('canceled', 'Canceled'),        # User canceled
]
```

## Concrete Job Types

### Job (Playbook Execution)

**File**: `awx/main/models/jobs.py`

```python
class JobTemplate(UnifiedJobTemplate):
    playbook = models.CharField(max_length=1024)
    inventory = models.ForeignKey(Inventory, ...)
    project = models.ForeignKey(Project, ...)
    # ... many more fields

    @property
    def unified_job_class(self):
        return Job


class Job(UnifiedJob):
    job_template = models.ForeignKey(JobTemplate, ...)
    playbook = models.CharField(max_length=1024)
    # ...

    @property
    def task_class(self):
        from awx.main.tasks.jobs import RunJob
        return RunJob
```

## Adding a New Job Type (Checklist)

1. Create a concrete template and job model that inherit the unified bases.
2. Implement `unified_job_class` on the template and `task_class` on the job.
3. Add serializer and view endpoints if user-launchable.
4. Ensure scheduler `get_tasks()` includes the new type.

### ProjectUpdate (SCM Sync)

```python
class Project(UnifiedJobTemplate):
    scm_type = models.CharField(...)  # git, svn, etc.
    scm_url = models.CharField(...)

    @property
    def unified_job_class(self):
        return ProjectUpdate


class ProjectUpdate(UnifiedJob):
    @property
    def task_class(self):
        from awx.main.tasks.jobs import RunProjectUpdate
        return RunProjectUpdate
```

### WorkflowJob

```python
class WorkflowJobTemplate(UnifiedJobTemplate):
    # Contains workflow nodes (WorkflowJobTemplateNode)

    @property
    def unified_job_class(self):
        return WorkflowJob


class WorkflowJob(UnifiedJob):
    # No task_class - handled by WorkflowDAG in task_manager
    pass
```

## WebSocket Notifications

**File**: `awx/main/models/unified_jobs.py:1255`

```python
def websocket_emit_status(self, status):
    """Emit job status change to WebSocket subscribers"""
    status_data = {
        'group_name': 'jobs',
        'type': 'job',
        'id': self.id,
        'status': status,
        'unified_job_id': self.pk,
        # ... more fields
    }

    # Emit to 'jobs-status_changed' channel
    emit_channel_notification('jobs-status_changed', status_data)

    # If this is part of a workflow, also emit to workflow channel
    if hasattr(self, 'workflow_job_id') and self.workflow_job_id:
        emit_channel_notification(
            f'workflow_events-{self.workflow_job_id}',
            status_data
        )
```

## Job Dependencies

Jobs can have dependencies that must complete before they run:

```python
def get_jobs_fail_chain(self):
    """Get list of jobs that would fail if this job fails"""
    # Used by task_manager to track dependency chains

@property
def dependent_jobs(self):
    """Jobs that depend on this job"""
    # e.g., Job depends on ProjectUpdate completing first
```

Common dependency patterns:
- **Job** depends on **ProjectUpdate** (project must be synced)
- **Job** depends on **InventoryUpdate** (inventory must be refreshed)
- **WorkflowJob nodes** depend on parent nodes in the DAG
