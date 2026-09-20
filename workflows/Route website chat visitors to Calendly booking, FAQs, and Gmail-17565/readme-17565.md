Route website chat visitors to Calendly booking, FAQs, and Gmail

https://n8nworkflows.xyz/workflows/route-website-chat-visitors-to-calendly-booking--faqs--and-gmail-17565


# Route website chat visitors to Calendly booking, FAQs, and Gmail

### 1. Workflow Overview

This workflow implements an interactive, rule-based website chat assistant designed to handle inbound visitor requests. It greets users, displays a structured main menu, and guides them toward scheduling a meeting, browsing service FAQs, or submitting a direct message that is automatically emailed to an inbox via Gmail. 

The execution logic is divided into five functional blocks:
- **1.1 Initialization & Welcome:** Configures global environment variables and sends the initial greeting containing a data privacy link.
- **1.2 Main Menu & Scheduling Routing:** Displays the primary options menu, evaluates user selections, and manages the Calendly meeting booking redirect flow.
- **1.3 Services Information Routing:** Presents detailed offerings for IT consulting and process automation, complete with navigation loops.
- **1.4 Contact Capture & Validation:** Collects visitor contact information (email or phone), extracts and validates parameters using regex, and prompts for message input.
- **1.5 Message Confirmation & Email Dispatch:** Handles final review options (send, replace, or abort), transmits the lead data through Gmail, and acknowledges receipt.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Welcome
- **Overview:** Initializes global workflow settings (contact emails, booking links, and privacy policy URLs) and establishes the entry point for the website chat widget.
- **Nodes Involved:** 
  - `Set Workflow Configuration`
  - `Send Welcome Message`
  - `Display Main Menu`

- **Node Details:**
  - **Set Workflow Configuration**
    - *Type & Role:* `n8n-nodes-base.set` (Set) — Establishes global configuration parameters.
    - *Configuration Choices:* Defines `contactEmail`, `calendlyLink`, and `imprintUrl` variables.
    - *Key Expressions:* `info@example.com`, `https://calendly.com/...`, `https://www.example.com/privacy.html`.
    - *Connections:* Input: None (Trigger-like entry) | Output: `Send Welcome Message`.
    - *Edge Cases:* Missing or invalid URL strings will cause downstream template evaluations to fail.
  - **Send Welcome Message**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chat` (Chat Trigger/Message) — Sends the introductory message to the visitor.
    - *Configuration Choices:* Greets the user and outputs the terms of service agreement link.
    - *Key Expressions:* `{{ $('Set Workflow Configuration').item.json.imprintUrl }}`
    - *Connections:* Input: `Set Workflow Configuration` | Output: `Display Main Menu`.
    - *Edge Cases:* Failure if the upstream configuration node does not resolve correctly.
  - **Display Main Menu**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chat` (Send and Wait) — Displays the numbered main menu options to the user and halts execution until input is received.
    - *Configuration Choices:* Configured with a 15-minute resume wait time limit.
    - *Connections:* Input: `Send Welcome Message` | Output: `Route by Main Menu Choice`.
    - *Edge Cases:* Session timeout after 15 minutes of inactivity.

---

#### 2.2 Main Menu & Scheduling Routing
- **Overview:** Evaluates the visitor’s menu input, directing them to the scheduling sequence, service menus, direct messaging, or handling navigation keywords like "menu" or "back".
- **Nodes Involved:**
  - `Route by Main Menu Choice`
  - `Initiate Meeting Scheduling`
  - `Route Post-Schedule Reply`

