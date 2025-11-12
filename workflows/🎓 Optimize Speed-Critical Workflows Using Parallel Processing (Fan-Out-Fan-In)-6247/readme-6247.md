🎓 Optimize Speed-Critical Workflows Using Parallel Processing (Fan-Out/Fan-In)

https://n8nworkflows.xyz/workflows/---optimize-speed-critical-workflows-using-parallel-processing--fan-out-fan-in--6247


# 🎓 Optimize Speed-Critical Workflows Using Parallel Processing (Fan-Out/Fan-In)

---

### 1. Workflow Overview

This workflow demonstrates an advanced parallel processing pattern (Fan-Out/Fan-In) designed to optimize speed-critical projects by running multiple independent tasks simultaneously. It is ideal for scenarios where independent subtasks (such as generating different types of content via AI) can be executed concurrently to reduce total completion time.

The workflow is divided into two main logical blocks:

- **1.1 Main Workflow (Project Manager)**  
  Responsible for initiating the project, defining tasks, dispatching them in parallel to sub-workflows (specialist teams), and waiting for all tasks to complete before aggregating the results and continuing.

- **1.2 Sub-Workflow (Specialist Teams)**  
  Receives individual work orders, dispatches each to the appropriate specialist node (AI content generation agents), and reports completion back to the main workflow via a project dashboard and status updates.

The core architecture leverages n8n features such as Static Data (shared memory), sub-workflows, webhook-based waiting and resuming, and conditional routing for task dispatch and completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Main Workflow (Project Manager)

**Overview:**  
This block manages the overall project lifecycle from start to finish. It defines independent tasks, fans them out to specialist teams via sub-workflows, waits for all parallel tasks to complete, and finally aggregates and processes results.

**Nodes Involved:**  
- Start Project (Manual Trigger)  
- The Project Brief (Set)  
- Split Out Tasks (Split Out)  
- Assign Tasks to Teams (Execute Workflow)  
- Wait for All Teams to Finish (Wait)  
- Project Complete! (Set)  
- The Project Dashboard (Code)  
- Is Project Complete? (If)  
- Resume Parent Workflow (HTTP Request)  
- Start Process (Execute Workflow)

**Node Details:**

- **Start Project**  
  - Type: Manual Trigger  
  - Role: Entry point to start the entire project manually.  
  - Config: No parameters; manual user trigger.  
  - Inputs: None   
  - Outputs: To The Project Brief  
  - Edge Cases: None typical; user must trigger manually.

- **The Project Brief**  
  - Type: Set  
  - Role: Defines the array of independent tasks with their process names and associated data.  
  - Config: Assigns an array `tasks` with three objects, each representing a different content generation task (`write_description`, `write_ad_copy`, `write_email`) with required input data.  
  - Key Expressions: Hardcoded JSON array assigned to `tasks` field.  
  - Inputs: From Start Project  
  - Outputs: To Split Out Tasks  
  - Edge Cases: Hardcoded tasks; modifying the array structure changes the downstream flow.

- **Split Out Tasks**  
  - Type: Split Out  
  - Role: Splits the `tasks` array into individual items for parallel processing.  
  - Config: Splits on the `tasks` field.  
  - Inputs: From The Project Brief  
  - Outputs: To Assign Tasks to Teams  
  - Edge Cases: Empty or malformed `tasks` array could cause no tasks to dispatch.

- **Assign Tasks to Teams**  
  - Type: Execute Workflow (Sub-Workflow Call)  
  - Role: Fan-Out moment; invokes the same workflow as a sub-workflow for each task item asynchronously (does not wait for completion).  
  - Config:  
    - Workflow ID: Current workflow ID (self-call)  
    - Passes inputs `process`, `data`, `status` (set to `"initialise"`), `main_execution_id`, and `resume_url`.  
    - `Wait For Sub-Workflow` is **OFF**, allowing parallel firing without blocking main flow.  
  - Inputs: From Split Out Tasks  
  - Outputs: To Wait for All Teams to Finish  
  - Edge Cases: If sub-workflow call fails or is delayed, overall parallelism may be affected.

