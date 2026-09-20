Create weekly research brief PDFs with Brave Search, Firecrawl, Claude, PDF.co, Gmail and Google Drive

https://n8nworkflows.xyz/workflows/create-weekly-research-brief-pdfs-with-brave-search--firecrawl--claude--pdf-co--gmail-and-google-drive-19662


# Create weekly research brief PDFs with Brave Search, Firecrawl, Claude, PDF.co, Gmail and Google Drive

### 1. Workflow Overview

This workflow automates the generation, conversion, and distribution of a weekly research brief. Operating on a scheduled basis, it searches target topics using Brave Search, scrapes and cleans source content with Firecrawl, processes the aggregated data through Anthropic Claude (via an LLM chain and structured output parser) to synthesize findings, formats the output into responsive HTML, converts it into an A4 PDF using PDF.co, and finally delivers the document via Gmail and archives it in Google Drive.

The architecture is divided into the following logical blocks:
- **1.1 Input Reception & Normalization:** Manages the trigger schedule, initial configuration variables, and splits comma-separated research topics into individual iteration items.
- **1.2 Source Collection & Scraping:** Performs web discovery via Brave Search, formats and limits the result set, and scrapes/cleans content via Firecrawl with built-in fallbacks.
- **1.3 AI Processing & Synthesis:** Aggregates sources, queries Anthropic Claude using strict evidence rules, and parses the output into a validated JSON schema.
- **1.4 Document Generation & Rendering:** Compiles structured JSON data and metadata into styled HTML and converts it to a downloadable PDF via PDF.co.
- **1.5 Distribution & Archiving:** Sends the compiled PDF file via email through Gmail while simultaneously uploading a copy to a designated Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
This block initiates the workflow weekly, loads global configuration parameters, normalizes the comma-separated topics list, and expands it into discrete execution items for parallel processing.

- **Weekly Trigger**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger)
  - **Configuration Choices:** Configured to trigger via a cron expression (`0 7 * * MON`, every Monday at 07:00).
  - **Key Expressions:** None.
  - **Connections:** Output connects to `Set Brief Parameters`.
  - **Edge Cases / Failures:** Timezone misconfigurations in workflow settings may cause runs to execute at unintended local times.

- **Set Brief Parameters**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Data Transformation)
  - **Configuration Choices:** Initializes global execution variables (`topics_csv`, `results_per_topic`, `recipient_email`, `brand_name`, `drive_folder_id`, `max_source_characters`, `logo_url`).
  - **Key Expressions:** Static assignment of configuration values.
  - **Connections:** Input from `Weekly Trigger`; output connects to `Set Normalized Topics`.
  - **Edge Cases / Failures:** Empty recipient email or Google Drive folder ID values will cause downstream delivery failures.

- **Set Normalized Topics**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Data Transformation)
  - **Configuration Choices:** Converts the raw `topics_csv` string into a sanitized array of strings.
  - **Key Expressions:** `={{ $json.topics_csv.split(',').map(topic => topic.trim()).filter(Boolean) }}`
  - **Connections:** Input from `Set Brief Parameters`; output connects to `Split Topics Individually`.
  - **Edge Cases / Failures:** Malformed CSV strings without valid entries will result in an empty array.

- **Split Topics Individually**
  - **Type & Technical Role:** `n8n-nodes-base.splitOut` (Data Transformation)
  - **Configuration Choices:** Expands the topics array into individual items for downstream iteration.
  - **Key Expressions:** Splits out the field `topics` with destination field name `topic`.
  - **Connections:** Input from `Set Normalized Topics`; output connects to `Brave Search Topics`.
  - **Edge Cases / Failures:** Empty topic arrays will halt execution propagation.

---

#### 2.2 Source Collection & Scraping
This block queries Brave Search for each topic, filters and formats the results, scrapes the target pages via Firecrawl, and cleans the extracted Markdown content.

- **Brave Search Topics**
  - **Type & Technical Role:** `@brave/n8n-nodes-brave-search.braveSearch` (Action / External API)
  - **Configuration Choices:** Searches web pages with a past-week freshness filter (`pw`) and queries limited by `results_per_topic`.
  - **Key Expressions:** Query: `={{ $json.topic }}`, Count: `={{ $json.results_per_topic }}`
  - **Connections:** Input from `Split Topics Individually`; output connects to `Format Source Data`.
  - **Credentials:** Brave Search API (supports Gateway credits).
  - **Edge Cases / Failures:** Rate limits, network timeouts, or API key authorization errors. Includes retry logic (`retryOnFail`, 2000ms delay).

