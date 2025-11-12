🛠️ State Management System for Long-Running Workflows with Wait Nodes

https://n8nworkflows.xyz/workflows/----state-management-system-for-long-running-workflows-with-wait-nodes-6269


# 🛠️ State Management System for Long-Running Workflows with Wait Nodes

### 1. Workflow Overview

This workflow implements a **State Management System for Long-Running Workflows with Wait Nodes** using the "Teleport" pattern. It is designed to manage workflows that require pausing and resuming multiple times, maintaining state across asynchronous external events.

- **Target Use Cases:**
  - Multi-day user onboarding sequences.
  - Processes that require human approval before continuing.
  - Chatbots or conversational workflows maintaining state across messages.
  - Any long-running, multi-step business process requiring external triggers to resume.

- **Logical Blocks:**

  **1.1 Main Process (Top Section)**  
  This is the primary business logic that can pause at checkpoints (`Wait` nodes) and waits for external events to resume. It initiates the session and processes data before and after checkpoints.

  **1.2 Async Portal (Bottom Section)**  
  This acts as the state-management engine. It tracks sessions in persistent static data, determines whether a session is new or existing, and "teleports" data to paused workflows by resuming them via HTTP requests.

  **1.3 Control and Error Handling**  
  Includes routing based on session state, error handling for teleport failures, and options to reset or stop sessions.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Main Process (Top Section)

- **Overview:**  
  Represents the core workflow that executes business logic, pauses at checkpoints, and resumes upon external triggers. It starts manually (or via webhook), calls the Async Portal to register or resume sessions, processes data, and uses `Wait` nodes as checkpoints.

- **Nodes Involved:**  
  - `1. Start Main Workflow (Manual)`  
  - `2. Call Async Portal`  
  - `Response From Checkpoint`  
  - `3. Process Initial Data (Before Checkpoint 1)`  
  - `4. PAUSE at Checkpoint 1`  
  - `5a. Receive Resumed Data`  
  - `5b. Send Back Data to Portal`  
  - `6. PAUSE at Checkpoint 2`  
  - `7. Process Final Data (After Checkpoint 2)`

