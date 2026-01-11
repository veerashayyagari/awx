# Frontend Launch Flow

This document details how the React frontend handles job launches.

## Key Files

| File | Purpose |
|------|---------|
| `awx/ui/src/components/LaunchButton/LaunchButton.js` | Main launch button component and relaunch handling |
| `awx/ui/src/components/LaunchPrompt/LaunchPrompt.js` | Prompt UI for inventory, credentials, survey, and vars |
| `awx/ui/src/api/models/JobTemplates.js` | API client for job templates and surveys |
| `awx/ui/src/api/models/Jobs.js` | Relaunch and job actions |
| `awx/ui/src/screens/Job/JobOutput/connectJobSocket.js` | Job output websocket stream |
| `awx/ui/src/api/Base.js` | Base HTTP client (Axios wrapper) |

## Launch Button Component

**File**: `awx/ui/src/components/LaunchButton/LaunchButton.js`

The `LaunchButton` component orchestrates launches and relaunches for multiple resource types:
- `job_template` and `workflow_job_template` launches
- `job`, `workflow_job`, `project_update`, `inventory_update`, and `ad_hoc_command` relaunches
- Prompt display when launch config requires user input

### handleLaunch() Flow

```javascript
const handleLaunch = async () => {
  setIsLaunching(true);

  // 1. Fetch launch configuration
  // GET /api/v2/job_templates/{id}/launch/ (or workflow equivalent)
  const { data: launch } = await readLaunch(resource.id);
  setLaunchConfig(launch);

  // 2. Fetch survey spec only if needed
  if (launch.survey_enabled) {
    const { data: survey } = await readSurvey(resource.id);
    setSurveyConfig(survey);
  }

  // 3. Decide between direct launch vs prompt
  if (canLaunchWithoutPrompt(launch)) {
    await launchWithParams({});
  } else {
    setShowLaunchPrompt(true);
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

  // GET /api/v2/job_templates/{id}/survey_spec/
  readSurvey(id) {
    return this.http.get(`${this.baseUrl}${id}/survey_spec/`);
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

## Launch Prompt UI

When prompts are required, the `LaunchPrompt` component is shown, which:
1. Steps through required prompts (inventory, credentials, survey, etc.)
2. Collects user input
3. Validates responses
4. Submits the launch with collected data

**File**: `awx/ui/src/components/LaunchPrompt/LaunchPrompt.js`

## Relaunch Flow

For existing jobs or updates, the LaunchButton uses relaunch endpoints:
- Jobs: `JobsAPI.relaunch(id, params)`
- Workflow jobs: `WorkflowJobsAPI.relaunch(id, params)`
- Project/inventory updates: `ProjectsAPI.launchUpdate(id)` / `InventorySourcesAPI.launchUpdate(id)`

The relaunch path reuses the same prompt logic when passwords or variables are required.

## WebSocket Connection

Job output uses a WebSocket connection to `/websocket/` for real-time updates:

**File**: `awx/ui/src/screens/Job/JobOutput/connectJobSocket.js`

```javascript
ws.send(JSON.stringify({
  group_name: `jobs`,
  group_data: { id: jobId }
}));
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