- **Format Source Data**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation)
  - **Configuration Choices:** Validates search responses, enforces result limits, filters non-HTTP URLs, and assigns unique source identifiers (`s1`, `s2`, etc.).
  - **Key Expressions:** JavaScript code referencing `Set Brief Parameters` and upstream Brave JSON output.
  - **Connections:** Input from `Brave Search Topics`; output connects to `Extract with Firecrawl`.
  - **Edge Cases / Failures:** Throws an explicit error if zero usable web results are found across all queried topics.

- **Extract with Firecrawl**
  - **Type & Technical Role:** `@mendable/n8n-nodes-firecrawl.firecrawl` (Action / External API)
  - **Configuration Choices:** Scrapes target URLs to extract Markdown content.
  - **Key Expressions:** URL: `={{ $json.url }}`
  - **Connections:** Input from `Format Source Data`; output connects to `Clean Source Text`.
  - **Credentials:** Firecrawl API (supports Gateway credits).
  - **Edge Cases / Failures:** Failed scrapes (status code >= 400 or unsuccessful flags) will halt execution due to downstream validation. Includes retry logic (`retryOnFail`, 2000ms delay).

- **Clean Source Text**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation)
  - **Configuration Choices:** Normalizes markdown line endings, strips image links, truncates text to `max_source_characters`, and provides search snippets as a fallback when page text is empty.
  - **Key Expressions:** JavaScript processing loop reading upstream Firecrawl data and mapping back to `Format Source Data`.
  - **Connections:** Input from `Extract with Firecrawl`; output connects to `Combine Sources`.
  - **Edge Cases / Failures:** Throws errors if HTTP response status codes indicate failure or if both markdown and search snippets are empty.

---

#### 2.3 AI Processing & Synthesis
This block aggregates all cleaned source items into a single dataset, submits them along with prompt instructions to Anthropic Claude via an LLM chain, and enforces a structured JSON schema using an output parser.

- **Combine Sources**
  - **Type & Technical Role:** `n8n-nodes-base.aggregate` (Data Transformation)
  - **Configuration Choices:** Collects all upstream source items into a single consolidated item containing an array field named `sources`.
  - **Key Expressions:** Aggregate mode: `aggregateAllItemData`, Destination Field: `sources`.
  - **Connections:** Input from `Clean Source Text`; output connects to `Generate Research Brief`.
  - **Edge Cases / Failures:** Large aggregated payloads may approach token limits if source counts or character lengths are exceptionally high.

- **Generate Research Brief**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.chainLlm` (AI / LLM Chain)
  - **Configuration Choices:** Coordinates the LLM prompt, injecting current dates, reporting windows, normalized topics, and aggregated source JSON. Enforces strict evidence and recency rules.
  - **Key Expressions:** Dynamic prompt insertion for report date, reporting window, topics array, and source material.
  - **Connections:** Inputs from `Combine Sources`, `Claude Sonnet Model` (ai_languageModel), and `Parse Structure Output` (ai_outputParser); output connects to `Create HTML Brief`.
  - **Edge Cases / Failures:** Hallucinated citations or structural parser mismatches if the model fails to adhere to the schema.

- **Claude Sonnet Model**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` (AI / Language Model)
  - **Configuration Choices:** Configured to use the Claude Sonnet model with maximum sample tokens set to 4000.
  - **Key Expressions:** None.
  - **Connections:** Output connects to `Generate Research Brief` (`ai_languageModel`).
  - **Credentials:** Anthropic API (supports Gateway credits).
  - **Edge Cases / Failures:** API quota limits, service outages, or token length overflows.