- **Node Details:**
  - **Route by Main Menu Choice**
    - *Type & Role:* `n8n-nodes-base.switch` (Switch) — Routes execution branch based on user selection.
    - *Configuration Choices:* Evaluates `chatInput` against outputs "1", "2", "3", "back", and "menu" with a fallback condition.
    - *Key Expressions:* `{{ $json.chatInput }}`
    - *Connections:* Input: `Display Main Menu` | Output: `Initiate Meeting Scheduling`, `Provide Services Information`, `Request Contact Details`, or loops back to `Display Main Menu`.
    - *Edge Cases:* Unmatched string values trigger the fallback output.
  - **Initiate Meeting Scheduling**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chat` (Send and Wait) — Provides the external booking link and waits for further user input.
    - *Configuration Choices:* 15-minute timeout window.
    - *Key Expressions:* `{{ $('Set Workflow Configuration').item.json.calendlyLink }}`
    - *Connections:* Input: `Route by Main Menu Choice` | Output: `Route Post-Schedule Reply`.
  - **Route Post-Schedule Reply**
    - *Type & Role:* `n8n-nodes-base.switch` (Switch) — Handles post-scheduling inputs (e.g., returning to the main menu).
    - *Key Expressions:* `{{ $json.chatInput.toLowerCase() }}`
    - *Connections:* Input: `Initiate Meeting Scheduling` | Output: Loops back to `Display Main Menu` or re-triggers scheduling.

---

#### 2.3 Services Information Routing
- **Overview:** Displays detailed service descriptions for IT consulting and process automation, letting users toggle between topics or return to the main menu.
- **Nodes Involved:**
  - `Provide Services Information`
  - `Route by Service Choice`
  - `Present IT Consulting Services`
  - `Route Post-IT Consulting Chat`
  - `Present Automation Services`
  - `Route Post-Automation Info`

- **Node Details:**
  - **Provide Services Information**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chat` (Send and Wait) — Presents the services submenu.
    - *Connections:* Input: `Route by Main Menu Choice` | Output: `Route by Service Choice`.
  - **Route by Service Choice**
    - *Type & Role:* `n8n-nodes-base.switch` (Switch) — Directs user to IT Consulting (1), Process Automation (2), or navigation resets.
    - *Connections:* Input: `Provide Services Information` | Output: `Present IT Consulting Services`, `Present Automation Services`, or `Display Main Menu`.
  - **Present IT Consulting Services** & **Present Automation Services**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chat` (Send and Wait) — Outlines technical expertise, toolsets, and scope.
    - *Connections:* Input: `Route by Service Choice` | Output: Respective `Route Post-*` switches.
  - **Route Post-IT Consulting Chat** & **Route Post-Automation Info**
    - *Type & Role:* `n8n-nodes-base.switch` (Switch) — Allows looping back to services or returning to the main menu.
    - *Connections:* Input: Service presentation nodes | Output: `Display Main Menu` or `Provide Services Information`.

---

#### 2.4 Contact Capture & Validation
- **Overview:** Collects a phone number or email address from the visitor, runs a code-based extraction script, validates the results, and loops back if data extraction fails.
- **Nodes Involved:**
  - `Request Contact Details`
  - `Extract Contact from Message`
  - `Check for Contact Information`
  - `Retry Contact Information Request`
  - `Verify Contact and Request Message`
  - `Route Post-Contact Confirmation`

- **Node Details:**
  - **Request Contact Details**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chat` (Send and Wait) — Prompts for user email or phone.
    - *Connections:* Input: `Route by Main Menu Choice` | Output: `Extract Contact from Message`.
  - **Extract Contact from Message**
    - *Type & Role:* `n8n-nodes-base.code` (Code) — Parses chat input using regular expressions to isolate valid email addresses and phone numbers.
    - *Configuration Choices:* Executes JavaScript utilizing `emailRegex` and `phoneRegex` filters.
    - *Key Expressions:* `{{ $json.chatInput }}`
    - *Connections:* Input: `Request Contact Details` | Output: `Check for Contact Information`.
    - *Edge Cases:* Unformatted inputs or edge-case phone strings require length validation (`>= 6` digits).
  - **Check for Contact Information**
    - *Type & Role:* `n8n-nodes-base.if` (If) — Evaluates whether `extractedEmail` or `extractedPhone` fields contain valid data.
    - *Key Expressions:* `{{ $json.extractedEmail }}` / `{{ $json.extractedPhone }}`
    - *Connections:* Input: `Extract Contact from Message` | Output: True branch to `Verify Contact and Request Message`; False branch to `Retry Contact Information Request`.
  - **Retry Contact Information Request**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chat` (Message) — Notifies the user that parsing failed and provides a fallback email address.
    - *Connections:* Input: `Check for Contact Information` (False) | Output: Loops back to `Request Contact Details`.
  - **Verify Contact and Request Message**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chat` (Send and Wait) — Confirms the captured contact data and asks the user to provide their message.
    - *Key Expressions:* Evaluates ternary operators depending on whether an email or phone was captured.
    - *Connections:* Input: `Check for Contact Information` (True) | Output: `Confirm Message Prior to Sending`.

