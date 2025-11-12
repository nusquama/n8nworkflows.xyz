Create Secure Human-in-the-Loop Approval Flows with Postgres and Telegram

https://n8nworkflows.xyz/workflows/create-secure-human-in-the-loop-approval-flows-with-postgres-and-telegram-9039


# Create Secure Human-in-the-Loop Approval Flows with Postgres and Telegram

---

## 1. Workflow Overview

This workflow implements a **secure human-in-the-loop approval system** integrating n8n, PostgreSQL, and Telegram. It is designed for scenarios where automated processes require manager approval or rejection via secure, time-limited links sent through Telegram. The workflow ensures that each approval or rejection action is authenticated, logged, and notified, maintaining traceability and security.

**Target Use Cases:**  
- Ticket or request approval management where human validation is mandatory.  
- Workflow automation requiring secure, auditable decision points.  
- Real-time status updates and alerts to ticket owners via Telegram.

**Logical Blocks:**

- **1.1 Input Reception and Validation**: Accepts approval decision requests via webhook and verifies the authenticity and timeliness of the request using HMAC signatures and TTL checks.

- **1.2 Decision Routing and Processing**: Routes the request based on the approval action (approve, reject, expired, invalid) and performs database updates, audit logging, and notifications accordingly.

- **1.3 Database Operations**: Executes PostgreSQL queries to update ticket status, retrieve ticket information, and log audit entries and workflow errors.

- **1.4 Telegram Notifications**: Sends customized Telegram messages to ticket owners or managers based on the outcome (approval, rejection, expired link, invalid attempt).

- **1.5 Error Handling and Logging**: Captures and records any workflow errors into a dedicated database table for monitoring.

- **1.6 Setup and Documentation**: Includes sticky notes with setup instructions, security notes, and operational guidance.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Validation

**Overview:**  
This block receives HTTP approval requests containing query parameters and verifies their authenticity and validity using HMAC signatures and expiration timestamps.

**Nodes Involved:**  
- 01 Webhook Trigger: Approval Decision  
- 02 FN: Verify Signature + TTL  
- Actions (Switch node)

**Node Details:**

- **01 Webhook Trigger: Approval Decision**  
  - Type: Webhook Trigger  
  - Role: Entry point receiving approval decisions via HTTP GET requests.  
  - Configuration: Path `/approval`; responds with a simple confirmation message "✅ Thanks, your decision was recorded."  
  - Inputs: HTTP request with query parameters (`cid`, `status`, `action`, `exp`, `sig`).  
  - Outputs: Passes received data to the verification node.  
  - Failure cases: Missing or malformed query parameters might cause downstream failures.

- **02 FN: Verify Signature + TTL**  
  - Type: Code (Python)  
  - Role: Validates the HMAC signature of the request and checks if the link is expired based on TTL.  
  - Configuration:  
    - Uses environment variable `SECRET_KEY` as HMAC secret.  
    - Extracts `cid` (correlation ID), `status`, `action`, `exp` (expiry timestamp), and `sig` (signature) from query.  
    - Recomputes HMAC-SHA256 signature on payload string `cid|status|exp`.  
    - Verifies signature correctness and expiry.  
    - Determines action (`approve`, `reject`, `expired`, `invalid`, or `unknown`) and sets actor as `"manager"` for valid approvals/rejections.  
  - Expressions: Uses Python environment variables and item JSON query parsing.  
  - Inputs: Output from webhook node.  
  - Outputs: JSON with keys: `correlation_id`, `new_status`, `reason`, `action`, and `actor` where applicable.  
  - Failure cases: Missing `SECRET_KEY` environment variable; invalid or missing query parameters; expired links; unsupported actions.  
  - Version: Python code node requires Python environment in n8n.

- **Actions (Switch node)**  
  - Type: Switch  
  - Role: Routes flow based on the `action` field (`approve`, `reject`, `expired`, `invalid`).  
  - Configuration: Four output branches named accordingly.  
  - Inputs: Output from verification node.  
  - Outputs: Each branch triggers specific handling logic.  
  - Failure cases: Unexpected action values not matching predefined cases.

