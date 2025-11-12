Copyright Infringement Detector with ScrapeGraphAI and Automated Legal Response

https://n8nworkflows.xyz/workflows/copyright-infringement-detector-with-scrapegraphai-and-automated-legal-response-6642


# Copyright Infringement Detector with ScrapeGraphAI and Automated Legal Response

### 1. Workflow Overview

This workflow automates the detection of copyright infringements on the web by leveraging ScrapeGraphAI for external content search and a series of analytical and decision-making nodes to assess infringement risk and trigger legal and monitoring alerts. It is designed for legal and brand protection teams to proactively identify unauthorized use of copyrighted material, brand names, and media, and respond with appropriate legal measures or monitoring.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Web Search Trigger:** Periodically initiates the workflow every 24 hours.
- **1.2 Copyright Infringement Search (ScrapeGraphAI):** Performs an AI-driven web search for potential copyright violations based on customized search queries.
- **1.3 Content Analysis and Similarity Comparison:** Processes search results to evaluate similarity against copyrighted content and brand assets, scoring risk and determining if action is required.
- **1.4 Legal Action Determination:** Applies predefined rules to assign risk levels and legal responses, generating detailed case reports.
- **1.5 Legal Action Routing:** Routes cases based on urgency to appropriate alert channels.
- **1.6 Alerting and Notification:** Sends Telegram notifications to legal and monitoring teams with contextual case details.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Web Search Trigger

- **Overview:**  
  This block initiates the workflow execution at regular intervals to ensure continuous monitoring.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - *Type:* Schedule Trigger node  
    - *Role:* Starts the workflow on a timed basis  
    - *Configuration:* Set to trigger once every 24 hours (interval in hours = 24)  
    - *Input:* None (start node)  
    - *Output:* Triggers the ScrapeGraphAI Web Search node  
    - *Edge Cases:* Missed triggers due to downtime; misconfiguration of interval could result in too frequent or infrequent runs  
    - *Version-specific:* Uses typeVersion 1.2, compatible with standard scheduling features  

#### 2.2 Copyright Infringement Search (ScrapeGraphAI)

- **Overview:**  
  Executes an AI-enhanced web search to locate potential copyright infringements by searching for exact text, paraphrases, brand name misuse, or image theft. Returns results in a structured JSON schema.

- **Nodes Involved:**  
  - ScrapeGraphAI Web Search  
  - Sticky Note - Search (documentation)

- **Node Details:**  
  - **ScrapeGraphAI Web Search**  
    - *Type:* ScrapeGraphAI node  
    - *Role:* Searches the web using AI to find matches against copyrighted content  
    - *Configuration:*  
      - `userPrompt` includes instructions to detect exact matches, paraphrases, brand misuse, stolen media, with a structured JSON schema for results  
      - `websiteUrl` is a Google search URL with quoted phrases and brand names for targeted queries  
    - *Input:* Triggered by Schedule Trigger  
    - *Output:* Sends raw results to Content Comparer node  
    - *Edge Cases:* API failures, rate limits, incomplete or noisy search results, inaccurate matches if queries are too broad or narrow  
    - *Version-specific:* typeVersion 1  
  - **Sticky Note - Search**  
    - Provides detailed guidance on search purpose, configuration, and expected outputs, aiding maintainers in understanding and customizing the search criteria.

#### 2.3 Content Analysis and Similarity Comparison

- **Overview:**  
  Analyzes the search results to quantify similarity between found content and copyrighted material, evaluates brand misuse, competitor domains, and assigns risk levels.

- **Nodes Involved:**  
  - Content Comparer (Code node)  
  - Sticky Note - Comparer (documentation)

- **Node Details:**  
  - **Content Comparer**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Processes search results, compares content snippets against protected texts and brand names, calculates similarity scores, assesses risk, and flags items needing action  
    - *Configuration Highlights:*  
      - Defines copyrighted text snippets, brand names, and own domains to exclude  
      - Uses a custom similarity function based on word intersection ratio  
      - Flags high similarity (>0.8) as high risk requiring action; medium similarity (>0.5) as medium risk  
      - Detects unauthorized brand usage and competitor domains to increase risk level  
      - Filters out own domain content  
      - Outputs an array of flagged cases with detailed analysis and alert-ready messages  
    - *Input:* Receives JSON from ScrapeGraphAI node  
    - *Output:* Sends filtered, analyzed infringement cases to Infringement Detector node  
    - *Edge Cases:* Missing or malformed input data, text similarity function limitations (e.g., language nuances), false positives/negatives, empty outputs if no matches  
    - *Version-specific:* typeVersion 2  
  - **Sticky Note - Comparer**  
    - Explains the purpose, key features, and customization options for the content comparer logic.

#### 2.4 Legal Action Determination