---

#### 2.5 Message Confirmation & Email Dispatch
- **Overview:** Summarizes the user's message, manages send/replace/abort options, dispatches the notification lead to Gmail, and closes the chat interaction.
- **Nodes Involved:**
  - `Confirm Message Prior to Sending`
  - `Route Message Send Options`
  - `Handle Invalid Response`
  - `Send Email via Gmail`
  - `Acknowledge Email Receipt`
  - `Route Post-Email Confirmation`

- **Node Details:**
  - **Confirm Message Prior to Sending**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chat` (Send and Wait) — Asks the user to confirm sending or rewriting their message.
    - *Connections:* Input: `Verify Contact and Request Message` | Output: `Route Message Send Options`.
  - **Route Message Send Options**
    - *Type & Role:* `n8n-nodes-base.switch` (Switch) — Branches based on user commands ("send", "replace", "menu", "back", or invalid input).
    - *Key Expressions:* `{{ $json.chatInput.toLowerCase() }}`
    - *Connections:* Input: Confirmations / Fallbacks | Output: `Send Email via Gmail`, `Verify Contact and Request Message`, `Display Main Menu`, `Request Contact Details`, or `Handle Invalid Response`.
  - **Handle Invalid Response**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chat` (Send and Wait) — Prompts the user again if invalid control keywords are provided.
    - *Connections:* Input: `Route Message Send Options` (Fallback) | Output: Loops back to `Route Message Send Options`.
  - **Send Email via Gmail**
    - *Type & Role:* `n8n-nodes-base.gmail` (Gmail) — Formats an HTML email containing the lead details and sends it to the configured recipient.
    - *Configuration Choices:* Utilizes Gmail OAuth2 credentials.
    - *Key Expressions:* `{{ $('Set Workflow Configuration').item.json.contactEmail }}` and outputs from extraction nodes.
    - *Credentials Required:* Gmail OAuth2.
    - *Edge Cases:* Authentication expiration or missing OAuth scopes will cause delivery failure.
  - **Acknowledge Email Receipt**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chat` (Send and Wait) — Confirms to the user that their message was successfully sent.
    - *Connections:* Input: `Send Email via Gmail` | Output: `Route Post-Email Confirmation`.
  - **Route Post-Email Confirmation**
    - *Type & Role:* `n8n-nodes-base.switch` (Switch) — Directs user post-submission options.
    - *Connections:* Input: `Acknowledge Email Receipt` | Output: `Display Main Menu`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow overview and general setup instructions | None | None | Route website chat visitors to booking, FAQs, and Gmail... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Chat entry documentation | None | None | Chat entry setup... |
| Sticky Note2 | n8n-nodes-base.stickyNote | Main menu routing documentation | None | None | Main menu routing... |
| Sticky Note3 | n8n-nodes-base.stickyNote | Schedule meeting loop documentation | None | None | 1. Schedule meeting loop... |
| Sticky Note4 | n8n-nodes-base.stickyNote | Contact details collection documentation | None | None | 3. Collect contact details... |
| Sticky Note5 | n8n-nodes-base.stickyNote | Lead message confirmation documentation | None | None | Confirm lead message... |
| Sticky Note6 | n8n-nodes-base.stickyNote | Send decision handling documentation | None | None | Send decision handling... |
| Sticky Note7 | n8n-nodes-base.stickyNote | Email confirmation documentation | None | None | Email lead confirmation... |
| Sticky Note8 | n8n-nodes-base.stickyNote | Post-send routing documentation | None | None | Post-send routing... |
| Sticky Note9 | n8n-nodes-base.stickyNote | Services menu selection documentation | None | None | 2. Services menu selection... |
| Sticky Note10 | n8n-nodes-base.stickyNote | Service detail loops documentation | None | None | Service detail loops... |
| Set Workflow Configuration | n8n-nodes-base.set | Defines global configuration variables | None | Send Welcome Message | |
| Send Welcome Message | @n8n/n8n-nodes-langchain.chat | Sends initial greeting and privacy policy link | Set Workflow Configuration | Display Main Menu | Chat entry setup |
| Display Main Menu | @n8n/n8n-nodes-langchain.chat | Displays main options menu | Send Welcome Message | Route by Main Menu Choice | Chat entry setup |
| Route by Main Menu Choice | n8n-nodes-base.switch | Routes visitor based on primary menu choice | Display Main Menu | Initiate Meeting Scheduling, Provide Services Information, Request Contact Details, Display Main Menu | Main menu routing |
| Initiate Meeting Scheduling | @n8n/n8n-nodes-langchain.chat | Provides Calendly scheduling link | Route by Main Menu Choice | Route Post-Schedule Reply | 1. Schedule meeting loop |
| Route Post-Schedule Reply | n8n-nodes-base.switch | Routes post-schedule interaction | Initiate Meeting Scheduling | Display Main Menu, Initiate Meeting Scheduling | 1. Schedule meeting loop |
| Provide Services Information | @n8n/n8n-nodes-langchain.chat | Presents services options menu | Route by Main Menu Choice | Route by Service Choice | 2. Services menu selection |
| Route by Service Choice | n8n-nodes-base.switch | Directs user to specific service overview | Provide Services Information | Present IT Consulting Services, Present Automation Services, Display Main Menu, Provide Services Information | 2. Services menu selection |
| Present IT Consulting Services | @n8n/n8n-nodes-langchain.chat | Displays IT consulting details | Route by Service Choice | Route Post-IT Consulting Chat | Service detail loops |
| Route Post-IT Consulting Chat | n8n-nodes-base.switch | Handles post-IT consulting navigation | Present IT Consulting Services | Display Main Menu, Provide Services Information, Present IT Consulting Services | Service detail loops |
| Present Automation Services | @n8n/n8n-nodes-langchain.chat | Displays automation engineering details | Route by Service Choice | Route Post-Automation Info | Service detail loops |
| Route Post-Automation Info | n8n-nodes-base.switch | Handles post-automation navigation | Present Automation Services | Display Main Menu, Provide Services Information, Present Automation Services | Service detail loops |
| Request Contact Details | @n8n/n8n-nodes-langchain.chat | Prompts user for email or phone number | Route by Main Menu Choice, Route Message Send Options, Retry Contact Information Request | Extract Contact from Message | 3. Collect contact details |
| Extract Contact from Message | n8n-nodes-base.code | Extracts email and phone numbers via regex | Request Contact Details | Check for Contact Information | 3. Collect contact details |
| Check for Contact Information | n8n-nodes-base.if | Verifies presence of contact details | Extract Contact from Message | Verify Contact and Request Message, Retry Contact Information Request | 3. Collect contact details |
| Retry Contact Information Request | @n8n/n8n-nodes-langchain.chat | Prompts retry when contact info is missing | Check for Contact Information | Request Contact Details | 3. Collect contact details |
| Verify Contact and Request Message | @n8n/n8n-nodes-langchain.chat | Confirms contact info and asks for message | Check for Contact Information, Route Message Send Options | Route Post-Contact Confirmation | Confirm lead message |
| Route Post-Contact Confirmation | n8n-nodes-base.switch | Handles post-contact navigation | Verify Contact and Request Message | Display Main Menu, Request Contact Details, Confirm Message Prior to Sending | Confirm lead message |
| Confirm Message Prior / Sending | @n8n/n8n-nodes-langchain.chat | Asks user to confirm message before sending | Route Post-Contact Confirmation | Route Message Send Options | Confirm lead message |
| Route Message Send Options | n8n-nodes-base.switch | Handles send, replace, or abort actions | Confirm Message Prior to Sending | Send Email via Gmail, Verify Contact and Request Message, Display Main Menu, Request Contact Details, Handle Invalid Response | Send decision handling |
| Handle Invalid Response | @n8n/n8n-nodes-langchain.chat | Prompts user after invalid command entry | Route Message Send Options | Route Message Send Options | Send decision handling |
| Send Email via Gmail | n8n-nodes-base.gmail | Sends captured lead details to inbox via email | Route Message Send Options | Acknowledge Email Receipt | Email lead confirmation |
| Acknowledge Email Receipt | @n8n/n8n-nodes-langchain.chat | Confirms message delivery to the visitor | Send Email via Gmail | Route Post-Email Confirmation | Email lead confirmation |
| Route Post-Email Confirmation | n8n-nodes-base.switch | Handles final navigation after email delivery | Acknowledge Email Receipt | Display Main Menu, Acknowledge Email Receipt | Post-send routing |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Configuration Node:**
   - Add a **Set** node (`n8n-nodes-base.set`), rename it to `Set Workflow Configuration`.
   - Add three string assignments: `contactEmail` (`info@example.com`), `calendlyLink` (`https://calendly.com/...`), and `imprintUrl` (`https://www.example.com/privacy.html`).
