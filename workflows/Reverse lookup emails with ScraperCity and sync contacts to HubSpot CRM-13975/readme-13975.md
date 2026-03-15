Reverse lookup emails with ScraperCity and sync contacts to HubSpot CRM

https://n8nworkflows.xyz/workflows/reverse-lookup-emails-with-scrapercity-and-sync-contacts-to-hubspot-crm-13975


# Reverse lookup emails with ScraperCity and sync contacts to HubSpot CRM

# 1. Workflow Overview

This workflow performs a reverse email lookup through ScraperCity’s People Finder API, waits for the asynchronous scrape job to complete, downloads the resulting contact data, normalizes and filters it, and then creates or updates matching contacts in HubSpot CRM.

Typical use cases:
- Enriching a list of email addresses with public contact details
- Syncing reverse-lookup results into HubSpot as contact records
- Running manual prospecting or contact recovery batches from inside n8n

## 1.1 Input Reception and Configuration
The workflow starts manually and uses a Set node to define the target email list and the maximum number of lookup results to request from ScraperCity.

## 1.2 Scrape Job Submission
It submits the configured emails to ScraperCity’s People Finder endpoint, receives an asynchronous `runId`, and stores that identifier for later polling.

## 1.3 Asynchronous Polling Loop
After an initial delay, the workflow repeatedly checks the scrape job status. If the scrape is not complete, it waits and retries. If the status becomes `SUCCEEDED`, it proceeds to download the output.

## 1.4 Result Retrieval and Normalization
The workflow downloads the scrape output, parses the returned CSV or fallback JSON-like payload into itemized contact records, removes duplicates by email, filters out invalid rows, and maps fields into a HubSpot-friendly structure.

## 1.5 HubSpot Contact Upsert
Each normalized contact is upserted into HubSpot using the email address as the unique identifier, with first name, last name, phone, city, state, zip, and address context included.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Configuration

### Overview
This block defines the manual entry point and prepares lookup parameters for the scrape request. It is the main user-editable section of the workflow.

### Nodes Involved
- When clicking 'Execute workflow'
- Configure Lookup Parameters

### Node Details

#### 1) When clicking 'Execute workflow'
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry node used to start the workflow interactively from the editor.
- **Configuration choices:**  
  No custom parameters are configured.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Configure Lookup Parameters`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - Only runs when manually executed
  - Not suitable for unattended scheduling unless replaced by another trigger
- **Sub-workflow reference:**  
  None.

#### 2) Configure Lookup Parameters
- **Type and technical role:** `n8n-nodes-base.set`  
  Defines the input configuration values used downstream.
- **Configuration choices:**  
  Creates two fields:
  - `emails`: comma-separated string of target email addresses
  - `maxResults`: numeric maximum number of records to return per email
- **Key expressions or variables used:**  
  Static values in the example:
  - `emails = "user@example.com,user@example.com"`
  - `maxResults = 5`
- **Input and output connections:**  
  - Input: `When clicking 'Execute workflow'`
  - Output: `Start People Finder Scrape`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Empty `emails` string will lead to a useless request or possible API validation failure
  - Duplicate emails are allowed at this stage
  - Non-numeric `maxResults` would break expectations, though the node defines it as number
- **Sub-workflow reference:**  
  None.

---

## 2.2 Scrape Job Submission

### Overview
This block sends the configured lookup request to ScraperCity and captures the returned asynchronous job identifier. That `runId` is required for all later status and download requests.

### Nodes Involved
- Start People Finder Scrape
- Store Run ID
- Wait Before First Poll

### Node Details

#### 3) Start People Finder Scrape
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to ScraperCity’s People Finder scrape endpoint.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://app.scrapercity.com/api/v1/scrape/people-finder`
  - Body format: JSON
  - Authentication: generic credential type using HTTP Header Auth
- **Key expressions or variables used:**  
  JSON body is dynamically built:
  - `email`: derived from `emails.split(',').map(e => e.trim())`
  - `max_results`: from `$json.maxResults`
  - Other arrays are sent empty:
    - `name: []`
    - `phone_number: []`
    - `street_citystatezip: []`
- **Input and output connections:**  
  - Input: `Configure Lookup Parameters`
  - Output: `Store Run ID`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Missing or invalid ScraperCity credential causes auth failure
  - If `emails` contains malformed values, ScraperCity may reject the request or return poor results
  - If the API response does not include `runId`, downstream polling fails
  - Network timeout or rate limiting may interrupt the request
