# Job Execution

This document details how AWX executes Ansible playbooks.

## Key Files

| File | Purpose |
|------|---------|
| `awx/main/tasks/jobs.py` | RunJob, BaseTask, and other task classes |
| `awx/main/tasks/receptor.py` | Receptor integration for distributed execution |
| `awx/main/tasks/callback.py` | RunnerCallback for event handling |
| `awx/main/queue.py` | CallbackQueueDispatcher |

## Task Class Hierarchy

```
BaseTask (abstract)
├── RunJob              - Execute playbooks
├── RunProjectUpdate    - Git clone/pull
├── RunInventoryUpdate  - Dynamic inventory sync
├── RunAdHocCommand     - One-off commands
└── RunSystemJob        - Cleanup tasks
```

## BaseTask.run() - Main Entry Point

**File**: `awx/main/tasks/jobs.py:397`

```python
class BaseTask:
    """Base class for all executable tasks"""

    model = None        # e.g., Job
    event_model = None  # e.g., JobEvent

    def run(self, pk, **kwargs):
        """Main execution entry point"""

        # 1. Load job from database
        self.instance = self.model.objects.get(pk=pk)

        # 2. Set status to running
        self.instance.status = 'running'
        self.instance.started = now()
        self.instance.save()
        self.instance.websocket_emit_status('running')

        # 3. Send start notifications
        self.instance.send_notification_templates('running')

        try:
            # 4. Build execution context
            self.private_data_dir = self.build_private_data_dir()

            # 5. Run pre-execution hooks
            self.pre_run_hook(self.instance)

            # 6. Build all execution artifacts
            self.build_private_data_files(self.instance, self.private_data_dir)
            passwords = self.build_passwords(self.instance, kwargs)
            self.build_extra_vars_file(self.instance, self.private_data_dir)
            args = self.build_args(self.instance, self.private_data_dir)
            env = self.build_env(self.instance, self.private_data_dir)

            # 7. Inject credentials
            for credential in self.instance.credentials.all():
                credential.credential_type.inject_credential(
                    credential, env, passwords
                )

            # 8. Execute via Receptor or directly
            status = self.run_playbook(
                args, env, passwords, self.private_data_dir
            )

        except Exception as e:
            status = 'error'
            self.instance.result_traceback = traceback.format_exc()

        finally:
            # 9. Finalize
            self.instance.status = status
            self.instance.finished = now()
            self.instance.save()

            # 10. Post-execution hooks
            self.post_run_hook(self.instance, status)

            # 11. Send completion notifications
            self.instance.websocket_emit_status(status)
            self.instance.send_notification_templates(status)

            # 12. Cleanup
            self.final_run_hook(self.instance, status)
```

## RunJob - Playbook Execution

**File**: `awx/main/tasks/jobs.py:597`

```python
@task(queue=get_local_queuename)
class RunJob(BaseTask):
    """Run a job using ansible-playbook"""

    model = Job
    event_model = JobEvent

    def build_args(self, job, private_data_dir):
        """Build ansible-playbook command arguments"""
        args = ['ansible-playbook']

        # Playbook path
        playbook_path = os.path.join(
            job.project.get_project_path(),
            job.playbook
        )
        args.append(playbook_path)

        # Inventory
        args.extend(['-i', self.build_inventory(job, private_data_dir)])

        # Limit
        if job.limit:
            args.extend(['--limit', job.limit])

        # Verbosity
        if job.verbosity:
            args.append('-' + 'v' * job.verbosity)

        # Tags
        if job.job_tags:
            args.extend(['--tags', job.job_tags])

        if job.skip_tags:
            args.extend(['--skip-tags', job.skip_tags])

        # Forks
        if job.forks:
            args.extend(['--forks', str(job.forks)])

        return args

    def build_env(self, job, private_data_dir):
        """Build environment variables for ansible-playbook"""
        env = os.environ.copy()

        # AWX-specific vars
        env['AWX_PRIVATE_DATA_DIR'] = private_data_dir
        env['ANSIBLE_CALLBACK_PLUGINS'] = '...'  # AWX callback

        # Project-specific env vars
        if job.project.custom_virtualenv:
            env['VIRTUAL_ENV'] = job.project.custom_virtualenv

        return env
```

## Receptor Execution

For distributed execution, jobs are submitted to Receptor mesh.

**File**: `awx/main/tasks/receptor.py`

```python
class AWXReceptorJob:
    """
    Execute job via Receptor mesh.
    Receptor handles network transport to execution nodes.
    """

    def __init__(self, task, runner_params):
        self.task = task
        self.runner_params = runner_params
        self.work_type = 'ansible-runner'

    def run(self):
        receptor_ctl = get_receptor_ctl()

        # 1. Prepare work submission
        work_submit_kw = {
            'worktype': 'ansible-runner',
            'node': self.task.instance.execution_node,
            'params': self.receptor_params,
            'signwork': True
        }

        # 2. Stream job data to receptor
        sockin, sockout = socket.socketpair()

        with ThreadPoolExecutor() as executor:
            # Transmit in background thread
            transmitter_future = executor.submit(self.transmit, sockin)

            # Submit work to receptor
            result = receptor_ctl.submit_work(
                payload=sockout.makefile('rb'),
                **work_submit_kw
            )

            self.unit_id = result['unitid']

        # 3. Monitor execution
        while True:
            status = receptor_ctl.simple_command(f'work status {self.unit_id}')
            state = status.get('StateName')

            if state == 'Succeeded':
                return 'successful'
            elif state == 'Failed':
                return 'failed'

            time.sleep(1)
```

## Runner Callback

**File**: `awx/main/tasks/callback.py:24`

The RunnerCallback processes events from ansible-runner:

```python
class RunnerCallback:
    """
    Callback handler for ansible-runner events.
    Processes each event and sends to callback queue.
    """

    def __init__(self, job):
        self.job = job
        self.event_ct = 0
        self.dispatcher = CallbackQueueDispatcher()

    def event_handler(self, event_data):
        """Handle a single ansible event"""

        # 1. Add job metadata
        event_data['job_id'] = self.job.id
        if self.job.workflow_job_id:
            event_data['workflow_job_id'] = self.job.workflow_job_id

        # 2. Map host names to IDs
        if 'host' in event_data:
            host = self.job.inventory.hosts.filter(
                name=event_data['host']
            ).first()
            if host:
                event_data['host_id'] = host.id

        # 3. Redact sensitive data
        event_data = self.redact_sensitive_data(event_data)

        # 4. Dispatch to callback queue
        self.dispatcher.dispatch(event_data)

        # 5. Rate-limited WebSocket emission
        if self.should_emit_websocket():
            self.emit_websocket(event_data)

        self.event_ct += 1

    def status_handler(self, status_data, runner_config):
        """Handle runner status changes"""
        if status_data['status'] == 'starting':
            self.job.websocket_emit_status('running')
        elif status_data['status'] in ('successful', 'failed', 'canceled'):
            self.job.websocket_emit_status(status_data['status'])
```

## Callback Queue Dispatcher

**File**: `awx/main/queue.py`

Events are sent to a separate callback receiver process:

```python
class CallbackQueueDispatcher:
    """
    Send job events to callback receiver via PostgreSQL.
    Events are batched for efficiency.
    """

    def __init__(self):
        self.queue = []
        self.last_flush = time.time()

    def dispatch(self, event_data):
        """Queue an event for dispatch"""
        self.queue.append(event_data)

        # Flush if batch is large enough or time elapsed
        if len(self.queue) >= 100 or time.time() - self.last_flush > 0.5:
            self.flush()

    def flush(self):
        """Send queued events to callback receiver"""
        if not self.queue:
            return

        # Send via PostgreSQL NOTIFY
        with pg_bus_conn() as conn:
            conn.notify('callback_events', json.dumps(self.queue))

        self.queue = []
        self.last_flush = time.time()
```

## Event Types

ansible-runner emits these event types:

| Event | Description |
|-------|-------------|
| `playbook_on_start` | Playbook execution started |
| `playbook_on_play_start` | New play starting |
| `playbook_on_task_start` | New task starting |
| `runner_on_start` | Task running on host |
| `runner_on_ok` | Task succeeded on host |
| `runner_on_failed` | Task failed on host |
| `runner_on_skipped` | Task skipped on host |
| `runner_on_unreachable` | Host unreachable |
| `playbook_on_stats` | Final playbook statistics |

## Private Data Directory

Jobs execute in an isolated directory structure:

```
/tmp/awx_<job_id>_<random>/
├── artifacts/
│   └── <job_id>/
│       ├── stdout
│       └── status
├── env/
│   ├── extravars        # Extra variables JSON
│   ├── passwords        # Credential passwords
│   └── settings         # ansible-runner settings
├── inventory/
│   └── hosts            # Generated inventory
└── project/             # Symlink to project
```

## Credential Injection

**File**: `awx/main/models/credential.py`

Credentials are injected into the job environment:

```python
class CredentialType:
    def inject_credential(self, credential, env, passwords):
        """Inject credential into job execution"""

        # SSH credentials
        if self.kind == 'ssh':
            env['ANSIBLE_PRIVATE_KEY_FILE'] = credential.get_ssh_key_path()
            passwords['ssh_password'] = credential.get_input('password')

        # Vault credentials
        elif self.kind == 'vault':
            passwords['vault_password'] = credential.get_input('vault_password')

        # Cloud credentials (AWS, Azure, GCP)
        elif self.kind == 'aws':
            env['AWS_ACCESS_KEY_ID'] = credential.get_input('username')
            env['AWS_SECRET_ACCESS_KEY'] = credential.get_input('password')
```

## Execution Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Worker receives message                       │
│   RunJob.run(pk=42)                                             │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Load job from DB                              │
│   job = Job.objects.get(pk=42)                                  │
│   job.status = 'running'                                        │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                 Build execution context                          │
│   - Create private data directory                               │
│   - Build inventory file                                        │
│   - Build extra vars                                            │
│   - Inject credentials                                          │
│   - Build ansible-playbook args                                 │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│               Submit to Receptor                                 │
│   AWXReceptorJob.run()                                          │
│   - Stream data to execution node                               │
│   - receptor_ctl.submit_work()                                  │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│              Execution Node                                      │
│   ansible-runner executes playbook                              │
│   Events streamed back via Receptor                             │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│              RunnerCallback                                      │
│   For each event:                                               │
│   - Add metadata                                                │
│   - Redact secrets                                              │
│   - Dispatch to callback queue                                  │
│   - Emit WebSocket (rate-limited)                               │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│              Callback Receiver                                   │
│   - Save JobEvent records to DB                                 │
│   - Update host statistics                                      │
│   - Emit WebSocket notifications                                │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Finalization                                  │
│   job.status = 'successful' or 'failed'                         │
│   job.finished = now()                                          │
│   Send notification templates                                   │
│   Cleanup private data directory                                │
└─────────────────────────────────────────────────────────────────┘
```

## Execution Environments

Jobs can run in container execution environments:

```python
# Specified on JobTemplate
execution_environment = models.ForeignKey(
    'ExecutionEnvironment',
    on_delete=models.SET_NULL,
    null=True
)
```

When set, ansible-runner uses podman/docker to run in the specified container image.
