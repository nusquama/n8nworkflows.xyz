Meta Ads Performance Analysis with GPT-4 & Gemini AI Comparisons

https://n8nworkflows.xyz/workflows/meta-ads-performance-analysis-with-gpt-4---gemini-ai-comparisons-6545


# Meta Ads Performance Analysis with GPT-4 & Gemini AI Comparisons

### 1. Workflow Overview

This workflow, titled **Meta Ads Performance Analysis with GPT-4 & Gemini AI Comparisons**, automates a comprehensive analysis of Meta (Facebook) ads performance data using advanced AI models—OpenAI GPT-4.1-NANO and Google Gemini. Its primary purpose is to fetch, prepare, and analyze ad creative performance data, compare it against historical benchmarks, and provide actionable recommendations for scaling, optimizing, or stopping ads.

The workflow is designed for two main use cases:  
- **Analyzing ads directly from Meta campaigns by specifying a campaign ID**  
- **Analyzing ads listed in a Google Sheet**, allowing integration with upstream workflows or manual input

The logical blocks of the workflow are:

- **1.1 Input Reception and Configuration:** Determines data source and initializes parameters.  
- **1.2 Data Acquisition:** Retrieves Meta ads and insights or reads ads from a Google Sheet.  
- **1.3 Data Preparation:** Splits, merges, and formats raw data, extracting key metrics into structured formats and CSV strings for AI consumption.  
- **1.4 AI Analysis:** Sends prepared data to OpenAI GPT-4.1-NANO and Google Gemini models for independent evaluation and recommendation generation.  
- **1.5 Post-Processing and Storage:** Parses AI outputs, updates Google Sheets with raw metrics and AI evaluations, and prepares final data outputs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Configuration

- **Overview:**  
  This block initializes the workflow on schedule trigger, sets configuration parameters such as data source type and benchmarks data, and routes execution based on the chosen source (`Meta` or `Sheets`).

- **Nodes Involved:**  
  - Schedule Trigger  
  - Set parameters  
  - If

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Trigger — Scheduled start of the workflow (interval-based, default daily).  
    - Configuration: Runs automatically on a scheduled interval.  
    - Outputs: Triggers the next node `Set parameters`.  
    - Edge cases: Misconfigured schedule can cause no runs or excessive runs.

  - **Set parameters**  
    - Type: Set — Defines initial variables.  
    - Configuration:  
      - `source`: "Sheets" or "Meta" (string).  
      - `campaign_id`: Campaign ID string (empty by default).  
      - `benchmarks_data`: CSV string containing benchmark metrics for top-performing creatives.  
    - Purpose: Central control for data source and benchmark reference.  
    - Outputs: Passes parameters to the conditional node `If`.  
    - Edge cases: Blank or invalid campaign ID when `source` is `Meta` will result in no data fetched.

  - **If**  
    - Type: Conditional — Checks if `source` equals `"Meta"`.  
    - Configuration: Uses strict string equality on `$json.source`.  
    - Behavior:  
      - If true: routes to `Get Ads` (fetch Meta ads by campaign).  
      - If false: routes to `Get Ads from Sheet`.  
    - Edge cases: Case sensitivity or unexpected source values may block proper routing.

#### 2.2 Data Acquisition

- **Overview:**  
  Retrieves ads and performance insights either from Meta’s Graph API or a Google Sheet, depending on the source.

- **Nodes Involved:**  
  - Get Ads (Meta Graph API)  
  - Ads Split Out  
  - Get Insights (Meta Graph API)  
  - Insights Split Out  
  - Get Ads from Sheet (Google Sheets)  
  - Get Ad Details (Meta Graph API)

