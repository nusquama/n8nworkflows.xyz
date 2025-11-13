Create Multi-Channel Content with O3 Director & GPT-4 Specialist Agents

https://n8nworkflows.xyz/workflows/create-multi-channel-content-with-o3-director---gpt-4-specialist-agents-6910


# Create Multi-Channel Content with O3 Director & GPT-4 Specialist Agents

---

### 1. Workflow Overview

This n8n workflow, titled **"Create Multi-Channel Content with O3 Director & GPT-4 Specialist Agents"**, automates the generation of cohesive, multi-format marketing content across several communication channels. It is designed for content marketers, digital agencies, and businesses seeking to produce blog articles, social media posts, video scripts, email newsletters, website copy, and strategic content plans in a coordinated manner.

The workflow is architected as a multi-agent AI system, where a central **Content Director Agent** (powered by the OpenAI O3 model) orchestrates specialized AI agents (each powered by GPT-4.1-mini) responsible for specific content types. It leverages a chat trigger to receive content requests and then delegates tasks to experts in blog writing, social media, video scripts, email marketing, website copy, and content strategy.

**Logical Blocks:**

- **1.1 Input Reception**: Receives incoming content requests via a chat message webhook trigger.
- **1.2 Content Director Processing**: Uses the O3 AI model to analyze requests and orchestrate content generation.
- **1.3 Specialist Content Generation Agents**: Six specialized agents produce content in their respective formats, each using GPT-4.1-mini.
- **1.4 Model Configuration and Credentialing**: Each agent is paired with a dedicated OpenAI GPT model node configured with required credentials.
- **1.5 Documentation & User Guidance**: Visual sticky notes provide workflow overview, usage instructions, and contact information.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives the initial user input via a webhook chat message, serving as the entry point for the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Listens for inbound chat messages via webhook to trigger the workflow.  
    - Configuration: Uses webhook ID `content-webhook-id`. No additional options set.  
    - Key expressions: None. Raw user message is the input.  
    - Input: External HTTP request via webhook.  
    - Output: Triggers the Content Director Agent node.  
    - Version: 1.1  
    - Edge cases: Webhook connectivity failure, missing or malformed chat message payload, webhook authorization issues.  

---

#### 1.2 Content Director Processing

- **Overview:**  
  This block analyzes the input request and coordinates the entire content creation process. It uses the O3 OpenAI chat model to strategically plan and dispatch tasks to specialist agents.

- **Nodes Involved:**  
  - Content Director Agent  
  - OpenAI Chat Model Content Director  
  - Think  

- **Node Details:**

  - **Content Director Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Central AI agent that coordinates multiple specialist AI tools based on user input and strategic content planning.  
    - Configuration: No special parameters; receives input from chat trigger and specialist agents.  
    - Input: Main input from "When chat message received" node; tool inputs from specialist agents.  
    - Output: Routes tasks to specialist agents via tool connections.  
    - Version: 2.1  
    - Edge cases: Agent logic failure, misrouting of tasks, unexpected input formats.  

  - **OpenAI Chat Model Content Director**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides the O3 model for strategic content direction.  
    - Configuration: Uses model `"o3"` with default options.  
    - Credentials: Requires valid OpenAI API credentials.  
    - Input: Connected to Content Director Agent as the language model backend.  
    - Output: AI-generated strategic content direction.  
    - Version: 1.2  
    - Edge cases: API quota limits, network timeout, invalid credentials.  

  - **Think**  
    - Type: `@n8n/n8n-nodes-langchain.toolThink`  
    - Role: Auxiliary tool for internal reasoning or intermediate processing by the Content Director Agent.  
    - Configuration: Default settings.  
    - Input: Triggered by Content Director Agent.  
    - Output: Feeds back into Content Director Agent.  
    - Version: 1.1  
    - Edge cases: Processing delays, unexpected input formats.

---

#### 1.3 Specialist Content Generation Agents

