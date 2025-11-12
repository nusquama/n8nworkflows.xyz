Prevent Duplicate Processing with Redis Item State Tracking

https://n8nworkflows.xyz/workflows/prevent-duplicate-processing-with-redis-item-state-tracking-6828


# Prevent Duplicate Processing with Redis Item State Tracking

### 1. Workflow Overview

This workflow, titled **"Prevent Duplicate Processing with Redis Item State Tracking"**, is designed as a centralized utility to manage and track the processing state of items in distributed workflows using Redis as a state store. Its main purpose is to prevent duplicate processing of the same item by multiple workflow instances through idempotency enforcement. It receives commands from other workflows to check if an item is currently processed, mark it as in-progress, update its final processing status, or delete its tracking key.

The workflow is logically organized into the following functional blocks:

- **1.1 Input and Context Preparation**: Receives input parameters from other workflows and constructs a consistent context including Redis keys and TTL values.
- **1.2 Action Routing and Redis Operations**: Uses a Switch node to route the request based on the action (`check`, `add`, `delete`, `update`) and performs the corresponding Redis operation (GET, SET, DELETE).
- **1.3 Result Formatting and Audit Logging**: Formats the output for each action and prepares a detailed audit log entry which is published to a Redis channel.
- **1.4 Error Handling**: Stops execution with an error if an invalid action is provided.
- **1.5 Audit Log Listener & Recorder (Separate Workflow Section)**: Independently listens to Redis Pub/Sub audit logs and records them into a Google Sheet for historical tracking and monitoring.

---

### 2. Block-by-Block Analysis

#### 2.1 Input and Context Preparation

- **Overview:**  
  This block receives input parameters from external workflows and constructs a consistent context object, including the Redis key format and TTL. It standardizes inputs for later use.

- **Nodes Involved:**  
  - `Trigger (from other workflows)`  
  - `Set: Context for Response`

- **Node Details:**

  - **Trigger (from other workflows)**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point accepting parameters from calling workflows.  
    - Config: Receives inputs such as `keyPrefix`, `itemId`, `action`, `TTL`, `parentExecution`, `keySuffix`, `time`, `updateValue`.  
    - Connections: Output to `Set: Context for Response`.  
    - Potential Failures: Missing or malformed inputs can cause downstream errors.

  - **Set: Context for Response**  
    - Type: Set Node  
    - Role: Constructs the full Redis key (`fullKey`) by concatenating `keyPrefix`, `itemId`, and `keySuffix`. Sets defaults for TTL and time if not provided.  
    - Key expressions:  
      - `fullKey = {{ $json.keyPrefix }}:{{ $json.itemId }}:{{ $json.keySuffix }}`  
      - `TTL = {{ $json.TTL || 1250000 }}` (defaults to ~20 minutes if TTL is missing)  
    - Connections: Output to `Switch Action`.  
    - Edge Cases: Incorrect `keyPrefix`, `itemId` or missing suffix might cause wrong Redis keys or unexpected behavior.

#### 2.2 Action Routing and Redis Operations

- **Overview:**  
  Routes the workflow based on the requested `action` input. Executes corresponding Redis operations to check, add, delete, or update item state.

- **Nodes Involved:**  
  - `Switch Action`  
  - `Redis: GET (Check Processed)`  
  - `Redis: SET (Add to Processed)`  
  - `Redis: DELETE (Untrack Item)`  
  - `Redis: SET (Update Result)2`  
  - `Handle Invalid Action`

