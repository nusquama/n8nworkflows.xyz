Build Stripe dispute evidence from WooCommerce orders, Gmail, and OpenAI

https://n8nworkflows.xyz/workflows/build-stripe-dispute-evidence-from-woocommerce-orders--gmail--and-openai-20549


# Build Stripe dispute evidence from WooCommerce orders, Gmail, and OpenAI

### 1. Workflow Overview

This workflow automates the entire lifecycle of Stripe charge disputes (`charge.dispute.created` and `charge.dispute.closed`). It ingests Stripe webhook events, gathers context from WooCommerce and Gmail, uses OpenAI to construct factual and tailored rebuttal arguments, posts the evidence back to Stripe, logs metrics in Google Sheets, broadcasts updates to Slack, and handles time-sensitive deadline reminders.

The processing logic is divided into the following functional blocks:
- **1.1 Input Reception & Configuration:** Receives Stripe webhook payloads and injects core store policies and configuration flags.
- **1.2 Event Routing & Validation:** Determines whether the event requires evidence preparation or outcome recording, fetches Stripe dispute/charge data, parses critical security signals, and screens out disputes that do not require merchant intervention.
- **1.3 Data Retrieval & Case Assembly:** Pulls associated orders, customer history from WooCommerce, and correspondence history from Gmail, consolidating everything into a structured case file.
- **1.4 AI Evidence Drafting:** Utilizes an OpenAI chat model configured with a specialized system prompt and strict structured output schema to draft the dispute defense.
- **1.5 Stripe Evidence Submission & Logging:** Translates the AI output into Stripe API formatting, submits or saves the draft evidence, notifies the team via Slack, and logs entries in Google Sheets.
- **1.6 Deferred Deadline Monitoring:** Waits until a calculated reminder threshold is met, re-checks Stripe submission status, and fires a Slack alert if the dispute remains unsubmitted.
- **1.7 Dispute Outcome Recording:** Processes `charge.dispute.closed` events by updating Google Sheets records and posting resolution announcements to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes the workflow execution upon receiving a Stripe dispute webhook event and establishes global operational parameters, store metadata, and rules for automated submission.
- **Nodes Involved:** 
  - `When Stripe Dispute Event Triggers`
  - `Set Store Configuration`
- **Node Details:**
  - **When Stripe Dispute Event Triggers**
    - *Type and Technical Role:* `n8n-nodes-base.stripeTrigger` (Webhook Trigger).
    - *Configuration Choices:* Listens for events `charge.dispute.created` and `charge.dispute.closed`.
    - *Key Expressions:* None.
    - *Connections:* Output connects to `Set Store Configuration`.
    - *Version/Errors:* Requires valid Stripe webhook credentials and an active public URL configuration.
  - **Set Store Configuration**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation).
    - *Configuration Choices:* Sets global parameters including store name, URLs (refund, terms, shipping policies), checkout policy disclosures, auto-submit boolean flag (`false` by default), minimum dispute amount threshold, email lookback window (180 days), reminder hours (48 hours), Slack channel (`#chargebacks`), and tracker Google Sheet URL.
    - *Key Expressions:* Static string and numeric assignments.
    - *Connections:* Input from `When Stripe Dispute Event Triggers`; output to `Route by Event Type`.

---

#### 2.2 Event Routing & Validation
- **Overview:** Routes events based on their type, queries the Stripe API for full expanded charge and dispute details, extracts payment risk signals, and filters out non-actionable disputes.
- **Nodes Involved:**
  - `Route by Event Type`
  - `Fetch Stripe Dispute Details`
  - `Extract Payment Signals`
  - `Check If Response Needed`