- **Overview:**  
  Six specialist agents generate channel-specific content based on the directives from the Content Director. Each agent uses a tailored GPT-4.1-mini model to produce content such as blog posts, social media updates, video scripts, emails, website copy, and content strategies.

- **Nodes Involved:**  
  - Blog Content Writer  
  - Social Media Content Creator  
  - Video Script Writer  
  - Email Newsletter Writer  
  - Website Copy Specialist  
  - Content Strategist & Planner  

  Each agent is linked with a corresponding OpenAI Chat Model node:

  - OpenAI Chat Model1 (Blog)  
  - OpenAI Chat Model2 (Social)  
  - OpenAI Chat Model3 (Video)  
  - OpenAI Chat Model4 (Email)  
  - OpenAI Chat Model5 (Website)  
  - OpenAI Chat Model6 (Strategy)  

- **Node Details:**

  For each content agent node:

  - **Blog Content Writer** (and analogous for others)  
    - Type: `@n8n/n8n-nodes-langchain.agentTool`  
    - Role: AI tool specialized in generating a specific type of content. For example, Blog Content Writer focuses on long-form articles and thought leadership.  
    - Configuration:  
      - Text input: Interpolated from the user message via expression `={{ $fromAI('Prompt__User_Message_', '', 'string') }}` to pass the original prompt.  
      - Tool description specifies the content specialty (e.g., blog writing, social media, video scripts).  
    - Input: Receives text input from the Content Director Agent.  
    - Output: Sends generated content back to Content Director Agent for aggregation.  
    - Version: 2.2  
    - Edge cases: Model misinterpretation, content quality issues, API timeouts.  

  For each OpenAI Chat Model node tied to an agent:

  - **OpenAI Chat Model1** (and others similarly)  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides the GPT-4.1-mini language model for the associated content specialist agent.  
    - Configuration: Uses model `"gpt-4.1-mini"` with default options.  
    - Credentials: Requires valid OpenAI API credentials.  
    - Input: Connected to the respective content specialist agent as the language model.  
    - Output: AI-generated content.  
    - Version: 1.2  
    - Edge cases: API key limits, service unavailability.

---

#### 1.4 Model Configuration and Credentialing

- **Overview:**  
  Each AI agent node is linked to an OpenAI chat model node, which requires valid OpenAI credentials. Credential configuration is centralized and must be completed before workflow execution.

- **Nodes Involved:**  
  - All OpenAI Chat Model nodes (Content Director and specialists)

- **Node Details:**

  - Credentials:  
    - All OpenAI Chat Model nodes require the `OpenAI account` credential setup with a valid API key.  
    - Credential placeholder ID: `"YOUR_OPENAI_CREDENTIAL_ID"` must be replaced with real credentials in deployment.  

  - Edge cases:  
    - Missing or invalid credentials cause authentication errors.  
    - API rate-limiting may interrupt processing.  

---

#### 1.5 Documentation & User Guidance

- **Overview:**  
  Sticky Notes provide comprehensive user instructions, workflow overview, use cases, and contact information for support.

- **Nodes Involved:**  
  - Sticky Note Header  
  - Sticky Note Main

- **Node Details:**

  - **Sticky Note Header**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Content: Contact info, branding, and tutorial links for support.  
    - Position: Top-left corner of the canvas for visibility.  
    - Edge cases: None (purely informational).  

  - **Sticky Note Main**  
    - Type: `n8n-nodes-base.stickyNote`  
    - Content: Detailed workflow overview, agent roles, use cases, cost optimization, and hashtags.  
    - Format: Markdown with tables and sections for clarity.  
    - Edge cases: None (informational only).  

---

### 3. Summary Table