- **Node Details:**

  - **Switch Action**  
    - Type: Switch Node  
    - Role: Routes execution paths based on the `action` attribute in input JSON.  
    - Conditions: Matches `check`, `add`, `delete`, `update` exactly (case-sensitive).  
    - Outputs: Each action routes to a Redis operation node; invalid actions route to error handler.  
    - Connections: Outputs to Redis nodes or error node.  
    - Failures: If input action is misspelled or unsupported, triggers error node.

  - **Redis: GET (Check Processed)**  
    - Type: Redis Node (get operation)  
    - Role: Retrieves value of the key to check if the item is currently tracked.  
    - Key: Uses `fullKey` from context.  
    - Output Property: `retrievedValue`  
    - Connections: Output to `Set CHECK Output`.  
    - Failures: Redis connection/auth errors, key not found returns null (handled downstream).

  - **Redis: SET (Add to Processed)**  
    - Type: Redis Node (set operation)  
    - Role: Adds a new Redis key marking the item as "in_progress".  
    - Key: `fullKey`  
    - Value: JSON string including `status: 'in_progress'`, current timestamp, and `parentExecution` context.  
    - TTL: From context, applies expiration.  
    - Connections: Output to `Set ADD Output`.  
    - Failures: Redis write errors, JSON serialization issues.

  - **Redis: DELETE (Untrack Item)**  
    - Type: Redis Node (delete operation)  
    - Role: Deletes the Redis key to untrack the item.  
    - Key: `fullKey`  
    - Connections: Output to `Set DELETE Output`.  
    - Failures: Redis connection errors, key not existing is not an error.

  - **Redis: SET (Update Result)2**  
    - Type: Redis Node (set operation)  
    - Role: Updates the Redis key with a new value representing final processing status.  
    - Key: `fullKey`  
    - Value: Serialized `updateValue` JSON from input.  
    - TTL: From context.  
    - Connections: Output to `Set UPDATE Output2`.  
    - Failures: Redis write/auth issues, invalid updateValue format.

  - **Handle Invalid Action**  
    - Type: Stop and Error Node  
    - Role: Stops workflow execution and returns an error message for invalid actions.  
    - Configuration: Fixed error message `"Invalid 'action' provided. Expected 'add' or 'check'."`  
    - Connections: Terminal (no outputs).  
    - Failures: Runs intentionally on invalid input.

#### 2.3 Result Formatting and Audit Logging

- **Overview:**  
  Formats the output JSON for each Redis operation result, constructs a detailed audit log message including execution context and status, and publishes the log to a Redis Pub/Sub channel.

- **Nodes Involved:**  
  - `Set CHECK Output`  
  - `Set ADD Output`  
  - `Set DELETE Output`  
  - `Set UPDATE Output2`  
  - `Code: Prepare Audit Log & Response`  
  - `To_redis-audit-log`

- **Node Details:**

  - **Set CHECK Output**  
    - Type: Set Node  
    - Role: Formats output for `check` action, setting properties:  
      - `success = true`  
      - `isMember = "true" if Redis value is not null/empty else "false"`  
      - `itemId` from switch input  
    - Connections: Output to `Code: Prepare Audit Log & Response`.  
    - Edge Cases: Redis value could be null or unexpected format.

  - **Set ADD Output**  
    - Type: Set Node  
    - Role: Formats output confirming successful addition with `success = true`.  
    - Connections: Output to `Code: Prepare Audit Log & Response`.

  - **Set DELETE Output**  
    - Type: Set Node  
    - Role: Confirms successful deletion with `success = true`.  
    - Connections: Output to `Code: Prepare Audit Log & Response`.

  - **Set UPDATE Output2**  
    - Type: Set Node  
    - Role: Formats output for update action including:  
      - `success = true`  
      - `status` from `updateValue.status`  
      - `errorMessage` and `errorDetails` if any from `updateValue`  
    - Connections: Output to `Code: Prepare Audit Log & Response`.

  - **Code: Prepare Audit Log & Response**  
    - Type: Code Node (JavaScript)  
    - Role:  
      - Reads the context and the result data.  
      - Constructs a detailed audit log object containing timestamp, utility name, action, keys, TTL, status, and parent execution context.  
      - Generates a direct URL to the parent workflow execution for troubleshooting.  
      - Sets output JSON including success flags, action performed, item ID, status, error info, and the audit log message as a JSON string.  
    - Connections: Output to `To_redis-audit-log`.  
    - Potential Failures: Errors in object access, missing parent execution info, or JSON serialization.

  - **To_redis-audit-log**  
    - Type: Redis Node (publish operation)  
    - Role: Publishes the audit log message to the Redis Pub/Sub channel `redis-audit-log`.  
    - Configuration: Uses the log message JSON string from code node output.  
    - Connections: Terminal.  
    - Failures: Redis connection issues, publish failures.

#### 2.4 Error Handling

- **Overview:**  
  Handles unexpected or unsupported actions by stopping workflow execution with an error.

- **Nodes Involved:**  
  - `Handle Invalid Action`

- **Node Details:**  
  Refer to 2.2 Error Handling.