- **Node Details:**
  - **Route by Event Type**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Flow Control).
    - *Configuration Choices:* Branches execution based on the Stripe event type. Output 0 (“New dispute”) matches `charge.dispute.created`; Output 1 (“Dispute closed”) matches `charge.dispute.closed`.
    - *Key Expressions:* Evaluates `{{ $('When Stripe Dispute Event Triggers').first().json.type }}`.
    - *Connections:* Input from `Set Store Configuration`; Output 0 goes to `Fetch Stripe Dispute Details`, Output 1 goes to `Format Dispute Outcome Data`.
  - **Fetch Stripe Dispute Details**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request).
    - *Configuration Choices:* Performs a GET request to Stripe's dispute endpoint with query parameter `expand[]=charge`. Authenticated via Stripe API credentials.
    - *Key Expressions:* Uses `={{ $('When Stripe Dispute Event Triggers').first().json.data.object.id }}` in the URL path.
    - *Connections:* Input from `Route by Event Type`; output to `Extract Payment Signals`.
    - *Edge Cases:* Rate limits, API downtime, or invalid dispute IDs resulting in 4xx/5xx errors.
  - **Extract Payment Signals**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation).
    - *Configuration Choices:* Normalizes raw Stripe charge objects, handles zero-decimal currencies, parses CVC/AVS checks, 3D Secure status, customer contact data, metadata order IDs, and due dates.
    - *Key Expressions:* Custom JS block parsing `$input.first().json`.
    - *Connections:* Input from `Fetch Stripe Dispute Details`; output to `Check If Response Needed`.
    - *Edge Cases:* Missing nested properties (e.g., empty `payment_method_details`) handled using optional chaining.
  - **Check If Response Needed**
    - *Type and Technical Role:* `n8n-nodes-base.filter` (Conditional Filter).
    - *Configuration Choices:* Enforces rules: dispute status contains `needs_response`, `submissionCount` equals `0`, `pastDue` is `false`, and dispute `amount` is greater than or equal to `minimumDisputeAmount`.
    - *Key Expressions:* References fields generated by `Extract Payment Signals` and `Set Store Configuration`.
    - *Connections:* Input from `Extract Payment Signals`; output to `Check WooCommerce Order`.

---

#### 2.3 Data Retrieval & Case Assembly
- **Overview:** Determines if a WooCommerce order identifier is linked to the transaction, retrieves order and customer history records, searches customer communication in Gmail, and combines all artifacts into a structured case file.
- **Nodes Involved:**
  - `Check WooCommerce Order`
  - `Fetch WooCommerce Order`
  - `Fetch Customer Order History`
  - `Search Customer Emails in Gmail`
  - `Assemble Dispute Case File`
- **Node Details:**
  - **Check WooCommerce Order**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration Choices:* Checks if `orderId` is present.
    - *Key Expressions:* `={{ $json.orderId }}` (Not Empty).
    - *Connections:* Input from `Check If Response Needed`. True branch connects to `Fetch WooCommerce Order`; False branch bypasses to `Search Customer Emails in Gmail`.
  - **Fetch WooCommerce Order**
    - *Type and Technical Role:* `n8n-nodes-base.wooCommerce` (API Integration).
    - *Configuration Choices:* Retrieves order details by ID. Configured with error handling (`onError: continueRegularOutput`) and `alwaysOutputData: true`.
    - *Key Expressions:* `={{ $json.orderId }}`.
    - *Connections:* Input from `Check WooCommerce Order`; output to `Fetch Customer Order History`.
    - *Edge Cases:* Order deleted in WooCommerce or invalid ID; caught via error handling configuration.
  - **Fetch Customer Order History**
    - *Type and Technical Role:* `n8n-nodes-base.wooCommerce` (API Integration).
    - *Configuration Choices:* Queries all orders matching the customer's billing email with a limit of 50. Executes once.
    - *Key Expressions:* `={{ $json.billing ? $json.billing.email : '' }}`.
    - *Connections:* Input from `Fetch WooCommerce Order`; output to `Search Customer Emails in Gmail`.
  - **Search Customer Emails in Gmail**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Search).
    - *Configuration Choices:* Searches messages up to the lookback window specified in configuration, limited to 15 results. Executes once with error continuation.
    - *Key Expressions:* Constructs query using customer email and `emailLookbackDays`.
    - *Connections:* Input from `Fetch Customer Order History` (or `Check WooCommerce Order` if order ID is missing); output to `Assemble Dispute Case File`.
  - **Assemble Dispute Case File**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation).
    - *Configuration Choices:* Aggregates data sources into a standardized JSON payload, parses shipment tracking metadata, filters prior customer orders, summarizes Gmail snippets, and computes deterministic strengths and gaps.
    - *Key Expressions:* Custom JS script aggregating previous node outputs.
    - *Connections:* Input from `Search Customer Emails in Gmail`; output to `Draft Dispute Response with AI`.

---

