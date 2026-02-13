Schedule and track interviews using Calendly, Zoom, Asana, and Gmail

https://n8nworkflows.xyz/workflows/schedule-and-track-interviews-using-calendly--zoom--asana--and-gmail-12271


# Schedule and track interviews using Calendly, Zoom, Asana, and Gmail

## 1. Workflow Overview

**Purpose:** Automate interview scheduling operations end-to-end when a candidate books via **Calendly**: normalize booking data, decide interview type (**HR vs Technical**), create a **Zoom meeting**, create an **Asana task** containing the Zoom link and timing, then notify the right people via **Gmail**. A separate **Error Trigger** path emails an alert if any node fails.

**Typical use cases:**
- Recruiting teams that want consistent interview logistics without manual coordination
- Ensuring every booking generates: Zoom meeting + tracking task + timely notifications

### 1.1 Input Reception & Data Preparation
Receives Calendly “invitee.created” events and extracts/normalizes the fields used downstream.

### 1.2 Interview Type Routing
Classifies the interview as HR/People/Recruiter vs everything else (treated as “Technical” path).

### 1.3 Meeting & Task Creation
Creates a Zoom meeting and an Asana task (HR task or Technical task) with due date and Zoom link.

### 1.4 Notifications
Sends an email notification to the HR team or the technical interviewer.

### 1.5 Error Handling
Catches workflow errors globally and emails diagnostic details.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception & Data Preparation
**Overview:** Listens for Calendly bookings and converts Calendly’s webhook payload into a compact, consistent JSON object used by all later nodes.

**Nodes involved:**
- Receive interview booking
- Prepare interview data

#### Node: Receive interview booking
- **Type / role:** `Calendly Trigger` — webhook trigger for scheduling events.
- **Configuration (interpreted):**
  - **Scope:** Organization
  - **Event:** `invitee.created` (fires when an invitee schedules)
  - **Auth:** OAuth2 (Calendly)
- **Inputs:** None (trigger node)
- **Outputs:** Calendly webhook payload at `{{$json.payload}}` (used by the Code node)
- **Connections:** → *Prepare interview data*
- **Failure/edge cases:**
  - OAuth token expired/revoked
  - Calendly webhook disabled or wrong scope
  - Payload shape differences if Calendly changes API fields (would break the Code node)

#### Node: Prepare interview data
- **Type / role:** `Code` — transforms raw payload into normalized interview fields.
- **Configuration (interpreted):**
  - Reads `const event = $json.payload;`
  - Outputs:
    - `event_id`: `event.uuid`
    - `interview_type`: `event.scheduled_event.name`
    - `candidate_name`: `event.name`
    - `start_time`: `event.scheduled_event.start_time`
    - `end_time`: `event.scheduled_event.end_time`
    - `timezone`: `event.timezone`
- **Key variables/expressions:** `$json.payload`, returning an array of one item.
- **Inputs:** From *Receive interview booking*
- **Outputs:** Normalized JSON for routing and meeting/task creation
- **Connections:** → *Route by interview type*
- **Failure/edge cases:**
  - `payload` missing (expression/JS error)
  - `scheduled_event` missing or renamed
  - Time fields not in a format expected by downstream nodes (Zoom/Asana)

---

### Block 1.2 — Interview Type Routing
**Overview:** Determines whether the interview is HR-related by keyword matching on the interview type name, otherwise routes to the “Technical” branch.

**Nodes involved:**
- Route by interview type

#### Node: Route by interview type
- **Type / role:** `IF` — conditional branch.
- **Configuration (interpreted):**
  - Condition group: **OR**
  - Checks whether `{{$json.interview_type}}` **contains** any of:
    - `"HR"`
    - `"People"`
    - `"Recruiter"`
  - **True output (index 0):** HR path
  - **False output (index 1):** Technical path
- **Key expressions:** `={{ $json.interview_type }}`
- **Inputs:** From *Prepare interview data*
- **Outputs:** Same item passed to either branch
- **Connections:**
  - True → *Create Zoom meeting* (HR branch)
  - False → *Create Zoom meeting1* (Technical branch)
- **Failure/edge cases:**
  - Case sensitivity: “hr” would not match “HR” (the node is configured as case-sensitive)
  - Interview naming conventions in Calendly must contain these keywords reliably, otherwise misrouting occurs

