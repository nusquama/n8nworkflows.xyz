Send LinkedIn outreach messages using Google Sheets and SourceGeek

https://n8nworkflows.xyz/workflows/send-linkedin-outreach-messages-using-google-sheets-and-sourcegeek-13029


# Send LinkedIn outreach messages using Google Sheets and SourceGeek

## 1. Workflow Overview

**Workflow name:** LeadGen SourceGeek - Jan 26 - Outreach  
**Stated title:** Send LinkedIn outreach messages using Google Sheets and SourceGeek

**Purpose:**  
Send a first LinkedIn outreach message to multiple leads whose data is stored in a Google Sheet, using the SourceGeek integration. After each message is sent, the workflow writes a timestamp back to the same Google Sheet row, then waits briefly to throttle messages before processing the next lead.

**Primary use cases:**
- Batch outreach campaigns driven by a spreadsheet (simple CRM-like control).
- Logging/traceability (timestamping “1st message sent”).
- Rate-limiting outbound actions to avoid sending too quickly.

### 1.1 Manual Start (Execution entry)
Starts the workflow manually from the n8n UI.

### 1.2 Lead Selection from Google Sheets
Reads only rows that are eligible for first-message outreach based on sheet filters.

### 1.3 Batch Loop + Send LinkedIn Message
Iterates row-by-row and sends the “First message” to the “LinkedIn profile URL” via SourceGeek.

### 1.4 Timestamp + Sheet Update + Throttling
Generates a human-readable timestamp, updates the “1st message sent” column in Google Sheets, waits 1 minute, and continues the loop.

---

## 2. Block-by-Block Analysis

### Block 1 — Manual Start
**Overview:** Initiates the workflow execution manually for controlled outreach runs.  
**Nodes involved:**  
- When clicking ‘Execute workflow’

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point.
- **Configuration choices:** No parameters; it simply emits a single empty item to start.
- **Inputs:** None (trigger).
- **Outputs:** Connects to **Get row(s) in sheet**.
- **Edge cases / failures:** None typical; only n8n runtime availability.

---

### Block 2 — Filter eligible leads in Google Sheets
**Overview:** Pulls rows from a Google Sheet that match outreach eligibility (start outreach = “y”, and not already messaged / not stopped).  
**Nodes involved:**
- Get row(s) in sheet

#### Node: Get row(s) in sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — read rows by filters.
- **Key configuration (interpreted):**
  - **Operation:** “Get row(s)” (read).
  - **Spreadsheet:** “Lead Get SourceGeek - Jan 26” (documentId `1Trxz...`).
  - **Sheet/tab:** “Data” (`gid=0`).
  - **Filters (filtersUI):**
    - `Start outreach` must equal **"y"**
    - `1st message sent` is blank (no lookupValue provided → intended “empty”)
    - `Stop outreach` is blank (no lookupValue provided → intended “empty”)
- **Credentials:** Google Sheets OAuth2 (“Google Sheets account”).
- **Inputs:** From **Manual Trigger**.
- **Outputs:** To **Loop Over Items**.
- **Edge cases / failures:**
  - OAuth token expiry/permission issues (403/401).
  - Filter behavior ambiguity: blank filter entries may behave differently depending on node version and n8n’s Google Sheets implementation. If it does not correctly filter empty values, you may retrieve unintended rows.
  - If required columns are misspelled/renamed in the sheet, results may be empty or mapping may break later.
- **Version-specific notes:** Node typeVersion `4.7`. Filtering UI and schema behaviors can differ from earlier versions.

---

### Block 3 — Batch processing and sending the first message
**Overview:** Iterates through each eligible row and sends a LinkedIn message via SourceGeek using fields from the sheet.  
**Nodes involved:**
- Loop Over Items
- Send message to LinkedIn profile

#### Node: Loop Over Items
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — controls iteration.
- **Configuration choices:** Uses defaults (batch size not explicitly set).
  - In n8n, the default batch size is typically **1**, meaning one lead per cycle (depends on UI defaults).
- **Inputs:** From **Get row(s) in sheet** and later from **Wait** (to continue loop).
- **Outputs:**
  - **Main output 0:** (not used here)
  - **Main output 1:** Goes to **Send message to LinkedIn profile** (this workflow uses the “continue” output path)
- **Edge cases / failures:**
  - If the node is miswired (using the wrong output), the loop can stall or never process items.
  - If there are 0 rows, the loop will effectively do nothing (expected).
  - If batch size is >1 (manually changed), downstream nodes must handle multiple items properly.

#### Node: Send message to LinkedIn profile
- **Type / role:** SourceGeek node (`n8n-nodes-sourcegeek.sourcegeek`) — sends a LinkedIn DM/outreach message.
- **Operation:** `sendMessage`
- **Key fields (expressions):**
  - **linkedinUrl:** `={{ $json["LinkedIn profile URL"] }}`
  - **message:** `={{ $json["First message"] }}`