#### 2.5 Audit Log Listener & Recorder (Separate Workflow Section)

- **Overview:**  
  Independently listens for Redis Pub/Sub messages on the `redis-audit-log` channel, parses incoming log messages, and appends them as rows to a Google Sheet for permanent audit trail and monitoring.

- **Nodes Involved:**  
  - `On new Redis event`  
  - `Parse Log Message`  
  - `redis-audit-log` (Google Sheets Append)

- **Node Details:**

  - **On new Redis event**  
    - Type: Redis Trigger Node  
    - Role: Listens continuously on `redis-audit-log` channel for new messages.  
    - Connections: Output to `Parse Log Message`.  
    - Failures: Redis connection loss or subscription issues.

  - **Parse Log Message**  
    - Type: Code Node  
    - Role: Parses the incoming Redis Pub/Sub string message into JSON object for further processing.  
    - Connections: Output to `redis-audit-log` Google Sheet node.  
    - Failures: Malformed JSON in message payload.

  - **redis-audit-log** (Google Sheets)  
    - Type: Google Sheets Node (append operation)  
    - Role: Appends parsed log data as new rows in a specified Google Sheet.  
    - Configuration: Maps audit log fields (timestamp, action, itemId, status, etc.) to sheet columns.  
    - Credentials: Requires configured Google Sheets OAuth2 credentials.  
    - Failures: Google API quota, authentication, or sheet permission errors.

---

### 3. Summary Table

| Node Name                     | Node Type                  | Functional Role                                    | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                                              |
|-------------------------------|----------------------------|---------------------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Trigger (from other workflows) | Execute Workflow Trigger   | Entry point receiving input from other workflows  | None                          | Set: Context for Response      |                                                                                                                                          |
| Set: Context for Response      | Set Node                  | Constructs Redis fullKey and default TTL          | Trigger (from other workflows) | Switch Action                 |                                                                                                                                          |
| Switch Action                 | Switch Node              | Routes workflow based on `action` input           | Set: Context for Response      | Redis: GET (Check Processed), Redis: SET (Add to Processed), Redis: DELETE (Untrack Item), Redis: SET (Update Result)2, Handle Invalid Action |                                                                                                                                          |
| Redis: GET (Check Processed)   | Redis Node (get)           | Retrieves Redis key value to check item state     | Switch Action (check)           | Set CHECK Output              |                                                                                                                                          |
| Set CHECK Output              | Set Node                  | Formats output for `check` action                  | Redis: GET (Check Processed)   | Code: Prepare Audit Log & Response |                                                                                                                                          |
| Redis: SET (Add to Processed)  | Redis Node (set)           | Adds Redis key marking item as in-progress        | Switch Action (add)             | Set ADD Output                |                                                                                                                                          |
| Set ADD Output                | Set Node                  | Formats output confirming successful add          | Redis: SET (Add to Processed)  | Code: Prepare Audit Log & Response |                                                                                                                                          |
| Redis: DELETE (Untrack Item)   | Redis Node (delete)        | Deletes Redis key to untrack item                  | Switch Action (delete)          | Set DELETE Output             |                                                                                                                                          |
| Set DELETE Output             | Set Node                  | Formats output confirming successful deletion     | Redis: DELETE (Untrack Item)   | Code: Prepare Audit Log & Response |                                                                                                                                          |
| Redis: SET (Update Result)2    | Redis Node (set)           | Updates Redis key with final status/result        | Switch Action (update)          | Set UPDATE Output2            |                                                                                                                                          |
| Set UPDATE Output2            | Set Node                  | Formats output for update action with status/error| Redis: SET (Update Result)2    | Code: Prepare Audit Log & Response |                                                                                                                                          |
| Handle Invalid Action         | Stop and Error Node       | Stops workflow for invalid action input            | Switch Action (invalid_action) | None                         |                                                                                                                                          |
| Code: Prepare Audit Log & Response | Code Node (JavaScript)     | Constructs audit log and final output JSON          | Set CHECK/ADD/DELETE/UPDATE Output Nodes | To_redis-audit-log         |                                                                                                                                          |
| To_redis-audit-log            | Redis Node (publish)       | Publishes audit log message to Redis Pub/Sub channel | Code: Prepare Audit Log & Response | None                         |                                                                                                                                          |
| On new Redis event            | Redis Trigger Node        | Listens for audit log messages on Redis channel   | None                          | Parse Log Message             | Audit Log Listener & Recorder: This background workflow listens to Redis channel `redis-audit-log` and records logs to Google Sheets.     |
| Parse Log Message             | Code Node (JavaScript)     | Parses Redis Pub/Sub message string to JSON        | On new Redis event             | redis-audit-log (Google Sheets) | Audit Log Listener & Recorder                                                                                                            |
| redis-audit-log               | Google Sheets Node         | Appends audit log data to Google Sheet             | Parse Log Message              | None                         | Audit Log Listener & Recorder                                                                                                            |
| Sticky Note                  | Sticky Note               | Documentation notes                                | None                          | None                         | No more messy loops with the Redis Item Tracker Utility. See detailed usage and key patterns in the note.                                |
| Sticky Note1                 | Sticky Note               | Documentation for Audit Log Listener & Recorder    | None                          | None                         | Explains the audit log listener’s purpose and requirements.                                                                              |
| Sticky Note2                 | Sticky Note               | Short note on audit log listener                    | None                          | None                         | Use this optional node to log workflow executions.                                                                                       |
| Sticky Note3                 | Sticky Note               | Summary of key concepts for Redis Item Tracker     | None                          | None                         | Provides quick reference for key parts of the Redis key and actions (`check`, `add`, `update`).                                          |