- **Wait for All Teams to Finish**  
  - Type: Wait (Webhook)  
  - Role: Pauses the main workflow indefinitely, waiting for a POST request to its webhook URL to resume.  
  - Config: Waits on a unique webhook, configured with POST method and `resume` mode.  
  - Inputs: From Assign Tasks to Teams  
  - Outputs: To Project Complete!  
  - Edge Cases: If webhook URL is not called (no resume), workflow stays paused indefinitely.

- **Project Complete!**  
  - Type: Set  
  - Role: Aggregates the results from all completed tasks and calculates timing statistics such as total time saved by parallelization vs sequential execution.  
  - Config: Assigns multiple fields extracting outputs from the final aggregated data (email text, ad copy, description, duration statistics).  
  - Inputs: From Wait for All Teams to Finish (resumed webhook)  
  - Outputs: None (end of main flow)  
  - Edge Cases: Requires all tasks to have completed successfully with correct data format.

- **The Project Dashboard (Code)**  
  - Type: Code  
  - Role: Central brain and state manager using Static Data as a shared project dashboard. Handles project initialization and task completion updates.  
  - Config: JavaScript code that:  
    - On `status: 'initialise'`, creates new project entry and marks all tasks as pending.  
    - On `status: 'completed'`, updates task status, calculates duration, and checks if all tasks are done. If all complete, returns a special item triggering resumption of the main workflow's wait node.  
  - Inputs: From Check Work Order Status  
  - Outputs: To Is Project Complete?  
  - Edge Cases: Static Data persistence issues, concurrency race conditions, malformed inputs.

- **Is Project Complete?**  
  - Type: If  
  - Role: Checks if the project dashboard code has indicated all tasks are done (`end_process` boolean).  
  - Config: Condition on `$json.end_process === true`.  
  - Inputs: From The Project Dashboard (Code)  
  - Outputs:  
    - True branch: Resume Parent Workflow  
    - False branch: Start Process (calls sub-workflow with pending tasks)  
  - Edge Cases: Incorrect or missing `end_process` flag could stall workflow.

- **Resume Parent Workflow**  
  - Type: HTTP Request  
  - Role: Sends a POST request to the wait node's unique webhook URL with final project results to resume the paused main workflow.  
  - Config:  
    - URL from `$json.resume_url`  
    - JSON body with aggregated results `$json.response`  
    - Method: POST  
  - Inputs: True branch from Is Project Complete?  
  - Outputs: None  
  - Edge Cases: HTTP errors (timeout, 4xx/5xx), invalid URL.

- **Start Process**  
  - Type: Execute Workflow (Sub-Workflow Call)  
  - Role: If project is not complete, triggers the sub-workflow again with `status: 'pending'` for ongoing processing of tasks.  
  - Config: Similar to Assign Tasks to Teams but with `status: 'pending'`.  
  - Inputs: False branch from Is Project Complete?  
  - Outputs: None  
  - Edge Cases: Potential for infinite loops if tasks never complete.

---

#### 2.2 Sub-Workflow (Specialist Teams)

**Overview:**  
This block processes individual task work orders dispatched by the main workflow. It routes each task to the appropriate team based on `process` type, executes AI-based content generation, then reports completion back to the project dashboard.

**Nodes Involved:**  
- Receive Work Order (Execute Workflow Trigger)  
- Check Work Order Status (If)  
- The Dispatcher (Switch)  
- Description Team (LangChain Agent)  
- Ad Copy Team (LangChain Agent)  
- Email Team (LangChain Agent)  
- Report Back to Manager (Execute Workflow)

**Node Details:**

- **Receive Work Order**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for receiving a single work order item from the main workflow's sub-workflow call.  
  - Config: Receives parameters `main_execution_id`, `process`, `data` (object), `status`, and `resume_url`.  
  - Inputs: External call from main workflow  
  - Outputs: To Check Work Order Status  
  - Edge Cases: Missing or malformed input fields.

- **Check Work Order Status**  
  - Type: If  
  - Role: Checks if the work order status is `"pending"` to proceed.  
  - Config: Condition on `$json.status === "pending"`.  
  - Inputs: From Receive Work Order  
  - Outputs:  
    - True branch: To The Dispatcher  
    - False branch: To The Project Dashboard (Code)  
  - Edge Cases: Unexpected status values.

