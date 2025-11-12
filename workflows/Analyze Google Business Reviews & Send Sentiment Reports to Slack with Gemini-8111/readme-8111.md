Analyze Google Business Reviews & Send Sentiment Reports to Slack with Gemini

https://n8nworkflows.xyz/workflows/analyze-google-business-reviews---send-sentiment-reports-to-slack-with-gemini-8111


# Analyze Google Business Reviews & Send Sentiment Reports to Slack with Gemini

### 1. Workflow Overview

This workflow automates the process of analyzing Google Business Profile reviews, performing sentiment analysis using Google Gemini models, and sending summarized sentiment reports to a Slack channel. It is designed to help businesses monitor customer feedback trends systematically over a defined time period.

**Target Use Cases:**  
- Businesses wanting automated sentiment analysis of customer reviews from Google Business Profile.  
- Teams requiring periodic consolidated reports on customer sentiment shared directly in Slack channels.  
- Marketing or customer service teams tracking feedback trends over time to identify strengths and weaknesses.

**Logical Blocks:**  
- **1.1 Scheduled Trigger & Time Period Setup:** Initiates the workflow monthly and configures the review extraction period.  
- **1.2 Fetch & Filter Google Reviews:** Retrieves reviews from Google Business Profile and filters them by the specified time period.  
- **1.3 Data Mapping for Sentiment Analysis:** Prepares review data by mapping comments, ratings, and dates into a consistent format.  
- **1.4 Sentiment Analysis via Google Gemini:** Runs batch sentiment analysis on the mapped review data using a Google Gemini language model.  
- **1.5 Consolidation of Sentiment Analysis:** Uses AI agents to aggregate and summarize sentiment results into a structured JSON report.  
- **1.6 Formatting Slack Messages:** Converts the consolidated JSON report into Slack message blocks tailored to sentiment categories (positive, neutral, negative).  
- **1.7 Slack Notification:** Sends the formatted sentiment reports as Slack messages to a specified channel, differentiated by overall sentiment.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Time Period Setup

- **Overview:**  
Triggers the workflow every month and sets the review period (default 3 years) for filtering reviews.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Set time period  

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow on a monthly interval.  
    - Configuration: Interval set to trigger every month.  
    - Connections: Triggers "Set time period" node.  
    - Edge Cases: Misconfiguration could cause the workflow not to run periodically.

  - **Set time period**  
    - Type: Set  
    - Role: Defines workflow variables `period` ("year") and `time` ("3") to specify the time window for reviews.  
    - Configuration: Assigns static values for period and time (e.g., 3 years).  
    - Connections: Outputs to "Get many reviews".  
    - Edge Cases: Hardcoded period/time could limit flexibility; user must update for different ranges.

#### 1.2 Fetch & Filter Google Reviews

- **Overview:**  
Fetches up to 1000 reviews from the configured Google Business Profile and filters them based on the defined time period.

- **Nodes Involved:**  
  - Get many reviews  
  - Filter review based on the time period  

- **Node Details:**  

  - **Get many reviews**  
    - Type: Google Business Profile  
    - Role: Retrieves reviews using Google Business Profile API.  
    - Configuration: Retrieves up to 1000 reviews for a specified account and location.  
    - Expressions: Account and location are configured by name dynamically.  
    - Connections: Passes output to the filter node.  
    - Edge Cases: API quota limits, authentication errors, or invalid account/location may cause failure.

  - **Filter review based on the time period**  
    - Type: Filter  
    - Role: Filters reviews by update time, retaining only those newer than the calculated threshold (current date minus period).  
    - Configuration: Uses an expression comparing each review's updateTime to a dynamically calculated cutoff date.  
    - Connections: Forward filtered data to "Map the comment and rating".  
    - Edge Cases: Date format inconsistencies, timezone issues, or empty review sets.

#### 1.3 Data Mapping for Sentiment Analysis

- **Overview:**  
Maps and standardizes each review’s comment, rating, and date fields to consistent JSON properties used by sentiment analysis.

