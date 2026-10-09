Schedule AWS ECS services for office hours with a self-service form and Teams alerts

https://n8nworkflows.xyz/workflows/schedule-aws-ecs-services-for-office-hours-with-a-self-service-form-and-teams-alerts-20522


# Schedule AWS ECS services for office hours with a self-service form and Teams alerts

### 1. Workflow Overview

This workflow automates the cost-saving schedule of Amazon ECS services by scaling them down to zero outside of office hours and restoring them on weekday mornings. It supports multi-account, multi-cluster, and multi-project environments, using ECS resource tags (`saved-desired-count` and `override-until`) for state persistence without requiring an external database. It also provides a self-service n8n Form for teammates to manually restart or extend services during off-hours, and dispatches rich Adaptive Card alerts to Microsoft Teams via incoming webhooks.

The workflow logic is grouped into the following functional blocks:

- **1.1 Input Reception & Configuration:** Entry triggers (Schedule triggers for daily stops/weekday starts and Form trigger for manual restarts), initial mode definitions, and global parameter settings.
- **1.2 Project Scan & Discovery:** Project configuration filtering, dynamic cluster enumeration, batching, and API sub-workflow execution to retrieve ECS services and filter them by the scheduling opt-in tag.
- **1.3 Operating Mode Routing & Nightly/Morning Operations:** Routing based on the chosen mode (stop, start, or restart), executing nightly task count backups and zero-scaling, or morning task count restorations and tag cleanup, complete with Teams summary notifications.
- **1.4 Self-Service Restart & Health Polling:** Interactive form option presentation, tag-based override application, restart execution, and continuous health checking (polling every 30 seconds up to a maximum threshold) with Teams notifications for ready or failure states.
- **1.5 Expiration & Re-Stop Automation:** Waiting until the specified override window elapses, verifying active ECS state, re-stopping services outside office hours if un-extended, cleaning up override tags, and broadcasting final status reports to Teams.
- **1.6 Multi-Account Sub-Workflow Engine:** A self-contained routing and HTTP abstraction layer that handles all upstream ECS API calls (`ListServices`, `DescribeServices`, `UpdateService`, `TagResource`, `UntagResource`) by routing requests to the appropriate AWS IAM Assume Role credentials based on account identifiers.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes the execution flow from one of three entry points (Nightly Stop Schedule, Weekday Start Schedule, or Restart Request Form), tags the execution context with a specific mode (`stop`, `start`, or `restart`), and injects core operational configurations.
- **Nodes Involved:** `When Every Day at 8PM`, `When Weekdays at 8AM`, `When Restart Requested`, `Set Mode to Stop`, `Set Mode to Start`, `Set Mode to Restart`, `Set Config Parameters`, `Check Restart Mode`, `Select Project Form`, `List All Projects`.
- **Node Details:**
  - **When Every Day at 8PM** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Triggers the nightly cost-saving shutdown.
    - *Config:* Cron expression `0 20 * * *`.
    - *Inputs:* None | *Outputs:* `Set Mode to Stop`.
    - *Edge Cases:* Ensure n8n timezone settings match the global configuration parameters to prevent off-by-one hour execution errors.
  - **When Weekdays at 8AM** (`n8n-nodes-base.scheduleTrigger`)
    - *Role:* Triggers the morning startup sequence.
    - *Config:* Cron expression `0 8 * * 1-5`.
    - *Inputs:* None | *Outputs:* `Set Mode to Start`.
  - **When Restart Requested** (`n8n-nodes-base.formTrigger`)
    - *Role:* Captures user-submitted restart requests from the n8n Form.
    - *Config:* Path `ecs-restart`, captures inputs for requester name and reason.
    - *Inputs:* None | *Outputs:* `Set Mode to Restart`.
  - **Set Mode to Stop / Start / Restart** (`n8n-nodes-base.set`)
    - *Role:* Normalizes execution input by assigning a string `mode` property (`stop`, `start`, or `restart`).
    - *Inputs:* Corresponding trigger nodes | *Outputs:* `Set Config Parameters`.
  - **Set Config Parameters** (`n8n-nodes-base.set`)
    - *Role:* Establishes global environment parameters including project definitions, AWS regions, cluster identifiers, office hours, timezone, and webhook URLs.
    - *Config:* Sets variables for `projects`, `scheduleTagKey` (`auto-schedule`), `scheduleTagValue` (`office-hours`), `timezone`, `officeStartHour` (`8`), `officeEndHour` (`20`), `defaultDesiredCount` (`1`), `maxHealthChecks` (`20`), `durationHours`, `teamsWebhookUrl`, and `restartFormUrl`.
    - *Inputs:* Mode assignment nodes | *Outputs:* `Check Restart Mode`.
  - **Check Restart Mode** (`n8n-nodes-base.if`)
    - *Role:* Evaluates if the current execution is a restart request spanning multiple unique projects.
    - *Config:* Expression checking `mode === 'restart'` and distinct project count > 1.
    - *Inputs:* `Set Config Parameters` | *Outputs:* True branch to `Select Project Form`, False branch to `List All Projects`.
  - **Select Project Form** (`n8n-nodes-base.form`)
    - *Role:* Renders an interactive form allowing the user to select which project to target when multiple projects are configured.
    - *Config:* Dynamically builds dropdown field options from project names defined in config parameters.
    - *Inputs:* `Check Restart Mode` | *Outputs:* `List All Projects`.
  - **List All Projects** (`n8n-nodes-base.code`)
    - *Role:* Compiles the final list of projects, clusters, and regions to scan based on configuration and user selections.
    - *JS Logic:* Filters projects by user choice if in restart mode with multiple options; maps values to standard payload objects.
    - *Inputs:* `Check Restart Mode`, `Select Project Form` | *Outputs:* `List ECS Services`.