2. **Add Welcome & Menu Nodes:**
   - Connect a **Chat Trigger** / Chat Message node named `Send Welcome Message`. Set the message property to output the privacy policy using the expression `{{ $('Set Workflow Configuration').item.json.imprintUrl }}`.
   - Connect a **Send and Wait Chat** node named `Display Main Menu`. Set the resume wait time limit to 15 minutes.
3. **Setup Main Switch:**
   - Add a **Switch** node (`n8n-nodes-base.switch`) named `Route by Main Menu Choice`.
   - Configure rules to evaluate `{{ $json.chatInput }}` against values `1`, `2`, `3`, `back`, and `menu`.
4. **Build Scheduling Branch:**
   - Create a **Send and Wait Chat** node named `Initiate Meeting Scheduling`. Use expression `{{ $('Set Workflow Configuration').item.json.calendlyLink }}`.
   - Add a **Switch** node named `Route Post-Schedule Reply` to route the `menu` keyword back to `Display Main Menu`.
5. **Build Services Branch:**
   - Create a **Send and Wait Chat** node named `Provide Services Information` with options for IT Consulting (1) and Automation (2).
   - Add a **Switch** (`Route by Service Choice`) to branch execution accordingly.
   - Add nodes `Present IT Consulting Services` and `Present Automation Services`, each followed by respective **Switch** nodes (`Route Post-IT Consulting Chat` and `Route Post-Automation Info`) to manage navigation loops.
