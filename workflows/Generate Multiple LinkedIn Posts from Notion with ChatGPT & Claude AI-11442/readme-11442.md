Generate Multiple LinkedIn Posts from Notion with ChatGPT & Claude AI

https://n8nworkflows.xyz/workflows/generate-multiple-linkedin-posts-from-notion-with-chatgpt---claude-ai-11442


# Generate Multiple LinkedIn Posts from Notion with ChatGPT & Claude AI

### 1. Workflow Overview

This workflow automates the generation of LinkedIn posts from content notes stored in a Notion database using advanced AI language models (Claude and ChatGPT). It is designed for content creators or marketing teams who want to scale LinkedIn content production with high-quality, brand-consistent posts derived from existing planning notes.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Validation**: Watches for updates in the Notion "Content Plan" database and filters entries tagged for LinkedIn and marked as ready for writing.
- **1.2 Main LinkedIn Post Generation**: Uses Claude AI to create a primary LinkedIn post from the project name and detailed notes.
- **1.3 Multi-Post Ideation and Generation**: Utilizes ChatGPT to generate three distinct post concepts/outlines, then Claude AI creates a complete LinkedIn post for each concept.
- **1.4 Saving Posts Back to Notion**: Stores both the main post and the multiple generated posts back into the Notion database for review and publication.
- **1.5 Status Update**: Modifies the status in Notion to prevent duplicate processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Validation

- **Overview:**  
  This block monitors the Notion content plan database hourly for any updated pages. It ensures that only items tagged specifically for LinkedIn and marked as ready are processed.

- **Nodes Involved:**  
  - Notion Content Plan Trigger  
  - If (Channel check)  
  - If1 (Status check)  
  - Update Status to prevent re-running

- **Node Details:**  

  - **Notion Content Plan Trigger**  
    - *Type:* Notion Trigger  
    - *Role:* Watches the Notion database for page updates hourly.  
    - *Configuration:* Trigger event set to "pagedUpdatedInDatabase" with polling every hour. Connected to a specific Notion database ID.  
    - *Inputs:* None (event-driven)  
    - *Outputs:* Emits updated page data.  
    - *Potential Failures:* API rate limits, credential/authentication errors, missing database ID.  
    - *Sticky Note:* Explains the trigger and verification logic (Sticky Note1).  

  - **If (Channel check)**  
    - *Type:* If Node  
    - *Role:* Checks if the "Channel" field in the Notion item equals "LinkedIn_Scratch".  
    - *Configuration:* Uses expression to compare the first element of the Channel multi-select property.  
    - *Inputs:* From Notion Content Plan Trigger  
    - *Outputs:* Passes only relevant records for LinkedIn processing.  
    - *Edge Cases:* Tag mismatch or missing field causes nodes downstream not to run.  

  - **If1 (Status check)**  
    - *Type:* If Node  
    - *Role:* Checks if the "Status" field is "n8n_ready" to confirm content is ready for writing.  
    - *Inputs:* From If node (Channel check)  
    - *Outputs:* Branches to processing nodes if ready, else stops.  
    - *Edge Cases:* Status field missing or different spelling.  

  - **Update Status to prevent re-running**  
    - *Type:* Notion (DatabasePage Update)  
    - *Role:* Updates the Notion page status to "n8n_runing" to avoid multiple triggers on the same content.  
    - *Inputs:* From If1 node’s true branch  
    - *Configuration:* Updates the "Status" property on the current page.  
    - *Edge Cases:* API write failures, incorrect page ID.  

---

#### 2.2 Main LinkedIn Post Generation

- **Overview:**  
  Generates a professional, polished LinkedIn post from the project name and notes using the Claude AI agent with a custom system prompt tailored to a startup founder’s voice and brand style.

- **Nodes Involved:**  
  - Claude Model 1 (Language model node)  
  - Claude Agent - Main LinkedIn Post  
  - Save Main Post to Notion