- **Parse Structure Output**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.outputParserStructured` (AI / Output Parser)
  - **Configuration Choices:** Enforces a JSON schema requiring a two-sentence summary, topic sections with findings and source IDs, and up to three watch items.
  - **Key Expressions:** JSON schema example defining `summary`, `sections`, and `watch_items`.
  - **Connections:** Output connects to `Generate Research Brief` (`ai_outputParser`).
  - **Edge Cases / Failures:** Parsing failures if the LLM response contains markdown wrappers or invalid JSON structures.

---

#### 2.4 Document Generation & Rendering
This block validates the AI-generated JSON structure, compiles the layout into responsive HTML with custom CSS, generates an A4 PDF via PDF.co, and downloads the binary file.

- **Create HTML Brief**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation)
  - **Configuration Choices:** Parses and validates AI output, verifies source ID references against original metadata, escapes HTML entities, and injects styling, headers, footers, and branding.
  - **Key Expressions:** JavaScript processing constructing the complete HTML document, metadata strings, and filename.
  - **Connections:** Input from `Generate Research Brief`; output connects to `Generate PDF with PDF.co`.
  - **Edge Cases / Failures:** Throws validation errors if summary, sections, or citations fail structural consistency checks.

- **Generate PDF with PDF.co**
  - **Type & Technical Role:** `n8n-nodes-pdfco.PDFco Api` (Action / External API)
  - **Configuration Choices:** Converts HTML input to PDF using A4 portrait paper size, specific margins (`22mm 18mm 20mm 18mm`), and custom footer templates with page numbering.
  - **Key Expressions:** HTML: `={{ $json.html }}`, Name: `={{ $json.filename }}`, Footer: `={{ $json.footer_html }}`
  - **Connections:** Input from `Create HTML Brief`; output connects to `Download PDF File`.
  - **Credentials:** PDF.co API (supports Gateway credits).
  - **Edge Cases / Failures:** Rendering timeouts or malformed HTML elements.

- **Download PDF File**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Action / HTTP Request)
  - **Configuration Choices:** Downloads the generated PDF from the temporary PDF.co URL into the binary field `data`.
  - **Key Expressions:** URL: `={{ $json.url }}`
  - **Connections:** Input from `Generate PDF with PDF.co`; outputs connect to `Send Brief via Email` and `Save Brief to Drive`.
  - **Edge Cases / Failures:** Broken or expired download URLs returned by the PDF generation service. Includes retry logic (`retryOnFail`, 2000ms delay).

---

#### 2.5 Distribution & Archiving
This block simultaneously delivers the research brief via email using Gmail and uploads a copy to the specified folder in Google Drive.

- **Send Brief via Email**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` (Action / Integration)
  - **Configuration Choices:** Sends a text email containing the attached binary PDF (`data`) to the configured recipient.
  - **Key Expressions:** Send To: `={{ $('Set Brief Parameters').first().json.recipient_email }}`, Subject: `={{ $('Create HTML Brief').first().json.email_subject }}`
  - **Connections:** Input from `Download PDF File`. No downstream nodes.
  - **Credentials:** Gmail OAuth2 account.
  - **Edge Cases / Failures:** Authentication token expiration, empty recipient fields, or attachment size limits.

- **Save Brief to Drive**
  - **Type & Technical Role:** `n8n-nodes-base.googleDrive` (Action / Integration)
  - **Configuration Choices:** Uploads the binary PDF file to Google Drive using a designated folder ID.
  - **Key Expressions:** Name: `={{ $('Create HTML Brief').first().json.filename }}`, Folder ID: `={{ $('Set Brief Parameters').first().json.drive_folder_id }}`
  - **Connections:** Input from `Download PDF File`. No downstream nodes.
  - **Credentials:** Google Drive OAuth2 account.
  - **Edge Cases / Failures:** Invalid folder IDs, permission errors, or quota exhaustion.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note** | `n8n-nodes-base.stickyNote` | Workflow documentation and setup instructions | None | None | ## Create weekly research PDFs with PDF.co, Brave Search, Firecrawl and Claude<br><br>### How it works<br><br>Generate a branded research brief PDF from configured topics. Brave finds pages, Firecrawl reads them, and Claude writes a structured brief. The workflow builds HTML, creates the PDF with PDF.co, downloads it, then sends it through Gmail and archives it in Google Drive.<br><br>### Setup steps<br><br>1. Configure **Weekly Trigger** and set the intended workflow timezone. The default schedule is Monday at 07:00.<br>2. Update **Set Brief Parameters**: `topics_csv`, `results_per_topic`, `recipient_email`, `brand_name`, and `drive_folder_id`. Adjust `max_source_characters` as needed; `logo_url` is optional.<br>3. For Brave Search, Firecrawl, Anthropic Claude and PDF.co, you can choose **Use Gateway credits** where supported by your workspace and the selected operation. Otherwise, connect your own provider credentials. Provider usage may incur charges.<br>4. Connect Gmail and Google Drive separately using your Google account credentials (OAuth). Follow the Google Drive instructions in the delivery note.<br>5. Test with your own email address and inspect the PDF before enabling scheduled runs. Delivery sends a real email and creates a Drive file.<br><br>### Customization<br><br>Adjust topics, source limits, the Claude prompt/output schema, HTML styling, PDF.co rendering options, and delivery destinations.<br><br>**Review required:** AI-generated findings and citations need review. Search freshness alone does not prove an event happened during the reporting window. |