| Node Name                      | Node Type                             | Functional Role                          | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                                  |
|--------------------------------|-------------------------------------|----------------------------------------|------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| When chat message received      | chatTrigger                         | Entry point: receives user content requests | External webhook             | Content Director Agent           |                                                                                                              |
| Content Director Agent          | agent                              | Coordinates content generation and strategy | When chat message received, Specialist agents | Think, Specialist agents         |                                                                                                              |
| Think                          | toolThink                         | Auxiliary reasoning tool for Director Agent | Content Director Agent        | Content Director Agent           |                                                                                                              |
| Blog Content Writer            | agentTool                         | Generates blog content                   | Content Director Agent        | Content Director Agent           |                                                                                                              |
| Social Media Content Creator    | agentTool                         | Generates social media content           | Content Director Agent        | Content Director Agent           |                                                                                                              |
| Video Script Writer            | agentTool                         | Generates video scripts                   | Content Director Agent        | Content Director Agent           |                                                                                                              |
| Email Newsletter Writer        | agentTool                         | Generates email marketing content        | Content Director Agent        | Content Director Agent           |                                                                                                              |
| Website Copy Specialist        | agentTool                         | Generates website copy                    | Content Director Agent        | Content Director Agent           |                                                                                                              |
| Content Strategist & Planner    | agentTool                         | Generates content strategy and planning  | Content Director Agent        | Content Director Agent           |                                                                                                              |
| OpenAI Chat Model Content Director | lmChatOpenAi                   | Language model for Content Director      | Content Director Agent        | Content Director Agent           | Requires OpenAI API credentials; model: "o3"                                                                  |
| OpenAI Chat Model1             | lmChatOpenAi                     | Language model for Blog Content Writer   | Blog Content Writer           | Blog Content Writer              | Requires OpenAI API credentials; model: "gpt-4.1-mini"                                                        |
| OpenAI Chat Model2             | lmChatOpenAi                     | Language model for Social Media Content Creator | Social Media Content Creator | Social Media Content Creator     | Requires OpenAI API credentials; model: "gpt-4.1-mini"                                                        |
| OpenAI Chat Model3             | lmChatOpenAi                     | Language model for Video Script Writer   | Video Script Writer           | Video Script Writer              | Requires OpenAI API credentials; model: "gpt-4.1-mini"                                                        |
| OpenAI Chat Model4             | lmChatOpenAi                     | Language model for Email Newsletter Writer | Email Newsletter Writer       | Email Newsletter Writer          | Requires OpenAI API credentials; model: "gpt-4.1-mini"                                                        |
| OpenAI Chat Model5             | lmChatOpenAi                     | Language model for Website Copy Specialist | Website Copy Specialist       | Website Copy Specialist          | Requires OpenAI API credentials; model: "gpt-4.1-mini"                                                        |
| OpenAI Chat Model6             | lmChatOpenAi                     | Language model for Content Strategist & Planner | Content Strategist & Planner | Content Strategist & Planner     | Requires OpenAI API credentials; model: "gpt-4.1-mini"                                                        |
| Sticky Note Header             | stickyNote                       | Provides contact info and branding       | None                        | None                           | =======================================<br> For any questions or support, please contact: Yaron@nofluff.online<br> YouTube: https://www.youtube.com/@YaronBeen/videos<br> LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note Main               | stickyNote                       | Workflow overview, use cases, and tips   | None                        | None                           | Multi-agent AI content creation workflow overview with use cases, cost optimization, tags, and hashtags      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and set metadata as desired.**

2. **Add the Input Reception node:**
   - Add a **chatTrigger** node named `"When chat message received"`.
   - Set its webhook ID to `"content-webhook-id"` or a unique identifier.
   - No additional options required.
   - This node receives user content requests via HTTP webhook.

3. **Add Content Director Processing nodes:**
   - Add an **agent** node named `"Content Director Agent"`.
   - Connect `"When chat message received"` node's main output to this node's main input.
   - Add an **lmChatOpenAi** node named `"OpenAI Chat Model Content Director"`.
     - Set model to `"o3"` (select from the model list).
     - Configure with valid OpenAI API credentials.
   - Connect `"OpenAI Chat Model Content Director"` node's output to `"Content Director Agent"` node's language model input.
   - Add a **toolThink** node named `"Think"`.
   - Connect `"Think"` node's output to `"Content Director Agent"` node's tool input.
   - Connect `"Content Director Agent"` node's tool output to `"Think"` node to enable reasoning feedback loop.

