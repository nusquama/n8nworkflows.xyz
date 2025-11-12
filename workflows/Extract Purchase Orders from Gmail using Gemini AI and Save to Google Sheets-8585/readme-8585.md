Extract Purchase Orders from Gmail using Gemini AI and Save to Google Sheets

https://n8nworkflows.xyz/workflows/extract-purchase-orders-from-gmail-using-gemini-ai-and-save-to-google-sheets-8585


# Extract Purchase Orders from Gmail using Gemini AI and Save to Google Sheets

---

## 1. Workflow Overview

This workflow automates the extraction of purchase order data from unread Gmail messages and records the structured information into a Google Sheet. It leverages Google Gemini AI for advanced natural language processing to interpret email contents, detect purchase order details, enrich them with product metadata from a Google Sheet, and normalize delivery dates into ISO calendar weeks.

The workflow is logically divided into the following blocks:

- **1.1 Email Retrieval and Filtering:** Periodically fetches unread emails from Gmail, filtering them by subject to isolate relevant purchase order requests.
- **1.2 Email Content Validation:** Checks if the filtered emails contain attachments or only textual content to determine processing suitability.
- **1.3 AI Processing and Data Extraction:** Uses a Gemini AI-powered agent to parse the email body, extract purchase order details, normalize dates, and query product metadata from Google Sheets.
- **1.4 Data Transformation and Insertion:** Parses the AI output JSON into individual records and appends them as rows into the target Google Sheet with predefined columns.

This structure ensures continuous monitoring of incoming purchase order emails, intelligent extraction of relevant details regardless of format or language, and seamless integration with Google Sheets for record-keeping.

---

## 2. Block-by-Block Analysis

### 2.1 Email Retrieval and Filtering

**Overview:**  
This block triggers the workflow every minute to fetch up to 100 unread Gmail messages received within the last day. It then filters emails whose subject contains keywords related to purchase orders ("Marketing" or "Buchungsanfrage").

**Nodes Involved:**  
- Cron  
- Get many messages (Gmail)  
- Filter emails  
- If  

**Node Details:**

- **Cron**  
  - Type: Trigger node  
  - Configuration: Triggers workflow every minute  
  - Inputs: None (trigger)  
  - Outputs: Triggers "Get many messages" node  
  - Edge cases: Potential delays in trigger execution; ensure server time sync  
  - Notes: Controls the workflow frequency to avoid API rate limits  

- **Get many messages (Gmail)**  
  - Type: Gmail API node  
  - Configuration:  
    - Operation: getAll messages  
    - Limit: 100 messages max  
    - Filters: unread messages received after yesterday (calculated dynamically with expression `$today.minus({ days: 1 }).toISODate()`)  
    - Options: download attachments enabled  
  - Inputs: Trigger from Cron  
  - Outputs: Array of Gmail messages (including attachments)  
  - Edge cases: Gmail API quota exceeded; network errors; no messages found  
  - Expressions: Dynamic date filter ensures only recent emails are processed  

- **Filter emails**  
  - Type: Filter node  
  - Configuration: Checks if the email subject contains either "Marketing" or "Buchungsanfrage"  
  - Inputs: Messages from Gmail node  
  - Outputs: Passes only filtered messages onward  
  - Edge cases: Case sensitivity managed; may miss emails with different subject phrasing  

- **If**  
  - Type: Conditional node  
  - Configuration: Checks existence of email attachments (binary data) to branch processing logic  
  - Inputs: Filtered emails  
  - Outputs:  
    - True branch (emails with attachments) is unused here (empty)  
    - False branch (no attachments) continues to AI processing  
  - Edge cases: Some emails may not have attachments but contain purchase orders in body text  

---

### 2.2 Email Content Validation

**Overview:**  
This block verifies whether the email contains attachments to decide if purchase order extraction should proceed from the email body text.

**Nodes Involved:**  
- If  

**Node Details:**

