Get LinkedIn profile data via TexAU API

https://n8nworkflows.xyz/workflows/get-linkedin-profile-data-via-texau-api-13669


# Get LinkedIn profile data via TexAU API

# LinkedIn Profile Scraper: TexAU API Reference

This document provides a technical breakdown of the n8n workflow designed to extract structured data from LinkedIn profiles using the TexAU API.

---

### 1. Workflow Overview

The workflow automates the interaction with TexAU's LinkedIn scraping automation. It follows a request-wait-retrieve pattern to handle the asynchronous nature of web scraping jobs. This workflow is ideal for enrichment pipelines where a LinkedIn URL is provided, and structured professional data (experience, skills, education) is required as output.

**Logical Blocks:**
- **1.1 Input Reception:** Receives the target LinkedIn URL via a sub-workflow trigger.
- **1.2 Automation Trigger:** Initiates the "LinkedIn Profile Scraper" task on TexAU.
- **1.3 Execution Monitoring:** Polls the execution status and waits for completion.
- **1.4 Data Retrieval:** Fetches the final scraped JSON data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
*   **Overview:** This block acts as the entry point, allowing this workflow to be called by other parent workflows.
*   **Nodes Involved:** `When Executed by Another Workflow`.
*   **Node Details:**
    *   **Type:** Execute Workflow Trigger.
    *   **Configuration:** Defines a required input variable: `Linkedin_url`.
    *   **Input/Output:** Receives a JSON object containing the URL; outputs the same to the next node.

#### 2.2 Automation Trigger
*   **Overview:** Sends a POST request to TexAU to start the scraping job.
*   **Nodes Involved:** `LinkedIn_Profile_Scrape`.
*   **Node Details:**
    *   **Type:** HTTP Request (v4.2).
    *   **Technical Role:** API Caller.
    *   **Configuration:** 
        *   **Method:** POST
        *   **URL:** `https://api.texau.com/api/v1/public/run`
        *   **Headers:** Includes `Authorization` (Bearer token), `X-TexAu-Context` (Org/Workspace IDs), and `accept`.
        *   **Body:** JSON containing `automationId` ("63f48ee97022e05c116fc798"), `connectedAccountId`, and the input `liProfileUrl`.
    *   **Potential Failures:** Invalid API key (401), invalid LinkedIn URL, or expired `connectedAccountId`.

#### 2.3 Execution Monitoring & Delay
*   **Overview:** Retrieves the job ID from the previous step, checks the execution status, and pauses the workflow to allow TexAU time to process the page.
*   **Nodes Involved:** `Get Results`, `Wait`.
*   **Node Details:**
    *   **Get Results (HTTP Request):** 
        *   Queries `https://api.texau.com/api/v1/public/executions/{{ $json.data.id }}` to confirm the job is registered.
    *   **Wait:**
        *   **Configuration:** Set to **60 seconds**.
        *   **Role:** Prevents the workflow from attempting to fetch data before the scraping engine has finished rendering the LinkedIn profile.

#### 2.4 Data Retrieval
*   **Overview:** The final step that fetches the actual structured profile content.
*   **Nodes Involved:** `Get_Results`.
*   **Node Details:**
    *   **Type:** HTTP Request.
    *   **Configuration:** 
        *   **URL:** `https://api.texau.com/api/v1/public/results/{{ $json.data.id }}`.
        *   **Headers:** Uses the same Authorization and Context headers as the initial trigger.
    *   **Output:** A JSON object containing detailed profile attributes (Headline, Summary, Positions, Education, etc.).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Executed by Another Workflow | Execute Workflow Trigger | Workflow Entry Point | None | LinkedIn_Profile_Scrape | LinkedIn Profile Retrieval via TexAU. This workflow automates the extraction... |
| LinkedIn_Profile_Scrape | HTTP Request | Trigger TexAU Automation | When Executed by Another Workflow | Get Results | LinkedIn Profile Retrieval via TexAU. This workflow automates the extraction... |
| Get Results | HTTP Request | Fetch Execution Status | LinkedIn_Profile_Scrape | Wait | LinkedIn Profile Retrieval via TexAU. This workflow automates the extraction... |
| Wait | Wait | Delay for processing | Get Results | Get_Results | LinkedIn Profile Retrieval via TexAU. This workflow automates the extraction... |
| Get_Results | HTTP Request | Retrieve Scraped Data | Wait | None | LinkedIn Profile Retrieval via TexAU. This workflow automates the extraction... |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Create a **Execute Workflow Trigger** node.
    *   Add a variable named `Linkedin_url` under "Workflow Inputs".

2.  **Initial API Call (Trigger Scraping):**
    *   Add an **HTTP Request** node named `LinkedIn_Profile_Scrape`.
    *   Set Method to `POST` and URL to `https://api.texau.com/api/v1/public/run`.
    *   In **Headers**, add:
        *   `Authorization`: `Bearer [Your_TexAU_Token]`
        *   `X-TexAu-Context`: `{"orgUserId":"your_id","workspaceId":"your_id"}`
    *   In **Body Parameters** (JSON), include the `automationId` for LinkedIn Scraper and map `liProfileUrl` to `{{ $json.Linkedin_url }}`.

3.  **Job Verification:**
    *   Add an **HTTP Request** node named `Get Results`.
    *   Set Method to `GET`.
    *   Set URL to `https://api.texau.com/api/v1/public/executions/{{ $json.data.id }}`.
    *   Reuse the same Headers as Step 2.

4.  **Temporal Delay:**
    *   Add a **Wait** node.
    *   Set the "Wait Amount" to `60` and "Wait Unit" to `Seconds`.

5.  **Final Data Fetch:**
    *   Add an **HTTP Request** node named `Get_Results`.
    *   Set Method to `GET`.
    *   Set URL to `https://api.texau.com/api/v1/public/results/{{ $json.data.id }}`.
    *   Reuse the same Headers.

6.  **Connections:**
    *   Connect nodes in this sequence: Trigger -> LinkedIn_Profile_Scrape -> Get Results -> Wait -> Get_Results.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **TexAU API Documentation** | Ensure your `automationId` and `connectedAccountId` are active in your TexAU dashboard. |
| **Rate Limiting** | LinkedIn scraping is subject to TexAU's internal proxy and account limits. |
| **Data Privacy** | Ensure compliance with GDPR/CCPA when storing or processing scraped personal data. |
| **Processing Time** | If the workflow returns empty results, increase the **Wait** node duration beyond 60 seconds. |