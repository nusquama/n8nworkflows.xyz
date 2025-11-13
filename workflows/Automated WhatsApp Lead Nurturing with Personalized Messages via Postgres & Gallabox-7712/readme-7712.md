Automated WhatsApp Lead Nurturing with Personalized Messages via Postgres & Gallabox

https://n8nworkflows.xyz/workflows/automated-whatsapp-lead-nurturing-with-personalized-messages-via-postgres---gallabox-7712


# Automated WhatsApp Lead Nurturing with Personalized Messages via Postgres & Gallabox

### 1. Workflow Overview

This workflow automates WhatsApp lead nurturing by sending personalized messages to unqualified leads stored in a PostgreSQL database. It targets marketing-qualified leads (MQLs) based on a defined cadence, incrementing message counts and scheduling follow-ups depending on elapsed time since last contact. The workflow integrates with Gallabox’s WhatsApp API for message dispatch and logs all message activities back into PostgreSQL for tracking.

Logical blocks:

- **1.1 Scheduled Lead Query**: Periodically retrieves unqualified leads from PostgreSQL that need nurturing messages based on count and time intervals.

- **1.2 Lead Processing Loop**: Iterates over each qualifying lead to generate personalized message content.

- **1.3 Message Personalization**: Uses a code node to select message variants depending on lead attributes (model and count).

- **1.4 WhatsApp Message Dispatch**: Sends the personalized message through Gallabox’s WhatsApp API.

- **1.5 Logging and State Update**: Records message sending results and increments lead contact count in PostgreSQL.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Lead Query

- **Overview:**  
  This block triggers the workflow at regular intervals and queries the database to fetch unqualified leads whose messaging cadence matches specified time intervals and count conditions.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Execute a SQL query  
  - Sticky Note1 (comment)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts workflow execution every second (interval set to 1 second).  
    - Configuration: Interval field set to trigger every second.  
    - Input: None  
    - Output: Triggers downstream SQL query node.  
    - Edge Cases: High frequency might cause DB overload; consider rate limiting.

  - **Execute a SQL query**  
    - Type: PostgreSQL node  
    - Role: Runs a SQL SELECT query to fetch leads from the "MQL.mql_contacts" table.  
    - Configuration: Complex WHERE clause filters leads where count is 0 OR count = 1 and more than 3 minutes since last webhook_time, etc., ensuring messages follow a spaced schedule. Only leads with `disposition='unqualified'` are selected.  
    - Input: Triggered by Schedule Trigger  
    - Output: List of lead records matching criteria.  
    - Retry on fail: Enabled to handle transient DB issues.  
    - On error: Continue workflow output to avoid blocking entire workflow.  
    - Edge Cases: Time zone handling critical (Asia/Kolkata); query performance depends on indexes.

  - **Sticky Note1**  
    - Comment: "Gets unqualified leads which count 0,1,2,3 respective time"  
    - No functional impact.

---

#### 1.2 Lead Processing Loop

- **Overview:**  
  Splits the lead records into individual items for sequential processing, feeding each lead into the personalization logic.

- **Nodes Involved:**  
  - Loop Over Items4 (SplitInBatches)  
  - Sticky Note (comment)

- **Node Details:**

  - **Loop Over Items4**  
    - Type: SplitInBatches  
    - Role: Processes lead items one at a time or in batches (default batch size).  
    - Input: Lead records from Execute a SQL query  
    - Output: Each lead passed individually to message personalization (Code1).  
    - Edge Cases: Large lead lists may impact performance; batch size configuration affects throughput.

  - **Sticky Note**  
    - Content: "Matrixed code to add unique message for pushing that to message with variable"  
    - Context: Describes the following code node’s purpose.

---

#### 1.3 Message Personalization

- **Overview:**  
  Generates a personalized message snippet based on lead’s `model` and message `count` using a predefined mapping matrix.

- **Nodes Involved:**  
  - Code1  

