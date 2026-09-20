Send RSS blog newsletters via email with Google Sheets tracking

https://n8nworkflows.xyz/workflows/send-rss-blog-newsletters-via-email-with-google-sheets-tracking-17747


# Send RSS blog newsletters via email with Google Sheets tracking

### 1. Workflow Overview

This workflow automates the process of publishing website RSS updates to newsletter subscribers. It executes on a daily schedule, queries a designated RSS feed, cross-references incoming items with an existing log of sent items in Google Sheets, isolates truly new content, gathers subscriber emails, dispatches individual emails for each unmailed article, and finally updates the historical log to avoid duplicates in future executions.

The logic is segmented into five functional blocks:
- **1.1 Intake & History Retrieval:** Triggers daily and fetches raw posts alongside previously dispatched records.
- **1.2 Differential Filtering:** Evaluates feed items against historical logs to filter out already-processed entries.
- **1.3 Subscriber Preparation:** Loads the target recipient database and structures processing loops.
- **1.4 Email Dispatch:** Iterates through new articles and subscriber lists to deliver newsletters.
- **1.5 History Logging:** Appends newly distributed posts to the tracking sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Intake & History Retrieval
- **Overview:** Initiates the automation on a fixed schedule, pulls raw publication items from an external website RSS feed, and extracts the tracking history of previously sent emails from Google Sheets.
- **Nodes Involved:**
  - `When Every Morning at 9am`
  - `Fetch Site RSS Feed`
  - `Read Sent Articles from Sheets`

- **Node Details:**
  - **When Every Morning at 9am**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Triggers execution).
    - *Configuration:* Cron expression set to `0 9 * * *`.
    - *Expressions/Variables:* None.
    - *Connections:* Input: None | Output: `Fetch Site RSS Feed`.
    - *Edge Cases:* Server timezone configuration mismatches.
  - **Fetch Site RSS Feed**
    - *Type & Role:* `n8n-nodes-base.rssFeedRead` (Retrieves website feed).
    - *Configuration:* Targets `https://YOUR-WEBSITE.com/feed/`.
    - *Expressions/Variables:* None.
    - *Connections:* Input: `When Every Morning at 9am` | Output: `Read Sent Articles from Sheets`.
    - *Edge Cases:* Feed down, invalid XML structure, or HTTP timeout.
  - **Read Sent Articles from Sheets**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Reads historical tracking records).
    - *Configuration:* Operation: `Read`, Sheet Name: `Sent`, Document ID: `YOUR_GOOGLE_SHEET_ID`.
    - *Expressions/Variables:* None.
    - *Connections:* Input: `Fetch Site RSS Feed` | Output: `Merge Articles to Find New`.
    - *Edge Cases:* Expired OAuth tokens, missing sheet names, API rate limits.

---

#### 1.2 Differential Filtering
- **Overview:** Compares live RSS items against historical Google Sheets entries using unique identifiers, determining if new material is available for distribution.
- **Nodes Involved:**
  - `Merge Articles to Find New`
  - `If New Articles Exist`

- **Node Details:**
  - **Merge Articles to Find New**
    - *Type & Role:* `n8n-nodes-base.merge` (Cross-references datasets).
    - *Configuration:* Mode: `Combine`, Join Mode: `keepNonMatches`, Match fields: `link` (Input 1) against `article_url` (Input 2).
    - *Expressions/Variables:* None.
    - *Connections:* Input: `Read Sent Articles from Sheets` | Output: `If New Articles Exist`.
    - *Edge Cases:* Field name mismatches between feeds and sheets causing false duplicates.
  - **If New Articles Exist**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional gate).
    - *Configuration:* Condition: Evaluates if item array length is greater than `0`.
    - *Expressions/Variables:* `={{ $input.all().length }}`.
    - *Connections:* Input: `Merge Articles to Find New` | Output (true): `Read Subscribers from Sheets`.
    - *Edge Cases:* Empty datasets returning zero items, halting the branch.

---

#### 1.3 Subscriber Preparation
- **Overview:** Pulls subscriber email addresses from Google Sheets and organizes articles into processing loops and contact structures.
- **Nodes Involved:**
  - `Read Subscribers from Sheets`
  - `Loop Over New Articles`
  - `Compile Contact List`