| **Sticky Note1** | `n8n-nodes-base.stickyNote` | Schedule and parameters documentation | None | None | ## Schedule and brief settings<br><br>**Weekly Trigger** defaults to Monday at 07:00. Set the intended workflow timezone before enabling the schedule.<br><br>Edit **Set Brief Parameters** for topics, recipient, branding, folder ID and source limits. Keep `results_per_topic` and `max_source_characters` as Numbers. |
| **Sticky Note2** | `n8n-nodes-base.stickyNote` | Topic preparation documentation | None | None | ## Prepare topic searches<br><br>Normalizes the comma-separated topic settings into a topic list, splits them into individual items, and sends each topic to Brave Search. |
| **Sticky Note3** | `n8n-nodes-base.stickyNote` | Source collection documentation | None | None | ## Collect source content<br><br>**Format Source Data** caps search results per topic and assigns source IDs. **Extract with Firecrawl** reads the pages; **Clean Source Text** cleans and shortens the text.<br><br>**Current limitation:** an unsuccessful scrape stops the run. The search-snippet fallback only covers a successful response with empty page text. |
| **Sticky Note4** | `n8n-nodes-base.stickyNote` | Research brief generation documentation | None | None | ## Generate research brief<br><br>**Combine Sources** collects the source text into one item. **Generate Research Brief** uses Claude and the structured output parser to create the summary, topic findings, source references and follow-up items.<br><br>The parser checks structure, not factual accuracy. Source-ID validation does not independently verify the claims; review the brief before relying on it. |
| **Sticky Note5** | `n8n-nodes-base.stickyNote` | PDF creation documentation | None | None | ## Create and download PDF<br><br>**Create HTML Brief** validates topic names and source IDs, escapes the generated text and applies the layout.<br><br>**Generate PDF with PDF.co** converts the HTML into a PDF. **Download PDF File** retrieves that PDF into binary field `data` for delivery. |
| **Sticky Note6** | `n8n-nodes-base.stickyNote` | Delivery and archive documentation | None | None | ## Deliver and archive brief<br><br>The downloaded PDF feeds two delivery branches: **Send Brief via Email** attaches binary field `data` to a Gmail message; **Save Brief to Drive** uploads the same file to Google Drive. Both connect directly to **Download PDF File**.<br><br>### Google Drive setup<br><br>After connecting your Google Drive account in **Save Brief to Drive**, select **Parent Drive** from the dropdown. Leave **Parent Folder** set to **By ID**, and enter your destination folder's ID in `drive_folder_id` in **Set Brief Parameters**. Do not leave that setting blank.<br><br>### Before testing<br><br>Confirm the recipient and archive destination. Executing these nodes sends a real email and creates a Drive file. Check existing deliveries before rerunning a partially failed execution to avoid duplicates. |
| **Set Brief Parameters** | `n8n-nodes-base.set` | Initializes global configuration variables | Weekly Trigger | Set Normalized Topics | ## Schedule and brief settings<br><br>**Weekly Trigger** defaults to Monday at 07:00. Set the intended workflow timezone before enabling the schedule.<br><br>Edit **Set Brief Parameters** for topics, recipient, branding, folder ID and source limits. Keep `results_per_topic` and `max_source_characters` as Numbers. |
| **Set Normalized Topics** | `n8n-nodes-base.set` | Sanitizes and converts topics CSV into an array | Set Brief Parameters | Split Topics Individually | ## Prepare topic searches<br><br>Normalizes the comma-separated topic settings into a topic list, splits them into individual items, and sends each topic to Brave Search. |
| **Format Source Data** | `n8n-nodes-base.code` | Validates search results and assigns unique source IDs | Brave Search Topics | Extract with Firecrawl | ## Collect source content<br><br>**Format Source Data** caps search results per topic and assigns source IDs. **Extract with Firecrawl** reads the pages; **Clean Source Text** cleans and shortens the text.<br><br>**Current limitation:** an unsuccessful scrape stops the run. The search-snippet fallback only covers a successful response with empty page text. |
| **Clean Source Text** | `n8n-nodes-base.code` | Normalizes, truncates, and cleans extracted Markdown | Extract with Firecrawl | Combine Sources | ## Collect source content<br><br>**Format Source Data** caps search results per topic and assigns source IDs. **Extract with Firecrawl** reads the pages; **Clean Source Text** cleans and shortens the text.<br><br>**Current limitation:** an unsuccessful scrape stops the run. The search-snippet fallback only covers a successful response with empty page text. |
| **Combine Sources** | `n8n-nodes-base.aggregate` | Aggregates all source items into a single collection | Clean Source Text | Generate Research Brief | ## Generate research brief<br><br>**Combine Sources** collects the source text into one item. **Generate Research Brief** uses Claude and the structured output parser to create the summary, topic findings, source references and follow-up items.<br><br>The parser checks structure, not factual accuracy. Source-ID validation does not independently verify the claims; review the brief before relying on it. |
| **Generate Research Brief** | `@n8n/n8n-nodes-langchain.chainLlm` | Queries Anthropic Claude to synthesize findings | Combine Sources, Claude Sonnet Model, Parse Structure Output | Create HTML Brief | ## Generate research brief<br><br>**Combine Sources** collects the source text into one item. **Generate Research Brief** uses Claude and the structured output parser to create the summary, topic findings, source references and follow-up items.<br><br>The parser checks structure, not factual accuracy. Source-ID validation does not independently verify the claims; review the brief before relying on it. |
| **Claude Sonnet Model** | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | Provides the Claude Sonnet LLM backend | None | Generate Research Brief | ## Generate research brief<br><br>**Combine Sources** collects the source text into one item. **Generate Research Brief** uses Claude and the structured output parser to create the summary, topic findings, source references and follow-up items.<br><br>The parser checks structure, not factual accuracy. Source-ID validation does not independently verify the claims; review the brief before relying on it. |
| **Parse Structure Output** | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON output schema on LLM responses | None | Generate Research Brief | ## Generate research brief<br><br>**Combine Sources** collects the source text into one item. **Generate Research Brief** uses Claude and the structured output parser to create the summary, topic findings, source references and follow-up items.<br><br>The parser checks structure, not factual accuracy. Source-ID validation does not independently verify the claims; review the brief before relying on it. |
| **Create HTML Brief** | `n8n-nodes-base.code` | Validates data and builds responsive HTML document | Generate Research Brief | Generate PDF with PDF.co | ## Create and download PDF<br><br>**Create HTML Brief** validates topic names and source IDs, escapes the generated text and applies the layout.<br><br>**Generate PDF with PDF.co** converts the HTML into a PDF. **Download PDF File** retrieves that PDF into binary field `data` for delivery. |
| **Save Brief to Drive** | `n8n-nodes-base.googleDrive` | Uploads the PDF file to Google Drive | Download PDF File | None | ## Deliver and archive brief<br><br>The downloaded PDF feeds two delivery branches: **Send Brief via Email** attaches binary field `data` to a Gmail message; **Save Brief to Drive** uploads the same file to Google Drive. Both connect directly to **Download PDF File**.<br><br>### Google Drive setup<br><br>After connecting your Google Drive account in **Save Brief to Drive**, select **Parent Drive** from the dropdown. Leave **Parent Folder** set to **By ID**, and enter your destination folder's ID in `drive_folder_id` in **Set Brief Parameters**. Do not leave that setting blank.<br><br>### Before testing<br><br>Confirm the recipient and archive destination. Executing these nodes sends a real email and creates a Drive file. Check existing deliveries before rerunning a partially failed execution to avoid duplicates. |
| **Send Brief via Email** | `n8n-nodes-base.gmail` | Sends the PDF attachment via Gmail | Download PDF File | None | ## Deliver and archive brief<br><br>The downloaded PDF feeds two delivery branches: **Send Brief via Email** attaches binary field `data` to a Gmail message; **Save Brief to Drive** uploads the same file to Google Drive. Both connect directly to **Download PDF File**.<br><br>### Google Drive setup<br><br>After connecting your Google Drive account in **Save Brief to Drive**, select **Parent Drive** from the dropdown. Leave **Parent Folder** set to **By ID**, and enter your destination folder's ID in `drive_folder_id` in **Set Brief Parameters**. Do not leave that setting blank.<br><br>### Before testing<br><br>Confirm the recipient and archive destination. Executing these nodes sends a real email and creates a Drive file. Check existing deliveries before rerunning a partially failed execution to avoid duplicates. |
| **Weekly Trigger** | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow every Monday at 07:00 | None | Set Brief Parameters | ## Schedule and brief settings<br><br>**Weekly Trigger** defaults to Monday at 07:00. Set the intended workflow timezone before enabling the schedule.<br><br>Edit **Set Brief Parameters** for topics, recipient, branding, folder ID and source limits. Keep `results_per_topic` and `max_source_characters` as Numbers. |
| **Split Topics Individually** | `n8n-nodes-base.splitOut` | Expands normalized topic array into individual items | Set Normalized Topics | Brave Search Topics | ## Prepare topic searches<br><br>Normalizes the comma-separated topic settings into a topic list, splits them into individual items, and sends each topic to Brave Search. |
| **Brave Search Topics** | `@brave/n8n-nodes-brave-search.braveSearch` | Executes web search queries via Brave | Split Topics Individually | Format Source Data | ## Collect source content<br><br>**Format Source Data** caps search results per topic and assigns source IDs. **Extract with Firecrawl** reads the pages; **Clean Source Text** cleans and shortens the text.<br><br>**Current limitation:** an unsuccessful scrape stops the run. The search-snippet fallback only covers a successful response with empty page text. |
| **Extract with Firecrawl** | `@mendable/n8n-nodes-firecrawl.firecrawl` | Scrapes target web pages to extract Markdown | Format Source Data | Clean Source Text | ## Collect source content<br><br>**Format Source Data** caps search results per topic and assigns source IDs. **Extract with Firecrawl** reads the pages; **Clean Source Text** cleans and shortens the text.<br><br>**Current limitation:** an unsuccessful scrape stops the run. The search-snippet fallback only covers a successful response with empty page text. |
| **Generate PDF with PDF.co** | `n8n-nodes-pdfco.PDFco Api` | Converts HTML input into an A4 PDF document | Create HTML Brief | Download PDF File | ## Create and download PDF<br><br>**Create HTML Brief** validates topic names and source IDs, escapes the generated text and applies the layout.<br><br>**Generate PDF with PDF.co** converts the HTML into a PDF. **Download PDF File** retrieves that PDF into binary field `data` for delivery. |
| **Download PDF File** | `n8n-nodes-base.httpRequest` | Retrieves the generated PDF binary file | Generate PDF with PDF.co | Send Brief via Email, Save Brief to Drive | ## Create and download PDF<br><br>**Create HTML Brief** validates topic names and source IDs, escapes the generated text and applies the layout.<br><br>**Generate PDF with PDF.co** converts the HTML into a PDF. **Download PDF File** retrieves that PDF into binary field `data` for delivery. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow in n8n:

1. **Create Schedule Trigger:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Set interval rule to Cron expression: `0 7 * * MON`.
   - Set workflow timezone in workflow settings.

2. **Configure Parameters:**
   - Add a **Set** node (`n8n-nodes-base.set`), named `Set Brief Parameters`.
   - Configure string and number assignments: `topics_csv` (`document automation, AI document processing`), `results_per_topic` (`3`), `recipient_email` (target email), `brand_name` (`Your company`), `drive_folder_id` (Google Drive folder ID), `max_source_characters` (`8000`), `logo_url` (optional URL).
   - Connect `Weekly Trigger` -> `Set Brief Parameters`.

3. **Normalize Topics:**
   - Add a second **Set** node (`n8n-nodes-base.set`), named `Set Normalized Topics`.
   - Add assignment `topics` with expression: `={{ $json.topics_csv.split(',').map(topic => topic.trim()).filter(Boolean) }}`.
   - Connect `Set Brief Parameters` -> `Set Normalized Topics`.

4. **Split Topics:**
   - Add a **Split Out** node (`n8n-nodes-base.splitOut`), named `Split Topics Individually`.
   - Set field to split out to `topics` and destination field name to `topic`.
   - Connect `Set Normalized Topics` -> `Split Topics Individually`.

5. **Perform Brave Searches:**
   - Add a **Brave Search** node (`@brave/n8n-nodes-brave-search.braveSearch`), named `Brave Search Topics`.
   - Configure query: `={{ $json.topic }}`, count: `={{ $json.results_per_topic }}`. Set freshness to past week (`pw`) and result filter to `web`.
   - Configure Brave Search API credentials (or Gateway credits).
   - Connect `Split Topics Individually` -> `Brave Search Topics`.