#### 2.2 Project Scan & Discovery
- **Overview:** Enumerates ECS services across targeted AWS clusters, batches service ARNs to comply with AWS API payload limits, retrieves detailed service metadata and tags, and filters for services flagged for automated scheduling.
- **Nodes Involved:** `List ECS Services`, `Batch ECS Services`, `Describe ECS Services`, `Filter Scheduled Services`.
- **Node Details:**
  - **List ECS Services** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Invokes the sub-workflow engine to call AWS ECS `ListServices` for a given cluster, account, and region.
    - *Config:* Executes current workflow ID (`=$workflow.id`), passes action parameter `ListServices`.
    - *Inputs:* `List All Projects` | *Outputs:* `Batch ECS Services`.
    - *Edge Cases:* AWS throttling limits (Rate Exceeded) or invalid IAM assume role configurations will cause execution failure.
  - **Batch ECS Services** (`n8n-nodes-base.code`)
    - *Role:* Chunks incoming service ARNs into arrays of up to 10 items to accommodate AWS `DescribeServices` API limitations.
    - *JS Logic:* Uses `Array.prototype.flatMap` to split `serviceArns` into batches of 10.
    - *Inputs:* `List ECS Services` | *Outputs:* `Describe ECS Services`.
  - **Describe ECS Services** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Invokes the sub-workflow engine to call AWS ECS `DescribeServices` with tag inclusion enabled.
    - *Config:* Passes action `DescribeServices` with body containing cluster name, service batch, and `include: ['TAGS']`.
    - *Inputs:* `Batch ECS Services` | *Outputs:* `Filter Scheduled Services`.
  - **Filter Scheduled Services** (`n8n-nodes-base.code`)
    - *Role:* Parses AWS response objects, extracts custom bookkeeping tags (`saved-desired-count`, `override-until`, `display-name`), filters for active services carrying the opt-in schedule tag, and splits payloads according to the current operation mode.
    - *JS Logic:* Validates `auto-schedule=office-hours`; handles mode-specific exclusions (e.g., ignoring active override windows during nightly stops, or compiling available restart options).
    - *Inputs:* `Describe ECS Services` | *Outputs:* `Route by Service Mode`.

#### 2.3 Operating Mode Routing & Nightly/Morning Operations
- **Overview:** Directs filtered services down distinct branches based on the operating mode (`stop` or `start`), performs tag updates and desired count modifications against AWS ECS, and dispatches summary notifications to Microsoft Teams.
- **Nodes Involved:** `Route by Service Mode`, `Save ECS Task Count`, `Scale ECS to Zero`, `Compile Stop Summary`, `Notify Stop Summary to Teams`, `Restore ECS Task Count`, `Clear ECS Schedule Tags`, `Compile Start Summary`, `Notify Start Summary to Teams`.
- **Node Details:**
  - **Route by Service Mode** (`n8n-nodes-base.switch`)
    - *Role:* Routes execution flow to index 0 (`stop`), index 1 (`start`), or index 2 (`restart`).
    - *Config:* Expression mapping `['stop', 'start', 'restart'].indexOf($json.mode)`.
    - *Inputs:* `Filter Scheduled Services` | *Outputs:* Branch-specific nodes.
  - **Save ECS Task Count** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Tags the ECS service with its current desired task count prior to shutdown.
    - *Config:* Action `TagResource` setting key `saved-desired-count` to current `desiredCount`.
    - *Inputs:* `Route by Service Mode` (Stop path) | *Outputs:* `Scale ECS to Zero`.
  - **Scale ECS to Zero** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Scales down the target ECS service desired task count to zero.
    - *Config:* Action `UpdateService` setting `desiredCount: 0`.
    - *Inputs:* `Save ECS Task Count` | *Outputs:* `Compile Stop Summary`.
  - **Compile Stop Summary** (`n8n-nodes-base.code`)
    - *Role:* Formats a summary payload detailing all services switched off during the nightly run.
    - *Inputs:* `Scale ECS to Zero` | *Outputs:* `Notify Stop Summary to Teams`.
  - **Notify Stop Summary to Teams** (`n8n-nodes-base.httpRequest`)
    - *Role:* Sends an Adaptive Card message to the configured Microsoft Teams webhook URL.
    - *Config:* HTTP POST with JSON Adaptive Card payload.
    - *Inputs:* `Compile Stop Summary` | *Outputs:* None (terminal branch).
  - **Restore ECS Task Count** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Restores the ECS service desired count back to its pre-shutdown value.
    - *Config:* Action `UpdateService` setting `desiredCount` to `savedCount`.
    - *Inputs:* `Route by Service Mode` (Start path) | *Outputs:* `Clear ECS Schedule Tags`.
  - **Clear ECS Schedule Tags** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Removes bookkeeping tags from the ECS service once restored.
    - *Config:* Action `UntagResource` removing `saved-desired-count` and `override-until`.
    - *Inputs:* `Restore ECS Task Count` | *Outputs:* `Compile Start Summary`.
  - **Compile Start Summary** & **Notify Start Summary to Teams**
    - *Role:* Formats and posts the morning startup confirmation card to Microsoft Teams.
    - *Inputs:* `Clear ECS Schedule Tags` | *Outputs:* None (terminal branch).

