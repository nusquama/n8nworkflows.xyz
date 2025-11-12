Automate Reddit Trend Analysis with GPT-4 and Slack/Gmail Distribution

https://n8nworkflows.xyz/workflows/automate-reddit-trend-analysis-with-gpt-4-and-slack-gmail-distribution-4373


# Automate Reddit Trend Analysis with GPT-4 and Slack/Gmail Distribution

---

### 1. Workflow Overview

This workflow automates the process of monitoring Reddit discussions within a specified subreddit, extracting insightful summaries using AI, and distributing these insights to designated communication channels (Slack and Gmail). It is designed for daily execution, enabling teams to stay updated on trending topics and relevant conversations without manual effort.

The workflow is logically organized into three main functional blocks:

- **1.1 Input Reception**: Automatically trigger the workflow daily and fetch relevant Reddit posts based on keyword and subreddit filters.
- **1.2 AI Processing**: Use AI models (Langchain Agent and OpenAI GPT-4) to analyze, summarize, and generate insights from the fetched Reddit posts.
- **1.3 Output Distribution**: Send the AI-generated summaries to Slack and Gmail for team communication and archival.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
This block triggers the workflow once per day and fetches the latest Reddit posts from a specific subreddit using keyword search criteria. It ensures the workflow operates on fresh, targeted data.

- **Nodes Involved:**  
  - Trigger Daily Reddit Scan  
  - Fetch Reddit Posts

- **Node Details:**

  - **Trigger Daily Reddit Scan**  
    - *Type & Role:* Schedule Trigger — initiates workflow execution daily at a specified hour (9 AM).  
    - *Configuration:* Runs once per day at 9:00. No additional parameters.  
    - *Inputs:* None (starting node).  
    - *Outputs:* Triggers "Fetch Reddit Posts".  
    - *Edge Cases:* Scheduler misfire, timezone considerations, or node disabled could prevent workflow start.

  - **Fetch Reddit Posts**  
    - *Type & Role:* Reddit node — searches the "Chatgpt" subreddit for posts containing the keyword "why I stopped using it".  
    - *Configuration:*  
      - Operation: Search  
      - Subreddit: "Chatgpt"  
      - Keyword: "why I stopped using it"  
      - Limit: 5 posts  
    - *Inputs:* Trigger Daily Reddit Scan  
    - *Outputs:* Passes posts individually to the AI summarization node.  
    - *Edge Cases:* Reddit API rate limits, no posts found, network errors, or invalid subreddit/keyword inputs.

---

#### 1.2 AI Processing

- **Overview:**  
This block processes each fetched Reddit post through AI to extract a concise thematic classification and summary, then further generates insights with OpenAI’s GPT-4 model for enhanced analysis.

- **Nodes Involved:**  
  - Summarize Posts with AI  
  - Generate AI Insights

- **Node Details:**

  - **Summarize Posts with AI**  
    - *Type & Role:* Langchain AI Agent — analyzes and summarizes each post using a custom prompt.  
    - *Configuration:*  
      - Prompt requests classification of main topic in 3-5 words and a one-sentence summary.  
      - Input text includes post title and content (`title` and `selftext` fields from Reddit JSON).  
    - *Inputs:* Output from "Fetch Reddit Posts"  
    - *Outputs:* Summary data passed to distribution nodes.  
    - *Expressions:* Uses expressions to inject Reddit post data into prompt (`{{ $json["title"] }}`, `{{ $json.selftext }}`).  
    - *Edge Cases:* Empty or malformed post content, AI model timeouts, or API quota limits.

  - **Generate AI Insights**  
    - *Type & Role:* OpenAI GPT-4 chat model — supplements the summarization with additional AI-generated insights.  
    - *Configuration:*  
      - Model set to "gpt-4o-mini" (a GPT-4 variant).  
      - No additional prompt parameters provided (implied usage from Langchain node).  
    - *Inputs:* Connected as AI language model resource for the Langchain agent.  
    - *Outputs:* Feeds back into the summarization process.  
    - *Edge Cases:* API authentication failures, rate limits, or network issues.

