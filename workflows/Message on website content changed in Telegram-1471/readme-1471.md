Message on website content changed in Telegram

https://n8nworkflows.xyz/workflows/message-on-website-content-changed-in-telegram-1471


# Message on website content changed in Telegram

### 1. Workflow Overview

This workflow is designed to monitor changes on a specified website by periodically fetching its HTML content and comparing it to a previous snapshot. Its primary use case is to track updates on competitor blogs or news sites, for example by monitoring their sitemap or homepage for new articles. When a change is detected, the workflow automatically sends a notification via Telegram.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Periodically initiates the check every 5 minutes using a Cron node.
- **1.2 First Content Fetch:** Retrieves the current state of the target webpage’s HTML content.
- **1.3 Wait Period:** Pauses the workflow for a defined interval (5 minutes) before rechecking.
- **1.4 Second Content Fetch:** Retrieves the webpage content again after the wait.
- **1.5 Change Detection:** Compares the two fetched versions. If unchanged, stops; if changed, sends a Telegram notification.
- **1.6 Notification / No-Operation:** Either sends a Telegram message about the change or does nothing if no change is detected.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** Triggers the workflow every 5 minutes to start the monitoring cycle.
- **Nodes Involved:** `Cron`
- **Node Details:**
  - **Type:** Cron Trigger node; initiates workflow runs on schedule.
  - **Configuration:** Set to trigger every 5 minutes (`everyX` mode).
  - **Expressions/Variables:** None.
  - **Connections:** Outputs to `HTTP Request`.
  - **Edge Cases:** Cron node failure is rare; ensure n8n server uptime.
  - **Version:** Compatible with n8n v0.100+.
  
#### 2.2 First Content Fetch

- **Overview:** Fetches the current HTML content of the target website.
- **Nodes Involved:** `HTTP Request`
- **Node Details:**
  - **Type:** HTTP Request node; performs a GET request.
  - **Configuration:** URL set to `https://news.ycombinator.com/` (example site); response format is plain string.
  - **Expressions/Variables:** None.
  - **Connections:** Input from `Cron`; output to `Wait`.
  - **Edge Cases:** Possible network errors, HTTP errors, or site downtime. No authentication is needed here.
  - **Version:** Standard node available in all recent n8n versions.
  
#### 2.3 Wait Period

- **Overview:** Delays the workflow for 5 minutes before fetching the page again, allowing for time-based comparison.
- **Nodes Involved:** `Wait`
- **Node Details:**
  - **Type:** Wait node; pauses workflow execution.
  - **Configuration:** Waits for 5 minutes (`unit: minutes`, `amount: 5`).
  - **Expressions/Variables:** None.
  - **Connections:** Input from `HTTP Request`; output to `HTTP Request1`.
  - **Edge Cases:** Workflow timeouts if wait exceeds limits; ensure workflow settings permit long-running executions.
  - **Version:** Available in n8n v0.92+.
  
#### 2.4 Second Content Fetch

- **Overview:** Fetches the latest HTML content of the same website, after the wait.
- **Nodes Involved:** `HTTP Request1`
- **Node Details:**
  - **Type:** HTTP Request node.
  - **Configuration:** Same URL and response format as the first HTTP request.
  - **Expressions/Variables:** None.
  - **Connections:** Input from `Wait`; output to `IF`.
  - **Edge Cases:** Same as first HTTP request.
  - **Version:** Standard.
  
#### 2.5 Change Detection

- **Overview:** Compares the HTML content fetched in the first and second HTTP requests to detect any changes.
- **Nodes Involved:** `IF`
- **Node Details:**
  - **Type:** IF node; evaluates a boolean condition.
  - **Configuration:** Checks if the string content from first HTTP Request equals the content from the second HTTP Request.
  - **Expressions/Variables:** 
    - Condition expression: `{{$node["HTTP Request"].json["data"]}} == {{$node["HTTP Request"].json["data"]}}`
    - Note: The expression as provided appears flawed (compares the same node output to itself). The intended logic likely is to compare outputs of `HTTP Request` and `HTTP Request1`.
  - **Connections:** True branch to `NoOp`, False branch to `Telegram1`.
  - **Edge Cases:** The condition might fail due to expression syntax errors or unexpected data structure changes.
  - **Version:** IF node standard in all recent n8n versions.
  
#### 2.6 Notification / No-Operation