---

### Block 1.3 — Meeting & Task Creation
**Overview:** Creates a Zoom meeting for the scheduled time and then creates a corresponding Asana task with the Zoom join URL and due date.

**Nodes involved (HR branch):**
- Create Zoom meeting
- Create HR Interview Task

**Nodes involved (Technical branch):**
- Create Zoom meeting1
- Create Technical Interview Task

#### Node: Create Zoom meeting (HR branch)
- **Type / role:** `Zoom` — create scheduled meeting.
- **Configuration (interpreted):**
  - **Topic:** `{{ interview_type }} – {{ candidate_name }}`
  - **Type:** Scheduled meeting (type = 2)
  - **Agenda:** `Interview with {{ candidate_name }}`
  - **Duration:** 60 minutes
  - **Time zone:** `{{ timezone }}`
  - **Start time:** `{{ start_time }}`
  - **Auth:** OAuth2 (Zoom)
- **Inputs:** True output from *Route by interview type*
- **Outputs:** Zoom meeting object (used for `join_url`, `start_time`)
- **Connections:** → *Create HR Interview Task*
- **Failure/edge cases:**
  - Zoom OAuth expired / insufficient scopes
  - `start_time` format/timezone mismatch causing Zoom API validation errors
  - Rate limits or transient Zoom API failures

#### Node: Create Zoom meeting1 (Technical branch)
- **Type / role:** `Zoom` — create scheduled meeting (same config as HR branch).
- **Configuration (interpreted):** Same as *Create Zoom meeting*, but on the technical branch.
- **Inputs:** False output from *Route by interview type*
- **Outputs:** Zoom meeting object
- **Connections:** → *Create Technical Interview Task*
- **Failure/edge cases:** Same as HR Zoom node

#### Node: Create HR Interview Task
- **Type / role:** `Asana` — creates a task for HR interview tracking.
- **Configuration (interpreted):**
  - **Task name:** `HR Interview – {{ candidate_name }}` (candidate name pulled via node reference)
  - **Workspace:** fixed ID `1205967598025927`
  - **Project:** `1212555032005130`
  - **Assignee:** `1212557659831213`
  - **Notes:** `Zoom link: {{ $json.join_url }}` (reads join URL from the Zoom node output)
  - **Due on:** `{{ $json.start_time }}`
  - **Auth:** OAuth2 (Asana)
- **Key expressions/variables:**
  - Candidate name referenced as: `{{ $('Route by interview type').item.json.candidate_name }}`
  - Zoom fields referenced as `$json.join_url`, `$json.start_time` (coming from Zoom node output)
- **Inputs:** From *Create Zoom meeting* (so `$json` here is the Zoom response)
- **Outputs:** Asana task object
- **Connections:** → *Notify HR team*
- **Failure/edge cases:**
  - Asana OAuth expired / missing permissions to workspace/project
  - Using `due_on` with a datetime (Asana `due_on` is typically date-only; Asana also has `due_at` for datetime). If Asana rejects the value format, task creation fails.
  - Hard-coded IDs (workspace/project/assignee) must exist in the connected Asana account.

#### Node: Create Technical Interview Task
- **Type / role:** `Asana` — creates a task for technical interview tracking.
- **Configuration (interpreted):**
  - **Task name:** `Technical Interview – {{ candidate_name }}`
  - Same workspace/project/assignee, notes, and due date strategy as HR task
- **Key expressions/variables:**
  - Candidate name via: `{{ $('Route by interview type').item.json.candidate_name }}`
  - Zoom fields from `$json.join_url`, `$json.start_time`
- **Inputs:** From *Create Zoom meeting1*
- **Outputs:** Asana task object
- **Connections:** → *Notify interviewer*
- **Failure/edge cases:** Same as HR task node (including `due_on` formatting)

---

### Block 1.4 — Notifications
**Overview:** Emails the relevant party with candidate name, time, and Zoom link.

**Nodes involved:**
- Notify HR team
- Notify interviewer

#### Node: Notify HR team
- **Type / role:** `Gmail` — send email.
- **Configuration (interpreted):**
  - **To:** `user@example.com` (placeholder)
  - **Subject:** `HR Interview Scheduled – {{ candidate_name }}`
  - **Body:** Includes:
    - Candidate: from *Route by interview type*
    - Time + Zoom URL: from *Create Zoom meeting*
  - **Email type:** text