---

#### 1.3 Output Distribution

- **Overview:**  
This block broadcasts the AI-generated summaries to team communication platforms, ensuring insights reach stakeholders promptly via Slack and email.

- **Nodes Involved:**  
  - Send Summary to Slack  
  - Send Summary to Gmail

- **Node Details:**

  - **Send Summary to Slack**  
    - *Type & Role:* Slack node — posts the summary message to a specific Slack channel.  
    - *Configuration:*  
      - Message text includes subreddit topic, AI summary output, and a link to the Reddit post’s media content.  
      - Channel specified by ID `"C08TTV0CC3E"`.  
      - Webhook ID configured for authentication.  
    - *Inputs:* Output from "Summarize Posts with AI"  
    - *Outputs:* None (terminal node).  
    - *Expressions:* References Reddit post fields for dynamic content.  
    - *Edge Cases:* Slack API rate limits, invalid channel ID, or authentication failures.

  - **Send Summary to Gmail**  
    - *Type & Role:* Gmail node — sends an email containing the summary to a predefined recipient.  
    - *Configuration:*  
      - Recipient: `uu941401@gmail.com`  
      - Subject includes subreddit name dynamically.  
      - Message body mirrors Slack message content.  
      - Webhook ID configured for authentication.  
    - *Inputs:* Output from "Summarize Posts with AI"  
    - *Outputs:* None (terminal node).  
    - *Expressions:* Dynamically injects Reddit post and summary data.  
    - *Edge Cases:* Gmail API limits, invalid credentials, or sending failures.

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role             | Input Node(s)            | Output Node(s)                   | Sticky Note                                                                                                                             |
|-------------------------|--------------------------------|----------------------------|--------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Trigger Daily Reddit Scan| Schedule Trigger                | Workflow trigger           |                          | Fetch Reddit Posts               | Step 1: Get the Post — entry point, daily scan trigger                                                                                  |
| Fetch Reddit Posts       | Reddit                         | Fetch Reddit posts         | Trigger Daily Reddit Scan | Summarize Posts with AI          | Step 1: Get the Post — Reddit search with filters                                                                                      |
| Summarize Posts with AI  | Langchain AI Agent             | Post summarization         | Fetch Reddit Posts        | Send Summary to Slack, Send Summary to Gmail | Step 2: Post Summarization — AI analysis with Langchain and OpenAI Chat Model                                                           |
| Generate AI Insights     | Langchain OpenAI Chat Model    | Generate AI insights       |                          | Summarize Posts with AI (as AI model) | Step 2: Post Summarization — OpenAI GPT-4 model for insight generation                                                                  |
| Send Summary to Slack    | Slack                         | Send summary to Slack      | Summarize Posts with AI   |                                 | Step 3: Send the Summary — Slack distribution                                                                                          |
| Send Summary to Gmail    | Gmail                         | Send summary via email     | Summarize Posts with AI   |                                 | Step 3: Send the Summary — Gmail distribution                                                                                          |
| Sticky Note              | Sticky Note                   | Documentation note         |                          |                                 | Step 1: Get the Post — detailed explanation of input and Reddit fetch                                                                   |
| Sticky Note1             | Sticky Note                   | Documentation note         |                          |                                 | Step 2: Post Summarization — explanation of AI summarization steps                                                                     |
| Sticky Note2             | Sticky Note                   | Documentation note         |                          |                                 | Step 3: Send the Summary — explanation of distribution steps                                                                           |
| Sticky Note3             | Sticky Note                   | Documentation note         |                          |                                 | Detailed workflow breakdown, setup, and usage guide                                                                                     |
| Sticky Note9             | Sticky Note                   | Workflow assistance        |                          |                                 | Contact and resources for support                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: "Trigger Daily Reddit Scan"  
   - Set to trigger daily at 9:00 AM (set `triggerAtHour` to 9).  
   - No credentials needed.

