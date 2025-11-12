Altcoin News to LinkedIn Posts with DeepSeek AI & Google Sheets

https://n8nworkflows.xyz/workflows/altcoin-news-to-linkedin-posts-with-deepseek-ai---google-sheets-3668


# Altcoin News to LinkedIn Posts with DeepSeek AI & Google Sheets

### 1. Workflow Overview

This workflow automates the process of transforming altcoin news articles into LinkedIn-style posts using AI, storing the results, and optionally publishing them on LinkedIn. It targets crypto content creators or marketers who want to automate daily content curation and posting without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Schedule & Fetch RSS Feed:** Periodically triggers the workflow to fetch the latest altcoin news articles from a specified RSS feed.
- **1.2 Filter & Store Raw Articles:** Filters articles published today and containing relevant keywords, then stores raw data in Google Sheets.
- **1.3 AI Content Generation:** Uses DeepSeek AI and LangChain agents to rewrite filtered articles into LinkedIn-style posts.
- **1.4 Store Refined Posts & Batch Processing:** Converts AI output to plain text, stores refined posts in Google Sheets, and splits them into batches for processing.
- **1.5 LinkedIn Posting & Cleanup:** Posts the generated content to LinkedIn, removes posted entries from the sheet, and waits before restarting the cycle.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Fetch RSS Feed

- **Overview:**  
  This block triggers the workflow at defined intervals and fetches the latest news articles from the configured RSS feed.

- **Nodes Involved:**  
  - Schedule Tweet  
  - Fetch RSS Feed

- **Node Details:**

  - **Schedule Tweet**  
    - Type: Interval Trigger  
    - Role: Periodically triggers the workflow (default interval configurable, e.g., every 6 hours)  
    - Configuration: Interval time set in seconds (e.g., 21600 for 6 hours)  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Fetch RSS Feed"  
    - Edge Cases: Misconfiguration of interval could cause too frequent or infrequent triggers.

  - **Fetch RSS Feed**  
    - Type: RSS Feed Reader  
    - Role: Retrieves latest articles from the specified RSS feed URL (e.g., CoinDesk altcoin news)  
    - Configuration: RSS URL field set to the desired news source  
    - Inputs: Trigger from "Schedule Tweet"  
    - Outputs: Sends raw RSS items to "Filter Today & Script"  
    - Edge Cases: RSS feed downtime, malformed XML, or network errors.

#### 2.2 Filter & Store Raw Articles

- **Overview:**  
  Filters articles published on the current day and containing relevant keywords, then stores raw article data into Google Sheets for record-keeping.

- **Nodes Involved:**  
  - Filter Today & Script (Function)  
  - Store Raw Rss In A Sheet (Google Sheets)

- **Node Details:**

  - **Filter Today & Script**  
    - Type: Function  
    - Role: Filters RSS items by publication date (today) and keywords ("altcoin", "crypto", "defi", etc.)  
    - Configuration: Custom JavaScript filtering logic (likely checks date and keywords in title/content)  
    - Inputs: RSS feed items from "Fetch RSS Feed"  
    - Outputs: Filtered articles to "Store Raw Rss In A Sheet"  
    - Edge Cases: Timezone mismatches, missing publication dates, keyword case sensitivity.

  - **Store Raw Rss In A Sheet**  
    - Type: Google Sheets  
    - Role: Saves raw filtered RSS data into a designated Google Sheet for archival  
    - Configuration: Spreadsheet ID and sheet name configured; columns mapped to article fields (title, link, date, etc.)  
    - Inputs: Filtered articles from "Filter Today & Script"  
    - Outputs: Passes data to "Turn the rss into Linkedin post"  
    - Edge Cases: Google API quota limits, authentication errors, sheet permission issues.

#### 2.3 AI Content Generation

- **Overview:**  
  Converts raw news articles into engaging LinkedIn posts using DeepSeek AI and LangChain agents with memory support.

- **Nodes Involved:**  
  - DeepSeek Chat Model  
  - Simple Memory (LangChain Memory Buffer)  
  - Turn the rss into Linkedin post (LangChain Agent)  
  - Post From Gpt (Google Sheets)