#### 2.4 Self-Service Restart & Health Polling
- **Overview:** Handles manual restart requests submitted via the n8n form, prompts the user to select specific services and durations, updates override tags, scales services up, and polls ECS health metrics until tasks are fully running or health check attempts are exhausted.
- **Nodes Involved:** `Check Services To Restart`, `Display No Restart Options`, `Select ECS Services`, `Prepare ECS Restart`, `Mark ECS for Override`, `Start ECS Services`, `Display Confirmation Form`, `Compile Restart Message`, `Notify Restart Message to Teams`, `Wait 30 Seconds`, `Organize Services by Cluster`, `Verify ECS Services`, `Assess ECS Health`, `Check ECS Health Status`, `Compose Ready Notification`, `Notify Ready Status to Teams`, `Compose Failure Notification`, `Notify Failure to Teams`.
- **Node Details:**
  - **Check Services to Restart** (`n8n-nodes-base.if`)
    - *Role:* Validates whether any eligible stopped or overridden services exist for restart or extension.
    - *Inputs:* `Route by Service Mode` (Restart path) | *Outputs:* True to `Select ECS Services`, False to `Display No Restart Options`.
  - **Display No Restart Options** (`n8n-nodes-base.form`)
    - *Role:* Completion form displayed when no services are available for restart in the selected project.
    - *Inputs:* `Check Services to Restart` | *Outputs:* None.
  - **Select ECS Services** (`n8n-nodes-base.form`)
    - *Role:* Renders a multi-select form allowing users to choose target services and requested run duration.
    - *Inputs:* `Check Services to Restart` | *Outputs:* `Prepare ECS Restart`.
  - **Prepare ECS Restart** (`n8n-nodes-base.code`)
    - *Role:* Calculates expiration timestamps (`override-until`), validates selection limits (max 10 services), and classifies actions as `start` or `extend`.
    - *Inputs:* `Select ECS Services` | *Outputs:* `Mark ECS for Override`.
  - **Mark ECS for Override** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Tags the ECS service with the override expiration timestamp (`override-until`) so the nightly shutdown ignores it.
    - *Config:* Action `TagResource`.
    - *Inputs:* `Prepare ECS Restart` | *Outputs:* `Start ECS Services`.
  - **Start ECS Services** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Scales up the ECS service to its saved desired task count.
    - *Config:* Action `UpdateService`.
    - *Inputs:* `Mark ECS for Override` | *Outputs:* `Display Confirmation Form` & `Compile Restart Message`.
  - **Display Confirmation Form** (`n8n-nodes-base.form`)
    - *Role:* Shows a completion screen to the form user confirming restart/extension progress.
    - *Inputs:* `Start ECS Services` | *Outputs:* None.
  - **Compile Restart Message** & **Notify Restart Message to Teams**
    - *Role:* Broadcasts the restart initiation notice to Microsoft Teams.
    - *Inputs:* `Start ECS Services` | *Outputs:* `Wait 30 Seconds`.
  - **Wait 30 Seconds** & **Organize Services by Cluster** & **Verify ECS Services**
    - *Role:* Pauses execution for 30 seconds, groups target services by cluster, and calls `DescribeServices` to check task states.
    - *Inputs:* `Notify Restart Message to Teams` (or health retry loop) | *Outputs:* `Assess ECS Health`.
  - **Assess ECS Health** & **Check ECS Health Status** (`n8n-nodes-base.switch`)
    - *Role:* Evaluates running vs desired task counts across all target services. Routes to `ready` (index 0), `waiting` (index 1, looping back to wait), or `failed` (index 2).
    - *Inputs:* `Verify ECS Services` | *Outputs:* Ready notification path, Wait loop, or Failure alert path.
  - **Compose Ready Notification** / **Compose Failure Notification** & respective **Teams HTTP** nodes
    - *Role:* Dispatches success or failure alerts to Microsoft Teams depending on whether health verification succeeded within max attempts (`maxHealthChecks`).
    - *Inputs:* `Check ECS Health Status` | *Outputs:* Ready routes to `Wait Until Specified Time`; Failure routes to `Wait Until Specified Time`.

