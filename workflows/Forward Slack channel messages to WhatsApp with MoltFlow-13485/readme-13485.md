Forward Slack channel messages to WhatsApp with MoltFlow

https://n8nworkflows.xyz/workflows/forward-slack-channel-messages-to-whatsapp-with-moltflow-13485


# Forward Slack channel messages to WhatsApp with MoltFlow

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Forward Slack Channel Messages to WhatsApp with MoltFlow  
**Purpose:** Receive Slack channel message events via an HTTP webhook, format them, and forward them to a WhatsApp number using the MoltFlow (waiflow) API.  
**Typical use cases:** On-call/ops alerts, incident updates, critical channel monitoring, executive notifications, and “don’t miss Slack” delivery to mobile.

### 1.1 Input Reception (Slack → n8n)
Slack calls an n8n webhook URL when a message is posted (or when Slack sends a verification challenge).

### 1.2 Message Normalization & Formatting
The workflow extracts text/user/channel from different possible Slack payload shapes, handles Slack “challenge” requests, and builds a WhatsApp-ready message.

### 1.3 Validation Gate
A boolean check prevents sending empty or intentionally skipped messages.

### 1.4 WhatsApp Delivery via MoltFlow
An HTTP request sends the formatted message to MoltFlow’s API using an API key in an HTTP header credential.

### 1.5 Logging / Output
A final code node returns a compact “forwarded” status response.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Slack Webhook)
**Overview:** Exposes a POST endpoint that Slack can call. It is the workflow entry point.  
**Nodes involved:** `Slack Webhook`

#### Node: Slack Webhook
- **Type / role:** `Webhook` node — inbound HTTP trigger.
- **Key configuration:**
  - **HTTP Method:** POST
  - **Path:** `slack-forward`  
  - **Webhook ID:** `slack-forward` (internal n8n identifier)
- **Inputs / outputs:**
  - **Input:** External HTTP request from Slack (payload in request body)
  - **Output:** One item containing the incoming request data (n8n typically provides body under `json.body` depending on configuration/version).
- **Connections:**
  - Outputs to: `Format Message`
- **Edge cases / failures:**
  - Slack may send different payload shapes depending on whether you use Events API, outgoing webhooks, or custom integrations.
  - Slack URL verification (`challenge`) must be answered correctly/quickly.
  - If Slack sends `application/x-www-form-urlencoded`, your extraction logic must match what n8n stores in `json.body` vs `json`.
- **Version notes:** `typeVersion: 2` — behavior for webhook payload structure can differ slightly across n8n versions; this workflow defensively checks both `$input.first().json.body` and `$input.first().json`.

---

### Block 2 — Message Normalization & Formatting (Code)
**Overview:** Converts Slack payloads into a unified structure for WhatsApp delivery, including handling Slack’s `challenge` handshake and skipping empty messages.  
**Nodes involved:** `Format Message`

#### Node: Format Message
- **Type / role:** `Code` node — transforms input JSON and constructs outbound message fields.
- **Key configuration choices:**
  - **Mode:** “Run Once for All Items” (processes all incoming items together; here it effectively uses the first item).
  - **Hardcoded constants (must be set):**
    - `SESSION_ID = 'YOUR_SESSION_ID'`
    - `MY_PHONE = 'YOUR_PHONE'`
- **Important logic / variables:**
  - **Payload extraction:**
    - `const body = $input.first().json.body || $input.first().json;`
  - **Slack verification challenge handling:**
    - If `body.challenge` exists, returns `{ challenge: body.challenge }` immediately.
    - Note: the rest of the workflow still proceeds unless you explicitly stop it with additional logic; here it returns an item that will reach the `Valid?` node but won’t have `skip` explicitly set (see edge cases).
  - **Event normalization:**
    - `const event = body.event || body;`
    - `text = event.text || event.message || ''`
    - `user = event.user_name || event.user || 'Someone'`
    - `channel = event.channel_name || event.channel || 'a channel'`
  - **Skip logic:**
    - If no `text`, return `{ skip: true, reason: 'Empty message' }`.
  - **WhatsApp message formatting:**
    - `*Slack — #${channel}*\n${user}: ${text}`
  - **Outbound fields produced:**
    - `session_id`
    - `chat_id` = `MY_PHONE + '@c.us'` (WhatsApp JID-style identifier)
    - `message`
    - plus `channel`, `sender`, `skip:false`