- **Node Details:**  

  - **Get Ads**  
    - Type: Facebook Graph API — Fetches ads under a campaign ID.  
    - Config:  
      - Node: campaign_id (dynamic from `Set parameters`)  
      - Edge: "ads"  
      - Fields: ad id, name, adset_id, creative details including image/video URLs  
      - API Version: v22.0  
      - Credentials: Facebook Graph API (OAuth)  
    - Outputs: Returns ads data JSON array under `data` field. Passes to `Ads Split Out`.  
    - Edge cases: Auth token expiry, invalid campaign ID, API rate limits.

  - **Ads Split Out**  
    - Type: Split Out — Separates array in `data` into individual items for processing.  
    - FieldToSplitOut: "data".  
    - Outputs: Single ad objects to `Get Insights`.  
    - Edge cases: Empty or malformed `data`.

  - **Get Insights**  
    - Type: Facebook Graph API — Fetches performance metrics for each ad.  
    - Config:  
      - Node: ad id from previous node  
      - Edge: "insights"  
      - Fields: spend, impressions, clicks, reach, frequency, actions, video watch metrics etc.  
      - API Version: v22.0  
      - Credentials: Facebook Graph API OAuth  
    - Outputs: Insight data array under `data`. Passes to `Insights Split Out`.  
    - Edge cases: API throttling, missing metrics, incomplete data.

  - **Insights Split Out**  
    - Type: Split Out — Separates insights array into single records.  
    - FieldToSplitOut: "data".  
    - Outputs: Single insight objects to `Merge`.  
    - Edge cases: Empty insights.

  - **Get Ads from Sheet**  
    - Type: Google Sheets — Reads ad IDs from a configured sheet.  
    - Config:  
      - Document ID and Sheet ID point to a specific Google Sheet holding ads for analysis.  
      - Credentials: Google Sheets OAuth2  
    - Outputs: Passes ad IDs to `Get Ad Details`.  
    - Edge cases: Sheet permissions, empty or malformed rows.

  - **Get Ad Details**  
    - Type: Facebook Graph API — Fetches detailed ad creative info for each AdID from the sheet.  
    - Config:  
      - Node: dynamic AdID from sheet  
      - Fields: id, name, adset_id, creative info similar to `Get Ads` node  
      - Credentials: Facebook Graph API OAuth  
    - Outputs: Passes to `Get Insights` for metrics retrieval.  
    - Edge cases: Invalid AdIDs, API errors.

#### 2.3 Data Preparation

- **Overview:**  
  Processes and merges ad creative data and insights, extracts key metrics for analysis, and formats data into CSV suitable for AI models.

- **Nodes Involved:**  
  - Merge  
  - Set Metrics  
  - Prepare CSV for OpenAI  
  - Prepare CSV for Gemini  
  - Ad metrics

- **Node Details:**  

  - **Merge**  
    - Type: Merge — Combines ad details and insights based on `id` and `ad_id`.  
    - Mode: Combine (join) by fields: `id` = `ad_id`.  
    - Outputs: Merged JSON with ad creative and insight metrics.  
    - Edge cases: Mismatched IDs causing missing data.

  - **Set Metrics**  
    - Type: Set — Extracts and assigns key performance metrics for analysis.  
    - Assigns: spend, impressions, link clicks, app installs, registrations, add to cart, checkouts initiated, purchases, and their corresponding values.  
    - Uses expressions to safely extract metrics from nested arrays, defaulting to 0 if missing.  
    - Also extracts ad identifiers and creative media URLs.  
    - Outputs: Ready-to-analyze JSON to CSV preparation nodes.  
    - Edge cases: Missing or malformed action arrays.

  - **Prepare CSV for OpenAI**  
    - Type: Code — Converts JSON metrics into CSV string format for OpenAI input.  
    - Logic: Defines CSV headers, maps JSON fields into CSV rows, escapes quotes in ad names.  
    - Outputs: Adds field `creative_csv_data` with CSV string.  
    - Edge cases: Special characters in names, missing metrics.

  - **Prepare CSV for Gemini**  
    - Type: Code — Same logic as above but for Google Gemini input.  
    - Outputs: CSV string for Gemini AI.  
    - Edge cases: Same as OpenAI CSV preparation.

  - **Ad metrics**  
    - Type: Google Sheets — Appends or updates raw metrics and creative info into a Google Sheet for record keeping.  
    - Fields: AdID, type, image (video or image), spend, value, creative ID, clicks, impressions, and funnel metrics.  
    - Credentials: Google Sheets OAuth2.  
    - Edge cases: Sheet write permissions, data schema mismatches.

#### 2.4 AI Analysis

- **Overview:**  
  Sends prepared CSV data and benchmarks to two AI models (OpenAI GPT-4.1-NANO and Google Gemini) for independent ad creative analysis and recommendations.

- **Nodes Involved:**  
  - Send data to 4.1-NANO (OpenAI)  
  - Split Out (to handle AI outputs)  
  - Send data to Gemini (Google Gemini agent)  
  - Structured Output Parser (parses Gemini JSON output)  
  - Google Gemini Chat Model (language model node feeding Gemini agent)