- **Nodes Involved:**  
  - Map the comment and rating  

- **Node Details:**  

  - **Map the comment and rating**  
    - Type: Set  
    - Role: Normalizes review data fields:  
      - `Comment`: uses review comment or defaults to "no comment"  
      - `Raiting`: uses starRating or defaults to "0" (note the typo in 'Raiting')  
      - `Date`: uses updateTime field  
    - Configuration: Uses expressions to populate each field.  
    - Connections: Sends output to "Sentiment Analysis".  
    - Edge Cases: Missing or malformed review fields; typo in `Raiting` might cause confusion downstream.

#### 1.4 Sentiment Analysis via Google Gemini

- **Overview:**  
Performs batch sentiment analysis on prepared review data using Google Gemini language model with a custom system prompt.

- **Nodes Involved:**  
  - Sentiment Analysis  
  - Google Gemini Chat Model1  

- **Node Details:**  

  - **Google Gemini Chat Model1**  
    - Type: LangChain Google Gemini LM Chat  
    - Role: Provides the language model backend for sentiment analysis node.  
    - Configuration: Model "models/gemini-2.0-flash" with max tokens 5000.  
    - Credentials: Google Palm API credentials assigned.  
    - Connections: Linked as AI language model to "Sentiment Analysis".  
    - Edge Cases: API quota limits, network errors, or invalid credentials.

  - **Sentiment Analysis**  
    - Type: LangChain Sentiment Analysis  
    - Role: Runs sentiment analysis on batches of review JSON objects.  
    - Configuration:  
      - Batch size 20 with 60s delay between batches to avoid rate limits.  
      - Categories defined: Positive, Neutral, Negative.  
      - System prompt instructs detailed sentiment extraction rules and grouping by month.  
      - Input text concatenates Comment, Raiting, and Date fields.  
      - Auto-fixing enabled for prompt errors.  
    - Connections: Outputs to three parallel "Convert to json string" nodes for different sentiment categories.  
    - Edge Cases: Large data sets may cause timeout, malformed JSON inputs, or unexpected AI responses.

#### 1.5 Consolidation of Sentiment Analysis

- **Overview:**  
Consolidates batch sentiment results into a single comprehensive JSON report describing overall sentiment, ratings, and trends.

- **Nodes Involved:**  
  - Convert to json string  
  - Convert to json string1  
  - Convert to json string2  
  - AI Agent  
  - AI Agent1  
  - AI Agent2  
  - Google Gemini Chat Model  

- **Node Details:**  

  - **Convert to json string / Convert to json string1 / Convert to json string2**  
    - Type: Code  
    - Role: Converts array of individual sentiment analysis results into a single JSON string under a key named `feedback`.  
    - Configuration: Executes once, serializes all input JSON items with indentation.  
    - Connections: Each sends its output to a corresponding AI Agent node.  
    - Edge Cases: Empty inputs, serialization errors.

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini LM Chat  
    - Role: Shared AI language model resource for the three AI Agent nodes.  
    - Configuration: Model "models/gemini-2.0-flash" with max 5000 tokens.  
    - Connections: Provides language model to AI Agent, AI Agent1, AI Agent2.  
    - Edge Cases: Same as other Gemini model nodes.

  - **AI Agent / AI Agent1 / AI Agent2**  
    - Type: LangChain Agent  
    - Role: Each agent processes consolidated feedback from the respective sentiment category (positive, neutral, negative).  
    - Configuration:  
      - Prompt instructs to ignore "no comment" entries, convert text ratings to numeric, confirm sentiment consistency, detect trends by date.  
      - Output: A single JSON report with keys: overallSentiment, averageRating, highlights, weaknessesConcerns, timeTrend, finalSummary.  
    - Connections: Output passes to respective "Restructure the sentiment ... data to slack block" code nodes.  
    - Edge Cases: AI model misinterpretation; invalid JSON output causing downstream errors.

#### 1.6 Formatting Slack Messages