- **Node Details:**

  - **Code1**  
    - Type: Code (JavaScript)  
    - Role: Applies matrix logic to select a message variant emoji and text based on lead attributes.  
    - Configuration:  
      - A nested object `matrix` maps models (`nexus`, `magnus`, `reo`, `general`) and count (0-3) to string messages with emojis.  
      - Inputs: `model` and `count` from lead data (`$input.first().json`).  
      - Outputs: JSON containing `model`, `count`, and selected `output` message.  
    - Input: Single lead from Loop Over Items4  
    - Output: JSON with personalized message snippet.  
    - Edge Cases: Missing model or count keys fallback to default message `"🛵 default"`. Expression failures if input data missing or malformed.

---

#### 1.4 WhatsApp Message Dispatch

- **Overview:**  
  Sends the personalized message to the lead’s WhatsApp number via Gallabox API with templated message and quick reply buttons.

- **Nodes Involved:**  
  - new_lead_4 (HTTP Request)  
  - Sticky Note2 (comment)

- **Node Details:**

  - **new_lead_4**  
    - Type: HTTP Request  
    - Role: Sends POST request to Gallabox WhatsApp message API endpoint.  
    - Configuration:  
      - URL: `https://server.gallabox.com/devapi/messages/whatsapp`  
      - Method: POST  
      - Headers: Includes API Key and Secret (to be added by user).  
      - Body: JSON template message with dynamic fields: recipient name, phone (prefixed with country code `91`), and message content from Code1’s output.  
      - Template includes quick reply buttons: "Show Brochure", "Get Showroom Location", "Not Interested".  
      - Authentication: Generic HTTP custom auth (credentials required).  
      - On error: Continue workflow output to avoid stopping entire processing.  
    - Input: Personalized message JSON from Code1 and lead details from SQL.  
    - Output: Response from Gallabox API.  
    - Edge Cases: Authentication failures if API keys missing/invalid; API downtime; malformed JSON causing 400 errors; phone number format issues.

  - **Sticky Note2**  
    - Content: "Gallabox/whatsapp sending api"  
    - Contextual reminder about API integration.

---

#### 1.5 Logging and State Update

- **Overview:**  
  Logs message sending results into a PostgreSQL logs table and increments the lead’s message count and updates last message sent time in the main leads table.

- **Nodes Involved:**  
  - Insert rows in a table4 (PostgreSQL Insert)  
  - Update rows in a table4 (PostgreSQL Update)  
  - Sticky Note3 (comment)

- **Node Details:**

  - **Insert rows in a table4**  
    - Type: PostgreSQL Insert node  
    - Role: Inserts a new record into "MQL.mql_logs" capturing message metadata and lead details.  
    - Configuration:  
      - Columns: name, phone, remarks ("Workflow 1"), last_sent, mes_count, message_id (from API response), disposition, Gallabox API status code and message.  
      - Matching by ID disabled (always insert new).  
    - Input: Gallabox API response and lead data.  
    - Output: Insert confirmation.  
    - Edge Cases: DB connection errors, data type mismatches.

  - **Update rows in a table4**  
    - Type: PostgreSQL Update node  
    - Role: Increments the `count` field for the lead by 1, updates `last_message_sent` timestamp, keyed by phone number.  
    - Configuration:  
      - Updates `count` to previous count + 1  
      - Sets `last_message_sent` to current or last sent timestamp  
      - Matches row by phone number.  
    - Input: Lead record and insert confirmation.  
    - Output: Update confirmation.  
    - Edge Cases: Concurrency issues if multiple updates on same lead; missing phone number; DB errors.

  - **Sticky Note3**  
    - Content: "Postgres connection to store data"  
    - Indicates DB operations for persistence.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                          | Input Node(s)             | Output Node(s)         | Sticky Note                                      |