---

### 4. Reproducing the Workflow from Scratch

To recreate this workflow manually in n8n, follow these steps:

**4.1 Setup Credentials**  
- Configure Redis credentials in n8n with access to your Redis database.  
- Configure Google Sheets OAuth2 credentials with edit access to a target spreadsheet for logs.

**4.2 Create Nodes**

1. **Trigger (from other workflows)**  
   - Type: Execute Workflow Trigger  
   - Parameters: Define inputs: `keyPrefix` (string), `itemId` (string), `action` (string), `TTL` (number, optional), `parentExecution` (object, optional), `keySuffix` (string, optional), `time` (array, optional), `updateValue` (any, optional).  
   - No input connections.

2. **Set: Context for Response**  
   - Type: Set Node  
   - Settings:  
     - Assign `fullKey = {{ $json.keyPrefix }}:{{ $json.itemId }}:{{ $json.keySuffix }}`  
     - Assign `itemId = {{ $json.itemId }}`  
     - Assign `TTL = {{ $json.TTL || 1250000 }}` (default if TTL missing)  
     - Assign `time = {{ $json.time || null }}`  
   - Connect output of Trigger to this node.

3. **Switch Action**  
   - Type: Switch Node  
   - Rules: For `action` field, with strict equals and case-sensitive:  
     - Output 1: `check`  
     - Output 2: `add`  
     - Output 3: `delete`  
     - Output 4: `update`  
     - Fallback: `invalid_action`  
   - Connect output of Set Context to input of this node.

4. **Redis: GET (Check Processed)**  
   - Type: Redis Node  
   - Operation: `get`  
   - Key: `{{ $json.fullKey }}`  
   - Property Name: `retrievedValue`  
   - Connect Switch output `check` to this node.

5. **Set CHECK Output**  
   - Type: Set Node  
   - Assignments:  
     - `success = true` (boolean)  
     - `isMember = {{ $json.retrievedValue !== null && $json.retrievedValue !== '' }}` (string)  
     - `itemId = {{ $('Switch Action').item.json.itemId }}`  
   - Connect Redis GET output to this node.

6. **Redis: SET (Add to Processed)**  
   - Type: Redis Node  
   - Operation: `set`  
   - Key: `{{ $json.fullKey }}`  
   - TTL: `{{ $json.TTL }}`  
   - Value: JSON-stringified object `{ status: 'in_progress', startedAt: current ISO date, parentExecution: $json.parentExecution }`  
   - Expire: true  
   - Connect Switch output `add` to this node.

7. **Set ADD Output**  
   - Type: Set Node  
   - Assignments: `success = true` (boolean)  
   - Connect Redis SET (Add) output to this node.

8. **Redis: DELETE (Untrack Item)**  
   - Type: Redis Node  
   - Operation: `delete`  
   - Key: `{{ $json.fullKey }}`  
   - Connect Switch output `delete` to this node.

