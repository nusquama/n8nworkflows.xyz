Categorise and label Gmail emails with GPT-4o and OpenAI

https://n8nworkflows.xyz/workflows/categorise-and-label-gmail-emails-with-gpt-4o-and-openai-13221


# Categorise and label Gmail emails with GPT-4o and OpenAI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automatically categorize new Gmail emails using OpenAI (GPT-4o), then (for non-urgent emails) remove them from the Inbox and apply a Gmail label to keep the inbox clean. Emails categorized as **“Action Required”** remain in the Inbox.

**Primary use cases**
- Auto-sorting inbound email into operational categories (admin, leads, newsletters, notifications, etc.)
- Keeping only “Action Required” items visible in the Inbox
- Consistent triage logic using an LLM prompt + examples

### 1.1 Input Reception (Gmail polling)
Gmail is polled every minute for new messages, which triggers the workflow.

### 1.2 AI Categorization (OpenAI)
The email subject and snippet are sent to GPT-4o with a strict JSON-only output contract. The model returns a single category.

### 1.3 Filtering (keep urgent in inbox)
A filter checks whether the category is **not** “Action Required”. Only non-urgent emails proceed to filing steps.

### 1.4 Filing (remove Inbox label + apply category label)
The workflow fetches matching Gmail messages, removes the `INBOX` label, then applies a configured label ID.

### 1.5 Setup/Utility (label discovery)
A helper node lists all Gmail labels so you can copy label IDs into the “Add to folder” node.

---

## 2. Block-by-Block Analysis

### Block 1 — Receive new email events (Gmail polling)
**Overview:** Watches Gmail for new emails and passes the email metadata (e.g., subject, snippet, sender) downstream.

**Nodes involved**
- **Gmail Trigger**

#### Node: Gmail Trigger
- **Type / role:** `n8n-nodes-base.gmailTrigger` — entry point; polls Gmail and emits new email items.
- **Configuration choices:**
  - Polling schedule: **every minute**
  - No additional Gmail filter rules configured (filters object empty).
- **Key expressions/variables used:** None inside the node.
- **Connections:**
  - **Output →** Categorize Email
- **Version-specific requirements:** Type version **1.3**
- **Edge cases / failures:**
  - Gmail OAuth/credential issues (expired token, missing scopes).
  - Polling frequency limits and Gmail API quotas.
  - Duplicate processing possible depending on trigger behavior and mailbox activity (consider deduping by message id if needed).

---

### Block 2 — Categorize email with OpenAI (GPT-4o)
**Overview:** Sends a structured instruction prompt plus the current email’s subject/snippet to GPT-4o and expects strict JSON output with one category label.

**Nodes involved**
- **Categorize Email**

#### Node: Categorize Email
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — LLM call for classification.
- **Configuration choices:**
  - **Model:** `gpt-4o`
  - **JSON output enabled:** `jsonOutput: true` (expects machine-readable JSON)
  - Prompt structure:
    - System: “helpful, intelligent administrative assistant”
    - Instruction: pick **ONE** label from a fixed list; apply decision rules; **return JSON only**: `{"category":"exact label name here"}`
    - Includes several example emails with expected assistant JSON responses (few-shot classification).
    - Final message uses workflow data:
      - `subject: {{ $json.Subject }}`
      - `text: {{ $json.snippet }}`
- **Key expressions/variables used:**
  - `{{ $json.Subject }}`
  - `{{ $json.snippet }}`
- **Connections:**
  - **Input ←** Gmail Trigger
  - **Output →** Not Worthwhile (Filter)
- **Version-specific requirements:** Type version **1.8**
- **Edge cases / failures:**
  - OpenAI credential missing/invalid; model not available on account.
  - JSON contract violations (model returns extra text, malformed JSON). Downstream filter expects `message.content.category`.
  - Misclassification risk due to short context (only snippet, not full body).
  - Prompt examples include lines starting with `=` in places; ensure n8n treats them as intended message text and not as expressions (in n8n, expressions are typically `{{ }}`; leading `=` can have special meaning in some fields—verify behavior after import).

---

