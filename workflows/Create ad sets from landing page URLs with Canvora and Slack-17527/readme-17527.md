Create ad sets from landing page URLs with Canvora and Slack

https://n8nworkflows.xyz/workflows/create-ad-sets-from-landing-page-urls-with-canvora-and-slack-17527


# Create ad sets from landing page URLs with Canvora and Slack

### 1. Workflow Overview

The **Landing Page URL → Complete Ad Set (Canvora)** workflow automates the generation of multi-format digital advertising creatives (square, landscape, and portrait placements) using the Canvora API based on a submitted landing page URL. It provides an end-to-end integration starting from user input collection, asynchronous API processing with polling, result sorting, and notification delivery via Slack.

The workflow is categorized into the following logical blocks:
- **1.1 Input Reception:** Captures the target landing page URL via an interactive n8n form.
- **1.2 API Generation Trigger:** Dispatches the URL and requested output specifications to the Canvora Generations API.
- **1.3 Status Polling & Loop:** Delays execution, checks the generation status against terminal states or retry limits, and controls looping logic.
- **1.4 Result Delivery & Notification:** Evaluates final generation outcomes, formats successful ad sets, or triggers failure alerts to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception
- **Overview:** This block collects the necessary input parameters from an end-user to initiate the creative generation process.
- **Nodes Involved:** `When Ad Form Submitted`
- **Node Details:**
  - **When Ad Form Submitted**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (v2.2) — Acts as a webhook-backed web form entry point.
    - *Configuration Choices:* Configured with a single required text field labeled "Landing page URL" and form title "Ad set generator".
    - *Key Expressions or Variables:* Extracts `$('When Ad Form Submitted').item.json['Landing page URL']`.
    - *Input and Output Connections:* Input: None (Webhook trigger); Output: Connects to `Post URL to Canvora API`.
    - *Version-specific Requirements:* Form trigger v2.2.
    - *Edge Cases / Failure Types:* Submission validation errors if the user leaves the URL field empty.

#### 1.2 API Generation Trigger
- **Overview:** This block sends an HTTP POST request to Canvora to start generating the requested ad formats from the provided URL.
- **Nodes Involved:** `Post URL to Canvora API`
- **Node Details:**
  - **Post URL to Canvora API**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — Communicates with the external Canvora REST API.
    - *Configuration Choices:* HTTP POST method targeting `https://api.canvora.ai/api/generations` using Generic Header Authentication (`X-API-Key`).
    - *Key Expressions or Variables:* Constructs a JSON string body containing `inputType: 'url'`, the input URL from the form trigger, a dynamic title, and `outputFormats: ['ad_square', 'ad_landscape', 'ad_portrait']`.
    - *Input and Output Connections:* Input: `When Ad Form Submitted`; Output: `Wait 30 Seconds for Canvora`.
    - *Edge Cases / Failure Types:* Authentication errors due to missing/invalid API keys, rate-limiting, or upstream API service outages.

#### 1.3 Status Polling & Loop
- **Overview:** This block implements a polling mechanism that waits between checks, queries the generation status from Canvora, and evaluates whether the process has finished or timed out.
- **Nodes Involved:** `Wait 30 Seconds for Canvora`, `Fetch Canvora Generation Status`, `If Generation Completed`, `If Generation Timeout`
- **Node Details:**
  - **Wait 30 Seconds for Canvora**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (v1.1) — Pauses execution for a set duration to allow asynchronous rendering on the API side.
    - *Configuration Choices:* Fixed delay amount of 30 seconds.
    - *Input and Output Connections:* Input: `Post URL to Canvora API` or index 1 of `If Generation Timeout`; Output: `Fetch Canvora Generation Status`.
  - **Fetch Canvora Generation Status**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — Queries the current job status.
    - *Configuration Choices:* HTTP GET method using Generic Header Authentication (`X-API-Key`).
    - *Key Expressions or Variables:* Dynamically builds the URL using `={{ 'https://api.canvora.ai/api/generations/' + $('Post URL to Canvora API').item.json.generation.id }}`.
    - *Input and Output Connections:* Input: `Wait 30 Seconds for Canvora`; Output: `If Generation Completed`.
  - **If Generation Completed**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2) — Branches workflow execution based on the job state.
    - *Configuration Choices:* Checks if the status array `['completed', 'failed', 'partial', 'cancelled']` includes `$json.generation.status`.
    - *Input and Output Connections:* Input: `Fetch Canvora Generation Status`; Output True: `If Generation Successful`; Output False: `If Generation Timeout`.
  - **If Generation Timeout**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2) — Prevents infinite loops by checking the run execution index.
    - *Configuration Choices:* Evaluates whether `$runIndex >= 20` (maximum of 20 polling attempts / 10 minutes).
    - *Input and Output Connections:* Input: `If Generation Completed` (false branch); Output True: `Send Slack Failure Notification`; Output False: `Wait 30 Seconds for Canvora` (loop continuation).

