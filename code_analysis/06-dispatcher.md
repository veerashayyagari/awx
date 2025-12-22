# Dispatcher

This document details how AWX dispatches jobs to workers using PostgreSQL LISTEN/NOTIFY.

## Key Files

| File | Purpose |
|------|---------|
| `awx/main/dispatch/publish.py` | `@task` decorator and message publishing |
| `awx/main/dispatch/worker/base.py` | Worker pool implementation |
| `awx/main/dispatch/pool.py` | Process pool for task execution |
| `awx/main/dispatch/__init__.py` | PostgreSQL connection helpers |

## Why PostgreSQL Instead of RabbitMQ/Redis?

AWX chose PostgreSQL LISTEN/NOTIFY over traditional message brokers because:
1. **Reduced infrastructure** - No separate broker to manage
2. **Transactional consistency** - Messages tied to DB transactions
3. **Simplicity** - Single source of truth for state

Trade-offs:
- Less throughput than dedicated brokers
- No built-in retry/dead-letter queues

## @task Decorator

**File**: `awx/main/dispatch/publish.py:19`

The `@task` decorator makes a function or class dispatchable:

```python
@task(queue=get_local_queuename)
def run_task_manager():
    TaskManager().schedule()


@task(queue=get_local_queuename)
class RunJob(BaseTask):
    def run(self, pk):
        # Execute playbook
        ...
```

### How It Works

```python
class task:
    def __init__(self, queue=None):
        self.queue = queue

    def __call__(self, fn=None):
        queue = self.queue

        class PublisherMixin:
            @classmethod
            def apply_async(cls, args=None, kwargs=None, queue=None, uuid=None):
                task_id = uuid or str(uuid4())
                args = args or []
                kwargs = kwargs or {}
                queue = queue or cls.queue

                # Build message
                obj = {
                    'uuid': task_id,
                    'args': args,
                    'kwargs': kwargs,
                    'task': cls.name  # e.g., 'awx.main.tasks.jobs.RunJob'
                }

                # Add correlation ID
                guid = get_guid()
                if guid:
                    obj['guid'] = guid

                # Resolve callable queue
                if callable(queue):
                    queue = queue()

                # Publish via PostgreSQL NOTIFY
                if not settings.IS_TESTING():
                    with pg_bus_conn() as conn:
                        conn.notify(queue, json.dumps(obj))

                return (obj, queue)

            @classmethod
            def delay(cls, *args, **kwargs):
                return cls.apply_async(args, kwargs)

        # Create new class with PublisherMixin
        cls = type(fn.__name__, (PublisherMixin,), {'name': serialize_task(fn)})
        return cls
```

## Message Format

Messages are JSON objects sent via PostgreSQL NOTIFY:

```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "task": "awx.main.tasks.jobs.RunJob",
  "args": [42],
  "kwargs": {},
  "guid": "abc123",
  "callbacks": [],
  "errbacks": []
}
```

| Field | Purpose |
|-------|---------|
| `uuid` | Unique task ID (stored as `celery_task_id` on job) |
| `task` | Fully qualified Python path to task class/function |
| `args` | Positional arguments (usually just job PK) |
| `kwargs` | Keyword arguments |
| `guid` | Correlation ID for request tracing |

## PostgreSQL Connection

**File**: `awx/main/dispatch/__init__.py`

```python
@contextmanager
def pg_bus_conn(new_connection=False):
    """
    Context manager for PostgreSQL pub/sub connection.
    Uses a dedicated connection separate from Django ORM.
    """
    conn = psycopg2.connect(
        dbname=settings.DATABASES['default']['NAME'],
        host=settings.DATABASES['default']['HOST'],
        user=settings.DATABASES['default']['USER'],
        password=settings.DATABASES['default']['PASSWORD']
    )
    conn.set_isolation_level(ISOLATION_LEVEL_AUTOCOMMIT)

    try:
        yield PubSub(conn)
    finally:
        conn.close()
```

## Queue Names