### Block 3 — Filter out “Action Required” (only file non-urgent)
**Overview:** Checks the AI category. If it is *not* “Action Required”, the email is considered not urgent and will be filed away.

**Nodes involved**
- **Not Worthwhile**

#### Node: Not Worthwhile
- **Type / role:** `n8n-nodes-base.filter` — conditional routing gate.
- **Configuration choices:**
  - Condition (string):  
    - Left: `{{ $json.message.content.category }}`
    - Operation: `notEquals`
    - Right: `Action Required`
  - Meaning: pass through only if category is anything other than “Action Required”.
- **Key expressions/variables used:**
  - `{{ $json.message.content.category }}`
- **Connections:**
  - **Input ←** Categorize Email
  - **Output (true path) →** Get Email  
  - No explicit false-path node (so “Action Required” simply ends the run and stays in inbox).
- **Version-specific requirements:** Type version **2.2**
- **Edge cases / failures:**
  - If the LLM output path differs (e.g., `category` not present, or output structure differs), expression resolves to `undefined`, which may:
    - cause strict validation errors, or
    - evaluate in an unexpected way depending on filter settings.
  - Category spelling must exactly match “Action Required”.

---

### Block 4 — Locate the Gmail message(s) to modify
**Overview:** Uses the subject and sender from the trigger item to search Gmail and return matching message IDs for label operations.

**Nodes involved**
- **Get Email**

#### Node: Get Email
- **Type / role:** `n8n-nodes-base.gmail` — reads Gmail messages via search.
- **Configuration choices:**
  - Operation: **getAll**
  - Filters:
    - Gmail query `q`: `subject:"{{ $('Gmail Trigger').item.json.Subject }}"`
    - Sender: `{{ $('Gmail Trigger').item.json.From }}`
  - This attempts to find the message(s) matching the incoming trigger’s subject and sender.
- **Key expressions/variables used:**
  - `{{ $('Gmail Trigger').item.json.Subject }}`
  - `{{ $('Gmail Trigger').item.json.From }}`
- **Connections:**
  - **Input ←** Not Worthwhile
  - **Output →** Remove from Inbox
- **Version-specific requirements:** Type version **2.1**
- **Edge cases / failures:**
  - Subject+From may match multiple emails (threading, repeated newsletters). `getAll` can return multiple items and the workflow will label/remove inbox for each returned message.
  - Subject quoting/escaping: special characters or long subjects can reduce match accuracy.
  - If Gmail Trigger already provides a stable message ID, using it directly would be more reliable than searching.
  - Gmail API quota/auth issues.

---

### Block 5 — File the email (remove Inbox + apply label)
**Overview:** Removes the `INBOX` label from each matched message and then adds a configured category label (currently a placeholder) to move/organize the email.

**Nodes involved**
- **Remove from Inbox**
- **Add to folder**

#### Node: Remove from Inbox
- **Type / role:** `n8n-nodes-base.gmail` — modifies labels on a message.
- **Configuration choices:**
  - Operation: **removeLabels**
  - `messageId`: `{{ $json.id }}`
  - `labelIds`: `["INBOX"]`
- **Key expressions/variables used:**
  - `{{ $json.id }}` (coming from Get Email results)
- **Connections:**
  - **Input ←** Get Email
  - **Output →** Add to folder
- **Version-specific requirements:** Type version **2.1**
- **Edge cases / failures:**
  - Missing permissions (must have modify scopes).
  - If `$json.id` is missing because Get Email returned different structure, operation fails.
  - Removing `INBOX` effectively archives the message; user expectations should match.

#### Node: Add to folder
- **Type / role:** `n8n-nodes-base.gmail` — applies a Gmail label to a message.
- **Configuration choices:**
  - Operation: **addLabels**
  - `messageId`: `{{ $json.id }}`
  - `labelIds`: `YOUR_LABEL_ID` (placeholder)
- **Key expressions/variables used:**
  - `{{ $json.id }}`
- **Connections:**
  - **Input ←** Remove from Inbox
  - **Output →** (end)
- **Version-specific requirements:** Type version **2.1**
- **Edge cases / failures:**
  - Placeholder label ID must be replaced with a real label id (e.g., `Label_123456789`).
  - Current workflow does **not** dynamically map category → label ID. As written, it always applies the same label ID regardless of category (unless you manually change it per run).
  - If label doesn’t exist or ID is wrong, Gmail returns an error.

