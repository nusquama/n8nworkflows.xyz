Extract Amazon invoice links nightly with Airtop, OpenAI, and Google Sheets

https://n8nworkflows.xyz/workflows/extract-amazon-invoice-links-nightly-with-airtop--openai--and-google-sheets-18618


# Extract Amazon invoice links nightly with Airtop, OpenAI, and Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of Amazon order IDs and invoice download links using a cloud browser, parses the unstructured page data via an LLM, cleans tracking parameters, and records the results into Google Sheets. It runs on a nightly schedule to minimize manual bookkeeping.

The logic is categorized into the following functional blocks:
- **1.1 Trigger & Browser Initialization:** Schedules the nightly execution and initializes an authenticated Airtop cloud browser session to open the Amazon order history page.
- **1.2 Content Scrape & AI Extraction:** Waits for the DOM to render, scrapes the visible text, and uses an OpenAI-powered AI Agent with a structured output parser and auto-fix capability to extract order IDs and invoice URLs.
- **1.3 Data Filtering, Cleansing & Storage:** Validates extracted links, strips Amazon tracking parameters using a custom JavaScript code block, appends records to Google Sheets, and cleanly terminates the cloud browser session.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Trigger & Browser Initialization

- **Overview:** Initiates the workflow on a nightly schedule, provisions an Airtop cloud browser session leveraging a pre-authenticated persistent profile, navigates to the Amazon order history page, and pauses briefly to allow complete rendering.
- **Nodes Involved:** 
  - `Run every night`
  - `Start browser session`
  - `Open Amazon order history`
  - `Wait for page to load`

##### Node Details:
- **Run every night**
  - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node)
  - *Configuration:* Set to trigger daily at hour 2 (2:00 AM).
  - *Input Connections:* None (Root entry point).
  - *Output Connections:* Main output connects to `Start browser session`.
  - *Edge Cases / Failures:* Missed triggers if the n8n instance is offline at 2:00 AM.

- **Start browser session**
  - *Type & Technical Role:* `n8n-nodes-base.airtop` (Action Node)
  - *Configuration:* Profile name configured (parameterized placeholder `YOUR_AIRTOP_PROFILE_NAME`), timeout set to 8 minutes, and `saveProfileOnTermination` enabled to preserve cookies/login state.
  - *Input Connections:* `Run every night`.
  - *Output Connections:* Main output connects to `Open Amazon order history`.
  - *Credentials:* Airtop API.
  - *Edge Cases / Failures:* API authentication failure, invalid profile name, or session timeout if queue times are high.

- **Open Amazon order history**
  - *Type & Technical Role:* `n8n-nodes-base.airtop` (Action Node)
  - *Configuration:* Resource type set to `window`, target URL set to `https://www.amazon.com/gp/your-account/order-history`, live view enabled.
  - *Input Connections:* `Start browser session`.
  - *Output Connections:* Main output connects to `Wait for page to load`.
  - *Credentials:* Airtop API.
  - *Edge Cases / Failures:* Network errors, Amazon bot-detection challenges, or redirection to login pages if session cookies have expired.

- **Wait for page to load**
  - *Type & Technical Role:* `n8n-nodes-base.wait` (Utility Node)
  - *Configuration:* Fixed delay of 15 seconds.
  - *Input Connections:* `Open Amazon order history`.
  - *Output Connections:* Main output connects to `Scrape the order page`.
  - *Edge Cases / Failures:* 15 seconds might be insufficient during high latency, resulting in incomplete text scraping.

---

#### Block 1.2: Content Scrape & AI Extraction

- **Overview:** Scrapes the raw text content from the active browser window and processes it through an LLM agent configured with strict output schemas and self-healing auto-fix parsers to reliably extract order identifiers and invoice links.
- **Nodes Involved:**
  - `Scrape the order page`
  - `Find order IDs and invoice links`
  - `Chat model - Extraction`
  - `Auto-fixing Output Parser`
  - `Output Parser - Order and link`
  - `Chat model - Parser auto-fix`

