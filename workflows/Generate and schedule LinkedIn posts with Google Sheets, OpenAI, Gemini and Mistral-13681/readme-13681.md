Generate and schedule LinkedIn posts with Google Sheets, OpenAI, Gemini and Mistral

https://n8nworkflows.xyz/workflows/generate-and-schedule-linkedin-posts-with-google-sheets--openai--gemini-and-mistral-13681


# Generate and schedule LinkedIn posts with Google Sheets, OpenAI, Gemini and Mistral

This technical document provides a comprehensive analysis of the **Automate LinkedIn Trending Content Creation & Scheduling** workflow. This system automates the lifecycle of social media management: from fetching scheduled topics in Google Sheets to AI-driven trend research, content generation, image creation, and final publication to LinkedIn.

---

### 1. Workflow Overview

The workflow is designed to streamline the creation of high-quality LinkedIn content by leveraging multiple AI models and external data sources. It operates on a scheduled basis, pulling "seeds" or "topics" from a Google Sheet and transforming them into professional posts optimized for current trends.

**Logical Blocks:**
*   **1.1 Data Ingestion & Looping:** Triggers the process, retrieves rows from Google Sheets, and iterates through each item.
*   **1.2 Trending Topic Research:** Uses an AI Agent equipped with search tools (Google Trends, Taplio) to refine a raw topic into a trending narrative.
*   **1.3 Content & Visual Generation:** Generates the actual LinkedIn post copy and a corresponding AI-generated image.
*   **1.4 SEO & Optimization:** A second AI Agent optimizes the generated content for searchability and engagement based on real-time SEO data.
*   **1.5 Distribution & Logging:** Publishes the final result to LinkedIn (both Page and Profile) and updates the Google Sheet status.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion & Looping
This block handles the entry point and ensures each row in the source spreadsheet is processed individually.
*   **Nodes Involved:** `Schedule Trigger`, `Get row(s) in sheet`, `Loop Over Items`, `Topic`.
*   **Node Details:**
    *   **Schedule Trigger:** Sets the frequency (e.g., daily/weekly) for the automation to run.
    *   **Get row(s) in sheet:** Connects to Google Sheets to fetch rows marked for processing.
    *   **Loop Over Items:** A Split in Batches node that handles items one by one to prevent API rate limiting.
    *   **Topic (Set Node):** Extracts the specific topic or keyword from the current spreadsheet row for use in subsequent AI steps.

#### 2.2 Trending Topic Research (AI Agent)
This block utilizes an agentic approach to research what is currently performing well regarding the input topic.
*   **Nodes Involved:** `Content topic generator`, `OpenAI Model`, `Structured Output1`, `AI Agent Tool`, `HTTP (google trends)`, `HTTP (taplio)`, `OpenAI Chat`.
*   **Node Details:**
    *   **Content topic generator (AI Agent):** The "brain" that decides which tools to use to research the topic.
    *   **AI Agent Tool:** A wrapper for the HTTP tools allowing the agent to perform web requests.
    *   **HTTP Tools:** Interface with Google Trends and Taplio APIs to pull engagement metrics and trending keywords.
    *   **Structured Output1:** Ensures the AI returns a consistent JSON schema (e.g., specific headline, researched keywords).

#### 2.3 Content & Visual Generation
Once researched, the content is drafted and an image is generated to accompany the post.
*   **Nodes Involved:** `Content creator`, `OpenAI Model1`, `Structured Output`, `OpenAI (Creates images for post)`.
*   **Node Details:**
    *   **Content creator (ChainLLM):** Takes the researched data and writes a LinkedIn-style post.
    *   **OpenAI (Creates images for post):** Uses DALL-E (via OpenAI node) to generate an image based on the post's theme.
    *   **Error Handling:** Failures in image generation do not stop the text generation but may result in a post without media.

#### 2.4 SEO & Optimization
This block refines the draft to maximize LinkedIn algorithm reach.
*   **Nodes Involved:** `SEO`, `HTTP (google trends)1`, `OpenAI Model1`.
*   **Node Details:**
    *   **SEO (AI Agent):** Analyzes the draft against latest SEO trends.
    *   **HTTP (google trends)1:** Provides a secondary check on keyword volume to ensure the post uses high-traffic terminology.