- **Sub-workflow reference:**  
  None.

#### 4) Store Run ID
- **Type and technical role:** `n8n-nodes-base.set`  
  Extracts and stores the `runId` from the ScraperCity response in a clean field.
- **Configuration choices:**  
  Creates one output field:
  - `runId = {{ $json.runId }}`
- **Key expressions or variables used:**  
  `={{ $json.runId }}`
- **Input and output connections:**  
  - Input: `Start People Finder Scrape`
  - Output: `Wait Before First Poll`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If the previous node does not return `runId`, this field becomes empty and downstream status checks break
- **Sub-workflow reference:**  
  None.

#### 5) Wait Before First Poll
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses the workflow before the first status check to allow the asynchronous job to start processing.
- **Configuration choices:**  
  Wait amount: `30`  
  No explicit unit is shown in the JSON; in practice this should be validated in the n8n UI after import or manual creation.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Store Run ID`
  - Output: `Poll Loop`
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  - If wait configuration is interpreted unexpectedly due to UI defaults, the delay may not match the intended 30 seconds
  - Wait nodes resume asynchronously and depend on proper n8n execution persistence setup in some environments
- **Sub-workflow reference:**  
  None.

---

## 2.3 Asynchronous Polling Loop

### Overview
This block repeatedly checks the status of the ScraperCity job until the result is ready. It branches on scrape completion and loops back through a wait node when the scrape is still in progress.

### Nodes Involved
- Poll Loop
- Check Scrape Status
- Is Scrape Complete?
- Wait Before Retry

### Node Details

#### 6) Poll Loop
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Used here as a loop control mechanism rather than for true batching.
- **Configuration choices:**  
  - `batchSize = 1`
  - `reset = false`
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  - Inputs:
    - `Wait Before First Poll`
    - `Wait Before Retry`
  - Output:
    - Main output 0 → `Check Scrape Status`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - This usage relies on a common n8n looping pattern; modifications can accidentally break the loop
  - If upstream items are malformed or missing, loop behavior can become confusing
- **Sub-workflow reference:**  
  None.

#### 7) Check Scrape Status
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Queries the ScraperCity status endpoint using the stored `runId`.
- **Configuration choices:**  
  - Method: `GET`
  - URL is dynamic:
    `https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Run ID').item.json.runId }}`
  - Authentication: HTTP Header Auth credential
- **Key expressions or variables used:**  
  `$('Store Run ID').item.json.runId`
- **Input and output connections:**  
  - Input: `Poll Loop`
  - Output: `Is Scrape Complete?`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - If `runId` is missing, the URL becomes invalid
  - Auth failures, rate limits, or temporary API errors can stop the loop
  - Unexpected status payload shape may break the next IF condition
- **Sub-workflow reference:**  
  None.

#### 8) Is Scrape Complete?
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on whether the scrape job is finished successfully.
- **Configuration choices:**  
  Checks whether:
  - `$json.status` equals `SUCCEEDED`
  - Comparison is case-sensitive with strict validation
- **Key expressions or variables used:**  
  `={{ $json.status }}`
- **Input and output connections:**  
  - Input: `Check Scrape Status`
  - True output → `Download Scrape Results`
  - False output → `Wait Before Retry`
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Jobs with statuses like `FAILED`, `CANCELLED`, or unexpected values are treated the same as “not complete” and will continue looping forever
  - If the API returns lowercase or differently formatted statuses, strict equality fails
- **Sub-workflow reference:**  
  None.

#### 9) Wait Before Retry
- **Type and technical role:** `n8n-nodes-base.wait`  
  Adds a delay between polling attempts.
- **Configuration choices:**  
  Wait amount: `60`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: false branch of `Is Scrape Complete?`
  - Output: `Poll Loop`
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Same persistence/resume considerations as other Wait nodes
  - No retry limit exists, so failed jobs may poll indefinitely
- **Sub-workflow reference:**  
  None.

---

## 2.4 Result Retrieval and Normalization

### Overview
This block downloads the completed scrape output and transforms it into clean, normalized contact items suitable for CRM import. It also deduplicates by email and removes rows without a valid email field.

### Nodes Involved
- Download Scrape Results
- Parse CSV Results
- Remove Duplicate Contacts
- Filter Valid Contacts
- Map Contact Fields

### Node Details

