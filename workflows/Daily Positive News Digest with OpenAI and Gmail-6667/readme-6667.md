Daily Positive News Digest with OpenAI and Gmail

https://n8nworkflows.xyz/workflows/daily-positive-news-digest-with-openai-and-gmail-6667


# Daily Positive News Digest with OpenAI and Gmail

### 1. Workflow Overview

This n8n workflow automates the creation and delivery of a **daily positive news digest** email. Its target use case is to provide users with a curated summary of uplifting and positive news articles every morning, enhancing daily motivation and well-being.

The workflow is logically divided into the following blocks:

- **1.1 Daily Trigger**: Automatically starts the workflow every day at 7:00 AM.
- **1.2 News Retrieval**: Fetches the latest positive news articles from an RSS feed.
- **1.3 Data Preparation for AI**: Formats and prepares the news data for AI processing.
- **1.4 AI Summarization**: Uses OpenAI to summarize articles, filtering for positive content.
- **1.5 Post-AI Filtering**: Removes non-positive summaries and prepares final data.
- **1.6 Conditional Branching**: Checks if positive news exists to decide email content.
- **1.7 Email Formatting**: Constructs the email body for positive news or fallback message.
- **1.8 Email Delivery**: Sends the compiled digest via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger

- **Overview:**  
  Triggers the workflow automatically every day at 7:00 AM server time to start the news digest process.

- **Nodes Involved:**  
  - Daily Morning Trigger (7 AM)

- **Node Details:**  
  - **Node Name:** Daily Morning Trigger (7 AM)  
  - **Type:** Cron Trigger  
  - **Configuration:** Set to trigger every day at 07:00 (Hour=7, Minute=0). No additional options.  
  - **Input/Output:** No input; output triggers the next node (Fetch Positive News (RSS)).  
  - **Edge Cases:** Workflow depends on server time zone; if server time is incorrect, trigger time shifts. Cron node failure unlikely but possible due to system errors or server downtime.  
  - **Notes:** The schedule can be customized by changing hour and minute parameters.

#### 1.2 News Retrieval

- **Overview:**  
  Fetches the latest articles from a positive news RSS feed to gather raw data for summarization.

- **Nodes Involved:**  
  - Fetch Positive News (RSS)

- **Node Details:**  
  - **Node Name:** Fetch Positive News (RSS)  
  - **Type:** RSS Feed  
  - **Configuration:** URL set to `https://www.goodnewsnetwork.org/feed/` (Good News Network).  
  - **Input/Output:** Input from Cron trigger; output is the list of RSS feed items to the Prepare for AI node.  
  - **Edge Cases:** RSS feed downtime or changes in feed structure may cause failures or empty outputs. Network issues may cause timeouts.  
  - **Notes:** Additional RSS sources can be added in parallel and merged if desired.

#### 1.3 Data Preparation for AI

- **Overview:**  
  Processes the raw RSS items into a structured format with a combined text field for AI consumption, while preserving original metadata.

- **Nodes Involved:**  
  - Prepare for AI

- **Node Details:**  
  - **Node Name:** Prepare for AI  
  - **Type:** Function  
  - **Configuration:**  
    - Extracts `title`, `description` (contentSnippet or description), and `link` from each RSS item.  
    - Creates a new field `articleText` combining the title and description for AI input.  
    - Preserves original title, description, and link for later use.  
  - **Input/Output:** Input from RSS Feed; output to OpenAI summarization node.  
  - **Edge Cases:** Missing title/description handled by fallback strings ('No Title', 'No Description').  
  - **Notes:** No manual configuration needed; script embedded in node.

#### 1.4 AI Summarization

- **Overview:**  
  Sends each prepared article to OpenAI's GPT model to generate a concise positive summary or mark it as 'SKIP' if not sufficiently positive.

- **Nodes Involved:**  
  - AI: Summarize Positive News