- **Node Details:**

  - **DeepSeek Chat Model**  
    - Type: LangChain DeepSeek AI Chat Model  
    - Role: Provides AI language model capabilities to rewrite articles  
    - Configuration: Uses API key credential; model selection configurable (e.g., GPT-4)  
    - Inputs: Connected as AI language model input for the agent  
    - Outputs: AI-generated text to "Turn the rss into Linkedin post"  
    - Edge Cases: API rate limits, invalid API key, model unavailability.

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversational context or memory window for AI agent  
    - Configuration: Buffer size configurable to maintain recent interactions  
    - Inputs: AI agent memory input  
    - Outputs: Connected to "Turn the rss into Linkedin post" as memory context  
    - Edge Cases: Memory overflow or context loss if buffer too small.

  - **Turn the rss into Linkedin post**  
    - Type: LangChain Agent  
    - Role: Orchestrates AI prompt and response to convert article into LinkedIn post  
    - Configuration: Uses prompt templates including Hook, Body, CTA; integrates DeepSeek Chat Model and Simple Memory  
    - Inputs: Raw article data from "Store Raw Rss In A Sheet" and AI memory  
    - Outputs: AI-generated post text to "Post From Gpt"  
    - Edge Cases: Prompt failures, incomplete AI responses, expression errors.

  - **Post From Gpt**  
    - Type: Google Sheets  
    - Role: Stores AI-generated LinkedIn posts into a separate sheet for refined content  
    - Configuration: Spreadsheet and sheet configured for storing posts  
    - Inputs: AI-generated posts from "Turn the rss into Linkedin post"  
    - Outputs: Connects to "Covert to normal text" for text normalization  
    - Edge Cases: Google API errors, data format mismatches.

#### 2.4 Store Refined Posts & Batch Processing

- **Overview:**  
  Normalizes AI output text, stores it, and splits the posts into batches for controlled processing.

- **Nodes Involved:**  
  - Covert to normal text (Code)  
  - Store Refine Text (Google Sheets)  
  - Loop Over Items (Split In Batches)

- **Node Details:**

  - **Covert to normal text**  
    - Type: Code (JavaScript)  
    - Role: Cleans or normalizes AI-generated text to plain text format  
    - Configuration: Custom code to remove unwanted characters or formatting  
    - Inputs: AI posts from "Post From Gpt"  
    - Outputs: Cleaned text to "Store Refine Text"  
    - Edge Cases: Unexpected text formats, encoding issues.

  - **Store Refine Text**  
    - Type: Google Sheets  
    - Role: Saves cleaned LinkedIn posts into a dedicated sheet for posting queue  
    - Configuration: Spreadsheet and sheet configured for refined posts  
    - Inputs: Cleaned text from "Covert to normal text"  
    - Outputs: Connects to "Loop Over Items" for batch processing  
    - Edge Cases: Google API limits, concurrency issues.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes posts in manageable batches (default batch size configurable)  
    - Configuration: Batch size parameter (e.g., 1 or more)  
    - Inputs: Refined posts from "Store Refine Text"  
    - Outputs: Sends each batch to "Post To LinkedIn" and optionally to second output for other uses  
    - Edge Cases: Batch size misconfiguration, empty batches.

#### 2.5 LinkedIn Posting & Cleanup

- **Overview:**  
  Posts the generated content to LinkedIn, removes posted entries from the sheet, and waits before restarting the cycle.

- **Nodes Involved:**  
  - Post To LinkedIn  
  - Romove post after posting (Google Sheets)  
  - Wait For 3 Minute (Wait)