#### 10) Download Scrape Results
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the completed scrape output from ScraperCity.
- **Configuration choices:**  
  - Method: `GET`
  - URL:
    `https://app.scrapercity.com/api/downloads/{{ $('Store Run ID').item.json.runId }}`
  - Authentication: HTTP Header Auth credential
- **Key expressions or variables used:**  
  `$('Store Run ID').item.json.runId`
- **Input and output connections:**  
  - Input: true branch of `Is Scrape Complete?`
  - Output: `Parse CSV Results`
- **Version-specific requirements:**  
  Type version `4.2`.
- **Edge cases or potential failure types:**  
  - Invalid `runId` or expired download URL could fail
  - Response format may differ from expected CSV text
  - Large result sets may increase memory usage downstream
- **Sub-workflow reference:**  
  None.

#### 11) Parse CSV Results
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts the download response into one item per contact.
- **Configuration choices:**  
  Custom JavaScript parses multiple possible response formats:
  1. Raw string CSV
  2. JSON object containing `data` as string
  3. JSON object containing `csv` as string
  4. Fallback array/object payloads using `raw`, `raw.results`, or direct array content
- **Key expressions or variables used:**  
  Internally uses:
  - `$input.first().json`
  - Fallback logic for `raw.data`, `raw.csv`, `raw.results`
- **Input and output connections:**  
  - Input: `Download Scrape Results`
  - Output: `Remove Duplicate Contacts`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - CSV parsing is simplistic: it splits on commas and newlines only
  - Quoted CSV fields containing commas, line breaks, or escaped quotes may parse incorrectly
  - If the response is binary rather than JSON/string, this code will not handle it directly
  - Empty CSV returns no items
- **Sub-workflow reference:**  
  None.

#### 12) Remove Duplicate Contacts
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`  
  Deduplicates parsed records using the `email` field.
- **Configuration choices:**  
  - Compare field: `email`
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Parse CSV Results`
  - Output: `Filter Valid Contacts`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Deduplication happens before email normalization to lowercase in the next block, so case variants like `A@x.com` and `a@x.com` may not deduplicate
  - Blank emails may still survive until the filter step
- **Sub-workflow reference:**  
  None.

#### 13) Filter Valid Contacts
- **Type and technical role:** `n8n-nodes-base.filter`  
  Removes records that do not contain an `email` field.
- **Configuration choices:**  
  - Condition: `email` exists
  - Case-insensitive and loose validation enabled
- **Key expressions or variables used:**  
  `={{ $json.email }}`
- **Input and output connections:**  
  - Input: `Remove Duplicate Contacts`
  - Output: `Map Contact Fields`
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - “Exists” does not guarantee the email is valid, non-empty after trimming, or syntactically correct
  - Null-like or malformed values may still pass depending on actual payload shape
- **Sub-workflow reference:**  
  None.

#### 14) Map Contact Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes source field names into a stable schema for HubSpot.
- **Configuration choices:**  
  Maps and transforms:
  - `email` → trimmed lowercase
  - `firstName` from `first_name` or `firstName`
  - `lastName` from `last_name` or `lastName`
  - `phone` from `phone` or `phone_number`
  - `address` from `address` or `street`
  - `city`, `state`
  - `zip` from `zip` or `zipcode`
- **Key expressions or variables used:**  
  Examples:
  - `={{ ($json.email || '').trim().toLowerCase() }}`
  - `={{ $json.first_name || $json.firstName || '' }}`
- **Input and output connections:**  
  - Input: `Filter Valid Contacts`
  - Output: `Upsert Contact in HubSpot`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Missing fields become empty strings
  - No validation is applied to phone, zip, or address quality
  - If `email` is blank but present, normalization still produces empty string
- **Sub-workflow reference:**  
  None.

---

## 2.5 HubSpot Contact Upsert

### Overview
This final block writes each normalized contact into HubSpot, creating or updating the record based on the email address.

### Nodes Involved
- Upsert Contact in HubSpot

### Node Details

#### 15) Upsert Contact in HubSpot
- **Type and technical role:** `n8n-nodes-base.hubspot`  
  Creates or updates a HubSpot contact using app-token authentication.
