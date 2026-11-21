Classify and label new Gmail emails with OpenAI GPT-4.1 mini

https://n8nworkflows.xyz/workflows/classify-and-label-new-gmail-emails-with-openai-gpt-4-1-mini-20988


# Classify and label new Gmail emails with OpenAI GPT-4.1 mini

### 1. Workflow Overview

This workflow automates the sorting and organization of incoming Gmail messages by combining scheduled event triggers, configuration variables, generative AI categorization, and Gmail label management. Its primary purpose is to help small businesses, freelancers, and support teams process inbound emails without manual intervention.

The logic is organized into three sequential functional blocks:
- **1.1 Input Reception & Normalization:** Polls Gmail for newly received messages every minute, establishes global execution settings (such as business identity, model identifiers, and character limits), and structures the raw email payloads into a clean schema.
- **1.2 AI Processing & Parsing:** Sends normalized email content to an OpenAI model configured with a strict JSON schema and role-based triage instructions, then securely parses and validates the generated response.
- **1.3 Label Resolution & Application:** Retrieves all available workspace labels from Gmail, maps the categorized output to specific destination label IDs, verifies label existence, and updates the targeted email messages.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Normalization
- **Overview:** This block establishes the entry point for new communications, configures operational variables, and sanitizes email content to prepare it for downstream AI analysis.
- **Nodes Involved:** `Gmail Trigger`, `Settings`, `Prepare Email`