---

### 2.2 Decision Routing and Processing

**Overview:**  
Processes the verification outcome by updating ticket status, inserting audit records, and triggering notifications based on approval, rejection, expiration, or invalid attempts.

**Nodes Involved:**  
- 04c DB: Update Ticket Status  
- 04c2 DB: Insert Audit Row  
- 04c1 DB: Get Ticket Owner  
- 04c1a IF: Resolved or In Progress  
- If Resolved  
- If in_progress  
- 05c Telegram: Update Confirmation  
- 04r0 DB: Get Ticket ID  
- Execute a SQL query1 (Insert Reject Audit)  
- Text (reject)  
- 04e DB: Insert Audit Expired  
- 05e Telegram: Notify Expired  
- 04i DB: Insert Audit Invalid  
- 05i Telegram: Alert Invalid

**Node Details:**

- **04c DB: Update Ticket Status**  
  - Type: PostgreSQL node  
  - Role: Updates ticket status in the `tickets` table for the given `correlation_id`.  
  - Config: SQL `UPDATE tickets SET status = $2, updated_at = NOW() WHERE correlation_id = $1::uuid RETURNING id, status, updated_at, correlation_id;`  
  - Inputs: From Switch node on `approve` branch.  
  - Outputs: Returns updated ticket info for audit and notification steps.  
  - Failure cases: DB connection errors, invalid UUID, no matching ticket.

- **04c2 DB: Insert Audit Row**  
  - Type: PostgreSQL node  
  - Role: Inserts an audit entry recording the update action by the manager.  
  - Config: Inserts into `ticket_audit` table with ticket ID, correlation ID, action `'update'`, new status, and actor chat ID.  
  - Inputs: From Update Ticket Status node.  
  - Outputs: Passes data to next step for owner retrieval.  
  - Failure cases: DB write errors.

- **04c1 DB: Get Ticket Owner**  
  - Type: PostgreSQL node  
  - Role: Fetches ticket owner info (`chat_id`, etc.) for notifications.  
  - Config: SQL `SELECT chat_id, correlation_id, status, subject FROM tickets WHERE correlation_id = $1::uuid;`  
  - Inputs: From audit insertion step.  
  - Outputs: Provides chat ID for Telegram notifications.  
  - Failure cases: Missing ticket record.

- **04c1a IF: Resolved or In Progress**  
  - Type: If node  
  - Role: Checks if ticket status is `resolved` or `in_progress` to branch notifications.  
  - Inputs: From Get Ticket Owner.  
  - Outputs: Branches to respective notification nodes.

- **If Resolved**  
  - Type: If node  
  - Role: Checks specifically for status `resolved`. Passes to resolved notification.  
  - Inputs: From 04c1a IF node.  
  - Outputs: Triggers resolved notification branch.

- **If in_progress**  
  - Type: If node  
  - Role: Checks specifically for status `in_progress`. Passes to in-progress notification.  
  - Inputs: From 04c1a IF node.  
  - Outputs: Triggers in-progress notification branch.

- **05c Telegram: Update Confirmation**  
  - Type: Telegram node  
  - Role: Sends confirmation message to ticket owner about the update.  
  - Config: HTML formatted message with ticket ID, new status, and update time.  
  - Inputs: From Get Ticket Owner node for general confirmation.  
  - Failure cases: Invalid `chat_id`, Telegram API failures.

- **04r0 DB: Get Ticket ID**  
  - Type: PostgreSQL node  
  - Role: Retrieves ticket ID and correlation ID for reject audit insertion.  
  - Inputs: From Switch node on `reject` branch.  
  - Outputs: Provides data for rejection audit logging.