- **The Dispatcher**  
  - Type: Switch  
  - Role: Routes the task to the correct specialist team node based on the `process` field (`write_description`, `write_ad_copy`, `write_email`).  
  - Config: String equality conditions on `$json.process`.  
  - Inputs: From Check Work Order Status (true)  
  - Outputs: To Description Team, Ad Copy Team, or Email Team respectively  
  - Edge Cases: Unknown or unsupported process names.

- **Description Team**  
  - Type: LangChain Agent (AI Node)  
  - Role: Generates a product description based on the provided product name.  
  - Config: Prompt template `"Write a product description for: {{$json.data.product}}"` with a system message instructing no preamble. Temperature set to 0 for deterministic output.  
  - Inputs: From The Dispatcher (for `write_description`)  
  - Outputs: To Report Back to Manager  
  - Edge Cases: API errors, invalid input data, model unavailability.

- **Ad Copy Team**  
  - Type: LangChain Agent (AI Node)  
  - Role: Creates ad copy for the product with a specified tone.  
  - Config: Prompt `"Write ad copy for: {{$json.data.product}}. The tone should be: {{$json.data.tone}}"` with same system message and temperature 0.  
  - Inputs: From The Dispatcher (for `write_ad_copy`)  
  - Outputs: To Report Back to Manager  
  - Edge Cases: Same as Description Team.

- **Email Team**  
  - Type: LangChain Agent (AI Node)  
  - Role: Writes a cold email targeted at a specified audience for the product.  
  - Config: Prompt `"Write a cold email about {{$json.data.product}} for the following audience: {{$json.data.audience}}"`, same system message and temperature 0.  
  - Inputs: From The Dispatcher (for `write_email`)  
  - Outputs: To Report Back to Manager  
  - Edge Cases: Same as above.

- **Report Back to Manager**  
  - Type: Execute Workflow (Sub-Workflow Call)  
  - Role: Calls the sub-workflow again but with `status: 'completed'` to update the project dashboard with results.  
  - Config: Passes current task data as `data`, `status: 'completed'`, and metadata like `process` and `main_execution_id`.  
  - Inputs: From each Specialist Team node  
  - Outputs: None  
  - Edge Cases: Failure to report back would stall main workflow progress.

---

### 3. Summary Table

| Node Name                 | Node Type                    | Functional Role                                    | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                            |
|---------------------------|------------------------------|---------------------------------------------------|-------------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Start Project             | Manual Trigger               | Entry point to start the project                   | —                             | The Project Brief             | © 2025 Lucas Peyrin                                                                                                     |
| The Project Brief         | Set                          | Defines array of independent tasks for the project| Start Project                 | Split Out Tasks               | ### The Project Brief: Defines all independent tasks with process and data.                                            |
| Split Out Tasks           | Split Out                    | Splits tasks array into individual task items     | The Project Brief             | Assign Tasks to Teams         | © 2025 Lucas Peyrin                                                                                                     |
| Assign Tasks to Teams     | Execute Workflow             | Fan-Out: calls sub-workflow for each task asynchronously | Split Out Tasks               | Wait for All Teams to Finish  | ### Assign Tasks (Fan-Out): Wait for Sub-Workflow OFF for immediate parallel dispatch.                                  |
| Wait for All Teams to Finish | Wait (Webhook)               | Pauses main workflow until all teams report back  | Assign Tasks to Teams         | Project Complete!             | ### Wait for Completion (Pause): Wait node pauses until resumed by POST request.                                       |
| Project Complete!         | Set                          | Aggregates final results and calculates timing    | Wait for All Teams to Finish  | —                            | ### Project Complete! (Fan-In): Aggregates all parallel results after wait node resumes.                                |
| The Project Dashboard (Code) | Code                         | Manages project state and task statuses using Static Data | Check Work Order Status       | Is Project Complete?          | ### The Project Dashboard: Central brain using Static Data to track project progress and trigger workflow resumption.  |
| Is Project Complete?      | If                           | Checks if all tasks have completed                  | The Project Dashboard (Code)  | Resume Parent Workflow, Start Process | © 2025 Lucas Peyrin                                                                                                     |
| Resume Parent Workflow    | HTTP Request                 | Sends POST request to resume paused main workflow  | Is Project Complete?          | —                            | ### Resume Parent Workflow: Wakes main workflow by POSTing results to wait node URL.                                   |
| Start Process             | Execute Workflow             | Calls sub-workflow with pending task status        | Is Project Complete?          | —                            | © 2025 Lucas Peyrin                                                                                                     |
| Receive Work Order        | Execute Workflow Trigger    | Entry point for specialist teams receiving tasks   | —                             | Check Work Order Status       | ### Receive Work Order: Entry point for specialist teams receiving single task.                                         |
| Check Work Order Status   | If                           | Checks if task status is "pending"                  | Receive Work Order            | The Dispatcher, The Project Dashboard (Code) | © 2025 Lucas Peyrin                                                                                                     |
| The Dispatcher            | Switch                       | Routes tasks to correct specialist team            | Check Work Order Status       | Description Team, Ad Copy Team, Email Team | ### The Dispatcher: Dispatches tasks based on process name.                                                             |
| Description Team          | LangChain Agent              | Generates product description                       | The Dispatcher               | Report Back to Manager        | © 2025 Lucas Peyrin                                                                                                     |
| Ad Copy Team              | LangChain Agent              | Generates ad copy with tone                          | The Dispatcher               | Report Back to Manager        | © 2025 Lucas Peyrin                                                                                                     |
| Email Team                | LangChain Agent              | Generates cold email for target audience            | The Dispatcher               | Report Back to Manager        | © 2025 Lucas Peyrin                                                                                                     |
| Report Back to Manager    | Execute Workflow             | Reports task completion back to project dashboard  | Description Team, Ad Copy Team, Email Team | —                            | ### Report Back to Manager: Calls sub-workflow with status 'completed' to update dashboard.                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create node:** *Start Project*  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create node:** *The Project Brief*  
   - Type: Set  
   - Add an assignment: Field `tasks` (array), value:  
     ```json
     [
       {"process": "write_description", "data": {"product": "Super Widget"}},
       {"process": "write_ad_copy", "data": {"product": "Super Widget", "tone": "Excited"}},
       {"process": "write_email", "data": {"product": "Super Widget", "audience": "Tech CEOs"}}
     ]
     ```  
   - Connect from *Start Project*.