- **Overview:**  
Transforms AI-generated JSON reports into Slack Block Kit message format tailored for each sentiment category for rich display in Slack channels.

- **Nodes Involved:**  
  - Restructure the sentiment positive data to slack block  
  - Restructure the sentiment neutral data to slack block  
  - Restructure the sentiment negative data to slack block  

- **Node Details:**  

  - **Restructure the sentiment ... data to slack block (3 nodes)**  
    - Type: Code  
    - Role:  
      - Parses AI agent JSON output, removing markdown code fences.  
      - Builds Slack message blocks with sections for overall sentiment, average rating, highlights, weaknesses/concerns, time trend, and final summary.  
    - Configuration: Executes once, throws error if JSON parsing fails.  
    - Connections: Each node connects to respective "Send message to slack channel" node.  
    - Edge Cases: Invalid JSON format, missing fields, or empty data arrays.

#### 1.7 Slack Notification

- **Overview:**  
Sends the formatted Slack message blocks to a designated Slack channel based on the sentiment category.

- **Nodes Involved:**  
  - Send message to slack channel if the analysis is positive  
  - Send message to slack channel if the analysis is neutral  
  - Send message to slack channel if the analysis is negative  

- **Node Details:**  

  - **Send message to slack channel if the analysis ... (3 nodes)**  
    - Type: Slack  
    - Role: Sends Slack messages with block content for respective sentiment reports.  
    - Configuration:  
      - Channel ID set to a specific Slack channel (ID: C09DQ7EJ57A).  
      - Message type: Blocks (rich message format).  
      - Authentication: OAuth2 with Slack account credentials.  
    - Connections: Terminal nodes of the workflow branches.  
    - Edge Cases: Slack API rate limits, invalid OAuth credentials, channel permission errors.

---

### 3. Summary Table