- **Execute a SQL query1 (Insert Reject Audit)**  
  - Type: PostgreSQL node  
  - Role: Inserts an audit record for rejection action.  
  - Inputs: From Get Ticket ID.  
  - Outputs: Passes to rejection notification.

- **Text (reject)**  
  - Type: Telegram node  
  - Role: Sends rejection notification to ticket owner.  
  - Config: HTML message indicating manager rejected update and ticket unchanged.  
  - Inputs: From rejection audit insertion.  
  - Failure cases: Telegram API failures, invalid chat IDs.

- **04e DB: Insert Audit Expired**  
  - Type: PostgreSQL node  
  - Role: Logs expired approval link action in audit table with actor 'system'.  
  - Inputs: From Switch node on `expired` branch.  
  - Outputs: Passes to expired notification.

- **05e Telegram: Notify Expired**  
  - Type: Telegram node  
  - Role: Notifies manager/admin about expired approval link attempt.  
  - Config: Message includes ticket correlation ID.  
  - Inputs: From expired audit insertion node.  
  - Failure cases: Telegram notification issues.

- **04i DB: Insert Audit Invalid**  
  - Type: PostgreSQL node  
  - Role: Logs invalid approval attempt in audit table with actor 'system'.  
  - Inputs: From Switch node on `invalid` branch.  
  - Outputs: Passes to invalid alert notification.

- **05i Telegram: Alert Invalid**  
  - Type: Telegram node  
  - Role: Sends alert notification to manager/admin about invalid approval attempt.  
  - Config: Includes ticket correlation ID in message.  
  - Inputs: From invalid audit insertion node.

---

### 2.3 Error Handling and Logging

**Overview:**  
This block detects any errors during notification sending or other nodes and logs them into a dedicated database table for operational monitoring.

**Nodes Involved:**  
- Notify Failed? (If node)  
- Execute a SQL query (Insert workflow error)

**Node Details:**

- **Notify Failed?**  
  - Type: If node  
  - Role: Checks if error property exists in the JSON payload (indicating failure in previous nodes).  
  - Inputs: From Telegram notification nodes (resolved, in-progress notifications).  
  - Outputs: If error detected, triggers workflow error logging.

- **Execute a SQL query (Insert workflow error)**  
  - Type: PostgreSQL node  
  - Role: Inserts error details into `workflow_errors` table with workflow metadata and error message.  
  - Inputs: From Notify Failed? node on true condition.  
  - Failure cases: DB connection issues.

---

### 2.4 Setup and Documentation (Sticky Notes)

**Overview:**  
Sticky notes provide detailed setup instructions, security notes, usage guidance, and project overview for users and administrators.

**Nodes Involved:**  
- Sticky Note (main overview)  
- Sticky Note1 (Security note)  
- Sticky Note2 (Telegram chat ID info)  
- Sticky Note3 (Approval link generation and verification explanation)  
- Sticky Note4 (Step-by-step beginner setup guide)

**Node Details:**

- **Sticky Note**  
  - Content: Presents workflow summary, features, requirements, and high-level setup steps.

- **Sticky Note1**  
  - Content: Emphasizes necessity of setting `SECRET_KEY` environment variable for security.

- **Sticky Note2**  
  - Content: Explains management of Telegram chat IDs for user and admin messages.

- **Sticky Note3**  
  - Content: Describes HMAC link generation and verification logic.

- **Sticky Note4**  
  - Content: Detailed stepwise instructions for environment setup, database table creation, credential configuration, workflow import, and testing.

---

## 3. Summary Table

