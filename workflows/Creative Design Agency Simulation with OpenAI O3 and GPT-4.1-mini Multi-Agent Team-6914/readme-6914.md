Creative Design Agency Simulation with OpenAI O3 and GPT-4.1-mini Multi-Agent Team

https://n8nworkflows.xyz/workflows/creative-design-agency-simulation-with-openai-o3-and-gpt-4-1-mini-multi-agent-team-6914


# Creative Design Agency Simulation with OpenAI O3 and GPT-4.1-mini Multi-Agent Team

---

### 1. Workflow Overview

This workflow simulates a **Creative Design Agency** environment using a multi-agent AI system powered by OpenAI models within n8n. It targets users who want to automate comprehensive creative design requests such as brand identity, UI/UX design, motion graphics, web design, and copywriting through a chat interface.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures chat messages as design requests.
- **1.2 Creative Direction:** A strategic Creative Director agent (powered by OpenAI O3) analyzes the input and orchestrates the workflow.
- **1.3 Specialist Creative Agents:** Six specialized agents (Graphic Designer, UI/UX Designer, Brand Strategist, Motion Graphics Designer, Web Designer, Creative Copywriter) each use GPT-4.1-mini to generate focused creative outputs.
- **1.4 Language Model Configuration:** Each agent is paired with an OpenAI language model node configured with the appropriate model version.
- **1.5 User Guidance and Documentation:** Sticky notes provide detailed workflow instructions, agent roles, use cases, and contact info.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages that initiate the creative design request workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Entry point webhook that captures chat messages from users.  
    - Configuration: Uses a unique webhook ID (`creative-webhook-id`) to listen for incoming chat messages. No extra parameters configured.  
    - Input: External chat system sends messages to this webhook.  
    - Output: Triggers the Creative Director Agent node.  
    - Edge Cases: Possible webhook unavailability, malformed input, or network errors.  
    - Version: 1.1

---

#### 2.2 Creative Direction

- **Overview:**  
  The Creative Director agent acts as the strategic decision maker, analyzing user input and delegating tasks to specialized design agents.

- **Nodes Involved:**  
  - Creative Director Agent  
  - OpenAI Chat Model Director  
  - Think

- **Node Details:**  
  - **Creative Director Agent**  
    - Type: LangChain Agent  
    - Role: Core agent that receives chat input, assesses requirements, and coordinates the workflow.  
    - Configuration: Default options, no special parameters.  
    - Input: Triggered by the chat message node.  
    - Output: Delegates tasks to all specialized creative agents and the Think node.  
    - Version: 2.1  
    - Edge Cases: Model API failure, delegation logic errors.

  - **OpenAI Chat Model Director**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides the language model backend (OpenAI O3) for the Creative Director Agent.  
    - Configuration: Model set to `o3` for strategic reasoning. Credentials required for OpenAI API access.  
    - Input: Connected only to Creative Director Agent node.  
    - Output: Supplies AI-generated responses to Creative Director Agent.  
    - Version: 1.2  
    - Edge Cases: API key invalidation, rate limiting.

  - **Think**  
    - Type: LangChain Tool Think  
    - Role: Acts as an internal tool for the Creative Director Agent to perform reasoning or hold intermediate thought processes.  
    - Configuration: Default, no parameters.  
    - Input: Triggered by Creative Director Agent.  
    - Output: Returns to Creative Director Agent.  
    - Version: 1.1  
    - Edge Cases: Expression evaluation failures.

---

#### 2.3 Specialist Creative Agents

- **Overview:**  
  These six agents take the delegated tasks from the Creative Director and execute specialized creative design work using GPT-4.1-mini models.

- **Nodes Involved:**  
  - Graphic Designer  
  - UI/UX Designer  
  - Brand Strategist  
  - Motion Graphics Designer  
  - Web Designer  
  - Creative Copywriter  
  - OpenAI Chat Model1 through OpenAI Chat Model6

- **Node Details:**  

  For each specialist agent:  
  - **Agent Node (e.g., Graphic Designer)**  
    - Type: LangChain Agent Tool  
    - Role: Executes specialized creative task using AI.  
    - Configuration:  
      - `text`: dynamically assigned from AI prompt output of the Creative Director (`{{ $fromAI('Prompt__User_Message_', '', 'string') }}`).  
      - `toolDescription`: Describes agent specialty (e.g., "visual identity creation, logo design...").  
    - Input: Receives delegated prompt from Creative Director Agent.  
    - Output: Produces specialized creative output.  
    - Version: 2.2  
    - Edge Cases: Model timeout, prompt formatting errors.

  - **OpenAI Chat Model Nodes (e.g., OpenAI Chat Model1)**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4.1-mini backend for corresponding agent.  
    - Configuration:  
      - Model: `gpt-4.1-mini`  
      - Credentials: OpenAI API credentials required.  
    - Input: Connected to corresponding specialist agent node.  
    - Output: Supplies AI-generated responses.  
    - Version: 1.2  
    - Edge Cases: API rate limits, invalid credentials.

