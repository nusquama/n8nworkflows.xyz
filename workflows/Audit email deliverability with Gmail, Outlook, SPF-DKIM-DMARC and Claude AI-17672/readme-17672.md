Audit email deliverability with Gmail, Outlook, SPF/DKIM/DMARC and Claude AI

https://n8nworkflows.xyz/workflows/audit-email-deliverability-with-gmail--outlook--spf-dkim-dmarc-and-claude-ai-17672


# Audit email deliverability with Gmail, Outlook, SPF/DKIM/DMARC and Claude AI

### 1. Workflow Overview

This workflow automates the comprehensive auditing of a sending domain’s email deliverability and mailbox placement. It performs checks across cryptographic authentication standards (SPF, DKIM, DMARC, and MX records), analyzes real email placement inside connected Gmail and Outlook mailboxes, leverages Anthropic’s Claude AI to interpret results and rank prioritization fixes, and compiles a final text report dispatched via Gmail.

The workflow logic is grouped into five functional blocks:
- **1.1 Initialization & Configuration:** Triggers the workflow execution and establishes core parameters such as the target domain, lookback windows, DKIM selectors, and sender address.
- **1.2 Technical DNS & Authentication Branch:** Constructs DNS queries and queries Google's DNS-over-HTTPS (DoH) API to evaluate SPF policy validity, lookup limits, DMARC configuration, MX existence, and active DKIM selectors.
- **1.3 Mailbox Placement Verification Branches:** Independently inspects real-world message delivery behavior across two major webmail providers (Gmail system labels and Microsoft Outlook folders/inference classifications).
- **1.4 AI Triage & Report Compilation:** Synchronizes authentication and placement signals, structures a prompt for Anthropic Claude, submits it via the Messages API, and parses the structured response into a formatted text report.
- **1.5 Notification Dispatch:** Delivers the finalized deliverability report directly to a specified recipient via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Configuration
- **Overview:** Initializes the manual trigger and establishes the configuration baseline used across all subsequent authentication and mailbox reading branches.
- **Nodes Involved:** 
  - `Run manually`
  - `Set config: deliverability`
- **Node Details:**
  - **Run manually**
    - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` (Manual Execution Trigger)
    - *Configuration Choices:* Standard manual trigger with no parameters.
    - *Key Expressions:* None.
    - *Connections:* Input: None | Output: `Set config: deliverability`
    - *Edge Cases/Failures:* Requires manual execution by a user or webhook invocation.
  - **Set config: deliverability**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Defines configuration constants including `LOOKBACK_DAYS` (30), `SINCE_ISO`, `DOMAIN` (`nocode.expert`), a comprehensive list of `DKIM_SELECTORS` for major ESPs, `SENDER_ADDRESS` (`user@example.com`), `MAX_MESSAGES` (25), and `SPF_LOOKUP_WARN` (8).
    - *Key Expressions:* Dynamic ISO date generation for filtering (`new Date(Date.now() - LOOKBACK_DAYS * 86400000).toISOString()`).
    - *Connections:* Input: `Run manually` | Output: `Build DNS queries`, `Read Gmail`, `Read Outlook inbox`
    - *Edge Cases/Failures:* Malformed domain strings or invalid JS syntax within the code block.

---

#### 2.2 Technical DNS & Authentication Branch
- **Overview:** Programmatically generates DNS queries for SPF, DMARC, MX, and common DKIM selectors, resolves them via Google's DNS-over-HTTPS service, pairs answers back to their respective questions, and grades the domain's email authentication posture.
- **Nodes Involved:**
  - `Build DNS queries`
  - `Look up DNS over HTTPS`
  - `Attach question to answer`
  - `Assess authentication`
- **Node Details:**
  - **Build DNS queries**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Parses the domain string and loops through the configured `DKIM_SELECTORS` array to assemble an array of query payloads (`spf`, `dmarc`, `mx`, `dkim`).
    - *Key Expressions:* Accesses config via `$('Set config: deliverability').first().json`.
    - *Connections:* Input: `Set config: deliverability` | Output: `Look up DNS over HTTPS`
  - **Look up DNS over HTTPS**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (HTTP API Request)
    - *Configuration Choices:* Calls `https://dns.google/resolve` using query parameters (`name`, `type`) with error suppression enabled (`neverError: true`).
    - *Key Expressions:* Query parameters bound to `={{ $json.name }}` and `={{ $json.type }}`.
    - *Connections:* Input: `Build DNS queries` | Output: `Attach question to answer`
    - *Edge Cases/Failures:* Network timeouts or Google DoH rate-limiting.
  - **Attach question to answer**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Re-aligns HTTP response items with their originating questions using index matching.
    - *Connections:* Input: `Look up DNS over HTTPS` | Output: `Assess authentication`
  - **Assess authentication**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Analyzes returned DoH records to grade SPF records (checking for multiple records, `+all` policies, and exceeding the hard limit of 10 DNS lookups), DMARC policy levels (`p=none`, `p=quarantine`, `p=reject`), reporting flags (`rua`), DKIM public keys, and MX record presence. Outputs a consolidated grading checklist and verdict (`clean`, `at risk`, `failing`).
    - *Connections:* Input: `Attach question to answer` | Output: `Merge branches` (Input 0)