- **Node Details:**
  - **Gmail Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.gmailTrigger` (v1.3) — Polls the connected Gmail account at regular intervals to capture incoming messages.
    - *Configuration Choices:* Operates with simplified output set to `false`, skips spam and trash (`includeSpamTrash: false`), disables attachment downloading to conserve bandwidth, and sets polling frequency to every minute.
    - *Key Expressions or Variables:* None (triggers execution).
    - *Input and Output Connections:* Input: None (Trigger); Output: `Settings`.
    - *Version-specific Requirements:* Requires active OAuth2 credentials authenticated against a Google account with Gmail read permissions.
    - *Edge Cases or Potential Failure types:* Token expiration, rate limiting by Google APIs, or intermittent network dropouts.

  - **Settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) — Injects static configuration parameters into the execution context.
    - *Configuration Choices:* Defines fixed key-value assignments for downstream logic.
    - *Key Expressions or Variables:* 
      - `business_name`: `"Your Business Name"` (String)
      - `openai_model`: `"gpt-4.1-mini"` (String)
      - `label_prefix`: `"AI/"` (String)
      - `add_priority_labels`: `false` (Boolean)
      - `max_body_chars`: `6000` (Number)
      - `classification_notes`: `"(optional) e.g. Emails from @bigclient.com are always urgent."` (String)
    - *Input and Output Connections:* Input: `Gmail Trigger`; Output: `Prepare Email`.
    - *Version-specific Requirements:* None.
    - *Edge Cases or Potential Failure types:* None (purely internal variable assignment).

  - **Prepare Email**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2.0) — Executes custom JavaScript to normalize email payloads from the trigger node and trim the body length.
    - *Configuration Choices:* Extracts sender addresses, sender names, subjects, truncated body text (bounded by `max_body_chars`), and timestamps.
    - *Key Expressions or Variables:* References `$('[Settings]').first().json.max_body_chars` and evaluates incoming payloads from `$('Gmail Trigger').all()`.
    - *Input and Output Connections:* Input: `Settings`; Output: `Classify Email`.
    - *Version-specific Requirements:* JavaScript environment enabled.
    - *Edge Cases or Potential Failure types:* Missing object keys in non-standard email headers default to empty strings or fallback values like `(no subject)`.

---

#### Block 1.2: AI Processing & Parsing
- **Overview:** This block leverages an LLM to evaluate the semantic content of each prepared email against specific business rules, returning a strict JSON response that is subsequently parsed and normalized.
- **Nodes Involved:** `Classify Email`, `Parse Classification`

- **Node Details:**
  - **Classify Email**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (v2.3) — Interacts with the OpenAI Responses API to categorize and summarize email content.
    - *Configuration Choices:* Uses the dynamic model identifier from settings (`openai_model`), enforces a maximum of 400 tokens, applies a strict JSON schema (`email_triage`) ensuring output properties match predefined categories and priorities, and runs with `simplify: true`.
    - *Key Expressions or Variables:* 
      - System Prompt references `{{ $('Settings').first().json.business_name }}` and `{{ $('Settings').first().json.classification_notes }}`.
      - User Prompt evaluates `{{ $json.from }}`, `{{ $json.subject }}`, and `{{ $json.body }}`.
    - *Input and Output Connections:* Input: `Prepare Email`; Output: `Parse Classification`.
    - *Version-specific Requirements:* Valid OpenAI credential with API key access.
    - *Edge Cases or Potential Failure types:* API rate limits, upstream OpenAI service outages, invalid response formats, or model refusal triggers.

  - **Parse Classification**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2.0) — Parses the raw model response and enforces default fallbacks if the output is malformed or invalid.
    - *Configuration Choices:* Validates categories against an explicit array (`urgent`, `sales_lead`, `support`, `billing`, `newsletter_spam`, `other`) and priorities (`high`, `medium`, `low`).
    - *Key Expressions or Variables:* Pulls data from `$('Prepare Email').all()` and parses the raw text content from the language model response object.
    - *Input and Output Connections:* Input: `Classify Email`; Output: `Get Gmail Labels`.
    - *Version-specific Requirements:* JavaScript environment enabled.
    - *Edge Cases or Potential Failure types:* Unparseable JSON strings trigger fallback handling, assigning the category to `'other'` and priority to `'medium'` to prevent workflow halts.

---

#### Block 1.3: Label Resolution & Application
- **Overview:** This block queries available Gmail labels, maps the classification outputs to their corresponding label identifiers, validates that the destination labels exist within the account, and applies them to the original messages.
- **Nodes Involved:** `Get Gmail Labels`, `Resolve Labels`, `Has Label?`, `Apply Gmail Label`

- **Node Details:**
  - **Get Gmail Labels**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (v2.2) — Fetches all user labels configured in the targeted Gmail account.
    - *Configuration Choices:* Resource set to `label`, operation set to `getAll`, with `returnAll` enabled and `executeOnce` set to `true`.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: `Parse Classification`; Output: `Resolve Labels`.
    - *Version-specific Requirements:* Gmail OAuth2 credentials with label read permissions.
    - *Edge Cases or Potential Failure types:* API connection drops or insufficient scope permissions.

  - **Resolve Labels**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2.0) — Cross-references the parsed email classification categories and optional priority attributes with the retrieved Gmail label inventory.
    - *Configuration Choices:* Generates target label names using the configured prefix (`AI/`) combined with mapped category names (`Urgent`, `Sales Lead`, `Support`, `Billing`, `Newsletter-Spam`, `Other`).
    - *Key Expressions or Variables:* Evaluates settings via `$('Settings').first().json` and label data from `$('Get Gmail Labels').all()`.
    - *Input and Output Connections:* Input: `Get Gmail Labels`; Output: `Has Label?`.
    - *Version-specific Requirements:* JavaScript environment enabled.
    - *Edge Cases or Potential Failure types:* Missing target labels set `labelMissing` to `true`, allowing downstream filtering.

  - **Has Label?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2.2) — Evaluates whether the resolved destination label exists in the Gmail account.
    - *Configuration Choices:* Condition checks if `$json.labelMissing` equals `false` using loose type validation.
    - *Key Expressions or Variables:* Evaluates `{{ $json.labelMissing }}`.
    - *Input and Output Connections:* Input: `Resolve Labels`; Outputs: True branch connects to `Apply Gmail Label`; False branch terminates execution path.
    - *Version-specific Requirements:* None.
    - *Edge Cases or Potential Failure types:* None.

  - **Apply Gmail Label**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (v2.2) — Applies the resolved label identifiers to the specified Gmail message.
    - *Configuration Choices:* Resource set to `message`, operation set to `addLabels`.
    - *Key Expressions or Variables:* 
      - `labelIds`: `={{ $json.labelIds }}`
      - `messageId`: `={{ $json.messageId }}`
    - *Input and Output Connections:* Input: `Has Label?` (True branch); Output: None (Terminal node).
    - *Version-specific Requirements:* Gmail OAuth2 credentials with message modification permissions.
    - *Edge Cases or Potential Failure types:* Target message deleted prior to labeling, or insufficient write scopes on the OAuth2 token.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Gmail Trigger** | `n8n-nodes-base.gmailTrigger` | Polls for new messages every minute | None (Trigger) | Settings | # Classify and label new Gmail emails with OpenAI<br><br>## Who is this for<br>Small business owners, freelancers and support teams whose Gmail inbox mixes customer requests, sales leads, invoices and newsletters.<br><br>## How it works<br>1. **Gmail Trigger** checks for new emails every minute.<br>2. **Settings** holds your business name, model and optional sorting rules.<br>3. **OpenAI** classifies each email as Urgent, Sales Lead, Support, Billing, Newsletter/Spam or Other, with a high/medium/low priority and a one-line summary (strict JSON schema).<br>4. The workflow looks up your Gmail labels and adds the matching one, e.g. `AI/Sales Lead`. Anything the AI can't classify is filed as Other.<br><br>## How to set up<br>1. Attach your **Gmail OAuth2** credential to the Gmail nodes and your **OpenAI** credential to *Classify Email*.<br>2. In Gmail, create the labels `AI/Urgent`, `AI/Sales Lead`, `AI/Support`, `AI/Billing`, `AI/Newsletter-Spam`, `AI/Other`.<br>3. Open **Settings**: set `business_name`, `openai_model` (default `gpt-4.1-mini`) and optional `classification_notes`, e.g. "emails from @bigclient.com are urgent".<br>4. Send yourself a test email, click **Execute workflow**, check the label, then activate.<br><br>## Requirements<br>- n8n (Cloud or self-hosted)<br>- Gmail account<br>- OpenAI API key (usage billed by OpenAI)<br><br>## How to customize<br>- Change category definitions in the *Classify Email* system prompt.<br>- Rename labels in the `NAMES` map in *Resolve Labels*.<br>- Turn on `add_priority_labels` to also add `AI/Priority-High/Medium/Low`.<br>- Swap the OpenAI node for another provider's chat model with an LLM Chain.<br><br>## 1. Fetch and prepare<br>Poll Gmail for new emails, load your settings, and trim each email to the fields the AI needs. |
| **Settings** | `n8n-nodes-base.set` | Injects global workflow configuration parameters | Gmail Trigger | Prepare Email | # Classify and label new Gmail emails with OpenAI<br><br>## Who is this for<br>Small business owners, freelancers and support teams whose Gmail inbox mixes customer requests, sales leads, invoices and newsletters.<br><br>## How it works<br>1. **Gmail Trigger** checks for new emails every minute.<br>2. **Settings** holds your business name, model and optional sorting rules.<br>3. **OpenAI** classifies each email as Urgent, Sales Lead, Support, Billing, Newsletter/Spam or Other, with a high/medium/low priority and a one-line summary (strict JSON schema).<br>4. The workflow looks up your Gmail labels and adds the matching one, e.g. `AI/Sales Lead`. Anything the AI can't classify is filed as Other.<br><br>## How to set up<br>1. Attach your **Gmail OAuth2** credential to the Gmail nodes and your **OpenAI** credential to *Classify Email*.<br>2. In Gmail, create the labels `AI/Urgent`, `AI/Sales Lead`, `AI/Support`, `AI/Billing`, `AI/Newsletter-Spam`, `AI/Other`.<br>3. Open **Settings**: set `business_name`, `openai_model` (default `gpt-4.1-mini`) and optional `classification_notes`, e.g. "emails from @bigclient.com are urgent".<br>4. Send yourself a test email, click **Execute workflow**, check the label, then activate.<br><br>## Requirements<br>- n8n (Cloud or self-hosted)<br>- Gmail account<br>- OpenAI API key (usage billed by OpenAI)<br><br>## How to customize<br>- Change category definitions in the *Classify Email* system prompt.<br>- Rename labels in the `NAMES` map in *Resolve Labels*.<br>- Turn on `add_priority_labels` to also add `AI/Priority-High/Medium/Low`.<br>- Swap the OpenAI node for another provider's chat model with an LLM Chain.<br><br>## 1. Fetch and prepare<br>Poll Gmail for new emails, load your settings, and trim each email to the fields the AI needs. |
| **Prepare Email** | `n8n-nodes-base.code` | Normalizes and truncates email payload fields | Settings | Classify Email | # Classify and label new Gmail emails with OpenAI<br><br>## Who is this for<br>Small business owners, freelancers and support teams whose Gmail inbox mixes customer requests, sales leads, invoices and newsletters.<br><br>## How it works<br>1. **Gmail Trigger** checks for new emails every minute.<br>2. **Settings** holds your business name, model and optional sorting rules.<br>3. **OpenAI** classifies each email as Urgent, Sales Lead, Support, Billing, Newsletter/Spam or Other, with a high/medium/low priority and a one-line summary (strict JSON schema).<br>4. The workflow looks up your Gmail labels and adds the matching one, e.g. `AI/Sales Lead`. Anything the AI can't classify is filed as Other.<br><br>## How to set up<br>1. Attach your **Gmail OAuth2** credential to the Gmail nodes and your **OpenAI** credential to *Classify Email*.<br>2. In Gmail, create the labels `AI/Urgent`, `AI/Sales Lead`, `AI/Support`, `AI/Billing`, `AI/Newsletter-Spam`, `AI/Other`.<br>3. Open **Settings**: set `business_name`, `openai_model` (default `gpt-4.1-mini`) and optional `classification_notes`, e.g. "emails from @bigclient.com are urgent".<br>4. Send yourself a test email, click **Execute workflow**, check the label, then activate.<br><br>## Requirements<br>- n8n (Cloud or self-hosted)<br>- Gmail account<br>- OpenAI API key (usage billed by OpenAI)<br><br>## How to customize<br>- Change category definitions in the *Classify Email* system prompt.<br>- Rename labels in the `NAMES` map in *Resolve Labels*.<br>- Turn on `add_priority_labels` to also add `AI/Priority-High/Medium/Low`.<br>- Swap the OpenAI node for another provider's chat model with an LLM Chain.<br><br>## 1. Fetch and prepare<br>Poll Gmail for new emails, load your settings, and trim each email to the fields the AI needs. |
| **Classify Email** | `@n8n/n8n-nodes-langchain.openAi` | Categorizes email content via OpenAI API using strict JSON schema | Prepare Email | Parse Classification | # Classify and label new Gmail emails with OpenAI<br><br>## Who is this for<br>Small business owners, freelancers and support teams whose Gmail inbox mixes customer requests, sales leads, invoices and newsletters.<br><br>## How it works<br>1. **Gmail Trigger** checks for new emails every minute.<br>2. **Settings** holds your business name, model and optional sorting rules.<br>3. **OpenAI** classifies each email as Urgent, Sales Lead, Support, Billing, Newsletter/Spam or Other, with a high/medium/low priority and a one-line summary (strict JSON schema).<br>4. The workflow looks up your Gmail labels and adds the matching one, e.g. `AI/Sales Lead`. Anything the AI can't classify is filed as Other.<br><br>## How to set up<br>1. Attach your **Gmail OAuth2** credential to the Gmail nodes and your **OpenAI** credential to *Classify Email*.<br>2. In Gmail, create the labels `AI/Urgent`, `AI/Sales Lead`, `AI/Support`, `AI/Billing`, `AI/Newsletter-Spam`, `AI/Other`.<br>3. Open **Settings**: set `business_name`, `openai_model` (default `gpt-4.1-mini`) and optional `classification_notes`, e.g. "emails from @bigclient.com are urgent".<br>4. Send yourself a test email, click **Execute workflow**, check the label, then activate.<br><br>## Requirements<br>- n8n (Cloud or self-hosted)<br>- Gmail account<br>- OpenAI API key (usage billed by OpenAI)<br><br>## How to customize<br>- Change category definitions in the *Classify Email* system prompt.<br>- Rename labels in the `NAMES` map in *Resolve Labels*.<br>- Turn on `add_priority_labels` to also add `AI/Priority-High/Medium/Low`.<br>- Swap the OpenAI node for another provider's chat model with an LLM Chain.<br><br>## 2. Classify with AI<br>OpenAI returns category, priority and a summary in a strict JSON format. Invalid answers fall back to "other". |
| **Parse Classification** | `n8n-nodes-base.code` | Parses and validates the model's JSON response with fallback handling | Classify Email | Get Gmail Labels | # Classify and label new Gmail emails with OpenAI<br><br>## Who is this for<br>Small business owners, freelancers and support teams whose Gmail inbox mixes customer requests, sales leads, invoices and newsletters.<br><br>## How it works<br>1. **Gmail Trigger** checks for new emails every minute.<br>2. **Settings** holds your business name, model and optional sorting rules.<br>3. **OpenAI** classifies each email as Urgent, Sales Lead, Support, Billing, Newsletter/Spam or Other, with a high/medium/low priority and a one-line summary (strict JSON schema).<br>4. The workflow looks up your Gmail labels and adds the matching one, e.g. `AI/Sales Lead`. Anything the AI can't classify is filed as Other.<br><br>## How to set up<br>1. Attach your **Gmail OAuth2** credential to the Gmail nodes and your **OpenAI** credential to *Classify Email*.<br>2. In Gmail, create the labels `AI/Urgent`, `AI/Sales Lead`, `AI/Support`, `AI/Billing`, `AI/Newsletter-Spam`, `AI/Other`.<br>3. Open **Settings**: set `business_name`, `openai_model` (default `gpt-4.1-mini`) and optional `classification_notes`, e.g. "emails from @bigclient.com are urgent".<br>4. Send yourself a test email, click **Execute workflow**, check the label, then activate.<br><br>## Requirements<br>- n8n (Cloud or self-hosted)<br>- Gmail account<br>- OpenAI API key (usage billed by OpenAI)<br><br>## How to customize<br>- Change category definitions in the *Classify Email* system prompt.<br>- Rename labels in the `NAMES` map in *Resolve Labels*.<br>- Turn on `add_priority_labels` to also add `AI/Priority-High/Medium/Low`.<br>- Swap the OpenAI node for another provider's chat model with an LLM Chain.<br><br>## 2. Classify with AI<br>OpenAI returns category, priority and a summary in a strict JSON format. Invalid answers fall back to "other". |
| **Get Gmail Labels** | `n8n-nodes-base.gmail` | Fetches all existing user labels from the Gmail account | Parse Classification | Resolve Labels | # Classify and label new Gmail emails with OpenAI<br><br>## Who is this for<br>Small business owners, freelancers and support teams whose Gmail inbox mixes customer requests, sales leads, invoices and newsletters.<br><br>## How it works<br>1. **Gmail Trigger** checks for new emails every minute.<br>2. **Settings** holds your business name, model and optional sorting rules.<br>3. **OpenAI** classifies each email as Urgent, Sales Lead, Support, Billing, Newsletter/Spam or Other, with a high/medium/low priority and a one-line summary (strict JSON schema).<br>4. The workflow looks up your Gmail labels and adds the matching one, e.g. `AI/Sales Lead`. Anything the AI can't classify is filed as Other.<br><br>## How to set up<br>1. Attach your **Gmail OAuth2** credential to the Gmail nodes and your **OpenAI** credential to *Classify Email*.<br>2. In Gmail, create the labels `AI/Urgent`, `AI/Sales Lead`, `AI/Support`, `AI/Billing`, `AI/Newsletter-Spam`, `AI/Other`.<br>3. Open **Settings**: set `business_name`, `openai_model` (default `gpt-4.1-mini`) and optional `classification_notes`, e.g. "emails from @bigclient.com are urgent".<br>4. Send yourself a test email, click **Execute workflow**, check the label, then activate.<br><br>## Requirements<br>- n8n (Cloud or self-hosted)<br>- Gmail account<br>- OpenAI API key (usage billed by OpenAI)<br><br>## How to customize<br>- Change category definitions in the *Classify Email* system prompt.<br>- Rename labels in the `NAMES` map in *Resolve Labels*.<br>- Turn on `add_priority_labels` to also add `AI/Priority-High/Medium/Low`.<br>- Swap the OpenAI node for another provider's chat model with an LLM Chain.<br><br>## 3. Apply Gmail label<br>Look up your `AI/...` labels and add the matching one. Emails whose label doesn't exist in Gmail are skipped. |
| **Resolve Labels** | `n8n-nodes-base.code` | Maps email categories to corresponding Gmail label IDs | Get Gmail Labels | Has Label? | # Classify and label new Gmail emails with OpenAI<br><br>## Who is this for<br>Small business owners, freelancers and support teams whose Gmail inbox mixes customer requests, sales leads, invoices and newsletters.<br><br>## How it works<br>1. **Gmail Trigger** checks for new emails every minute.<br>2. **Settings** holds your business name, model and optional sorting rules.<br>3. **OpenAI** classifies each email as Urgent, Sales Lead, Support, Billing, Newsletter/Spam or Other, with a high/medium/low priority and a one-line summary (strict JSON schema).<br>4. The workflow looks up your Gmail labels and adds the matching one, e.g. `AI/Sales Lead`. Anything the AI can't classify is filed as Other.<br><br>## How to set up<br>1. Attach your **Gmail OAuth2** credential to the Gmail nodes and your **OpenAI** credential to *Classify Email*.<br>2. In Gmail, create the labels `AI/Urgent`, `AI/Sales Lead`, `AI/Support`, `AI/Billing`, `AI/Newsletter-Spam`, `AI/Other`.<br>3. Open **Settings**: set `business_name`, `openai_model` (default `gpt-4.1-mini`) and optional `classification_notes`, e.g. "emails from @bigclient.com are urgent".<br>4. Send yourself a test email, click **Execute workflow**, check the label, then activate.<br><br>## Requirements<br>- n8n (Cloud or self-hosted)<br>- Gmail account<br>- OpenAI API key (usage billed by OpenAI)<br><br>## How to customize<br>- Change category definitions in the *Classify Email* system prompt.<br>- Rename labels in the `NAMES` map in *Resolve Labels*.<br>- Turn on `add_priority_labels` to also add `AI/Priority-High/Medium/Low`.<br>- Swap the OpenAI node for another provider's chat model with an LLM Chain.<br><br>## 3. Apply Gmail label<br>Look up your `AI/...` labels and add the matching one. Emails whose label doesn't exist in Gmail are skipped. |
| **Has Label?** | `n8n-nodes-base.if` | Verifies whether the target label exists in Gmail | Resolve Labels | Apply Gmail Label (True), None (False) | # Classify and label new Gmail emails with OpenAI<br><br>## Who is this for<br>Small business owners, freelancers and support teams whose Gmail inbox mixes customer requests, sales leads, invoices and newsletters.<br><br>## How it works<br>1. **Gmail Trigger** checks for new emails every minute.<br>2. **Settings** holds your business name, model and optional sorting rules.<br>3. **OpenAI** classifies each email as Urgent, Sales Lead, Support, Billing, Newsletter/Spam or Other, with a high/medium/low priority and a one-line summary (strict JSON schema).<br>4. The workflow looks up your Gmail labels and adds the matching one, e.g. `AI/Sales Lead`. Anything the AI can't classify is filed as Other.<br><br>## How to set up<br>1. Attach your **Gmail OAuth2** credential to the Gmail nodes and your **OpenAI** credential to *Classify Email*.<br>2. In Gmail, create the labels `AI/Urgent`, `AI/Sales Lead`, `AI/Support`, `AI/Billing`, `AI/Newsletter-Spam`, `AI/Other`.<br>3. Open **Settings**: set `business_name`, `openai_model` (default `gpt-4.1-mini`) and optional `classification_notes`, e.g. "emails from @bigclient.com are urgent".<br>4. Send yourself a test email, click **Execute workflow**, check the label, then activate.<br><br>## Requirements<br>- n8n (Cloud or self-hosted)<br>- Gmail account<br>- OpenAI API key (usage billed by OpenAI)<br><br>## How to customize<br>- Change category definitions in the *Classify Email* system prompt.<br>- Rename labels in the `NAMES` map in *Resolve Labels*.<br>- Turn on `add_priority_labels` to also add `AI/Priority-High/Medium/Low`.<br>- Swap the OpenAI node for another provider's chat model with an LLM Chain.<br><br>## 3. Apply Gmail label<br>Look up your `AI/...` labels and add the matching one. Emails whose label doesn't exist in Gmail are skipped. |
| **Apply Gmail Label** | `n8n-nodes-base.gmail` | Adds resolved label IDs to the matching Gmail message | Has Label? | None (Terminal) | # Classify and label new Gmail emails with OpenAI<br><br>## Who is this for<br>Small business owners, freelancers and support teams whose Gmail inbox mixes customer requests, sales leads, invoices and newsletters.<br><br>## How it works<br>1. **Gmail Trigger** checks for new emails every minute.<br>2. **Settings** holds your business name, model and optional sorting rules.<br>3. **OpenAI** classifies each email as Urgent, Sales Lead, Support, Billing, Newsletter/Spam or Other, with a high/medium/low priority and a one-line summary (strict JSON schema).<br>4. The workflow looks up your Gmail labels and adds the matching one, e.g. `AI/Sales Lead`. Anything the AI can't classify is filed as Other.<br><br>## How to set up<br>1. Attach your **Gmail OAuth2** credential to the Gmail nodes and your **OpenAI** credential to *Classify Email*.<br>2. In Gmail, create the labels `AI/Urgent`, `AI/Sales Lead`, `AI/Support`, `AI/Billing`, `AI/Newsletter-Spam`, `AI/Other`.<br>3. Open **Settings**: set `business_name`, `openai_model` (default `gpt-4.1-mini`) and optional `classification_notes`, e.g. "emails from @bigclient.com are urgent".<br>4. Send yourself a test email, click **Execute workflow**, check the label, then activate.<br><br>## Requirements<br>- n8n (Cloud or self-hosted)<br>- Gmail account<br>- OpenAI API key (usage billed by OpenAI)<br><br>## How to customize<br>- Change category definitions in the *Classify Email* system prompt.<br>- Rename labels in the `NAMES` map in *Resolve Labels*.<br>- Turn on `add_priority_labels` to also add `AI/Priority-High/Medium/Low`.<br>- Swap the OpenAI node for another provider's chat model with an LLM Chain.<br><br>## 3. Apply Gmail label<br>Look up your `AI/...` labels and add the matching one. Emails whose label doesn't exist in Gmail are skipped. |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to construct the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Gmail Trigger** node (`n8n-nodes-base.gmailTrigger`).
   - Configure parameters: Set `simple` to `false`, disable spam and trash inclusion (`includeSpamTrash: false`), disable attachment downloads, and set polling frequency (`pollTimes`) to run every minute.
   - Configure a valid **Gmail OAuth2** credential.

2. **Create the Settings Node:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Settings`.
   - Connect the output of **Gmail Trigger** to the input of **Settings**.
   - Configure assignments:
     - `business_name` (String): `"Your Business Name"`
     - `openai_model` (String): `"gpt-4.1-mini"`
     - `label_prefix` (String): `"AI/"`
     - `add_priority_labels` (Boolean): `false`
     - `max_body_chars` (Number): `6000`
     - `classification_notes` (String): `"(optional) e.g. Emails from @bigclient.com are always urgent."`

