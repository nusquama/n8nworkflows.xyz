Auto-reply to 'Where Is My Order?' Emails with Shopify, Gmail and OpenRouter AI

https://n8nworkflows.xyz/workflows/auto-reply-to--where-is-my-order---emails-with-shopify--gmail-and-openrouter-ai-9705


# Auto-reply to 'Where Is My Order?' Emails with Shopify, Gmail and OpenRouter AI

### 1. Workflow Overview

This workflow automates customer support replies to emails asking about order status ("Where Is My Order?" or WISMO) for Shopify store customers. It listens for incoming emails via Gmail, classifies them to detect WISMO-related queries, and then uses an AI agent powered by OpenRouter (GPT-5 Nano) to generate personalized, friendly responses. The AI agent interacts with Shopify to retrieve order details, ensuring factual and reassuring answers. Non-WISMO emails are ignored by design.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Gmail Trigger node monitors incoming emails.
- **1.2 Email Classification:** Text Classifier classifies emails into "WISmo" or "other".
- **1.3 AI Processing:** The AI Agent node processes classified WISMO emails, retrieving order details from Shopify and generating a response using OpenRouter's GPT-5 model.
- **1.4 Output Action:** Replies to the customer email with the AI-generated message.
- **1.5 No-Op Handling:** Non-WISMO emails go through a no-operation node to end the flow quietly.
- **1.6 Configuration and Documentation:** Sticky Notes provide setup instructions and contact information.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Continuously polls Gmail for new emails every minute and triggers the workflow with new message snippets.

- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**

| Node Name    | Gmail Trigger                            |
|--------------|----------------------------------------|
| Type         | Trigger node for Gmail                  |
| Configuration| Polls new emails every minute, no filters applied |
| Key Expressions | N/A                                 |
| Input        | None (trigger node)                     |
| Output       | Email metadata and snippet (partial content) |
| Version Requirements | n8n version supporting gmailTrigger v1.3 |
| Potential Failures | Authentication issues, API rate limits, network errors |
| Sub-Workflow | None                                   |

---

#### 2.2 Email Classification

- **Overview:**  
Classifies incoming emails as either "WISmo" (order-related queries) or "other" using LangChain's Text Classifier node based on email snippet content.

- **Nodes Involved:**  
  - Text Classifier1

- **Node Details:**

| Node Name        | Text Classifier1                      |
|------------------|-------------------------------------|
| Type             | LangChain Text Classifier            |
| Configuration    | Input text from email snippet; categories: "WISmo" (order-related), "other" (non-order) |
| Key Expressions  | `={{ $json.snippet }}` (email snippet as input text) |
| Input            | Gmail Trigger output                  |
| Output           | Classification result with category label |
| Version Requirements | LangChain integration v1.1         |
| Potential Failures | Misclassification if snippet is unclear; expression errors if snippet missing |
| Sub-Workflow     | None                                |

---

#### 2.3 AI Processing

- **Overview:**  
Uses LangChain AI Agent to analyze classified WISmo emails, fetch relevant order data from Shopify, and generate personalized, empathetic replies via OpenRouter GPT-5 model.

- **Nodes Involved:**  
  - AI Agent1  
  - OpenRouter Chat Model  
  - get order details (Shopify Tool)

- **Node Details:**

| Node Name            | AI Agent1                            |
|----------------------|------------------------------------|
| Type                 | LangChain AI Agent                  |
| Configuration        | Input text template combining email subject and snippet; system prompt defines role as Shopify Customer Support Assistant with instructions and sample dialogues; uses OpenRouter GPT-5 Nano as language model and Shopify Tool as data source |
| Key Expressions      | `"=subject of email:{{ $('Gmail Trigger').item.json.Subject }}\ncontent:{{ $('Gmail Trigger').item.json.snippet }}"` for input; system message with detailed instructions and dynamic date `{{ $now }}` |
| Input                | Output from Text Classifier1 (WISmo category only) |
| Output               | AI-generated reply text             |
| Version Requirements | LangChain agent v2.1, OpenRouter integration v1 |
| Potential Failures    | API errors (OpenRouter or Shopify), missing order ID detection, incomplete data retrieval, prompt formatting errors |
| Sub-Workflow         | None                              |

| Node Name           | OpenRouter Chat Model               |
|---------------------|-----------------------------------|
| Type                | LangChain OpenRouter GPT Chat Node|
| Configuration       | Model set to "openai/gpt-5-nano", no extra options |
| Input               | AI Agent1 (as language model for responses) |
| Output              | Generated text for AI Agent1       |
| Version Requirements| LangChain v1                       |
| Potential Failures   | API key invalid or missing; API limits; network issues |
| Sub-Workflow        | None                              |

