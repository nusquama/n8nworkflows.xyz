Validate Leads Email Addresses with Hunter.io and Google Sheets

https://n8nworkflows.xyz/workflows/validate-leads-email-addresses-with-hunter-io-and-google-sheets-6992


# Validate Leads Email Addresses with Hunter.io and Google Sheets

### 1. Workflow Overview

This workflow automates the validation of lead email addresses by fetching them from a Google Sheets source, verifying each email through the Hunter.io Email Verifier API, and then writing the enriched verification results back to a Google Sheets output. It is designed for sales, marketing, or lead generation teams who want to maintain clean and validated contact lists.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Scheduled trigger initiates the workflow, which fetches raw email and contact data from a Google Sheets input spreadsheet.
- **1.2 Data Cleaning & Verification:** Cleans and prepares email data, filtering out invalid entries, and sends each email to Hunter.io for verification.
- **1.3 Post-Processing & Output:** Parses Hunter.io’s response, formats verification metadata into a human-friendly summary, and appends or updates the results into a Google Sheets output spreadsheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
This block starts the workflow automatically on a schedule and retrieves the raw lead data from a designated Google Sheets document. The sheet is expected to contain columns such as Email, FirstName, LastName, and Company.

- **Nodes Involved:**  
  - ⏯️ Triggers Everyday  
  - 📥 Fetch Emails from Source Sheet

- **Node Details:**

  - **⏯️ Triggers Everyday**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow at a fixed interval (every day).  
    - Configuration: Default interval with no specific time configuration (runs daily).  
    - Connections: Outputs to "📥 Fetch Emails from Source Sheet".  
    - Edge Cases: If the scheduler fails or the system is down, the workflow won’t trigger. No manual fallback is configured here.

  - **📥 Fetch Emails from Source Sheet**  
    - Type: Google Sheets node  
    - Role: Reads data from the input spreadsheet (Sheet1) specified by a document ID parameter.  
    - Configuration:  
      - Document ID: Parameterized as `{{YOUR_INPUT_SHEET_ID}}`.  
      - Sheet Name: Uses `gid=0` targeting the first sheet.  
      - Credentials: Uses OAuth2 for Google Sheets API access.  
    - Key Expressions: Retrieves all rows, expecting fields like Email, FirstName, LastName, Company.  
    - Input: Trigger node output.  
    - Output: Provides raw rows for cleaning.  
    - Edge Cases:  
      - Invalid or missing document ID would cause failure.  
      - OAuth token expiration or permission errors may cause API access failures.  
      - Empty or malformed sheets result in no data or errors downstream.

#### 1.2 Data Cleaning & Verification

- **Overview:**  
Filters and prepares email addresses by discarding invalid entries, extracting relevant contact fields, then sends each email to Hunter.io’s Email Verifier API for validation and enrichment.

- **Nodes Involved:**  
  - 🧹 Clean & Prepare Email Data  
  - 🕵️‍♂️ Hunter.io Email Verifier

- **Node Details:**

  - **🧹 Clean & Prepare Email Data**  
    - Type: Code node (JavaScript)  
    - Role: Processes raw input rows, extracts valid emails (containing '@'), and collects associated contact details.  
    - Configuration:  
      - Script loops through all input items, filters out invalid emails, and constructs a clean JSON object with email, first name, last name, company, and original row index for tracking.  
    - Input: Raw rows from Google Sheets.  
    - Output: Array of cleaned email objects for verification.  
    - Edge Cases:  
      - Rows lacking an email or with malformed emails are discarded silently.  
      - Missing contact fields default to empty strings.  
      - Risk if email data is under a different column name than expected.

  - **🕵️‍♂️ Hunter.io Email Verifier**  
    - Type: HTTP Request  
    - Role: Calls Hunter.io API `/v2/email-verifier` endpoint to validate each email.  
    - Configuration:  
      - URL: `https://api.hunter.io/v2/email-verifier`  
      - Query Parameters: `email` (from cleaned data), `api_key` (parameterized as `{{HUNTER_API_KEY}}`)  
      - Method: GET  
    - Input: Cleaned emails from previous code node.  
    - Output: Raw Hunter.io API responses with verification metadata.  
    - Edge Cases:  
      - API key invalid or missing causes authentication failures.  
      - API rate limits or network errors can cause timeouts or errors.  
      - Emails not recognized by Hunter.io may return minimal or empty data.

#### 1.3 Post-Processing & Output

- **Overview:**  
This block processes the Hunter.io responses to extract verification results, creates a human-readable summary, and writes the enriched data back to a Google Sheets output spreadsheet, updating existing rows or appending new ones using the email as the key.

- **Nodes Involved:**  
  - 🧠 Format Hunter Response  
  - 📤 Write Results to Output Sheet