3. **Create the Email Preparation Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Prepare Email`.
   - Connect the output of **Settings** to the input of **Prepare Email**.
   - Set mode to JavaScript and insert the following code snippet:
     ```javascript
     const s = $('Settings').first().json;
     const max = Number(s.max_body_chars) || 6000;
     return $('Gmail Trigger').all().map(item => {
       const m = item.json;
       const fromAddr = m.from?.value?.[0]?.address || m.From || m.from?.text || '';
       const fromName = m.from?.value?.[0]?.name || '';
       const body = (m.text || m.snippet || '').toString().replace(/\n{3,}/g, '\n\n').slice(0, max);
       return { json: {
         messageId: m.id, threadId: m.threadId,
         from: fromAddr, fromName,
         subject: m.subject || m.Subject || '(no subject)',
         body, date: m.date || new Date().toISOString(),
       }};
     });
     ```

4. **Create the AI Classification Node:**
   - Add an **OpenAI** node (`@n8n/n8n-nodes-langchain.openAi`) named `Classify Email`.
   - Connect the output of **Prepare Email** to the input of **Classify Email**.
   - Configure parameters:
     - Resource: `text`
     - Operation: `response`
     - Model ID: `={{ $('Settings').first().json.openai_model }}` (Mode: ID)
     - Options -> Max Tokens: `400`
     - Text Format -> Type: `json_schema`, Schema name: `email_triage`, Strict: `true`, Schema JSON:
       ```json
       {
         "type": "object",
         "additionalProperties": false,
         "required": ["category", "priority", "summary", "needs_reply", "reason"],
         "properties": {
           "category": { "type": "string", "enum": ["urgent", "sales_lead", "support", "billing", "newsletter_spam", "other"] },
           "priority": { "type": "string", "enum": ["high", "medium", "low"] },
           "summary": { "type": "string" },
           "needs_reply": { "type": "boolean" },
           "reason": { "type": "string" }
         }
       }
       ```
     - System Message: `=You are an email triage assistant for a small business called "{{ $('Settings').first().json.business_name }}". Classify the email into exactly ONE category... [refer to source prompt for full text]`
     - User Message: `=From: {{ $json.from }}\nSubject: {{ $json.subject }}\n\nBody:\n{{ $json.body }}`
   - Configure a valid **OpenAI API** credential.

5. **Create the Classification Parser Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Parse Classification`.
   - Connect the output of **Classify Email** to the input of **Parse Classification**.
   - Insert parsing JavaScript containing category and priority fallback arrays (`urgent`, `sales_lead`, `support`, `billing`, `newsletter_spam`, `other`).