- **Inputs / outputs:**
  - **Input:** Slack webhook item
  - **Output:** A single standardized item for downstream nodes
- **Connections:**
  - Inputs from: `Slack Webhook`
  - Outputs to: `Valid?`
- **Edge cases / failures:**
  - **Slack `challenge` path:** The returned item `{challenge: ...}` does **not** set `skip`; the IF node expects `$json.skip === false`. With strict type validation, `skip` being `undefined` can prevent forwarding (good) but may also not respond to Slack as required unless the Webhook node is configured to use that response. If you intend to reply to Slack with the challenge, ensure the Webhook node response mode is compatible (see reproduction notes).
  - If `YOUR_SESSION_ID` / `YOUR_PHONE` are not replaced, MoltFlow will reject requests or messages go to the wrong recipient.
  - Messages containing special formatting may render differently in WhatsApp.
- **Version notes:** `typeVersion: 2` — standard Code node.

---

### Block 3 — Validation Gate (IF)
**Overview:** Ensures only valid, non-skipped items are sent to WhatsApp.  
**Nodes involved:** `Valid?`

#### Node: Valid?
- **Type / role:** `IF` node — conditional routing.
- **Key configuration choices:**
  - Condition checks a boolean field:
    - **Left:** `={{ $json.skip }}`
    - **Operation:** equals
    - **Right:** `false`
  - **Type validation:** strict
  - **Case sensitive:** true (not relevant for boolean but present in options)
- **Inputs / outputs:**
  - **Input:** Output from `Format Message`
  - **True branch:** to `Send to WhatsApp`
  - **False branch:** unused (no node connected)
- **Connections:**
  - Inputs from: `Format Message`
  - Outputs to: `Send to WhatsApp` on `true`
- **Edge cases / failures:**
  - If `$json.skip` is missing/undefined (e.g., Slack challenge response item), strict boolean equals may evaluate to false or fail type validation depending on n8n behavior/version; typically it won’t pass, so no send occurs (safe).
  - If upstream accidentally sets `"skip": "false"` as a string, it will fail strict boolean validation.
- **Version notes:** `typeVersion: 2`.

---

### Block 4 — WhatsApp Delivery (HTTP Request to MoltFlow)
**Overview:** Sends the formatted message to MoltFlow’s API endpoint using header-based authentication.  
**Nodes involved:** `Send to WhatsApp`

#### Node: Send to WhatsApp
- **Type / role:** `HTTP Request` node — outbound API call to MoltFlow.
- **Key configuration choices:**
  - **Method:** POST
  - **URL:** `https://apiv2.waiflow.app/api/v2/messages/send`
  - **Body type:** JSON
  - **Body expression:**
    - `={{ JSON.stringify({ session_id: $json.session_id, chat_id: $json.chat_id, message: $json.message }) }}`
    - Note: This explicitly stringifies JSON. n8n can also accept an object directly depending on settings; current config sends a JSON string as body while “JSON” body mode is enabled.
  - **Authentication:**
    - `genericCredentialType` → `httpHeaderAuth`
    - Credential name: **MoltFlow API Key**
    - Expected header per sticky note: `X-API-Key: <your key>`
- **Inputs / outputs:**
  - **Input:** Item containing `session_id`, `chat_id`, `message`
  - **Output:** API response from MoltFlow (status/body)
- **Connections:**
  - Inputs from: `Valid?` (true path)
  - Outputs to: `Log`
- **Edge cases / failures:**
  - 401/403 if API key missing/invalid or header name mismatched.
  - 400 if `session_id`, `chat_id`, or `message` invalid or missing.
  - Network timeouts / MoltFlow downtime.
  - If the API expects JSON object but receives a stringified JSON (depending on n8n request serialization), it could fail; validate by checking request payload in execution logs. If issues occur, set body to an object without `JSON.stringify`.
- **Version notes:** `typeVersion: 4.2` — HTTP Request node UI/behavior differs across versions.

---

### Block 5 — Logging / Final Status (Code)
**Overview:** Produces a concise “forwarded” status payload for downstream consumption or easier execution inspection.  
**Nodes involved:** `Log`