- **Node Details:**  

  - **Claude Model 1**  
    - *Type:* LangChain Anthropic Model Node  
    - *Role:* Connects the Claude Sonnet 4.5 AI model to the workflow for text generation.  
    - *Configuration:* Uses "claude-sonnet-4-5-20250929" model with temperature 0.8 for creative output.  
    - *Inputs:* Receives prompt from Claude Agent - Main LinkedIn Post node.  
    - *Outputs:* Returns AI-generated post text.  
    - *Edge Cases:* API quota limits, network timeouts, malformed input.  

  - **Claude Agent - Main LinkedIn Post**  
    - *Type:* LangChain Agent  
    - *Role:* Prepares a detailed prompt for Claude AI to write one compelling LinkedIn post.  
    - *Configuration:* Uses a rich system message defining voice, tone, structure, and brand guidelines; inputs project name and notes dynamically from Notion trigger.  
    - *Inputs:* From If1 true branch, project name and notes from trigger node.  
    - *Outputs:* Text of generated LinkedIn post.  
    - *Expressions:* Dynamic reference to Notion fields for topic and notes.  
    - *Edge Cases:* Missing or empty notes causing poor output.  

  - **Save Main Post to Notion**  
    - *Type:* Notion node (databasePage create)  
    - *Role:* Saves the generated main LinkedIn post back into a Notion database marked as completed and linked to the original content.  
    - *Inputs:* Output of Claude Agent - Main LinkedIn Post  
    - *Configuration:* Sets title, priority, channel (Linkedin_Repurpose), status (n8n_completed), and relation to original article.  
    - *Edge Cases:* Write permission errors, invalid database ID.  
    - *Sticky Note:* Describes main post creation and saving (Sticky Note2).  

---

#### 2.3 Multi-Post Ideation and Generation

- **Overview:**  
  Generates three distinct LinkedIn post concept outlines using ChatGPT, then creates a finalized post for each concept with Claude AI. Allows content creators to select from multiple options.

- **Nodes Involved:**  
  - ChatGPT idea generation  
  - Split Out1  
  - Claude Model 2  
  - Claude Agent - 3 LinkedIn Posts  
  - Save Each Post to Notion

- **Node Details:**  

  - **ChatGPT idea generation**  
    - *Type:* LangChain OpenAI Node  
    - *Role:* Generates three distinct LinkedIn post concepts with outlines and drafts based on the project name and notes.  
    - *Configuration:* Uses GPT-5 model, outputs JSON with concepts and selected post.  
    - *Inputs:* From If1 true branch, project name and notes.  
    - *Outputs:* JSON containing post concept array and selected post draft.  
    - *Expressions:* Dynamic insertion of Notion fields for title and notes.  
    - *Edge Cases:* Malformed JSON output, missing input data, API limits.  
    - *Sticky Note:* Explains two-step generation for creativity and flexibility (Sticky Note3).  

  - **Split Out1**  
    - *Type:* Split Out Node  
    - *Role:* Extracts each post concept outline from ChatGPT output to process individually.  
    - *Inputs:* Output of ChatGPT idea generation  
    - *Outputs:* One output per concept for further processing.  
    - *Edge Cases:* Empty or malformed "outline" field.  

  - **Claude Model 2**  
    - *Type:* LangChain Anthropic Model Node  
    - *Role:* Provides Claude AI model for final post generation on each concept.  
    - *Configuration:* Same model "claude-sonnet-4-5-20250929" with default temperature (more consistent).  
    - *Inputs:* Prompt from Claude Agent - 3 LinkedIn Posts node.  
    - *Outputs:* Final LinkedIn post for each concept.  

  - **Claude Agent - 3 LinkedIn Posts**  
    - *Type:* LangChain Agent  
    - *Role:* Converts each concept outline into a full LinkedIn post using Claude AI with the brand voice and style system prompt.  
    - *Inputs:* Split outlines from Split Out1 node.  
    - *Outputs:* Completed LinkedIn posts per outline.  
    - *Expressions:* Uses topic, key points, and angle from each split outline item.  
    - *Edge Cases:* Missing fields, generation errors.  

  - **Save Each Post to Notion**  
    - *Type:* Notion node (databasePage create)  
    - *Role:* Saves each generated LinkedIn post variation back to Notion with metadata linking it to the original article.  
    - *Inputs:* Outputs of Claude Agent - 3 LinkedIn Posts  
    - *Configuration:* Similar metadata as main post; titles prefixed with "[LinkedIn]".  
    - *Edge Cases:* Write failures, API limits.  

---

### 3. Summary Table