#### 2.5 Expiration & Re-Stop Automation
- **Overview:** Pauses execution until the temporary restart override window expires, re-verifies the ECS state to ensure no other user extended the lease during office hours, and automatically shuts down services if outside office hours.
- **Nodes Involved:** `Wait Until Specified Time`, `Classify Services for Re-stop`, `Verify Before Re-Stop`, `Decide on Re-stop Action`, `Re-Stop ECS Services`, `Remove ECS Override`, `Compose Stopped Again Notification`, `Notify Stopped Again to Teams`.
- **Node Details:**
  - **Wait Until Specified Time** (`n8n-nodes-base.wait`)
    - *Role:* Pauses workflow execution until the ISO timestamp stored in `override-until` is reached.
    - *Config:* Resume type `specificTime`, evaluated from `override-until`.
    - *Inputs:* Ready or Failure notification paths | *Outputs:* `Classify Services for Re-stop`.
  - **Classify Services for Re-stop** & **Verify Before Re-Stop** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Groups services by cluster and queries AWS ECS for fresh state and tag data.
    - *Inputs:* `Wait Until Specified Time` | *Outputs:* `Decide on Re-stop Action`.
  - **Decide on Re-stop Action** (`n8n-nodes-base.code`)
    - *Role:* Checks if current time is within office hours. If office hours have started, skips re-stopping. Otherwise, verifies that the `override-until` tag still matches the original request (preventing interference if another user extended the service).
    - *Inputs:* `Verify Before Re-Stop` | *Outputs:* `Re-Stop ECS Services`.
  - **Re-Stop ECS Services** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Scales eligible services back down to zero desired tasks.
    - *Config:* Action `UpdateService` setting `desiredCount: 0`.
    - *Inputs:* `Decide on Re-stop Action` | *Outputs:* `Remove ECS Override`.
  - **Remove ECS Override** (`n8n-nodes-base.executeWorkflow`)
    - *Role:* Clears the temporary `override-until` tag from the ECS service.
    - *Config:* Action `UntagResource`.
    - *Inputs:* `Re-Stop ECS Services` | *Outputs:* `Compose Stopped Again Notification`.
  - **Compose Stopped Again Notification** & **Notify Stopped Again to Teams**
    - *Role:* Sends the final shutdown notification card to Microsoft Teams.
    - *Inputs:* `Remove ECS Override` | *Outputs:* None (terminal branch).

#### 2.6 Multi-Account Sub-Workflow Engine
- **Overview:** Acts as an internal API routing layer. Every upstream node that interacts with AWS ECS executes this workflow recursively via sub-workflow calls, routing requests to the correct AWS credentials based on the target account key.
- **Nodes Involved:** `On ECS API Call`, `Route by AWS Account`, `Handle ECS for Account-A`, `Handle ECS for Account-B`, `Halt on Unknown Account`, `Return ECS API Results`.
- **Node Details:**
  - **On ECS API Call** (`n8n-nodes-base.executeWorkflowTrigger`)
    - *Role:* Entry point for recursive sub-workflow invocations carrying action payloads, regions, accounts, and context.
    - *Inputs:* Upstream ECS execute-workflow nodes | *Outputs:* `Route by AWS Account`.
  - **Route by AWS Account** (`n8n-nodes-base.switch`)
    - *Role:* Directs the API request to the appropriate account handler based on `$json.account`.
    - *Config:* Rules matching account keys (`account-a`, `account-b`), with a fallback to `Halt on Unknown Account`.
    - *Inputs:* `On ECS API Call` | *Outputs:* Account-specific HTTP request nodes or error handler.
  - **Handle ECS for Account-A / Account-B** (`n8n-nodes-base.httpRequest`)
    - *Role:* Executes authenticated HTTP POST requests against the AWS ECS endpoint using account-specific IAM Assume Role credentials.
    - *Config:* URL `https://ecs.={{ $json.region }}.amazonaws.com/`, AWS Assume Role credentials, headers `X-Amz-Target: AmazonEC2ContainerServiceV20141113.{{ $json.action }}` and `Content-Type: application/x-amz-json-1.1`.
    - *Inputs:* `Route by AWS Account` | *Outputs:* `Return ECS API Results`.
  - **Halt on Unknown Account** (`n8n-nodes-base.stopAndError`)
    - *Role:* Stops execution and returns a descriptive error if an unregistered AWS account key is passed.
    - *Inputs:* `Route by AWS Account` (Fallback) | *Outputs:* None.
  - **Return ECS API Results** (`n8n-nodes-base.code`)
    - *Role:* Parses AWS JSON response data and merges it back with the original calling context.
    - *JS Logic:* Parses raw string data returned by Amazon's ECS API content-type headers.
    - *Inputs:* Account HTTP handlers | *Outputs:* Returns payload to calling upstream node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow documentation and setup guide | None | None | Schedule AWS ECS services for office hours with a self-service restart form and Teams alerts... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Trigger and set mode documentation | None | None | Trigger and set mode: Entry cluster for the evening stop schedule, weekday start schedule, and restart request form... |
