Daily AI News Digest with Perplexity Pro, GPT Format & Gmail Delivery

https://n8nworkflows.xyz/workflows/daily-ai-news-digest-with-perplexity-pro--gpt-format---gmail-delivery-5469


# Daily AI News Digest with Perplexity Pro, GPT Format & Gmail Delivery

### 1. Workflow Overview

This workflow automates the daily collection, formatting, and email delivery of the latest artificial intelligence news headlines. It is designed to run every morning at 9 AM, fetching AI news published within the last 24 hours using Perplexity's Sonar Pro model, formatting the results into a clean and readable email via an OpenAI GPT model, and finally sending the digest via Gmail.

The workflow is logically divided into four main blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow daily at a set time.
- **1.2 AI Research Agent**: Queries Perplexity’s Sonar Pro to retrieve the most recent AI news headlines and summaries.
- **1.3 Formatter AI Agent with Memory**: Uses the OpenAI GPT-4.1-mini model to format the raw news data into a polished email-ready digest, optionally utilizing session memory for contextual continuity.
- **1.4 Gmail Delivery**: Sends the formatted digest as a plain text email to the configured recipient(s).

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow execution once every day at 9 AM.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - *Type & Role:* n8n built-in schedule trigger node that initiates workflows at defined intervals.  
    - *Configuration:* Set to trigger daily at 9:00 AM (hour 9, minute 0 by default).  
    - *Expressions/Variables:* None.  
    - *Connections:* Outputs to the Research Agent node.  
    - *Version Requirements:* n8n version supporting scheduleTrigger v1.2 or higher.  
    - *Potential Failures:* Scheduling misconfiguration, time zone mismatches.  
    - *Sticky Note:* “Scheduled Trigger” — visually groups this node.

#### 1.2 AI Research Agent

- **Overview:**  
  This block uses Perplexity’s Sonar Pro AI agent to search for AI news headlines published in the last 24 hours. It retrieves headlines, summaries, source information, URLs, and publication dates.

- **Nodes Involved:**  
  - Research Agent

- **Node Details:**  
  - **Research Agent**  
    - *Type & Role:* Perplexity node configured to use the “sonar-pro” model, specialized for AI news research.  
    - *Configuration:*  
      - Model: sonar-pro  
      - Search Recency: day (last 24 hours)  
      - System prompt defines the agent as an AI news researcher tasked with retrieving headlines, summaries, full URLs, dates, and sources.  
      - User prompt asks for latest AI development headlines, including model launches and market news.  
    - *Expressions/Variables:* None beyond fixed prompt content.  
    - *Connections:* Outputs main data to the Formatter AI Agent.  
    - *Credentials:* Requires valid Perplexity API credentials.  
    - *Version Requirements:* Perplexity node v1 or higher.  
    - *Potential Failures:* API authentication errors, rate limits, no news found, network timeouts.  
    - *Sticky Note:* “Perplexity Research Agent” — visually groups this node.

#### 1.3 Formatter AI Agent with Memory

- **Overview:**  
  This block formats the raw news data from Perplexity into a clean, readable email digest using an OpenAI GPT-4.1-mini model. It employs a memory buffer to provide session continuity and context for improved formatting.

- **Nodes Involved:**  
  - Formatter AI Agent  
  - OpenAI Chat Model  
  - Simple Memory

- **Node Details:**  
  - **Formatter AI Agent**  
    - *Type & Role:* LangChain Agent node specialized for text formatting tasks.  
    - *Configuration:*  
      - Input text: the content of the first choice message from Perplexity output (`{{$json.choices[0].message.content}}`).  
      - System message: instructs the agent to act as a helpful formatter assisting in producing an email-readable format of AI headlines and news.  
      - Prompt type: "define" (custom prompt definition).  
    - *Connections:*  
      - Receives data from Research Agent (main) and OpenAI Chat Model (ai_languageModel).  
      - Sends formatted output to Gmail node.  
      - Uses Simple Memory (ai_memory) for session context based on the Research Agent’s item ID.  
    - *Credentials:* OpenAI API credentials for GPT-4.1-mini model access.  
    - *Potential Failures:* API errors, prompt parsing errors, memory session key missing, formatting issues if input is malformed.  
    - *Sticky Note:* “Formatter Agent” — groups related nodes visually.  

  - **OpenAI Chat Model**  
    - *Type & Role:* Language model node invoking OpenAI GPT-4.1-mini.  
    - *Configuration:*  
      - Model: gpt-4.1-mini  
      - No additional options set.  
    - *Connections:* Linked as language model used by Formatter AI Agent.  
    - *Potential Failures:* API key invalid/expired, rate limits, model unavailability.  

  - **Simple Memory**  
    - *Type & Role:* Buffer window memory node to maintain conversational context.  
    - *Configuration:*  
      - Session key set dynamically using Research Agent’s item JSON id for session identification.  
      - Session ID type: custom key.  
    - *Connections:* Feeds memory context into Formatter AI Agent.  
    - *Potential Failures:* Missing or invalid session keys, memory storage limits.

#### 1.4 Gmail Delivery

- **Overview:**  
  Sends the formatted AI news digest to designated email recipients as a plain text email.

- **Nodes Involved:**  
  - Send a message (Gmail)

- **Node Details:**  
  - **Send a message**  
    - *Type & Role:* Gmail node used to send emails via OAuth2 credentials.  
    - *Configuration:*  
      - Recipient email address: initially empty, must be set to desired recipient(s).  
      - Email subject: "AI News Update in the last 24 hours".  
      - Email body: bound dynamically to the formatted output (`{{$json.output}}`) from Formatter AI Agent.  
      - Email type: plain text.  
    - *Credentials:* Requires Gmail OAuth2 credentials configured in n8n.  
    - *Connections:* Receives input from Formatter AI Agent.  
    - *Potential Failures:* Authentication errors, quota exceeded, invalid recipient address, network issues.  
    - *Sticky Note:* “Gmail Output Node” — visually groups this node.