#### Node: Log
- **Type / role:** `Code` node — builds a small result object.
- **Key configuration choices:**
  - Mode: “Run Once for All Items”
  - Reads from another node directly:
    - `const data = $('Format Message').first().json;`
  - Returns:
    - `{ status: 'forwarded', channel: data.channel, from: data.sender }`
- **Inputs / outputs:**
  - **Input:** Response item from `Send to WhatsApp` (though it does not use it)
  - **Output:** One item with status/channel/from
- **Connections:**
  - Inputs from: `Send to WhatsApp`
  - Outputs: none
- **Edge cases / failures:**
  - If `Format Message` did not run or produced unexpected structure, `$('Format Message').first()` could be undefined (rare in this linear workflow, but possible if node renamed/deleted).
  - If the workflow is later extended to handle multiple items, `.first()` may hide other results.
- **Version notes:** `typeVersion: 2`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / context | — | — | ## Slack → WhatsApp Forwarder; Forward important Slack messages to WhatsApp so you never miss critical updates, powered by [MoltFlow](https://molt.waiflow.app).; **How it works:**; 1. Slack sends a webhook when a message is posted in a channel; 2. The message text and sender are extracted; 3. A formatted message is forwarded to your WhatsApp number; 4. Stay on top of Slack without checking the app |
| Sticky Note1 | Sticky Note | Setup instructions | — | — | ## Setup (5 min); 1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp; 2. Activate this workflow — copy the webhook URL; 3. In Slack → Apps → Incoming Webhooks (or use Slack API), configure an outgoing webhook to this n8n URL; 4. Set `YOUR_SESSION_ID` and `YOUR_PHONE` in the Format Message node; 5. Add MoltFlow API Key: Header Auth → `X-API-Key`; **Tip:** Filter by channel or keywords in the code node |
| Slack Webhook | Webhook | Receives Slack POST events | — | Format Message | ## Slack → WhatsApp Forwarder; Forward important Slack messages to WhatsApp so you never miss critical updates, powered by [MoltFlow](https://molt.waiflow.app).; **How it works:**; 1. Slack sends a webhook when a message is posted in a channel; 2. The message text and sender are extracted; 3. A formatted message is forwarded to your WhatsApp number; 4. Stay on top of Slack without checking the app |
| Format Message | Code | Normalize Slack payload + build WhatsApp message + skip empty | Slack Webhook | Valid? | ## Setup (5 min); 1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp; 2. Activate this workflow — copy the webhook URL; 3. In Slack → Apps → Incoming Webhooks (or use Slack API), configure an outgoing webhook to this n8n URL; 4. Set `YOUR_SESSION_ID` and `YOUR_PHONE` in the Format Message node; 5. Add MoltFlow API Key: Header Auth → `X-API-Key`; **Tip:** Filter by channel or keywords in the code node |
| Valid? | IF | Gate: only forward when `skip=false` | Format Message | Send to WhatsApp (true) | ## Setup (5 min); 1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp; 2. Activate this workflow — copy the webhook URL; 3. In Slack → Apps → Incoming Webhooks (or use Slack API), configure an outgoing webhook to this n8n URL; 4. Set `YOUR_SESSION_ID` and `YOUR_PHONE` in the Format Message node; 5. Add MoltFlow API Key: Header Auth → `X-API-Key`; **Tip:** Filter by channel or keywords in the code node |
| Send to WhatsApp | HTTP Request | Call MoltFlow API to send WhatsApp message | Valid? | Log | ## Setup (5 min); 1. Create a [MoltFlow account](https://molt.waiflow.app) and connect WhatsApp; 2. Activate this workflow — copy the webhook URL; 3. In Slack → Apps → Incoming Webhooks (or use Slack API), configure an outgoing webhook to this n8n URL; 4. Set `YOUR_SESSION_ID` and `YOUR_PHONE` in the Format Message node; 5. Add MoltFlow API Key: Header Auth → `X-API-Key`; **Tip:** Filter by channel or keywords in the code node |
| Log | Code | Return simplified forwarded status | Send to WhatsApp | — | ## Slack → WhatsApp Forwarder; Forward important Slack messages to WhatsApp so you never miss critical updates, powered by [MoltFlow](https://molt.waiflow.app).; **How it works:**; 1. Slack sends a webhook when a message is posted in a channel; 2. The message text and sender are extracted; 3. A formatted message is forwarded to your WhatsApp number; 4. Stay on top of Slack without checking the app |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Forward Slack Channel Messages to WhatsApp with MoltFlow**
   - (Optional) Add tags: `whatsapp`, `slack`, `notifications`, `moltflow`, `team`