#### 2.4 AI Evidence Drafting
- **Overview:** Passes the compiled case file to an OpenAI language model constrained by a structured output schema to formulate an appropriate dispute defense and analysis.
- **Nodes Involved:**
  - `Draft Dispute Response with AI`
  - `OpenAI Chat Model`
  - `Define Evidence Output Format`
- **Node Details:**
  - **Draft Dispute Response with AI**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (LangChain Chain Node).
    - *Configuration Choices:* Acts as an AI Agent/LLM Chain utilizing system instructions tailored for chargeback analysts (factual, professional, reason-specific rules).
    - *Key Expressions:* `={{ JSON.stringify($json.caseFile, null, 2) }}`.
    - *Connections:* Connected to `OpenAI Chat Model` (AI Model) and `Define Evidence Output Format` (Output Parser); input from `Assemble Dispute Case File`; output to `Map Evidence to Stripe Fields`.
  - **OpenAI Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model Provider).
    - *Configuration Choices:* Configured with model `gpt-4o-mini` and a low temperature (`0.2`) for deterministic output.
    - *Connections:* Connected as a sub-node provider to `Draft Dispute Response with AI`.
  - **Define Evidence Output Format**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Structured Output Parser).
    - *Configuration Choices:* Manually defined JSON schema forcing keys: `recommendation` (submit/accept), `win_likelihood` (high/medium/low), `rationale`, `product_description`, `rebuttal_letter`, `customer_communication_summary`, `refund_refusal_explanation`, `cancellation_rebuttal`, `duplicate_charge_explanation`, and `missing_evidence` (array of strings).
    - *Connections:* Connected as a sub-node parser to `Draft Dispute Response with AI`.

---

#### 2.5 Stripe Evidence Submission & Logging
- **Overview:** Formats the AI-generated defense into Stripe-compliant parameters, posts the evidence back to the Stripe API, broadcasts summary notifications to Slack, and appends a tracking entry in Google Sheets.
- **Nodes Involved:**
  - `Map Evidence to Stripe Fields`
  - `Post Evidence to Stripe`
  - `Notify Team in Slack`
  - `Prepare Tracker Row Data`
  - `Log to Dispute Tracker Sheets`
- **Node Details:**
  - **Map Evidence to Stripe Fields**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Mapping).
    - *Configuration Choices:* Maps AI output properties and raw case signals to Stripe form-encoded evidence fields (`customer_name`, `shipping_tracking_number`, `uncategorized_text`, etc.), validates character limits (20,000 chars per field), computes the auto-submit condition, and determines the reminder timestamp.
    - *Key Expressions:* Custom JS processing object entries into URLSearchParams format.
    - *Connections:* Input from `Draft Dispute Response with AI`; output to `Post Evidence to Stripe`.
  - **Post Evidence to Stripe**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request).
    - *Configuration Choices:* POST request to Stripe disputes endpoint using `form-urlencoded` body content. Authenticated via Stripe API credentials.
    - *Key Expressions:* `={{ https://api.stripe.com/v1/disputes/{{ $('Extract Payment Signals').first().json.disputeId }} }}` and `={{ $json.formBody }}`.
    - *Connections:* Input from `Map Evidence to Stripe Fields`; outputs to `Notify Team in Slack`, `Prepare Tracker Row Data`, and `Check if Saved as Draft`.
  - **Notify Team in Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Messaging Integration).
    - *Configuration Choices:* Formats a detailed Markdown alert highlighting dispute reasons, amounts, actions taken, recommendations, deadlines, and dashboard links.
    - *Key Expressions:* Dynamic template expressions referencing previous nodes.
    - *Connections:* Input from `Post Evidence to Stripe`.
  - **Prepare Tracker Row Data**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation).
    - *Configuration Choices:* Formats row assignments matching Google Sheets column requirements (Opened date, Dispute ID, Order, Customer email, Amount, Currency, Reason, Network reason code, Respond by, Recommendation, Win likelihood, Action, Missing evidence, Outcome, Closed on, Stripe link).
    - *Key Expressions:* Dynamic expressions utilizing `$now.toFormat()` and upstream node data.
    - *Connections:* Input from `Post Evidence to Stripe`; output to `Log to Dispute Tracker Sheets`.
  - **Log to Dispute Tracker Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet Integration).
    - *Configuration Choices:* Appends data to worksheet `Disputes` using auto-map input data mode.
    - *Key Expressions:* Document ID mapped to `={{ $('Set Store Configuration').first().json.trackerSheetUrl }}`.
    - *Connections:* Input from `Prepare Tracker Row Data`.