6. **Format Source Data:**
   - Add a **Code** node (`n8n-nodes-base.code`), named `Format Source Data`.
   - Paste JavaScript code validating search responses, enforcing limits, filtering HTTP URLs, and assigning source IDs (`s1`, `s2`, etc.).
   - Connect `Brave Search Topics` -> `Format Source Data`.

7. **Scrape via Firecrawl:**
   - Add a **Firecrawl** node (`@mendable/n8n-nodes-firecrawl.firecrawl`), named `Extract with Firecrawl`.
   - Set operation to `scrape` with URL expression: `={{ $json.url }}`.
   - Configure Firecrawl API credentials (or Gateway credits).
   - Connect `Format Source Data` -> `Extract with Firecrawl`.

8. **Clean Source Text:**
   - Add a **Code** node (`n8n-nodes-base.code`), named `Clean Source Text`.
   - Paste JavaScript code normalizing markdown, removing image links, truncating content, and managing search snippet fallbacks.
   - Connect `Extract with Firecrawl` -> `Clean Source Text`.

9. **Aggregate Sources:**
   - Add an **Aggregate** node (`n8n-nodes-base.aggregate`), named `Combine Sources`.
   - Set aggregation mode to `aggregateAllItemData` and destination field name to `sources`.
   - Connect `Clean Source Text` -> `Combine Sources`.