- **Node Details:**

  - **Post To LinkedIn**  
    - Type: LinkedIn Node  
    - Role: Publishes or schedules posts on LinkedIn using API  
    - Configuration: OAuth2 credentials for LinkedIn; post content mapped from batch items  
    - Inputs: Batches from "Loop Over Items"  
    - Outputs: Connects to "Romove post after posting"  
    - Edge Cases: LinkedIn API rate limits, authentication failures, content rejection.

  - **Romove post after posting**  
    - Type: Google Sheets  
    - Role: Deletes or marks posts as posted in the refined posts sheet to avoid duplicates  
    - Configuration: Spreadsheet and sheet configured; row deletion or flag update  
    - Inputs: Confirmation from "Post To LinkedIn"  
    - Outputs: Connects to "Wait For 3 Minute"  
    - Edge Cases: Race conditions, sheet permission errors.

  - **Wait For 3 Minute**  
    - Type: Wait  
    - Role: Pauses workflow for 3 minutes before next iteration or cleanup  
    - Configuration: Fixed wait time of 180 seconds  
    - Inputs: From "Romove post after posting"  
    - Outputs: Connects back to "Store Refine Text" to continue processing or end cycle  
    - Edge Cases: Workflow timeout if wait too long.

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                            | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                      |
|-------------------------|------------------------------------|--------------------------------------------|--------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Schedule Tweet          | Interval Trigger                   | Periodic workflow trigger                   | None                     | Fetch RSS Feed              |                                                                                                |
| Fetch RSS Feed          | RSS Feed Reader                   | Fetches latest altcoin news articles        | Schedule Tweet           | Filter Today & Script       |                                                                                                |
| Filter Today & Script   | Function                         | Filters articles by date and keywords       | Fetch RSS Feed           | Store Raw Rss In A Sheet    |                                                                                                |
| Store Raw Rss In A Sheet| Google Sheets                    | Stores raw filtered RSS data                 | Filter Today & Script    | Turn the rss into Linkedin post |                                                                                                |
| DeepSeek Chat Model     | LangChain DeepSeek AI Chat Model | Provides AI language model for rewriting    | AI memory input          | Turn the rss into Linkedin post |                                                                                                |
| Simple Memory           | LangChain Memory Buffer Window   | Maintains AI conversational context         | None                     | Turn the rss into Linkedin post |                                                                                                |
| Turn the rss into Linkedin post | LangChain Agent             | Converts articles into LinkedIn posts       | Store Raw Rss In A Sheet, Simple Memory, DeepSeek Chat Model | Post From Gpt               |                                                                                                |
| Post From Gpt           | Google Sheets                    | Stores AI-generated posts                    | Turn the rss into Linkedin post | Covert to normal text      |                                                                                                |
| Covert to normal text   | Code                            | Cleans AI-generated text                     | Post From Gpt            | Store Refine Text           |                                                                                                |
| Store Refine Text       | Google Sheets                    | Stores cleaned LinkedIn posts                | Covert to normal text    | Loop Over Items             |                                                                                                |
| Loop Over Items         | Split In Batches                 | Processes posts in batches                    | Store Refine Text        | Post To LinkedIn (main), (optional second output) |                                                                                                |
| Post To LinkedIn        | LinkedIn                        | Publishes posts to LinkedIn                   | Loop Over Items          | Romove post after posting   |                                                                                                |
| Romove post after posting | Google Sheets                  | Removes posted entries from sheet            | Post To LinkedIn         | Wait For 3 Minute           |                                                                                                |
| Wait For 3 Minute       | Wait                           | Pauses workflow for 3 minutes                 | Romove post after posting | Store Refine Text           |                                                                                                |
| Sticky Note             | Sticky Note                    | Various notes (empty content in JSON)        | N/A                      | N/A                        |                                                                                                |
| Sticky Note1            | Sticky Note                    | Various notes (empty content in JSON)        | N/A                      | N/A                        |                                                                                                |
| Sticky Note2            | Sticky Note                    | Various notes (empty content in JSON)        | N/A                      | N/A                        |                                                                                                |
| Sticky Note3            | Sticky Note                    | Various notes (empty content in JSON)        | N/A                      | N/A                        |                                                                                                |
| Sticky Note4            | Sticky Note                    | Various notes (empty content in JSON)        | N/A                      | N/A                        |                                                                                                |
| Sticky Note5            | Sticky Note                    | Various notes (empty content in JSON)        | N/A                      | N/A                        |                                                                                                |
| Sticky Note6            | Sticky Note                    | Various notes (empty content in JSON)        | N/A                      | N/A                        |                                                                                                |
| Sticky Note7            | Sticky Note                    | Various notes (empty content in JSON)        | N/A                      | N/A                        |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Interval Trigger Node: "Schedule Tweet"**  
   - Type: Interval  
   - Set interval to desired frequency (e.g., 21600 seconds for 6 hours).

2. **Create RSS Feed Read Node: "Fetch RSS Feed"**  
   - Type: RSS Feed Read  
   - Configure RSS URL to your preferred altcoin news source (e.g., CoinDesk RSS).  
   - Connect "Schedule Tweet" output to this node's input.

3. **Create Function Node: "Filter Today & Script"**  
   - Type: Function  
   - Add JavaScript code to filter RSS items published today and containing keywords like "altcoin", "crypto", "defi".  
   - Connect "Fetch RSS Feed" output to this node.

4. **Create Google Sheets Node: "Store Raw Rss In A Sheet"**  
   - Type: Google Sheets (Append Rows)  
   - Configure with your Google Sheets credentials.  
   - Set Spreadsheet ID and Sheet Name for raw RSS data storage.  
   - Map columns to article fields (title, link, pubDate, etc.).  
   - Connect "Filter Today & Script" output to this node.