- **Configuration choices:**  
  - Authentication: `appToken`
  - Email used as the primary identifier:
    `={{ $json.email }}`
  - Additional mapped properties:
    - `firstName`
    - `lastName`
    - `phone`
    - `city`
    - `state`
    - `zip`
    - `message`
  - The `message` field stores a formatted address string plus source attribution
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
  - `={{ $json.firstName }}`
  - `={{ $json.lastName }}`
  - `={{ $json.phone }}`
  - `={{ $json.city }}`
  - `={{ $json.state }}`
  - `={{ $json.zip }}`
  - `=Address: {{ $json.address }}, {{ $json.city }}, {{ $json.state }} {{ $json.zip }}`
    `Source: ScraperCity People Finder reverse lookup`
- **Input and output connections:**  
  - Input: `Map Contact Fields`
  - Output: none
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Invalid or missing HubSpot token causes authentication failure
  - If HubSpot property names differ from expected defaults, some fields may fail to write
  - The `message` property may not exist in some HubSpot portals or may not be intended for contact records
  - Empty email values will prevent proper upsert behavior
  - Rate limiting can occur on larger contact batches
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Manual workflow start |  | Configure Lookup Parameters | ## How it works<br>1. **Configure Lookup Parameters** -- set the email addresses you want to reverse-lookup and the max results per search.<br>2. **Start People Finder Scrape** -- submits the emails to ScraperCity and receives an async job ID.<br>3. **Polling loop** -- checks job status every 60 seconds until the scrape status is SUCCEEDED.<br>4. **Download and parse** -- fetches the CSV result, parses rows into contact records, deduplicates by email, and filters out rows missing an email.<br>5. **Upsert to HubSpot** -- maps name, phone, and address fields and creates or updates each contact in HubSpot CRM.<br><br>## Setup steps<br>1. Add a **ScraperCity API Key** credential: HTTP Header Auth, header `Authorization`, value `Bearer YOUR_KEY`.<br>2. Add a **HubSpot App Token** credential using your HubSpot private app token.<br>3. In **Configure Lookup Parameters**, replace the example emails with your target list (comma-separated).<br>4. Click **Execute workflow** to run. |
| Configure Lookup Parameters | Set | Defines emails and max results | When clicking 'Execute workflow' | Start People Finder Scrape | ## How it works<br>1. **Configure Lookup Parameters** -- set the email addresses you want to reverse-lookup and the max results per search.<br>2. **Start People Finder Scrape** -- submits the emails to ScraperCity and receives an async job ID.<br>3. **Polling loop** -- checks job status every 60 seconds until the scrape status is SUCCEEDED.<br>4. **Download and parse** -- fetches the CSV result, parses rows into contact records, deduplicates by email, and filters out rows missing an email.<br>5. **Upsert to HubSpot** -- maps name, phone, and address fields and creates or updates each contact in HubSpot CRM.<br><br>## Setup steps<br>1. Add a **ScraperCity API Key** credential: HTTP Header Auth, header `Authorization`, value `Bearer YOUR_KEY`.<br>2. Add a **HubSpot App Token** credential using your HubSpot private app token.<br>3. In **Configure Lookup Parameters**, replace the example emails with your target list (comma-separated).<br>4. Click **Execute workflow** to run.<br>## Configuration<br>**Configure Lookup Parameters** is the only node you must edit. Enter target emails as a comma-separated string. **max_results** controls how many address records ScraperCity returns per email. All credentials are managed through n8n credential store -- no API keys in node parameters. |
| Start People Finder Scrape | HTTP Request | Submits reverse-lookup scrape job to ScraperCity | Configure Lookup Parameters | Store Run ID | ## How it works<br>1. **Configure Lookup Parameters** -- set the email addresses you want to reverse-lookup and the max results per search.<br>2. **Start People Finder Scrape** -- submits the emails to ScraperCity and receives an async job ID.<br>3. **Polling loop** -- checks job status every 60 seconds until the scrape status is SUCCEEDED.<br>4. **Download and parse** -- fetches the CSV result, parses rows into contact records, deduplicates by email, and filters out rows missing an email.<br>5. **Upsert to HubSpot** -- maps name, phone, and address fields and creates or updates each contact in HubSpot CRM.<br><br>## Setup steps<br>1. Add a **ScraperCity API Key** credential: HTTP Header Auth, header `Authorization`, value `Bearer YOUR_KEY`.<br>2. Add a **HubSpot App Token** credential using your HubSpot private app token.<br>3. In **Configure Lookup Parameters**, replace the example emails with your target list (comma-separated).<br>4. Click **Execute workflow** to run.<br>## Scrape Submission<br>**Start People Finder Scrape** posts the email list to ScraperCity's People Finder endpoint. **Store Run ID** captures the returned `runId` so the polling loop can reference it. **Wait Before First Poll** pauses 30 seconds before the first status check. |
| Store Run ID | Set | Stores returned ScraperCity runId | Start People Finder Scrape | Wait Before First Poll | ## Scrape Submission<br>**Start People Finder Scrape** posts the email list to ScraperCity's People Finder endpoint. **Store Run ID** captures the returned `runId` so the polling loop can reference it. **Wait Before First Poll** pauses 30 seconds before the first status check. |
| Wait Before First Poll | Wait | Delays before first poll | Store Run ID | Poll Loop | ## Scrape Submission<br>**Start People Finder Scrape** posts the email list to ScraperCity's People Finder endpoint. **Store Run ID** captures the returned `runId` so the polling loop can reference it. **Wait Before First Poll** pauses 30 seconds before the first status check.<br>## Async Polling Loop<br>**Poll Loop** iterates the status check. **Check Scrape Status** queries `/api/v1/scrape/status/{runId}`. **Is Scrape Complete?** routes to download on `SUCCEEDED`. Otherwise **Wait Before Retry** pauses 60 seconds and routes back into **Poll Loop** to retry. ScraperCity scrapes typically finish in 10--60 minutes. |
| Poll Loop | Split In Batches | Controls retry loop for polling | Wait Before First Poll; Wait Before Retry | Check Scrape Status | ## Async Polling Loop<br>**Poll Loop** iterates the status check. **Check Scrape Status** queries `/api/v1/scrape/status/{runId}`. **Is Scrape Complete?** routes to download on `SUCCEEDED`. Otherwise **Wait Before Retry** pauses 60 seconds and routes back into **Poll Loop** to retry. ScraperCity scrapes typically finish in 10--60 minutes. |
| Check Scrape Status | HTTP Request | Queries ScraperCity job status | Poll Loop | Is Scrape Complete? | ## Async Polling Loop<br>**Poll Loop** iterates the status check. **Check Scrape Status** queries `/api/v1/scrape/status/{runId}`. **Is Scrape Complete?** routes to download on `SUCCEEDED`. Otherwise **Wait Before Retry** pauses 60 seconds and routes back into **Poll Loop** to retry. ScraperCity scrapes typically finish in 10--60 minutes. |
| Is Scrape Complete? | If | Branches on scrape completion status | Check Scrape Status | Download Scrape Results; Wait Before Retry | ## Async Polling Loop<br>**Poll Loop** iterates the status check. **Check Scrape Status** queries `/api/v1/scrape/status/{runId}`. **Is Scrape Complete?** routes to download on `SUCCEEDED`. Otherwise **Wait Before Retry** pauses 60 seconds and routes back into **Poll Loop** to retry. ScraperCity scrapes typically finish in 10--60 minutes. |
| Wait Before Retry | Wait | Delays between polling attempts | Is Scrape Complete? | Poll Loop | ## Async Polling Loop<br>**Poll Loop** iterates the status check. **Check Scrape Status** queries `/api/v1/scrape/status/{runId}`. **Is Scrape Complete?** routes to download on `SUCCEEDED`. Otherwise **Wait Before Retry** pauses 60 seconds and routes back into **Poll Loop** to retry. ScraperCity scrapes typically finish in 10--60 minutes. |
| Download Scrape Results | HTTP Request | Downloads completed scrape output | Is Scrape Complete? | Parse CSV Results | ## Results Processing<br>**Download Scrape Results** fetches the completed CSV. **Parse CSV Results** converts rows to JSON objects. **Remove Duplicate Contacts** deduplicates by email field. **Filter Valid Contacts** drops any rows without an email address. **Map Contact Fields** normalises field names ready for HubSpot. |
| Parse CSV Results | Code | Parses CSV or fallback result formats into contacts | Download Scrape Results | Remove Duplicate Contacts | ## Results Processing<br>**Download Scrape Results** fetches the completed CSV. **Parse CSV Results** converts rows to JSON objects. **Remove Duplicate Contacts** deduplicates by email field. **Filter Valid Contacts** drops any rows without an email address. **Map Contact Fields** normalises field names ready for HubSpot. |
| Remove Duplicate Contacts | Remove Duplicates | Deduplicates contacts by email | Parse CSV Results | Filter Valid Contacts | ## Results Processing<br>**Download Scrape Results** fetches the completed CSV. **Parse CSV Results** converts rows to JSON objects. **Remove Duplicate Contacts** deduplicates by email field. **Filter Valid Contacts** drops any rows without an email address. **Map Contact Fields** normalises field names ready for HubSpot. |
| Filter Valid Contacts | Filter | Keeps only records with email field present | Remove Duplicate Contacts | Map Contact Fields | ## Results Processing<br>**Download Scrape Results** fetches the completed CSV. **Parse CSV Results** converts rows to JSON objects. **Remove Duplicate Contacts** deduplicates by email field. **Filter Valid Contacts** drops any rows without an email address. **Map Contact Fields** normalises field names ready for HubSpot. |
| Map Contact Fields | Set | Normalizes contact fields for HubSpot | Filter Valid Contacts | Upsert Contact in HubSpot | ## Results Processing<br>**Download Scrape Results** fetches the completed CSV. **Parse CSV Results** converts rows to JSON objects. **Remove Duplicate Contacts** deduplicates by email field. **Filter Valid Contacts** drops any rows without an email address. **Map Contact Fields** normalises field names ready for HubSpot.<br>## HubSpot Output<br>**Upsert Contact in HubSpot** creates or updates a contact for each result using the email as the unique key. Address, phone, city, state, and zip are all written to standard HubSpot contact properties. Add a HubSpot App Token credential before running. |
| Upsert Contact in HubSpot | HubSpot | Creates or updates HubSpot contacts | Map Contact Fields |  | ## HubSpot Output<br>**Upsert Contact in HubSpot** creates or updates a contact for each result using the email as the unique key. Address, phone, city, state, and zip are all written to standard HubSpot contact properties. Add a HubSpot App Token credential before running. |
| Overview | Sticky Note | Documentation note |  |  |  |
| Section -- Config | Sticky Note | Documentation note |  |  |  |
| Section -- Scrape Submit | Sticky Note | Documentation note |  |  |  |
| Section -- Async Polling Loop | Sticky Note | Documentation note |  |  |  |
| Section -- Results Processing | Sticky Note | Documentation note |  |  |  |
| Section -- HubSpot Output | Sticky Note | Documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Name it: **Reverse lookup emails with ScraperCity and sync contacts to HubSpot CRM**.