- **Key expressions:**
  - `{{ $('Route by interview type').item.json.candidate_name }}`
  - `{{ $('Create Zoom meeting').item.json.start_time }}`
  - `{{ $('Create Zoom meeting').item.json.join_url }}`
- **Inputs:** From *Create HR Interview Task* (but it references upstream nodes explicitly)
- **Outputs:** Gmail send result
- **Failure/edge cases:**
  - Gmail OAuth expired / restricted scopes
  - If the Zoom node did not run (branching change) the node references would fail; current graph ensures it runs on this branch.

#### Node: Notify interviewer
- **Type / role:** `Gmail` — send email to technical interviewer.
- **Configuration (interpreted):**
  - **To:** `youremail@email.com` (placeholder; should be interviewer email)
  - **Subject:** `Technical Interview Scheduled – {{ candidate_name }}`
  - **Body:** Candidate + time + Zoom URL from the technical Zoom node
  - **Email type:** text
- **Key expressions:**
  - `{{ $('Route by interview type').item.json.candidate_name }}`
  - `{{ $('Create Zoom meeting1').item.json.start_time }}`
  - `{{ $('Create Zoom meeting1').item.json.join_url }}`
- **Inputs:** From *Create Technical Interview Task*
- **Outputs:** Gmail send result
- **Failure/edge cases:** Same as HR email node; additionally ensure the “To” address is set correctly.

---

### Block 1.5 — Error Handling
**Overview:** If any node in the workflow errors, the Error Trigger starts a separate path that emails error details for rapid diagnosis.

**Nodes involved:**
- Error Handler Trigger
- Error Email Notification

#### Node: Error Handler Trigger
- **Type / role:** `Error Trigger` — starts when another workflow execution fails.
- **Configuration (interpreted):** Default; emits error context including node and message.
- **Inputs:** None (system-triggered)
- **Outputs:** Error object containing `node`, `error`, `timestamp`, etc.
- **Connections:** → *Error Email Notification*
- **Failure/edge cases:**
  - If the Gmail credential used for alerts is broken, you lose error visibility.

#### Node: Error Email Notification
- **Type / role:** `Gmail` — sends error alert email.
- **Configuration (interpreted):**
  - **To:** `user@example.com` (placeholder)
  - **Subject:** “The Orchestrate interview scheduling with Calendly, Zoom, and Asana workflow encountered an error.”
  - **Body:** Includes node name, error message, timestamp:
    - `{{ $json.node.name }}`
    - `{{ $json.error.message }}`
    - `{{ $json.timestamp }}`
  - **Email type:** text