| Node Name           | get order details (Shopify Tool)   |
|---------------------|-----------------------------------|
| Type                | Shopify API Tool                   |
| Configuration       | Operation: getAll orders; returns all orders for authenticated user; uses accessToken authentication method |
| Input               | AI Agent1 (as AI tool)             |
| Output              | Shopify order data fed back to AI Agent1 for context |
| Version Requirements| Shopify API credentials required   |
| Potential Failures   | Authentication errors, permission issues, empty order data |
| Sub-Workflow        | None                              |

---

#### 2.4 Output Action

- **Overview:**  
Replies to the original email thread with the AI-generated response.

- **Nodes Involved:**  
  - Reply to the query

- **Node Details:**

| Node Name          | Reply to the query                   |
|--------------------|------------------------------------|
| Type               | Gmail node (send email reply)      |
| Configuration      | Replies to thread ID from trigger; message body is AI Agent output; plain text email; disables attribution footer |
| Key Expressions    | Message: `={{ $json.output }}`; ThreadId: `={{ $('Gmail Trigger').item.json.threadId }}` |
| Input              | AI Agent1 (final response)         |
| Output             | None (send action)                  |
| Version Requirements| Gmail node v2.1                    |
| Potential Failures  | Email sending errors, invalid thread ID, Gmail API quota limits |
| Sub-Workflow       | None                              |

---

#### 2.5 No-Op Handling

- **Overview:**  
Handles emails classified as "other" by doing nothing, ending the workflow silently.

- **Nodes Involved:**  
  - No Operation, do nothing

- **Node Details:**

| Node Name             | No Operation, do nothing            |
|-----------------------|-----------------------------------|
| Type                  | No Operation node (ends flow)      |
| Configuration         | None                              |
| Input                 | Text Classifier1 (non-WISmo branch)|
| Output                | None                             |
| Version Requirements  | n8n base noOp node v1              |
| Potential Failures     | None                             |
| Sub-Workflow          | None                             |

---

#### 2.6 Configuration and Documentation

- **Overview:**  
Contains sticky notes with setup instructions for Gmail, OpenRouter API, Shopify API credentials, and contact info for customization support.

- **Nodes Involved:**  
  - Sticky Note (Gmail setup)  
  - Sticky Note1 (OpenRouter setup)  
  - Sticky Note2 (Shopify setup)  
  - Sticky Note3 (Contact info)

- **Node Details:**

| Node Name        | Sticky Note (Gmail Trigger setup)  |
|------------------|-----------------------------------|
| Type             | Sticky Note (documentation)        |
| Content          | Link to Gmail Trigger node tutorial video; reminder to configure authentication before running workflow |
| Position         | Top-left corner                    |
| Usage            | Guidance for initial setup         |

| Node Name        | Sticky Note1 (OpenRouter API)      |
|------------------|-----------------------------------|
| Type             | Sticky Note                       |
| Content          | Steps to sign up at openrouter.ai, get API key, create credential in node |
| Usage            | API connection setup guide        |

| Node Name        | Sticky Note2 (Shopify API)         |
|------------------|-----------------------------------|
| Type             | Sticky Note                      |
| Content          | Detailed Shopify app creation steps: create app, set read_orders scope, install app, get access token & secret, get subdomain from URL |
| Usage            | Shopify API setup guide            |