- **Node Details:**  
  - **Node Name:** AI: Summarize Positive News  
  - **Type:** OpenAI  
  - **Configuration:**  
    - Credentials: Uses OpenAI API key (user must provide).  
    - Model: `gpt-3.5-turbo` by default, with option to upgrade to `gpt-4o` for higher quality.  
    - Prompt: System prompt instructs AI to summarize only positive or neutral-to-positive news, outputting 'SKIP' otherwise. User prompt injects the combined article text.  
  - **Input/Output:** Input from Prepare for AI; output is AI response per article.  
  - **Edge Cases:**  
    - API authentication errors if credentials invalid.  
    - Rate limits or timeouts from OpenAI API.  
    - Unexpected AI responses or malformed messages.  
  - **Notes:** AI summarization is the core filtering mechanism.

#### 1.5 Post-AI Filtering

- **Overview:**  
  Filters out all AI responses marked 'SKIP', leaving only positive summaries, and reformats data for email composition.

- **Nodes Involved:**  
  - Filter & Prepare Positive Summaries

- **Node Details:**  
  - **Node Name:** Filter & Prepare Positive Summaries  
  - **Type:** Function  
  - **Configuration:**  
    - Iterates over AI outputs; removes entries where the AI response is 'SKIP'.  
    - Extracts and packages original title, link, and AI summary for each positive article.  
  - **Input/Output:** Input from OpenAI; output filtered positive summaries to the If node.  
  - **Edge Cases:** Empty array if no positive news found; handled downstream.  
  - **Notes:** No manual configuration required.

#### 1.6 Conditional Branching

- **Overview:**  
  Decides workflow path based on whether any positive news summaries exist.

- **Nodes Involved:**  
  - If Positive News Found

- **Node Details:**  
  - **Node Name:** If Positive News Found  
  - **Type:** If (Conditional)  
  - **Configuration:** Checks if the JSON array length of filtered summaries is not zero.  
  - **Input/Output:** Input from Filter & Prepare Positive Summaries;  
    - True branch (positive news found) connects to Format Positive News Email.  
    - False branch (no positive news) connects to Format No Positive News Message.  
  - **Edge Cases:** Faulty empty checks could misroute flow, but unlikely due to explicit length check.  
  - **Notes:** No manual configuration needed.

#### 1.7 Email Formatting

- **Overview:**  
  Formats the email body content differently depending on the presence of positive news.

- **Nodes Involved:**  
  - Format Positive News Email  
  - Format No Positive News Message

- **Node Details:**  
  - **Format Positive News Email**  
    - **Type:** Function  
    - **Configuration:**  
      - Builds an email body in Markdown style with greetings, each article’s title in bold, summary, and a "Read more" link.  
      - Closes with a friendly sign-off and n8n credit.  
      - Sets email subject to "☀️ Your Daily Positive News Digest!".  
    - **Input/Output:** Input from If node True branch; output to Gmail node.  
    - **Edge Cases:** If input items are malformed, email formatting may degrade.  
  - **Format No Positive News Message**  
    - **Type:** Function  
    - **Configuration:**  
      - Creates a fallback email message indicating no positive news was found.  
      - Sets email subject to "☁️ Daily Positive News Digest: No Positive News Today".  
    - **Input/Output:** Input from If node False branch; output to Gmail node.  
    - **Edge Cases:** None significant; default message always present.  
  - **Notes:** Both nodes automatically generate content; customization possible by editing the scripts.

#### 1.8 Email Delivery

- **Overview:**  
  Sends the final compiled email digest to the configured recipient via Gmail API.

- **Nodes Involved:**  
  - Send Daily Digest Email

- **Node Details:**  
  - **Node Name:** Send Daily Digest Email  
  - **Type:** Gmail  
  - **Configuration:**  
    - Gmail API credential required (OAuth2).  
    - From email: Must match authenticated Gmail account.  
    - To email: User must replace placeholder `YOUR_RECIPIENT_EMAIL@example.com` with the actual recipient address.  
    - Subject and Text: Dynamically pulled from previous formatting nodes’ output.  
  - **Input/Output:** Input from either email formatting node; no output.  
  - **Edge Cases:**  
    - Authentication failure if credentials expired or invalid.  
    - Email rejected if recipient address invalid or quota exceeded.  
    - Network issues causing send failures.  
  - **Notes:** Testing recommended to verify email delivery.

---

### 3. Summary Table