| Node Name                           | Node Type         | Functional Role                                | Input Node(s)                       | Output Node(s)                            | Sticky Note                                                                                   |
|-----------------------------------|-------------------|-----------------------------------------------|-----------------------------------|------------------------------------------|-----------------------------------------------------------------------------------------------|
| 01 Webhook Trigger: Approval Decision | Webhook           | Receives approval decisions via HTTP requests | None                              | 02 FN: Verify Signature + TTL             |                                                                                               |
| 02 FN: Verify Signature + TTL      | Code (Python)     | Validates HMAC signature and TTL expiration   | 01 Webhook Trigger                 | Actions                                  | Security Note about SECRET_KEY env variable (Sticky Note1)                                    |
| Actions                          | Switch            | Routes flow based on action type               | 02 FN: Verify Signature + TTL      | 04c DB: Update Ticket Status, 04r0 DB: Get Ticket ID, 04e DB: Insert Audit Expired, 04i DB: Insert Audit Invalid | Approval link verification explanation (Sticky Note3)                                         |
| 04c DB: Update Ticket Status       | PostgreSQL        | Updates ticket status in DB                     | Actions (approve branch)           | 04c2 DB: Insert Audit Row                  | Requires tickets and audit tables (Sticky Note main overview)                                 |
| 04c2 DB: Insert Audit Row          | PostgreSQL        | Inserts audit log for update                    | 04c DB: Update Ticket Status       | 04c1 DB: Get Ticket Owner                 |                                                                                               |
| 04c1 DB: Get Ticket Owner          | PostgreSQL        | Retrieves ticket owner info                      | 04c2 DB: Insert Audit Row          | 04c1a IF: Resolved or In Progress, 05c Telegram: Update Confirmation |                                                                                               |
| 04c1a IF: Resolved or In Progress  | If                | Checks ticket status for notification routing  | 04c1 DB: Get Ticket Owner          | If Resolved, If in_progress                |                                                                                               |
| If Resolved                      | If                | Checks if status is 'resolved'                   | 04c1a IF: Resolved or In Progress  | 05c1a Telegram: Notify Resolved           |                                                                                               |
| If in_progress                   | If                | Checks if status is 'in_progress'                | 04c1a IF: Resolved or In Progress  | 05c1b Telegram: Notify In Progress        |                                                                                               |
| 05c Telegram: Update Confirmation  | Telegram          | Sends confirmation message to ticket owner      | 04c1 DB: Get Ticket Owner          | None                                     |                                                                                               |
| 05c1a Telegram: Notify Resolved    | Telegram          | Notifies user that ticket is resolved            | If Resolved                       | Notify Failed?                            | Telegram chat ID usage note (Sticky Note2)                                                    |
| 05c1b Telegram: Notify In Progress | Telegram          | Notifies user that ticket is in progress          | If in_progress                    | Notify Failed?                            | Telegram chat ID usage note (Sticky Note2)                                                    |
| Notify Failed?                   | If                | Detects notification sending errors              | 05c1a Telegram: Notify Resolved, 05c1b Telegram: Notify In Progress | Execute a SQL query (Insert workflow error) |                                                                                               |
| Execute a SQL query (Insert workflow error) | PostgreSQL        | Logs workflow errors into DB                      | Notify Failed?                    | None                                     |                                                                                               |
| 04r0 DB: Get Ticket ID             | PostgreSQL        | Gets ticket ID for rejection audit                | Actions (reject branch)           | Execute a SQL query1 (Insert Reject Audit) |                                                                                               |
| Execute a SQL query1 (Insert Reject Audit) | PostgreSQL        | Inserts rejection audit record                    | 04r0 DB: Get Ticket ID            | Text (reject)                            |                                                                                               |
| Text (reject)                   | Telegram          | Sends rejection notice to ticket owner            | Execute a SQL query1               | None                                     |                                                                                               |
| 04e DB: Insert Audit Expired       | PostgreSQL        | Logs expired approval link attempts                | Actions (expired branch)          | 05e Telegram: Notify Expired              |                                                                                               |
| 05e Telegram: Notify Expired       | Telegram          | Notifies admin/manager about expired approval     | 04e DB: Insert Audit Expired      | None                                     |                                                                                               |
| 04i DB: Insert Audit Invalid       | PostgreSQL        | Logs invalid approval attempts                      | Actions (invalid branch)          | 05i Telegram: Alert Invalid                |                                                                                               |
| 05i Telegram: Alert Invalid        | Telegram          | Alerts admin/manager about invalid approval attempts | 04i DB: Insert Audit Invalid      | None                                     |                                                                                               |
| Sticky Note                      | Sticky Note       | Workflow overview and feature summary              | None                            | None                                     | Main workflow overview and requirements                                                      |
| Sticky Note1                     | Sticky Note       | Security note on SECRET_KEY environment variable    | None                            | None                                     | Security environment variable note                                                           |
| Sticky Note2                     | Sticky Note       | Explanation of Telegram chat ID usage                | None                            | None                                     | Telegram chat ID usage explanation                                                           |
| Sticky Note3                     | Sticky Note       | Explanation of approval link generation and verification | None                            | None                                     | Approval signature and TTL verification explanation                                          |
| Sticky Note4                     | Sticky Note       | Step-by-step setup instructions                        | None                            | None                                     | Detailed beginner-friendly setup guide                                                       |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - Name: `01 Webhook Trigger: Approval Decision`  
   - Path: `/approval`  
   - Response: Static text `✅ Thanks, your decision was recorded.`  
   - Accepts query parameters: `cid`, `status`, `action`, `exp`, `sig`.