3. **Create node:** *Split Out Tasks*  
   - Type: Split Out  
   - Field to split out: `tasks`  
   - Connect from *The Project Brief*.

4. **Create node:** *Assign Tasks to Teams*  
   - Type: Execute Workflow (sub-workflow call)  
   - Workflow ID: Select current workflow (self-call)  
   - Workflow Inputs: Map `process`, `data`, set `status` to `"initialise"`, pass `main_execution_id` and `resume_url` from execution context.  
   - Options: Ensure `Wait for Sub-Workflow` is **OFF** (to fan out tasks asynchronously).  
   - Connect from *Split Out Tasks*.

5. **Create node:** *Wait for All Teams to Finish*  
   - Type: Wait (Webhook)  
   - Configure to pause until POST request to its webhook URL resumes execution.  
   - Connect from *Assign Tasks to Teams*.

6. **Create node:** *Project Complete!*  
   - Type: Set  
   - Assign fields for final aggregated outputs:  
     - `email`: `{{$json.body.write_email.result.output}}`  
     - `ad_copy`: `{{$json.body.write_ad_copy.result.output}}`  
     - `description`: `{{$json.body.write_description.result.output}}`  
     - `seconds_saved`: sum of all tasks’ duration minus max duration  
     - `task_duration_seconds`: max duration  
     - `task_duration_without_parallelisation`: sum of durations  
   - Connect from *Wait for All Teams to Finish*.

7. **Create node:** *Receive Work Order*  
   - Type: Execute Workflow Trigger  
   - Define expected inputs: `main_execution_id`, `process`, `data` (object), `status`, `resume_url`.

8. **Create node:** *Check Work Order Status*  
   - Type: If  
   - Condition: `$json.status === "pending"`  
   - Connect from *Receive Work Order*.

9. **Create node:** *The Dispatcher*  
   - Type: Switch  
   - Add rules based on `$json.process` equals:  
     - `"write_description"` → outputKey `write_description`  
     - `"write_ad_copy"` → outputKey `write_ad_copy`  
     - `"write_email"` → outputKey `write_email`  
   - Connect from *Check Work Order Status* on true branch.