**File**: `awx/main/dispatch/__init__.py`

```python
def get_local_queuename():
    """
    Get the queue name for this node.
    Format: awx_<hostname>_<pid>
    """
    return f"awx_{socket.gethostname()}_{os.getpid()}"
```

Special queues:
- `tower_broadcast` - Fan-out to all nodes
- `awx_<hostname>` - Specific node queue
- `controlplane` - Control plane nodes only

## Worker Process

**File**: `awx/main/dispatch/worker/base.py`

The dispatcher worker:
1. Subscribes to PostgreSQL channels (LISTEN)
2. Receives NOTIFY messages
3. Deserializes and routes to task handler
4. Executes task in worker pool

```python
class AWXConsumerBase:
    def __init__(self):
        self.pool = AutoscalePool(
            min_workers=settings.JOB_EVENT_WORKERS,
            max_workers=settings.JOB_EVENT_WORKERS
        )

    def run(self):
        with pg_bus_conn() as conn:
            # Subscribe to queues
            conn.listen(get_local_queuename())
            conn.listen('tower_broadcast')

            # Main event loop
            for msg in conn.events():
                self.process_message(msg)

    def process_message(self, msg):
        body = json.loads(msg.payload)
        task_name = body['task']
        args = body['args']
        kwargs = body['kwargs']

        # Import and instantiate task
        task_cls = import_task(task_name)

        # Execute in worker pool
        self.pool.submit(task_cls.run, *args, **kwargs)
```

## Process Pool

**File**: `awx/main/dispatch/pool.py`

```python
class AutoscalePool:
    """
    Process pool that scales workers based on load.
    Uses multiprocessing for isolation.
    """

    def __init__(self, min_workers=4, max_workers=16):
        self.min_workers = min_workers
        self.max_workers = max_workers
        self.workers = []
        self.task_queue = multiprocessing.Queue()

    def submit(self, fn, *args, **kwargs):
        """Submit task to worker pool"""
        self.task_queue.put((fn, args, kwargs))

        # Scale up if needed
        if self.should_scale_up():
            self.add_worker()

    def worker_loop(self):
        """Main worker process loop"""
        while True:
            fn, args, kwargs = self.task_queue.get()
            try:
                fn(*args, **kwargs)
            except Exception as e:
                logger.exception(f"Task failed: {e}")
```

## Dispatch Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     start_task()                                 │
│   task.task_class.apply_async([task.pk], queue=queue_name)      │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    @task decorator                               │
│   PublisherMixin.apply_async()                                  │
│   Build message: {uuid, task, args, kwargs}                     │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                  PostgreSQL NOTIFY                               │
│   conn.notify(queue_name, json.dumps(message))                  │
│   e.g., NOTIFY awx_control_12345, '{"task": "RunJob", ...}'     │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                 PostgreSQL Server                                │
│   Routes NOTIFY to all LISTENing connections                    │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Worker Process                                  │
│   AWXConsumerBase.run() - LISTEN on queue                       │
│   Receives NOTIFY message                                       │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                 Process Message                                  │
│   Deserialize JSON                                              │
│   Import task class: awx.main.tasks.jobs.RunJob                 │
│   Submit to worker pool                                         │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Worker Pool                                     │
│   AutoscalePool executes: RunJob.run(pk)                        │
│   Job execution begins                                          │
└─────────────────────────────────────────────────────────────────┘
```

## Broadcast Messages

For cluster-wide operations, use the `tower_broadcast` queue:

```python
@task(queue='tower_broadcast')
def clear_all_caches():
    """Runs on every node in the cluster"""
    cache.clear()
```

## Connection on Commit

Important: Messages are sent inside `connection.on_commit()` to ensure:
1. Job record is committed to DB before message sent
2. If transaction rolls back, message isn't sent
3. Worker can find job in DB when it starts

```python
# From start_task()
def dispatch_task():
    task.task_class.apply_async([task.pk], opts, queue=queue_name)

connection.on_commit(dispatch_task)
```