---

#### 2.3 Mailbox Placement Verification Branches
- **Overview:** Fetches recent messages received from the specified sender address across Gmail and Microsoft Outlook mailboxes, mapping internal system labels and folder attributes to inbox placement categories (Primary/Promotions/Spam or Focused/Other/Junk).
- **Nodes Involved:**
  - `Read Gmail`
  - `Read Gmail placement`
  - `Read Outlook inbox`
  - `Read Outlook junk`
  - `Read Outlook placement`
- **Node Details:**
  - **Read Gmail**
    - *Type & Technical Role:* `n8n-nodes-base.gmail` (Gmail Integration)
    - *Configuration Choices:* Retrieves messages using query filters for sender address and time window (`newer_than`), including Spam and Trash folders (`includeSpamTrash: true`). Configured with error continuation (`continueRegularOutput`).
    - *Key Expressions:* Message limits and search queries dynamically bound to `Set config: deliverability` values.
    - *Connections:* Input: `Set config: deliverability` | Output: `Read Gmail placement`
    - *Credentials Required:* Gmail OAuth2.
    - *Edge Cases/Failures:* OAuth token expiration or missing API permissions.
  - **Read Gmail placement**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Maps Gmail `labelIds` (`SPAM`, `TRASH`, `CATEGORY_PROMOTIONS`, `INBOX`, etc.) into standardized placement tabs and calculates percentage metrics.
    - *Connections:* Input: `Read Gmail` | Output: `Merge branches` (Input 1)
  - **Read Outlook inbox**
    - *Type & Technical Role:* `n8n-nodes-base.microsoftOutlook` (Microsoft Outlook Integration)
    - *Configuration Choices:* Fetches raw messages from the Inbox folder filtered by sender and received date. Configured with error continuation.
    - *Key Expressions:* Filter parameters reference configuration variables.
    - *Connections:* Input: `Set config: deliverability` | Output: `Read Outlook junk`
    - *Credentials Required:* Microsoft Outlook OAuth2 (Graph API).
  - **Read Outlook junk**
    - *Type & Technical Role:* `n8n-nodes-base.microsoftOutlook` (Microsoft Outlook Integration)
    - *Configuration Choices:* Fetches raw messages specifically from the Junk email folder. Configured with error continuation.
    - *Key Expressions:* Filter parameters reference configuration variables.
    - *Connections:* Input: `Read Outlook inbox` | Output: `Read Outlook placement`
    - *Credentials Required:* Microsoft Outlook OAuth2 (Graph API).
  - **Read Outlook placement**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Aggregates inbox and junk message arrays, evaluates `inferenceClassification` (`focused` vs `other`), tallies items, and calculates delivery percentages.
    - *Connections:* Input: `Read Outlook junk` | Output: `Merge branches` (Input 2)

---