2. **Create Reddit Node**  
   - Type: Reddit  
   - Name: "Fetch Reddit Posts"  
   - Operation: Search  
   - Subreddit: "Chatgpt"  
   - Keyword: "why I stopped using it"  
   - Limit: 5 posts  
   - Connect input from "Trigger Daily Reddit Scan".  
   - Set up Reddit credentials (OAuth2 or API key as required).

3. **Create Langchain AI Agent Node**  
   - Type: Langchain Agent  
   - Name: "Summarize Posts with AI"  
   - Prompt Type: Define  
   - Text prompt:  
     ```
     Analyze the following Reddit post. Classify its main topic in 3-5 words, and write a 1-sentence summary.

     Title: {{ $json["title"] }}  
     Content: {{ $json.selftext }}
     ```  
   - Connect input from "Fetch Reddit Posts".  
   - No special credentials here; will use linked AI language model node.

4. **Create Langchain OpenAI Chat Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Name: "Generate AI Insights"  
   - Model: Select "gpt-4o-mini" or appropriate GPT-4 model  
   - Connect as AI language model resource to "Summarize Posts with AI" node.  
   - Configure OpenAI API credentials (OpenAI API key with GPT-4 access).

5. **Create Slack Node**  
   - Type: Slack  
   - Name: "Send Summary to Slack"  
   - Channel: Select channel ID "C08TTV0CC3E" or your team’s channel.  
   - Message Text:  
     ```
     Topic:
     {{ $('Fetch Reddit Posts').item.json.subreddit }}

     Summary:
     {{ $json.output }}

     View on Reddit:
     {{ $('Fetch Reddit Posts').item.json.media_metadata.cnn80k7excte1.p[0].u }}
     ```  
   - Connect input from "Summarize Posts with AI".  
   - Configure Slack OAuth2 credentials for your workspace.

6. **Create Gmail Node**  
   - Type: Gmail  
   - Name: "Send Summary to Gmail"  
   - To: "uu941401@gmail.com" or desired email recipient.  
   - Subject: `New Reddit Post on {{ $('Fetch Reddit Posts').item.json.subreddit }}`  
   - Message:  
     ```
     Topic:
     {{ $('Fetch Reddit Posts').item.json.subreddit }}

     Summary:
     {{ $json.output }}

     View on Reddit:
     {{ $('Fetch Reddit Posts').item.json.media_metadata.cnn80k7excte1.p[0].u }}
     ```  
   - Connect input from "Summarize Posts with AI".  
   - Configure Gmail OAuth2 credentials.

7. **Connect Workflow**  
   - Connect nodes in this order:  
     Schedule Trigger → Fetch Reddit Posts → Summarize Posts with AI → (Send Summary to Slack & Send Summary to Gmail)  
   - Configure error handling and retry policies as needed.

8. **Test Workflow**  
   - Run manually or wait for scheduled trigger.  
   - Verify Reddit posts are fetched, summarized, and summaries are posted to Slack and sent via Gmail.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                              |
|--------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| For workflow assistance or questions, contact Yaron at: Yaron@nofluff.online                                 | Contact information                                                                            |
| Explore more tips and tutorials by Yaron Been on YouTube and LinkedIn                                        | YouTube: https://www.youtube.com/@YaronBeen/videos <br> LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Workflow supports customization such as dynamic subreddit input, filtering by upvotes, and branching logic | Suggested enhancements for advanced users                                                    |
| Workflow designed to run daily and automatically fetch, analyze, and distribute Reddit insights             | Project goal and usage scenario                                                              |

---

*Disclaimer:* The provided text is derived exclusively from an automated workflow created with n8n, a no-code automation tool. All data handled is legal and publicly available, complying strictly with content policies. No illegal, offensive, or protected content is included.

---