#### 1.4 Result Delivery & Notification
- **Overview:** This block handles final outcomes by either processing and formatting successful ad creative assets for Slack or dispatching failure notifications.
- **Nodes Involved:** `If Generation Successful`, `Sort Ad Outputs by Order`, `Post Ad Set to Slack Channel`, `Send Slack Failure Notification`
- **Node Details:**
  - **If Generation Successful**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2) — Differentiates between successful completions and failed/cancelled states.
    - *Configuration Choices:* Checks if the status array `['completed', 'partial']` includes `$json.generation.status`.
    - *Input and Output Connections:* Input: `If Generation Completed` (true branch); Output True: `Sort Ad Outputs by Order`; Output False: `Send Slack Failure Notification`.
  - **Sort Ad Outputs by Order**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript execution node to sort, map, and format ad outputs.
    - *Configuration Choices:* Sorts output items by `sortOrder` and constructs a Markdown-formatted Slack text message.
    - *Key Expressions or Variables:* Accesses `$input.first().json.generation` and maps asset file URLs.
    - *Input and Output Connections:* Input: `If Generation Successful` (true branch); Output: `Post Ad Set to Slack Channel`.
  - **Post Ad Set to Slack Channel**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (v2.2) — Sends formatted rich messages to a Slack workspace.
    - *Configuration Choices:* Posts to channel name `#content` using text content generated upstream.
    - *Input and Output Connections:* Input: `Sort Ad Outputs by Order`; Output: None.
    - *Edge Cases / Failure Types:* Slack OAuth credential expiration, invalid channel names, or API rate limits.
  - **Send Slack Failure Notification**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (v2.2) — Alerts users of failures or timeouts.
    - *Configuration Choices:* Posts failure context and status to channel `#content`.
    - *Input and Output Connections:* Input: `If Generation Timeout` (true branch) or `If Generation Successful` (false branch); Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **When Ad Form Submitted** | `n8n-nodes-base.formTrigger` | Collects landing page URL | None | Post URL to Canvora API | Collect the URL: A built-in n8n form collects the landing page URL that the ad set will be designed from. |
| **Post URL to Canvora API** | `n8n-nodes-base.httpRequest` | Triggers Canvora ad generation | When Ad Form Submitted | Wait 30 Seconds for Canvora | Start Canvora generation: Sends the URL to the Canvora generation API requesting square, landscape, and portrait ad formats. |
| **Wait 30 Seconds for Canvora** | `n8n-nodes-base.wait` | Delays status checking | Post URL to Canvora API, If Generation Timeout | Fetch Canvora Generation Status | Poll generation status: Waits between status checks, fetches the generation state, and loops until completion or timeout. |
| **Fetch Canvora Generation Status** | `n8n-nodes-base.httpRequest` | Fetches generation progress | Wait 30 Seconds for Canvora | If Generation Completed | Poll generation status: Waits between status checks, fetches the generation state, and loops until completion or timeout. |
| **If Generation Completed** | `n8n-nodes-base.if` | Checks if job reached terminal state | Fetch Canvora Generation Status | If Generation Successful, If Generation Timeout | Poll generation status: Waits between status checks, fetches the generation state, and loops until completion or timeout. |
| **If Generation Timeout** | `n8n-nodes-base.if` | Enforces maximum poll limit | If Generation Completed | Send Slack Failure Notification, Wait 30 Seconds for Canvora | Poll generation status: Waits between status checks, fetches the generation state, and loops until completion or timeout. |
| **If Generation Successful** | `n8n-nodes-base.if` | Verifies success vs partial/failed | If Generation Completed | Sort Ad Outputs by Order, Send Slack Failure Notification | Poll generation status: Waits between status checks, fetches the generation state, and loops until completion or timeout. |
| **Sort Ad Outputs by Order** | `n8n-nodes-base.code` | Sorts and formats ad assets | If Generation Successful | Post Ad Set to Slack Channel | Deliver the ad set: Sorts the finished ads and posts them to Slack with a review link. |
| **Post Ad Set to Slack Channel** | `n8n-nodes-base.slack` | Posts final ad assets to Slack | Sort Ad Outputs by Order | None | Deliver the ad set: Sorts the finished ads and posts them to Slack with a review link. |
| **Send Slack Failure Notification** | `n8n-nodes-base.slack` | Posts failure alert to Slack | If Generation Successful, If Generation Timeout | None | Send failure alert: Posts a Slack notification when generation fails or times out. |