- **Node Details:**

  - **1. Start Main Workflow (Manual)**  
    - Type: Manual Trigger  
    - Role: Entry point to start the main workflow for testing or manual execution.  
    - Inputs: None  
    - Outputs: Connected to `2. Call Async Portal`  
    - Edge Cases: None specific. Manual starts only.

  - **2. Call Async Portal**  
    - Type: Execute Workflow (Sub-workflow call)  
    - Role: Calls the Async Portal (state manager) to register or resume sessions.  
    - Configuration:  
      - Wait for Sub-Workflow: ON (ensures synchronous response)  
      - Inputs: `session_id`, `resume_url`, `input_items` (all input data), `workflow_id`, `execution_id`, `stop_session_on_error`  
      - Workflow ID: Self-referential (calls the portal workflow within the same workflow)  
    - Outputs: Connected to `Response From Checkpoint`  
    - Edge Cases: Requires correct session data; failure to receive response will affect flow.

  - **Response From Checkpoint**  
    - Type: IF  
    - Role: Checks if data was received from the portal indicating resumption from a checkpoint.  
    - Condition: Checks existence of `teleport_data` in JSON.  
    - Outputs:  
      - True: Processes teleported data (`Split Out Teleported Items`)  
      - False: Continues initial processing (`3. Process Initial Data (Before Checkpoint 1)`)  
    - Edge Cases: Expression failure if `teleport_data` missing or malformed.

  - **3. Process Initial Data (Before Checkpoint 1)**  
    - Type: Set  
    - Role: Placeholder for initial business logic before first checkpoint.  
    - Configuration: No specific assignments (user-defined in practice).  
    - Outputs: To `4. PAUSE at Checkpoint 1`  
    - Edge Cases: None (placeholder).

  - **4. PAUSE at Checkpoint 1**  
    - Type: Wait  
    - Role: First checkpoint where workflow execution pauses and waits for external resume.  
    - Configuration:  
      - Resume Mode: `webhook`  
      - HTTP Method: POST  
      - Response Mode: `responseNode` (two-way communication)  
      - Generates a unique `resume_url` for resuming this wait.  
    - Outputs: To `5a. Receive Resumed Data`  
    - Edge Cases: Timeout or missing resume call would keep workflow paused indefinitely.

  - **5a. Receive Resumed Data**  
    - Type: Split Out  
    - Role: Extracts resumed data from the POST body received at the wait node's webhook.  
    - Configuration: Field to split out: `body`  
    - Outputs: To `5b. Send Back Data to Portal`  
    - Edge Cases: Malformed body payload could cause errors.

  - **5b. Send Back Data to Portal**  
    - Type: Respond to Webhook  
    - Role: Sends response back to the Async Portal after resuming checkpoint 1, enabling two-way data flow.  
    - Configuration: Responds with all incoming items under the key `teleport_data`.  
    - Outputs: To `6. PAUSE at Checkpoint 2`  
    - Edge Cases: Network or response errors may disrupt data flow.

  - **6. PAUSE at Checkpoint 2**  
    - Type: Wait  
    - Role: Second checkpoint representing a pause without response data back to the portal.  
    - Configuration:  
      - Resume Mode: `webhook`  
      - HTTP Method: POST  
      - Response Mode: `never` (one-way continuation)  
    - Outputs: To `7. Process Final Data (After Checkpoint 2)`  
    - Edge Cases: Same as checkpoint 1 but no response expected.

  - **7. Process Final Data (After Checkpoint 2)**  
    - Type: Set  
    - Role: Final business logic after all checkpoints are passed.  
    - Configuration: Placeholder for final actions (e.g., complete order, send notification).  
    - Outputs: None (end of main process)  
    - Edge Cases: None (user-defined logic).

---

#### 2.2 Async Portal (Bottom Section)

- **Overview:**  
  This block manages session state persistently using workflow static data. It checks whether a session is new or existing and either registers it or resumes the paused workflow via a HTTP POST teleport call.

- **Nodes Involved:**  
  - `A. Entry: Receive Session Info`  
  - `B. Check if Session is New or Existing`  
  - `C. Route Based on Session State`  
  - `D. TELEPORT: Resume Paused Workflow`  
  - `Wait Node Sent Response Items ?`  
  - `Send Items From Checkpoint`  
  - `Stop Session On Error ?`  
  - `E. Stop This Execution`  
  - `Reset Session`  
  - `Respond with Input Items`  
  - `F. Prepare Initial Data`  
  - `G. Return Data to Main Workflow`

