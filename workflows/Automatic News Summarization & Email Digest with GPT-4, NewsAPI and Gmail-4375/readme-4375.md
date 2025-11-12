Automatic News Summarization & Email Digest with GPT-4, NewsAPI and Gmail

https://n8nworkflows.xyz/workflows/automatic-news-summarization---email-digest-with-gpt-4--newsapi-and-gmail-4375


# Automatic News Summarization & Email Digest with GPT-4, NewsAPI and Gmail

### 1. Workflow Overview

This workflow automates the process of fetching the latest news headlines, generating AI-powered summaries, and sending personalized email digests to a subscriber list. The target use cases include newsletter creators, teams, organizations, and content curators who want to deliver concise, up-to-date news summaries effortlessly.

The workflow is logically grouped into the following blocks:

- **1.1 Scheduled News Retrieval**: Triggers the workflow periodically and pulls the latest headlines from an external news API.
- **1.2 AI Summarization**: Uses GPT-4 via an AI Agent and OpenAI Chat Model to generate concise 5-bullet summaries of the news headlines.
- **1.3 Subscriber List Access**: Retrieves subscriber names and email addresses from a Google Sheets document.
- **1.4 Personalized Email Delivery**: Sends customized news digest emails to each subscriber using Gmail.
- **1.5 Documentation & Guidance**: A sticky note providing detailed explanation, setup instructions, and best practices.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled News Retrieval

**Overview:**  
This block initiates the workflow at a fixed interval and fetches the latest top news headlines from NewsAPI, focusing on US-based sources.

**Nodes Involved:**  
- Schedule Trigger  
- Pull News

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow every 10 minutes.  
  - *Configuration:* Interval set to 10 minutes.  
  - *Input/Output:* No input; outputs trigger signal to "Pull News".  
  - *Potential Failures:* Workflow not running if n8n instance is down; time zone considerations may affect scheduling.  

- **Pull News**  
  - *Type:* HTTP Request  
  - *Role:* Fetches top headlines from NewsAPI using a GET request.  
  - *Configuration:*  
    - URL: `https://newsapi.org/v2/top-headlines`  
    - Query Parameters: `country=us`, `apiKey=NEWS_API_KEY` (replace with actual key)  
  - *Input/Output:* Receives trigger from Schedule Trigger; outputs JSON containing news articles to "AI Agent".  
  - *Potential Failures:*  
    - API key invalid or expired → authentication error.  
    - Rate limiting by NewsAPI.  
    - Network timeout or connectivity issues.  
    - Empty or malformed response if no news returned.

---

#### 1.2 AI Summarization

**Overview:**  
This block processes the fetched news headlines by generating a concise summary in five bullet points using GPT-4 via an AI Agent and OpenAI Chat Model.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Acts as a news expert, extracts headlines, and formulates a summarization prompt.  
  - *Configuration:*  
    - Text prompt includes instruction: "You are act as news expert and collect all news headlines in below and summarise in 5 bullets."  
    - Injects the title of the second article from pulled news: `{{ $json.articles[1].title }}` (Note: zero-based index, second item)  
    - Connects to OpenAI Chat Model for language model processing.  
  - *Input/Output:* Receives news JSON from Pull News; outputs summary text to "Email list".  
  - *Potential Failures:*  
    - If `articles[1]` is undefined (e.g., less than 2 articles), expression error.  
    - API quota exhaustion or network issues with OpenAI.  
    - Model response latency or unexpected output format.

- **OpenAI Chat Model**  
  - *Type:* LangChain LM Chat OpenAI  
  - *Role:* Executes GPT-4 based summarization.  
  - *Configuration:*  
    - Model: `gpt-4o-mini` (a GPT-4 variant)  
    - No additional options configured.  
  - *Input/Output:* Receives prompt from AI Agent; outputs summarized text back to AI Agent.  
  - *Potential Failures:*  
    - API key invalid or rate limits.  
    - Model downtime or version deprecation.

---

#### 1.3 Subscriber List Access