| Node Name                   | Node Type        | Functional Role                        | Input Node(s)             | Output Node(s)                   | Sticky Note                                                                                                           |
|-----------------------------|------------------|-------------------------------------|---------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Daily Morning Trigger (7 AM) | Cron             | Daily execution trigger               | —                         | Fetch Positive News (RSS)        | Triggers workflow daily at 7:00 AM. Adjust 'Hour' and 'Minute' to change schedule.                                   |
| Fetch Positive News (RSS)    | RSS Feed         | Retrieve positive news articles      | Daily Morning Trigger      | Prepare for AI                  | Fetches from `https://www.goodnewsnetwork.org/feed/`. Add more RSS nodes to combine sources if needed.                |
| Prepare for AI               | Function         | Format articles for AI input          | Fetch Positive News (RSS)  | AI: Summarize Positive News      | Formats title and description into a single text field for AI processing; preserves original data.                    |
| AI: Summarize Positive News | OpenAI           | Summarize/filter for positive news   | Prepare for AI             | Filter & Prepare Positive Summaries | Uses OpenAI GPT-3.5-turbo by default; summarizes or outputs 'SKIP' for non-positive news. Requires OpenAI API Key.     |
| Filter & Prepare Positive Summaries | Function | Filter out non-positive articles      | AI: Summarize Positive News | If Positive News Found           | Filters out 'SKIP' responses; prepares clean summary data.                                                             |
| If Positive News Found       | If (Conditional) | Branch based on presence of positives | Filter & Prepare Positive Summaries | Format Positive News Email / Format No Positive News Message | Checks if any positive news summaries exist to decide email content path.                                             |
| Format Positive News Email   | Function         | Compose email body with positive news | If Positive News Found (True) | Send Daily Digest Email           | Formats email in Markdown with greetings, summaries, and links.                                                       |
| Format No Positive News Message | Function      | Compose fallback email message        | If Positive News Found (False) | Send Daily Digest Email           | Provides default message if no positive news found.                                                                   |
| Send Daily Digest Email      | Gmail            | Send compiled email digest            | Format Positive News Email / Format No Positive News Message | —                               | Sends email via Gmail API. Requires Gmail API credential and correct email addresses.                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**  
   - Type: Cron  
   - Name: `Daily Morning Trigger (7 AM)`  
   - Set mode to `Every Day`  
   - Set Hour to `7` and Minute to `0`  
   - No credentials needed  
   - Connect output to next node.

2. **Create RSS Feed Node**  
   - Type: RSS Feed  
   - Name: `Fetch Positive News (RSS)`  
   - URL: `https://www.goodnewsnetwork.org/feed/`  
   - Connect input to `Daily Morning Trigger (7 AM)` output.

3. **Create Function Node (Prepare for AI)**  
   - Type: Function  
   - Name: `Prepare for AI`  
   - Paste this JavaScript code:
     ```js
     const preparedItems = [];

     for (const item of items) {
       const title = item.json.title || 'No Title';
       const description = item.json.contentSnippet || item.json.description || 'No Description';
       const link = item.json.link || '#';

       preparedItems.push({
         json: {
           originalTitle: title,
           originalDescription: description,
           originalLink: link,
           articleText: `Title: ${title}\nDescription: ${description}`
         }
       });
     }

     return preparedItems;
     ```
   - Connect input to `Fetch Positive News (RSS)` output.

4. **Create OpenAI Node (AI Summarization)**  
   - Type: OpenAI  
   - Name: `AI: Summarize Positive News`  
   - Credentials: Create new OpenAI API credential with your API key (must start with `sk-`).  
   - Model: Select `gpt-3.5-turbo` or optionally `gpt-4o`.  
   - Messages:  
     - System Role:  
       `"You are a news summarizer focused only on positive and uplifting news. Read the provided article text. If it is clearly positive or neutral-to-positive, summarize its core message in 2-3 concise sentences, focusing on the positive aspects. If it is negative, neutral, or not news (e.g., ads), output the single word 'SKIP'."`  
     - User Role:  
       `"Article:\n{{ $json.articleText }}"`  
   - Connect input to `Prepare for AI` output.