2. **Add a Manual Trigger node**.
   - Node type: **Manual Trigger**
   - Keep default settings.
   - This is the workflow entry point.

3. **Add a Set node** after the trigger.
   - Rename it to **Configure Lookup Parameters**.
   - Add these fields:
     - `emails` as **String**
       - Example value: `user@example.com,user@example.com`
     - `maxResults` as **Number**
       - Example value: `5`
   - Connect:
     - `When clicking 'Execute workflow'` → `Configure Lookup Parameters`

4. **Create the ScraperCity credential** before configuring request nodes.
   - Credential type: **HTTP Header Auth**
   - Suggested name: **ScraperCity API Key**
   - Header name: `Authorization`
   - Header value: `Bearer YOUR_KEY`

5. **Add an HTTP Request node** to submit the scrape.
   - Rename it to **Start People Finder Scrape**
   - Method: `POST`
   - URL: `https://app.scrapercity.com/api/v1/scrape/people-finder`
   - Authentication: **Generic Credential Type**
   - Generic Auth Type: **HTTP Header Auth**
   - Select credential: **ScraperCity API Key**
   - Enable body sending
   - Body content type: **JSON**
   - Use this JSON body expression:
     ```json
     {
       "name": [],
       "email": {{ JSON.stringify($json.emails.split(',').map(e => e.trim())) }},
       "phone_number": [],
       "street_citystatezip": [],
       "max_results": {{ $json.maxResults }}
     }
     ```
   - Connect:
     - `Configure Lookup Parameters` → `Start People Finder Scrape`