**Overview:**  
Retrieves subscriber data (names and emails) from a Google Sheets document to personalize email delivery.

**Nodes Involved:**  
- Email list

**Node Details:**

- **Email list**  
  - *Type:* Google Sheets  
  - *Role:* Reads subscriber data from a specific Google Sheet.  
  - *Configuration:*  
    - Document ID: `1L2dLObdw_aicD6fTd-ElHSBBJIj1aBmKT7FecMtbyyE`  
    - Sheet Name: `gid=0` (Sheet1)  
    - Expected columns: "Name" and "Email"  
  - *Input/Output:* Receives summarized news from AI Agent; outputs each subscriber row to "Send Mail".  
  - *Potential Failures:*  
    - OAuth2 credential issues for Google Sheets access.  
    - Sheet access permission denied or document moved/deleted.  
    - Missing or malformed data in sheet rows.

---

#### 1.4 Personalized Email Delivery

**Overview:**  
Sends customized news digest emails to each subscriber with their personalized greeting and the AI-generated summary.

**Nodes Involved:**  
- Send Mail

**Node Details:**

- **Send Mail**  
  - *Type:* Gmail  
  - *Role:* Sends email to each subscriber with news digest content.  
  - *Configuration:*  
    - Recipient email: `={{ $json.Email }}` (from Google Sheets)  
    - Subject: "News Headlines"  
    - Message body:  
      ```
      Hi {{ $json.Name }},  
      Please find the top news headlines as below:  
      {{ $('AI Agent').item.json.output }}
      ```  
    - Email type: Plain text  
  - *Input/Output:* Receives subscriber info from Email list; no output nodes.  
  - *Potential Failures:*  
    - Gmail OAuth2 authentication issues.  
    - Sending limits or throttling by Gmail.  
    - Invalid email addresses causing send failures.

---

#### 1.5 Documentation & Guidance

**Overview:**  
Provides a detailed sticky note explaining the workflow, use cases, setup instructions, and tips for customization.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - *Type:* Sticky Note (documentation node)  
  - *Role:* Offers comprehensive user guidance for the workflow including setup, usage, and customization tips.  
  - *Content Highlights:*  
    - Summary of workflow purpose and nodes used.  
    - Credential requirements: NewsAPI, OpenAI, Google Sheets OAuth2, Gmail OAuth2.  
    - Google Sheets structure for subscriber emails.  
    - Stepwise workflow explanation and pro tips.  
    - Suggestions for further customization and scaling.  
  - *Input/Output:* None (informational only).

---

### 3. Summary Table

| Node Name       | Node Type                      | Functional Role                        | Input Node(s)     | Output Node(s)      | Sticky Note                                                                                                    |
|-----------------|--------------------------------|-------------------------------------|-------------------|---------------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Trigger| Schedule Trigger               | Initiates workflow every 10 minutes | None              | Pull News           | 📰 AI News Digest Agent: Auto News Summarizer & Emailer - detailed explanation and setup guidance              |
| Pull News       | HTTP Request                  | Fetches latest US top headlines     | Schedule Trigger  | AI Agent            | 📰 AI News Digest Agent: Auto News Summarizer & Emailer - detailed explanation and setup guidance              |
| AI Agent        | LangChain Agent               | Generates summarization prompt      | Pull News         | Email list          | 📰 AI News Digest Agent: Auto News Summarizer & Emailer - detailed explanation and setup guidance              |
| OpenAI Chat Model| LangChain LM Chat OpenAI      | Executes GPT-4 summarization        | AI Agent (ai_languageModel) | AI Agent (lmChatOpenAi) | 📰 AI News Digest Agent: Auto News Summarizer & Emailer - detailed explanation and setup guidance              |
| Email list      | Google Sheets                 | Reads subscriber info               | AI Agent          | Send Mail           | 📰 AI News Digest Agent: Auto News Summarizer & Emailer - detailed explanation and setup guidance              |
| Send Mail       | Gmail                        | Sends personalized news digest emails | Email list        | None                | 📰 AI News Digest Agent: Auto News Summarizer & Emailer - detailed explanation and setup guidance              |
| Sticky Note     | Sticky Note                  | Documentation & workflow guidance   | None              | None                | 📰 AI News Digest Agent: Auto News Summarizer & Emailer - detailed explanation and setup guidance              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger every 10 minutes (field: minutesInterval = 10).  
   - This node starts the process regularly.