---

#### 2.6 Deferred Deadline Monitoring
- **Overview:** Pauses execution if the evidence was saved as a draft, waits until the designated reminder timestamp, re-queries Stripe, and triggers a Slack warning if the dispute remains unanswered.
- **Nodes Involved:**
  - `Check if Saved as Draft`
  - `Wait Until Reminder Time`
  - `Recheck Dispute Status in Stripe`
  - `Check Submission Status`
  - `Send Deadline Reminder via Slack`
- **Node Details:**
  - **Check if Saved as Draft**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration Choices:* Checks if `autoSubmit` is false.
    - *Key Expressions:* `={{ $('Map Evidence to Stripe Fields').first().json.autoSubmit }}` (Is False).
    - *Connections:* Input from `Post Evidence to Stripe`; output to `Wait Until Reminder Time`.
  - **Wait Until Reminder Time**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Timing Control).
    - *Configuration Choices:* Resumes at specific time based on calculated `reminderAt` ISO string.
    - *Key Expressions:* `={{ $('Map Evidence to Stripe Fields').first().json.reminderAt }}`.
    - *Connections:* Input from `Check if Saved as Draft`; output to `Recheck Dispute Status in Stripe`.
  - **Recheck Dispute Status in Stripe**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request).
    - *Configuration Choices:* GET request to Stripe dispute endpoint to verify current submission status. Authenticated via Stripe API credentials.
    - *Key Expressions:* `={{ https://api.stripe.com/v1/disputes/{{ $('Extract Payment Signals').first().json.disputeId }} }}`.
    - *Connections:* Input from `Wait Until Reminder Time`; output to `Check Submission Status`.
  - **Check Submission Status**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router).
    - *Configuration Choices:* Validates that the dispute status still equals `needs_response` and `submission_count` equals `0`.
    - *Key Expressions:* References `$json.status` and `$json.evidence_details.submission_count`.
    - *Connections:* Input from `Recheck Dispute Status in Stripe`; output to `Send Deadline Reminder via Slack`.
  - **Send Deadline Reminder via Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Messaging Integration).
    - *Configuration Choices:* Posts an alarm notification to the designated Slack channel warning that the draft remains unsubmitted.
    - *Key Expressions:* Dynamic template calculations for hours remaining until deadline.
    - *Connections:* Input from `Check Submission Status`.

---

#### 2.7 Dispute Outcome Recording
- **Overview:** Processes closed dispute webhooks by formatting outcome details, updating the Google Sheets log, and broadcasting resolution statuses to Slack.
- **Nodes Involved:**
  - `Format Dispute Outcome Data`
  - `Append Outcome to Tracker`
  - `Announce Outcome in Slack`
