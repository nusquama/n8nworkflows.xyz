Extracts and organizes academic publications using GPT-4 Mini, Google Sheets, Gmail

https://n8nworkflows.xyz/workflows/extracts-and-organizes-academic-publications-using-gpt-4-mini--google-sheets--gmail-9282


# Extracts and organizes academic publications using GPT-4 Mini, Google Sheets, Gmail

### 1. Workflow Overview

This workflow automates the extraction, organization, and reporting of academic publications from university staff web pages using GPT-4 Mini, Google Sheets, and Gmail. It targets academic researchers, university administrators, librarians, department heads, and PhD students who need to track, categorize, and maintain publication records efficiently.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures staff publication data submission via a form.
- **1.2 Web Content Fetching and Parsing:** Retrieves the staff’s publication web page and extracts publication lists.
- **1.3 Publication Splitting and AI Extraction:** Splits the publication list into individual entries and uses GPT-4 Mini to extract structured metadata (authors, year, journal/conference).
- **1.4 Data Storing and Statistics Calculation:** Saves publication data to a master Google Sheet and calculates publication counts by venue and year.
- **1.5 Publication Type Routing and Categorization:** Routes publications by type (journal, conference, book, magazine, patent, others) using a switch node, sorts each category by year, and appends them to respective categorized Google Sheets.
- **1.6 Reporting and Notification:** Converts processed data to CSV and optionally emails it to the user with a summary.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block triggers the workflow when a user submits a form with academic staff publication details.
- **Nodes Involved:**  
  - `On form submission`

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Entry point capturing the form data: Staff Name, URL, Email, Google Scholar URL.  
    - *Configuration:* Uses a webhook with form fields defined for academic staff publication details.  
    - *Key expressions:* Utilizes `$json.URL` and `$json.Email` downstream.  
    - *Input:* External HTTP form submission.  
    - *Output:* Triggers `Fetch website content`.  
    - *Failure cases:* Missing or malformed form data, webhook connectivity issues.  
    - *Sub-workflow:* None.

#### 2.2 Web Content Fetching and Parsing

- **Overview:** Fetches the staff publication webpage and extracts the raw HTML content relevant to publications, summary, and bio.
- **Nodes Involved:**  
  - `Fetch website content`  
  - `Extract all publications from the page`  
  - `Separate Each Publication`

- **Node Details:**

  - **Fetch website content**  
    - *Type:* HTTP Request  
    - *Role:* Downloads the web page content at the submitted URL with `#publications` anchor.  
    - *Configuration:* Allows unauthorized SSL certificates (useful for self-signed certs).  
    - *Input:* Triggered by form submission node.  
    - *Output:* Passes HTML content to extraction node.  
    - *Failure cases:* Network errors, invalid URLs, timeouts, HTTP errors.

  - **Extract all publications from the page**  
    - *Type:* HTML Extract  
    - *Role:* Parses HTML to extract:  
      - Summary text (CSS selector `.dropDownSummary.textArea h3`)  
      - List of publications (CSS selector `.dropDownSummary.textArea ul ul li`)  
      - Bio section (CSS selector `.dropDownSummary.textArea`)  
    - *Configuration:* Cleans up text automatically, returns arrays for multiple entries.  
    - *Input:* HTML content from previous node.  
    - *Output:* Passes extracted publication list to splitting node.  
    - *Failure cases:* CSS selector mismatch if page structure changes; empty or malformed HTML.

  - **Separate Each Publication**  
    - *Type:* Split Out  
    - *Role:* Splits the array of extracted publications into individual items for further processing.  
    - *Configuration:* Splits by the `publications` field; includes "Year" field if present.  
    - *Input:* Array of publications extracted.  
    - *Output:* Individual publication entries to AI extraction node.  
    - *Failure cases:* Empty array, wrong field name, malformed data.

#### 2.3 Publication Splitting and AI Extraction

- **Overview:** Uses GPT-4 Mini to parse each publication entry into structured metadata: authors, journal or conference name, year, etc.
- **Nodes Involved:**  
  - `Generate Summary Report`  
  - `OpenAI Chat Model`

- **Node Details:**

  - **Generate Summary Report**  
    - *Type:* Langchain Information Extractor  
    - *Role:* Extracts from text the author names, journal/conference, and year using a system prompt.  
    - *Configuration:* Uses a system prompt instructing to only extract specified fields, omitting unknowns.  
    - *Input:* Text of single publication entry from previous split.  
    - *Output:* Structured JSON with `authors`, `journal_conference`, and `year`.  
    - *Failure cases:* GPT response failures, API rate limits, malformed text input.

  - **OpenAI Chat Model**  
    - *Type:* Langchain Chat Model Node  
    - *Role:* Invokes GPT-4 Mini model to perform the extraction task.  
    - *Configuration:* Model set to `gpt-4.1-mini`; no additional options specified.  
    - *Input:* Receives publication text from split node.  
    - *Output:* Provides AI-extracted structured data to next nodes.  
    - *Credentials:* Requires valid OpenAI API key with GPT-4 Mini access.  
    - *Failure cases:* API errors, authentication failures, timeout.