*(Note: The global workflow sticky note content summarizing the entire workflow setup, customization options, and requirements applies broadly to the workflow canvas and is detailed in Section 1 and Section 5).*

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node:**
   - Add a **Form Trigger** node named `When Ad Form Submitted`.
   - Set form title to `Ad set generator`.
   - Add a form field: field label `Landing page URL`, mark as a required field.
2. **Create the Canvora Generation Trigger Node:**
   - Add an **HTTP Request** node named `Post URL to Canvora API`.
   - Set Method to `POST`, URL to `https://api.canvora.ai/api/generations`.
   - Configure Authentication to Generic Credential Type -> Header Auth (`X-API-Key`).
   - Enable JSON body parameter specification and add expression:
     ```javascript
     ={{ JSON.stringify({
       inputType: 'url',
       inputUrl: $('When Ad Form Submitted').item.json['Landing page URL'],
       title: 'Ad set: ' + $('When Ad Form Submitted').item.json['Landing page URL'],
       outputFormats: ['ad_square', 'ad_landscape', 'ad_portrait']
     }) }}
     ```
   - Connect `When Ad Form Submitted` output to this node.
3. **Create the Polling Loop Mechanism:**
   - Add a **Wait** node named `Wait 30 Seconds for Canvora`. Set amount to `30` seconds. Connect `Post URL to Canvora API` output to this node.
   - Add an **HTTP Request** node named `Fetch Canvora Generation Status`. Set Method to `GET` and URL to `={{ 'https://api.canvora.ai/api/generations/' + $('Post URL to Canvora API').item.json.generation.id }}` using the same Header Auth credential. Connect `Wait 30 Seconds for Canvora` output here.
   - Add an **IF** node named `If Generation Completed`. Set condition to evaluate whether `={{ ['completed', 'failed', 'partial', 'cancelled'].includes($json.generation.status) }}` equals `true`. Connect `Fetch Canvora Generation Status` output here.
   - Add an **IF** node named `If Generation Timeout` connected to the `false` output of `If Generation Completed`. Set condition to evaluate whether `={{ $runIndex >= 20 }}` equals `true`.
   - Connect the `false` output of `If Generation Timeout` back to the input of `Wait 30 Seconds for Canvora` to establish the loop.
4. **Create the Outcome Evaluation Nodes:**
   - Add an **IF** node named `If Generation Successful` connected to the `true` output of `If Generation Completed`. Set condition to evaluate whether `={{ ['completed', 'partial'].includes($json.generation.status) }}` equals `true`.
5. **Create the Success Formatting and Delivery Nodes:**
   - Add a **Code** node named `Sort Ad Outputs by Order` connected to the `true` output of `If Generation Successful`. Paste the following JavaScript code:
     ```javascript
     const g = $input.first().json.generation;
     const outputs = (g.outputs || []).slice().sort((a, b) => (a.sortOrder ?? 0) - (b.sortOrder ?? 0));
     const lines = outputs.map((o, i) => `${i + 1}. ${o.format || 'ad'} — ${o.fileUrl}`);
     const text = [
       `📣 *Ad set ready* for ${$('When Ad Form Submitted').item.json['Landing page URL']}`,
       `${outputs.length} placement(s)${g.status === 'partial' ? ' (some failed and were refunded)' : ''}:`,
       '',
       ...lines,
       '',
       `Review or edit: https://canvora.ai/generations/${g.id}`
     ].join('\n');
     return [{ json: { text, outputs, generationId: g.id } }];
     ```
   - Add a **Slack** node named `Post Ad Set to Slack Channel` connected to `Sort Ad Outputs by Order`.
   - Set Slack action/resource parameters to post message to channel `#content` with text expression `={{ $json.text }}` using connected Slack OAuth credentials.
6. **Create the Failure Notification Node:**
   - Add a **Slack** node named `Send Slack Failure Notification`. Connect the `true` output of `If Generation Timeout` and the `false` output of `If Generation Successful` to this node.
   - Configure it to post to channel `#content` with text expression:
     ```text
     =⚠️ Canvora ad set didn't finish for {{ $('When Ad Form Submitted').item.json['Landing page URL'] }} (status: {{ $json.generation ? $json.generation.status : 'timeout' }}). Failed generations refund credits automatically.
     ```

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Canvora Platform and API Key Generation | [canvora.ai](https://canvora.ai) |
| Canvora API Formats Reference Guide | [Canvora Formats API](https://api.canvora.ai/api/formats) |
| Workflow Agent Integration Documentation | [canvora.ai/for-agents](https://canvora.ai/for-agents) |