5. **Create LangChain DeepSeek Chat Model Node: "DeepSeek Chat Model"**  
   - Type: LangChain DeepSeek AI Chat Model  
   - Configure with DeepSeek AI API credentials.  
   - Select preferred AI model (e.g., GPT-4).  
   - No direct input connection; used as AI language model by agent.

6. **Create LangChain Memory Node: "Simple Memory"**  
   - Type: LangChain Memory Buffer Window  
   - Configure buffer size (e.g., 5 messages).  
   - No direct input connection; used as AI memory by agent.

7. **Create LangChain Agent Node: "Turn the rss into Linkedin post"**  
   - Type: LangChain Agent  
   - Configure prompt template with sections: Hook, Body, CTA for LinkedIn style posts.  
   - Set AI language model to "DeepSeek Chat Model".  
   - Set AI memory to "Simple Memory".  
   - Connect "Store Raw Rss In A Sheet" output as input data.  
   - Connect outputs to "Post From Gpt".

8. **Create Google Sheets Node: "Post From Gpt"**  
   - Type: Google Sheets (Append Rows)  
   - Configure with credentials and target sheet for AI-generated posts.  
   - Map AI output fields accordingly.  
   - Connect "Turn the rss into Linkedin post" output to this node.

9. **Create Code Node: "Covert to normal text"**  
   - Type: Code  
   - Add JavaScript code to clean or normalize AI-generated text (e.g., remove HTML tags, special characters).  
   - Connect "Post From Gpt" output to this node.

10. **Create Google Sheets Node: "Store Refine Text"**  
    - Type: Google Sheets (Append Rows)  
    - Configure with credentials and sheet for cleaned LinkedIn posts.  
    - Map cleaned text fields.  
    - Connect "Covert to normal text" output to this node.

11. **Create Split In Batches Node: "Loop Over Items"**  
    - Type: Split In Batches  
    - Set batch size (e.g., 1 for one post at a time).  
    - Connect "Store Refine Text" output to this node.

12. **Create LinkedIn Node: "Post To LinkedIn"**  
    - Type: LinkedIn  
    - Configure OAuth2 credentials for LinkedIn API.  
    - Map post content from batch items.  
    - Connect "Loop Over Items" main output to this node.

13. **Create Google Sheets Node: "Romove post after posting"**  
    - Type: Google Sheets (Delete Rows or Update Rows)  
    - Configure to remove or flag posted entries in refined posts sheet.  
    - Connect "Post To LinkedIn" output to this node.

14. **Create Wait Node: "Wait For 3 Minute"**  
    - Type: Wait  
    - Set wait time to 180 seconds (3 minutes).  
    - Connect "Romove post after posting" output to this node.

15. **Connect "Wait For 3 Minute" output back to "Store Refine Text" or appropriate node to continue processing or loop.**

16. **Test the workflow end-to-end, verify API credentials, sheet permissions, and LinkedIn posting.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| To customize the RSS feed URL, edit the "Fetch RSS Feed" node's URL field.                                    | Workflow Setup Instructions                                                                     |
| To change the AI model, update the "DeepSeek Chat Model" node's Model field (e.g., GPT-4).                    | Workflow Setup Instructions                                                                     |
| Modify the AI prompt in the "Turn the rss into Linkedin post" LangChain Agent node to adjust tone or keywords.| Workflow Setup Instructions                                                                     |
| Adjust posting frequency by changing the "Schedule Tweet" node's interval parameter (e.g., 21600 for 6 hours).| Workflow Setup Instructions                                                                     |
| LinkedIn API or scheduling tool is optional; the workflow can be adapted to other platforms or manual posting.| Workflow Description                                                                            |
| Google Sheets API must be enabled and credentials properly configured for all Google Sheets nodes.            | Prerequisites                                                                                   |
| DeepSeek AI API key or OpenAI API key required for AI content generation nodes.                               | Prerequisites                                                                                   |
| LinkedIn OAuth2 credentials required for automated posting to LinkedIn.                                       | Prerequisites                                                                                   |
| Potential failure points include API rate limits, authentication errors, network issues, and malformed data. | General Notes                                                                                   |

---

This document provides a detailed, structured reference to understand, reproduce, and maintain the "Altcoin News to LinkedIn Posts with DeepSeek AI & Google Sheets" workflow.