- **Node Details:**

  - **A. Entry: Receive Session Info**  
    - Type: Execute Workflow Trigger  
    - Role: Entry trigger node receiving session data and commands from external calls or main process.  
    - Inputs: Expects parameters: `session_id`, `resume_url`, `input_items`, `workflow_id`, `execution_id`, `stop_session_on_error`  
    - Outputs: To `B. Check if Session is New or Existing`  
    - Edge Cases: Missing required parameters cause errors downstream.

  - **B. Check if Session is New or Existing**  
    - Type: Code  
    - Role: Core logic for session state management using `workflowStaticData` as persistent memory.  
    - Configuration:  
      - Checks if session exists in static data under `[workflowId][sessionId]`.  
      - If new, stores `resume_url`, `execution_id`, and timestamp.  
      - Outputs: `{ new: boolean, resume_url, main_execution }`  
    - Key Expressions: Uses `$getWorkflowStaticData('global')` for persistence.  
    - Outputs: To `C. Route Based on Session State`  
    - Edge Cases: Missing inputs (`session_id`, `workflow_id`, `execution_id`) throw errors.

  - **C. Route Based on Session State**  
    - Type: IF  
    - Role: Routes flow based on whether session is new or existing.  
    - Condition: Checks if `new` is NOT true (i.e., existing session).  
    - Outputs:  
      - True (existing session): to `D. TELEPORT: Resume Paused Workflow`  
      - False (new session): to `F. Prepare Initial Data`  
    - Edge Cases: Expression failures if `new` is undefined.

  - **D. TELEPORT: Resume Paused Workflow**  
    - Type: HTTP Request  
    - Role: Sends HTTP POST to the paused workflow's `resume_url` to wake it up and pass new data.  
    - Configuration:  
      - URL: from `$json.resume_url`  
      - Method: POST  
      - JSON Body: passes `input_items` from entry node (`A. Entry`)  
      - Error Handling: On error, continues to error output branch.  
    - Outputs: Two branches:  
      - Success: to `Wait Node Sent Response Items ?`  
      - Error: to `Stop Session On Error ?`  
    - Edge Cases: Network errors, invalid resume URL, or stopped executions cause errors.

  - **Wait Node Sent Response Items ?**  
    - Type: IF  
    - Role: Checks if the resumed wait node sent back response data (`message` field is present and not "Workflow was started").  
    - Outputs:  
      - True: to `Send Items From Checkpoint` (which is a NoOp node)  
      - False: to `E. Stop This Execution` (filter to end workflow)  
    - Edge Cases: Missing or unexpected `message` field.

  - **Send Items From Checkpoint**  
    - Type: NoOp  
    - Role: Placeholder node for processing returned items from checkpoint if needed.  
    - Outputs: None  
    - Edge Cases: None.

  - **Stop Session On Error ?**  
    - Type: IF  
    - Role: Decision node to determine whether to stop session on teleport error or reset it.  
    - Condition: Checks if `stop_session_on_error` is true from entry node data.  
    - Outputs:  
      - True: to `E. Stop This Execution` (ends session)  
      - False: to `Reset Session` (resets session state)  
    - Edge Cases: Missing `stop_session_on_error` defaults to false.

  - **E. Stop This Execution**  
    - Type: Filter  
    - Role: Stops the current execution flow by filtering all data (no output).  
    - Outputs: None  
    - Edge Cases: None.

  - **Reset Session**  
    - Type: Code  
    - Role: Resets static data for the session, allowing a new main process to start fresh.  
    - Configuration: Similar logic to `B. Check if Session is New or Existing` but forcibly resets the session data.  
    - Outputs: To `Respond with Input Items`  
    - Edge Cases: Missing inputs cause errors.

  - **Respond with Input Items**  
    - Type: NoOp  
    - Role: Passes initial input items back to the main workflow for new sessions.  
    - Outputs: To `F. Prepare Initial Data`  
    - Edge Cases: None.

  - **F. Prepare Initial Data**  
    - Type: Set  
    - Role: Prepares the initial data payload by extracting `input_items` from entry node.  
    - Outputs: To `G. Return Data to Main Workflow`  
    - Edge Cases: Missing `input_items` causes empty data.

  - **G. Return Data to Main Workflow**  
    - Type: Split Out  
    - Role: Splits the `input_items` array to send back to the main workflow as individual items.  
    - Outputs: None (end of portal process for new sessions)  
    - Edge Cases: Empty or malformed arrays may produce no output.

---

### 3. Summary Table

