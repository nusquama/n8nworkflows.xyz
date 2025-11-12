Create Ideal Customer Profiles from LinkedIn Data with Airtop and Claude AI

https://n8nworkflows.xyz/workflows/create-ideal-customer-profiles-from-linkedin-data-with-airtop-and-claude-ai-4204


# Create Ideal Customer Profiles from LinkedIn Data with Airtop and Claude AI

### 1. Workflow Overview

This workflow automates the creation of Ideal Customer Profiles (ICPs) by analyzing LinkedIn profile URLs of current customers through data enrichment and AI-powered synthesis. It targets marketing and sales teams aiming to better understand their best customers, generate detailed ICPs, and create scoring methodologies for future prospect evaluation.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Waits for incoming chat messages containing LinkedIn profile URLs.
- **1.2 AI Processing:** Parses and organizes URLs, manages memory, enriches data via Airtop, and uses Claude AI to analyze and synthesize the ICP.
- **1.3 Data Enrichment:** Calls the Airtop tool to extract structured LinkedIn profile data.
- **1.4 Memory Management:** Maintains conversational or session memory for ongoing interactions.
- **1.5 Output Generation:** (Disabled in this workflow) Creates and updates Google Docs with the ICP analysis.
- **1.6 Documentation & Instructions:** Provides embedded notes and instructions on setup and usage.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for chat messages that provide LinkedIn profile URLs of customers to be analyzed.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - Type: Chat Trigger (LangChain)  
    - Role: Triggers workflow upon receiving chat input (LinkedIn URLs)  
    - Configuration: No special options; webhook enabled with ID "668cdf30-a5a6-4589-a0b4-857fc3ee23c1"  
    - Inputs: External chat interface  
    - Outputs: Passes received URLs to AI Agent node  
    - Edge Cases: Malformed or missing URLs in chat input may cause parsing issues downstream  
    - Version: 1.1

#### 1.2 AI Processing

- **Overview:**  
  This core block manages parsing LinkedIn URLs, enriching data, maintaining memory, and synthesizing the ICP using AI agents.