##### Node Details:
- **Scrape the order page**
  - *Type & Technical Role:* `n8n-nodes-base.airtop` (Action Node)
  - *Configuration:* Resource set to `extraction`, operation set to `scrape`.
  - *Input Connections:* `Wait for page to load`.
  - *Output Connections:* Main output connects to `Find order IDs and invoice links`.
  - *Credentials:* Airtop API.
  - *Edge Cases / Failures:* Empty string returns if the page fails to render text content correctly.

- **Find order IDs and invoice links**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent Node)
  - *Configuration:* Prompt type set to define. Reads text from `={{ $json.data.modelResponse.scrapedContent.text }}`.
  - *Input Connections:* `Scrape the order page` (Main), `Chat model - Extraction` (AI Model), `Auto-fixing Output Parser` (AI Output Parser).
  - *Output Connections:* Main output connects to `Found an invoice link?`.
  - *Edge Cases / Failures:* Token length overflows if order history page contains excessive markup or reviews.

- **Chat model - Extraction**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model Sub-Node)
  - *Configuration:* Model set to `gpt-4.1-mini`.
  - *Input Connections:* Connected to `Find order IDs and invoice links`.
  - *Credentials:* Open AI (`Open AI - Misc for testing`).
  - *Edge Cases / Failures:* Rate limiting (`429 Too Many Requests`), upstream API outages, or invalid API keys.