---

### 3. Summary Table

| Node Name          | Node Type                          | Functional Role                    | Input Node(s)    | Output Node(s)       | Sticky Note                        |
|--------------------|----------------------------------|----------------------------------|------------------|----------------------|----------------------------------|
| Schedule Trigger    | n8n-nodes-base.scheduleTrigger   | Starts workflow daily at 9 AM    | —                | Research Agent       | Scheduled Trigger                 |
| Research Agent      | n8n-nodes-base.perplexity        | Fetches AI news headlines        | Schedule Trigger | Formatter AI Agent   | Perplexity Research Agent        |
| Formatter AI Agent  | @n8n/n8n-nodes-langchain.agent   | Formats news for email delivery  | Research Agent, OpenAI Chat Model, Simple Memory | Send a message | Formatter Agent                  |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4.1-mini model for formatting | —                | Formatter AI Agent   | Formatter Agent                  |
| Simple Memory       | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains context for formatter | —                | Formatter AI Agent   | Formatter Agent                  |
| Send a message      | n8n-nodes-base.gmail             | Sends final email digest         | Formatter AI Agent | —                    | Gmail Output Node                |
| Sticky Note         | n8n-nodes-base.stickyNote        | Visual grouping & documentation  | —                | —                    | Scheduled Trigger                |
| Sticky Note1        | n8n-nodes-base.stickyNote        | Visual grouping & documentation  | —                | —                    | Perplexity Research Agent       |
| Sticky Note2        | n8n-nodes-base.stickyNote        | Visual grouping & documentation  | —                | —                    | Formatter Agent                 |
| Sticky Note3        | n8n-nodes-base.stickyNote        | Visual grouping & documentation  | —                | —                    | Gmail Output Node               |
| Sticky Note4        | n8n-nodes-base.stickyNote        | Workflow overview & tutorial link | —                | —                    | See detailed workflow description below |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Scheduled Trigger Node**  
   - Type: Schedule Trigger  
   - Set trigger interval to daily at 9:00 AM (hour: 9, minute: 0)  
   - Position on canvas: Left-most start node

2. **Add Perplexity Research Agent Node**  
   - Type: Perplexity (n8n-nodes-base.perplexity)  
   - Credentials: Connect your Perplexity API credentials  
   - Parameters:  
     - Model: sonar-pro  
     - Options: searchRecency = "day"  
     - Messages:  
       - System role message: Define as AI news researcher with instructions to fetch headlines, summaries, full URLs, dates, and sources from last 24 hours  
       - User message: Request latest AI development headlines including model launches and market news  
   - Connect Schedule Trigger’s main output to this node’s main input

3. **Set Up the Formatter AI Agent Node**  
   - Type: LangChain Agent (@n8n/n8n-nodes-langchain.agent)  
   - Credentials: OpenAI API key configured in n8n  
   - Parameters:  
     - Text input: Use expression to map Perplexity output: `{{$json.choices[0].message.content}}`  
     - System message: “You are a helpful formatter assistant, you will receive a chunk of AI Headlines and news, your job is to format it such that it is in an easily readable format for email.”  
     - Prompt type: define  
   - Connect Research Agent’s main output to Formatter AI Agent’s main input

4. **Add OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model (@n8n/n8n-nodes-langchain.lmChatOpenAi)  
   - Credentials: Use same OpenAI API credentials  
   - Parameters:  
     - Model: gpt-4.1-mini  
   - Connect this node as the ai_languageModel input to Formatter AI Agent

5. **Add Simple Memory Buffer Node**  
   - Type: LangChain Memory Buffer Window (@n8n/n8n-nodes-langchain.memoryBufferWindow)  
   - Parameters:  
     - Session Key: Use expression referencing Research Agent’s item ID: `={{ $('Research Agent').item.json.id }}`  
     - Session ID Type: customKey  
   - Connect as ai_memory input to Formatter AI Agent

6. **Add Gmail Send Node**  
   - Type: Gmail (n8n-nodes-base.gmail)  
   - Credentials: Configure Gmail OAuth2 credentials  
   - Parameters:  
     - Send To: specify recipient email(s)  
     - Subject: “AI News Update in the last 24 hours”  
     - Message: Bind to Formatter AI Agent output: `={{ $json.output }}`  
     - Email Type: text (plain text)  
   - Connect Formatter AI Agent’s main output to Gmail node’s main input

7. **Optional - Add Sticky Notes for Clarity**  
   - Add sticky notes near each logical block with descriptive text such as:  
     - “Scheduled Trigger” near Schedule Trigger node  
     - “Perplexity Research Agent” near Research Agent node  
     - “Formatter Agent” grouping Formatter AI Agent, OpenAI Chat Model, and Simple Memory nodes  
     - “Gmail Output Node” near Gmail node  
     - Add a large sticky note describing workflow purpose, usage instructions, and links (e.g., video tutorial).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                         | Context or Link                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| 🧠 AI News Update Every 24 Hours (with Perplexity + GPT Formatter) This workflow automatically delivers a clean, AI-curated summary of the latest AI news headlines from the past 24 hours directly to your inbox — every morning at 9 AM. For step-by-step video tutorial for this build, watch here: https://youtu.be/O-DLvaMVLso | Workflow overview and video tutorial link (Sticky Note) |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow created using n8n, a workflow automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.