| Node Name                         | Node Type                | Functional Role                                       | Input Node(s)                   | Output Node(s)                    | Sticky Note                                                                                                                                            |
|----------------------------------|--------------------------|-----------------------------------------------------|--------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1. Start Main Workflow (Manual)  | Manual Trigger           | Entry point to start main process                    | None                           | 2. Call Async Portal             | © 2025 Lucas Peyrin                                                                                                                                     |
| 2. Call Async Portal              | Execute Workflow         | Calls Async Portal to register/resume session       | 1. Start Main Workflow (Manual)| Response From Checkpoint         | "### 2. Call Async Portal" - Main process interaction with portal, waits for response to continue or teleport.                                         |
| Response From Checkpoint          | IF                       | Routes based on presence of teleported data         | 2. Call Async Portal            | Split Out Teleported Items / 3. Process Initial Data | "### 2a. Handle Portal's Response" - Different paths for first run or teleported data.                                                                |
| Split Out Teleported Items        | Split Out                | Extract teleported data                              | Response From Checkpoint        | None                            | © 2025 Lucas Peyrin                                                                                                                                     |
| 3. Process Initial Data (Before Checkpoint 1) | Set                      | Placeholder for initial business logic              | Response From Checkpoint        | 4. PAUSE at Checkpoint 1        | "### 3. Process Initial Data" - placeholder for user logic before checkpoint.                                                                           |
| 4. PAUSE at Checkpoint 1          | Wait                     | First checkpoint pause with response mode enabled   | 3. Process Initial Data         | 5a. Receive Resumed Data         | "### 4. PAUSE at Checkpoint 1" - Wait node with `responseMode` set to `responseNode` for two-way communication.                                         |
| 5a. Receive Resumed Data           | Split Out                | Extracts resumed data from webhook body             | 4. PAUSE at Checkpoint 1        | 5b. Send Back Data to Portal     | "### 5. Resumption & Response" - Receives data from portal on resume.                                                                                   |
| 5b. Send Back Data to Portal       | Respond to Webhook       | Sends response data back to portal                   | 5a. Receive Resumed Data        | 6. PAUSE at Checkpoint 2         | "### 5b. Send Back Data to Portal" - Two-way communication support.                                                                                    |
| 6. PAUSE at Checkpoint 2           | Wait                     | Second checkpoint pause without response             | 5b. Send Back Data to Portal    | 7. Process Final Data            | "### 6. PAUSE at Checkpoint 2" - Wait node with `responseMode` set to `never` (one-way continuation).                                                   |
| 7. Process Final Data (After Checkpoint 2) | Set                      | Final business logic after all checkpoints           | 6. PAUSE at Checkpoint 2        | None                            | "### 7. Process Final Data" - Final steps after checkpoints passed.                                                                                    |
| A. Entry: Receive Session Info     | Execute Workflow Trigger | Entry point for portal, receives session info        | None                           | B. Check if Session is New or Existing | "### Portal Entry" - Single entry for all external events interacting with stateful sessions.                                                        |
| B. Check if Session is New or Existing | Code                     | Manages persistent session state in static data     | A. Entry                       | C. Route Based on Session State | "### Session State Manager (Code)" - Core logic using workflow static data to track sessions.                                                         |
| C. Route Based on Session State    | IF                       | Routes based on session new/existing state           | B. Check if Session is New or Existing | D. TELEPORT / F. Prepare Initial Data | "### Route: New vs. Existing Session" - Decides if session resumes or starts fresh.                                                                   |
| D. TELEPORT: Resume Paused Workflow | HTTP Request             | Sends POST to resume paused workflow at resume_url  | C. Route Based on Session State (existing) | Wait Node Sent Response Items ? / Stop Session On Error ? | "### TELEPORT: Resume Paused Process" - Wakes paused workflow via HTTP POST.                                                                          |
| Wait Node Sent Response Items ?    | IF                       | Checks if resumed wait node sent response items      | D. TELEPORT                    | Send Items From Checkpoint / E. Stop This Execution | "### Handle Teleport Error" - Decides next step based on response presence.                                                                            |
| Send Items From Checkpoint         | NoOp                     | Placeholder for processing returned items            | Wait Node Sent Response Items ? (true) | None                            | © 2025 Lucas Peyrin                                                                                                                                     |
| Stop Session On Error ?            | IF                       | Decides to stop or reset session on teleport error   | D. TELEPORT (error branch)     | E. Stop This Execution / Reset Session | "### Handle Teleport Error" - Safety feature for teleport failure handling.                                                                           |
| E. Stop This Execution             | Filter                   | Stops execution flow                                  | Stop Session On Error ? (true) | None                            | © 2025 Lucas Peyrin                                                                                                                                     |
| Reset Session                     | Code                     | Resets session state in static data                   | Stop Session On Error ? (false) | Respond with Input Items         | © 2025 Lucas Peyrin                                                                                                                                     |
| Respond with Input Items           | NoOp                     | Passes initial input items back for new sessions     | Reset Session                  | F. Prepare Initial Data          | © 2025 Lucas Peyrin                                                                                                                                     |
| F. Prepare Initial Data            | Set                      | Prepares input_items for output                        | Respond with Input Items        | G. Return Data to Main Workflow | © 2025 Lucas Peyrin                                                                                                                                     |
| G. Return Data to Main Workflow    | Split Out                | Splits input_items array to individual items          | F. Prepare Initial Data         | None                            | © 2025 Lucas Peyrin                                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node: `1. Start Main Workflow (Manual)`**  
   - Type: Manual Trigger  
   - No parameters needed.  
   - Position: Beginning of main process.