2. **Add node: Webhook**
   - Node type: **Webhook**
   - Name: **Slack Webhook**
   - Set:
     - **HTTP Method:** POST
     - **Path:** `slack-forward`
   - Activate or test-run once to obtain the **Production/Test Webhook URL**.

3. **Configure Slack to call the webhook**
   - In Slack, set up an integration that POSTs message events to the n8n webhook URL.
   - If using Slack Events API:
     - Slack may send a `challenge` during verification; ensure n8n’s webhook response behavior returns the `challenge` correctly (see step 5 note).
   - If using a simpler outgoing webhook/custom app, ensure message text/user/channel are included in the payload.

4. **Add node: Code**
   - Node type: **Code**
   - Name: **Format Message**
   - Mode: **Run Once for All Items**
   - Paste logic equivalent to:
     - Define and set:
       - `SESSION_ID` = your MoltFlow WhatsApp session id
       - `MY_PHONE` = your WhatsApp number (in the format MoltFlow expects; this workflow appends `@c.us`)
     - Extract Slack payload from either `json.body` or `json`
     - If `challenge` exists, return `{challenge}`
     - Else derive `text`, `user`, `channel`, build formatted message
     - If no text, set `skip: true`
     - Output `session_id`, `chat_id`, `message`, `channel`, `sender`, `skip:false`
   - **Connect:** `Slack Webhook` → `Format Message`

5. **(Important) Ensure Slack challenge responses work**
   - If you use Slack Events API URL verification, configure the Webhook node response to return the `{ "challenge": "..." }` body produced by `Format Message`.
   - In many n8n setups, you’ll do this by setting the Webhook node “Response” options to respond with the last node output (or a dedicated Respond-to-Webhook node). If verification fails, add a **Respond to Webhook** node after `Format Message` specifically for `challenge` handling.

6. **Add node: IF**
   - Node type: **IF**
   - Name: **Valid?**
   - Condition:
     - Boolean → `={{ $json.skip }}`
     - “Equals” → `false`
     - Keep strict validation enabled (as in the provided workflow).
   - **Connect:** `Format Message` → `Valid?`

7. **Add node: HTTP Request**
   - Node type: **HTTP Request**
   - Name: **Send to WhatsApp**
   - Set:
     - **Method:** POST
     - **URL:** `https://apiv2.waiflow.app/api/v2/messages/send`
     - **Send Body:** enabled
     - **Body Content Type:** JSON
     - Body fields (either):
       - Use the expression to send:
         - `session_id: {{$json.session_id}}`
         - `chat_id: {{$json.chat_id}}`
         - `message: {{$json.message}}`
       - Or keep the original approach using a JSON-stringifying expression (but change if MoltFlow rejects it).
   - **Credentials (required):**
     - Create credential: **HTTP Header Auth**
     - Name it: **MoltFlow API Key**
     - Header name: `X-API-Key`
     - Value: your MoltFlow API key
   - **Connect:** `Valid?` (true output) → `Send to WhatsApp`

8. **Add node: Code (logging)**
   - Node type: **Code**
   - Name: **Log**
   - Mode: **Run Once for All Items**
   - Code logic:
     - Read `channel` and `sender` from `Format Message` node output
     - Return `{ status: 'forwarded', channel, from }`
   - **Connect:** `Send to WhatsApp` → `Log`

9. **Activate the workflow**
   - Use the Production Webhook URL in Slack.
   - Post a message in the configured channel and verify the WhatsApp delivery.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| MoltFlow service used to forward messages to WhatsApp | https://molt.waiflow.app |
| MoltFlow API endpoint used by this workflow | `https://apiv2.waiflow.app/api/v2/messages/send` |
| Setup reminders: set `YOUR_SESSION_ID`, `YOUR_PHONE`, and configure header auth `X-API-Key` | Included in workflow sticky note “Setup (5 min)” |
| Filtering suggestion: add channel/keyword filters in the Code node | Mentioned in sticky note: “Tip: Filter by channel or keywords in the code node” |