#### 2.5 Distribution & Logging
The final phase executes the posting and records the success back to the spreadsheet.
*   **Nodes Involved:** `Merge`, `LinkedIn page`, `LinkedIn profile`, `Update Status row in sheet`.
*   **Node Details:**
    *   **Merge:** Synchronizes the data from the SEO agent and the image generation node.
    *   **LinkedIn Nodes:** Use OAuth2 credentials to post the text and image to the user's personal profile and/or company page.
    *   **Update Status row in sheet:** Changes the status in the original Google Sheet to "Published" or "Completed" to prevent duplicate posts.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule | Workflow Entry | (None) | Get row(s) in sheet | |
| Get row(s) in sheet | Google Sheets | Data Retrieval | Schedule Trigger | Loop Over Items | |
| Loop Over Items | Split in Batches | Flow Control | Get row(s) in sheet | Topic | |
| Topic | Set | Variable Prep | Loop Over Items | Content topic generator | |
| Content topic generator | AI Agent | Trend Research | Topic | Content creator | |
| AI Agent Tool | Agent Tool | Tool Wrapper | OpenAI Chat, HTTP nodes | Content topic generator | |
| Content creator | Chain LLM | Post Writing | Content topic generator | OpenAI (Images), SEO | |
| OpenAI (Creates images) | OpenAI | Media Generation | Content creator | Merge | |
| SEO | AI Agent | Content Optimization | Content creator | Merge | |
| Merge | Merge | Data Sync | OpenAI (Images), SEO | LinkedIn nodes, Sheet update | |
| LinkedIn page | LinkedIn | Distribution | Merge | (None) | |
| LinkedIn profile | LinkedIn | Distribution | Merge | (None) | |
| Update Status row | Google Sheets | Logging | Merge | (None) | |
| OpenAI Model / Model1 | OpenAI Chat | AI Inference | (Various) | AI Agents / Chains | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Create a Google Sheet with headers: `Topic`, `Status`, `Post Link`.
    *   Obtain API keys for: OpenAI, LinkedIn (Developer App), and optionally Taplio/Google Trends (via SerpApi or similar).

2.  **Trigger & Data Setup:**
    *   Add a **Schedule Trigger** (set to your desired frequency).
    *   Add a **Google Sheets Node** (`Get row(s)`) filtered by `Status = "Pending"`.
    *   Add a **Split in Batches** node to process one row at a time.
    *   Add a **Set Node** (`Topic`) to isolate the "Topic" column value.

3.  **Research Agent Setup:**
    *   Add an **AI Agent Node** (`Content topic generator`).
    *   Connect an **OpenAI Chat Model** (GPT-4o recommended).
    *   Attach a **Structured Output Parser** to define the research format.
    *   Connect an **AI Agent Tool** node linked to **HTTP Request** nodes that fetch data from Google Trends and Taplio.

4.  **Generation Setup:**
    *   Add a **Basic LLM Chain** (`Content creator`) to draft the post using the output from the research agent.
    *   Connect an **OpenAI Node** (`Image Generation`) using the content draft as the prompt for DALL-E 3.

5.  **Optimization Setup:**
    *   Add another **AI Agent** (`SEO`) to review the content. Use an **HTTP Tool** to double-check trending hashtags or keywords.

6.  **Distribution Setup:**
    *   Add a **Merge Node** (Mode: Wait for all inputs) to combine the SEO text and the generated Image URL/Binary.
    *   Connect **LinkedIn Nodes** for "Create a Post". Configure one for the "Person" and one for the "Organization" if needed.
    *   Connect a final **Google Sheets Node** (`Update row`) using the Row ID from the start of the loop to set the status to "Live".

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **LinkedIn API Permissions** | Ensure your LinkedIn App has `w_member_social` and `w_organization_social` scopes. |
| **Model Selection** | Using GPT-4o for Agents is recommended for better tool-calling reliability. |
| **Rate Limiting** | The Split in Batches node is crucial to avoid hitting LinkedIn's daily posting limits or OpenAI's TPM limits. |