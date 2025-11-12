Generate Viral Video Ideas from Social Media Trends with GPT-4o and Discord

https://n8nworkflows.xyz/workflows/generate-viral-video-ideas-from-social-media-trends-with-gpt-4o-and-discord-7523


# Generate Viral Video Ideas from Social Media Trends with GPT-4o and Discord

### 1. Workflow Overview

This workflow automates the generation and posting of viral short-form video ideas to a Discord channel by leveraging current social media trends. It is designed for content creators, social media managers, and trend analysts who want to generate engaging video concepts based on trending topics from various platforms like TikTok, Instagram Reels, and YouTube Shorts.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Triggers:** Multiple schedule triggers activate the workflow at different hours to fetch fresh trend data regularly. Additionally, an external workflow trigger allows manual or on-demand execution.
- **1.2 Trend Data Retrieval:** An HTTP request fetches social media trend data from an external source (social-searcher.com).
- **1.3 Data Formatting:** The raw trend data is transformed from HTML into a Markdown format suitable for AI processing.
- **1.4 AI Processing:** An AI Agent powered by OpenAI's GPT-4o-mini model analyzes the trend data and generates detailed, structured video ideas optimized for viral short-form content.
- **1.5 Structured Output Parsing:** The AI's output is parsed into defined JSON parts for segmented posting.
- **1.6 Discord Posting:** The workflow posts the generated video ideas in three sequential messages to a specified Discord channel, corresponding to the parsed parts of the AI output.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Triggers

**Overview:**  
This block initiates the workflow either on a scheduled basis at multiple preset hours or via an external workflow trigger. This ensures trend analysis runs frequently and can be manually controlled.

**Nodes Involved:**  
- Schedule Trigger  
- Schedule Trigger1  
- Schedule Trigger2  
- Schedule Trigger3  
- Schedule Trigger4  
- Schedule Trigger5  
- Schedule Trigger6  
- When Executed by Another Workflow

**Node Details:**

- **Schedule Trigger, Schedule Trigger1, Schedule Trigger2, Schedule Trigger3, Schedule Trigger4, Schedule Trigger6**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiate workflow execution at specified hours (01:00, 02:00, 03:00, 04:00, 01:00, 05:00 respectively)  
  - *Configuration:* Each trigger set with a specific trigger hour to fetch fresh data multiple times a day.  
  - *Input/Output:* No input; output connects to HTTP Request node.  
  - *Edge Cases:* Timezone misconfiguration can cause unexpected trigger times. If multiple triggers fire simultaneously, workflow load may increase.  
  - *Version:* n8n v1.2 or higher recommended for schedule trigger stability.

- **Schedule Trigger5**  
  - *Type:* Schedule Trigger  
  - *Role:* Runs with default interval (every minute or as per default) - likely a placeholder or fallback trigger.  
  - *Edge Cases:* Without a specified trigger time, could cause unintended frequent executions or overload.

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Allows this workflow to be triggered externally by another workflow or manual execution.  
  - *Input/Output:* Passthrough input to HTTP Request node.  
  - *Edge Cases:* If upstream workflow fails or does not pass expected input, this node may not trigger properly.

---

#### 2.2 Trend Data Retrieval

**Overview:**  
Fetches current social media trend data from an external web resource for processing.

**Nodes Involved:**  
- HTTP Request

**Node Details:**

- **HTTP Request**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves social media trend data by sending a GET request to `https://www.social-searcher.com/social-trends/?q7=ai`  
  - *Configuration:* Default GET method; no authentication or headers specified.  
  - *Input:* Connected from any trigger node above.  
  - *Output:* Raw HTML or JSON data in response, passed to Markdown node.  
  - *Edge Cases:*  
    - Endpoint downtime or HTTP errors (timeouts, 4xx, 5xx) lead to workflow failure or empty data.  
    - Changes in website structure may break parsing downstream.  
    - No retry mechanism present.  
  - *Version:* HTTP Request v4.2 or higher recommended for stable connection.

---

#### 2.3 Data Formatting