| Node Name                    | Node Type                        | Functional Role                                 | Input Node(s)                   | Output Node(s)                  | Sticky Note                                               |
|------------------------------|---------------------------------|------------------------------------------------|--------------------------------|--------------------------------|-----------------------------------------------------------|
| Notion Content Plan Trigger   | notionTrigger                   | Triggers on Notion DB page updates for LinkedIn| None                           | If                             | ## Trigger and Verification: hourly check, channel/status |
| If                           | If Node                        | Checks if Channel is "LinkedIn_Scratch"         | Notion Content Plan Trigger    | If1                            | ## Trigger and Verification                                |
| If1                          | If Node                        | Checks if Status is "n8n_ready"                  | If                            | Claude Agent - Main LinkedIn Post, ChatGPT idea generation, Update Status to prevent re-running | ## Trigger and Verification                                |
| Update Status to prevent re-running | notion (databasePage update) | Updates status to prevent duplicate runs        | If1                           | None                          |                                                           |
| Claude Model 1               | LangChain Anthropic Model       | Claude AI model for main LinkedIn post          | Claude Agent - Main LinkedIn Post | Claude Agent - Main LinkedIn Post |                                                           |
| Claude Agent - Main LinkedIn Post | LangChain Agent             | Prepares prompt and generates main LinkedIn post| If1                           | Claude Model 1                 | ## Main Post: creates standardized post                   |
| Save Main Post to Notion      | notion (databasePage create)    | Saves main LinkedIn post back to Notion          | Claude Agent - Main LinkedIn Post | None                          | ## Main Post                                               |
| ChatGPT idea generation       | LangChain OpenAI                | Generates 3 post concepts/outlines & draft       | If1                           | Split Out1                    | ## Create Variations for creativity                        |
| Split Out1                   | Split Out                      | Splits ChatGPT output into individual post outlines| ChatGPT idea generation       | Claude Agent - 3 LinkedIn Posts | ## Create Variations for creativity                        |
| Claude Model 2               | LangChain Anthropic Model       | Claude AI model for multi-post generation        | Claude Agent - 3 LinkedIn Posts | Claude Agent - 3 LinkedIn Posts |                                                           |
| Claude Agent - 3 LinkedIn Posts | LangChain Agent              | Generates final LinkedIn posts from concepts      | Split Out1                    | Claude Model 2                | ## Create Variations for creativity                        |
| Save Each Post to Notion      | notion (databasePage create)    | Saves each generated LinkedIn post variation     | Claude Agent - 3 LinkedIn Posts | None                          | ## Create Variations for creativity                        |
| Sticky Note                  | Sticky Note                    | Documentation and explanation                     | None                         | None                          | See summary notes content                                  |
| Sticky Note1                 | Sticky Note                    | Explains trigger and validation                   | None                         | None                          | See summary notes content                                  |
| Sticky Note2                 | Sticky Note                    | Explains main post generation                      | None                         | None                          | See summary notes content                                  |
| Sticky Note3                 | Sticky Note                    | Explains multi-post ideation and generation       | None                         | None                          | See summary notes content                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Notion Content Plan Trigger Node**  
   - Type: Notion Trigger  
   - Event: "pagedUpdatedInDatabase"  
   - Polling: Every hour  
   - Database ID: Set your Notion "Content Plan" database ID  
   - Credentials: Link your Notion API credentials  

2. **Add If Node (Channel Check)**  
   - Condition: Check if `Channel[0]` equals `"LinkedIn_Scratch"`  
   - Input: Connect from Notion Content Plan Trigger  
   - Outputs: True branch continues processing, false branch stops  

3. **Add If Node (Status Check)**  
   - Condition: Check if `Status` equals `"n8n_ready"`  
   - Input: Connect from If (Channel Check) true output  
   - Outputs: True branch proceeds to content generation, else stops  

4. **Add Update Status Node**  
   - Type: Notion (databasePage update)  
   - Operation: Update current page's `Status` property to `"n8n_runing"`  
   - Input: Connect from If1 (Status Check) true branch  
   - Set page ID dynamically from trigger item  

5. **Add Claude Model 1 Node**  
   - Type: LangChain Anthropic Model Node  
   - Model: `claude-sonnet-4-5-20250929`  
   - Temperature: 0.8 (creative)  
   - Credentials: Anthropic API key  

6. **Add Claude Agent - Main LinkedIn Post Node**  
   - Type: LangChain Agent  
   - Connect to Claude Model 1 (ai_languageModel input)  
   - Configure prompt:  
     - Use system prompt for LinkedIn post writer with founder tone (as in workflow)  
     - Input text includes:  
       - Topic: from Notion trigger field `Project name`  
       - Article notes: from Notion trigger field `Notes`  
   - Input: Connect from If1 (Status Check) true branch  