2. **Create Node: `2. Call Async Portal`**  
   - Type: Execute Workflow (sub-workflow call)  
   - Configuration:  
     - Workflow ID: Set to the current workflow (self-call) or Async Portal workflow ID.  
     - Wait for Sub-Workflow: Enabled (true)  
     - Parameters to send:  
       - `session_id`: string, e.g., "test123" for testing or from input data  
       - `resume_url`: dynamic from current execution resume URL if available  
       - `input_items`: all current input JSON items  
       - `workflow_id`: current workflow ID  
       - `execution_id`: current execution ID  
       - `stop_session_on_error`: boolean, false by default  
   - Connect output of Manual Trigger to this node.

3. **Create Node: `Response From Checkpoint`**  
   - Type: IF  
   - Condition: Check if `teleport_data` exists in incoming JSON.  
   - True path: Connect to `Split Out Teleported Items`  
   - False path: Connect to `3. Process Initial Data (Before Checkpoint 1)`

4. **Create Node: `Split Out Teleported Items`**  
   - Type: Split Out  
   - Field to split: `teleport_data`  
   - Connect from `Response From Checkpoint` true path.

5. **Create Node: `3. Process Initial Data (Before Checkpoint 1)`**  
   - Type: Set  
   - Placeholder for business logic, no assignments by default.  
   - Connect from `Response From Checkpoint` false path.

6. **Create Node: `4. PAUSE at Checkpoint 1`**  
   - Type: Wait  
   - Configuration:  
     - Resume: webhook  
     - HTTP Method: POST  
     - Response Mode: `responseNode` (enables two-way communication)  
   - Connect from `3. Process Initial Data`.

7. **Create Node: `5a. Receive Resumed Data`**  
   - Type: Split Out  
   - Field to split: `body` (from webhook payload)  
   - Connect from `4. PAUSE at Checkpoint 1`.

8. **Create Node: `5b. Send Back Data to Portal`**  
   - Type: Respond to Webhook  
   - Respond with: all incoming items  
   - Response key: `teleport_data`  
   - Connect from `5a. Receive Resumed Data`.

9. **Create Node: `6. PAUSE at Checkpoint 2`**  
   - Type: Wait  
   - Configuration:  
     - Resume: webhook  
     - HTTP Method: POST  
     - Response Mode: `never` (one-way continuation)  
   - Connect from `5b. Send Back Data to Portal`.

10. **Create Node: `7. Process Final Data (After Checkpoint 2)`**  
    - Type: Set  
    - Placeholder for final business logic.  
    - Connect from `6. PAUSE at Checkpoint 2`.

---

11. **Create Async Portal Nodes:**

- **`A. Entry: Receive Session Info`**  
  - Type: Execute Workflow Trigger  
  - Parameters: Define inputs for `session_id`, `resume_url`, `input_items`, `workflow_id`, `execution_id`, `stop_session_on_error`.  
  - This is the portal entry point.