**Overview:**  
Converts raw trend data from the HTTP response into Markdown format for easier AI consumption.

**Nodes Involved:**  
- Markdown

**Node Details:**

- **Markdown**  
  - *Type:* Markdown  
  - *Role:* Converts HTML content from HTTP response to Markdown text in the workflow’s JSON data structure.  
  - *Configuration:* Uses expression `={{ $json.data }}` to extract content from HTTP Request node output.  
  - *Input/Output:* Input from HTTP Request; output goes to AI Agent node.  
  - *Edge Cases:* If input data is malformed or empty, output will be invalid or empty, potentially causing AI prompt issues.

---

#### 2.4 AI Processing

**Overview:**  
Processes the Markdown-formatted trend data using an AI agent configured with a custom system message prompt to generate viral video ideas in three structured parts.

**Nodes Involved:**  
- AI Agent2  
- OpenAI Chat Model2  
- Structured Output Parser1

**Node Details:**

- **AI Agent2**  
  - *Type:* LangChain AI Agent  
  - *Role:* Core AI processing node that sends trend data to the OpenAI model and receives generated video ideas.  
  - *Configuration:*  
    - Uses the expression `={{ $json.data }}` to pass Markdown trend data as input text.  
    - Custom system message defines the AI persona as "TrendSage," focusing on generating detailed, viral video ideas optimized for TikTok, Instagram Reels, and YouTube Shorts.  
    - Instructed to split output into three parts below 1500 characters each for Discord message size limits.  
  - *Input/Output:* Input from Markdown node; outputs parsed JSON parts to Discord nodes.  
  - *Edge Cases:*  
    - If OpenAI API key is invalid or quota exceeded, node fails.  
    - Responses exceeding length or malformed JSON cause parsing errors.  
    - Model selection `gpt-4o-mini` requires appropriate API access and may have rate limits.  
  - *Sub-workflow:* Uses OpenAI Chat Model and Structured Output Parser as internal components.

- **OpenAI Chat Model2**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Provides the language model backend (gpt-4o-mini) for the AI Agent.  
  - *Configuration:* Model set to `gpt-4o-mini`. Requires OpenAI API credentials.  
  - *Input/Output:* Connected internally to AI Agent2.  
  - *Edge Cases:* API key issues, rate limiting, model availability.

- **Structured Output Parser1**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses the AI Agent's raw output into a strict JSON schema with three parts (`part_1`, `part_2`, `part_3`) for segmented posting.  
  - *Configuration:* Uses a JSON schema example to validate and structure AI output.  
  - *Input/Output:* Connected internally from AI Agent2. Ensures downstream nodes receive structured JSON.  
  - *Edge Cases:* Parsing failure if AI output is malformed or deviates from schema.

---

#### 2.5 Discord Posting

**Overview:**  
Posts each of the three generated parts of video ideas sequentially as messages to a dedicated Discord channel.

**Nodes Involved:**  
- Discord6  
- Discord  
- Discord1

**Node Details:**

- **Discord6**  
  - *Type:* Discord node  
  - *Role:* Posts `part_1` of the AI output to the Discord channel.  
  - *Configuration:*  
    - Uses webhook authentication.  
    - Target guild ID: `1236784625196601386` (YungCEO SOCIETY💰 Discord server).  
    - Target channel ID: `1332673633965051914` (`📈│trend-tracker` channel).  
    - Content expression: `={{ $('AI Agent2').item.json.output.part_1 }}` to post the first part.  
  - *Input/Output:* Input from AI Agent2 node’s output parser. Output connects to next Discord node.  
  - *Edge Cases:*  
    - Discord API errors (rate limiting, invalid webhook, permission denied).  
    - Message length exceeding Discord limits (1500 char limit is respected upstream).  

- **Discord**  
  - *Type:* Discord node  
  - *Role:* Posts `part_2` of the AI output to the same Discord channel.  
  - *Configuration:* Same as Discord6 but posts `part_2`.  
  - *Input/Output:* Input from Discord6 node, output to Discord1.  
  - *Edge Cases:* Same as Discord6.