- **Inputs:** From *Error Handler Trigger*
- **Outputs:** Gmail send result
- **Failure/edge cases:**
  - Gmail OAuth expired / quota limits
  - If error payload shape differs (older/newer n8n versions), expressions may need adjustment

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow overview | Sticky Note | Documentation / overview |  |  | ## Workflow Overview<br><br>### How it works<br><br>This workflow coordinates the scheduling of interviews using Asana, Zoom, and Calendly. The workflow records the event when a candidate schedules an interview, normalizes the interview data, and uses the event details to determine the type of interview.<br><br>The workflow uses conditional routing to assign the interview to the right interviewer, set up a Zoom meeting, and produce an Asana task with all necessary context. The appropriate stakeholders are then notified at the appropriate moment using role-based notifications.<br><br>The automation is intended to minimize human coordination, avoid scheduling mistakes, and offer a dependable, production-ready interview flow appropriate for expanding teams.<br><br>### Setup steps<br><br>1. Connect Calendly, Zoom, Asana, and Gmail credentials.<br>2. Set up keywords for the interview type (e.g., HR or Technical).<br>3. Update interviewer email addresses and Asana project IDs.<br>4. If necessary, modify the task name or notification message. |
| Receive interview booking | Calendly Trigger | Entry point: receive booking event |  | Prepare interview data | ### Trigger and data preparation<br><br>Receives interview booking events from Calendly and prepares the interview details for further steps. |
| Prepare interview data | Code | Normalize payload fields | Receive interview booking | Route by interview type | ### Trigger and data preparation<br><br>Receives interview booking events from Calendly and prepares the interview details for further steps. |
| Route by interview type | IF | Branch HR vs Technical | Prepare interview data | Create Zoom meeting; Create Zoom meeting1 | ### Interview Type Routing<br><br>Checks the interview type (HR or Technical) and routes the workflow to the correct path. |
| Create Zoom meeting | Zoom | Create Zoom meeting (HR branch) | Route by interview type | Create HR Interview Task | ### Meeting and Task Creation<br><br>Creates a Zoom meeting for the interview and an Asana task with the meeting link and interview details. |
| Create HR Interview Task | Asana | Create Asana task (HR interview) | Create Zoom meeting | Notify HR team | ### Meeting and Task Creation<br><br>Creates a Zoom meeting for the interview and an Asana task with the meeting link and interview details. |
| Notify HR team | Gmail | Email HR notification | Create HR Interview Task |  | ### Notifications<br><br>Sends interview notifications to the right people. |
| Create Zoom meeting1 | Zoom | Create Zoom meeting (Technical branch) | Route by interview type | Create Technical Interview Task | ### Meeting and Task Creation<br><br>Creates a Zoom meeting for the interview and an Asana task with the meeting link and interview details. |
| Create Technical Interview Task | Asana | Create Asana task (Technical interview) | Create Zoom meeting1 | Notify interviewer | ### Meeting and Task Creation<br><br>Creates a Zoom meeting for the interview and an Asana task with the meeting link and interview details. |
| Notify interviewer | Gmail | Email technical interviewer | Create Technical Interview Task |  | ### Notifications<br><br>Sends interview notifications to the right people. |
| Error Handler Trigger | Error Trigger | Global error entry point |  | Error Email Notification | ## 🚨 Error Handling<br><br>Catches any workflow failure and posts an alert to Gmail.<br>Includes node name, error message, and timestamp for quick debugging. |
| Error Email Notification | Gmail | Send error alert email | Error Handler Trigger |  | ## 🚨 Error Handling<br><br>Catches any workflow failure and posts an alert to Gmail.<br>Includes node name, error message, and timestamp for quick debugging. |
| Sticky Note | Sticky Note | Section label/comment |  |  | ### Trigger and data preparation<br><br>Receives interview booking events from Calendly and prepares the interview details for further steps. |
| Sticky Note1 | Sticky Note | Section label/comment |  |  | ### Interview Type Routing<br><br>Checks the interview type (HR or Technical) and routes the workflow to the correct path. |
| Sticky Note2 | Sticky Note | Section label/comment |  |  | ### Meeting and Task Creation<br><br>Creates a Zoom meeting for the interview and an Asana task with the meeting link and interview details. |
| Sticky Note3 | Sticky Note | Section label/comment |  |  | ### Notifications<br><br>Sends interview notifications to the right people. |
| Sticky Note7 | Sticky Note | Section label/comment |  |  | ## 🚨 Error Handling<br><br>Catches any workflow failure and posts an alert to Gmail.<br>Includes node name, error message, and timestamp for quick debugging. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Schedule interviews using Calendly, Zoom, and Asana”** (or your preferred name).
   - (Optional) Add a Sticky Note with the “Workflow overview” text.

2. **Add trigger: Calendly**
   - Add node: **Calendly Trigger** named **“Receive interview booking”**
   - Configure:
     - Authentication: **OAuth2**
     - Scope: **Organization**
     - Events: **invitee.created**
   - Create/attach **Calendly OAuth2** credentials.
   - Save so n8n can register the webhook with Calendly.

3. **Add data normalization: Code**
   - Add node: **Code** named **“Prepare interview data”**
   - Paste logic equivalent to:
     - Read `payload` from the trigger output and map to:
       - `event_id`, `interview_type`, `candidate_name`, `start_time`, `end_time`, `timezone`
   - Connect: **Receive interview booking → Prepare interview data**

4. **Add routing: IF**
   - Add node: **IF** named **“Route by interview type”**
   - Condition:
     - Use **String → contains** on `{{$json.interview_type}}`
     - OR group with right-hand values: `HR`, `People`, `Recruiter`
     - Keep **case sensitive** if you enforce naming conventions; otherwise disable case sensitivity or normalize casing in the Code node.
   - Connect: **Prepare interview data → Route by interview type**