- **If** (same as above)  
  - Role: Discriminates emails based on presence of attachments, focusing here on those without attachments for text parsing  
  - Inputs: Filtered emails  
  - Outputs: Emails without attachments proceed to AI Agent for text extraction  
  - Edge cases: Emails with attachments are currently not processed further, which may miss some orders  

---

### 2.3 AI Processing and Data Extraction

**Overview:**  
This critical block uses the Gemini AI Agent to analyze the email text, extract detailed purchase order information (products, quantities, dates), normalize calendar weeks, enrich data with product details from Google Sheets, and produce a structured JSON array output.

**Nodes Involved:**  
- Set final output keys  
- AI Agent  
- Google Gemini Chat Model (used internally by AI Agent)  
- Get row(s) in sheet in Google Sheets (used internally by AI Agent)  
- Sticky Note2 (documentation)  

**Node Details:**

- **Set final output keys**  
  - Type: Set node  
  - Configuration: Defines an array of keys (`final_json_keys`) to guide the AI output format, including fields like "Laufende Nummer", "Lieferant", "Kalenderwoche Start", etc.  
  - Inputs: Emails without attachments from "If" node  
  - Outputs: Passes email text and keys to AI Agent  
  - Edge cases: None significant; static key array  

- **AI Agent**  
  - Type: LangChain AI Agent node (Gemini AI-powered)  
  - Configuration:  
    - Role: Expert AI specialized in reading emails and extracting purchase order details, normalizing dates into calendar weeks, and enriching data by querying Google Sheets for product info  
    - Input: Email text from "If" node  
    - Instructions: Detailed prompt specifying extraction rules, date normalization logic, output JSON format, language awareness (accepts any email language), and excluding sensitive customer info  
  - Inputs: Email text and keys array  
  - Outputs: JSON string containing array of purchase order items  
  - Edge cases:  
    - AI may fail to extract structured data if email format is unusual  
    - Date parsing ambiguities (e.g., vague references) may cause incorrect calendar week mapping  
    - Google Sheets query failures if product data missing or credentials invalid  
  - Sub-nodes (internal to AI Agent):  
    - Google Gemini Chat Model: Handles AI language model inference  
    - Get row(s) in sheet in Google Sheets: Queries product details for enrichment  

---

### 2.4 Data Transformation and Insertion

**Overview:**  
This block parses the JSON array returned by the AI Agent into individual items and appends each as a new row in the designated Google Sheet.

**Nodes Involved:**  
- Reformatted to upload in Google sheet  
- Append row in sheet  
- Sticky Note3 (documentation)  

**Node Details:**

- **Reformatted to upload in Google sheet**  
  - Type: Code node (JavaScript)  
  - Configuration:  
    - Cleans AI output by removing markdown JSON code fences (```json ...```)  
    - Parses the cleaned string into JSON array  
    - Converts each JSON object into a separate item for n8n  
  - Inputs: AI Agent output (string)  
  - Outputs: Array of JSON objects, each representing a purchase order line  
  - Edge cases:  
    - JSON parsing failure if AI output malformed  
    - Empty or null input from AI Agent  

- **Append row in sheet**  
  - Type: Google Sheets node  
  - Configuration:  
    - Operation: Append row  
    - Target: Configured Google Sheet and Sheet Name (dynamic, values replaced at runtime)  
    - Columns mapped explicitly from JSON keys to Google Sheet columns (e.g., "Marke", "Produkt", "Kosten", "Kalenderwoche Start")  
    - Credentials: Google Sheets OAuth2 configured  
  - Inputs: Reformatted purchase order items  
  - Outputs: Confirmation of row append operation  
  - Edge cases:  
    - Google Sheets API quota or permission issues  
    - Column mismatch or missing fields causing insertion errors  

---

## 3. Summary Table