- **Discord1**  
  - *Type:* Discord node  
  - *Role:* Posts `part_3` of the AI output, completing the series of video idea posts.  
  - *Configuration:* Same channel and guild, content from `part_3`.  
  - *Input:* From Discord node.  
  - *Edge Cases:* Same as above.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                        | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                                                                        |
|---------------------------|--------------------------------|-------------------------------------|------------------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger                | Scheduled workflow start at 01:00   | None                         | HTTP Request               |                                                                                                                                                    |
| Schedule Trigger1          | Schedule Trigger                | Scheduled workflow start at 02:00   | None                         | HTTP Request               |                                                                                                                                                    |
| Schedule Trigger2          | Schedule Trigger                | Scheduled workflow start at 03:00   | None                         | HTTP Request               |                                                                                                                                                    |
| Schedule Trigger3          | Schedule Trigger                | Scheduled workflow start at 04:00   | None                         | HTTP Request               |                                                                                                                                                    |
| Schedule Trigger4          | Schedule Trigger                | Scheduled workflow start at 01:00   | None                         | HTTP Request               |                                                                                                                                                    |
| Schedule Trigger5          | Schedule Trigger                | Default interval trigger (unspecified) | None                      | HTTP Request               |                                                                                                                                                    |
| Schedule Trigger6          | Schedule Trigger                | Scheduled workflow start at 05:00   | None                         | HTTP Request               |                                                                                                                                                    |
| When Executed by Another Workflow | Execute Workflow Trigger  | External trigger for manual execution | None                         | HTTP Request               | Acts as a trigger allowing external control over when trend analysis begins.                                                                       |
| HTTP Request              | HTTP Request                    | Fetches social media trends data    | Schedule Trigger*, When Executed by Another Workflow | Markdown            | Fetches from social-searcher.com. Multiple schedule triggers feed into this node.                                                                  |
| Markdown                  | Markdown                       | Converts raw HTML data to Markdown  | HTTP Request                 | AI Agent2                  | Converts HTML trend data for AI processing.                                                                                                       |
| AI Agent2                 | LangChain AI Agent             | Generates viral video ideas via GPT-4o-mini | Markdown                    | Discord6                   | Uses GPT-4o-mini with custom prompt to create viral video ideas split into three parts.                                                           |
| OpenAI Chat Model2        | LangChain OpenAI Chat Model   | Provides GPT-4o-mini model for AI Agent | AI Agent2 (ai_languageModel) | AI Agent2                  | Requires OpenAI API key.                                                                                                                           |
| Structured Output Parser1 | LangChain Structured Output Parser | Parses AI output into structured JSON | AI Agent2 (ai_outputParser)  | AI Agent2                  | Parses output into three parts for segmented Discord posting.                                                                                      |
| Discord6                  | Discord                        | Posts first part of video ideas     | AI Agent2                   | Discord                    | Posts part 1 to Discord channel `📈│trend-tracker`.                                                                                                |
| Discord                   | Discord                        | Posts second part of video ideas    | Discord6                    | Discord1                   | Posts part 2 to Discord channel.                                                                                                                  |
| Discord1                  | Discord                        | Posts third part of video ideas     | Discord                     | None                       | Posts part 3 to Discord channel.                                                                                                                  |
| Workflow Summary          | Sticky Note                   | Workflow overview documentation     | None                         | None                       | Describes overall workflow purpose and logic.                                                                                                     |
| Setup Note 1              | Sticky Note                   | Explains external workflow trigger  | None                         | None                       | Describes external trigger node role.                                                                                                            |
| Setup Note 2              | Sticky Note                   | Explains HTTP Request and schedule triggers | None                     | None                       | Notes HTTP Request gets trends from social-searcher.com; multiple triggers schedule runs.                                                        |
| Setup Note 3              | Sticky Note                   | Explains AI Agent and prompt        | None                         | None                       | Details AI Agent configuration for generating viral video ideas.                                                                                  |
| Setup Note 4              | Sticky Note                   | Explains Discord posting nodes      | None                         | None                       | Describes posting of AI outputs in three parts to Discord.                                                                                        |
| Setup Note 5              | Sticky Note                   | Explains OpenAI Chat Model node     | None                         | None                       | Notes use of OpenAI GPT-4o-mini requiring API key.                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add multiple **Schedule Trigger** nodes, each set to trigger at specific hours (01:00, 02:00, 03:00, 04:00, 05:00).  
     - In each node’s parameters, set rule interval to trigger at the desired hour.  
   - Add an **Execute Workflow Trigger** node configured to accept passthrough input, allowing external workflow triggering.