- **Node Details:**
  - **Read Subscribers from Sheets**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Fetches contact data).
    - *Configuration:* Operation: `Read`, Sheet Name: `Contacts`, Document ID: `YOUR_GOOGLE_SHEET_ID`.
    - *Expressions/Variables:* None.
    - *Connections:* Input: `If New Articles Exist` | Output: `Loop Over New Articles`, `Compile Contact List`.
    - *Edge Cases:* Blank contact rows or improperly formatted email columns.
  - **Loop Over New Articles**
    - *Type & Role:* `n8n-nodes-base.splitInBatches` (Iterative batch controller).
    - *Configuration:* Standard batch looping setup.
    - *Expressions/Variables:* None.
    - *Connections:* Input: `Read Subscribers from Sheets` | Output: `Send Newsletter Email`.
    - *Edge Cases:* Infinite loop risks if batch pointers fail to advance.
  - **Compile Contact List**
    - *Type & Role:* `n8n-nodes-base.itemLists` (Data aggregation).
    - *Configuration:* Operation: `Aggregate`.
    - *Expressions/Variables:* None.
    - *Connections:* Input: `Read Subscribers from Sheets` | Output: None (Reference node for grouping).
    - *Edge Cases:* Large contact lists exceeding memory limits during aggregation.

---

#### 1.4 Email Dispatch
- **Overview:** Formulates and transmits customized newsletter emails for each unmailed article to targeted recipients.
- **Nodes Involved:**
  - `Send Newsletter Email`

- **Node Details:**
  - **Send Newsletter Email**
    - *Type & Role:* `n8n-nodes-base.emailSend` (SMTP communication).
    - *Configuration:* From: `YOUR@EMAIL.COM`, To: evaluated dynamically.
    - *Expressions/Variables:*
      - Subject: `=New article: {{ $node["Loop Over New Articles"].json.title }}`
      - To Email: `={{ $json.email }}`
    - *Connections:* Input: `Loop Over New Articles` | Output: `Append Sent Articles to Sheets`.
    - *Edge Cases:* SMTP authentication failures, blacklisting, rate limits, or invalid email strings.

---

#### 1.5 History Logging
- **Overview:** Writes records of successfully sent articles back to Google Sheets to ensure they are excluded on future executions.
- **Nodes Involved:**
  - `Append Sent Articles to Sheets`

- **Node Details:**
  - **Append Sent Articles to Sheets**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Appends rows).
    - *Configuration:* Operation: `Append`, Sheet Name: `Sent`, Document ID: `YOUR_GOOGLE_SHEET_ID`.
    - *Expressions/Variables:*
      - Title: `={{ $node["Loop Over New Articles"].json.title }}`
      - Sent Date: `={{ $now.toISO() }}`
      - Article URL: `={{ $node["Loop Over New Articles"].json.link }}`
    - *Connections:* Input: `Send Newsletter Email` | Output: None.
    - *Edge Cases:* API quotas, connection drops mid-append resulting in partial tracking states.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Workflow documentation & setup guide. | None | None | ## RSS-to-Email Newsletter Automation<br><br>### How it works<br><br>This workflow runs every morning to read new posts from an RSS feed and compare them against a Google Sheets log of articles that were already sent. If new articles exist, it loads the subscriber list, prepares recipients, sends newsletter emails for each new article, and records sent articles back to Google Sheets.<br><br>### Setup steps<br><br>- Configure the schedule trigger for the desired newsletter send time and timezone.<br>- Set the RSS feed URL in the RSS Feed Read node.<br>- Connect Google Sheets credentials and select the spreadsheets/ranges for the already-sent article log and subscriber list.<br>- Configure the email sending credentials, sender address, subject/body template, and recipient mapping.<br>- Verify that the sent-article tracking sheet stores a unique article identifier such as URL or GUID so duplicates can be filtered reliably.<br><br>### Customization<br><br>Adjust the RSS source, send frequency, filtering rules, email template, and subscriber sheet columns to match the newsletter format and audience. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Intake documentation block. | None | None | ## Fetch feed and history<br><br>Starts the daily workflow, reads the site RSS feed, and retrieves the Google Sheets record of articles that have already been emailed. These nodes form the left-side intake cluster. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Filtering documentation block. | None | None | ## Filter new articles<br><br>Compares feed items with the sent-article history and checks whether any unsent articles remain before continuing. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Subscriber preparation documentation block. | None | None | ## Prepare subscribers<br><br>Reads the subscriber list and branches into nearby preparation steps for article iteration and contact aggregation. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Email dispatch documentation block. | None | None | ## Send newsletter email<br><br>Sends the email for each new article using the prepared article and subscriber data. |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | History logging documentation block. | None | None | ## Record sent article<br><br>Updates Google Sheets after a successful email send so the article is not included in future newsletter runs. |