5. **HR branch: Zoom meeting**
   - Add node: **Zoom** named **“Create Zoom meeting”**
   - Operation: **Create Meeting** (scheduled meeting)
   - Configure:
     - Topic: `{{$json.interview_type}} – {{$json.candidate_name}}`
     - Agenda: `Interview with {{$json.candidate_name}}`
     - Duration: `60`
     - Time zone: `{{$json.timezone}}`
     - Start time: `{{$json.start_time}}`
     - Authentication: **OAuth2**
   - Attach **Zoom OAuth2** credentials.
   - Connect: **Route by interview type (true) → Create Zoom meeting**

6. **HR branch: Asana task**
   - Add node: **Asana** named **“Create HR Interview Task”**
   - Operation: **Create Task**
   - Configure:
     - Workspace: your Asana workspace (ID in the original: `1205967598025927`)
     - Name: `HR Interview – {{ $('Route by interview type').item.json.candidate_name }}`
     - Assignee: your HR assignee user ID (original: `1212557659831213`)
     - Projects: add the target project (original: `1212555032005130`)
     - Notes: `Zoom link: {{ $json.join_url }}`
     - Due date field set to the interview start (original used **due_on** = `{{$json.start_time}}`)
       - If Asana rejects datetime, switch to **due_at** (datetime) or transform to date-only for **due_on**.
   - Attach **Asana OAuth2** credentials.
   - Connect: **Create Zoom meeting → Create HR Interview Task**

7. **HR branch: Gmail notification**
   - Add node: **Gmail** named **“Notify HR team”**
   - Operation: **Send**
   - Configure:
     - To: HR team email (original placeholder: `user@example.com`)
     - Subject: `HR Interview Scheduled – {{ $('Route by interview type').item.json.candidate_name }}`
     - Body (text), including:
       - Candidate from Route node
       - Time and Zoom link from **Create Zoom meeting**
   - Attach **Gmail OAuth2** credentials.
   - Connect: **Create HR Interview Task → Notify HR team**

8. **Technical branch: Zoom meeting**
   - Add node: **Zoom** named **“Create Zoom meeting1”**
   - Same meeting configuration as HR branch (topic, agenda, duration, timezone, start time).
   - Connect: **Route by interview type (false) → Create Zoom meeting1**

9. **Technical branch: Asana task**
   - Add node: **Asana** named **“Create Technical Interview Task”**
   - Same workspace/project/assignee approach as HR task, but:
     - Name: `Technical Interview – {{ $('Route by interview type').item.json.candidate_name }}`
   - Connect: **Create Zoom meeting1 → Create Technical Interview Task**

10. **Technical branch: Gmail notification**
    - Add node: **Gmail** named **“Notify interviewer”**
    - Configure:
      - To: interviewer email (original placeholder: `youremail@email.com`)
      - Subject/body referencing **Create Zoom meeting1** for time/link
    - Connect: **Create Technical Interview Task → Notify interviewer**

11. **Add global error handling**
    - Add node: **Error Trigger** named **“Error Handler Trigger”**
    - Add node: **Gmail** named **“Error Email Notification”**
      - To: ops/admin email (original placeholder: `user@example.com`)
      - Subject: error subject (customize)
      - Body includes:
        - `{{$json.node.name}}`, `{{$json.error.message}}`, `{{$json.timestamp}}`
    - Connect: **Error Handler Trigger → Error Email Notification**

12. **Finalize**
    - Ensure all credentials are valid:
      - Calendly OAuth2 (webhook permission)
      - Zoom OAuth2 (meeting create scope)
      - Asana OAuth2 (task create + project access)
      - Gmail OAuth2 (send email)
    - Activate the workflow.
    - Run an end-to-end test by booking a Calendly event whose name contains “HR” (and one that doesn’t) to verify branching.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer (source/workflow compliance statement) |
| Keyword routing depends on Calendly event naming (“HR”, “People”, “Recruiter”); case sensitivity may cause misrouting. | Applies to “Route by interview type” block |
| Asana due date field: workflow uses `due_on` with `start_time`; if Asana rejects datetime, use `due_at` or convert to date-only. | Applies to both Asana task nodes |
| Replace placeholder recipient emails (`user@example.com`, `youremail@email.com`) with real distribution lists/users. | Applies to Gmail notification nodes and error alert node |