- **Node Details:**

  - **🧠 Format Hunter Response**  
    - Type: Code node (JavaScript)  
    - Role: Merges original cleaned data with Hunter.io API results into a structured format.  
    - Configuration:  
      - Reads original data and Hunter results arrays in parallel.  
      - Extracts key fields: status, score, smtp_check, accept_all, source count, with fallbacks.  
      - Creates a friendly `statusSummary` string combining status and confidence score.  
    - Input: Two inputs — original cleaned data and Hunter.io responses.  
    - Output: Array of JSON objects with enriched verification details and original contact info.  
    - Edge Cases:  
      - Mismatched array lengths or missing Hunter data gracefully handled with defaults.  
      - Possible undefined fields in API response handled with fallback values.

  - **📤 Write Results to Output Sheet**  
    - Type: Google Sheets  
    - Role: Writes the formatted verification results to the output spreadsheet (Sheet1).  
    - Configuration:  
      - Document ID: Parameterized as `{{YOUR_OUTPUT_SHEET_ID}}`.  
      - Sheet Name: `gid=0` targeting first sheet.  
      - Operation: Append or update rows using `email` as the match key (updates existing rows or appends new).  
      - Columns mapped explicitly (email, score, status, company, lastName, firstName, smtp_check, statusSummary).  
      - Uses Google Sheets OAuth2 credentials.  
    - Input: Formatted verification results.  
    - Output: Writes data to Google Sheets, no further output.  
    - Edge Cases:  
      - Permission or token issues may block writing.  
      - If output sheet structure changes, mapping may fail or write incorrect data.  
      - Conflicts if multiple workflow runs overlap.

---

### 3. Summary Table

| Node Name                      | Node Type            | Functional Role                          | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                          |
|-------------------------------|----------------------|----------------------------------------|-----------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| ⏯️ Triggers Everyday            | Schedule Trigger     | Initiates workflow daily                | -                           | 📥 Fetch Emails from Source Sheet | **Workflow begins manually using the trigger.\n\nEmails are read from the source Google Sheet (Sheet1).\n\nInput data is expected to include columns like Email, FirstName, LastName, and Company.** |
| 📥 Fetch Emails from Source Sheet | Google Sheets       | Reads raw emails and contact data       | ⏯️ Triggers Everyday          | 🧹 Clean & Prepare Email Data     | See above                                                                                          |
| 🧹 Clean & Prepare Email Data    | Code (JavaScript)    | Cleans emails and prepares data for verification | 📥 Fetch Emails from Source Sheet | 🕵️‍♂️ Hunter.io Email Verifier  | **Filters out invalid or malformed emails.\n\nExtracts fields for each contact.\n\nSends each email to Hunter.io for verification using /v2/email-verifier.\n\nReturns rich metadata: status, score, smtp_check, disposable, etc.** |
| 🕵️‍♂️ Hunter.io Email Verifier  | HTTP Request         | Calls Hunter.io API to verify emails     | 🧹 Clean & Prepare Email Data | 🧠 Format Hunter Response        | See above                                                                                          |
| 🧠 Format Hunter Response        | Code (JavaScript)    | Parses and formats Hunter.io results     | 🕵️‍♂️ Hunter.io Email Verifier | 📤 Write Results to Output Sheet | **Parses Hunter.io API results and maps them to a clean format.\n\nAdds a readable statusSummary (like: valid (96% confidence)).\n\nAppends or updates the output Google Sheet (Email Verifier → Sheet1) using email as the match key.** |
| 📤 Write Results to Output Sheet | Google Sheets       | Writes verification results to output sheet | 🧠 Format Hunter Response    | -                               | See above                                                                                          |
| Sticky Note                    | Sticky Note          | Documentation note                       | -                           | -                               | **Workflow begins manually using the trigger.\n\nEmails are read from the source Google Sheet (Sheet1).\n\nInput data is expected to include columns like Email, FirstName, LastName, and Company.** |
| Sticky Note1                   | Sticky Note          | Documentation note                       | -                           | -                               | **Filters out invalid or malformed emails.\n\nExtracts fields for each contact.\n\nSends each email to Hunter.io for verification using /v2/email-verifier.\n\nReturns rich metadata: status, score, smtp_check, disposable, etc.** |
| Sticky Note2                   | Sticky Note          | Documentation note                       | -                           | -                               | **Parses Hunter.io API results and maps them to a clean format.\n\nAdds a readable statusSummary (like: valid (96% confidence)).\n\nAppends or updates the output Google Sheet (Email Verifier → Sheet1) using email as the match key.** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Name: `⏯️ Triggers Everyday`  
   - Type: Schedule Trigger  
   - Set to run daily at default time.

2. **Add a Google Sheets node to fetch input data:**  
   - Name: `📥 Fetch Emails from Source Sheet`  
   - Type: Google Sheets  
   - Operation: Read rows  
   - Document ID: Set as a parameter (e.g., `{{YOUR_INPUT_SHEET_ID}}`)  
   - Sheet Name: Use sheet with `gid=0` (usually first sheet)  
   - Credentials: Set up Google Sheets OAuth2 credentials with read access.  
   - Connect `⏯️ Triggers Everyday` output to this node.