|-------------------------|--------------------|----------------------------------------|---------------------------|------------------------|-------------------------------------------------|
| Schedule Trigger        | Schedule Trigger   | Triggers workflow every second          | None                      | Execute a SQL query    |                                                 |
| Execute a SQL query     | PostgreSQL         | Queries unqualified leads with count/time filters  | Schedule Trigger          | Loop Over Items4       | Gets unqualified leads which count 0,1,2,3 respective time |
| Loop Over Items4        | SplitInBatches     | Processes leads one by one               | Execute a SQL query        | Code1 (via second output) | Matrixed code to add unique message for pushing that to message with variable |
| Code1                   | Code (JS)          | Selects personalized message snippet based on model/count | Loop Over Items4          | new_lead_4             |                                                 |
| new_lead_4              | HTTP Request       | Sends WhatsApp message via Gallabox API | Code1                     | Insert rows in a table4 | Gallabox/whatsapp sending api                    |
| Insert rows in a table4 | PostgreSQL Insert  | Logs message sending status and details | new_lead_4                | Update rows in a table4 | Postgres connection to store data                |
| Update rows in a table4 | PostgreSQL Update  | Updates lead’s message count and last sent time | Insert rows in a table4   | Loop Over Items4       | Postgres connection to store data                |
| Sticky Note1            | Sticky Note        | Informational comment                    | None                      | None                   | Gets unqualified leads which count 0,1,2,3 respective time |
| Sticky Note             | Sticky Note        | Informational comment                    | None                      | None                   | Matrixed code to add unique message for pushing that to message with variable |
| Sticky Note2            | Sticky Note        | Informational comment                    | None                      | None                   | Gallabox/whatsapp sending api                    |
| Sticky Note3            | Sticky Note        | Informational comment                    | None                      | None                   | Postgres connection to store data                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Parameters: Set interval to trigger every 1 second (field "seconds", value 1)  
   - No credentials needed.

2. **Add PostgreSQL node ("Execute a SQL query")**  
   - Operation: Execute Query  
   - Credentials: Provide PostgreSQL credentials connected to the MQL database.  
   - Query:  
     ```sql
     SELECT * FROM "MQL".mql_contacts
     WHERE 
       (
         (count = 0)
         OR 
         (count = 1 AND (NOW() AT TIME ZONE 'Asia/Kolkata') - webhook_time > INTERVAL '3 minutes')
         OR 
         (count = 2 AND (NOW() AT TIME ZONE 'Asia/Kolkata') - webhook_time > INTERVAL '5 minutes')
         OR 
         (count = 3 AND (NOW() AT TIME ZONE 'Asia/Kolkata') - webhook_time > INTERVAL '8 minutes')
       )
       AND
       disposition = 'unqualified'
     ;
     ```  
   - Connect Schedule Trigger → Execute a SQL query.

3. **Add SplitInBatches node ("Loop Over Items4")**  
   - Default batch size (usually 1 or as preferred).  
   - Connect Execute a SQL query → Loop Over Items4.

4. **Add Code node ("Code1")**  
   - Use JavaScript code to define the message matrix and select output based on lead model and count:  
     ```javascript
     const matrix = {
       nexus: {
         0: "🛵 sample1",
         1: "⚡️ sample1",
         2: "🔋 sample1",
         3: "✨ sample1"
       },
       magnus: {
         0: "🛵 sample2",
         1: "⚡️ sample2",
         2: "🔋 sample2",
         3: "✨ sample2"
       },
       reo: {
         0: "🛵 sample3",
         1: "⚡️ sample3",
         2: "🔋 sample3",
         3: "✨sample3"
       },
       general: {
         0: "🛵 sample4",
         1: "⚡️sample4",
         2: "🔋 sample4",
         3: "✨ sample3"
       }
     };

     const model = $input.first().json.model;
     const count = $input.first().json.count;

     const output = matrix[model]?.[count] || "🛵 default";

     return [{ json: { model, count, output } }];
     ```  
   - Connect Loop Over Items4 (second output) → Code1.