#### 2.4 Data Storing and Statistics Calculation

- **Overview:** Saves all extracted publications to a master Google Sheet and calculates statistics (counts) grouped by journal/conference and year.
- **Nodes Involved:**  
  - `Save All to Master Sheet`  
  - `Calculate Publication Statistics (Count)`

- **Node Details:**

  - **Save All to Master Sheet**  
    - *Type:* Google Sheets  
    - *Role:* Appends extracted publication data (year, authors, journal/conference) to a master sheet named "all".  
    - *Configuration:* Defines columns explicitly; does not convert types; appends data.  
    - *Input:* Structured publication JSON data from AI extraction.  
    - *Output:* Passes data to statistics calculation node.  
    - *Credentials:* Google Sheets OAuth2 with write access.  
    - *Failure cases:* Authentication errors, quota limits, incorrect spreadsheet ID or sheet name.

  - **Calculate Publication Statistics (Count)**  
    - *Type:* Summarize  
    - *Role:* Groups data by `journal_conference` and `year` and counts occurrences (publications).  
    - *Configuration:* Splits by `journal_conference` and `year`.  
    - *Input:* Data from master sheet append.  
    - *Output:* Passes grouped publication counts to routing switch.  
    - *Failure cases:* Empty input, grouping errors.

#### 2.5 Publication Type Routing and Categorization

- **Overview:** Routes the publication counts to different Google Sheets based on publication type (journal, conference, book, magazine, patent, others). Each category is sorted by year descending before appending.
- **Nodes Involved:**  
  - `Switch`  
  - `Sort by Year (Journal)`  
  - `Sort by Year (Conference)`  
  - `Sort by Year (Book)`  
  - `Sort by Year (Magazine)`  
  - `Sort by Year (Patent)`  
  - `Sort by Year (Other)`  
  - `Append to Journal Sheet`  
  - `Append to Conference Sheet`  
  - `Append to Book Sheet`  
  - `Append to Magazine Sheet`  
  - `Append to Patent Sheet`  
  - `Append to Other Sheet`  
  - `Sticky Note` (comment on routing)

- **Node Details:**

  - **Switch**  
    - *Type:* Switch  
    - *Role:* Routes each publication record based on regex matching against lowercased `journal_conference` field for classification into journals, conferences, books, magazines, patents, or default "Others".  
    - *Configuration:* Uses multiple regex rules matching common publication venue keywords for each category.  
    - *Input:* Grouped counts from previous node.  
    - *Output:* Passes to sorting nodes per category.  
    - *Failure cases:* Misclassification due to incomplete regex or unexpected venue names.

  - **Sort by Year (Journal / Conference / Book / Magazine / Patent / Other)**  
    - *Type:* Sort  
    - *Role:* Sorts each category's publication list by `year` descending to organize recent publications first.  
    - *Configuration:* Sort on `year` field descending.  
    - *Input:* Publications from switch node output per category.  
    - *Output:* Passes sorted data to respective Google Sheets append nodes.  
    - *Failure cases:* Non-numeric or missing `year` field.

  - **Append to [Category] Sheet** (Journal, Conference, Book, Magazine, Patent, Other)  
    - *Type:* Google Sheets  
    - *Role:* Appends or updates the category sheet with publication counts, year, and venue name.  
    - *Configuration:* Defines column mappings explicitly, uses appendOrUpdate operation keyed on count field.  
    - *Input:* Sorted category data from sort nodes.  
    - *Output:* None (end of data routing for each category).  
    - *Credentials:* Google Sheets OAuth2 with write access.  
    - *Failure cases:* Authentication, quota limits, spreadsheet or sheet naming errors.

  - **Sticky Note**  
    - *Content:* "Switch Node: Route by publication type"  
    - *Purpose:* Clarifies routing logic visually for users.

#### 2.6 Reporting and Notification

- **Overview:** Converts the extracted and summarized publication data into CSV format and sends an email notification with the CSV attached.
- **Nodes Involved:**  
  - `Format as CSV Export`  
  - `Send Notification Email`