3. **Add a Code node to clean and prepare email data:**  
   - Name: `🧹 Clean & Prepare Email Data`  
   - Type: Code  
   - Language: JavaScript  
   - Paste the provided script:  
     ```javascript
     const items = [];
     for (const item of $input.all()) {
       const email = item.json.Email || item.json.email;
       if (email && email.includes('@')) {
         items.push({
           json: {
             email: email,
             originalRowIndex: item.json.row_number || items.length + 2,
             firstName: item.json.FirstName || '',
             lastName: item.json.LastName || '',
             company: item.json.Company || ''
           }
         });
       }
     }
     return items;
     ```
   - Connect from `📥 Fetch Emails from Source Sheet`.

4. **Add an HTTP Request node to call Hunter.io API:**  
   - Name: `🕵️‍♂️ Hunter.io Email Verifier`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.hunter.io/v2/email-verifier`  
   - Query Parameters:  
     - `email` = `{{$json.email}}`  
     - `api_key` = `{{HUNTER_API_KEY}}` (set as a workflow variable or environment variable)  
   - Connect from `🧹 Clean & Prepare Email Data`.

5. **Add a Code node to format Hunter.io responses:**  
   - Name: `🧠 Format Hunter Response`  
   - Type: Code  
   - Language: JavaScript  
   - Paste the provided script:  
     ```javascript
     const originalData = $('🧹 Clean & Prepare Email Data').all();
     const hunterResults = $('🕵️‍♂️ Hunter.io Email Verifier').all();
     
     const results = [];
     
     for (let i = 0; i < originalData.length; i++) {
       const original = originalData[i].json;
       const hunter = hunterResults[i]?.json?.data || {};
       
       const result = {
         email: original.email,
         firstName: original.firstName,
         lastName: original.lastName,
         company: original.company,
         rowIndex: original.originalRowIndex,
     
         status: hunter.status || 'unknown',
         score: hunter.score || 0,
         smtpCheck: hunter.smtp_check || false,
         acceptAll: hunter.accept_all || false,
         sourceCount: Array.isArray(hunter.sources) ? hunter.sources.length : 0,
     
         regex: hunter.regex || false,
         gibberish: hunter.gibberish || false,
         disposable: hunter.disposable || false,
         webmail: hunter.webmail || false,
         mxRecords: hunter.mx_records || false,
         smtpServer: hunter.smtp_server || false,
         block: hunter.block || false,
     
         statusSummary: `${hunter.status || 'unknown'} (${hunter.score || 0}% confidence)`
       };
       
       results.push({ json: result });
     }
     
     return results;
     ```
   - Connect from `🕵️‍♂️ Hunter.io Email Verifier`.

6. **Add a Google Sheets node to write results:**  
   - Name: `📤 Write Results to Output Sheet`  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Document ID: Parameterized as `{{YOUR_OUTPUT_SHEET_ID}}`  
   - Sheet Name: `gid=0` (Sheet1)  
   - Mapping Mode: Define mappings explicitly for columns:  
     - `email`: `{{$json.email}}`  
     - `score`: `{{$json.score}}`  
     - `status`: `{{$json.status}}`  
     - `company`: `{{$json.company}}`  
     - `lastName`: `{{$json.lastName}}`  
     - `firstName`: `{{$json.firstName}}`  
     - `smtp_check`: `{{$json.smtpCheck}}`  
     - `statusSummary`: `{{$json.statusSummary}}`  
   - Matching Columns: `email` (to update existing rows)  
   - Credentials: Google Sheets OAuth2 with write access  
   - Connect from `🧠 Format Hunter Response`.

7. **Credentials Setup:**  
   - Create or reuse Google Sheets OAuth2 credentials with read/write permissions for the input and output sheets.  
   - Store your Hunter.io API key securely as an environment variable or workflow parameter named `HUNTER_API_KEY`.

8. **Parameterize Document IDs:**  
   - Replace `{{YOUR_INPUT_SHEET_ID}}` and `{{YOUR_OUTPUT_SHEET_ID}}` with actual Google Sheets document IDs where input emails and output results reside.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow designed to run automatically every day for continuous email list validation.                        | N/A                                                                                                    |
| Hunter.io API provides detailed email verification including SMTP, disposable, and confidence scoring metadata. | https://hunter.io/api/email-verifier                                                                    |
| Google Sheets OAuth2 credentials must have permissions for both reading the source sheet and writing to output. | https://developers.google.com/identity/protocols/oauth2                                                 |
| The workflow relies on consistent input sheet column names: Email, FirstName, LastName, Company.               | If column names differ, adjust the code node accordingly.                                              |
| For better reliability, consider adding error handling nodes or notifications on API or Google Sheets failures.| Not included in this version but recommended for production use.                                        |

---

**Disclaimer:**  
The provided content is extracted exclusively from an automated n8n workflow designed using the n8n platform. It complies fully with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.