- **Node Details:**
  - **Format Dispute Outcome Data**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation).
    - *Configuration Choices:* Extracts dispute ID, final status (`won`, `lost`, etc.), and closing date.
    - *Key Expressions:* Uses `$now.toFormat('yyyy-MM-dd')` and webhook payload data.
    - *Connections:* Input from `Route by Event Type` (Output 1); output to `Append Outcome to Tracker`.
  - **Append Outcome to Tracker**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet Integration).
    - *Configuration Choices:* Appends or updates existing records in the `Disputes` sheet matching on `Dispute ID`.
    - *Key Expressions:* Document ID bound to `trackerSheetUrl`.
    - *Connections:* Input from `Format Dispute Outcome Data`; output to `Announce Outcome in Slack`.
  - **Announce Outcome in Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Messaging Integration).
    - *Configuration Choices:* Broadcasts resolution status (trophy icon for wins, cross icon for losses) with transaction details and direct Stripe dashboard links to the Slack channel.
    - *Key Expressions:* Conditional ternary statements checking event status and zero-decimal currency arrays.
    - *Connections:* Input from `Append Outcome to Tracker`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note10` | `stickyNote` | Documentation note | None | None | ## Record dispute outcome... |
| `Sticky Note9` | `stickyNote` | Documentation note | None | None | ## Send deadline reminder... |
| `Sticky Note8` | `stickyNote` | Documentation note | None | None | ## Monitor draft status... |
| `Sticky Note7` | `stickyNote` | Documentation note | None | None | ## Notify and log draft... |
| `Sticky Note6` | `stickyNote` | Documentation note | None | None | ## Save Stripe evidence... |
| `Sticky Note5` | `stickyNote` | Documentation note | None | None | ## Draft structured evidence... |
| `Sticky Note4` | `stickyNote` | Documentation note | None | None | ## Collect customer communications... |
| `Sticky Note3` | `stickyNote` | Documentation note | None | None | ## Retrieve order context... |
| `Sticky Note2` | `stickyNote` | Documentation note | None | None | ## Load Stripe dispute details... |
| `Sticky Note1` | `stickyNote` | Documentation note | None | None | ## Receive and route event... |
| `Sticky Note` | `stickyNote` | Documentation note | None | None | ## Build Stripe dispute evidence... |
| `When Stripe Dispute Event Triggers` | `stripeTrigger` | Webhook entry point for dispute events | None | `Set Store Configuration` | Receive and route event |
| `Set Store Configuration` | `set` | Assigns store-wide configuration parameters | `When Stripe Dispute Event Triggers` | `Route by Event Type` | Receive and route event |
| `Route by Event Type` | `switch` | Branches workflow based on event type | `Set Store Configuration` | `Fetch Stripe Dispute Details`, `Format Dispute Outcome Data` | Receive and route event |
| `Fetch Stripe Dispute Details` | `httpRequest` | Fetches expanded dispute and charge object | `Route by Event Type` | `Extract Payment Signals` | Load Stripe dispute details |
| `Extract Payment Signals` | `code` | Normalizes payment signals and security checks | `Fetch Stripe Dispute Details` | `Check If Response Needed` | Load Stripe dispute details |
| `Check If Response Needed` | `filter` | Filters out non-actionable disputes | `Extract Payment Signals` | `Check WooCommerce Order` | Load Stripe dispute details |
| `Check WooCommerce Order` | `if` | Checks if order ID exists in dispute metadata | `Check If Response Needed` | `Fetch WooCommerce Order`, `Search Customer Emails in Gmail` | Retrieve order context |
| `Fetch WooCommerce Order` | `wooCommerce` | Retrieves WooCommerce order details | `Check WooCommerce Order` | `Fetch Customer Order History` | Retrieve order context |
| `Fetch Customer Order History` | `wooCommerce` | Retrieves past orders for the customer email | `Fetch WooCommerce Order` | `Search Customer Emails in Gmail` | Retrieve order context |
| `Search Customer Emails in Gmail` | `gmail` | Searches Gmail messages for customer emails | `Fetch Customer Order History`, `Check WooCommerce Order` | `Assemble Dispute Case File` | Collect customer communications |
| `Assemble Dispute Case File` | `code` | Consolidates data sources into a case file | `Search Customer Emails in Gmail` | `Draft Dispute Response with AI` | Collect customer communications |
| `Draft Dispute Response with AI` | `chainLlm` | LLM chain generating structured evidence | `Assemble Dispute Case File`, `OpenAI Chat Model`, `Define Evidence Output Format` | `Map Evidence to Stripe Fields` | Draft structured evidence |
| `OpenAI Chat Model` | `lmChatOpenAi` | Provides GPT-4o-mini model configuration | None | `Draft Dispute Response with AI` | Draft structured evidence |
| `Define Evidence Output Format` | `outputParserStructured` | Enforces JSON output structure for AI response | None | `Draft Dispute Response with AI` | Draft structured evidence |
| `Map Evidence to Stripe Fields` | `code` | Prepares Stripe evidence payload and parameters | `Draft Dispute Response with AI` | `Post Evidence to Stripe` | Save Stripe evidence |
| `Post Evidence to Stripe` | `httpRequest` | Posts evidence data to Stripe API | `Map Evidence to Stripe Fields` | `Notify Team in Slack`, `Prepare Tracker Row Data`, `Check if Saved as Draft` | Save Stripe evidence |
| `Notify Team in Slack` | `slack` | Sends notification alert to Slack channel | `Post Evidence to Stripe` | None | Notify and log draft |
| `Prepare Tracker Row Data` | `set` | Formats data for Google Sheets logging | `Post Evidence to Stripe` | `Log to Dispute Tracker Sheets` | Notify and log draft |
| `Log to Dispute Tracker Sheets` | `googleSheets` | Appends dispute row to Google Sheets | `Prepare Tracker Row Data` | None | Notify and log draft |
| `Check if Saved as Draft` | `if` | Verifies whether draft submission requires reminders | `Post Evidence to Stripe` | `Wait Until Reminder Time` | Monitor draft status |
| `Wait Until Reminder Time` | `wait` | Pauses execution until reminder timestamp | `Check if Saved as Draft` | `Recheck Dispute Status in Stripe` | Monitor draft status |
| `Recheck Dispute Status in Stripe` | `httpRequest` | Queries Stripe for current dispute status | `Wait Until Reminder Time` | `Check Submission Status` | Monitor draft status |
| `Check Submission Status` | `if` | Checks if dispute remains unsubmitted | `Recheck Dispute Status in Stripe` | `Send Deadline Reminder via Slack` | Send deadline reminder |
| `Send Deadline Reminder via Slack` | `slack` | Sends reminder alert to Slack channel | `Check Submission Status` | None | Send deadline reminder |
| `Format Dispute Outcome Data` | `set` | Formats closed dispute resolution data | `Route by Event Type` | `Append Outcome to Tracker` | Record dispute outcome |
| `Append Outcome to Tracker` | `googleSheets` | Updates or appends outcome status in Google Sheets | `Format Dispute Outcome Data` | `Announce Outcome in Slack` | Record dispute outcome |
| `Announce Outcome in Slack` | `slack` | Announces dispute outcome result in Slack | `Append Outcome to Tracker` | None | Record dispute outcome |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Stripe Trigger** node (`When Stripe Dispute Event Triggers`).
   - Configure events: `charge.dispute.created` and `charge.dispute.closed`.
   - Set up valid Stripe Webhook credentials.

2. **Add Store Configuration:**
   - Create a **Set** node (`Set Store Configuration`).
   - Add string/numeric assignments for `storeName`, `storeUrl`, `refundPolicyUrl`, `termsUrl`, `shippingPolicyUrl`, `policyDisclosure`, `autoSubmit` (boolean: `false`), `minimumDisputeAmount` (number: `0`), `emailLookbackDays` (number: `180`), `reminderHoursBeforeDeadline` (number: `48`), `slackChannel` (string: `#chargebacks`), and `trackerSheetUrl` (string).
   - Connect `When Stripe Dispute Event Triggers` to this node.

