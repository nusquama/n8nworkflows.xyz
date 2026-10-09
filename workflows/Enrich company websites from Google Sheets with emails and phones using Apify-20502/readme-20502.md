Enrich company websites from Google Sheets with emails and phones using Apify

https://n8nworkflows.xyz/workflows/enrich-company-websites-from-google-sheets-with-emails-and-phones-using-apify-20502


# Enrich company websites from Google Sheets with emails and phones using Apify

### 1. Workflow Overview

This workflow is designed to automate lead enrichment by reading company websites from a Google Sheets document, processing them through Apify to extract contact details (emails, phone numbers, and social profiles), and saving the structured results back to another Google Sheets document.

The execution logic is divided into three functional blocks:
- **1.1 Input Preparation:** Initiates the process manually, defines runtime parameters for scraping depth and cost controls, reads company data from Google Sheets, and normalizes and de-duplicates the target domain list.
- **1.2 Scraping and Extraction:** Executes the Apify Contact Details Scraper actor as a batch job using the consolidated list of domains.
- **1.3 Result Formatting and Persistence:** Transforms the raw scraper dataset into flattened, spreadsheet-ready rows and appends them to a destination Google Sheets worksheet.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Preparation

- **Overview:** This block handles manual execution initialization, establishes global configuration parameters, ingests source records from Google Sheets, and aggregates the target websites into a clean, de-duplicated array.
- **Nodes Involved:** 
  - `When clicking ‘Test workflow’`
  - `Settings`
  - `Read websites`
  - `Collect unique websites`