| Node Name                        | Node Type                      | Functional Role                                  | Input Node(s)             | Output Node(s)                 | Sticky Note                                                                                                                             |
|---------------------------------|--------------------------------|-------------------------------------------------|---------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Cron                            | Trigger                       | Triggers workflow every minute                   | None                      | Get many messages             | ## Scan Email on every minute and read new emails. **Get new emails frequently and filter them which has purchase order**              |
| Get many messages               | Gmail API                     | Fetches unread emails from Gmail received in last day | Cron                      | Filter emails                 |                                                                                                                                         |
| Filter emails                  | Filter                       | Filters emails by subject keywords ("Marketing", "Buchungsanfrage") | Get many messages          | If                           |                                                                                                                                         |
| If                             | Conditional                  | Checks if email has attachments or not           | Filter emails             | Set final output keys (false branch) | ## Check the document has attachment **Check for email without attachment. To read purchase order from the email body**                 |
| Set final output keys          | Set                          | Defines keys for AI output JSON structure         | If                        | AI Agent                     |                                                                                                                                         |
| AI Agent                      | LangChain AI Agent           | Extracts purchase order data, normalizes dates, enriches from Google Sheets | Set final output keys       | Reformatted to upload in Google sheet | ## AI Agent to read and summarize the order.                                                                                             |
| Google Gemini Chat Model       | AI Language Model (internal) | Handles AI inference for text processing          | AI Agent (internal)        | AI Agent (internal)           |                                                                                                                                         |
| Get row(s) in sheet in Google Sheets | Google Sheets Tool (internal) | Queries product details for enrichment             | AI Agent (internal)        | AI Agent (internal)           |                                                                                                                                         |
| Reformatted to upload in Google sheet | Code                         | Parses AI JSON output into separate items         | AI Agent                   | Append row in sheet           | ## Append purchase order to Google sheet                                                                                                |
| Append row in sheet            | Google Sheets                | Appends extracted purchase orders as rows in Google Sheet | Reformatted to upload in Google sheet | None                        |                                                                                                                                         |
| Sticky Note                   | Sticky Note                  | Documentation note (disabled)                      | None                      | None                         | ## Scan Email on every minute and read new emails. **Get new emails frequently and filter them which has purchase order**              |
| Sticky Note1                  | Sticky Note                  | Documentation note                                | None                      | None                         | ## Check the document has attachment **Check for email without attachment. To read purchase order from the email body**                 |
| Sticky Note2                  | Sticky Note                  | Documentation note                                | None                      | None                         | ## AI Agent to read and summarize the order.                                                                                            |
| Sticky Note3                  | Sticky Note                  | Documentation note                                | None                      | None                         | ## Append purchase order to Google sheet                                                                                                |
| Sticky Note4                  | Sticky Note                  | Documentation note                                | None                      | None                         | ## 📧 Email Reading & Purchase Order Creation (AI-powered) **✨ What it does** - ⏱ Reads unread emails every minute - 🎯 Filters emails based on **Subject** - 🤖 Uses Gemini AI to summarize emails & extract purchase order details - 📊 Appends purchase order data to Google Sheets **🛠 Requirements** - 📩 Gmail account access - 🔑 Gemini AI credentials - 📑 Google Sheet with purchase order headers **⚙️ Setup Instructions** - 🔗 Set up Google Sheets & Gmail credentials - 📝 Configure subject filter - 🤝 Connect Gemini AI credentials - 📂 Create Google Sheet with order headers |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Cron Node**  
   - Type: Cron  
   - Set to trigger every minute (under Trigger Times, add an item with mode "everyMinute")  
   - This node starts the workflow periodically to check new emails.

2. **Add Gmail Node (Get many messages)**  
   - Type: Gmail  
   - Operation: "Get All"  
   - Limit: 100  
   - Filters:  
     - Read Status: unread  
     - Received After: set to expression `={{ $today.minus({ days: 1 }).toISODate() }}` to get emails from the last day  
   - Options: Download Attachments enabled  
   - Connect Cron output to this node input.

3. **Add Filter Node (Filter emails)**  
   - Type: Filter  
   - Condition: Check if email subject contains either "Marketing" or "Buchungsanfrage" (case sensitive)  
   - Connect "Get many messages" output to this filter node.

4. **Add If Node**  
   - Type: If  
   - Condition: Check if `binary` data exists on the item (to detect attachments) using expression: `{{$node["Filter emails"].item.binary}}` with operator "exists"  
   - Connect Filter emails output to If node.