#### 2.4 AI Triage & Report Compilation
- **Overview:** Consolidates authentication checks and mailbox placement metrics, structures a specialized prompt for Anthropic Claude, sends the API request, and parses the returned JSON payload into a comprehensive plain-text deliverability report.
- **Nodes Involved:**
  - `Merge branches`
  - `Combine signals`
  - `Build AI prompt`
  - `Explain and rank with Claude AI`
  - `compile deliverability report`
- **Node Details:**
  - **Merge branches**
    - *Type & Technical Role:* `n8n-nodes-base.merge` (Data Merger)
    - *Configuration Choices:* Uses 3 input streams to synchronize Branch A (authentication), Branch B (Gmail placement), and Branch C (Outlook placement).
    - *Connections:* Input: `Assess authentication`, `Read Gmail placement`, `Read Outlook placement` | Output: `Combine signals`
  - **Combine signals**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Gathers data from all three branches, calculates pooled mailbox placement averages, and formats a unified data object.
    - *Connections:* Input: `Merge branches` | Output: `Build AI prompt`
  - **Build AI prompt**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Assembles system prompts and user markdown instructions for Claude, establishing persona constraints, tone guidelines, and output JSON schemas.
    - *Connections:* Input: `Combine signals` | Output: `Explain and rank with Claude AI`
  - **Explain and rank with Claude AI**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (HTTP API Request)
    - *Configuration Choices:* Sends a POST request to Anthropic's Messages API (`https://api.anthropic.com/v1/messages`) using model `claude-haiku-4-5` with retry configuration (`maxTries: 3`, wait time 2000ms).
    - *Key Expressions:* Request body bound to `={{ $json.body }}`; API key header bound to `={{ $env.ANTHROPIC_API_KEY }}`.
    - *Credentials Required:* Custom Environment Variable (`ANTHROPIC_API_KEY`).
    - *Edge Cases/Failures:* API rate limits, invalid API keys, or timeout errors.
  - **compile deliverability report**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code Execution)
    - *Configuration Choices:* Parses Claude's JSON response, formats structured headers, technical assessment breakdowns, per-provider placement statistics, prioritized remediation steps, and recent email subject lines into an aligned plain-text report.
    - *Connections:* Input: `Explain and rank with Claude AI` | Output: `Send a message`

---

#### 2.5 Notification Dispatch
- **Overview:** Dispatches the finalized plain-text email deliverability report to the designated recipient via Gmail.
- **Nodes Involved:**
  - `Send a message`
