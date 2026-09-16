Find high-value expired domains with DomainKits and Moz and email a CSV report

https://n8nworkflows.xyz/workflows/find-high-value-expired-domains-with-domainkits-and-moz-and-email-a-csv-report-18101


# Find high-value expired domains with DomainKits and Moz and email a CSV report

### 1. Workflow Overview

This workflow automates the daily discovery, SEO enrichment, and reporting of high-value expired domain candidates. It operates on a fixed schedule to source domains matching specific keyword and age criteria, evaluates their authority via an external SEO provider, formats the data into a readable summary along with a CSV attachment, and delivers the report via email.

The workflow logic is categorized into three functional blocks:
- **1.1 Input Reception & Domain Discovery:** Handles execution scheduling, initializes search parameters, queries the DomainKits API for expired domains matching filters, and aggregates the results into a batch payload.
- **1.2 SEO Enrichment:** Sends the batched domains to the Moz Links API to retrieve performance and authority metrics (Domain Authority, Page Authority, Spam Score, and linking root domains).
- **1.3 Report Generation & Delivery:** Merges the domain records with SEO metrics, sorts them by Domain Authority, constructs a textual email body, converts the structured rows into a CSV file, and dispatches the complete report via SMTP.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Domain Discovery
- **Overview:** Initializes the daily execution cycle, defines target search criteria, queries DomainKits for relevant domains across the deletion lifecycle, and formats the output into a unified array for batch processing.
- **Nodes Involved:** `Every Day at 2pm`, `Set Target`, `Find Expired Domains`, `Prepare Batch`.
- **Node Details:**
  - **Every Day at 2pm**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node). Initiates workflow execution daily at 14:00.
    - *Configuration Choices:* Interval set to days, trigger hour configured to 14.
    - *Input/Output:* No inputs; outputs execution timestamp to `Set Target`.
    - *Edge Cases/Failures:* Missed executions if n8n instance is offline at the scheduled hour.
  - **Set Target**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation Node). Establishes workflow variables for subsequent API requests.
    - *Configuration Choices:* Assigns custom string and number parameters.
    - *Key Expressions:* `keyword` = `"travel"`, `domain_limit` = `50`.
    - *Input/Output:* Input from `Every Day at 2pm`; output to `Find Expired Domains`.
  - **Find Expired Domains**
    - *Type and Technical Role:* `n8n-nodes-domainkits.domainKits` (Service Integration Node). Queries DomainKits API for expired or deleting domains.
    - *Configuration Choices:* Operation set to `search`, resource set to `expired`, search mode set to `keyword`. Applies filters for age range (`10-20,20+`), disables hyphens (`no_hyphen: true`) and numbers (`no_number: true`). Limit mapped to variable.
    - *Key Expressions:* Keyword: `={{ $json.keyword }}`, Limit: `={{ $json.domain_limit }}`.
    - *Input/Output:* Input from `Set Target`; output to `Prepare Batch`.
    - *Version-Specific / Credential Requirements:* Requires a DomainKits Premium API key configured via n8n credentials.
    - *Edge Cases/Failures:* API rate limits, invalid API keys, or timeouts from the DomainKits service. `alwaysOutputData` is enabled to prevent workflow interruption on empty responses.
  - **Prepare Batch**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Execution Node). Maps and cleans raw domain items into a single array payload capped at 50 targets.
    - *Configuration Choices:* Custom JavaScript processing input items.
    - *Key Expressions:* Filters items containing valid domains and slices the array to a maximum of 50 elements.
    - *Input/Output:* Input from `Find Expired Domains`; output to `Get Moz Metrics`.
    - *Edge Cases/Failures:* Empty item lists returned if no domains match the initial search criteria.

#### 2.2 SEO Enrichment
- **Overview:** Communicates with the Moz Links API via a POST request to retrieve key SEO metrics for the batched domain candidates in a single HTTP call.
- **Nodes Involved:** `Get Moz Metrics`.
- **Node Details:**
  - **Get Moz Metrics**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (External API Integration Node). Sends domain targets to Moz to extract authority and link metrics.
    - *Configuration Choices:* Method: `POST`, URL: `https://lsapi.seomoz.com/v2/url_metrics`. Uses HTTP Basic Authentication.
    - *Key Expressions:* JSON body payload dynamically constructed using `={{ JSON.stringify({ targets: $json.domains }) }}`.
    - *Input/Output:* Input from `Prepare Batch`; output to `Build Report`.
    - *Credential Requirements:* Requires Moz Access ID and Secret Key configured via HTTP Basic Authentication credentials.
    - *Error Handling:* Configured with `onError: continueRegularOutput`, `retryOnFail: true`, up to `3` max tries, and a `2000ms` wait between tries. `alwaysOutputData` is enabled.
    - *Edge Cases/Failures:* Authentication failures due to invalid credentials, API rate limiting, or malformed JSON payloads.

