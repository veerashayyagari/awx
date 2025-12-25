# Frontend Launch Flow

This document details how the React frontend handles job launches.

## Key Files

| File | Purpose |
|------|---------|
| `awx/ui/src/components/LaunchButton/LaunchButton.js` | Main launch button component |
| `awx/ui/src/api/models/JobTemplates.js` | API client for job templates |
| `awx/ui/src/api/Base.js` | Base HTTP client (Axios wrapper) |

## Launch Button Component

**File**: `awx/ui/src/components/LaunchButton/LaunchButton.js`

The `LaunchButton` component handles the complexity of job launches, including:
- Checking if the template requires prompts (inventory, credentials, survey, etc.)
- Showing the appropriate wizard if prompts are needed
- Launching directly if no prompts required

### handleLaunch() Flow

```javascript
handleLaunch = async () => {
  const { resource } = this.props;

  // 1. Get launch configuration
  // GET /api/v2/job_templates/{id}/launch/
  const { data: launchConfig } = await JobTemplatesAPI.readLaunch(resource.id);

  // 2. Check if prompts are needed
  if (launchConfig.ask_inventory_on_launch ||
      launchConfig.ask_credential_on_launch ||
      launchConfig.survey_enabled ||
      /* other prompt conditions */) {
    // Show launch wizard modal
    this.setState({ showLaunchWizard: true, launchConfig });
  } else {
    // 3. Launch directly
    // POST /api/v2/job_templates/{id}/launch/
    const { data: job } = await JobTemplatesAPI.launch(resource.id, {});

    // 4. Redirect to job output
    history.push(`/jobs/${job.id}/output`);
  }
};
```

## API Client

**File**: `awx/ui/src/api/models/JobTemplates.js`

```javascript
class JobTemplates extends SchedulesMixin(
  InstanceGroupsMixin(NotificationsMixin(Base))
) {
  constructor(http) {
    super(http);
    this.baseUrl = 'api/v2/job_templates/';
  }

  // POST /api/v2/job_templates/{id}/launch/
  launch(id, data) {
    return this.http.post(`${this.baseUrl}${id}/launch/`, data);
  }

  // GET /api/v2/job_templates/{id}/launch/
  readLaunch(id) {
    return this.http.get(`${this.baseUrl}${id}/launch/`);
  }
}
```

## Launch Configuration Response

The `readLaunch()` endpoint returns configuration telling the UI what prompts are needed:

```json
{
  "can_start_without_user_input": false,
  "passwords_needed_to_start": [],
  "ask_scm_branch_on_launch": false,
  "ask_variables_on_launch": false,
  "ask_tags_on_launch": false,
  "ask_diff_mode_on_launch": false,
  "ask_skip_tags_on_launch": false,
  "ask_job_type_on_launch": false,
  "ask_limit_on_launch": false,
  "ask_verbosity_on_launch": false,
  "ask_inventory_on_launch": true,
  "ask_credential_on_launch": false,
  "ask_execution_environment_on_launch": false,
  "ask_labels_on_launch": false,
  "ask_forks_on_launch": false,
  "ask_job_slice_count_on_launch": false,
  "ask_timeout_on_launch": false,
  "ask_instance_groups_on_launch": false,
  "survey_enabled": false,
  "variables_needed_to_start": [],
  "credential_needed_to_start": false,
  "inventory_needed_to_start": true,
  "job_template_data": {
    "name": "Demo Job Template",
    "id": 7,
    "description": ""
  },
  "defaults": {
    "inventory": { "name": null, "id": null },
    "credentials": [],
    "job_tags": "",
    "skip_tags": ""
  }
}
```

## Launch Wizard

When prompts are required, the `LaunchPromptWizard` component is shown, which:
1. Steps through required prompts (inventory, credentials, survey, etc.)
2. Collects user input
3. Validates responses
4. Submits the launch with collected data

## WebSocket Connection

The frontend maintains a WebSocket connection to `/websocket/` for real-time updates:

**File**: `awx/ui/src/App.js` (WebSocket setup)

```javascript
// Subscribes to job status updates
socket.subscribe('jobs-status_changed', handleJobStatusChange);

// For workflow jobs
socket.subscribe('workflow_events-{id}', handleWorkflowEvent);
```

This enables real-time job output streaming and status updates without polling.

## Job Output View

After launch, the user is redirected to `/jobs/{id}/output`:

**File**: `awx/ui/src/screens/Job/JobOutput/JobOutput.js`

This component:
- Fetches existing job events via REST API
- Subscribes to WebSocket for new events
- Renders ANSI-colored terminal output
- Shows job status, timing, and host statistics