- **Node Details:**  

  - **Send data to 4.1-NANO**  
    - Type: OpenAI node using GPT-4.1-NANO model.  
    - Configuration:  
      - System prompt: Detailed instructions for the AI to analyze creatives against benchmarks using funnel metrics.  
      - Input: Benchmarks CSV + prepared creative CSV.  
      - Output: JSON array of objects containing evaluation, significance, summary, and recommendation.  
    - Outputs: Passes to `Split Out` to separate each creative’s result.  
    - Edge cases: API rate limits, token limits, prompt errors.

  - **Split Out**  
    - Type: Split Out — separates AI response array into single creative results.  
    - Outputs: To `Ad data from OpenAI` (Google Sheets storage).  
    - Edge cases: Empty or malformed AI response.

  - **Send data to Gemini**  
    - Type: Langchain agent node configured to use Google Gemini API.  
    - Configuration:  
      - Text prompt: Similar detailed instructions as OpenAI prompt, with metric formulas and evaluation rules.  
      - Input: Benchmarks CSV and creative CSV.  
      - Output parser enabled with JSON schema example.  
    - Outputs: Parsed JSON creative analysis to `Ad data from Gemini`.  
    - Edge cases: API errors, parsing failures.

  - **Structured Output Parser**  
    - Type: Langchain output parser — validates and structures Gemini’s JSON output.  
    - Configuration: JSON schema example specifying fields like ad_id, evaluation, significance, summary, and recommendation.  
    - Edge cases: Parsing errors if AI returns invalid JSON.

  - **Google Gemini Chat Model**  
    - Type: Langchain language model node — used internally by the Gemini agent.  
    - Configuration: Temperature 0.4 for balanced creativity and accuracy.  
    - Edge cases: API timeouts, rate limits.

#### 2.5 Post-Processing and Storage

- **Overview:**  
  Stores AI-generated analyses back into Google Sheets and finalizes data outputs for reporting or further workflows.

- **Nodes Involved:**  
  - Ad data from OpenAI (Google Sheets)  
  - Ad data from Gemini (Google Sheets)

- **Node Details:**  

  - **Ad data from OpenAI**  
    - Type: Google Sheets — Updates the sheet with OpenAI analysis results per ad.  
    - Columns mapped: ad_id, summary, evaluation, significance, recommendation.  
    - Credentials: Google Sheets OAuth2.  
    - Edge cases: Sheet write failures, data overwrites.

  - **Ad data from Gemini**  
    - Type: Google Sheets — Updates the sheet with Gemini analysis results per ad, including evaluation, significance, summary, recommendation.  
    - Credentials: Google Sheets OAuth2.  
    - Edge cases: Same as OpenAI sheet node.

---

### 3. Summary Table