- **Agent-Model Pairings:**  

| Agent                | Model Node          | Specialty Description                                                 |
|----------------------|---------------------|----------------------------------------------------------------------|
| Graphic Designer     | OpenAI Chat Model1   | Visual identity, logos, print materials, packaging, brand elements   |
| UI/UX Designer       | OpenAI Chat Model2   | User interface design, experience optimization, wireframing          |
| Brand Strategist     | OpenAI Chat Model3   | Brand positioning, market research, competitive analysis             |
| Motion Graphics      | OpenAI Chat Model4   | Animated graphics, video effects, explainer animations               |
| Web Designer         | OpenAI Chat Model5   | Website design, landing pages, responsive layouts                    |
| Creative Copywriter  | OpenAI Chat Model6   | Creative content writing, advertising copy, brand messaging          |

---

#### 2.4 User Guidance and Documentation

- **Overview:**  
  Provides embedded documentation, workflow description, agent roles, use cases, and contact information through sticky notes.

- **Nodes Involved:**  
  - Sticky Note Header  
  - Sticky Note Main

- **Node Details:**  
  - **Sticky Note Header**  
    - Type: Sticky Note  
    - Role: Displays header information including workflow title, contact, and social media links.  
    - Content: Title, contact email (Yaron@nofluff.online), YouTube and LinkedIn links.  
    - Visual: Sized 580x320, colored (color 5).  
    - Position: Top left area of workflow canvas.

  - **Sticky Note Main**  
    - Type: Sticky Note  
    - Role: Comprehensive documentation and overview of entire workflow, use cases, agent roles, cost optimization, and hashtags for categorization.  
    - Content: Detailed markdown-formatted text explaining workflow purpose and structure.  
    - Visual: Large (740x2500), colored (color 5).  
    - Position: Left side of canvas near input node.

- These notes are non-executable but essential for user understanding and maintenance.

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                         | Input Node(s)               | Output Node(s)                           | Sticky Note                                                                                                     |
|---------------------------|---------------------------------|---------------------------------------|-----------------------------|------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| When chat message received| LangChain Chat Trigger           | Entry point for chat request          | External webhook             | Creative Director Agent                   | See Sticky Note Main for overview and usage details                                                             |
| Creative Director Agent    | LangChain Agent                  | Strategic decision-making & delegation| When chat message received  | Think, Graphic Designer, UI/UX Designer, Brand Strategist, Motion Graphics Designer, Web Designer, Creative Copywriter | See Sticky Note Main for overview and usage details                                                             |
| Think                     | LangChain Tool Think             | Internal reasoning tool for Director  | Creative Director Agent      | Creative Director Agent                   |                                                                                                                |
| Graphic Designer          | LangChain Agent Tool             | Visual identity & graphic design      | Creative Director Agent      | None (end output)                         |                                                                                                                |
| UI/UX Designer            | LangChain Agent Tool             | User interface & experience design    | Creative Director Agent      | None (end output)                         |                                                                                                                |
| Brand Strategist          | LangChain Agent Tool             | Brand positioning & strategy          | Creative Director Agent      | None (end output)                         |                                                                                                                |
| Motion Graphics Designer  | LangChain Agent Tool             | Motion graphics & animation            | Creative Director Agent      | None (end output)                         |                                                                                                                |
| Web Designer             | LangChain Agent Tool             | Website & responsive design           | Creative Director Agent      | None (end output)                         |                                                                                                                |
| Creative Copywriter       | LangChain Agent Tool             | Creative writing & marketing copy      | Creative Director Agent      | None (end output)                         |                                                                                                                |
| OpenAI Chat Model Director| LangChain OpenAI Chat Model     | Language model for Creative Director   | None                        | Creative Director Agent                   | Requires OpenAI API credentials                                                                                 |
| OpenAI Chat Model1        | LangChain OpenAI Chat Model     | Language model for Graphic Designer    | None                        | Graphic Designer                          | Requires OpenAI API credentials                                                                                 |
| OpenAI Chat Model2        | LangChain OpenAI Chat Model     | Language model for UI/UX Designer      | None                        | UI/UX Designer                            | Requires OpenAI API credentials                                                                                 |
| OpenAI Chat Model3        | LangChain OpenAI Chat Model     | Language model for Brand Strategist    | None                        | Brand Strategist                          | Requires OpenAI API credentials                                                                                 |
| OpenAI Chat Model4        | LangChain OpenAI Chat Model     | Language model for Motion Graphics     | None                        | Motion Graphics Designer                  | Requires OpenAI API credentials                                                                                 |
| OpenAI Chat Model5        | LangChain OpenAI Chat Model     | Language model for Web Designer        | None                        | Web Designer                             | Requires OpenAI API credentials                                                                                 |
| OpenAI Chat Model6        | LangChain OpenAI Chat Model     | Language model for Creative Copywriter | None                        | Creative Copywriter                       | Requires OpenAI API credentials                                                                                 |
| Sticky Note Header        | Sticky Note                     | Documentation & contact info           | None                        | None                                     | Contains workflow title, contact email, YouTube & LinkedIn links                                                |
| Sticky Note Main          | Sticky Note                     | Comprehensive workflow documentation   | None                        | None                                     | Detailed overview, agent roles, use cases, cost optimization, and hashtags                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named `"Creative Director Agent with Creative Design Team"`.