- **Node Details:**
  - **`When clicking ‘Test workflow’`**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` — Serves as the manual entry point for test executions.
    - *Configuration Choices:* Standard default configuration (no parameters required).
    - *Input/Output Connections:* Output connects to `Settings`.
    - *Edge Cases / Potential Failures:* None.
  - **`Settings`**
    - *Type and Technical Role:* `n8n-nodes-base.set` — Defines global execution variables (`maxPagesPerWebsite` and `maxCostUsd`) to control crawl depth and spend limits.
    - *Configuration Choices:* Assigns `maxPagesPerWebsite` = 5 (number) and `maxCostUsd` = 5 (number).
    - *Input/Output Connections:* Input from `When clicking ‘Test workflow’`; output connects to `Read websites`.
    - *Edge Cases / Potential Failures:* Invalid numeric values could cause downstream actor configuration issues.
  - **`Read websites`**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Reads rows from a specified Google Sheets document.
    - *Configuration Choices:* Operation set to `read`. Document ID and Sheet Name must be configured by the user via the UI list/URL selectors.
    - *Input/Output Connections:* Input from `Settings`; output connects to `Collect unique websites`.
    - *Edge Cases / Potential Failures:* API authentication failure, missing document ID, or unselected sheet names will halt execution.
  - **`Collect unique websites`**
    - *Type and Technical Role:* `n8n-nodes-base.code` — JavaScript-based node that scans incoming rows for common website columns (`website`, `url`, `domain`), normalizes values into valid domains, and removes duplicates.
    - *Configuration Choices:* Uses custom JavaScript parsing logic leveraging `url` constructors to extract lowercase hostnames without `www.` prefixes. Throws an error if no valid websites are found.
    - *Key Expressions / Variables:* `$input.all()`
    - *Input/Output Connections:* Input from `Read websites`; output connects to `Find contacts (Contact Details Scraper)`.
    - *Edge Cases / Potential Failures:* Throws an explicit error if zero valid websites are identified in the input dataset.

---

#### Block 1.2: Scraping and Extraction

- **Overview:** This block invokes the Apify platform to execute a bulk scrape of the collected domain list using the Contact Details Scraper actor.
- **Nodes Involved:**
  - `Find contacts (Contact Details Scraper)`

- **Node Details:**
  - **`Find contacts (Contact Details Scraper)`**
    - *Type and Technical Role:* `@apify/n8n-nodes-apify.apify` — Community node that runs an Apify Actor and retrieves the resulting dataset.
    - *Configuration Choices:* Resource set to `Actors`, operation set to `Run actor and get dataset`. Actor ID set to `jipdiW9Rwbp1Lruzx`. Memory configured to 1024 MB. Custom body constructed via expression to pass the website array and max pages limit. Max total charge capped via expression.
    - *Key Expressions / Variables:* 
      - Custom Body: `={{ JSON.stringify({ websites: $json.websites, maxPagesPerWebsite: $('Settings').first().json.maxPagesPerWebsite }) }}`
      - Max Total Charge: `={{ $('Settings').first().json.maxCostUsd }}`
    - *Input/Output Connections:* Input from `Collect unique websites`; output connects to `Flatten for spreadsheet`.
    - *Credentials Required:* Apify API Token.
    - *Edge Cases / Potential Failures:* Insufficient Apify account balance, invalid Actor ID, or API rate limits/timeouts during large batch scrapes.

---

#### Block 1.3: Result Formatting and Persistence

- **Overview:** This block normalizes the raw hierarchical JSON output from the Apify scraper into flat key-value pairs and appends them to a destination Google Sheets document.
- **Nodes Involved:**
  - `Flatten for spreadsheet`
  - `Append contacts`

- **Node Details:**
  - **`Flatten for spreadsheet`**
    - *Type and Technical Role:* `n8n-nodes-base.code` — JavaScript-based transformation node that maps nested scraper fields into a clean flat schema for spreadsheet compatibility.
    - *Configuration Choices:* Maps fields such as `domain` to `website`, `companyName` to `company`, `primaryEmail` to `email`, phone numbers, social media links, contact pages, and mailbox providers.
    - *Key Expressions / Variables:* `$input.all()`
    - *Input/Output Connections:* Input from `Find contacts (Contact Details Scraper)`; output connects to `Append contacts`.
    - *Edge Cases / Potential Failures:* Missing optional properties in the scraper payload default to empty strings.
  - **`Append contacts`**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Appends rows to a specified Google Sheets document.
    - *Configuration Choices:* Operation set to `append`, mapping mode set to `autoMapInputData`. Document ID and Sheet Name must be configured by the user.
    - *Input/Output Connections:* Input from `Flatten for spreadsheet`; execution ends here.
    - *Credentials Required:* Google OAuth2 / Google Sheets account.
    - *Edge Cases / Potential Failures:* Column header mismatches between the auto-mapped input data and the target spreadsheet can lead to misplaced or rejected data.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Note 167cce | n8n-nodes-base.stickyNote | Workflow documentation and overview | None | None | # Enrich company websites from Google Sheets with emails and phone numbers using Apify<br><br>## Who's it for<br>Agencies, freelancers and sales teams who have a list of company websites and need contact details to reach them: emails, phone numbers and social profiles.<br><br>## How it works<br>1. Reads company websites from a Google Sheet (column `website`, `url` or `domain`).<br>2. Removes duplicates and sends all websites to the **Contact Details Scraper** on Apify in a single run.<br>3. The scraper visits each homepage plus its contact and about pages, decodes protected emails, validates phone numbers and checks that each email's domain can receive mail.<br>4. Writes one clean row per company to a results sheet: best email, all emails, phone, Facebook, Instagram, LinkedIn, X, YouTube, contact page and email provider.<br><br>## How to set up<br>1. Create a free Apify account and connect it in the **Find contacts** node (API token from Apify Console → Settings → API & Integrations).<br>2. Connect Google Sheets, pick your input sheet in **Read websites** and an empty results sheet in **Append contacts**.<br>3. Optionally adjust **Settings** (pages per website, spend cap).<br><br>## Requirements<br>- Apify account. You pay only for websites where an email or phone is found (about $3 per 1,000).<br>- Google Sheets account.<br>- Uses the verified Apify community node. Self-hosted users can install `@apify/n8n-nodes-apify`.<br><br>## How to customize the workflow<br>Swap Google Sheets for Airtable, HubSpot or a CSV. Add a filter after **Flatten for spreadsheet** to keep only rows with an email. |
| Note 6b16f2 | n8n-nodes-base.stickyNote | Input stage documentation | None | None | ### 1. Input<br>Run manually, then read your websites from Google Sheets. Set pages per website and the spend cap in **Settings**. |
| Note 7af13c | n8n-nodes-base.stickyNote | Find contacts stage documentation | None | None | ### 2. Find contacts<br>All unique websites go to the Contact Details Scraper in **one** Apify run. You pay only for websites where an email or phone is found. |
| Note 122cf9 | n8n-nodes-base.stickyNote | Save stage documentation | None | None | ### 3. Save<br>One clean row per company: email, phone, socials, contact page, email provider. |
| When clicking ‘Test workflow’ | n8n-nodes-base.manualTrigger | Manual execution trigger | None | Settings | |
| Settings | n8n-nodes-base.set | Sets crawl depth and spend limits | When clicking ‘Test workflow’ | Read websites | |
| Read websites | n8n-nodes-base.googleSheets | Reads source websites from Google Sheets | Settings | Collect unique websites | |
| Collect unique websites | n8n-nodes-base.code | Normalizes and de-duplicates domain list | Read websites | Find contacts (Contact Details Scraper) | |
| Find contacts (Contact Details Scraper) | @apify/n8n-nodes-apify.apify | Runs Apify Contact Details Scraper actor | Collect unique websites | Flatten for spreadsheet | Pay only for websites with an email or phone |
| Flatten for spreadsheet | n8n-nodes-base.code | Flattens scraper payload into schema rows | Find contacts (Contact Details Scraper) | Append contacts | |
| Append contacts | n8n-nodes-base.googleSheets | Writes enriched contact data to Google Sheets | Flatten for spreadsheet | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**
   - Add a `When clicking ‘Test workflow’` node (`n8n-nodes-base.manualTrigger`).
2. **Add Configuration Parameters:**
   - Add a `Set` node named `Settings` (`n8n-nodes-base.set`).
   - Define two number assignments: `maxPagesPerWebsite` (value: `5`) and `maxCostUsd` (value: `5`).
   - Connect `When clicking ‘Test workflow’` to `Settings`.
3. **Configure Source Input (Google Sheets):**
   - Add a Google Sheets node named `Read websites` (`n8n-nodes-base.googleSheets`).
   - Set operation to `read`. Select your target Document ID and Worksheet name containing website columns.
   - Connect `Settings` to `Read websites`.
4. **Add Domain Normalization & De-duplication Logic:**
   - Add a Code node named `Collect unique websites` (`n8n-nodes-base.code`).
   - Insert JavaScript code that iterates over `$input.all()`, extracts domains from fields (`website`, `url`, `domain`), normalizes URLs, eliminates duplicates using a `Set`, and outputs `{ websites, count }`.
   - Connect `Read websites` to `Collect unique websites`.
5. **Configure Apify Actor Integration:**
   - Add an Apify community node named `Find contacts (Contact Details Scraper)` (`@apify/n8n-nodes-apify.apify`).
   - Configure credentials with your Apify API Token.
   - Set Resource to `Actors`, Operation to `Run actor and get dataset`, and Actor ID to `jipdiW9Rwbp1Lruzx`. Set Memory to `1024`.
   - Set Custom Body expression: `={{ JSON.stringify({ websites: $json.websites, maxPagesPerWebsite: $('Settings').first().json.maxPagesPerWebsite }) }}`.
   - Set Max Total Charge USD expression: `={{ $('Settings').first().json.maxCostUsd }}`.
   - Connect `Collect unique websites` to `Find contacts (Contact Details Scraper)`.
6. **Format Data for Persistence:**
   - Add a Code node named `Flatten for spreadsheet` (`n8n-nodes-base.code`).
   - Insert JavaScript code mapping raw scraper properties (`domain`, `companyName`, `primaryEmail`, `allEmails`, `primaryPhone`, `allPhones`, social profiles, etc.) to a flat JSON structure.
   - Connect `Find contacts (Contact Details Scraper)` to `Flatten for spreadsheet`.
7. **Configure Destination Output (Google Sheets):**
   - Add a Google Sheets node named `Append contacts` (`n8n-nodes-base.googleSheets`).
   - Set operation to `append` and configure mapping mode to `autoMapInputData`. Select the destination Document ID and Worksheet name.
   - Connect `Flatten for spreadsheet` to `Append contacts`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Verified Apify Community Node requirement (`@apify/n8n-nodes-apify`) | On self-hosted n8n instances, install via Settings > Community nodes. |
| Compliance Warning | Contact details are extracted from public business websites; ensure compliance with CAN-SPAM, GDPR, and relevant regional communication regulations. |