3. **Set Up Event Routing:**
   - Add a **Switch** node (`Route by Event Type`).
   - Create two rules:
     - Rule 0: Left value `={{ $('When Stripe Dispute Event Triggers').first().json.type }}`, operator equals `charge.dispute.created`.
     - Rule 1: Left value `={{ $('When Stripe Dispute Event Triggers').first().json.type }}`, operator equals `charge.dispute.closed`.
   - Connect `Set Store Configuration` to this node.

4. **Build the New Dispute Branch:**
   - **Fetch Dispute Details:** Add an **HTTP Request** node (`Fetch Stripe Dispute Details`). Method: GET, URL: `https://api.stripe.com/v1/disputes/{{ $('When Stripe Dispute Event Triggers').first().json.data.object.id }}`, Query Parameter `expand[]` = `charge`. Authenticated with Stripe API credentials. Connect Switch Output 0 here.
   - **Extract Signals:** Add a **Code** node (`Extract Payment Signals`) using the JavaScript snippet provided in the reference JSON to normalize card checks, amounts, metadata, and timestamps. Connect HTTP Request output here.
   - **Filter Responses:** Add a **Filter** node (`Check If Response Needed`) requiring `status` contains `needs_response`, `submissionCount` equals `0`, `pastDue` is `false`, and `amount` >= `minimumDisputeAmount`. Connect Code output here.