| Node Name        | Sticky Note3 (Contact Info)        |
|------------------|-----------------------------------|
| Type             | Sticky Note                      |
| Content          | Contact email for customization requests and feedback |
| Usage            | Support and feedback channel       |

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                         | Input Node(s)       | Output Node(s)            | Sticky Note                                                                                                          |
|-------------------------|------------------------------------|---------------------------------------|---------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger           | gmailTrigger                       | Receive incoming emails                | None                | Text Classifier1          | *watch this to set up first node [Gmail Trigger node]* [Tutorial video](https://youtu.be/SBPU5a8-8Xo?si=u7k_-FDyPQc0-7g1) |
| Text Classifier1        | LangChain Text Classifier          | Classify email as WISmo or other       | Gmail Trigger       | AI Agent1, No Operation    |                                                                                                                      |
| AI Agent1               | LangChain AI Agent                 | Process WISmo emails with AI & Shopify| Text Classifier1    | Reply to the query         |                                                                                                                      |
| OpenRouter Chat Model   | LangChain OpenRouter GPT Chat      | Language model for AI Agent            | AI Agent1           | AI Agent1                  | **Connect AI:** sign up at openrouter.ai, create credential with API key (see Sticky Note1)                          |
| get order details       | Shopify Tool                      | Retrieve all Shopify orders            | AI Agent1 (ai_tool) | AI Agent1                  | **Connect Shopify:** setup app, get access token & secret, set subdomain (see Sticky Note2)                         |
| Reply to the query      | Gmail node (send email reply)      | Reply to customer's email thread       | AI Agent1           | None                      |                                                                                                                      |
| No Operation, do nothing| No Operation (noOp)                | End flow silently for non-WISmo emails | Text Classifier1    | None                      |                                                                                                                      |
| Sticky Note             | Sticky Note                       | Setup instructions for Gmail node      | None                | None                      | *watch this to set up first node [Gmail Trigger node]* [Tutorial video](https://youtu.be/SBPU5a8-8Xo?si=u7k_-FDyPQc0-7g1) |
| Sticky Note1            | Sticky Note                       | Setup instructions for OpenRouter API  | None                | None                      | **Connect AI:** sign up at openrouter.ai, create credential with API key                                            |
| Sticky Note2            | Sticky Note                       | Setup instructions for Shopify API     | None                | None                      | **Connect Shopify:** create app, set read_orders scope, install app, copy access token, secret, subdomain            |
| Sticky Note3            | Sticky Note                       | Contact info for customization & feedback | None                | None                      | Contact [me](mailto:uzmannazlin76@gmail.com) for customizations and feedback                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**
   - Node type: Gmail Trigger
   - Configure OAuth2 authentication with your Gmail account.
   - Set polling interval to every 1 minute.
   - No filters to capture all incoming emails.
   - Position node at left side for clarity.

2. **Add Text Classifier Node:**
   - Node type: LangChain Text Classifier
   - Input: Set to `={{ $json.snippet }}` to classify email snippet.
   - Define two categories:
     - "WISmo": For emails related to product orders (e.g., "where is my order", "when will my order come?")
     - "other": For all other emails.
   - Connect output of Gmail Trigger to this node.

3. **Add AI Agent Node:**
   - Node type: LangChain AI Agent
   - Input Text: Template combining email subject and snippet as:
     ```
     subject of email:{{ $('Gmail Trigger').item.json.Subject }}
     content:{{ $('Gmail Trigger').item.json.snippet }}
     ```
   - Paste the system message exactly as in the original workflow to define the AI assistant role, instructions, tools, examples, and tone.
   - Configure prompt type as "define".
   - Connect the "WISmo" output branch from Text Classifier to AI Agent.

4. **Add OpenRouter Chat Model Node:**
   - Node type: LangChain OpenRouter GPT Chat
   - Model: Select "openai/gpt-5-nano".
   - Link this node as the language model for AI Agent.
   - Create and configure OpenRouter API credentials:
     - Sign up at https://openrouter.ai/
     - Obtain API key.
     - Create API credential in n8n OpenRouter Chat Model node.
   - Connect AI Agent's language model input to this node.

5. **Add Shopify Tool Node (get order details):**
   - Node type: Shopify Tool
   - Operation: "getAll" orders
   - Return all orders: true
   - Authentication: OAuth2 or Access Token (create Shopify private app with read_orders scope)
   - Configure credentials with:
     - Access token from your Shopify app.
     - API secret key.
     - Subdomain (yourshop.myshopify.com → "yourshop")
   - Link this node as an AI tool to the AI Agent.
   
6. **Add Gmail Node to Reply to Query:**
   - Node type: Gmail (send email)
   - Operation: reply
   - Set message content to AI Agent output: `={{ $json.output }}`
   - Message ID: use threadId from Gmail Trigger: `={{ $('Gmail Trigger').item.json.threadId }}`
   - Disable attribution footer.
   - Connect AI Agent output to this node.

7. **Add No Operation Node:**
   - Node type: No Operation (noOp)
   - Connect the "other" output branch from Text Classifier to this node to silently ignore unrelated emails.

8. **Add Sticky Notes for Documentation:**
   - Add sticky notes with setup instructions for Gmail, OpenRouter API, Shopify API, and contact info.
   - Include links and step-by-step guides as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|
| *Watch this to set up first node [Gmail Trigger node]* [![Watch Tutorial](https://img.youtube.com/vi/SBPU5a8-8Xo/0.jpg)](https://youtu.be/SBPU5a8-8Xo?si=u7k_-FDyPQc0-7g1) | Gmail Trigger node setup tutorial video                                                  |
| Connect AI: Sign up at [OpenRouter](https://openrouter.ai/), get API key, create credential in node                                         | OpenRouter API setup instructions                                                       |
| Connect your Shopify store: Create private app with read_orders scope; get access token, API secret, subdomain                               | Shopify API setup instructions                                                          |
| Contact [me](mailto:uzmannazlin76@gmail.com) for customizations and feedback                                                                | Support and customization contact                                                       |

---

This documentation should enable advanced users and automation agents to fully understand, reproduce, and adapt the workflow for automated AI-driven customer support replies based on Shopify order data and Gmail email inputs.