- **Overview:** Based on IF evaluation, either sends a Telegram notification about the detected change or stops workflow execution without action.
- **Nodes Involved:** `Telegram1`, `NoOp`
- **Node Details:**
  - **Telegram1:**
    - **Type:** Telegram node; sends a message.
    - **Configuration:** Sends fixed text "Something got changed" to a specific chatId `1234`.
    - **Credentials:** Uses Telegram API credentials named "n8n test bot".
    - **Edge Cases:** Auth failures, network errors, invalid chatId.
  - **NoOp:**
    - **Type:** No-Operation node; effectively a placeholder that does nothing.
    - **Configuration:** No parameters.
  - **Connections:** IF true → `NoOp`; IF false → `Telegram1`.
  - **Version:** Telegram node requires valid credentials configured in n8n.

---

### 3. Summary Table

| Node Name     | Node Type          | Functional Role             | Input Node(s)    | Output Node(s) | Sticky Note                         |
|---------------|--------------------|----------------------------|------------------|----------------|-----------------------------------|
| Cron          | Cron Trigger       | Initiates workflow every 5 min | —                | HTTP Request   |                                   |
| HTTP Request  | HTTP Request       | First webpage content fetch | Cron             | Wait           |                                   |
| Wait          | Wait               | Pauses execution 5 minutes | HTTP Request     | HTTP Request1  |                                   |
| HTTP Request1 | HTTP Request       | Second content fetch       | Wait             | IF             |                                   |
| IF            | If Condition       | Compares first and second fetch | HTTP Request1    | NoOp, Telegram1| Expression compares same node data (likely an error) |
| NoOp          | No Operation       | Stops if no change detected | IF (true)        | —              |                                   |
| Telegram1     | Telegram           | Sends notification on change | IF (false)       | —              | Requires Telegram credentials     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Node:**
   - Type: Cron Trigger
   - Set to trigger every 5 minutes (`everyX` mode, unit: minutes, value: 5).
   - This node starts the workflow.

2. **Create First HTTP Request Node:**
   - Type: HTTP Request
   - Set URL to the target webpage, e.g. `https://news.ycombinator.com/`.
   - Set Response Format to `String`.
   - Connect Cron’s output to this node’s input.

3. **Create Wait Node:**
   - Type: Wait
   - Configure to wait for 5 minutes (`unit: minutes`, `amount: 5`).
   - Connect the first HTTP Request’s output to Wait node input.

4. **Create Second HTTP Request Node:**
   - Type: HTTP Request
   - Same URL and Response Format as the first HTTP request.
   - Connect the Wait node output to this node’s input.

5. **Create IF Node:**
   - Type: If Condition
   - Configure a boolean condition comparing the output data of the first and second HTTP requests to detect if they differ.
   - **Important:** Set expressions correctly to compare the first HTTP Request `data` with second HTTP Request `data`. For example:
     ```
     {{$node["HTTP Request"].json["data"]}} == {{$node["HTTP Request1"].json["data"]}}
     ```
   - Connect second HTTP Request output to IF node input.

6. **Create NoOp Node:**
   - Type: No Operation
   - Connect IF node’s **true** output (meaning no change detected) to NoOp.

7. **Create Telegram Node:**
   - Type: Telegram
   - Configure Telegram API credentials (OAuth2 or Bot Token) for your Telegram bot.
   - Set Chat ID to your target Telegram chat.
   - Set message text to: "Something got changed".
   - Connect IF node’s **false** output (meaning change detected) to Telegram node.

8. **Validate Credentials:**
   - Ensure Telegram credentials are set up and working.
   - Test Telegram node independently if necessary.

9. **Activate Workflow:**
   - Save and activate the workflow.
   - Confirm it triggers every 5 minutes and sends notifications on detected changes.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                 |
|------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The example URL used is HackerNews homepage (`https://news.ycombinator.com/`). Replace with any target site or sitemap.xml URL to monitor other pages. | Example site for demonstration.                                                                |
| The workflow includes a wait node to space out fetches, which avoids immediate repetition and gives time for meaningful changes. | Time delay helps reduce false positives from immediate fetches.                                |
| Telegram messaging requires a bot and chat ID; ensure you have created a Telegram bot via BotFather and obtained the chat ID where messages will be sent. | Telegram Bot API documentation: https://core.telegram.org/bots/api                             |
| The IF node expression as provided in the JSON appears to incorrectly compare the same node’s output to itself; to function correctly, it must compare outputs of `HTTP Request` and `HTTP Request1`. | Expression syntax and node data referencing are critical for accurate condition evaluation.    |
| Consider edge cases such as site downtime, network errors, or HTML content changes that do not affect business logic (e.g., ads, dynamic timestamps). | Additional logic may be needed to filter out irrelevant changes or handle errors gracefully.  |

---

This documentation fully describes the workflow structure, logic, potential pitfalls, and how to reproduce it accurately in n8n.