- **Auto-fixing Output Parser**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.outputParserAutofixing` (Output Parser Sub-Node)
  - *Configuration:* Wraps base parser and utilizes an auxiliary model to correct structural deviations.
  - *Input Connections:* Connected to `Find order IDs and invoice links` (AI Output Parser), `Output Parser - Order and link` (Base Parser), `Chat model - Parser auto-fix` (AI Model).

- **Output Parser - Order and link**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Structured Output Parser Sub-Node)
  - *Configuration:* Enforces JSON schema example matching an array of objects containing `"Order Id"` and `"Invoice URL"`.
  - *Input Connections:* Connected to `Auto-fixing Output Parser`.

- **Chat model - Parser auto-fix**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model Sub-Node)
  - *Configuration:* Model set to `gpt-4.1-mini`.
  - *Input Connections:* Connected to `Auto-fixing Output Parser`.
  - *Credentials:* Open AI (`Open AI - Misc for testing`).
  - *Edge Cases / Failures:* Fails if the model cannot map unstructured output to the required JSON schema after correction attempts.

---

#### Block 1.3: Data Filtering, Cleansing & Storage

- **Overview:** Validates extracted payload structures, strips unnecessary tracking parameters from invoice URLs via custom JavaScript, appends clean entries to a Google Sheets document, and securely terminates the cloud browser session.
- **Nodes Involved:**
  - `Found an invoice link?`
  - `Clean up the invoice links`
  - `Add links to the sheet`
  - `Close browser session`

##### Node Details:
- **Found an invoice link?**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control Node)
  - *Configuration:* Condition evaluates whether `={{$json["output"][0]["Invoice URL"]}}` is not empty (`notEmpty`).
  - *Input Connections:* `Find order IDs and invoice links`.
  - *Output Connections:* True branch connects to `Clean up the invoice links`.
  - *Edge Cases / Failures:* Structure mutations or index out-of-bounds if the array is unexpectedly empty.

- **Clean up the invoice links**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Code Node)
  - *Configuration:* JavaScript execution splitting URL strings at the `&` delimiter to isolate base invoice links:
    ```javascript
    const rows = items[0].json.output || [];
    return rows.map(obj => {
      const cleanUrl = String(obj["Invoice URL"] || '').split("&")[0];
      return {
        json: {
          order_id: obj["Order Id"],
          invoice_url: cleanUrl
        }
      };
    });
    ```
  - *Input Connections:* `Found an invoice link?` (True branch).
  - *Output Connections:* Main output connects to `Add links to the sheet`.
  - *Edge Cases / Failures:* Runtime reference errors if input payload format changes unexpectedly.

- **Add links to the sheet**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Action Node)
  - *Configuration:* Operation set to `append`. Sheet name set to `Amazon Invoices`. Document ID configured via placeholder `YOUR_GOOGLE_SHEET_ID`. Mappings:
    - `Order ID` = `={{ $json.order_id }}`
    - `Invoice URL ` = `={{ $json.invoice_url }}`
    - `Date` = `={{ $now.toFormat("dd LLL yyyy") }}`
  - *Input Connections:* `Clean up the invoice links`.
  - *Output Connections:* Main output connects to `Close browser session`.
  - *Credentials:* Google Sheets account.
  - *Edge Cases / Failures:* Sheet permission issues, quota limits, or missing target column headers (`Order ID`, `Invoice URL `, `Date`).

- **Close browser session**
  - *Type & Technical Role:* `n8n-nodes-base.airtop` (Action Node)
  - *Configuration:* Operation set to `terminate`. Session ID referenced dynamically via `={{ $('Scrape the order page').item.json.sessionId }}`.
  - *Input Connections:* `Add links to the sheet`.
  - *Output Connections:* None (Terminal node).
  - *Credentials:* Airtop API.
  - *Edge Cases / Failures:* Fails if session ID is missing or already expired/terminated.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note - Overview** | `n8n-nodes-base.stickyNote` | Documentation and metadata | None | None | # Scrape Amazon invoice links into Google Sheets... |
| **Sticky Note - Section 1** | `n8n-nodes-base.stickyNote` | Section 1 description | None | None | ## 1. Open your Amazon orders... |
| **Sticky Note - Section 2** | `n8n-nodes-base.stickyNote` | Section 2 description | None | None | ## 2. Read the invoice links... |
| **Sticky Note - Section 3** | `n8n-nodes-base.stickyNote` | Section 3 description | None | None | ## 3. Save and clean up... |
| **Run every night** | `n8n-nodes-base.scheduleTrigger` | Starts workflow daily at 2:00 AM | None | Start browser session | |
| **Start browser session** | `n8n-nodes-base.airtop` | Initializes persistent cloud browser profile | Run every night | Open Amazon order history | ## 1. Open your Amazon orders... |
| **Open Amazon order history** | `n8n-nodes-base.airtop` | Navigates to Amazon order history page | Start browser session | Wait for page to load | ## 1. Open your Amazon orders... |
| **Wait for page to load** | `n8n-nodes-base.wait` | Pauses execution for page render | Open Amazon order history | Scrape the order page | ## 1. Open your Amazon orders... |
| **Scrape the order page** | `n8n-nodes-base.airtop` | Extracts raw text from active DOM | Wait for page to load | Find order IDs and invoice links | ## 2. Read the invoice links... |
| **Find order IDs and invoice links** | `@n8n/n8n-nodes-langchain.agent` | Extracts structured orders/invoices via AI | Scrape the order page | Found an invoice link? | ## 2. Read the invoice links... |
| **Chat model - Extraction** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM backend for extraction agent | None | Find order IDs and invoice links | ## 2. Read the invoice links... |
| **Auto-fixing Output Parser** | `@n8n/n8n-nodes-langchain.outputParserAutofixing` | Validates and auto-corrects output JSON | None | Find order IDs and invoice links | ## 2. Read the invoice links... |
| **Output Parser - Order and link** | `@n8n/n8n-nodes-langchain.outputParserStructured` | Defines target schema for structured data | None | Auto-fixing Output Parser | ## 2. Read the invoice links... |
| **Chat model - Parser auto-fix** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM backend for parser recovery | None | Auto-fixing Output Parser | ## 2. Read the invoice links... |
| **Found an invoice link?** | `n8n-nodes-base.if` | Validates presence of invoice URLs | Find order IDs and invoice links | Clean up the invoice links | ## 2. Read the invoice links... |
| **Clean up the invoice links** | `n8n-nodes-base.code` | Strips tracking parameters from URLs | Found an invoice link? | Add links to the sheet | ## 3. Save and clean up |
| **Add links to the sheet** | `n8n-nodes-base.googleSheets` | Appends cleaned records to Google Sheet | Clean up the invoice links | Close browser session | ## 3. Save and clean up |
| **Close browser session** | `n8n-nodes-base.airtop` | Terminates active cloud browser session | Add links to the sheet | None | ## 3. Save and clean up |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** (`Run every night`). Set the interval rule to trigger at hour `2`.
2. **Initialize Browser Session:**
   - Add an **Airtop** node (`Start browser session`). Set credentials (Airtop API). Set profile name to your persistent profile value, timeout to `8`, and enable `saveProfileOnTermination`.
   - Connect `Run every night` to `Start browser session`.
3. **Open Target URL:**
   - Add an **Airtop** node (`Open Amazon order history`). Set resource to `window`, URL to `https://www.amazon.com/gp/your-account/order-history`, and enable live view.
   - Connect `Start browser session` to `Open Amazon order history`.