- **Node Details:**

  - **Format as CSV Export**  
    - *Type:* Convert To File  
    - *Role:* Converts structured JSON data from AI extraction into CSV file format for export.  
    - *Input:* Data from `Generate Summary Report`.  
    - *Output:* Passes CSV binary data to email node.  
    - *Failure cases:* Data conversion errors.

  - **Send Notification Email**  
    - *Type:* Gmail  
    - *Role:* Sends an email to the user submitting the form with a message and CSV attachment of publications.  
    - *Configuration:* Sends to `$json.Email`, subject includes staff name, message references the URL submitted.  
    - *Input:* CSV file from previous node.  
    - *Credentials:* Gmail OAuth2 account with send permission.  
    - *Failure cases:* Authentication errors, email delivery failure.

---

### 3. Summary Table

| Node Name                    | Node Type                           | Functional Role                          | Input Node(s)                   | Output Node(s)                            | Sticky Note                        |
|------------------------------|-----------------------------------|----------------------------------------|--------------------------------|------------------------------------------|----------------------------------|
| On form submission           | Form Trigger                      | Entry point for form data submission   | -                              | Fetch website content                     |                                  |
| Fetch website content        | HTTP Request                     | Fetches staff publication webpage      | On form submission             | Extract all publications from the page   |                                  |
| Extract all publications from the page | HTML Extract                      | Extracts publication lists & bio       | Fetch website content          | Separate Each Publication                 |                                  |
| Separate Each Publication    | Split Out                        | Splits publication list into entries   | Extract all publications       | Generate Summary Report                   |                                  |
| OpenAI Chat Model            | Langchain Chat Model             | Runs GPT-4 Mini for extraction          | Separate Each Publication      | Generate Summary Report                   |                                  |
| Generate Summary Report      | Langchain Information Extractor  | Extracts authors, journal/conference, year | OpenAI Chat Model             | Format as CSV Export, Save All to Master Sheet |                                  |
| Save All to Master Sheet     | Google Sheets                   | Appends raw data to master sheet       | Generate Summary Report        | Calculate Publication Statistics (Count) |                                  |
| Calculate Publication Statistics (Count) | Summarize                      | Counts publications grouped by venue & year | Save All to Master Sheet      | Switch                                   |                                  |
| Switch                      | Switch                          | Routes publications by type             | Calculate Publication Statistics | Sort by Year (Journal), (Conference), etc. | Switch Node: Route by publication type |
| Sort by Year (Journal)      | Sort                           | Sorts journal entries by year descending | Switch (journal_papers)        | Append to Journal Sheet                   |                                  |
| Sort by Year (Conference)   | Sort                           | Sorts conference entries by year descending | Switch (conference_papers)     | Append to Conference Sheet                |                                  |
| Sort by Year (Book)         | Sort                           | Sorts book entries by year descending   | Switch (books)                 | Append to Book Sheet                      |                                  |
| Sort by Year (Magazine)     | Sort                           | Sorts magazine entries by year descending | Switch (magazine)              | Append to Magazine Sheet                  |                                  |
| Sort by Year (Patent)       | Sort                           | Sorts patent entries by year descending | Switch (patent)                | Append to Patent Sheet                    |                                  |
| Sort by Year (Other)        | Sort                           | Sorts other entries by year descending  | Switch (extra / others)        | Append to Other Sheet                     |                                  |
| Append to Journal Sheet     | Google Sheets                  | Updates journal publications sheet      | Sort by Year (Journal)         | -                                        |                                  |
| Append to Conference Sheet  | Google Sheets                  | Updates conference publications sheet   | Sort by Year (Conference)      | -                                        |                                  |
| Append to Book Sheet        | Google Sheets                  | Updates book publications sheet         | Sort by Year (Book)            | -                                        |                                  |
| Append to Magazine Sheet    | Google Sheets                  | Updates magazine publications sheet     | Sort by Year (Magazine)        | -                                        |                                  |
| Append to Patent Sheet      | Google Sheets                  | Updates patent publications sheet       | Sort by Year (Patent)          | -                                        |                                  |
| Append to Other Sheet       | Google Sheets                  | Updates "others" publications sheet     | Sort by Year (Other)           | -                                        |                                  |
| Format as CSV Export        | Convert To File                | Converts extracted data into CSV format | Generate Summary Report        | Send Notification Email                   |                                  |
| Send Notification Email     | Gmail                         | Emails CSV file and publication summary | Format as CSV Export           | -                                        |                                  |
| Sticky Note3                | Sticky Note                   | Credits the author "By CSChin"          | -                              | -                                        | By CSChin                       |
| Sticky Note1                | Sticky Note                   | Workflow overview, source, use cases, instructions | -                              | -                                        | See detailed overview in section 5 |
| Sticky Note                 | Sticky Note                   | Explains Switch Node routing            | -                              | -                                        | Switch Node: Route by publication type |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Node type: Form Trigger  
   - Configure form fields: Staff Name, URL, Email, Google Scholar URL  
   - Note the webhook URL for external submissions.