| Node Name              | Node Type                        | Functional Role                                 | Input Node(s)                  | Output Node(s)                              | Sticky Note                                                                                                             |
|------------------------|---------------------------------|------------------------------------------------|-------------------------------|---------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Trigger                         | Starts workflow on schedule                      | —                             | Set parameters                              | ### Ads Performance Analysis with AI (overview, usage, flexibility)                                                     |
| Set parameters         | Set                             | Defines data source, campaign ID, benchmarks    | Schedule Trigger              | If                                          | **⚙️ 1. Main Configuration** (source selection and benchmark data guidance)                                              |
| If                     | If                              | Routes flow based on source ("Meta" or "Sheets")| Set parameters                | Get Ads / Get Ads from Sheet                 |                                                                                                                         |
| Get Ads                | Facebook Graph API              | Fetches ads for campaign                         | If                            | Ads Split Out                               |                                                                                                                         |
| Ads Split Out          | Split Out                      | Splits ads array into individual ads            | Get Ads                       | Get Insights                                | **📈 2. Metrics Preparation** (unpacking and preparing Meta API data for AI and sheets)                                  |
| Get Insights           | Facebook Graph API              | Fetches ads’ performance metrics                 | Ads Split Out / Get Ad Details| Insights Split Out                           |                                                                                                                         |
| Insights Split Out     | Split Out                      | Splits insights array into individual records   | Get Insights                  | Merge                                       |                                                                                                                         |
| Get Ads from Sheet     | Google Sheets                  | Reads AdIDs from Google Sheet                    | If                            | Get Ad Details                              |                                                                                                                         |
| Get Ad Details         | Facebook Graph API              | Fetches detailed ad info for given AdID         | Get Ads from Sheet            | Get Insights                                |                                                                                                                         |
| Merge                  | Merge                          | Combines ad details and insights by ad ID       | Insights Split Out + Ads Split Out | Prepare CSV for OpenAI / Set Metrics / Prepare CSV for Gemini |                                                                                                                         |
| Set Metrics            | Set                            | Extracts and assigns key funnel metrics          | Merge                         | Ad metrics                                  | **📈 2. Metrics Preparation**                                                                                            |
| Prepare CSV for OpenAI | Code                           | Converts JSON metrics to CSV string for OpenAI  | Merge                         | Send data to 4.1-NANO                        | **🤖 3. AI Scoring** (prompt customization encouragement)                                                                |
| Prepare CSV for Gemini | Code                           | Converts JSON metrics to CSV string for Gemini  | Merge                         | Send data to Gemini                          | **🤖 3. AI Scoring**                                                                                                     |
| Send data to 4.1-NANO  | OpenAI (GPT-4.1-NANO)          | AI analysis and recommendation generation        | Prepare CSV for OpenAI         | Split Out                                   |                                                                                                                         |
| Split Out              | Split Out                      | Splits AI array output into single creative outputs | Send data to 4.1-NANO         | Ad data from OpenAI                         |                                                                                                                         |
| Send data to Gemini    | Langchain Agent (Google Gemini)| AI analysis and recommendation generation        | Prepare CSV for Gemini         | Ad data from Gemini                          |                                                                                                                         |
| Structured Output Parser| Langchain Output Parser        | Parses Gemini JSON output                         | Send data to Gemini            | Ad data from Gemini                          |                                                                                                                         |
| Google Gemini Chat Model| Langchain Language Model       | Underlying AI model for Gemini agent             | —                             | Send data to Gemini                          |                                                                                                                         |
| Ad data from OpenAI    | Google Sheets                  | Saves OpenAI AI analysis results                  | Split Out                     | —                                           |                                                                                                                         |
| Ad data from Gemini    | Google Sheets                  | Saves Gemini AI analysis results                   | Structured Output Parser      | —                                           |                                                                                                                         |
| Ad metrics             | Google Sheets                  | Saves raw ad metrics and creative info            | Set Metrics                   | —                                           | **📈 2. Metrics Preparation**                                                                                            |
| Sticky Note            | Sticky Note                   | Overview and usage explanation                     | —                             | —                                           | ### Ads Performance Analysis with AI overview                                                                           |
| Sticky Note1           | Sticky Note                   | Main configuration instructions                    | —                             | —                                           | **⚙️ 1. Main Configuration** detailed explanation                                                                        |
| Sticky Note2           | Sticky Note                   | Metrics preparation explanation                     | —                             | —                                           | **📈 2. Metrics Preparation** details                                                                                    |
| Sticky Note3           | Sticky Note                   | AI scoring prompt guidance                           | —                             | —                                           | **🤖 3. AI Scoring** encouragement to experiment with prompts                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Scheduled Trigger node**  
   - Type: Schedule Trigger  
   - Configure interval (e.g., daily or as needed)  
   - Connect output to `Set parameters`.

2. **Create a Set node named "Set parameters"**  
   - Define variables:  
     - `source` (string, default "Sheets" or "Meta")  
     - `campaign_id` (string, optional but required if source is "Meta")  
     - `benchmarks_data` (string, paste benchmark CSV data here)  
   - Connect output to `If` node.

3. **Create an If node**  
   - Condition: `$json.source === "Meta"` (case sensitive)  
   - True output connects to `Get Ads`, false output connects to `Get Ads from Sheet`.

4. **Create Facebook Graph API node "Get Ads"**  
   - Node: dynamic `campaign_id` from `Set parameters`  
   - Edge: "ads"  
   - Fields: id, name, adset_id, creative with nested fields including video/image URLs  
   - API version: v22.0  
   - Use Facebook Graph API OAuth2 credentials.  
   - Connect output to `Ads Split Out`.

5. **Create Split Out node "Ads Split Out"**  
   - Field to split out: `data`  
   - Connect output to `Get Insights`.

6. **Create Facebook Graph API node "Get Insights"**  
   - Node: dynamic ad id from `Ads Split Out`  
   - Edge: "insights"  
   - Fields: standard ad metrics (spend, impressions, clicks, actions, video watch, etc.)  
   - API version: v22.0  
   - Connect output to `Insights Split Out`.

7. **Create Split Out node "Insights Split Out"**  
   - Field to split out: `data`  
   - Connect output to `Merge`.

8. **Create Google Sheets node "Get Ads from Sheet"**  
   - Configure with Google Sheets OAuth2 credentials.  
   - Set Document ID and Sheet ID of the ads list.  
   - Connect output to `Get Ad Details`.

9. **Create Facebook Graph API node "Get Ad Details"**  
   - Node: dynamic AdID from Google Sheet  
   - Fields: similar to `Get Ads` node  
   - Connect output to `Get Insights`.