- **Overview:**  
  Determines required legal responses based on infringement severity, generates a structured legal case report including next steps and timelines.

- **Nodes Involved:**  
  - Infringement Detector (Code node)  
  - Sticky Note - Detector (documentation)

- **Node Details:**  
  - **Infringement Detector**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Applies legal rules to assign actions such as cease & desist, DMCA takedown, legal consultation, or monitoring only  
    - *Configuration Highlights:*  
      - Defines conditions triggering immediate action (e.g., high similarity, unauthorized brand use, competitor domain)  
      - Assigns priority levels (high, medium, low)  
      - Builds comprehensive legal report object with case ID, infringement details, recommended actions, next steps, timelines  
      - Logs progress and outputs final report JSON with alert flags  
    - *Input:* Receives analyzed infringement cases from Content Comparer  
    - *Output:* Routes legal report to Legal Action Trigger node  
    - *Edge Cases:* Logic errors in rules, missing input data, date formatting issues, generation of duplicate case IDs if triggered simultaneously  
    - *Version-specific:* typeVersion 2  
  - **Sticky Note - Detector**  
    - Documents legal action logic, risk levels, and expected outputs to assist in maintenance and extension.

#### 2.5 Legal Action Routing

- **Overview:**  
  Routes cases based on whether immediate legal action is required, directing to urgent alerts or monitoring notifications accordingly.

- **Nodes Involved:**  
  - Legal Action Trigger (If node)  
  - Sticky Note - Trigger (documentation)

- **Node Details:**  
  - **Legal Action Trigger**  
    - *Type:* If node  
    - *Role:* Checks if the boolean field `requires_immediate_attention` is true to decide routing path  
    - *Configuration:* Condition: `={{ $json.requires_immediate_attention }} == true`  
    - *Input:* Legal report JSON from Infringement Detector  
    - *Output:*  
      - True branch → Brand Protection Alert  
      - False branch → Monitoring Alert  
    - *Edge Cases:* Missing or malformed `requires_immediate_attention` field may cause routing errors; logical errors in condition expression  
    - *Version-specific:* typeVersion 2  
  - **Sticky Note - Trigger**  
    - Explains routing logic and integration possibilities such as email alerts, ticket creation, and evidence tasks.

#### 2.6 Alerting and Notification

- **Overview:**  
  Sends Telegram notifications to designated channels to alert legal teams or monitoring staff about infringement cases and recommended actions.

- **Nodes Involved:**  
  - Brand Protection Alert (Telegram node)  
  - Monitoring Alert (Telegram node)  
  - Sticky Note - Alerts (documentation)

- **Node Details:**  
  - **Brand Protection Alert**  
    - *Type:* Telegram node  
    - *Role:* Sends urgent alerts for high-priority copyright infringement cases to a Telegram channel  
    - *Configuration:*  
      - Chat ID: `@copyright_alerts`  
      - Message text formatted in Markdown with case details, risk level, next steps, and a placeholder link for evidence report  
      - `parse_mode` set to Markdown for rich formatting  
    - *Input:* Cases flagged for immediate legal attention from Legal Action Trigger (true branch)  
    - *Output:* None (terminal node)  
    - *Edge Cases:* Telegram API failures, invalid chat ID, message formatting errors  
    - *Version-specific:* typeVersion 1.2  
  - **Monitoring Alert**  
    - *Type:* Telegram node  
    - *Role:* Sends alerts for medium-priority cases to a monitoring Telegram channel  
    - *Configuration:*  
      - Chat ID: `@copyright_monitoring`  
      - Markdown formatted message summarizing monitoring cases with assessment and actions  
    - *Input:* Cases not requiring immediate action (false branch from Legal Action Trigger)  
    - *Output:* None (terminal node)  
    - *Edge Cases:* Same as Brand Protection Alert  
    - *Version-specific:* typeVersion 1.2  
  - **Sticky Note - Alerts**  
    - Describes alert types, channels, and integration options with other communication systems and legal management tools.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                             | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                     |