7. **Connect Claude Agent - Main LinkedIn Post output to Save Main Post to Notion**  
   - Type: Notion (databasePage create)  
   - Database ID: Your target Notion posts database  
   - Title: `=[LinkedIn] {{Project name}} - 1`  
   - Content: Save AI-generated post output under a heading "LinkedIn Post"  
   - Properties:  
     - Priority: Medium  
     - Channel: Linkedin_Repurpose  
     - Status: n8n_completed  
     - Parent_article: Relation to original Notion item ID  
   - Credentials: Notion API credentials  

8. **Add ChatGPT Idea Generation Node**  
   - Type: LangChain OpenAI  
   - Model: GPT-5 (or latest available)  
   - Credentials: OpenAI API key  
   - Configure messages:  
     - System prompt instructing to generate 3 distinct LinkedIn post concepts and one full draft  
     - Input variables: project name and notes from Notion trigger  
   - Output: JSON with `post_concepts` and `selected_post`  

9. **Add Split Out Node (Split Out1)**  
   - Type: Split Out  
   - Field to split: `choices[0].message.content.outline` (from ChatGPT output)  
   - Input: Connect from ChatGPT idea generation  

10. **Add Claude Model 2 Node**  
    - Same configuration as Claude Model 1 but with default temperature (e.g., 0.0–0.7)  
    - Credentials: Anthropic API key  

11. **Add Claude Agent - 3 LinkedIn Posts Node**  
    - Type: LangChain Agent  
    - Connect to Claude Model 2 (ai_languageModel)  
    - Configure prompt:  
      - Similar system prompt as main post but tailored to generate post per outline  
      - Inputs: topic, key points, and angle from Split Out1 data  
    - Input: Connect from Split Out1  

12. **Add Save Each Post to Notion Node**  
    - Type: Notion (databasePage create)  
    - Database ID: Same as main post save node  
    - Title: `=[LinkedIn] {{topic}}` (from each outline item)  
    - Content: AI-generated LinkedIn post text under heading "LinkedIn Post"  
    - Properties:  
      - Priority: Medium  
      - Channel: Linkedin_Repurpose  
      - Status: n8n_completed  
      - Parent_article: Relation to original Notion item ID  

13. **Set Connections**  
    - Notion Content Plan Trigger → If (Channel Check)  
    - If (Channel Check) true → If1 (Status Check)  
    - If1 true → Claude Agent - Main LinkedIn Post  
    - Claude Agent - Main LinkedIn Post → Claude Model 1 → Save Main Post to Notion  
    - If1 true → ChatGPT Idea Generation → Split Out1 → Claude Agent - 3 LinkedIn Posts → Claude Model 2 → Save Each Post to Notion  
    - If1 true → Update Status to prevent re-running  

14. **Credentials Setup**  
    - Configure Notion API credentials with access to both content plan and posts databases  
    - Configure Anthropic API credentials for Claude nodes  
    - Configure OpenAI API credentials for ChatGPT node  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| Automated workflow that transforms Notion content notes into publication-ready LinkedIn posts using Claude AI. Monitors Notion database and generates multiple variations based on structured outlines, so that the author can pick the one they like most. Use cases include automating LinkedIn content creation, generating multiple post variations, and maintaining consistent voice and formatting across posts. AI setup uses Claude Sonnet 4.5 and GPT-5, with custom brand voice prompts. Workflow triggers hourly on content updates.| General workflow summary (Sticky Note)  |
| Trigger block checks for new content hourly, verifying Channel and Status to ensure only LinkedIn content is processed. Note: You must update the Notion text field or headline for it to register as an update.                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note1                            |
| Main Post block creates a standardized post and writes the information back to Notion.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note2                            |
| Multi-post generation uses a two-step process: ChatGPT generates ideas for variation; Claude creates ready-to-publish LinkedIn posts for each variation. This approach avoids needing to retrigger the workflow if a version is disliked and allows flexible adjustment to different topics.                                                                                                                                                                                                                                                                                                                                                              | Sticky Note3                            |
| LinkedIn posts are optimized to be under 1800 characters, use a conversational but authoritative tone, incorporate personal reflections and data-driven insights, and follow a defined structure (Hook → Insight → Implication → Question). The workflow ensures posts are ready to publish without further editing.                                                                                                                                                                                                                                                                                                                                        | Brand and style guidelines in system prompts |
| Credential requirements: Notion API with database access, Anthropic API key for Claude AI, OpenAI API key for ChatGPT.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Credentials setup                       |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, respecting all applicable content policies and handling only legal and public data.