6. **Add a Set node** to store the run identifier.
   - Rename it to **Store Run ID**
   - Add field:
     - `runId` as **String**
     - Value: `={{ $json.runId }}`
   - Connect:
     - `Start People Finder Scrape` → `Store Run ID`

7. **Add a Wait node** for the first polling delay.
   - Rename it to **Wait Before First Poll**
   - Set wait amount to `30`
   - In the UI, confirm the intended unit is seconds
   - Connect:
     - `Store Run ID` → `Wait Before First Poll`

8. **Add a Split In Batches node** to control looping.
   - Rename it to **Poll Loop**
   - Batch size: `1`
   - Options:
     - `reset = false`
   - Connect:
     - `Wait Before First Poll` → `Poll Loop`

9. **Add an HTTP Request node** to check scrape status.
   - Rename it to **Check Scrape Status**
   - Method: `GET`
   - URL expression:
     ```text
     =https://app.scrapercity.com/api/v1/scrape/status/{{ $('Store Run ID').item.json.runId }}
     ```
   - Authentication: **Generic Credential Type**
   - Generic Auth Type: **HTTP Header Auth**
   - Select credential: **ScraperCity API Key**
   - Connect:
     - `Poll Loop` → `Check Scrape Status`

10. **Add an IF node** to detect completion.
    - Rename it to **Is Scrape Complete?**
    - Condition:
      - Left value: `={{ $json.status }}`
      - Operator: **equals**
      - Right value: `SUCCEEDED`
    - Use strict/case-sensitive matching if available
    - Connect:
      - `Check Scrape Status` → `Is Scrape Complete?`