---

### Block 6 — Setup utility: list Gmail labels
**Overview:** Fetches all Gmail labels so you can copy the correct label IDs for configuration.

**Nodes involved**
- **Get many labels**

#### Node: Get many labels
- **Type / role:** `n8n-nodes-base.gmail` — lists label resources.
- **Configuration choices:**
  - Resource: **label**
  - `returnAll: true`
- **Key expressions/variables used:** None.
- **Connections:** None (standalone utility node)
- **Version-specific requirements:** Type version **2.2**
- **Edge cases / failures:**
  - Gmail credential/scopes required.
  - Large label lists are usually fine, but still subject to API limits.

---

### Block 7 — Documentation notes (sticky notes)
**Overview:** Embedded notes describing what the workflow does and how to configure labels and categories.

**Nodes involved**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

(Sticky note nodes do not affect execution.)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Entry point; poll Gmail for new emails | — | Categorize Email | ### 📥 Step 1: Receive & Analyze / New emails trigger the workflow automatically. AI reads the content and decides which category fits best. |
| Categorize Email | @n8n/n8n-nodes-langchain.openAi | Classify email into one category via GPT-4o | Gmail Trigger | Not Worthwhile | ### 📥 Step 1: Receive & Analyze / New emails trigger the workflow automatically. AI reads the content and decides which category fits best. |
| Not Worthwhile | n8n-nodes-base.filter | Route only non–Action Required emails to filing steps | Categorize Email | Get Email | ### 📋 Step 2: Filter & File / Emails that don't need immediate attention are automatically moved out of inbox and organized with the correct label. |
| Get Email | n8n-nodes-base.gmail | Search and fetch matching messages (IDs) | Not Worthwhile | Remove from Inbox | ### 📋 Step 2: Filter & File / Emails that don't need immediate attention are automatically moved out of inbox and organized with the correct label. |
| Remove from Inbox | n8n-nodes-base.gmail | Remove INBOX label (archive) | Get Email | Add to folder | ### 📋 Step 2: Filter & File / Emails that don't need immediate attention are automatically moved out of inbox and organized with the correct label. |
| Add to folder | n8n-nodes-base.gmail | Apply a Gmail label (configured by label ID) | Remove from Inbox | — | ### 📋 Step 2: Filter & File / Emails that don't need immediate attention are automatically moved out of inbox and organized with the correct label. |
| Get many labels | n8n-nodes-base.gmail | Utility: list all Gmail labels + IDs | — | — | ### ⚙️ Setup: Gmail Labels / **Before running this workflow:** / 1. Create labels in Gmail for your categories / 2. Click 'Get many labels' node → Execute / 3. Copy the label IDs from the output / 4. Open 'Add to folder' node / 5. Replace 'YOUR_LABEL_ID' with your actual label ID / **Tip:** The label ID looks like "Label_123456789" |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / instructions | — | — | ## Categorise and label emails automatically with AI using Gmail and OpenAI / This workflow automatically categorises incoming emails using AI and applies Gmail labels to organize your inbox. Emails that don't require action are automatically filed away. / ### How it works / 1. Gmail monitors your inbox for new emails every minute / 2. AI analyzes each email and assigns a category / 3. Non-urgent emails are removed from inbox and labeled / 4. Action-required emails stay in your inbox / ### Setup steps / 1. Connect Gmail account (click Gmail nodes to authenticate) / 2. Add OpenAI API key (click Categorize Email node) / 3. Create Gmail labels for your categories / 4. Run 'Get many labels' node to find label IDs / 5. Update 'Add to folder' node with your label IDs / 6. Customize categories in the AI prompt (optional) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation / instructions | — | — | ### 📥 Step 1: Receive & Analyze / New emails trigger the workflow automatically. AI reads the content and decides which category fits best. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation / instructions | — | — | ### 📋 Step 2: Filter & File / Emails that don't need immediate attention are automatically moved out of inbox and organized with the correct label. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation / instructions | — | — | ### ⚙️ Setup: Gmail Labels / **Before running this workflow:** / 1. Create labels in Gmail for your categories / 2. Click 'Get many labels' node → Execute / 3. Copy the label IDs from the output / 4. Open 'Add to folder' node / 5. Replace 'YOUR_LABEL_ID' with your actual label ID / **Tip:** The label ID looks like "Label_123456789" |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation / instructions | — | — | ### ⚙️ Optional: Customize Categories / **Want different email categories?** / Open 'Categorize Email' node → Edit the system message / **You can:** / - Add/remove categories from the list / - Change the classification rules / - Add more example emails to improve accuracy / **Tip:** The AI learns from examples. Add 2-3 sample emails for each category you want. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Gmail Trigger**
   - Node type: **Gmail Trigger**
   - Polling: set **Every Minute**
   - Credentials: connect your **Gmail OAuth2** account (ensure it has permission to read mail).