9. **Set DELETE Output**  
   - Type: Set Node  
   - Assignments: `success = true` (boolean)  
   - Connect Redis DELETE output to this node.

10. **Redis: SET (Update Result)2**  
    - Type: Redis Node  
    - Operation: `set`  
    - Key: `{{ $json.fullKey }}`  
    - TTL: `{{ $json.TTL }}`  
    - Value: JSON-stringified `{{ $json.updateValue }}`  
    - Expire: true  
    - Connect Switch output `update` to this node.

11. **Set UPDATE Output2**  
    - Type: Set Node  
    - Assignments:  
      - `success = true` (boolean)  
      - `status = {{ $json.updateValue.status }}`  
      - `errorMessage = {{ $json.updateValue.errorMessage || null }}`  
      - `errorDetails = {{ $json.updateValue.errorDetails || null }}`  
    - Connect Redis SET (Update) output to this node.

12. **Handle Invalid Action**  
    - Type: Stop and Error Node  
    - Error Message: `"Invalid 'action' provided. Expected 'add' or 'check'."`  
    - Connect Switch fallback output to this node.

13. **Code: Prepare Audit Log & Response**  
    - Type: Code Node (JavaScript)  
    - JavaScript: Implement logic to build a detailed audit log JSON object and output final response JSON including the audit log message string. (Refer to the logic in the original code node.)  
    - Connect all Set output nodes (CHECK, ADD, DELETE, UPDATE) to this node.

14. **To_redis-audit-log**  
    - Type: Redis Node  
    - Operation: `publish`  
    - Channel: `redis-audit-log`  
    - Message Data: `{{ $json.logMessage }}`  
    - Connect Code node output to this node.

**4.3 Setup Audit Log Listener & Recorder**

15. **On new Redis event**  
    - Type: Redis Trigger Node  
    - Channel: `redis-audit-log`  
    - Credentials: Same Redis credentials.

16. **Parse Log Message**  
    - Type: Code Node  
    - JavaScript: Parse the incoming Redis message string with `JSON.parse($json.message);`

17. **redis-audit-log** (Google Sheets)  
    - Type: Google Sheets Node  
    - Operation: Append  
    - Document ID: Set your audit log spreadsheet ID  
    - Sheet Name: Set appropriate sheet (e.g., "Sheet1")  
    - Mapping: Map fields from parsed JSON to sheet columns (timestamp, action, itemId, status, etc.)  
    - Credentials: Google Sheets OAuth2 credentials

18. Connect `On new Redis event` ➔ `Parse Log Message` ➔ `redis-audit-log (Google Sheets)`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| ## No more messy loops with the Redis Item Tracker Utility. This workflow serves as a robust, centralized utility to prevent duplicate processing of items and manage their state in Redis. Designed to be called by other workflows, it provides a reliable mechanism to check if an item has been handled, mark it as in-progress, or update its final status. It automatically generates a direct link to the parent workflow's execution for rapid troubleshooting. Detailed explanation and usage instructions are embedded as sticky notes inside the workflow.                                                                                                                                                                                                                                                       | Workflow main documentation sticky note.                                                                                  |
| ## Audit Log Listener & Recorder. This companion workflow acts as a central monitoring system for the Item Tracker Utility, creating a permanent, easy-to-read record of every single action performed by the utility. It listens to Redis `redis-audit-log` channel and writes logs to Google Sheets. Requires Redis connection and Google Sheets credentials configured in n8n.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Audit Log Listener sticky notes and node documentation.                                                                    |
| ### The Item Tracker Playbook: Key Concepts at a Glance. Provides quick reference for Redis key parts and the core actions: `check` (read), `add` (set temporary lock), `update` (final state). Explains key format `keyPrefix:itemId:keySuffix` and example usages.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Key concepts sticky note inside the workflow.                                                                               |
| This workflow requires a running n8n instance accessible to your workflows, a Redis database, and configured Redis credentials in n8n. The audit log recorder requires Google Sheets credentials and a prepared Google Sheet with columns matching the audit log schema. | Setup and deployment notes.                                                                                                |
| For detailed troubleshooting, the workflow generates a direct URL link to the parent execution in n8n.cloud or your n8n instance, included in the audit log and output to speed up diagnostics.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Troubleshooting URL generation logic in Code node.                                                                          |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.