10. **Create Merge node "Merge"**  
    - Mode: Combine (join)  
    - Merge by fields: `id` (ads) and `ad_id` (insights)  
    - Connect output to `Prepare CSV for OpenAI`, `Set Metrics`, and `Prepare CSV for Gemini`.

11. **Create Set node "Set Metrics"**  
    - Extract funnel metrics and identifiers with expressions, e.g., spend, impressions, clicks, link_click, app_installs, registrations, purchases, values, ad ids, media URLs.  
    - Connect output to `Ad metrics`.

12. **Create Code node "Prepare CSV for OpenAI"**  
    - JavaScript code to convert JSON objects into CSV string with headers: ad_name, ad_id, spend, impressions, clicks, app_installs, registrations, purchases, purchase_value.  
    - Add field `creative_csv_data`.  
    - Connect output to `Send data to 4.1-NANO`.

13. **Create Code node "Prepare CSV for Gemini"**  
    - Same logic and CSV format as OpenAI preparation node.  
    - Connect output to `Send data to Gemini`.

14. **Create OpenAI node "Send data to 4.1-NANO"**  
    - Model: GPT-4.1-NANO  
    - Max tokens: 6000  
    - Temperature: 0.4  
    - System prompt: Include detailed instructions with benchmark CSV and analysis data placeholders.  
    - Enable JSON output.  
    - Connect output to `Split Out`.

15. **Create Split Out node**  
    - Splits AI JSON array output into individual creative analyses.  
    - Connect output to `Ad data from OpenAI`.

16. **Create Langchain agent node "Send data to Gemini"**  
    - Configure with Google Gemini API credentials.  
    - Use prompt similar to OpenAI with formulas and evaluation rules.  
    - Enable output parser with JSON schema example.  
    - Connect output to `Structured Output Parser`.

17. **Create Langchain output parser node "Structured Output Parser"**  
    - Provide JSON schema example specifying ad_id, evaluation, significance, summary, recommendation.  
    - Connect output to `Ad data from Gemini`.

18. **Create Google Sheets nodes "Ad data from OpenAI" and "Ad data from Gemini"**  
    - Configure with Google Sheets OAuth2 credentials.  
    - Document and sheet IDs same as `Get Ads from Sheet`.  
    - Set mapping for ad_id, summary, evaluation, significance, recommendation.  
    - Operation: update by matching on AdID.

19. **Create Google Sheets node "Ad metrics"**  
    - Append or update raw ad metrics and creative info.  
    - Map fields as per `Set Metrics` output.  
    - Configure OAuth2 credentials.

20. **Connect all nodes as per above flow**  
    - Schedule Trigger → Set parameters → If → (Get Ads or Get Ads from Sheet) → Get Ad Details (if from Sheet) → Get Insights → Split Out → Merge → Prepare CSV nodes and Set Metrics → AI nodes → Google Sheets.

21. **Add sticky notes** to document key sections: overview, configuration, metrics preparation, AI prompt guidance.

22. **Test the workflow** with valid credentials and data. Monitor for API errors, data mismatches, or parsing issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow supports flexible data sourcing: from Meta Ads campaigns or Google Sheets for integration versatility.  | Configuration node controls source selection.                                                                           |
| Use high-quality benchmark data in CSV format for accurate AI analysis.                                               | Benchmarks data is a CSV string pasted into `Set parameters` node.                                                     |
| AI prompts are designed as detailed system messages instructing the model to act as an expert Meta Ads performance marketer. | Prompts include funnel metrics and evaluation logic to ensure relevant, actionable output.                              |
| The workflow logs raw metrics before AI analysis to preserve core data in case AI model calls fail.                   | Google Sheets nodes save raw metrics and AI results separately.                                                        |
| Adjust AI temperature and max tokens carefully to balance creativity and response length.                             | Current setting: temp 0.4, max tokens 6000 for OpenAI.                                                                  |
| Facebook Graph API uses version v22.0; ensure OAuth credentials have required permissions and tokens are valid.       |                                                                                                                        |
| Google Sheets nodes require OAuth2 credentials with edit permissions on configured spreadsheets.                      |                                                                                                                        |
| Sticky notes in the workflow provide contextual explanations for key sections: configuration, metrics, AI scoring.   |                                                                                                                        |
| AI models expect a single JSON array response with analysis results; ensure no markdown or extraneous text is returned.| Strict output format enforced by prompt to facilitate parsing.                                                          |
| The workflow is designed to be fault tolerant: if AI calls fail, raw data is still saved for manual follow-up.        |                                                                                                                        |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.