10. **Create three LangChain Agent nodes:**  
    - *Description Team*  
      - Prompt: `"Write a product description for: {{$json.data.product}}"`  
      - System message: `"No preamble. Just what you're asked for."`  
      - Temperature: 0  
      - Connect from *The Dispatcher* (write_description output).  
    - *Ad Copy Team*  
      - Prompt: `"Write ad copy for: {{$json.data.product}}. The tone should be: {{$json.data.tone}}"`  
      - Same system message and temperature as above.  
      - Connect from *The Dispatcher* (write_ad_copy output).  
    - *Email Team*  
      - Prompt: `"Write a cold email about {{$json.data.product}} for the following audience: {{$json.data.audience}}"`  
      - Same system message and temperature as above.  
      - Connect from *The Dispatcher* (write_email output).

11. **Create node:** *Report Back to Manager*  
    - Type: Execute Workflow (sub-workflow call)  
    - Workflow ID: Current workflow (self-call)  
    - Workflow Inputs: Pass updated `data` with AI results, `status` set to `"completed"`, `process`, and `main_execution_id`.  
    - Connect from each Specialist Team node's output.

12. **Create node:** *The Project Dashboard (Code)*  
    - Type: Code  
    - Use the provided JavaScript code to:  
      - On `status: 'initialise'`, create a project entry with task statuses pending and start times.  
      - On `status: 'completed'`, update the task status to completed with end time and duration, check if all tasks complete, and if so return data that triggers resumption.  
    - Connect from *Check Work Order Status* false branch and *Report Back to Manager* (implicitly via sub-workflow).

13. **Create node:** *Is Project Complete?*  
    - Type: If  
    - Condition: `$json.end_process === true`  
    - Connect from *The Project Dashboard (Code)*.

14. **Create node:** *Resume Parent Workflow*  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `$json.resume_url`  
    - Body: JSON `$json.response`  
    - Connect from *Is Project Complete?* true branch.

15. **Create node:** *Start Process*  
    - Type: Execute Workflow (sub-workflow call)  
    - Inputs: Pass current task, `status: 'pending'`, etc.  
    - Connect from *Is Project Complete?* false branch.

**Credentials Required:**  
- Google Palm API credentials for LangChain AI nodes.  
- No special credentials for HTTP Request (unless webhook requires auth).

**Sub-Workflow Setup:**  
- The workflow calls itself with different input parameters to simulate sub-workflow tasks.  
- Ensure that the workflow supports recursive calling with appropriate input/output mappings.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is copyrighted and authored by Lucas Peyrin © 2025.                                                                                                                                                                                                                                                                                                                                          | Author and copyright notice inside sticky notes.                                                                                                                               |
| The workflow demonstrates a powerful parallel processing pattern (Fan-Out/Fan-In) suitable for speed-critical projects with independent tasks.                                                                                                                                                                                                                                                            | Concept explanation in Sticky Note4.                                                                                                                                             |
| The Project Dashboard node uses Static Data to maintain global project state, acting as a shared “whiteboard” for all parallel tasks.                                                                                                                                                                                                                                                                      | Sticky Note14 explains the importance and logic of the Project Dashboard code node.                                                                                              |
| For complex projects, the architecture can be modularized by splitting teams into separate dedicated workflows and using a status handler workflow to track progress.                                                                                                                                                                                                                                      | Sticky Note16 suggests alternative architecture for scalability.                                                                                                                |
| The Wait node uses a webhook that must be called via HTTP POST by the Resume Parent Workflow node to resume the main workflow. This pattern enables asynchronous parallel processing without blocking.                                                                                                                                                                                                    | Sticky Note7 explains the wait node’s role.                                                                                                                                      |
| Feedback and coaching resources are provided by the author, including links to n8n Academy and feedback forms.                                                                                                                                                                                                                                                                                             | Sticky Note10 contains links for feedback, coaching, and consulting:  
[Click here to give feedback](https://api.ia2s.app/form/templates/feedback?template=Parallel%20Processing%20Tutorial)  
[Book a Coaching Session](https://api.ia2s.app/form/templates/coaching?template=Parallel%20Processing%20Tutorial)  
[Inquire About Consulting Services](https://api.ia2s.app/form/templates/consulting?template=Parallel%20Processing%20Tutorial) |

---

**Disclaimer:** The text analyzed derives exclusively from an automated workflow constructed with n8n, respecting all content policies without illegal, offensive, or protected elements. All data handled is legal and public.