| Node Name                                       | Node Type                       | Functional Role                             | Input Node(s)                          | Output Node(s)                                         | Sticky Note                                                                                                  |
|------------------------------------------------|--------------------------------|---------------------------------------------|---------------------------------------|-------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                                | Schedule Trigger               | Trigger workflow monthly                    | -                                     | Set time period                                        |                                                                                                              |
| Set time period                                 | Set                           | Define review extraction period             | Schedule Trigger                      | Get many reviews                                      |                                                                                                              |
| Get many reviews                                | Google Business Profile        | Retrieve Google reviews                      | Set time period                       | Filter review based on the time period                 | ## Reading Google review<br>**Reading google review from the google business profile**                        |
| Filter review based on the time period          | Filter                        | Filter reviews by date                       | Get many reviews                     | Map the comment and rating                             |                                                                                                              |
| Map the comment and rating                      | Set                           | Normalize review data fields                 | Filter review based on the time period | Sentiment Analysis                                   | ## Run sentiment analysis on google review<br>**Based on the google review data a sentiment analysis run on top of the google review using Gemini model.** |
| Sentiment Analysis                              | LangChain Sentiment Analysis  | Perform batch sentiment analysis             | Map the comment and rating           | Convert to json string, Convert to json string1, Convert to json string2 |                                                                                                              |
| Convert to json string                          | Code                          | Serialize sentiment data to JSON string     | Sentiment Analysis                   | AI Agent                                             |                                                                                                              |
| Convert to json string1                         | Code                          | Serialize sentiment data to JSON string     | Sentiment Analysis                   | AI Agent1                                            |                                                                                                              |
| Convert to json string2                         | Code                          | Serialize sentiment data to JSON string     | Sentiment Analysis                   | AI Agent2                                            |                                                                                                              |
| AI Agent                                       | LangChain Agent               | Consolidate positive sentiment reports      | Convert to json string               | Restructure the sentiment positive data to slack block | ## Summarize the sentiment analysis report using AI model<br>**After the sentiment of each review is analysed consolidating the overall sentiment and restructure the output.** |
| AI Agent1                                      | LangChain Agent               | Consolidate neutral sentiment reports       | Convert to json string1              | Restructure the sentiment neutral data to slack block |                                                                                                              |
| AI Agent2                                      | LangChain Agent               | Consolidate negative sentiment reports      | Convert to json string2              | Restructure the sentiment negative data to slack block |                                                                                                              |
| Restructure the sentiment positive data to slack block | Code                          | Format positive sentiment report for Slack | AI Agent                           | Send message to slack channel if the analysis is positive | ## Sent notification to slack channel<br>**Restructure the data to slack block and send to slack channel.**   |
| Restructure the sentiment neutral data to slack block  | Code                          | Format neutral sentiment report for Slack  | AI Agent1                          | Send message to slack channel if the analysis is neutral |                                                                                                              |
| Restructure the sentiment negative data to slack block | Code                          | Format negative sentiment report for Slack | AI Agent2                          | Send message to slack channel if the analysis is negative |                                                                                                              |
| Send message to slack channel if the analysis is positive | Slack                         | Send positive sentiment report to Slack     | Restructure the sentiment positive data to slack block | -                                                      |                                                                                                              |
| Send message to slack channel if the analysis is neutral | Slack                         | Send neutral sentiment report to Slack      | Restructure the sentiment neutral data to slack block  | -                                                      |                                                                                                              |
| Send message to slack channel if the analysis is negative | Slack                         | Send negative sentiment report to Slack     | Restructure the sentiment negative data to slack block | -                                                      |                                                                                                              |
| Google Gemini Chat Model                        | LangChain Google Gemini LM Chat | AI language model for AI Agents              | -                                     | AI Agent, AI Agent1, AI Agent2                         |                                                                                                              |
| Google Gemini Chat Model1                       | LangChain Google Gemini LM Chat | AI language model for Sentiment Analysis     | -                                     | Sentiment Analysis                                    |                                                                                                              |
| Sticky Note                                     | Sticky Note                   | Note: Reading Google review                   | -                                     | -                                                     | ## Reading Google review<br>**Reading google review from the google business profile**                        |
| Sticky Note1                                    | Sticky Note                   | Note: Run sentiment analysis                  | -                                     | -                                                     | ## Run sentiment analysis on google review<br>**Based on the google review data a sentiment analysis run on top of the google review using Gemini model.** |
| Sticky Note2                                    | Sticky Note                   | Note: Summarize sentiment report              | -                                     | -                                                     | ## Summarize the sentiment analysis report using AI model<br>**After the sentiment of each review is analysed consolidating the overall sentiment and restructure the output.** |
| Sticky Note3                                    | Sticky Note                   | Note: Send notification to Slack              | -                                     | -                                                     | ## Sent notification to slack channel<br>**Restructure the data to slack block and send to slack channel.**    |
| Sticky Note5                                    | Sticky Note                   | Overview and setup instructions                | -                                     | -                                                     | ## Google review Sentiment analysis workflow<br>**What it does** - Read Google review ...<br>**Requirement** - Google business profile with approved project ...<br>**Setup Instructions:** - Setup google business profile ... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Set to trigger every month (interval: months).  
   - Connect output to "Set time period" node.

2. **Create a Set Node named "Set time period"**  
   - Assign two variables:  
     - `period` = "year" (string)  
     - `time` = "3" (string)  
   - Set to execute once per run.  
   - Connect output to "Get many reviews" node.

3. **Create a Google Business Profile Node named "Get many reviews"**  
   - Operation: getAll reviews.  
   - Limit: 1000 results.  
   - Configure account and location by name (select from your Google Business Profile credentials).  
   - Connect output to "Filter review based on the time period" node.

4. **Create a Filter Node named "Filter review based on the time period"**  
   - Condition: filter input items where `updateTime` is after (today minus `period` and `time` values).  
   - Use expression:  
     `{{$today.minus({ [$json["period"]]: $json["time"] }).toISO()}}`  
   - Connect output to "Map the comment and rating" node.