| Sticky Note2 | n8n-nodes-base.stickyNote | Configure project scan documentation | None | None | Configure project scan: Loads shared workflow settings, decides whether a project must be chosen interactively... |
| Sticky Note3 | n8n-nodes-base.stickyNote | Discover scheduled services documentation | None | None | Discover scheduled services: Lists ECS services, batches them for describe calls, retrieves service details... |
| Sticky Note4 | n8n-nodes-base.stickyNote | Route requested mode documentation | None | None | Route requested mode: Branches the discovered scheduled services into the stop, start, or restart handling paths... |
| Sticky Note5 | n8n-nodes-base.stickyNote | Stop scaling actions documentation | None | None | Stop scaling actions: For the stop path, saves each service's current task count and scales matching ECS services down to zero. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Stop Teams summary documentation | None | None | Stop Teams summary: Builds and posts the after-hours stop summary to Microsoft Teams. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Start restore actions documentation | None | None | Start restore actions: For the morning start path, restores saved desired task counts and clears schedule-related tags... |
| Sticky Note8 | n8n-nodes-base.stickyNote | Start Teams summary documentation | None | None | Start Teams summary: Builds and posts the weekday start summary to Microsoft Teams. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Select restart services documentation | None | None | Select restart services: Checks whether any services are eligible for restart or extension, then shows options... |
| Sticky Note10 | n8n-nodes-base.stickyNote | Prepare service restart documentation | None | None | Prepare service restart: Transforms selected services, marks restart override, and starts requested ECS services. |
| Sticky Note11 | n8n-nodes-base.stickyNote | Confirm and announce restart documentation | None | None | Confirm and announce restart: Shows confirmation page while building and posting initial restart message to Teams. |
| Sticky Note12 | n8n-nodes-base.stickyNote | Initial health check documentation | None | None | Initial health check: Waits briefly after restart, groups services by cluster, and calls ECS to check current status. |
| Sticky Note13 | n8n-nodes-base.stickyNote | Evaluate health status documentation | None | None | Evaluate health status: Evaluates returned ECS status and routes flow to ready, retry, or failure handling. |
| Sticky Note14 | n8n-nodes-base.stickyNote | Ready Teams notice documentation | None | None | Ready Teams notice: Builds and posts a Teams notification when restarted services become healthy. |
| Sticky Note15 | n8n-nodes-base.stickyNote | Failure Teams alert documentation | None | None | Failure Teams alert: Builds and posts a Teams alert if restarted services fail to become healthy within expected checks. |
| Sticky Note16 | n8n-nodes-base.stickyNote | Wait restart window documentation | None | None | Wait restart window: Waits until temporary restart or override window has expired before considering re-stop. |
| Sticky Note17 | n8n-nodes-base.stickyNote | Check re-stop eligibility documentation | None | None | Check re-stop eligibility: Groups services, checks latest ECS state, and decides which services to stop again outside office hours. |
| Sticky Note18 | n8n-nodes-base.stickyNote | Execute re-stop cleanup documentation | None | None | Execute re-stop cleanup: Stops eligible services again and clears temporary restart override metadata. |
| Sticky Note19 | n8n-nodes-base.stickyNote | Notify re-stop result documentation | None | None | Notify re-stop result: Builds and posts final Teams message confirming services were stopped again. |
| Sticky Note20 | n8n-nodes-base.stickyNote | ECS API entry documentation | None | None | ECS API entry: Every ECS node above calls this same workflow... |
| Sticky Note21 | n8n-nodes-base.stickyNote | Account ECS handlers documentation | None | None | Account ECS handlers: Sends ECS API request using correct account-specific HTTP request node, returns responses... |
| When Every Day at 8PM | scheduleTrigger | Triggers nightly shutdown schedule | None | Set Mode to Stop | |
| When Weekdays at 8AM | scheduleTrigger | Triggers weekday startup schedule | None | Set Mode to Start | |
| When Restart Requested | formTrigger | Entry point for self-service restart form | None | Set Mode to Restart | |
| Set Mode to Stop | set | Sets execution mode to stop | When Every Day at 8PM | Set Config Parameters | |
| Set Mode to Start | set | Sets execution mode to start | When Weekdays at 8AM | Set Config Parameters | |
| Set Mode to Restart | set | Sets execution mode to restart | When Restart Requested | Set Config Parameters | |
| Set Config Parameters | set | Stores global configuration parameters | Set Mode to Stop, Set Mode to Start, Set Mode to Restart | Check Restart Mode | |
| Check Restart Mode | if | Checks if restart mode requires project selection | Set Config Parameters | Select Project Form, List All Projects | |
| Select Project Form | form | Renders project selection form | Check Restart Mode | List All Projects | |
| List All Projects | code | Builds array of projects to scan | Check Restart Mode, Select Project Form | List ECS Services | |
| List ECS Services | executeWorkflow | Lists ECS services in target cluster | List All Projects | Batch ECS Services | |
| Batch ECS Services | code | Batches service ARNs into groups of 10 | List ECS Services | Describe ECS Services | |
| Describe ECS Services | executeWorkflow | Retrieves service details and tags | Batch ECS Services | Filter Scheduled Services | |
| Filter Scheduled Services | code | Filters services by scheduling opt-in tag | Describe ECS Services | Route by Service Mode | |
| Route by Service Mode | switch | Routes services based on operation mode | Filter Scheduled Services | Save ECS Task Count, Restore ECS Task Count, Check Services to Restart | |
| Save ECS Task Count | executeWorkflow | Tags ECS service with saved desired count | Route by Service Mode | Scale ECS to Zero | |
| Scale ECS to Zero | executeWorkflow | Scales ECS service desired count to 0 | Save ECS Task Count | Compile Stop Summary | |
| Compile Stop Summary | code | Compiles nightly stop summary message | Scale ECS to Zero | Notify Stop Summary to Teams | |
| Notify Stop Summary to Teams | httpRequest | Posts stop summary to Microsoft Teams webhook | Compile Stop Summary | None | |
| Restore ECS Task Count | executeWorkflow | Restores ECS service desired count | Route by Service Mode | Clear ECS Schedule Tags | |
| Clear ECS Schedule Tags | executeWorkflow | Clears scheduling tags from ECS service | Restore ECS Task Count | Compile Start Summary | |
| Compile Start Summary | code | Compiles morning start summary message | Clear ECS Schedule Tags | Notify Start Summary to Teams | |
| Notify Start Summary to Teams | httpRequest | Posts start summary to Microsoft Teams webhook | Compile Start Summary | None | |
| Check Services to Restart | if | Checks if valid restart options exist | Route by Service Mode | Select ECS Services, Display No Restart Options | |
| Display No Restart Options | form | Shows completion page when no services found | Check Services to Restart | None | |
| Select ECS Services | form | Renders service and duration selection form | Check Services to Restart | Prepare ECS Restart | |
| Prepare ECS Restart | code | Prepares restart payload and override timestamp | Select ECS Services | Mark ECS for Override | |
| Mark ECS for Override | executeWorkflow | Tags ECS service with override-until | Prepare ECS Restart | Start ECS Services | |
| Start ECS Services | executeWorkflow | Scales up ECS service to desired count | Mark ECS for Override | Display Confirmation Form, Compile Restart Message | |
| Display Confirmation Form | form | Shows restart confirmation page | Start ECS Services | None | |
| Compile Restart Message | code | Formats restart notification message | Start ECS Services | Notify Restart Message to Teams | |
| Notify Restart Message to Teams | httpRequest | Posts restart message to Microsoft Teams | Compile Restart Message | Wait 30 Seconds | |
| Wait 30 Seconds | wait | Pauses 30 seconds before health check | Notify Restart Message to Teams, Check Health Status (waiting) | Organize Services by Cluster | |
| Organize Services by Cluster | code | Groups service checks by cluster | Wait 30 Seconds | Verify ECS Services | |
| Verify ECS Services | executeWorkflow | Describes ECS services for health assessment | Organize Services by Cluster | Assess ECS Health | |
| Assess ECS Health | code | Evaluates running vs desired task counts | Verify ECS Services | Check ECS Health Status | |
| Check Health Status | switch | Routes based on health check assessment | Assess ECS Health | Compose Ready Notification, Wait 30 Seconds, Compose Failure Notification | |
| Compose Ready Notification | code | Formats ready status message | Check Health Status | Notify Ready Status to Teams | |
| Notify Ready Status to Teams | httpRequest | Posts ready status to Microsoft Teams | Compose Ready Notification | Wait Until Specified Time | |
| Compose Failure Notification | code | Formats failure notification message | Check Health Status | Notify Failure to Teams | |
| Notify Failure to Teams | httpRequest | Posts failure alert to Microsoft Teams | Compose Failure Notification | Wait Until Specified Time | |
| Wait Until Specified Time | wait | Pauses until override expiration time | Notify Ready Status to Teams, Notify Failure to Teams | Classify Services for Re-stop | |
| Classify Services for Re-stop | code | Groups services for re-stop verification | Wait Until Specified Time | Verify Before Re-Stop | |
| Verify Before Re-Stop | executeWorkflow | Describes services to verify latest tags | Classify Services for Re-stop | Decide on Re-stop Action | |
| Decide on Re-stop Action | code | Checks office hours and override tag match | Verify Before Re-Stop | Re-Stop ECS Services | |
| Re-Stop ECS Services | executeWorkflow | Scales service back down to 0 | Decide on Re-stop Action | Remove ECS Override | |
| Remove ECS Override | executeWorkflow | Removes override-until tag from ECS service | Re-Stop ECS Services | Compose Stopped Again Notification | |
| Compose Stopped Again Notification | code | Formats re-stop notification message | Remove ECS Override | Notify Stopped Again to Teams | |
| Notify Stopped Again to Teams | httpRequest | Posts re-stop notification to Microsoft Teams | Compose Stopped Again Notification | None | |
| On ECS API Call | executeWorkflowTrigger | Entry point for sub-workflow ECS calls | None | Route by AWS Account | |
| Route by AWS Account | switch | Routes ECS API call by account identifier | On ECS API Call | Handle ECS for Account-A, Handle ECS for Account-B, Halt on Unknown Account | |
| Handle ECS for Account-A | httpRequest | Executes ECS API call for Account A via Assume Role | Route by AWS Account | Return ECS API Results | |
| Handle ECS for Account-B | httpRequest | Executes ECS API call for Account B via Assume Role | Route by AWS Account | Return ECS API Results | |
| Halt on Unknown Account | stopAndError | Stops execution if account is unregistered | Route by AWS Account | None | |
| Return ECS API Results | code | Parses ECS API response and returns context | Handle ECS for Account-A, Handle ECS for Account-B | None | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these step-by-step instructions:

1. **Create Triggers and Entry Points:**
   - Add a **Schedule Trigger** (`When Every Day at 8PM`) with Cron expression `0 20 * * *`. Connect it to a **Set** node (`Set Mode to Stop`) assigning `mode = "stop"`.
   - Add a **Schedule Trigger** (`When Weekdays at 8AM`) with Cron expression `0 8 * * 1-5`. Connect it to a **Set** node (`Set Mode to Start`) assigning `mode = "start"`.
   - Add a **Form Trigger** (`When Restart Requested`) with path `ecs-restart`. Connect it to a **Set** node (`Set Mode to Restart`) assigning `mode = "restart"`.

2. **Configure Global Settings:**
   - Connect all three `Set Mode` nodes to a single **Set** node (`Set Config Parameters`). Include fields: `projects` (Array JSON with `project`, `account`, `region`, `cluster`), `scheduleTagKey` (`auto-schedule`), `scheduleTagValue` (`office-hours`), `timezone` (`UTC`), `officeStartHour` (`8`), `officeEndHour` (`20`), `defaultDesiredCount` (`1`), `maxHealthChecks` (`20`), `durationHours` (`[0.5, 1, 2, 3, 4, 6, 8]`), `teamsWebhookUrl`, and `restartFormUrl`.

3. **Build Project Scan & Discovery Logic:**
   - Add an **If** node (`Check Restart Mode`) to verify if mode is restart and project count > 1.
   - If true, connect to a **Form** node (`Select Project Form`) presenting project dropdown options.
   - Connect both paths to a **Code** node (`List All Projects`) containing the JavaScript snippet to filter and output active project objects.
   - Add an **Execute Workflow** node (`List ECS Services`) pointing to `{{ $workflow.id }}` with action `ListServices`.
   - Add a **Code** node (`Batch ECS Services`) to chunk service ARNs into batches of 10.
   - Add an **Execute Workflow** node (`Describe ECS Services`) calling `{{ $workflow.id }}` with action `DescribeServices` and `include: ['TAGS']`.
   - Add a **Code** node (`Filter Scheduled Services`) to parse tags, filter for active scheduled services (`auto-schedule=office-hours`), and handle mode-specific filtering.