11. **Add a Wait node** for retries.
    - Rename it to **Wait Before Retry**
    - Set wait amount to `60`
    - Confirm intended unit is seconds
    - Connect:
      - False output of `Is Scrape Complete?` → `Wait Before Retry`

12. **Close the polling loop**.
    - Connect:
      - `Wait Before Retry` → `Poll Loop`

13. **Add an HTTP Request node** to download completed results.
    - Rename it to **Download Scrape Results**
    - Method: `GET`
    - URL expression:
      ```text
      =https://app.scrapercity.com/api/downloads/{{ $('Store Run ID').item.json.runId }}
      ```
    - Authentication: **Generic Credential Type**
    - Generic Auth Type: **HTTP Header Auth**
    - Select credential: **ScraperCity API Key**
    - Connect:
      - True output of `Is Scrape Complete?` → `Download Scrape Results`

14. **Add a Code node** to parse the returned dataset.
    - Rename it to **Parse CSV Results**
    - Language: JavaScript
    - Paste this logic:
      ```javascript
      // Parse CSV text returned by the download endpoint
      // Handles both a raw CSV string and a JSON body with a 'data' or 'csv' field

      const raw = $input.first().json;
      let csvText = '';

      if (typeof raw === 'string') {
        csvText = raw;
      } else if (raw.data && typeof raw.data === 'string') {
        csvText = raw.data;
      } else if (raw.csv && typeof raw.csv === 'string') {
        csvText = raw.csv;
      } else {
        // Fallback: treat entire body as an array of contact objects
        const contacts = Array.isArray(raw) ? raw : (raw.results || []);
        return contacts.map(c => ({ json: c }));
      }

      const lines = csvText.trim().split('\n').filter(l => l.trim() !== '');
      if (lines.length < 2) return [];

      const headers = lines[0].split(',').map(h => h.trim().replace(/^\"|\"$/g, ''));

      const results = [];
      for (let i = 1; i < lines.length; i++) {
        const values = lines[i].split(',').map(v => v.trim().replace(/^\"|\"$/g, ''));
        const obj = {};
        headers.forEach((h, idx) => {
          obj[h] = values[idx] || '';
        });
        results.push({ json: obj });
      }

      return results;
      ```
    - Connect:
      - `Download Scrape Results` → `Parse CSV Results`

15. **Add a Remove Duplicates node**.
    - Rename it to **Remove Duplicate Contacts**
    - Compare field:
      - `email`
    - Connect:
      - `Parse CSV Results` → `Remove Duplicate Contacts`

16. **Add a Filter node**.
    - Rename it to **Filter Valid Contacts**
    - Add condition:
      - Field/expression: `={{ $json.email }}`
      - Operator: **exists**
    - Connect:
      - `Remove Duplicate Contacts` → `Filter Valid Contacts`

17. **Add a Set node** to normalize fields.
    - Rename it to **Map Contact Fields**
    - Add these assignments:
      - `email` → `={{ ($json.email || '').trim().toLowerCase() }}`
      - `firstName` → `={{ $json.first_name || $json.firstName || '' }}`
      - `lastName` → `={{ $json.last_name || $json.lastName || '' }}`
      - `phone` → `={{ $json.phone || $json.phone_number || '' }}`
      - `address` → `={{ $json.address || $json.street || '' }}`
      - `city` → `={{ $json.city || '' }}`
      - `state` → `={{ $json.state || '' }}`
      - `zip` → `={{ $json.zip || $json.zipcode || '' }}`
    - Connect:
      - `Filter Valid Contacts` → `Map Contact Fields`