2. **Add HTTP Request Node:**
   - Create an **HTTP Request** node connected from all trigger nodes.  
   - Configure it to perform a GET request to `https://www.social-searcher.com/social-trends/?q7=ai`.  
   - No authentication needed. Default options suffice.

3. **Add Markdown Node:**
   - Connect the HTTP Request node’s output to a **Markdown** node.  
   - Set its HTML parameter to `={{ $json.data }}` to convert the raw HTML response into markdown text.

4. **Configure AI Agent:**
   - Add an **AI Agent** node (LangChain AI Agent).  
   - Connect the Markdown node output to the AI Agent input.  
   - In AI Agent parameters, set the input text to `={{ $json.data }}` to pass Markdown content.  
   - In options, set the system message prompt to the provided custom message defining the "TrendSage" persona with instructions for generating viral video ideas in three parts under 1500 characters each.  
   - Configure the AI Agent to use:  
     - A **LangChain OpenAI Chat Model** node set to `gpt-4o-mini` as the language model backend.  
     - A **Structured Output Parser** node with the JSON schema example that requires output parts `part_1`, `part_2`, and `part_3`.

   - Connect the OpenAI Chat Model and Structured Output Parser nodes internally to the AI Agent as per LangChain agent design.

5. **Discord Posting Nodes:**
   - Add three **Discord** nodes for posting messages, named here as Discord6, Discord, and Discord1 for clarity.  
   - Configure each with the same:  
     - Discord webhook credentials (ensure webhook ID and permissions are correctly set).  
     - Guild ID: `1236784625196601386` (YungCEO SOCIETY💰 server).  
     - Channel ID: `1332673633965051914` (`📈│trend-tracker` channel).  
   - Set message content expressions as follows:  
     - Discord6: `={{ $('AI Agent2').item.json.output.part_1 }}`  
     - Discord: `={{ $('AI Agent2').item.json.output.part_2 }}`  
     - Discord1: `={{ $('AI Agent2').item.json.output.part_3 }}`
   - Connect the nodes sequentially: AI Agent → Discord6 → Discord → Discord1.

6. **Credential Setup:**
   - Create and assign OpenAI API credentials with access to GPT-4o-mini model.  
   - Create and assign Discord webhook credentials with permission to post messages in the specified guild and channel.

7. **Optional: Add Sticky Notes:**
   - Add notes explaining workflow overview, trigger role, HTTP request purpose, AI agent configuration, and Discord posting for documentation clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The AI Agent uses a custom prompt to generate viral video ideas optimized for short-form videos on TikTok, Instagram Reels, and YouTube Shorts. | Prompt embedded in AI Agent node parameters.                                                     |
| The workflow posts the AI-generated content in three parts to avoid Discord message length limits. | Discord message character limit considerations (~2000 characters max, 1500 reserved here).      |
| The trend data source is social-searcher.com, which may change HTML structure, requiring maintenance. | URL: https://www.social-searcher.com/social-trends/?q7=ai                                       |
| OpenAI GPT-4o-mini is a specialized model and requires appropriate API access and credentials.   | OpenAI docs: https://platform.openai.com/docs/models/gpt-4o-mini                                |
| Discord webhooks must have permissions to post messages in the target channel/guild.             | Discord Developer Portal: https://discord.com/developers/applications                            |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, respecting all content policies. It does not contain any illegal, offensive, or protected elements. All handled data is legal and publicly accessible.