5. **Create a Set Node named "Map the comment and rating"**  
   - Assign three fields with expressions:  
     - `Comment` = `{{$json.comment ?? "no comment"}}`  
     - `Raiting` = `{{$json.starRating ?? "0"}}`  
     - `Date` = `{{$json.updateTime}}`  
   - Connect output to "Sentiment Analysis" node.

6. **Create a LangChain Google Gemini LM Chat Node named "Google Gemini Chat Model1"**  
   - Model: `models/gemini-2.0-flash`  
   - Max output tokens: 5000  
   - Attach Google Palm API credentials.  
   - Connect as AI language model to the "Sentiment Analysis" node.

7. **Create a LangChain Sentiment Analysis Node named "Sentiment Analysis"**  
   - Set batch size to 20 and delay between batches to 60000 ms.  
   - Categories: `Positive, Neutral, Negative`  
   - Enable auto-fixing.  
   - System prompt template: (copy from overview, describing sentiment rules and tasks)  
   - Input text:  
     `={{ $json.Comment }}{{ $json.Raiting }}{{ $json.Date }}`  
   - Connect output to three parallel Code nodes: "Convert to json string", "Convert to json string1", "Convert to json string2".

8. **Create three Code Nodes named "Convert to json string", "Convert to json string1", and "Convert to json string2"**  
   - Code for each: serialize all input JSON items into a single JSON string under `feedback` key.  
   - Execute once per run.  
   - Connect each to a respective LangChain Agent node: "AI Agent", "AI Agent1", "AI Agent2".

9. **Create one LangChain Google Gemini LM Chat Node named "Google Gemini Chat Model"**  
   - Model: `models/gemini-2.0-flash`  
   - Max output tokens: 5000  
   - Attach Google Palm API credentials.  
   - Connect as AI language model to the three AI Agent nodes.

10. **Create three LangChain Agent Nodes named "AI Agent", "AI Agent1", "AI Agent2"**  
    - Prompt text (same for all): instruct AI to consolidate feedback JSON into one report ignoring "no comment", convert ratings to numeric, consider sentiment and date trends, output valid JSON with overallSentiment, averageRating, highlights, weaknessesConcerns, timeTrend, finalSummary.  
    - Connect each to corresponding Code node for restructuring Slack blocks.

11. **Create three Code Nodes named "Restructure the sentiment positive/neutral/negative data to slack block"**  
    - Code to:  
      - Remove markdown code fences from AI JSON output.  
      - Parse JSON.  
      - Build Slack blocks with sections for overall sentiment, average rating, highlights, weaknesses, time trend, final summary.  
      - Return Slack blocks JSON.  
    - Connect each to respective Slack message node.

12. **Create three Slack Nodes named "Send message to slack channel if the analysis is positive/neutral/negative"**  
    - Select message type "Blocks".  
    - Channel ID: set to your Slack channel (example: C09DQ7EJ57A).  
    - Authentication: OAuth2 with Slack OAuth credentials.  
    - Use blocks UI expression:  
      `={{ '{ "blocks": ' + JSON.stringify($json.blocks) + ' }' }}`  
    - These are terminal nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow requires a Google Business Profile with enabled API access and appropriate OAuth credentials.      | Setup instruction for Google Business Profile API from Google Cloud Console documentation.     |
| Gemini model credentials (Google Palm API) must be correctly configured for sentiment analysis and AI agents.  | See Google Cloud AI and Gemini API setup guides.                                               |
| Slack OAuth2 credentials must be configured with permission to post messages to the specified Slack channel.    | Slack App configuration and OAuth2 setup best practices.                                       |
| The workflow uses batch processing with delay to avoid API rate limits on Google Gemini language model.          | Adjust batch size and delay based on API quota and performance.                                |
| The typo in the field name `Raiting` is consistent across nodes and should be corrected if modifying workflow. | To avoid confusion or errors in downstream processing, rename to `Rating` and update expressions.|
| Sticky notes in the workflow provide concise explanations and setup instructions within the n8n editor interface.| Use these for quick reference and user onboarding.                                             |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.