18. **Create the HubSpot credential**.
    - Credential type: **HubSpot App Token** / private app token
    - Suggested name: **HubSpot App Token**
    - Paste your HubSpot private app token
    - Ensure the private app has permission to read/write contacts

19. **Add a HubSpot node** to create or update contacts.
    - Rename it to **Upsert Contact in HubSpot**
    - Authentication: **App Token**
    - Choose credential: **HubSpot App Token**
    - Set the node to upsert/create-update a contact using email
    - Email field:
      - `={{ $json.email }}`
    - Map additional fields:
      - `firstName` → `={{ $json.firstName }}`
      - `lastName` → `={{ $json.lastName }}`
      - `phone` → `={{ $json.phone }}`
      - `city` → `={{ $json.city }}`
      - `state` → `={{ $json.state }}`
      - `zip` → `={{ $json.zip }}`
      - `message` →  
        `=Address: {{ $json.address }}, {{ $json.city }}, {{ $json.state }} {{ $json.zip }}`
        
        `Source: ScraperCity People Finder reverse lookup`
    - Connect:
      - `Map Contact Fields` → `Upsert Contact in HubSpot`

20. **Optionally add documentation sticky notes** matching the original layout.
    - Include:
      - Overview of the workflow
      - Configuration instructions
      - Scrape submission notes
      - Async polling explanation
      - Results processing notes
      - HubSpot output notes

21. **Test the workflow**.
    - Update `Configure Lookup Parameters` with real comma-separated emails
    - Execute manually
    - Confirm:
      - ScraperCity returns a `runId`
      - Polling continues until `SUCCEEDED`
      - Results are downloaded and parsed correctly
      - HubSpot contacts are created or updated

22. **Recommended hardening improvements** if rebuilding for production.
    - Add a branch for `FAILED` or `CANCELLED` scrape statuses
    - Add a maximum polling attempt counter
    - Validate emails before submitting to ScraperCity
    - Use a more robust CSV parser if fields may contain commas or quoted text
    - Normalize email before deduplication, not only before HubSpot upsert

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and is not itself documented as a callable sub-workflow. It has a single manual entry point.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## How it works: 1. Configure Lookup Parameters -- set the email addresses you want to reverse-lookup and the max results per search. 2. Start People Finder Scrape -- submits the emails to ScraperCity and receives an async job ID. 3. Polling loop -- checks job status every 60 seconds until the scrape status is SUCCEEDED. 4. Download and parse -- fetches the CSV result, parses rows into contact records, deduplicates by email, and filters out rows missing an email. 5. Upsert to HubSpot -- maps name, phone, and address fields and creates or updates each contact in HubSpot CRM. | Workflow-level note |
| Setup steps: 1. Add a ScraperCity API Key credential: HTTP Header Auth, header `Authorization`, value `Bearer YOUR_KEY`. 2. Add a HubSpot App Token credential using your HubSpot private app token. 3. In Configure Lookup Parameters, replace the example emails with your target list (comma-separated). 4. Click Execute workflow to run. | Workflow-level note |
| Configuration: Configure Lookup Parameters is the only node you must edit. Enter target emails as a comma-separated string. `max_results` controls how many address records ScraperCity returns per email. All credentials are managed through n8n credential store -- no API keys in node parameters. | Configuration note |
| Scrape Submission: Start People Finder Scrape posts the email list to ScraperCity's People Finder endpoint. Store Run ID captures the returned `runId` so the polling loop can reference it. Wait Before First Poll pauses 30 seconds before the first status check. | Submission note |
| Async Polling Loop: Poll Loop iterates the status check. Check Scrape Status queries `/api/v1/scrape/status/{runId}`. Is Scrape Complete? routes to download on `SUCCEEDED`. Otherwise Wait Before Retry pauses 60 seconds and routes back into Poll Loop to retry. ScraperCity scrapes typically finish in 10--60 minutes. | Polling note |
| Results Processing: Download Scrape Results fetches the completed CSV. Parse CSV Results converts rows to JSON objects. Remove Duplicate Contacts deduplicates by email field. Filter Valid Contacts drops any rows without an email address. Map Contact Fields normalises field names ready for HubSpot. | Processing note |
| HubSpot Output: Upsert Contact in HubSpot creates or updates a contact for each result using the email as the unique key. Address, phone, city, state, and zip are all written to standard HubSpot contact properties. Add a HubSpot App Token credential before running. | Output note |