|--------------------------|---------------------|---------------------------------------------|---------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger    | Initiates workflow every 24 hours            | None                      | ScrapeGraphAI Web Search    |                                                                                                |
| ScrapeGraphAI Web Search | ScrapeGraphAI       | Performs AI-powered web search for infringements | Schedule Trigger          | Content Comparer            | # Step 2: ScrapeGraphAI Web Search 🔍 Searches for matches, paraphrases, brand misuse, media theft |
| Content Comparer         | Code                | Analyzes similarity and infringement risk     | ScrapeGraphAI Web Search  | Infringement Detector       | # Step 3: Content Comparer 🔍 Calculates similarity scores, identifies brand violations, risk levels |
| Infringement Detector    | Code                | Determines legal action and generates reports | Content Comparer          | Legal Action Trigger        | # Step 4: Infringement Detector ⚖️ Legal action decisions, case ID, next steps, timelines        |
| Legal Action Trigger     | If                  | Routes cases based on immediate attention need | Infringement Detector     | Brand Protection Alert, Monitoring Alert | # Step 5: Legal Action Trigger 🎯 Routes cases for urgent or monitoring responses               |
| Brand Protection Alert   | Telegram            | Sends urgent Telegram alerts for high-priority cases | Legal Action Trigger (true) | None                       | # Step 6: Brand Protection Alerts 🚨 Urgent alerts, legal team notifications, integration options |
| Monitoring Alert         | Telegram            | Sends Telegram notifications for monitoring cases | Legal Action Trigger (false) | None                       | # Step 6: Brand Protection Alerts 🚨 Urgent alerts, legal team notifications, integration options |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node**:
   - Name: `Schedule Trigger`
   - Set interval to trigger every 24 hours (hoursInterval = 24)
   - Connect its main output to the next node.

3. **Add a ScrapeGraphAI node**:
   - Name: `ScrapeGraphAI Web Search`
   - Set `websiteUrl` to a Google search URL with queries for your copyrighted content and brand names, e.g.:  
     `"https://www.google.com/search?q=\"your+copyrighted+content+here\"+OR+\"brand+name\"+OR+\"unique+phrase\""`
   - Set `userPrompt` with instructions to detect exact matches, paraphrases, brand usage, image theft, and return results in JSON schema format as specified.
   - Connect Schedule Trigger output to this node input.

4. **Add a Code node** for content comparison:
   - Name: `Content Comparer`
   - Paste the provided JavaScript code that:
     - Extracts potential infringements from ScrapeGraphAI results
     - Defines your copyrighted texts, brand names, and domains
     - Calculates similarity scores using word intersections
     - Flags cases with risk levels and requires_action boolean
     - Returns an array of flagged infringement objects
   - Connect ScrapeGraphAI output to this node.

5. **Add another Code node** for legal determination:
   - Name: `Infringement Detector`
   - Paste the provided JavaScript code that:
     - Receives analyzed infringement data
     - Defines legal action rules for immediate or monitoring responses
     - Generates a detailed legal report including case ID, infringement details, recommended actions, next steps, timelines
     - Returns the report with alert flags
   - Connect Content Comparer output here.

6. **Add an If node** for legal action routing:
   - Name: `Legal Action Trigger`
   - Condition: Check if `{{$json.requires_immediate_attention}}` equals `true`
   - Connect Infringement Detector output to this node.

7. **Add a Telegram node** for urgent alerts:
   - Name: `Brand Protection Alert`
   - Set Chat ID to your legal alert Telegram channel, e.g., `@copyright_alerts`
   - Configure message text using Markdown with case details, risk level, next steps, timeline, and a placeholder for evidence link
   - Connect the true branch of Legal Action Trigger to this node.

8. **Add a Telegram node** for monitoring alerts:
   - Name: `Monitoring Alert`
   - Set Chat ID to your monitoring Telegram channel, e.g., `@copyright_monitoring`
   - Configure message text summarizing monitoring cases in Markdown
   - Connect the false branch of Legal Action Trigger to this node.

9. **Set up credentials**:
   - For ScrapeGraphAI, configure the required API credentials as per your ScrapeGraphAI account.
   - For Telegram nodes, set up Telegram Bot API credentials with appropriate bot token and channel permissions.

10. **Add sticky notes** (optional but recommended) to document each step with the content from the Sticky Notes nodes for clarity and maintainability.

11. **Save and activate** the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                              | Context or Link                                                    |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow is tagged under Revenue Optimization, Content Strategy, Trend Monitoring, Dynamic Pricing, and Simple RAG, indicating possible use cases beyond copyright detection.                                                                        | Workflow metadata tags                                            |
| Integration possibilities include email alerts, Slack, SMS, legal management system ticket creation, and evidence collection task generation for unified case management.                                                                                   | Sticky Note - Trigger                                              |
| Telegram channels used here are placeholders; ensure your bot has permission to post in these channels and update chat IDs accordingly.                                                                                                                   | Telegram node configuration                                       |
| The similarity function is basic and may require enhancement for advanced linguistic analysis or multilingual support.                                                                                                                                     | Content Comparer node notes                                       |
| Legal action thresholds and next steps are customizable based on organizational policies and jurisdictional requirements.                                                                                                                                   | Infringement Detector node notes                                  |
| This workflow can be extended by adding sub-workflows for automated cease & desist letter generation, DMCA filing, or integration with legal CRM systems.                                                                                                | Suggested extensions                                              |
| Example search queries and prompt templates can be refined to improve AI search accuracy and reduce false positives.                                                                                                                                         | Sticky Note - Search                                              |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to existing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.