10. **Configure LLM and Output Parser:**
    - Add an **Anthropic Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatAnthropic`), named `Claude Sonnet Model`. Set model to `Claude Sonnet 5` and configure Anthropic API credentials.
    - Add a **Structured Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserStructured`), named `Parse Structure Output`. Provide the JSON schema example for summary, sections, and watch items.

11. **Generate Research Brief:**
    - Add an **Advanced AI / LLM Chain** node (`@n8n/n8n-nodes-langchain.chainLlm`), named `Generate Research Brief`.
    - Configure prompt instructions enforcing strict evidence and recency rules.
    - Connect `Combine Sources` (main), `Claude Sonnet Model` (ai_languageModel), and `Parse Structure Output` (ai_outputParser) into `Generate Research Brief`.

12. **Build HTML Brief:**
    - Add a **Code** node (`n8n-nodes-base.code`), named `Create HTML Brief`.
    - Paste JavaScript code validating JSON structure, checking source ID citations, escaping HTML, and assembling the HTML document and footer template.
    - Connect `Generate Research Brief` -> `Create HTML Brief`.

13. **Generate PDF:**
    - Add a **PDF.co** node (`n8n-nodes-pdfco.PDFco Api`), named `Generate PDF with PDF.co`.
    - Set operation to `URL/HTML to PDF`, conversion type to `htmlToPDF`. Set HTML expression to `={{ $json.html }}`, name to `={{ $json.filename }}`, footer to `={{ $json.footer_html }}`, margins to `22mm 18mm 20mm 18mm`, paper size to `A4`, and enable background printing.
    - Configure PDF.co API credentials (or Gateway credits).
    - Connect `Create HTML Brief` -> `Generate PDF with PDF.co`.

14. **Download PDF File:**
    - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`), named `Download PDF File`.
    - Set method to GET, URL to `={{ $json.url }}`, and response format to `File`.
    - Connect `Generate PDF with PDF.co` -> `Download PDF File`.

15. **Distribute and Archive:**
    - Add a **Gmail** node (`n8n-nodes-base.gmail`), named `Send Brief via Email`. Set operation to send, recipient to `={{ $('Set Brief Parameters').first().json.recipient_email }}`, subject to `={{ $('Create HTML Brief').first().json.email_subject }}`, and attach binary file `data`. Configure Gmail OAuth2 credentials.
    - Add a **Google Drive** node (`n8n-nodes-base.googleDrive`), named `Save Brief to Drive`. Set operation to upload, file name to `={{ $('Create HTML Brief').first().json.filename }}`, and parent folder ID to `={{ $('Set Brief Parameters').first().json.drive_folder_id }}`. Configure Google Drive OAuth2 credentials.
    - Connect `Download PDF File` output to both `Send Brief via Email` and `Save Brief to Drive`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Gateway credits support and integration | Showcases providers of Gateway credits, featuring PDF.co combined with services that are easy to start using in n8n Cloud. |
| AI content review requirement | AI-generated findings and citations require manual review. Search freshness alone does not prove an event happened during the reporting window. |