| `When Every Morning at 9am` | `n8n-nodes-base.scheduleTrigger` | Triggers execution daily. | None | `Fetch Site RSS Feed` | |
| `Fetch Site RSS Feed` | `n8n-nodes-base.rssFeedRead` | Fetches posts from the RSS feed. | `When Every Morning at 9am` | `Read Sent Articles from Sheets` | |
| `Read Sent Articles from Sheets` | `n8n-nodes-base.googleSheets` | Loads sent history from Google Sheets. | `Fetch Site RSS Feed` | `Merge Articles to Find New` | |
| `Merge Articles to Find New` | `n8n-nodes-base.merge` | Filters out previously sent articles. | `Read Sent Articles from Sheets` | `If New Articles Exist` | |
| `If New Articles Exist` | `n8n-nodes-base.if` | Validates presence of unsent items. | `Merge Articles to Find New` | `Read Subscribers from Sheets` | |
| `Read Subscribers from Sheets` | `n8n-nodes-base.googleSheets` | Reads subscriber contacts. | `If New Articles Exist` | `Loop Over New Articles`, `Compile Contact List` | |
| `Loop Over New Articles` | `n8n-nodes-base.splitInBatches` | Iterates through unsent articles. | `Read Subscribers from Sheets` | `Send Newsletter Email` | |
| `Compile Contact List` | `n8n-nodes-base.itemLists` | Aggregates contact details. | `Read Subscribers from Sheets` | None | |
| `Send Newsletter Email` | `n8n-nodes-base.emailSend` | Sends newsletter via SMTP. | `Loop Over New Articles` | `Append Sent Articles to Sheets` | |
| `Append Sent Articles to Sheets` | `n8n-nodes-base.googleSheets` | Logs sent status back to Google Sheets. | `Send Newsletter Email` | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger:**
   - Add a **Schedule Trigger** node named `When Every Morning at 9am`.
   - Set interval type to `Cron Expression` and value to `0 9 * * *`.
2. **Configure RSS Feed Ingestion:**
   - Add an **RSS Feed Read** node named `Fetch Site RSS Feed`.
   - Set URL parameter to your target website RSS feed (e.g., `https://YOUR-WEBSITE.com/feed/`).
   - Connect `When Every Morning at 9am` to `Fetch Site RSS Feed`.
3. **Configure Historical Log Reading:**
   - Add a **Google Sheets** node named `Read Sent Articles from Sheets`.
   - Set operation to `Read`, provide your Google Sheets credential, specify Document ID, and set Sheet Name to `Sent`.
   - Connect `Fetch Site RSS Feed` to `Read Sent Articles from Sheets`.
4. **Implement Differential Filtering:**
   - Add a **Merge** node named `Merge Articles to Find New`.
   - Configure Mode: `Combine`, Join Mode: `keepNonMatches`. Match fields: `link` (Input 1) against `article_url` (Input 2).
   - Connect `Read Sent Articles from Sheets` to `Merge Articles to Find New`.
   - Add an **If** node named `If New Articles Exist`.
   - Set condition: Left value `={{ $input.all().length }}`, operator `Larger Than`, Right value `0`.
   - Connect `Merge Articles to Find New` to `If New Articles Exist`.
5. **Configure Subscriber Loading and Looping:**
   - Add a **Google Sheets** node named `Read Subscribers from Sheets`.
   - Set operation to `Read`, select your credential, specify Document ID, and set Sheet Name to `Contacts`.
   - Connect the `true` output of `If New Articles Exist` to `Read Subscribers from Sheets`.
   - Add a **Split In Batches** node named `Loop Over New Articles`.
   - Connect `Read Subscribers from Sheets` to `Loop Over New Articles`.
   - Add an **Item Lists** node named `Compile Contact List`.
   - Connect `Read Subscribers from Sheets` to `Compile Contact List` with operation set to `Aggregate`.
6. **Configure Email Dispatch:**
   - Add a **Send Email** node named `Send Newsletter Email`.
   - Configure SMTP credentials, set `fromEmail` to your sender address, `toEmail` to `={{ $json.email }}`, and subject to `=New article: {{ $node["Loop Over New Articles"].json.title }}`.
   - Connect `Loop Over New Articles` to `Send Newsletter Email`.
7. **Configure History Logging:**
   - Add a **Google Sheets** node named `Append Sent Articles to Sheets`.
   - Set operation to `Append`, select your credential, specify Document ID, and set Sheet Name to `Sent`.
   - Map columns via expression:
     - `title`: `={{ $node["Loop Over New Articles"].json.title }}`
     - `sent_date`: `={{ $now.toISO() }}`
     - `article_url`: `={{ $node["Loop Over New Articles"].json.link }}`
   - Connect `Send Newsletter Email` to `Append Sent Articles to Sheets`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| RSS-to-Email Newsletter Automation Template | Built via n8n integration patterns for automated content distribution. |