2. **Add an HTTP Request node named "Pull News"**  
   - Configure as GET request to `https://newsapi.org/v2/top-headlines`.  
   - Add query parameters:  
     - `country` = `us`  
     - `apiKey` = Your NewsAPI key (replace placeholder).  
   - Connect output of Schedule Trigger to input of Pull News.

3. **Add an AI Agent node**  
   - Type: LangChain Agent.  
   - Configure prompt text as:  
     ```
     You are act as news expert and collect all news headlines in below and summarise in 5 bullets.

     {{ $json.articles[1].title }}
     ```  
   - Connect output of Pull News to input of AI Agent.

4. **Add an OpenAI Chat Model node**  
   - Type: LangChain LM Chat OpenAI.  
   - Set model to `gpt-4o-mini`.  
   - Connect AI Agent's language model input to this node and output back to AI Agent.

5. **Add a Google Sheets node named "Email list"**  
   - Connect output of AI Agent to input of Email list node.  
   - Configure with:  
     - Document ID: your Google Sheets document ID containing subscriber list.  
     - Sheet Name: `gid=0` or the sheet containing subscriber data.  
   - Ensure the sheet has columns: "Name" and "Email".  
   - Use OAuth2 credentials with Google Sheets scope.

6. **Add a Gmail node named "Send Mail"**  
   - Connect output of Email list node to input of Send Mail node.  
   - Configure email parameters:  
     - Send To: `={{ $json.Email }}` (from Google Sheets row)  
     - Subject: "News Headlines"  
     - Message:  
       ```
       Hi {{ $json.Name }},  
       Please find the top news headlines as below:  
       {{ $('AI Agent').item.json.output }}
       ```  
     - Email Type: Text  
   - Use Gmail OAuth2 credentials authorized to send emails.

7. **(Optional) Add a Sticky Note node**  
   - Use to document workflow purpose, setup instructions, and customization tips.

8. **Credential Setup**  
   - NewsAPI key: Obtain free API key from https://newsapi.org.  
   - OpenAI API key: Configure in LangChain/OpenAI nodes.  
   - Google Sheets OAuth2: Create and authorize OAuth2 credentials for accessing subscriber sheet.  
   - Gmail OAuth2: Authorize OAuth2 credentials for sending emails.

9. **Connect nodes in sequence:**  
   - Schedule Trigger → Pull News → AI Agent (with OpenAI Chat Model as sub-node) → Email list → Send Mail

10. **Test the workflow with a small subscriber list and verify that emails with summarized news are received as expected.**

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow automatically fetches top news headlines, generates 5-bullet summaries, and emails them. | Workflow purpose and use case.                                                                   |
| NewsAPI provides reliable, real-time news headlines; requires free API key registration.          | https://newsapi.org                                                                             |
| OpenAI GPT-4 is used for AI summarization; requires API key and usage compliance.                  | https://platform.openai.com                                                                     |
| Google Sheets must have subscriber data with "Name" and "Email" columns for personalization.       | Google Sheets API and OAuth2 setup documentation.                                               |
| Gmail node sends personalized emails; requires OAuth2 with appropriate scopes.                      | Gmail API documentation and OAuth2 setup.                                                       |
| Customize country, schedule frequency, and news categories to tailor news digests.                  | Workflow customization suggestions.                                                             |
| Pro tip: Monitor API usage limits and email delivery effectiveness periodically.                     | Best practices for scaling and reliability.                                                     |
| Consider adding unsubscribe or list management features for compliance with email regulations.      | Compliance and legal considerations.                                                            |

---

**Disclaimer:** The text provided is derived exclusively from an n8n automated workflow. It fully complies with current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.