5. **Integrate WooCommerce and Gmail Context:**
   - **Order Check:** Add an **If** node (`Check WooCommerce Order`) checking if `orderId` is not empty. Connect Filter output here.
   - **Fetch Order:** Add a **WooCommerce** node (`Fetch WooCommerce Order`), resource `order`, operation `get`, order ID `={{ $json.orderId }}`. Enable error continuation (`onError: continueRegularOutput`). Connect True branch of If node here.
   - **Fetch Order History:** Add a **WooCommerce** node (`Fetch Customer Order History`), resource `order`, operation `getAll`, limit `50`, search `={{ $json.billing ? $json.billing.email : '' }}`. Connect WooCommerce Order output here.
   - **Search Gmail:** Add a **Gmail** node (`Search Customer Emails in Gmail`), operation `getAll`, limit `15`, filter query checking customer email and `newer_than` lookback window. Enable error continuation. Connect either History output or False branch of Order Check.
   - **Assemble Case File:** Add a **Code** node (`Assemble Dispute Case File`) to combine Stripe, WooCommerce, and Gmail objects into a structured case file with deterministic strengths and gaps. Connect Gmail output here.

6. **Configure OpenAI Evidence Drafting:**
   - Add an **Advanced AI / LLM Chain** node (`Draft Dispute Response with AI`).
   - Connect an **OpenAI Chat Model** sub-node (`OpenAI Chat Model`) configured with model `gpt-4o-mini` and temperature `0.2`.
   - Connect a **Structured Output Parser** sub-node (`Define Evidence Output Format`) specifying the required JSON schema for recommendations, rebuttals, and missing evidence.
   - Connect `Assemble Dispute Case File` to the main input of the LLM Chain.

7. **Format and Submit Stripe Evidence:**
   - **Map Fields:** Add a **Code** node (`Map Evidence to Stripe Fields`) to translate AI output and case files into form-urlencoded Stripe dispute parameters and compute reminder timestamps. Connect LLM Chain output here.
   - **Post Evidence:** Add an **HTTP Request** node (`Post Evidence to Stripe`). Method: POST, URL: `https://api.stripe.com/v1/disputes/{{ $('Extract Payment Signals').first().json.disputeId }}`, Body Content-Type: `form-urlencoded`, Body: `={{ $json.formBody }}`. Authenticated with Stripe API credentials. Connect Code output here.

8. **Add Post-Submission Notifications and Logging:**
   - **Slack Notification:** Add a **Slack** node (`Notify Team in Slack`) posting formatted dispute summaries to the configured channel. Connect Post Evidence output here.
   - **Prepare Tracker Data:** Add a **Set** node (`Prepare Tracker Row Data`) mapping fields for the spreadsheet. Connect Post Evidence output here.
   - **Google Sheets Log:** Add a **Google Sheets** node (`Log to Dispute Tracker Sheets`), operation `append`, sheet name `Disputes`, document ID mapped to configuration URL. Connect Set node output here.

9. **Configure Deadline Monitoring:**
   - **Check Draft Status:** Add an **If** node (`Check if Saved as Draft`) checking `autoSubmit` is false. Connect Post Evidence output here.
   - **Wait Node:** Add a **Wait** node (`Wait Until Reminder Time`), resume at specific time `={{ $('Map Evidence to Stripe Fields').first().json.reminderAt }}`. Connect True branch here.
   - **Recheck Stripe:** Add an **HTTP Request** node (`Recheck Dispute Status in Stripe`) performing a GET request to the Stripe dispute endpoint. Connect Wait node output here.
   - **Check Status:** Add an **If** node (`Check Submission Status`) verifying `status` contains `needs_response` and `submission_count` equals `0`. Connect HTTP Request output here.
   - **Send Reminder:** Add a **Slack** node (`Send Deadline Reminder via Slack`) posting the approaching deadline warning. Connect True branch here.

10. **Build Dispute Closed Branch:**
    - **Format Outcome:** Add a **Set** node (`Format Dispute Outcome Data`) extracting dispute ID, status, and close date from webhook payload `{{ $('When Stripe Dispute Event Triggers').first().json.data.object }}`. Connect Switch Output 1 here.
    - **Update Sheets:** Add a **Google Sheets** node (`Append Outcome to Tracker`), operation `appendOrUpdate`, matching columns `Dispute ID`. Connect Set node output here.
    - **Announce Outcome:** Add a **Slack** node (`Announce Outcome in Slack`) posting win/loss resolution details. Connect Google Sheets output here.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Workflow Title** | Build Stripe dispute evidence from WooCommerce orders, Gmail, and OpenAI |
| **Stripe Test Mode Card** | Use card `4000000000000259` in Stripe test mode to consistently trigger test disputes. |
| **Execution Requirement** | The Wait node requires an n8n instance that persists waiting executions (n8n Cloud or self-hosted with database storage). |