5. **Create Function Node (Filter & Prepare Positive Summaries)**  
   - Type: Function  
   - Name: `Filter & Prepare Positive Summaries`  
   - Paste this JavaScript code:
     ```js
     const positiveSummaries = [];

     for (const item of items) {
       const aiResponse = item.json.choices[0].message.content.trim();

       if (aiResponse.toUpperCase() !== 'SKIP') {
         positiveSummaries.push({
           json: {
             originalTitle: item.json.originalTitle,
             originalLink: item.json.originalLink,
             summary: aiResponse
           }
         });
       }
     }

     return positiveSummaries;
     ```
   - Connect input to `AI: Summarize Positive News` output.

6. **Create If Node (Check for Positive News)**  
   - Type: If  
   - Name: `If Positive News Found`  
   - Condition:  
     - Value 1: `={{ $json.length }}`  
     - Operation: `notEqual`  
     - Value 2: `0`  
   - Connect input to `Filter & Prepare Positive Summaries` output.

7. **Create Function Node (Format Positive News Email)**  
   - Type: Function  
   - Name: `Format Positive News Email`  
   - Paste this JavaScript code:
     ```js
     let emailBody = "";

     emailBody += "Good morning! Here's your daily dose of positive news:\n\n";

     for (const item of items) {
       emailBody += `**${item.json.originalTitle}**\n` +
                    `${item.json.summary}\n` +
                    `Read more: ${item.json.originalLink}\n\n---\n\n`;
     }

     emailBody += "Have a wonderful day!\n\nThis digest was brought to you by n8n.";

     return [{ json: { emailSubject: "☀️ Your Daily Positive News Digest!", emailBody: emailBody } }];
     ```
   - Connect to `If Positive News Found` node's **True** output.

8. **Create Function Node (Format No Positive News Message)**  
   - Type: Function  
   - Name: `Format No Positive News Message`  
   - Paste this JavaScript code:
     ```js
     return [{ json: { 
       emailSubject: "☁️ Daily Positive News Digest: No Positive News Today", 
       emailBody: "Good morning!\n\nUnfortunately, I couldn't find any predominantly positive news articles for your digest today.\n\nStay positive, and check back tomorrow!\n\nThis digest was brought to you by n8n." 
     } }];
     ```
   - Connect to `If Positive News Found` node's **False** output.

9. **Create Gmail Node (Send Daily Digest Email)**  
   - Type: Gmail  
   - Name: `Send Daily Digest Email`  
   - Credentials: Set up Gmail API OAuth2 credentials for your Gmail account.  
   - From Email: Enter your Gmail address (must match OAuth credentials).  
   - To Email: Replace `YOUR_RECIPIENT_EMAIL@example.com` with the actual email recipient.  
   - Subject: Set to `={{ $json.emailSubject }}` (expression to pull from previous node).  
   - Text: Set to `={{ $json.emailBody }}` (expression).  
   - Connect inputs from both `Format Positive News Email` and `Format No Positive News Message` nodes.

10. **Verify and Test**  
    - Run the workflow manually or wait for the scheduled trigger.  
    - Confirm receipt of the daily email digest.  
    - Adjust times, email addresses, or add RSS feeds as desired.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Good News Network RSS feed is used as the primary positive news source.                                          | https://www.goodnewsnetwork.org/feed/                                                               |
| OpenAI API key is required for summarization; ensure it is kept secure and usage costs are monitored.            | https://platform.openai.com/account/api-keys                                                        |
| Gmail API credentials must be created via Google Cloud Console with OAuth2 for sending emails programmatically.  | https://developers.google.com/gmail/api/quickstart/js                                              |
| Markdown formatting used in emails is supported by Gmail for bolding and line breaks.                            | https://support.google.com/mail/answer/6584                                                         |
| Consider upgrading OpenAI model to `gpt-4o` for improved summary quality at additional cost.                     | https://platform.openai.com/docs/models/gpt-4                                                       |
| Testing the Gmail node is crucial to verify proper email delivery and authentication before production use.     | n8n documentation on Gmail node usage                                                               |

---

*Disclaimer: The provided text is generated from an automated n8n workflow. It complies with current content policies and contains no illegal or offensive content. All data processed is legal and publicly available.*