- **Node Details:**
  - **Send a message**
    - *Type & Technical Role:* `n8n-nodes-base.gmail` (Gmail Integration)
    - *Configuration Choices:* Sends an email message containing the compiled deliverability report string.
    - *Key Expressions:* Message body bound to `={{ $json.report }}`.
    - *Credentials Required:* Gmail OAuth2.
    - *Edge Cases/Failures:* Invalid recipient configuration or Gmail daily sending limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Run manually` | `manualTrigger` | Triggers the workflow execution manually | None | `Set config: deliverability` | Email Deliverability Checker <br><br>Two independent branches. One audits the authentication on your sending domain. The other reads where your real campaigns actually landed, per mailbox provider. Most tools in this space need a paid seed network. This one reads each provider's own classification instead.<br><br>### How it works<br>- Technical branch resolves SPF, DKIM, DMARC and MX over DNS-over-HTTPS, so no DNS node and no credentials are needed.<br>- Gmail branch reads system labels, which is how Primary, Promotions and Spam are known exactly rather than guessed.<br>- Outlook branch reads the Inbox and Junk folders separately and uses inferenceClassification for the Focused and Other split.<br>- Both placement branches are optional. Any provider with no credential attached is reported as not connected and the run still completes.<br>- Claude ranks the fixes and reads the disagreement between providers, which is the signal that separates a domain fault from a content problem.<br><br>### Setup<br>1. Set DOMAIN in the Config node. That alone runs the technical branch, with no credentials.<br>2. Set SENDER_ADDRESS to the address your campaigns are sent FROM, then attach a Gmail or Outlook credential to whichever mailbox receives them.<br>3. Add ANTHROPIC_API_KEY to your environment.<br><br>Built by **nocode.expert** - done-for-you automation & tracking. https://nocode.expert |
| `Set config: deliverability` | `code` | Defines configuration parameters (domain, lookback days, selectors) | `Run manually` | `Build DNS queries`, `Read Gmail`, `Read Outlook inbox` | Email Deliverability Checker ... (refer to above) |
| `Build DNS queries` | `code` | Generates string queries for SPF, DMARC, MX, and DKIM selectors | `Set config: deliverability` | `Look up DNS over HTTPS` | Branch A, technical. No credentials required<br><br>One HTTP Request node answers every DNS question, including one per DKIM selector, so the list can grow without adding nodes. The check worth knowing about is the SPF lookup count: SPF allows 10 DNS lookups and going over does not degrade gracefully, the record returns permerror and every check fails. DMARC at p=none is reported as a flag rather than a pass, because it looks configured but protects nothing. |
| `Look up DNS over HTTPS` | `httpRequest` | Resolves DNS records via Google DoH API | `Build DNS queries` | `Attach question to answer` | Branch A, technical. No credentials required ... (refer to above) |
| `Attach question to answer` | `code` | Pairs DNS answers back to original questions | `Look up DNS over HTTPS` | `Assess authentication` | Branch A, technical. No credentials required ... (refer to above) |
| `Assess authentication` | `code` | Grades authentication records (SPF, DMARC, DKIM, MX) | `Attach question to answer` | `Merge branches` | Branch A, technical. No credentials required ... (refer to above) |
| `Read Gmail` | `gmail` | Fetches recent messages from Gmail | `Set config: deliverability` | `Read Gmail placement` | Branch B, Gmail<br><br>Labels are the placement. CATEGORY_PROMOTIONS is the Promotions tab, SPAM is the spam folder. |
| `Read Gmail placement` | `code` | Analyzes Gmail message label IDs into placement metrics | `Read Gmail` | `Merge branches` | Branch B, Gmail ... (refer to above) |
| `Read Outlook inbox` | `microsoftOutlook` | Fetches recent messages from Outlook Inbox | `Set config: deliverability` | `Read Outlook junk` | Branch C, Outlook. And why Yahoo and Apple are not here<br><br>A Graph message carries parentFolderId but not the folder name, so the Inbox and Junk folders are queried separately and the folder is known from which node returned the message. No ID lookup and no tenant-specific values. Within the inbox, inferenceClassification gives the Focused and Other split.<br><br>Yahoo and iCloud are deliberately absent. Neither exposes a usable mail API for this, and the only route is IMAP, where n8n's IMAP node is a trigger and cannot fetch mid-workflow. Adding a fake branch for them would report no data forever. If you need them, run a separate IMAP-trigger workflow that logs placement to a sheet.<br><br>Be honest about what this measures. It is mail delivered to the mailboxes you connected, each with its own engagement history, so it is a strong signal about authentication and content, not a prediction for your whole list. |
| `Read Outlook junk` | `microsoftOutlook` | Fetches recent messages from Outlook Junk folder | `Read Outlook inbox` | `Read Outlook placement` | Branch C, Outlook. And why Yahoo and Apple are not here ... (refer to above) |
| `Read Outlook placement` | `code` | Aggregates Outlook inbox and junk data into metrics | `Read Outlook junk` | `Merge branches` | Branch C, Outlook. And why Yahoo and Apple are not here ... (refer to above) |
| `Merge branches` | `merge` | Synchronizes authentication and placement branches | `Assess authentication`, `Read Gmail placement`, `Read Outlook placement` | `Combine signals` | |
| `Combine signals` | `code` | Merges branch objects and computes pooled metrics | `Merge branches` | `Build AI prompt` | |
| `Build AI prompt` | `code` | Formats system instructions and user prompt for Claude | `Combine signals` | `Explain and rank with Claude AI` | |
| `Explain and rank with Claude AI` | `httpRequest` | Calls Anthropic Messages API to triage findings | `Build AI prompt` | `compile deliverability report` | |
| `compile deliverability report` | `code` | Parses AI response and compiles plain-text report | `Explain and rank with Claude AI` | `Send a message` | |
| `Send a message` | `gmail` | Emails the final deliverability report via Gmail | `compile deliverability report` | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger:** Add a **Manual Trigger** (`Run manually`) node.
2. **Add Configuration Node:** Add a **Code** node named `Set config: deliverability`. Populate its JavaScript code to define `DOMAIN`, `LOOKBACK_DAYS`, `DKIM_SELECTORS` array, and `SENDER_ADDRESS`. Connect `Run manually` to this node.
3. **Set up Branch A (Technical DNS):**
   - Add a **Code** node named `Build DNS queries` to construct query payloads for SPF, DMARC, MX, and DKIM selectors. Connect `Set config: deliverability` to it.
   - Add an **HTTP Request** node named `Look up DNS over HTTPS`. Set the URL to `https://dns.google/resolve`, configure query parameters (`name` and `type`), and enable **Never Error** under options. Connect `Build DNS queries` here.
   - Add a **Code** node named `Attach question to answer` to align DoH responses with their queries. Connect `Look up DNS over HTTPS` here.
   - Add a **Code** node named `Assess authentication` to grade authentication records and output validation statuses. Connect `Attach question to answer` here.