2. **Create Python Code Node for Verification**  
   - Type: Code (Python)  
   - Name: `02 FN: Verify Signature + TTL`  
   - Set environment variable `SECRET_KEY` in `.env`.  
   - Code logic:  
     - Extract query params from incoming webhook.  
     - Compute HMAC-SHA256 signature on `cid|status|exp` using `SECRET_KEY`.  
     - Compare with `sig`.  
     - Check if `exp` (epoch seconds) is still valid (not expired).  
     - Set output JSON with action: `approve`, `reject`, `expired`, `invalid` or `unknown`.  
     - Add actor info (`manager` for valid actions).  
   - Connect output of webhook node to this node.

3. **Create Switch Node for Action Routing**  
   - Type: Switch  
   - Name: `Actions`  
   - Define four output rules matching `action` field: `approve`, `reject`, `expired`, `invalid`.  
   - Connect output of verification node to this switch.

4. **Approve Branch Setup:**  
   - Create PostgreSQL node `04c DB: Update Ticket Status`  
     - Query: Update `tickets` table status by `correlation_id`. Return updated record fields.  
     - Use credentials for Postgres connection.  
     - Connect from `approve` output of Switch node.

   - Create PostgreSQL node `04c2 DB: Insert Audit Row`  
     - Insert a record into `ticket_audit` with ticket ID, correlation ID, action `'update'`, new_status, actor chat ID.  
     - Connect from Update Ticket Status node.

   - Create PostgreSQL node `04c1 DB: Get Ticket Owner`  
     - Select `chat_id`, `correlation_id`, `status`, `subject` from tickets for notification.  
     - Connect from Insert Audit Row node.

   - Create If node `04c1a IF: Resolved or In Progress`  
     - Condition: Check if status is `'resolved'` or `'in_progress'`.  
     - Connect from Get Ticket Owner node.

   - Create If node `If Resolved`  
     - Condition: status equals `'resolved'`.  
     - Connect from `Resolved` output of previous If node.

   - Create If node `If in_progress`  
     - Condition: status equals `'in_progress'`.  
     - Connect from `in_progress` output of previous If node.

   - Create Telegram node `05c1a Telegram: Notify Resolved`  
     - Send HTML message to `chat_id` indicating ticket resolved.  
     - Connect from `If Resolved` node.

   - Create Telegram node `05c1b Telegram: Notify In Progress`  
     - Send HTML message to `chat_id` indicating ticket in progress.  
     - Connect from `If in_progress` node.

   - Create Telegram node `05c Telegram: Update Confirmation`  
     - Send confirmation message with ticket update details.  
     - Connect from `04c1 DB: Get Ticket Owner` node (general confirmation branch).

   - Create If node `Notify Failed?`  
     - Check if JSON contains error property.  
     - Connect from both Telegram notification nodes (`05c1a` and `05c1b`).

   - Create PostgreSQL node `Execute a SQL query` (error logging)  
     - Insert errors into `workflow_errors` table with workflow and error details.  
     - Connect from `Notify Failed?` node true branch.