- **Credentials:** “Sourcegeek Credentials account”.
- **Inputs:** From **Loop Over Items** (one lead item expected).
- **Outputs:** To **JS code to generate timestamp**.
- **Edge cases / failures:**
  - Missing/invalid LinkedIn URL → API rejection or send failure.
  - Empty “First message” → may send blank content (or error depending on SourceGeek rules).
  - SourceGeek auth issues, rate limits, or LinkedIn restrictions.
  - Transient network/API errors; consider adding retry/error handling if scaling.

---

### Block 4 — Timestamp generation, sheet update, and throttling
**Overview:** After sending, creates a formatted timestamp string, writes it back to the source row (matched by LinkedIn URL), waits 1 minute, then continues the loop.  
**Nodes involved:**
- JS code to generate timestamp
- Update selected row in sheet with timestamp
- Wait 1 minute before the next message is send

#### Node: JS code to generate timestamp
- **Type / role:** Code node (`n8n-nodes-base.code`) — generates timestamp fields.
- **Logic summary:**
  - Builds:
    - `date_today` = Date object
    - `day_today` = weekday short name (Sun–Sat)
    - `current` = string like `Mon 26 Jan 2026 14:5:33` (note: minutes/seconds are not zero-padded)
  - Writes into `items[0].json.*` and returns items.
- **Inputs:** From **Send message to LinkedIn profile** (likely includes SourceGeek response merged with the original item depending on n8n behavior; at minimum it will carry JSON).
- **Outputs:** To **Update selected row in sheet with timestamp**.
- **Edge cases / failures:**
  - If multiple items arrive, the code only writes to `items[0]` (others won’t get timestamp fields). This is fine if batch size = 1, but fragile otherwise.
  - Time formatting is not normalized (e.g., `14:5:3`); if you need consistent sorting, you should zero-pad.
  - Timezone is the n8n server timezone.

#### Node: Update selected row in sheet with timestamp
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — update row(s) by matching column.
- **Operation:** `update`
- **Matching logic:**
  - **Matching columns:** `LinkedIn profile URL`
  - It updates the row whose `LinkedIn profile URL` equals the provided value.
- **Columns written (mappingMode = defineBelow):**
  - `1st message sent` = `={{ $json.current }}`
  - `LinkedIn profile URL` = `={{ $('Loop Over Items').item.json["LinkedIn profile URL"] }}`
    - This explicitly pulls the LinkedIn URL from the current loop item (important if the current `$json` has changed due to SourceGeek response).
- **Spreadsheet/sheet:** Same document and “Data” tab as read step.
- **Credentials:** Google Sheets OAuth2.
- **Inputs:** From **JS code to generate timestamp**.
- **Outputs:** To **Wait 1 minute before the next message is send**.
- **Edge cases / failures:**
  - If LinkedIn profile URLs are not unique in the sheet, multiple rows could be updated (or an arbitrary match, depending on implementation).
  - If the update requires row identifiers and matching fails, nothing gets updated (silent failure risk depending on node behavior).
  - Permissions/auth errors.
  - If `current` is missing (e.g., code node failed), expression will resolve to empty → bad logging.

#### Node: Wait 1 minute before the next message is send
- **Type / role:** Wait node (`n8n-nodes-base.wait`) — throttling/rate-limiting.
- **Configuration:** Wait **1 minute**.
- **Inputs:** From **Update selected row in sheet with timestamp**.
- **Outputs:** Back to **Loop Over Items** to request the next batch item.
- **Edge cases / failures:**
  - Wait nodes pause executions; large lead lists can create long-running executions and may hit n8n execution retention or concurrency limits.
  - If the instance restarts during a wait, resumption depends on n8n configuration and persistence.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Get row(s) in sheet | ## Sending out 1st message |