4. **Implement Mode Routing & Nightly/Morning Operations:**
   - Add a **Switch** node (`Route by Service Mode`) with 3 outputs based on `$json.mode` (`stop`, `start`, `restart`).
   - **Stop Branch (Output 0):** Connect to **Execute Workflow** (`Save ECS Task Count`, action `TagResource`, key `saved-desired-count`). Connect to **Execute Workflow** (`Scale ECS to Zero`, action `UpdateService`, desired count `0`). Connect to a **Code** node (`Compile Stop Summary`) and an **HTTP Request** node (`Notify Stop Summary to Teams`) pointing to `teamsWebhookUrl`.
   - **Start Branch (Output 1):** Connect to **Execute Workflow** (`Restore ECS Task Count`, action `UpdateService`, desired count `savedCount`). Connect to **Execute Workflow** (`Clear ECS Schedule Tags`, action `UntagResource`, keys `saved-desired-count`, `override-until`). Connect to a **Code** node (`Compile Start Summary`) and an **HTTP Request** node (`Notify Start Summary to Teams`).

5. **Construct Self-Service Restart & Health Polling:**
   - **Restart Branch (Output 2):** Connect to an **If** node (`Check Services to Restart`). If empty, connect to a **Form** completion node (`Display No Restart Options`). If valid, connect to a **Form** node (`Select ECS Services`) with multi-select service and duration dropdowns.
   - Connect to a **Code** node (`Prepare ECS Restart`) to calculate `override-until` timestamps and validate request limits.
   - Connect to **Execute Workflow** (`Mark ECS for Override`, action `TagResource`, key `override-until`).
   - Connect to **Execute Workflow** (`Start ECS Services`, action `UpdateService`).
   - Connect to a **Form** completion node (`Display Confirmation Form`) and a **Code** node (`Compile Restart Message`) followed by an **HTTP Request** (`Notify Restart Message to Teams`).
   - Add a **Wait** node (`Wait 30 Seconds`) set to 30 seconds.
   - Add a **Code** node (`Organize Services by Cluster`) and an **Execute Workflow** node (`Verify ECS Services`, action `DescribeServices`).
   - Add a **Code** node (`Assess ECS Health`) and a **Switch** node (`Check ECS Health Status`) evaluating `status` (`ready`, `waiting`, `failed`).
   - **Ready Route:** Connect to **Code** (`Compose Ready Notification`) and **HTTP Request** (`Notify Ready Status to Teams`).
   - **Waiting Route:** Loop back to `Wait 30 Seconds`.
   - **Failure Route:** Connect to **Code** (`Compose Failure Notification`) and **HTTP Request** (`Notify Failure to Teams`).