- **`B. Check if Session is New or Existing`**  
  - Type: Code  
  - Code logic as described: uses `$getWorkflowStaticData('global')` to check and store sessions.  
  - Validates required fields and sets output `{ new, resume_url, main_execution }`.

- **`C. Route Based on Session State`**  
  - Type: IF  
  - Condition: `{{$json.new}}` not equal to true (existing session).  
  - True path: to `D. TELEPORT`  
  - False path: to `F. Prepare Initial Data`.

- **`D. TELEPORT: Resume Paused Workflow`**  
  - Type: HTTP Request  
  - Method: POST  
  - URL: `{{$json.resume_url}}`  
  - Body: JSON, passing `input_items` from entry node.  
  - Error handling: Continue on error (do not fail execution).

- **`Wait Node Sent Response Items ?`**  
  - Type: IF  
  - Condition: Checks if `message` field in resumed response is not "Workflow was started" and exists.  
  - True path: `Send Items From Checkpoint`  
  - False path: `E. Stop This Execution`.

- **`Send Items From Checkpoint`**  
  - Type: NoOp  
  - Placeholder for processing returned data.

- **`Stop Session On Error ?`**  
  - Type: IF  
  - Condition: `stop_session_on_error` equals true.  
  - True path: `E. Stop This Execution`  
  - False path: `Reset Session`.

- **`E. Stop This Execution`**  
  - Type: Filter  
  - Filters out all data to stop workflow.

- **`Reset Session`**  
  - Type: Code  
  - Resets session data in static storage for given `session_id` and `workflow_id`.  
  - Outputs initial input items.

- **`Respond with Input Items`**  
  - Type: NoOp  
  - Passes data for processing.

- **`F. Prepare Initial Data`**  
  - Type: Set  
  - Assigns `input_items` from entry node to output.

- **`G. Return Data to Main Workflow`**  
  - Type: Split Out  
  - Splits `input_items` array into individual items.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                                |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| # The "Teleport" Workflow Pattern This workflow demonstrates a powerful stateful pattern for managing long-running, multi-step processes that need to be paused and resumed by external events. Use Cases: - Multi-day user onboarding sequences. - Processes waiting for human approval (e.g., via email link). - Chatbots that need to maintain conversation state across multiple messages. It consists of two parts: 1. The Main Process (Top): Your primary business logic. It can be paused at any 'Checkpoint' (`Wait` node). 2. The Async Portal (Bottom): A state-management engine that remembers running processes and can 'teleport' new data to them. | Sticky Note6                                                                                                                  |