| Get row(s) in sheet | Google Sheets | Read eligible lead rows from sheet | When clicking ‘Execute workflow’ | Loop Over Items | ## Sending out 1st message; Get the required data from a Google Sheet. Needed is the LinkedIn Profile url and the 1st message |
| Loop Over Items | Split In Batches | Iterate through leads one-by-one | Get row(s) in sheet; Wait 1 minute before the next message is send | Send message to LinkedIn profile | ## Sending out 1st message |
| Send message to LinkedIn profile | SourceGeek | Send LinkedIn outreach message | Loop Over Items | JS code to generate timestamp | ## Sending out 1st message; Send out the message to the specific LinkedIn profile |
| JS code to generate timestamp | Code | Generate formatted timestamp fields | Send message to LinkedIn profile | Update selected row in sheet with timestamp | ## Sending out 1st message; Get the current date in the right format. Update the initial sheet so this action is tagged |
| Update selected row in sheet with timestamp | Google Sheets | Write “1st message sent” timestamp back to the sheet | JS code to generate timestamp | Wait 1 minute before the next message is send | ## Sending out 1st message; Get the current date in the right format. Update the initial sheet so this action is tagged |
| Wait 1 minute before the next message is send | Wait | Throttle between messages and continue loop | Update selected row in sheet with timestamp | Loop Over Items | ## Sending out 1st message; Wait a short moment before sending a new message |
| Sticky Note | Sticky Note | Comment block label | — | — | ## Sending out 1st message |
| Sticky Note1 | Sticky Note | Comment | — | — | Get the required data from a Google Sheet. Needed is the LinkedIn Profile url and the 1st message |
| Sticky Note2 | Sticky Note | Comment | — | — | Send out the message to the specific LinkedIn profile |
| Sticky Note3 | Sticky Note | Comment | — | — | Get the current date in the right format. Update the initial sheet so this action is tagged |
| Sticky Note5 | Sticky Note | Comment | — | — | Wait a short moment before sending a new message |
| Sticky Note6 | Sticky Note | Comment / explanation | — | — | ## How it works\nThis workflow can be used to send out a message to a specific LinkedIn profile. The data for the profile and the content of the message are coming from a Google Sheet.\n\nFor sending the message from your LinkedIn profile to another LinkedIn user, the SourceGeek node is being used.\n\nAfter the message is successfully sent, the initial row in the Google Sheet will be updated with a timestamp. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: *LeadGen SourceGeek - Jan 26 - Outreach* (or your preferred name).

2. **Add trigger**
   - Add node: **Manual Trigger**
   - Name: *When clicking ‘Execute workflow’*

3. **Add Google Sheets read node**
   - Add node: **Google Sheets**
   - Name: *Get row(s) in sheet*
   - **Credentials:** Create/choose **Google Sheets OAuth2** credential.
   - **Document:** Select your spreadsheet (e.g., “Lead Get SourceGeek - Jan 26”).
   - **Sheet:** Select tab “Data”.
   - **Operation:** Get row(s) (read).
   - **Filters:**
     - `Start outreach` equals `y`
     - `1st message sent` empty
     - `Stop outreach` empty
   - Connect: **Manual Trigger → Get row(s) in sheet**

4. **Add looping node**
   - Add node: **Split In Batches**
   - Name: *Loop Over Items*
   - Keep defaults (recommended batch size = 1 for this workflow’s code node behavior).
   - Connect: **Get row(s) in sheet → Loop Over Items**

5. **Add SourceGeek send-message node**
   - Add node: **SourceGeek**
   - Name: *Send message to LinkedIn profile*
   - **Credentials:** Create/choose **SourceGeek credentials**.
   - **Operation:** Send Message
   - **LinkedIn URL field:** set expression to `{{$json["LinkedIn profile URL"]}}`
   - **Message field:** set expression to `{{$json["First message"]}}`
   - Connect: **Loop Over Items (output 1 / continue output) → Send message to LinkedIn profile**

6. **Add timestamp Code node**
   - Add node: **Code**
   - Name: *JS code to generate timestamp*
   - Paste logic (equivalent behavior):
     - Create weekday/month arrays
     - Build `current` string
     - Set `items[0].json.current = ...`
     - Return items
   - Connect: **Send message to LinkedIn profile → JS code to generate timestamp**

7. **Add Google Sheets update node**
   - Add node: **Google Sheets**
   - Name: *Update selected row in sheet with timestamp*
   - **Credentials:** Same Google Sheets OAuth2 credential.
   - **Document/Sheet:** Same spreadsheet and “Data” tab.
   - **Operation:** Update
   - **Match by column:** `LinkedIn profile URL`
   - **Values to set:**
     - `1st message sent` = `{{$json.current}}`
     - `LinkedIn profile URL` = `{{$('Loop Over Items').item.json["LinkedIn profile URL"]}}`
       - (This ensures the match value is the original lead URL from the loop item.)
   - Connect: **JS code to generate timestamp → Update selected row in sheet with timestamp**

8. **Add Wait node**
   - Add node: **Wait**
   - Name: *Wait 1 minute before the next message is send*
   - **Unit:** minutes
   - **Amount:** 1
   - Connect: **Update selected row in sheet with timestamp → Wait**
   - Connect: **Wait → Loop Over Items** (to fetch the next batch item)

9. **(Optional but recommended) Add error handling**
   - Add an error branch from the SourceGeek node to log failures and/or write a “failed” status back to the sheet.
   - Consider adding retries or a shorter/longer wait depending on account limits.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Disclaimer provided by the requester |
| This workflow sends LinkedIn messages using SourceGeek and logs a timestamp back into Google Sheets. | Sticky note “How it works” content |
| Google Sheet used in the configuration: “Lead Get SourceGeek - Jan 26”, tab “Data”. | Embedded in node configuration (document + sheet selection) |