#### 2.3 Report Generation & Delivery
- **Overview:** Combines DomainKits registration and status data with Moz SEO metrics, sorts candidates by authority, builds a text-based email message, converts the dataset into a CSV file, and sends the report via email.
- **Nodes Involved:** `Build Report`, `Split Records`, `Records To CSV`, `Send Report`.
- **Node Details:**
  - **Build Report**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Execution Node). Merges API datasets, sorts records, and compiles formatted text summary and metadata.
    - *Configuration Choices:* Custom JavaScript processing data mapping, normalization, sorting (descending by Domain Authority, secondary alphabetical by domain), and string formatting.
    - *Key Expressions:* References upstream data from `$('Prepare Batch')`, `$('Set Target')`, and current node JSON.
    - *Input/Output:* Input from `Get Moz Metrics`; output to `Split Records`.
  - **Split Records**
    - *Type and Technical Role:* `n8n-nodes-base.splitOut` (Data Transformation Node). Converts the array of records inside the single JSON payload back into individual items for file conversion.
    - *Configuration Choices:* Field to split out: `records`.
    - *Input/Output:* Input from `Build Report`; output to `Records To CSV`.
  - **Records To CSV**
    - *Type and Technical Role:* `n8n-nodes-base.convertToFile` (Utility Node). Transforms structured JSON items into a CSV format string/binary attachment.
    - *Configuration Choices:* Operation set to `csv`.
    - *Input/Output:* Input from `Split Records`; output to `Send Report`.
  - **Send Report**
    - *Type and Technical Role:* `n8n-nodes-base.emailSend` (Delivery Node). Dispatches the compiled email report with the attached CSV file via SMTP.
    - *Configuration Choices:* Format set to `text`, options configured to attach data (`attachments: "data"`).
    - *KeyExpressions:* 
      - Subject: `={{ $('Build Report').first().json.subject }}`
      - Text: `={{ $('Build Report').first().json.message }}`
    - *Input/Output:* Input from `Records To CSV`; terminal node in the workflow.
    - *Credential Requirements:* Requires SMTP credentials and valid sender/recipient addresses.
    - *Edge Cases/Failures:* SMTP connection failures, authentication errors, or rejected sender/recipient addresses.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every Day at 2pm | n8n-nodes-base.scheduleTrigger | Initiates workflow daily at 14:00 | None | Set Target | ## Find valuable expired domains with DomainKits and Moz<br><br>### How it works<br><br>Once a day this workflow finds expired domains: names moving through the deletion lifecycle (expired, redemption, pending delete) that match your keyword. These names surface here before they reach drop lists and auctions, which is the point: you see them while there is still time to research and backorder.<br><br>The candidates then go to the Moz Links API in one batch call. Each domain comes back with Domain Authority, Page Authority, Spam Score and linking root domains. The report lists each candidate with its expiry stage and SEO metrics, sorted by Domain Authority. The workflow reports what the data says and leaves the judgement to you.<br><br>### Setup steps<br><br>- Set your keyword in the Set Target node. The downstream nodes derive from it.<br>- Add your DomainKits API credentials to the Find Expired Domains node.<br>- Create a free Moz account, generate an Access ID and Secret Key under API Access, and add them as a Basic Auth credential on the Get Moz Metrics node.<br>- Configure the Send Report node with your SMTP credentials and recipients.<br><br>### Customization<br><br>Swap the Send Report node for Google Sheets to build a rolling candidate pool instead of a daily email. Tighten the filters in Find Expired Domains (age, length, character set) to raise the bar before Moz spends a row on a name. Swap the Get Moz Metrics node for your Ahrefs or Semrush endpoint if that is where your subscription lives; only the URL, the auth and the field names in Build Report change.<br><br>### Data note<br><br>Results contain domain names, registration dates, expiry stages and SEO metrics only. No registrant personal data is included (no names, emails, addresses or phone numbers), so the output is GDPR compliant and safe to forward or store. |
