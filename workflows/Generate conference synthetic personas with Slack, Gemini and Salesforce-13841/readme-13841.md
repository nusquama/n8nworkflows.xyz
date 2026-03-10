Generate conference synthetic personas with Slack, Gemini and Salesforce

https://n8nworkflows.xyz/workflows/generate-conference-synthetic-personas-with-slack--gemini-and-salesforce-13841


# Generate conference synthetic personas with Slack, Gemini and Salesforce

This document provides a detailed technical breakdown of the **Digital Doppelgänger: Conference Synthetic Personas Generator** workflow. This system automates the creation of AI-generated audience archetypes based on real CRM data, allowing event organizers to simulate attendee feedback and preferences before an event occurs.

---

### 1. Workflow Overview

The workflow is designed to shift event planning from reactive measurement to predictive analysis. By analyzing existing attendee data from HubSpot, Salesforce, or Google Sheets, the system uses Google Gemini to cluster the audience into distinct synthetic personas. These personas are then "interviewed" automatically to provide insights into content preferences, networking goals, and potential pain points.

#### Logic Blocks:
1.  **Input Reception:** Captures the `/doppelganger` command from Slack and parses arguments (Event Name, Persona Count, CRM Source).
2.  **CRM Integration:** Fetches contact/attendee data from the specified CRM and normalizes it into a standard JSON schema.
3.  **AI Persona Generation:** Uses LLM clustering to generate detailed, GDPR-safe synthetic archetypes.
4.  **Storage & Delivery:** Saves personas to a database (Google Sheets) and posts rich visual cards to Slack.
5.  **Automated Interviewing:** Conducts a 14-question structured interview with each persona to generate deep behavioral insights.
6.  **Reporting:** Provides a final summary of the generation process to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Parsing
**Overview:** This block handles the initial handshake with Slack, ensuring the user receives an immediate response while the workflow processes the heavy AI tasks in the background.
*   **Nodes:** `Receive Slash Command`, `Acknowledge Slack Command`, `Parse Command Args`.
*   **Details:**
    *   **Receive Slash Command:** A Webhook node (POST) listening for the `/doppelganger` command.
    *   **Acknowledge Slack Command:** Responds with a 200 OK and a "Generating..." message to prevent Slack's 3-second timeout error.
    *   **Parse Command Args:** Uses Regex to extract the event name (handling quotes), the requested number of personas (clamped between 3-10), and the CRM source.

#### 2.2 CRM Data Fetching & Normalization
**Overview:** Retrieves raw attendee data and transforms it into a unified format for the AI engine.
*   **Nodes:** `Route by CRM Source`, `Fetch HubSpot Contacts`, `Fetch Salesforce Contacts`, `Fetch Google Sheets Contacts`, `Normalize Audience Data`.
*   **Details:**
    *   **Route by CRM Source:** A Switch node directing traffic based on the `crmSource` variable.
    *   **Fetch Nodes:** Specialized HTTP Request or Google Sheets nodes that query specific fields (Industry, Job Title, Company) based on the Event Name.
    *   **Normalize Audience Data:** A Code node that maps various CRM field names to a standard schema (name, title, company, industry, location) and calculates aggregate statistics (top industries/titles) to provide context for the LLM.

#### 2.3 AI Persona Engine
**Overview:** The "brain" of the workflow that transforms raw statistics into human-like archetypes.
*   **Nodes:** `Generate Persona Archetypes`, `Google Gemini for Personas`.
*   **Details:**
    *   **Generate Persona Archetypes:** A Chain LLM node that instructs Gemini to act as a data scientist and behavioral psychologist. It generates a JSON array of personas including motivations, pain points, and decision styles.
    *   **Google Gemini for Personas:** Configured with `gemini-2.5-flash-lite` for high-speed, cost-effective reasoning.

#### 2.4 Data Persistence & Slack Delivery
**Overview:** Formats the AI output for human consumption and long-term storage.
*   **Nodes:** `Parse Persona JSON`, `Store Persona in Google Sheets`, `Format Persona Card`, `Post Persona to Slack`, `Capture Thread Timestamp`.
*   **Details:**
    *   **Store Persona in Google Sheets:** Appends persona details to a central tracking sheet.
    *   **Format Persona Card:** Converts JSON data into Slack Block Kit markdown, including emojis and bulleted lists for readability.
    *   **Capture Thread Timestamp:** Extracts the `ts` (timestamp) from the Slack message to allow the auto-interview to be posted as a threaded reply.

#### 2.5 Structured Auto-Interview
**Overview:** Simulates a deep-dive conversation with the generated personas.
*   **Nodes:** `Run Structured Interview`, `Google Gemini for Auto-Interview`, `Format Interview Report`, `Post Interview Report to Slack`, `Post Generation Summary`.
*   **Details:**
    *   **Run Structured Interview:** Executes a second AI pass where the persona is asked 14 questions across categories like ROI, Networking, and Logistics.
    *   **Post Generation Summary:** A final Slack node providing a checklist of what was completed.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Slash Command | Webhook | Trigger | (None) | Acknowledge Slack Command, Parse Command Args | Receives /doppelganger command. |