3. **Add node: OpenAI (LangChain) — “Categorize Email”**
   - Node type: **OpenAI (LangChain)**
   - Model: **gpt-4o**
   - Enable **JSON output** (so downstream can parse category reliably).
   - Messages:
     - System: “You are my helpful, intelligent administrative assistant.”
     - User instruction: include the category list and the rule set; require **JSON only** output like `{"category":"..."}`.
     - (Optional but recommended) Add a few example emails + expected JSON category outputs.
     - Final user message: include email data from the trigger:
       - `subject: {{ $json.Subject }}`
       - `text: {{ $json.snippet }}`
   - Credentials: configure **OpenAI API key** in n8n credentials for this node.
4. **Connect:** Gmail Trigger → Categorize Email
5. **Add node: Filter — “Not Worthwhile”**
   - Node type: **Filter**
   - Condition:
     - Left value: `{{ $json.message.content.category }}`
     - Operator: **not equals**
     - Right value: `Action Required`
   - This ensures only non-urgent emails continue.
6. **Connect:** Categorize Email → Not Worthwhile
7. **Add node: Gmail — “Get Email”**
   - Node type: **Gmail**
   - Operation: **Get All**
   - Filters:
     - Query `q`: `subject:"{{ $('Gmail Trigger').item.json.Subject }}"`
     - Sender: `{{ $('Gmail Trigger').item.json.From }}`
   - Credentials: same Gmail credential (must allow reading messages).
8. **Connect:** Not Worthwhile → Get Email
9. **Add node: Gmail — “Remove from Inbox”**
   - Node type: **Gmail**
   - Operation: **Remove Labels**
   - Message ID: `{{ $json.id }}`
   - Labels to remove: `INBOX`
10. **Connect:** Get Email → Remove from Inbox
11. **Add node: Gmail — “Add to folder”**
   - Node type: **Gmail**
   - Operation: **Add Labels**
   - Message ID: `{{ $json.id }}`
   - Label IDs: set to a real label id (replace placeholder):
     - Example format: `Label_123456789`
12. **Connect:** Remove from Inbox → Add to folder
13. **Add helper node (optional but recommended): Gmail — “Get many labels”**
   - Node type: **Gmail**
   - Resource: **Label**
   - Return all: **true**
   - Execute it once to copy label IDs for step 11.
14. **Create Gmail labels** in Gmail matching your category set (manually in Gmail UI).
15. **Important limitation to address (if you want true category-based labeling):**
   - As built, “Add to folder” uses **one static label ID**.
   - To label per category, add logic (e.g., a Switch/IF or mapping table) to translate `category` → `labelId`, then pass that into the Gmail “Add Labels” node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically categorises incoming emails using AI and applies Gmail labels to organize your inbox. Emails that don't require action are automatically filed away. | Sticky note (workflow description) |
| Setup steps include connecting Gmail credentials, adding OpenAI API key, creating Gmail labels, fetching label IDs, and updating the label ID in “Add to folder”. | Sticky note (setup) |
| The label ID looks like `"Label_123456789"` and must replace `YOUR_LABEL_ID`. | Sticky note (Gmail label setup) |
| Categories and rules can be customized in the “Categorize Email” node; adding 2–3 examples per category improves accuracy. | Sticky note (prompt customization) |