5. **Add HTTP Request node ("new_lead_4")**  
   - URL: `https://server.gallabox.com/devapi/messages/whatsapp`  
   - Method: POST  
   - Authentication: Generic HTTP Custom Auth, set API Key and Secret credentials in node credentials settings.  
   - Headers: JSON with API Key and Secret.  
   - Body (JSON):  
     ```json
     {
       "channelId": "0",
       "channelType": "whatsapp",
       "recipient": {
         "name": "{{ $('Execute a SQL query').item.json.name }}",
         "phone": "91{{ $('Execute a SQL query').item.json.phone }}"
       },
       "whatsapp": {
         "type": "template",
         "template": {
           "templateName": "waba_qual_21july25",
           "bodyValues": {
             "Name": "{{ $('Execute a SQL query').item.json.name }}",
             "Details": "{{ $json.output }}"
           },
           "buttonValues": [
             {
               "index": 0,
               "sub_type": "quick_reply",
               "parameters": {
                 "type": "payload",
                 "payload": "Show Brochure"
               }
             },
             {
               "index": 1,
               "sub_type": "quick_reply",
               "parameters": {
                 "type": "payload",
                 "payload": "Get Showroom Location"
               }
             },
             {
               "index": 2,
               "sub_type": "quick_reply",
               "parameters": {
                 "type": "payload",
                 "payload": "Not Interested"
               }
             }
           ]
         }
       }
     }
     ```  
   - Connect Code1 → new_lead_4.

6. **Add PostgreSQL Insert node ("Insert rows in a table4")**  
   - Credentials: Same PostgreSQL connection to MQL database.  
   - Operation: Insert  
   - Schema: "MQL"  
   - Table: "mql_logs"  
   - Columns to insert:  
     - name: `={{ $('Execute a SQL query').item.json.name }}`  
     - phone: `={{ $('Execute a SQL query').item.json.phone }}`  
     - remarks: `"Workflow 1"`  
     - last_sent: `={{ $('Execute a SQL query').item.json.last_message_sent }}`  
     - mes_count: `={{ $('Execute a SQL query').item.json.count }}`  
     - message_id: `={{ $json.body.id }}` (from HTTP response)  
     - disposition: `={{ $('Execute a SQL query').item.json.disposition }}`  
     - gb_status_code: `={{ $json.statusCode }}`  
     - gb_status_message: `={{ $json.body.status }}`  
   - Connect new_lead_4 → Insert rows in a table4.

7. **Add PostgreSQL Update node ("Update rows in a table4")**  
   - Credentials: Same PostgreSQL connection to MQL database.  
   - Operation: Update  
   - Schema: "MQL"  
   - Table: "mql_contacts"  
   - Matching column: phone  
   - Columns to update:  
     - count: `={{ $('Execute a SQL query').item.json.count + 1 }}`  
     - last_message_sent: `={{ $json.last_sent || new Date().toISOString() }}`  
     - pincode: 0 (default)  
   - Connect Insert rows in a table4 → Update rows in a table4.

8. **Connect Update rows in a table4 → Loop Over Items4** (main output)  
   - This loops back to process the next lead batch.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                |
|--------------------------------------------------------------------------------------------------|-------------------------------|
| Use Asia/Kolkata timezone consistently in SQL time comparisons to align with local lead times.   | SQL Query node timezone usage |
| Gallabox WhatsApp API requires API Key and Secret configured in HTTP Request node credentials.   | Gallabox API documentation    |
| Message templates must be pre-approved in WhatsApp Business API with name "waba_qual_21july25".  | WhatsApp Business API docs    |
| Quick reply buttons payloads ("Show Brochure", etc.) must correspond to configured chatbot flows. | Gallabox WhatsApp templates   |
| Frequent schedule triggers (1 second) may cause DB load; adjust interval based on actual needs.  | Performance considerations    |

---

**Disclaimer:** The provided content is solely derived from an n8n workflow. It adheres strictly to applicable content policies and contains no illegal or protected data. All handled data is legal and publicly permissible.