| Acknowledge Slack Command | Respond to Webhook | API Response | Receive Slash Command | (None) | Acknowledges immediately (Slack 3s timeout). |
| Parse Command Args | Code | Data Parsing | Receive Slash Command | Route by CRM Source | Extracts Event Name, Count, and CRM source. |
| Route by CRM Source | Switch | Logic Routing | Parse Command Args | HubSpot, Salesforce, Sheets Fetchers | Routes to the correct CRM API. |
| Fetch HubSpot Contacts | HTTP Request | Data Extraction | Route by CRM Source | Normalize Audience Data | Contact search by event tag. |
| Fetch Salesforce Contacts | HTTP Request | Data Extraction | Route by CRM Source | Normalize Audience Data | SOQL query on CampaignMember. |
| Fetch Google Sheets Contacts | Google Sheets | Data Extraction | Route by CRM Source | Normalize Audience Data | Direct read (CSV import fallback). |
| Normalize Audience Data | Code | Transformation | CRM Fetchers | Generate Persona Archetypes | Normalizes sources to a common format. |
| Generate Persona Archetypes | Chain LLM | AI Processing | Normalize Audience Data | Parse Persona JSON | Clusters data into distinct archetypes. |
| Google Gemini for Personas | Gemini Chat | AI Model | (AI Provider) | Generate Persona Archetypes | Uses gemini-2.5-flash-lite. |
| Parse Persona JSON | Code | Data Extraction | Generate Persona Archetypes | Store in Sheets, Format Card | Extract JSON array from LLM response. |
| Store Persona in Sheets | Google Sheets | Persistence | Parse Persona JSON | (None) | Stored in Digital Doppelgängers sheet. |
| Format Persona Card | Code | Formatting | Parse Persona JSON | Post Persona to Slack | Build Slack Block Kit message. |
| Post Persona to Slack | Slack | Notification | Format Persona Card | Capture Thread Timestamp | Posted to requesting channel. |
| Capture Thread Timestamp | Code | Logic | Post Persona to Slack | Run Structured Interview | Capture timestamp for thread routing. |
| Run Structured Interview | Chain LLM | AI Processing | Capture Thread Timestamp | Format Interview Report | 14 questions across 5 categories. |
| Format Interview Report | Code | Formatting | Run Structured Interview | Post Interview Report | Format auto-interview response. |
| Post Interview Report | Slack | Notification | Format Interview Report | Post Generation Summary | Posted to Slack automatically. |
| Post Generation Summary | Slack | Notification | Post Interview Report | (None) | Final confirmation message. |

---

### 4. Reproducing the Workflow from Scratch

#### Prerequisites
- **Slack App:** Create an app, enable Slash Commands, and add `chat:write` and `commands` scopes.
- **Google Cloud:** API Key for Gemini.
- **CRM Access:** API Key for HubSpot, OAuth2 for Salesforce, or Google Sheets API.

#### Step-by-Step Setup
1.  **Trigger Setup:** Create a **Webhook** node. Set the path to `doppelganger` and the HTTP method to `POST`.
2.  **Immediate Response:** Connect a **Respond to Webhook** node. Set the response body to a "Processing" message.
3.  **Argument Parsing:** Add a **Code** node. Use Regex to split the Slack `text` body into `eventName`, `personaCount`, and `crmSource`.
4.  **CRM Routing:**
    - Add a **Switch** node based on the `crmSource` variable.
    - Create three paths: HubSpot (HTTP Request), Salesforce (HTTP Request), and Google Sheets.
    - **HubSpot:** Use a POST request to `/crm/v3/objects/contacts/search` filtering by `event_tag`.
    - **Salesforce:** Use a GET request to `/services/data/v59.0/query` with a SOQL query against `CampaignMember`.
5.  **Normalization:** Add a **Code** node that takes the array from any branch and outputs a single JSON object containing a `sampleAttendees` string and aggregate stats.
6.  **AI Persona Generation:**
    - Add a **Chain LLM** node.
    - Connect a **Google Gemini Chat Model** node.
    - **Prompt:** Instruct the model to "Generate [count] personas in JSON format" based on the provided attendee stats.
7.  **Data Processing Loop:**
    - Add a **Code** node to parse the JSON string from the LLM. Ensure it returns an array of items so n8n processes each persona individually.
    - Add a **Google Sheets** node (Append) to save the persona.
    - Add a **Code** node to build the Slack Block Kit card.
    - Add a **Slack** node to post the card.
8.  **Automated Interview:**
    - Add a **Code** node to capture the `ts` from the Slack output.
    - Add a **Chain LLM** node with a prompt: "You are [Persona Name]. Answer 14 specific questions about [Event Name]."
    - Add a final **Slack** node to post the interview report as a reply to the original card's thread.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Concept Origin** | VOK DAMS' Digital Doppelgänger concept for pre-event prediction. |
| **GDPR Compliance** | All personas are synthetic; no real PII is stored or processed by the LLM. |
| **Feedback & Consulting** | [Contact Milo Bravo / BRaiA Labs](https://tally.so/r/EkKGgB) |
| **Developer LinkedIn** | [Milo Bravo Profile](https://linkedin.com/in/MiloBravo/) |