6. **Add Expiration & Re-Stop Automation:**
   - From both Ready and Failure notification paths, connect to a **Wait** node (`Wait Until Specified Time`) resuming at specific time `{{ $('Prepare ECS Restart').first().json.overrideUntil }}`.
   - Connect to a **Code** node (`Classify Services for Re-stop`) and **Execute Workflow** (`Verify Before Re-Stop`, action `DescribeServices`).
   - Connect to a **Code** node (`Decide on Re-stop Action`) checking office hours and override tag matching.
   - Connect to **Execute Workflow** (`Re-Stop ECS Services`, action `UpdateService`, desired count `0`).
   - Connect to **Execute Workflow** (`Remove ECS Override`, action `UntagResource`, key `override-until`).
   - Connect to a **Code** node (`Compose Stopped Again Notification`) and **HTTP Request** (`Notify Stopped Again to Teams`).

7. **Build the Sub-Workflow Multi-Account ECS Engine:**
   - Add an **Execute Workflow Trigger** node (`On ECS API Call`).
   - Connect to a **Switch** node (`Route by AWS Account`) evaluating `$json.account`.
   - Add **HTTP Request** nodes (`Handle ECS for Account-A`, `Handle ECS for Account-B`) configured with **AWS Assume Role credentials**, URL `https://ecs.={{ $json.region }}.amazonaws.com/`, header `X-Amz-Target: AmazonEC2ContainerServiceV20141113.{{ $json.action }}`, and POST JSON body `{{ JSON.stringify($json.body) }}`.
   - Add a **Stop and Error** node (`Halt on Unknown Account`) for unhandled account fallback.
   - Connect all HTTP handlers to a **Code** node (`Return ECS API Results`) that parses raw text/JSON responses and returns the merged context.

8. **Workflow Settings:**
   - Open workflow settings and set **Timezone** to `UTC` (matching global parameters). Set **Execution Order** to `v1`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Microsoft Teams Adaptive Cards Schema Reference | [Adaptive Cards Official Documentation](http://adaptivecards.io/schemas/adaptive-card.json) |
| AWS ECS API Reference (Action Targets) | [Amazon ECS API Reference](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/Welcome.html) |