4. **Add Specialist Content Generation agents:**

   For each content type (Blog, Social, Video, Email, Website, Strategy):

   - Add an **agentTool** node with the respective name and description:
     - **Blog Content Writer**: specialism in long-form articles.
     - **Social Media Content Creator**: social posts and engagement.
     - **Video Script Writer**: video marketing scripts.
     - **Email Newsletter Writer**: email campaigns.
     - **Website Copy Specialist**: landing pages and product descriptions.
     - **Content Strategist & Planner**: editorial calendars and planning.
   - In each agentTool node, set the text parameter to:  
     `={{ $fromAI('Prompt__User_Message_', '', 'string') }}` to forward the user's original prompt.
   - Add an **lmChatOpenAi** node per agent:
     - Name appropriately (e.g., `"OpenAI Chat Model1"` for Blog Content Writer).
     - Set model to `"gpt-4.1-mini"`.
     - Configure with the same OpenAI API credentials.
   - Connect each **lmChatOpenAi** node's output to the corresponding **agentTool** node's language model input.
   - Connect each **agentTool** node's tool output back to `"Content Director Agent"` node's tool input, enabling the director to receive content outputs.

5. **Set up connections:**
   - Ensure `"Content Director Agent"` node's tool outputs connect to all six specialist agents.
   - Each specialist agent's output connects back to `"Content Director Agent"` node.
   - The `"Think"` node connects as a tool to `"Content Director Agent"` to support strategic reasoning.

6. **Add Sticky Notes for documentation:**
   - Add two sticky note nodes:
     - `"Sticky Note Header"` with contact info and branding.
     - `"Sticky Note Main"` with full workflow description, use cases, and agent roles.
   - Format content as Markdown for clarity.
   - Position notes for easy visibility on the canvas.

7. **Configure credentials:**
   - Create or import OpenAI API credentials in n8n.
   - Assign these credentials to all OpenAI Chat Model nodes.
   - Verify credential validity and API quota availability.

8. **Test the workflow:**
   - Trigger the webhook with a sample content request message, e.g.,  
     `"Create a complete content campaign for our new SaaS product launch"`.
   - Monitor node executions and outputs.
   - Troubleshoot any API errors or misconfigurations.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                             | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow powered by a multi-agent system combining OpenAI's O3 model for content direction and GPT-4.1-mini for cost-effective specialized content generation.                          | Workflow overview in sticky note main content.                                                    |
| Contact for support: Yaron@nofluff.online                                                                                                                                                 | Sticky Note Header content.                                                                       |
| Explore video tutorials and tips: https://www.youtube.com/@YaronBeen/videos                                                                                                               | Sticky Note Header content.                                                                       |
| LinkedIn profile for workflow author: https://www.linkedin.com/in/yaronbeen/                                                                                                              | Sticky Note Header content.                                                                       |
| Use cases include product launches, brand storytelling, lead generation, thought leadership, social engagement, and email marketing campaigns.                                           | Sticky Note Main content.                                                                         |
| Cost optimization strategies: O3 model used only for strategic direction; GPT-4.1-mini used for specialist content to reduce costs by ~90%; parallel processing enables simultaneous tasks. | Sticky Note Main content.                                                                         |
| Workflow tags: #BlogWriting #SocialMediaContent #VideoScripts #EmailMarketing #WebsiteCopy #ContentStrategy #ContentMarketing #BrandStorytelling #DigitalContent #ContentCreation #n8n   | Sticky Note Main content.                                                                         |

---

**Disclaimer:** The provided text is exclusively generated from an automated workflow created with n8n, a powerful integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.