4. **Set up Branch B (Gmail Placement):**
   - Add a **Gmail** node named `Read Gmail` (Operation: `Get Many`). Configure filters to query messages by `SENDER_ADDRESS` and time frame, including spam and trash. Set error handling to **Continue Regular Output**. Connect `Set config: deliverability` here. Configure **Gmail OAuth2** credentials.
   - Add a **Code** node named `Read Gmail placement` to parse Gmail `labelIds` into delivery categories. Connect `Read Gmail` here.
5. **Set up Branch C (Outlook Placement):**
   - Add a **Microsoft Outlook** node named `Read Outlook inbox` (Operation: `Get Many`, Output: `Raw`). Configure filters for the sender address, received date, and folder (`inbox`). Set error handling to **Continue Regular Output**. Connect `Set config: deliverability` here. Configure **Microsoft Outlook OAuth2** credentials.
   - Add a **Microsoft Outlook** node named `Read Outlook junk` (Operation: `Get Many`, Output: `Raw`). Filter by sender address, date, and folder (`junkemail`). Set error handling to **Continue Regular Output**. Connect `Read Outlook inbox` here. Configure the same **Microsoft Outlook OAuth2** credentials.
   - Add a **Code** node named `Read Outlook placement` to aggregate inbox and junk arrays and calculate percentages. Connect `Read Outlook junk` here.
6. **Synchronize & Merge Branches:**
   - Add a **Merge** node named `Merge branches` with `Number Inputs` set to `3`. Connect `Assess authentication` to Input 0, `Read Gmail placement` to Input 1, and `Read Outlook placement` to Input 2.
   - Add a **Code** node named `Combine signals` to unify branch metrics and calculate pooled averages. Connect `Merge branches` here.
7. **Configure AI Processing:**
   - Add a **Code** node named `Build AI prompt` to construct the prompt payload and system constraints for Claude. Connect `Combine signals` here.
   - Add an **HTTP Request** node named `Explain and rank with Claude AI`. Set method to `POST`, URL to `https://api.anthropic.com/v1/messages`, specify body as `JSON`, and configure headers (`x-api-key`, `anthropic-version`, `content-type`). Enable retries on fail (3 attempts). Connect `Build AI prompt` here. Set the `x-api-key` header to reference the environment variable `{{ $env.ANTHROPIC_API_KEY }}`.
8. **Compile & Dispatch Report:**
   - Add a **Code** node named `compile deliverability report` to parse Claude's JSON output and assemble a formatted plain-text report. Connect `Explain and rank with Claude AI` here.
   - Add a **Gmail** node named `Send a message` (Operation: `Send`). Bind the message parameter to `={{ $json.report }}`. Connect `compile deliverability report` here. Configure **Gmail OAuth2** credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built by **nocode.expert** - done-for-you automation & tracking. | [nocode.expert website](https://nocode.expert) |