6. **Build Contact Collection Branch:**
   - Add a **Send and Wait Chat** node named `Request Contact Details`.
   - Connect a **Code** node named `Extract Contact from Message` containing JavaScript regex parsing logic for email and phone numbers.
   - Connect an **If** node named `Check for Contact Information` verifying if `extractedEmail` or `extractedPhone` exists.
   - If false, route to a **Chat** node (`Retry Contact Information Request`) looping back to `Request Contact Details`.
   - If true, connect to a **Send and Wait Chat** (`Verify Contact and Request Message`) and a **Switch** (`Route Post-Contact Confirmation`).
7. **Build Confirmation and Dispatch Branch:**
   - Add a **Send and Wait Chat** node named `Confirm Message Prior to Sending`.
   - Connect a **Switch** (`Route Message Send Options`) evaluating `send`, `replace`, `back`, and `menu`.
   - Configure the `send` output route to a **Gmail** node (`Send Email via Gmail`). Set credentials to a valid **Gmail OAuth2** connection and map HTML body content using previous node outputs.
   - Connect the Gmail node to a **Send and Wait Chat** (`Acknowledge Email Receipt`) and a final **Switch** (`Route Post-Email Confirmation`) to loop users back to the main menu.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Official n8n Documentation | [https://docs.n8n.io](https://docs.n8n.io) |
| Calendly Scheduling Integration | [https://calendly.com](https://calendly.com) |