| Set Target | n8n-nodes-base.set | Assigns keyword and domain limit variables | Every Day at 2pm | Find Expired Domains | ## Find valuable expired domains with DomainKits and Moz<br><br>### How it works... |
| Find Expired Domains | n8n-nodes-domainkits.domainKits | Queries expired domains matching keyword and filter criteria | Set Target | Prepare Batch | ## Collect<br><br>Runs daily and searches expired domains across the deletion lifecycle that contain the keyword. Age and character filters keep the list to names with real history. |
| Prepare Batch | n8n-nodes-base.code | Formats and limits domain candidate list for batch request | Find Expired Domains | Get Moz Metrics | ## Collect<br><br>Runs daily and searches expired domains across the deletion lifecycle that contain the keyword. Age and character filters keep the list to names with real history. |
| Get Moz Metrics | n8n-nodes-base.httpRequest | Fetches SEO metrics from Moz Links API | Prepare Batch | Build Report | ## Score with Moz<br><br>Sends the candidate list to the Moz Links API in one call and reads Domain Authority, Page Authority, Spam Score and linking root domains for each name. |
| Build Report | n8n-nodes-base.code | Merges data, sorts results, and constructs message body | Get Moz Metrics | Split Records | ## Report and deliver<br><br>Merges expiry data with SEO metrics, sorts by Domain Authority, converts the records to a CSV attachment and emails both. |
| Split Records | n8n-nodes-base.splitOut | Explodes records array into individual items | Build Report | Records To CSV | ## Report and deliver<br><br>Merges expiry data with SEO metrics, sorts by Domain Authority, converts the records to a CSV attachment and emails both. |
| Records To CSV | n8n-nodes-base.convertToFile | Converts individual record items into CSV format | Split Records | Send Report | ## Report and deliver<br><br>Merges expiry data with SEO metrics, sorts by Domain Authority, converts the records to a CSV attachment and emails both. |
| Send Report | n8n-nodes-base.emailSend | Emails textual summary and CSV attachment via SMTP | Records To CSV | None | ## Report and deliver<br><br>Merges expiry data with SEO metrics, sorts by Domain Authority, converts the records to a CSV attachment and emails both. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Name it `Every Day at 2pm`.
   - Set interval to `Days` and trigger hour to `14`.

2. **Configure Search Parameters:**
   - Add a **Set** node (`n8n-nodes-base.set`) connected to the output of `Every Day at 2pm`.
   - Name it `Set Target`.
   - Add assignments:
     - String parameter: Name `keyword`, Value `travel`.
     - Number parameter: Name `domain_limit`, Value `50`.

3. **Query DomainKits:**
   - Add a **DomainKits** node (`n8n-nodes-domainkits.domainKits`) connected from `Set Target`.
   - Name it `Find Expired Domains`.
   - Set **Resource** to `expired`, **Operation** to `search`, and **Search Mode** to `keyword`.
   - Set **Keyword** expression: `={{ $json.keyword }}`.
   - Set **Limit** expression: `={{ $json.domain_limit }}`.
   - Configure **Filters**: Age Range `10-20,20+`, No Hyphen `true`, No Number `true`.
   - Configure a DomainKits API credential (requires a Premium plan). Enable `Always Output Data`.

4. **Prepare Batch Payload:**
   - Add a **Code** node (`n8n-nodes-base.code`) connected from `Find Expired Domains`.
   - Name it `Prepare Batch`.
   - Set JavaScript code to extract valid domains and package them into an array capped at 50 items:
     ```javascript
     const candidates = $input.all().map(i => i.json).filter(d => d.domain);
     return [{
       json: {
         domains: candidates.map(d => d.domain).slice(0, 50),
         candidates,
       },
     }];
     ```

5. **Fetch SEO Metrics:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) connected from `Prepare Batch`.
   - Name it `Get Moz Metrics`.
   - Set **Method** to `POST` and **URL** to `https://lsapi.seomoz.com/v2/url_metrics`.
   - Set **Authentication** to `Generic Credential Type` -> `HTTP Basic Auth` (using Moz Access ID and Secret Key).
   - Set **Specify Body** to `JSON`, and populate **JSON Body** with: `={{ JSON.stringify({ targets: $json.domains }) }}`.
   - Configure error handling: Enable `Continue on Fail` (`continueRegularOutput`), retry on fail with max tries `3`, and wait `2000ms`. Enable `Always Output Data`.

6. **Build Report and Summary:**
   - Add a **Code** node (`n8n-nodes-base.code`) connected from `Get Moz Metrics`.
   - Name it `Build Report`.
   - Set JavaScript code to map Moz metrics back to domain candidates, sort by Domain Authority (descending), construct summary text lines, and bundle fields into an email object containing subject, message body, and records array.

7. **Split Records for CSV Conversion:**
   - Add a **Split Out** node (`n8n-nodes-base.splitOut`) connected from `Build Report`.
   - Name it `Split Records`.
   - Set **Field To Split Out** to `records`.

8. **Convert Records to CSV:**
   - Add a **Convert to File** node (`n8n-nodes-base.convertToFile`) connected from `Split Records`.
   - Name it `Records To CSV`.
   - Set **Operation** to `CSV`.

9. **Send Email Report:**
   - Add a **Send Email** node (`n8n-nodes-base.emailSend`) connected from `Records To CSV`.
   - Name it `Send Report`.
   - Configure **From Email** and **To Email** addresses.
   - Set **Subject** expression: `={{ $('Build Report').first().json.subject }}`.
   - Set **Text** expression: `={{ $('Build Report').first().json.message }}`.
   - Set **Email Format** to `Text`.
   - Under **Options**, set **Attachments** to `data`.
   - Configure SMTP credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| API reference and key management | [https://domainkits.com/dev](https://domainkits.com/dev) |
| OpenAPI 3.0 spec | [https://domainkits.com/dev/openapi.yaml](https://domainkits.com/dev/openapi.yaml) |
| DomainKits platform signup | [https://domainkits.com](https://domainkits.com) |