2. **Add an HTTP Request node ("Fetch website content")**  
   - Connect from Form Trigger.  
   - Set URL to `={{ $json.URL }}#publications` to fetch the publication section.  
   - Enable "Allow Unauthorized Certificates" for SSL flexibility.

3. **Add an HTML Extract node ("Extract all publications from the page")**  
   - Connect from HTTP Request.  
   - Operation: Extract HTML Content.  
   - Extraction fields:  
     - `summary`: CSS `.dropDownSummary.textArea h3` (array)  
     - `publications`: CSS `.dropDownSummary.textArea ul ul li` (array)  
     - `bio`: CSS `.dropDownSummary.textArea`  
   - Enable text cleanup.

4. **Add a Split Out node ("Separate Each Publication")**  
   - Connect from HTML Extract.  
   - Split by field: `publications`  
   - Include field "Year" if available.

5. **Add OpenAI Chat Model node ("OpenAI Chat Model")**  
   - Connect from Split Out node.  
   - Model: `gpt-4.1-mini`  
   - Credentials: OpenAI API key with GPT-4 Mini enabled.

6. **Add Langchain Information Extractor node ("Generate Summary Report")**  
   - Connect from OpenAI Chat Model.  
   - Input text: `={{ $json.publications }}` (single publication text).  
   - System prompt: instruct extraction of `authors`, `journal_conference`, and `year` only.  
   - Output: structured JSON.

7. **Add Google Sheets node ("Save All to Master Sheet")**  
   - Connect from Generate Summary Report.  
   - Operation: Append.  
   - Map columns: `year`, `authors`, `journal_conference` from extracted data.  
   - Set Spreadsheet ID and sheet name for master data.  
   - Credentials: Google Sheets OAuth2.

8. **Add Summarize node ("Calculate Publication Statistics (Count)")**  
   - Connect from Save All to Master Sheet.  
   - Group by `journal_conference` and `year`.  
   - Summarize count of publications.

9. **Add Switch node ("Switch")**  
   - Connect from Summarize.  
   - Define multiple regex-based rules to match `journal_conference` lowercased string for categories:  
     - Journals, Conferences, Books, Magazines, Patents, Others (fallback).  
   - Set output keys accordingly.

10. **For each category (journal, conference, book, magazine, patent, others):**  
    - Add a Sort node to sort by `year` descending.  
    - Connect from corresponding Switch output.  
    - Add Google Sheets node to appendOrUpdate category-specific data:  
      - Map columns accordingly (e.g., for journals: `year`, `count`, `journal`).  
      - Configure target sheet name and spreadsheet ID.  
      - Use Google Sheets OAuth2 credentials.

11. **Add Convert To File node ("Format as CSV Export")**  
    - Connect from Generate Summary Report.  
    - Configure to convert JSON to CSV format.

12. **Add Gmail node ("Send Notification Email")**  
    - Connect from Convert To File.  
    - Configure recipient email from form submission `$json.Email`.  
    - Compose subject and message referencing staff name and URL.  
    - Attach CSV as binary attachment.  
    - Use Gmail OAuth2 credentials.

13. **Add Sticky Notes** for documentation and clarity within the workflow editor (optional).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Source URL: https://www.ncl.ac.uk/singapore/staff/profile/chengchin.html#publications | Workflow designed specifically for this page structure but adaptable. |
| Workflow author: By CSChin | Attribution note in Sticky Note3 node. |
| Workflow overview and detailed setup instructions included in Sticky Note1 | Provides comprehensive setup, API keys, credentials, and customization tips. |
| Switch Node uses extensive regex matching to classify publication types accurately | Critical for correct data routing; regex may require updates for new publication venues. |
| Requires valid API credentials for OpenAI (GPT-4 Mini), Google Sheets (OAuth2), and optionally Gmail (OAuth2) | Credentials must be configured in n8n prior to workflow activation. |
| Adaptation tips: Modify CSS selectors in HTML Extract node if website structure changes | Important for maintaining extraction accuracy over time. |

---

This documentation provides a detailed, stepwise understanding and reproduction guideline for the workflow "Extract & Organize Academic Publications with GPT-4 Mini, Google Sheets & Gmail." It is designed for advanced users and automation agents to modify, extend, or troubleshoot the workflow effectively.