5. **Reject Branch Setup:**  
   - Create PostgreSQL node `04r0 DB: Get Ticket ID`  
     - Select ticket ID and correlation ID by correlation_id.  
     - Connect from `reject` output of Switch node.

   - Create PostgreSQL node `Execute a SQL query1` (Insert Reject Audit)  
     - Insert rejection audit entry.  
     - Connect from Get Ticket ID node.

   - Create Telegram node `Text (reject)`  
     - Send rejection notice to ticket owner.  
     - Connect from Insert Reject Audit node.

6. **Expired Branch Setup:**  
   - Create PostgreSQL node `04e DB: Insert Audit Expired`  
     - Insert audit record for expired action with actor 'system'.  
     - Connect from `expired` output of Switch node.

   - Create Telegram node `05e Telegram: Notify Expired`  
     - Notify manager about expired approval link.  
     - Connect from Insert Audit Expired node.

7. **Invalid Branch Setup:**  
   - Create PostgreSQL node `04i DB: Insert Audit Invalid`  
     - Insert audit record for invalid action with actor 'system'.  
     - Connect from `invalid` output of Switch node.

   - Create Telegram node `05i Telegram: Alert Invalid`  
     - Alert manager about invalid approval attempt.  
     - Connect from Insert Audit Invalid node.

8. **Configure Credentials:**  
   - Add PostgreSQL credentials with access to your Postgres DB.  
   - Add Telegram API credentials with your bot token.  
   - Ensure environment variable `SECRET_KEY` is set in your n8n environment.

9. **Create Required PostgreSQL Tables:**  
   - `tickets` table: stores tickets with correlation ID, status, subject, chat_id, update timestamp.  
   - `ticket_audit` table: logs all changes with ticket ID, action, new status, actor, timestamp.  
   - `workflow_errors` table: logs workflow errors with metadata and JSON payload.

10. **Testing:**  
    - Generate HMAC signed approval/rejection URLs with parameters (`cid`, `status`, `action`, `exp`, `sig`).  
    - Simulate clicking URLs (sending HTTP requests) to trigger workflow.  
    - Verify database updates, audit logs, and Telegram messages.

11. **Add Sticky Notes:**  
    - Include detailed notes for overview, security, Telegram chat ID usage, approval link explanation, and setup instructions for maintainers and operators.

---

## 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow adds a secure approval step with HMAC signed links, TTL validation, Telegram notifications, and audit logging.     | Overview Sticky Note describes the workflow purpose and features.                              |
| The Python code node requires a valid `SECRET_KEY` environment variable set in n8n's environment or `.env` file.                   | Security note emphasizing environment variable importance (Sticky Note1).                      |
| Telegram chat IDs must be replaced with actual user or admin chat IDs for message delivery. Use Telegram's `@userinfobot` to find IDs. | Telegram chat ID usage explanation (Sticky Note2).                                            |
| Approval links must be generated with HMAC-SHA256 signatures on `cid|status|exp` payload, including TTL expiry.                    | Approval link generation and verification explanation (Sticky Note3).                          |
| Step-by-step setup includes creating required DB tables, setting environment variables, adding credentials, importing workflow, and testing. | Detailed beginner-friendly setup in Sticky Note4.                                            |
| Audit and error tables allow full traceability and debugging of approval flows.                                                    | Database schema notes in Sticky Note4 and main overview.                                       |

---

**Disclaimer:** The text provided is extracted exclusively from an n8n automated workflow and adheres strictly to content policies. It contains no illegal, offensive, or protected elements. All data processed is public and lawful.

---