2. **Create the entry point node:**
   - Add a **LangChain Chat Trigger** node.
   - Name: `When chat message received`.
   - Set Webhook ID to `creative-webhook-id` (or generate a new one).
   - Leave options default.

3. **Add the Creative Director Agent node:**
   - Add a **LangChain Agent** node.
   - Name: `Creative Director Agent`.
   - Connect the output of `When chat message received` to this node.
   - Use default options.
   - This node will orchestrate delegation.

4. **Add the Think tool node:**
   - Add a **LangChain Tool Think** node.
   - Name: `Think`.
   - Connect its input to `Creative Director Agent`.
   - No parameters required.
   - Connect output back to `Creative Director Agent`.

5. **Add OpenAI Chat Model Director node:**
   - Add a **LangChain OpenAI Chat Model** node.
   - Name: `OpenAI Chat Model Director`.
   - Configure the model to `o3`.
   - Set OpenAI API credentials (create or select your OpenAI API credential).
   - Connect output to `Creative Director Agent` as the language model backend.

6. **Create six pairs of Specialist Agents and their OpenAI Chat Models:**

   For each pair:

   - Add a **LangChain Agent Tool** node.
     - Name accordingly: `Graphic Designer`, `UI/UX Designer`, `Brand Strategist`, `Motion Graphics Designer`, `Web Designer`, `Creative Copywriter`.
     - Set the parameter `text` to:  
       `={{ $fromAI('Prompt__User_Message_', '', 'string') }}` to dynamically receive the prompt from the Creative Director.
     - Write a clear `toolDescription` reflecting their specialty, e.g.:  
       *Graphic Designer*: "call this AI Agent that specializes in visual identity creation, logo design, print materials, packaging design, and brand visual elements".
     - Connect input from `Creative Director Agent`.
   
   - Add a **LangChain OpenAI Chat Model** node.
     - Name accordingly: `OpenAI Chat Model1`, ..., `OpenAI Chat Model6`.
     - Configure the model to `gpt-4.1-mini`.
     - Set OpenAI API credentials.
     - Connect output to corresponding Specialist Agent node as language model backend.

   - Connect each Specialist Agent node output as final design output (no further nodes).

7. **Connect outputs:**
   - From the Creative Director Agent node, connect output edges to all six Specialist Agent nodes *and* the Think node.

8. **Add Sticky Notes for documentation:**
   - Add a **Sticky Note** node named `Sticky Note Header`.
     - Set color to 5, size approx. 580x320.
     - Paste header content with workflow title, contact email (`Yaron@nofluff.online`), YouTube, and LinkedIn links.
     - Position on top left for visibility.

   - Add a **Sticky Note** node named `Sticky Note Main`.
     - Set color to 5, size approx. 740x2500.
     - Paste detailed markdown content describing workflow overview, agent roles, use cases, and cost optimization.
     - Position left side near input node.

9. **Credentials Configuration:**
   - Ensure **OpenAI API Credentials** are created and linked for all OpenAI Chat Model nodes.
   - No other credentials needed.

10. **Testing:**
    - Deploy webhook.
    - Send a chat message input such as "Create brand identity for new tech startup".
    - Observe workflow activation, Creative Director analysis, delegation, and generation of specialized creative outputs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Contact for questions or support: Yaron@nofluff.online                                                                                    | Provided in Sticky Note Header                                                                     |
| Explore more tips and videos on YouTube: https://www.youtube.com/@YaronBeen/videos                                                        | Linked in Sticky Note Header                                                                       |
| Professional profile and updates: https://www.linkedin.com/in/yaronbeen/                                                                  | Linked in Sticky Note Header                                                                       |
| Workflow simulates multi-agent AI creative design agency with O3 for director and GPT-4.1-mini for specialists                           | See Sticky Note Main for detailed workflow overview                                               |
| Cost optimization via strategic use of different OpenAI models and parallel processing                                                    | See Sticky Note Main                                                                               |
| Use cases include brand identity creation, digital product design, marketing campaigns, web projects, and content creation                | See Sticky Note Main                                                                               |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated n8n workflow created using n8n, adhering strictly to current content policies with no illegal, offensive, or protected elements. All data processed is legal and public.

---