6. **Create the Label Fetcher Node:**
   - Add a **Gmail** node (`n8n-nodes-base.gmail`) named `Get Gmail Labels`.
   - Connect the output of **Parse Classification** to the input of `Get Gmail Labels`.
   - Configure parameters: Resource: `label`, Operation: `getAll`, `returnAll` enabled, `executeOnce` set to `true`.
   - Use the same **Gmail OAuth2** credential.

7. **Create the Label Resolver Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Resolve Labels`.
   - Connect the output of `Get Gmail Labels` to the input of `Resolve Labels`.
   - Insert mapping JavaScript to match email categories to corresponding Gmail label IDs using the configured prefix.

8. **Create the Condition Evaluation Node:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Has Label?`.
   - Connect the output of `Resolve Labels` to the input of `Has Label?`.
   - Configure conditions: Left Value: `={{ $json.labelMissing }}`, Operator: `false` (Boolean), loose type validation enabled.

9. **Create the Label Application Node:**
   - Add a **Gmail** node (`n8n-nodes-base.gmail`) named `Apply Gmail Label`.
   - Connect the **True** output of `Has Label?` to the input of `Apply Gmail Label`.
   - Configure parameters: Resource: `message`, Operation: `addLabels`, Message ID: `={{ $json.messageId }}`, Label IDs: `={{ $json.labelIds }}`.
   - Use the same **Gmail OAuth2** credential.

10. **Finalize Setup:**
    - Create the corresponding Gmail labels in your Google account (`AI/Urgent`, `AI/Sales Lead`, `AI/Support`, `AI/Billing`, `AI/Newsletter-Spam`, `AI/Other`) prior to activation.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Target Audience & Use Case | Designed for small business owners, freelancers, and support teams managing high-volume Gmail inboxes. |
| Operational Requirements | Requires an active n8n instance, a Gmail account with OAuth2 API access, and a funded OpenAI API key. |
| Customization Options | Category rules can be adjusted in the system prompt; label mappings can be modified inside the *Resolve Labels* code node. |