4. **Add Render Delay:**
   - Add a **Wait** node (`Wait for page to load`). Set amount to `15` seconds.
   - Connect `Open Amazon order history` to `Wait for page to load`.
5. **Scrape Page Content:**
   - Add an **Airtop** node (`Scrape the order page`). Set resource to `extraction` and operation to `scrape`.
   - Connect `Wait for page to load` to `Scrape the order page`.
6. **Configure AI Extraction Agent:**
   - Add an **AI Agent** node (`Find order IDs and invoice links`). Set text input to `={{ $json.data.modelResponse.scrapedContent.text }}` and enable output parser.
   - Connect `Scrape the order page` to `Find order IDs and invoice links`.
7. **Configure AI Sub-Nodes:**
   - Add an **OpenAI Chat Model** (`Chat model - Extraction`). Set model to `gpt-4.1-mini`, configure OpenAI API credentials, and connect to the AI Agent's language model input.
   - Add a **Structured Output Parser** (`Output Parser - Order and link`). Provide the JSON schema example:
     ```json
     [
       {"Order Id": "Order_Id", "Invoice URL": "Invoice_URL"}
     ]
     ```
   - Add an **OpenAI Chat Model** (`Chat model - Parser auto-fix`). Set model to `gpt-4.1-mini`, configure credentials, and connect to the auto-fixing parser.
   - Add an **Auto-fixing Output Parser** (`Auto-fixing Output Parser`). Connect the structured parser and auto-fix chat model to it, then link it to the AI Agent's output parser input.
8. **Add Conditional Logic:**
   - Add an **If** node (`Found an invoice link?`). Configure a condition where `={{$json["output"][0]["Invoice URL"]}}` is `notEmpty`.
   - Connect `Find order IDs and invoice links` to `Found an invoice link?`.
9. **Add Link Cleansing Code:**
   - Add a **Code** node (`Clean up the invoice links`). Paste the JavaScript code snippet to split strings by `&` and map order IDs and cleaned URLs.
   - Connect the `true` output of `Found an invoice link?` to `Clean up the invoice links`.
10. **Configure Google Sheets Integration:**
    - Add a **Google Sheets** node (`Add links to the sheet`). Configure operation to `append`, select your target spreadsheet, and map columns (`Order ID`, `Invoice URL `, `Date` using `={{ $now.toFormat("dd LLL yyyy") }}`). Configure Google Sheets credentials.
    - Connect `Clean up the invoice links` to `Add links to the sheet`.
11. **Close Browser Session:**
    - Add an **Airtop** node (`Close browser session`). Set operation to `terminate` and session ID to `={{ $('Scrape the order page').item.json.sessionId }}`.
    - Connect `Add links to the sheet` to `Close browser session`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Airtop Integration | Requires an active Airtop API account and a pre-configured profile authenticated against Amazon. |
| OpenAI API Usage | Requires valid OpenAI credentials with access to `gpt-4.1-mini` or compatible chat models. |