- **Nodes Involved:**  
  - AI Agent  
  - Anthropic Chat Model  
  - Simple Memory  
  - Airtop Data Enrichment

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain AI Agent  
    - Role: Central orchestrator parsing input, invoking enrichment tools, synthesizing ICP analysis, and generating output  
    - Configuration:  
      - System Message defines the AI as an expert ICP analyst with detailed instructions on input parsing, enrichment, analysis, output formatting, and scoring methodology.  
      - Handles multiple LinkedIn URLs separated by spaces or commas.  
      - Uses Airtop data enrichment tool for each profile sequentially.  
      - Tracks patterns and anomalies across customer data.  
      - Produces structured ICP output including customer list, executive summary, key attributes, weighting, exclusion criteria, scoring methodology, and a Google Boolean search query.  
    - Inputs: Receives URLs from "When chat message received" node  
    - Outputs: Sends enriched and analyzed data to Google Docs (disabled in current workflow)  
    - Edge Cases:  
      - Parsing errors if URLs are malformed or concatenated improperly  
      - Airtop tool failures or timeouts during enrichment  
      - AI model limitations or rate limits  
    - Version: 1.9

  - **Anthropic Chat Model**  
    - Type: LangChain Language Model (LLM) using Claude 3.7 Sonnet  
    - Role: Provides AI language model capabilities for ICP synthesis and reasoning  
    - Configuration: Model set to "claude-3-7-sonnet-20250219"  
    - Credentials: Uses Anthropic API key (Cesar's Key)  
    - Inputs: Connected as AI language model for AI Agent  
    - Outputs: Provides processed text back to AI Agent  
    - Edge Cases: API rate limits, response timeouts, malformed prompts  
    - Version: 1.3

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversational or session memory for the AI Agent to track context across multiple inputs or iterations  
    - Configuration: Default buffer window, no special parameters  
    - Inputs: Connected as AI memory for AI Agent  
    - Outputs: Provides stored memory context to AI Agent  
    - Edge Cases: Memory overflow if very large conversations or data sets occur  
    - Version: 1.3

  - **Airtop Data Enrichment**  
    - Type: Airtop Tool node (custom integration)  
    - Role: Enriches each LinkedIn profile URL by extracting structured data such as name, headline, experience, education, skills, certifications, and other profile details  
    - Configuration:  
      - URL parameter dynamically set from AI Agent output  
      - Prompt instructs extraction of detailed profile attributes (name, headline, location, current company/position, experience up to 10 positions, education up to 5 entries, skills up to 50, certifications up to 10, languages, connections, recommendations)  
      - Output schema strictly defined for structured data  
      - Uses Airtop profile "AmirLinkedin" in session mode "new"  
    - Credentials: Airtop API key (Airtop Official Org)  
    - Inputs: Invoked as AI tool from AI Agent for each profile URL  
    - Outputs: Returns enriched profile JSON for further AI analysis  
    - Edge Cases:  
      - API credential failures or limits  
      - Profile URL invalid or inaccessible  
      - Data extraction incomplete or inconsistent due to LinkedIn profile privacy settings  
    - Version: 1

#### 1.3 Output Generation (Disabled)

- **Overview:**  
  Intended to create and update Google Docs with the ICP analysis results. Currently disabled.

- **Nodes Involved:**  
  - Google Docs (Create)  
  - Google Docs1 (Update)

- **Node Details:**

  - **Google Docs**  
    - Type: Google Docs Node  
    - Role: Creates a new Google Document with timestamped title for ICP output  
    - Parameters: Title template includes ISO date-time string, folder ID set  
    - Credentials: Google Docs OAuth2 (Amir - Google Docs account)  
    - Disabled: true  
    - Edge Cases: OAuth token expiry, folder ID invalid, API quota exceeded  
    - Version: 2

  - **Google Docs1**  
    - Type: Google Docs Node  
    - Role: Updates the created document by inserting AI Agent output text  
    - Parameters: Insert text action configured, document URL dynamic from previous node  
    - Credentials: Same as above  
    - Disabled: true  
    - Edge Cases: Document URL invalid, concurrency issues, API failures  
    - Version: 2

#### 1.4 Documentation & Instructions

- **Overview:**  
  Provides embedded user instructions, setup notes, and workflow purpose descriptions via sticky notes.

- **Nodes Involved:**  
  - Sticky Note (Ideal Customer Profile Intro)  
  - Sticky Note1 (Airtop Setup Instructions)  
  - Sticky Note2 (README with detailed explanation)

- **Node Details:**

  - **Sticky Note**  
    - Content: Introduction to the ICP Airtop Agent, highlighting its purpose to find perfect customers faster using Airtop and AI.  
    - Positioned near trigger/start nodes for visibility.

  - **Sticky Note1**  
    - Content: Important setup steps to create an Airtop profile, connect to LinkedIn, and enter profile name in Airtop node. Includes link: https://portal.airtop.ai/browser-profiles

  - **Sticky Note2**  
    - Content: Comprehensive README-style instruction covering use case, input/output, workflow steps, setup requirements, and next steps for marketing teams.  
    - Explains how this automation generates ICP definitions, scoring models, and Boolean search queries from LinkedIn data.

---

### 3. Summary Table

| Node Name              | Node Type                                  | Functional Role                              | Input Node(s)              | Output Node(s)       | Sticky Note                                                                                   |
|------------------------|--------------------------------------------|----------------------------------------------|----------------------------|----------------------|-----------------------------------------------------------------------------------------------|
| When chat message received | Chat Trigger (LangChain)                  | Entry point; receives LinkedIn URLs           | External chat input         | AI Agent             |                                                                                               |
| AI Agent               | LangChain AI Agent                         | Parses input, orchestrates enrichment & analysis | When chat message received  | Google Docs (disabled) |                                                                                               |
| Anthropic Chat Model   | LangChain LLM (Claude AI)                  | Provides AI language capabilities for ICP synthesis | AI Agent (ai_languageModel) | AI Agent             |                                                                                               |
| Simple Memory          | LangChain Memory Buffer                    | Maintains session memory/context for AI Agent | AI Agent (ai_memory)        | AI Agent             |                                                                                               |
| Airtop Data Enrichment | Airtop Tool                               | Extracts structured LinkedIn profile data     | AI Agent (ai_tool)          | AI Agent             |                                                                                               |
| Google Docs            | Google Docs Node (Create)                  | Creates Google Doc for ICP output (disabled)  | AI Agent                   | Google Docs1         |                                                                                               |
| Google Docs1           | Google Docs Node (Update)                  | Updates Google Doc with ICP analysis (disabled) | Google Docs                |                      |                                                                                               |
| Sticky Note            | Sticky Note                               | Intro to ICP Airtop Agent                      |                            |                      | "## Ideal Customer Profile (ICP) Airtop Agent\n### Find your perfect customers faster.\nAirtop's AI agent defines your ICP with a few examples of your current best customers" |
| Sticky Note1           | Sticky Note                               | Airtop setup instructions                      |                            |                      | "## IMPORTANT\n\n1. Create an [Airtop Profile here](https://portal.airtop.ai/browser-profiles) \n\n2. Connect it to Linkedin\n\n3. Enter the Profile Name here"                 |
| Sticky Note2           | Sticky Note                               | Detailed README and instructions               |                            |                      | "README\n# Define Your ICP from Customer LinkedIn Profiles\n...\n[Full README content]"        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add **When chat message received** node (LangChain chat trigger)  
   - No special parameters needed  
   - Enable webhook for chat integration  

2. **Create AI Agent Node**  
   - Add **AI Agent** node (LangChain AI Agent)  
   - Configure system message with detailed ICP Analyst instructions including:  
     - Parsing multiple LinkedIn URLs separated by spaces/commas  
     - Using Airtop Data Enrichment tool sequentially for each URL  
     - Synthesizing ICP definition and scoring methodology  
     - Output formatting requirements  
   - Connect input from "When chat message received" node main output  

3. **Add Anthropic Chat Model Node**  
   - Add **Anthropic Chat Model** node (LangChain LLM)  
   - Set model to "claude-3-7-sonnet-20250219"  
   - Provide Anthropic API credentials (OAuth/API key)  
   - Connect as AI language model (ai_languageModel) input to AI Agent  

4. **Add Simple Memory Node**  
   - Add **Simple Memory** node (LangChain Memory Buffer)  
   - Use default buffer window parameters  
   - Connect as AI memory (ai_memory) input to AI Agent  

5. **Add Airtop Data Enrichment Node**  
   - Add **Airtop Data Enrichment** node (Airtop Tool)  
   - Configure with:  
     - URL parameter dynamically set from AI Agent output (expression)  
     - Prompt to extract detailed LinkedIn profile attributes (name, headline, experience, education, skills, certifications, languages, connections, recommendations)  
     - Output schema set to strict JSON as defined  
     - Use Airtop profile connected to LinkedIn (e.g., "AmirLinkedin")  
   - Provide Airtop API credentials  
   - Connect as AI tool (ai_tool) input to AI Agent  

6. **(Optional) Add Google Docs Nodes**  
   - Add **Google Docs** node to create a new document:  
     - Set title to dynamic template with ISO timestamp (e.g., "ICP Profile 2025-04-15_10-30")  
     - Set folder ID for document storage  
     - Provide Google Docs OAuth2 credentials  
   - Add **Google Docs1** node to update the document:  
     - Configure insert action to append AI Agent output text  
     - Connect output of Google Docs to Google Docs1 input  
     - Connect AI Agent output to Google Docs input  
   - (Note: These nodes are disabled by default and can be enabled if desired)  

7. **Add Sticky Notes for Documentation**  
   - Add sticky notes near relevant nodes with:  
     - Workflow introduction and purpose  
     - Airtop setup instructions with link (https://portal.airtop.ai/browser-profiles)  
     - Detailed README explaining use case, input/output, workflow steps, and next steps  

8. **Connect Nodes in Sequence**  
   - Connect "When chat message received" → "AI Agent" → "Google Docs" (optional) → "Google Docs1" (optional)  
   - Connect "Anthropic Chat Model" to AI Agent (ai_languageModel) input  
   - Connect "Simple Memory" to AI Agent (ai_memory) input  
   - Connect "Airtop Data Enrichment" to AI Agent (ai_tool) input  

9. **Set Credentials**  
   - Configure Airtop API credentials with an active Airtop profile connected to LinkedIn  
   - Configure Anthropic API credentials for Claude AI  
   - Configure Google Docs OAuth2 credentials if enabling document creation  

10. **Test Workflow**  
    - Send chat message with one or more LinkedIn profile URLs separated by spaces or commas  
    - Verify enrichment, AI analysis, and (if enabled) Google Docs creation and update  
    - Monitor for parsing errors, API limits, and connection issues  

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                      |
|------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Airtop Profile creation and LinkedIn integration are required before running this workflow                                               | https://portal.airtop.ai/browser-profiles           |
| This workflow is tagged as "Marketing" and targets ICP definition and customer segmentation use cases                                   | Workflow metadata                                    |
| Claude 3.7 Sonnet model by Anthropic is used for AI synthesis                                                                           | Anthropic API integration                            |
| Google Docs integration is optional and disabled by default; requires OAuth2 setup                                                      | Google Docs API                                      |
| Workflow includes detailed embedded README note explaining use case and setup steps                                                     | Sticky Note2 content                                 |
| The AI Agent is designed to parse multiple LinkedIn URLs and enrich them sequentially using Airtop tool                                 | AI Agent system message                              |
| The output includes a Google Boolean search query to find ICP-matching prospects                                                        | AI Agent system message                              |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, strictly adhering to content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.