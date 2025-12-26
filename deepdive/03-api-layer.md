# API Layer

This document details how the Django REST Framework API handles job launch requests.

## What to take away

- `GET /launch/` returns a launch configuration snapshot used by the UI to decide prompts.
- `POST /launch/` validates inputs, creates a job, and triggers scheduling.
- The serializer is the central validation gate; most launch failures surface here.

## Key Files

| File | Purpose |
|------|---------|
| `awx/api/views/__init__.py` | Main API views including JobTemplateLaunch |
| `awx/api/serializers.py` | Request/response serialization |
| `awx/api/urls.py` | URL routing |

## URL Routing

**File**: `awx/api/urls.py`

```python
# Job template launch endpoints
urlpatterns = [
    # GET: Returns launch configuration
    # POST: Launches the job
    path('job_templates/<int:pk>/launch/',
         views.JobTemplateLaunch.as_view(),
         name='job_template_launch'),
]
```

## JobTemplateLaunch View

**File**: `awx/api/views/__init__.py:2354`

```python
class JobTemplateLaunch(RetrieveAPIView):
    model = JobTemplate
    obj_permission_type = 'start'
    serializer_class = JobLaunchSerializer

    def update_raw_data(self, data):
        # Add launch configuration to response
        obj = self.get_object()
        extra_vars = data.pop('extra_vars', None) or {}
        ...

    def post(self, request, *args, **kwargs):
        obj = self.get_object()  # JobTemplate instance

        # 1. Validate the request
        serializer = self.serializer_class(
            instance=obj,
            data=request.data,
            context={'request': request}
        )
        if not serializer.is_valid():
            return Response(serializer.errors, status=400)

        # 2. Check permissions
        # Uses Django REST Framework permission classes
        # and AWX's custom RBAC

        # 3. Create the job
        new_job = obj.create_unified_job(**serializer.validated_data)

        # 4. Signal the job to start
        result = new_job.signal_start(**passwords)

        # 5. Return response
        if not result:
            # Job couldn't start (e.g., missing dependencies)
            data = JobSerializer(new_job, context=context).data
            return Response(data, status=status.HTTP_400_BAD_REQUEST)

        # Success - return job details
        data = JobSerializer(new_job, context=context).data
        headers = {'Location': new_job.get_absolute_url(request=request)}
        return Response(data, status=status.HTTP_201_CREATED, headers=headers)
```

## Launch Configuration vs Launch Execution

- **GET `/launch/`** uses the same serializer to build a launch configuration payload.
- **POST `/launch/`** uses the serializer for full validation and job creation.
- **Survey spec** is fetched separately by the UI: `GET /api/v2/job_templates/{id}/survey_spec/`.

## Request Validation

**File**: `awx/api/serializers.py`

The `JobLaunchSerializer` validates:
- Inventory (if `ask_inventory_on_launch`)
- Credentials (if `ask_credential_on_launch`)
- Extra variables (if `ask_variables_on_launch`)
- Survey responses (if `survey_enabled`)
- Limit, tags, job type, verbosity, etc.

```python
class JobLaunchSerializer(serializers.Serializer):
    # Optional fields based on template configuration
    inventory = serializers.PrimaryKeyRelatedField(
        queryset=Inventory.objects.all(),
        required=False,
        allow_null=True
    )
    credentials = serializers.PrimaryKeyRelatedField(
        queryset=Credential.objects.all(),
        required=False,
        many=True
    )
    extra_vars = serializers.JSONField(required=False, default=dict)
    # ... more fields

    def validate(self, attrs):
        obj = self.instance  # JobTemplate

        # Check if inventory is required but missing
        if obj.ask_inventory_on_launch:
            if not attrs.get('inventory') and not obj.inventory:
                raise serializers.ValidationError({
                    'inventory': 'Inventory is required for launch'
                })

        # Validate survey responses
        if obj.survey_enabled:
            self.validate_survey(attrs.get('extra_vars', {}))

        return attrs
```

## Permission Checking

AWX uses a custom RBAC system layered on Django REST Framework:

**File**: `awx/api/permissions.py`

```python
class JobTemplatePermission(BasePermission):
    def has_object_permission(self, request, view, obj):
        # Check if user can execute this template
        if view.action == 'launch':
            return request.user.can_access(
                JobTemplate, 'start', obj
            )
        return super().has_object_permission(request, view, obj)
```

Permission types:
- `start` - Can launch jobs from this template
- `read` - Can view the template
- `change` - Can edit the template
- `delete` - Can delete the template
- `admin` - Full control including permission management

## Where Launch Errors Surface

Common failure points and where to look:
- **Missing prompts**: `JobLaunchSerializer.validate()` raises `ValidationError`.
- **Pre-start checks**: `UnifiedJob.pre_start()` or `signal_start()` returns False.
- **Dependency conflicts**: project sync in progress, missing inventory, invalid credentials.

## Response Format

Successful launch returns:

```json
{
  "id": 42,
  "type": "job",
  "url": "/api/v2/jobs/42/",
  "created": "2024-01-15T10:30:00.000000Z",
  "name": "Demo Job Template",
  "status": "pending",
  "job_template": 7,
  "inventory": 1,
  "project": 6,
  "playbook": "hello_world.yml",
  "execution_node": "",
  "controller_node": "",
  "related": {
    "stdout": "/api/v2/jobs/42/stdout/",
    "job_events": "/api/v2/jobs/42/job_events/",
    "cancel": "/api/v2/jobs/42/cancel/",
    "relaunch": "/api/v2/jobs/42/relaunch/"
  }
}
```

## Error Responses

| Status | Meaning |
|--------|---------|
| 400 | Validation error (missing required field, invalid survey response) |
| 401 | Not authenticated |
| 403 | Permission denied |
| 404 | Job template not found |
| 409 | Conflict (e.g., project sync in progress) |

## Other Job-Related Endpoints

```
GET  /api/v2/jobs/                      # List jobs
GET  /api/v2/jobs/{id}/                 # Get job details
GET  /api/v2/jobs/{id}/stdout/          # Get job output
GET  /api/v2/jobs/{id}/job_events/      # Get job events
POST /api/v2/jobs/{id}/cancel/          # Cancel job
POST /api/v2/jobs/{id}/relaunch/        # Relaunch job
```