| ## MAIN PROCESS This is your primary business logic. It calls the Portal to register itself, processes some data, and then pauses at a Checkpoint, waiting for another event to resume it.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note1                                                                                                                  |
| ## ASYNC PORTAL This is the state-management engine. It's a separate, reusable workflow. Its only job is to check if a session is new or existing. If it's existing, it finds the correct paused workflow's resume url and sends the new data to it. If it's new, then it simply returns all the input items.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note2                                                                                                                  |
| ### Portal Entry This is the single entry point for the entire state management system. Any external event (a webhook, a form, another workflow) that needs to interact with a stateful session will call this trigger.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note10                                                                                                                 |
| ### Session State Manager (Code) This Code node is the brain of the portal. It uses **Workflow Static Data** (`$getWorkflowStaticData`) as a persistent, shared memory (a "whiteboard") to track all active sessions. Technical Breakdown: 1. It receives a `session_id` and a `workflow_id`. 2. It checks its whiteboard to see if an entry for `[workflow_id][session_id]` already exists. 3. If it exists (Existing Session): It retrieves the stored `resume_url` and outputs `new: false`. 4. If it's new (New Session): It stores the incoming `resume_url` and `execution_id` on the whiteboard and outputs `new: true`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note3                                                                                                                  |
| ### Route: New vs. Existing Session This `IF` node directs traffic based on the output of the State Manager. - True (Existing Session): The flow proceeds to the 'Teleport' node to resume the paused process. - False (New Session): The flow proceeds to the 'Pass-through' nodes, which simply return the initial data back to the Main Process so it can start.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Sticky Note11                                                                                                                 |
| ### TELEPORT: Resume Paused Process This is the "Teleport" action. It's triggered only for existing sessions. Technical Breakdown: It takes the `resume_url` retrieved by the State Manager and makes an HTTP POST request to it. This action wakes up the specific `Wait` node that is paused in the Main Process, effectively delivering the new data directly to the correct checkpoint.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Sticky Note14                                                                                                                 |
| ### Handle Teleport Error This is a crucial safety feature. If the 'Teleport' action fails (e.g., the main execution was manually stopped and the `resume_url` is invalid), this branch is triggered. It allows you to decide whether to stop the session entirely or to reset the state, allowing a new Main Process to begin for that `session_id`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note12                                                                                                                 |
| ### 1. Start Main Process This is the entry point for your stateful process. It could be a webhook, a form submission, or a manual trigger like this one.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note5                                                                                                                  |
| ### 4. PAUSE at Checkpoint 1 This `Wait` node is a **Checkpoint**. The workflow execution physically stops here and is persisted. It generates a unique `resume_url` that is stored by the Portal. The only way to wake this execution up is for the Portal to make an HTTP POST request to that specific URL.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note7                                                                                                                  |
| ### 5. Resumption & Response When the Portal 'teleports' data, this sequence is triggered. - **5a:** Receives the new data from the body of the resume request. - **5b:** Immediately sends a response back to the Portal. This is optional and can be used for passing data back to the Portal if needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note8                                                                                                                  |
| ### 6. PAUSE at Checkpoint 2 This demonstrates that a single Main Process can have multiple, sequential checkpoints. The Portal will always resume the *latest* Checkpoint that was registered for the session.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Sticky Note9                                                                                                                  |
| ### Two Types of Checkpoints This workflow demonstrates two different ways to use `Wait` nodes as checkpoints. - **Checkpoint 1 (Two-Way):** This `Wait` node has **`Response Mode`** set to `On Resume`. This means it can not only be resumed, but it can also **send data back** to the Portal that resumed it. This is useful for two-way communication. - **Checkpoint 2 (One-Way):** This `Wait` node has **`Response Mode`** set to `Never`. It can only be resumed; it **cannot send data back**. This is useful for simple, one-way continuation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note18                                                                                                                 |
| ## How to Test This Workflow This pattern requires a special testing method to see it in action. 1. Open Two Tabs: Open this workflow in two separate browser tabs (Tab A and Tab B). 2. Start First Session (Tab A): Execute `1. Start Main Workflow`. It pauses at checkpoint 1. 3. Trigger Teleport (Tab B): Execute `1. Start Main Workflow` again. Portal detects existing session and teleports data, resuming Tab A. 4. Resume Final Checkpoint (Tab B): Execute `1. Start Main Workflow` once more to resume checkpoint 2 in Tab A and complete workflow. This proves external events from one execution can resume and pass data to paused executions. | Sticky Note (How to Test)                                                                                                      |
| ## Was this helpful? Let me know! [![clic](https://supastudio.ia2s.app/storage/v1/object/public/assets/n8n/clic_down_lucas.gif)](https://n8n.ac) I really hope this utility helped you implement asynchronous workflows in n8n easily. Your feedback is incredibly valuable and helps me create better resources for the n8n community. Share your thoughts & ideas: [Click here to give feedback](https://api.ia2s.app/form/templates/feedback?template=Async%20Portal) Coaching & Consulting offers are also linked for advanced help. Happy Automating! Lucas Peyrin | [n8n Academy](https://n8n.ac) | Sticky Note20                                                                                                                 |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.