5. **Add Set Node (Set final output keys)**  
   - Type: Set  
   - Create a new field named `final_json_keys` of type Array  
   - Value:  
     ```json
     [
       "Laufende Nummer",
       "Lieferant",
       "Lieferanten-Nr",
       "Marke",
       "Marken-Nr",
       "Kalenderwoche\nStart",
       "Kalenderwoche\nEnde",
       "Marketing Status",
       "Paket",
       "Produkt",
       "Länder-Aktivierung",
       "Kosten"
     ]
     ```  
   - Connect the "false" output branch of the If node (emails without attachments) to this Set node.

6. **Add LangChain AI Agent Node (AI Agent)**  
   - Type: LangChain Agent (requires LangChain integration)  
   - Role: Expert AI for purchase order extraction and enrichment with Google Sheets data  
   - Input Text: Use the email text extracted from the "If" node item JSON (e.g., `={{ $json.text }}` or appropriate field containing email body)  
   - Prompt: Use the provided prompt specifying tasks, date normalization rules, output format as JSON array  
   - Configure AI language model as Google Gemini Chat Model node (linked inside the AI Agent)  
   - Configure Google Sheets Tool node inside AI Agent to query product details (set documentId and sheetName)  
   - Connect Set final output keys output to this AI Agent node input.

7. **Add Code Node (Reformatted to upload in Google sheet)**  
   - Type: Code (JavaScript)  
   - Paste the following code to clean and parse AI output JSON:
     ```javascript
     const input = items[0].json.output;
     const cleaned = input.replace(/```json|```/g, '').trim();
     let parsed;
     try {
       parsed = JSON.parse(cleaned);
     } catch (error) {
       throw new Error(`Failed to parse JSON: ${error.message}\nRaw input: ${cleaned}`);
     }
     return parsed.map(obj => ({ json: obj }));
     ```
   - Connect AI Agent output to this Code node.

8. **Add Google Sheets Node (Append row in sheet)**  
   - Type: Google Sheets  
   - Operation: Append row  
   - Configure credentials with Google Sheets OAuth2 credentials  
   - Set Document ID and Sheet Name to target purchase order sheet  
   - Map columns explicitly: Map from JSON keys to Sheet columns as per workflow field mapping (e.g., `Marke: {{$json.Marke}}`, `Paket: {{$json.Paket}}`, etc.)  
   - Connect Code node output to this Append row node.

9. **Add Sticky Notes (optional for documentation)**  
   - Add sticky notes to relevant parts for explanation and comments as per original workflow.

10. **Test and activate the workflow**  
    - Validate Gmail and Google Sheets credentials  
    - Ensure AI Agent has access to Gemini AI and Google Sheets Tool configured with correct document IDs  
    - Save and activate the workflow for periodic execution

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                       | Context or Link                                                                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 📧 Email Reading & Purchase Order Creation (AI-powered) - Reads unread emails every minute - Filters emails by subject - Uses Gemini AI for summarization and extraction - Appends data to Google Sheets. Setup requires Gmail, Gemini AI, and Google Sheets credentials. | Sticky Note4 content included in workflow documentation.                                                                                                        |
| AI prompt includes detailed instructions on date normalization to ISO calendar weeks, handling direct week references (e.g., KW36) and vague references (e.g., end of October), ensuring consistent output.                                           | Critical for understanding AI processing expectations and output format.                                                                                        |
| Uses Google Sheets Tool integration inside AI Agent to enrich products with metadata from configured sheet, supporting accurate purchase order records.                                                                                         | This sub-integration requires proper Google Sheets document and sheet name configuration.                                                                        |
| The workflow currently only processes emails without attachments, ignoring those with attachments which may contain purchase orders in other formats (e.g., PDFs). Consider enhancing with attachment parsing for completeness.                       | Potential future improvement area.                                                                                                                              |

---

*Disclaimer: The provided content is exclusively derived from an automated workflow created using n8n, respecting all current content policies without illegal or protected elements. All processed data is legal and publicly available.*