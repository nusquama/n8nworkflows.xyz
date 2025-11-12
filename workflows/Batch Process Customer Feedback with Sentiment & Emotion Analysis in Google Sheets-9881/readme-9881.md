Batch Process Customer Feedback with Sentiment & Emotion Analysis in Google Sheets

https://n8nworkflows.xyz/workflows/batch-process-customer-feedback-with-sentiment---emotion-analysis-in-google-sheets-9881


# Batch Process Customer Feedback with Sentiment & Emotion Analysis in Google Sheets

### 1. Workflow Overview

This workflow automates the processing of customer feedback stored in Google Sheets by applying AI-powered tagging, sentiment, and emotion analysis using OpenAI. It targets teams managing user feedback who want to categorize comments into meaningful tags, understand overall sentiment, and detect emotional nuances for better product and support insights.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception:** Triggers and sheet data fetching (manual or scheduled trigger; fetch allowed tags and new feedback).
- **1.2 Data Preparation:** Merging tag and feedback data, attaching tags array, and batching feedback items.
- **1.3 AI Processing:** Aggregating batch items and sending them to OpenAI for tagging, sentiment, and emotion analysis.
- **1.4 Result Handling:** Splitting AI response into individual feedback records and updating the Google Sheet with tags and analysis.
- **1.5 Workflow Control:** Batch processing control and looping through all feedback entries without exceeding API limits.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow either manually or automatically on schedule and fetches required data from Google Sheets: allowed tags and new, unprocessed feedback entries.

**Nodes Involved:**  
- When clicking 'Execute workflow' (Manual Trigger)  
- Schedule Trigger  
- Fetch Allowed Tags (Google Sheets)  
- Fetch New Feedbacks (Google Sheets)

**Node Details:**  

- **When clicking 'Execute workflow'**  
  - Type: Manual Trigger  
  - Role: Starts workflow on user demand for testing or ad-hoc runs.  
  - Configuration: Default manual trigger with no parameters.  
  - Input: None  
  - Output: Triggers Fetch Allowed Tags and Fetch New Feedbacks nodes.  
  - Edge Cases: No input failures; manual runs depend on user execution.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers workflow every 60 minutes for continuous processing.  
  - Configuration: Interval set to 60 minutes.  
  - Input: None  
  - Output: Triggers Fetch Allowed Tags and Fetch New Feedbacks nodes.  
  - Edge Cases: Scheduler downtime or misconfiguration could delay runs.

- **Fetch Allowed Tags**  
  - Type: Google Sheets (Read)  
  - Role: Reads the list of allowed tags from a dedicated "Tags" sheet column for later AI matching.  
  - Configuration: Sheet named "Tags", column "Tags" expected; document ID and OAuth2 credentials set.  
  - Key Expressions: Uses Sheet URL parameterized by `GOOGLE_SHEETS_URL`.  
  - Input: Trigger from Manual or Schedule Trigger  
  - Output: Provides allowed tags array to Merge Tags & Feedbacks node.  
  - Edge Cases: Authorization errors, missing sheet or column, empty tag list.

- **Fetch New Feedbacks**  
  - Type: Google Sheets (Read)  
  - Role: Reads feedback entries from "Feedbacks" sheet where `Status` column is empty (new feedback).  
  - Configuration: Sheet named "Feedbacks", filter on empty `Status`; OAuth2 credentials set.  
  - Input: Trigger from Manual or Schedule Trigger  
  - Output: Provides new feedback entries to Merge Tags & Feedbacks node.  
  - Edge Cases: Authorization errors, missing expected columns, empty feedback set.

---

#### 2.2 Data Preparation

**Overview:**  
Combines fetched tags and feedbacks, enriches feedback data with tag lists, and batches the feedback for efficient AI processing.

**Nodes Involved:**  
- Merge Tags & Feedbacks (Merge)  
- Attach Tags Array (Set)  
- Process Feedbacks in Batches (SplitInBatches)

**Node Details:**  

- **Merge Tags & Feedbacks**  
  - Type: Merge  
  - Role: Combines two input streams — the feedback list and the allowed tags — so each feedback item can access the full tags array.  
  - Configuration: Mode set to "Choose Branch" to keep feedback data while waiting for tags.  
  - Input: Feedbacks (Input 1), Allowed Tags (Input 2)  
  - Output: Merged items where feedbacks contain the allowed tags.  
  - Edge Cases: Timing issues if one input delayed; empty inputs propagate downstream.

- **Attach Tags Array**  
  - Type: Set  
  - Role: Adds a new field `tags` to each feedback item containing the array of allowed tags for AI reference.  
  - Configuration: Sets `tags` field from the allowed tags fetched previously using expression mapping.  
  - Key Expressions: `={{ $items('Fetch Allowed Tags').map(i => i.json['Tags']) }}`  
  - Input: From Merge Tags & Feedbacks  
  - Output: Feedback items with `tags` field attached.  
  - Edge Cases: Empty tags array causes AI to fall back to AI-generated tags.

- **Process Feedbacks in Batches**  
  - Type: SplitInBatches  
  - Role: Splits large feedback lists into smaller batches (default size 10) to manage API call load.  
  - Configuration: Batch size set via expression to 10.  
  - Input: From Attach Tags Array  
  - Output: Processes batches sequentially with two outputs: empty output for loop continue, main output for batch processing.  
  - Edge Cases: Large batch sizes may cause timeouts or API limits; too small batch sizes increase API calls.

---

#### 2.3 AI Processing

**Overview:**  
Aggregates the batch items into a single payload and sends them to OpenAI for tagging, sentiment, and emotion analysis based on a detailed prompt.

**Nodes Involved:**  
- Aggregate Batch Items (Code)  
- Tag Feedbacks with AI (OpenAI)

**Node Details:**  

- **Aggregate Batch Items**  
  - Type: Code (JavaScript)  
  - Role: Consolidates all feedback items in the batch into a single JSON object to send as one request.  
  - Configuration: Custom JS code maps input items into `{row_number, text}` objects and extracts the tags array for AI input.  
  - Input: From Process Feedbacks in Batches (batch output)  
  - Output: Single JSON item with `feedbackBatch` array and `tags` array.  
  - Edge Cases: Code errors parsing input data; empty batch.

- **Tag Feedbacks with AI**  
  - Type: OpenAI (LangChain node)  
  - Role: Sends batch feedback and allowed tags to OpenAI GPT-4o-mini model with a comprehensive system prompt to auto-tag, analyze sentiment, and detect emotions.  
  - Configuration:  
    - Model: GPT-4o-mini  
    - System prompt: Detailed rules for matching allowed tags first, fallback AI tags, sentiment scale, emotion labeling, multilingual support.  
    - Input message: JSON string containing `allowed_tags` and `feedbacks`.  
  - Input: Aggregated batch JSON  
  - Output: AI response as a JSON array with tagging results per feedback.  
  - Edge Cases: API key errors, rate limits, malformed responses, prompt failure, multilingual edge cases.

---

#### 2.4 Result Handling

**Overview:**  
Parses the AI response JSON back into individual feedback items and updates the Google Sheet with tagging and analysis results.

**Nodes Involved:**  
- Split Batch Results (Code)  
- Update Google Sheet (Tagged) (Google Sheets)

**Node Details:**  

- **Split Batch Results**  
  - Type: Code (JavaScript)  
  - Role: Parses the OpenAI JSON response and maps each tagged feedback into individual items with fields for tags, sentiment, emotions, and row number.  
  - Configuration: Parses JSON from AI response, supports array or object with results property, maps keys safely.  
  - Input: From Tag Feedbacks with AI  
  - Output: Individual feedback records for update.  
  - Edge Cases: Malformed AI response, parsing errors.

- **Update Google Sheet (Tagged)**  
  - Type: Google Sheets (Update)  
  - Role: Updates the original Feedbacks sheet rows with AI-generated tags, sentiment, emotions, status, and timestamp.  
  - Configuration:  
    - Sheet: "Feedbacks"  
    - Matching column: `row_number`  
    - Columns updated: Tag 1-3, AI Tag 1-2, Sentiment, Primary Emotion, Secondary Emotion, Status ("Updated" or "Needs Review"), Updated Date (ISO Date)  
    - OAuth2 credentials configured.  
  - Input: From Split Batch Results  
  - Output: Feeds back to Process Feedbacks in Batches to continue processing next batch.  
  - Edge Cases: Update conflicts, missing columns, auth failures.

---

#### 2.5 Workflow Control

**Overview:**  
Controls batch iteration and loops the workflow until all new feedback rows are processed.

**Nodes Involved:**  
- Process Feedbacks in Batches (SplitInBatches)  
- Update Google Sheet (Tagged) (feeds back to batch node)

**Node Details:**  

- **Process Feedbacks in Batches** (already described)  
  - Controls batch size and looping through all items until none remain unprocessed.  
  - Receives feedback items from Update Google Sheet to continue next batch.

- **Update Google Sheet (Tagged)** (already described)  
  - Emits output that loops back to batch node to fetch next batch or end.

---

### 3. Summary Table

| Node Name                    | Node Type                | Functional Role                              | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                                                |
|------------------------------|--------------------------|----------------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| When clicking 'Execute workflow' | Manual Trigger           | Manual start of workflow for testing         | None                             | Fetch Allowed Tags, Fetch New Feedbacks | 🖱️ Manual Trigger: Run workflow manually for testing small batches.                                                        |
| Schedule Trigger              | Schedule Trigger          | Automatic periodic workflow start (60 min)  | None                             | Fetch Allowed Tags, Fetch New Feedbacks | ⏰ Schedule Trigger: Runs every 60 minutes automatically.                                                                    |
| Fetch Allowed Tags            | Google Sheets (Read)      | Reads allowed tags from Tags sheet            | Manual Trigger, Schedule Trigger | Merge Tags & Feedbacks (Input 2) | 🏷️ Fetch Allowed Tags: Reads available tag names from Tags sheet column "Tags".                                            |
| Fetch New Feedbacks           | Google Sheets (Read)      | Reads new feedback (Status empty) from sheet | Manual Trigger, Schedule Trigger | Merge Tags & Feedbacks (Input 1) | 💬 Fetch New Feedbacks: Reads feedbacks with empty Status from Feedbacks sheet.                                             |
| Merge Tags & Feedbacks        | Merge                    | Combines feedbacks and allowed tags           | Fetch New Feedbacks, Fetch Allowed Tags | Attach Tags Array               | 🔗 Merge Tags & Feedbacks: Combines feedback and tags to allow each feedback access to full tag list.                       |
| Attach Tags Array             | Set                      | Adds tags array field to feedback items       | Merge Tags & Feedbacks            | Process Feedbacks in Batches     | 🧩 Attach Tags Array: Adds tags array to each feedback item for AI usage.                                                   |
| Process Feedbacks in Batches  | SplitInBatches            | Splits feedback list into batches for AI      | Attach Tags Array, Update Google Sheet (Tagged) | Aggregate Batch Items (main output) | 🔁 Split In Batches: Processes feedback in batches (default 10) to avoid rate limits.                                       |
| Aggregate Batch Items         | Code                     | Aggregates batch items into single JSON payload | Process Feedbacks in Batches      | Tag Feedbacks with AI            | 📦 Aggregate Batch Items: Combines batch feedback into one request to OpenAI for cost savings.                              |
| Tag Feedbacks with AI         | OpenAI (LangChain)        | Sends batch to OpenAI for tagging, sentiment, emotion | Aggregate Batch Items            | Split Batch Results              | 🤖 Tag Feedbacks with OpenAI: Sends batch to AI for tagging with allowed tags preference and fallback AI tags.             |
| Split Batch Results           | Code                     | Parses AI response and splits into feedback items | Tag Feedbacks with AI             | Update Google Sheet (Tagged)     | 🔀 Split Batch Results: Parses AI response into individual tagged feedback items.                                          |
| Update Google Sheet (Tagged) | Google Sheets (Update)    | Updates feedback rows with tags, sentiment, emotions | Split Batch Results               | Process Feedbacks in Batches     | 📊 Update Google Sheet (Tagged): Writes tags, sentiment, emotions, status, and timestamp back to Feedbacks sheet.         |
| Sticky Note                  | Sticky Note               | Documentation and explanation notes           | None                             | None                          | Provides detailed workflow overview, usage instructions, and customization tips.                                            |
| Sticky Note2                 | Sticky Note               | Explains Fetch Allowed Tags node               | None                             | None                          | Explains allowed tags fetching and sheet requirements.                                                                      |
| Sticky Note3                 | Sticky Note               | Explains Fetch New Feedbacks node              | None                             | None                          | Details expected feedback sheet columns and status values.                                                                  |
| Sticky Note4                 | Sticky Note               | Explains Tag Feedbacks with AI node            | None                             | None                          | Details AI tagging rules, output format, and multilingual support.                                                          |
| Sticky Note5                 | Sticky Note               | Explains Update Google Sheet node               | None                             | None                          | Describes columns updated in the Feedbacks sheet.                                                                            |
| Sticky Note6                 | Sticky Note               | Explains Split In Batches node                   | None                             | None                          | Explains batch processing role and default batch size.                                                                      |
| Sticky Note7                 | Sticky Note               | Explains Merge Tags & Feedbacks node            | None                             | None                          | Explains merging of allowed tags with feedback data.                                                                        |
| Sticky Note8                 | Sticky Note               | Explains Attach Tags Array node                  | None                             | None                          | Describes adding tags array to each feedback item.                                                                           |
| Sticky Note9                 | Sticky Note               | Explains Manual Trigger node                      | None                             | None                          | Explains manual execution for testing.                                                                                       |
| Sticky Note10                | Sticky Note               | Explains Schedule Trigger node                    | None                             | None                          | Explains scheduled runs every 60 minutes.                                                                                    |
| Sticky Note11                | Sticky Note               | Credits the workflow author                        | None                             | None                          | Created by: Parhum Khoshbakht, LinkedIn profile link included.                                                               |
| Sticky Note12                | Sticky Note               | Explains Aggregate Batch Items node               | None                             | None                          | Describes aggregation of batch items into one API call for cost savings.                                                    |
| Sticky Note13                | Sticky Note               | Explains Split Batch Results node                  | None                             | None                          | Describes parsing AI response and splitting into individual feedback updates.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Name: `When clicking 'Execute workflow'`  
   - No parameters, used to manually run workflow.

2. **Create Schedule Trigger node:**  
   - Name: `Schedule Trigger`  
   - Set interval rule to every 60 minutes.

3. **Create Google Sheets node `Fetch Allowed Tags`:**  
   - Operation: Read  
   - Document ID: Set your Google Sheets URL (parameterized or fixed)  
   - Sheet Name: `Tags`  
   - Read rows from column named `Tags`  
   - Credentials: Google Sheets OAuth2  
   - Output: Allowed tags list

4. **Create Google Sheets node `Fetch New Feedbacks`:**  
   - Operation: Read  
   - Document ID: Same Google Sheets URL  
   - Sheet Name: `Feedbacks`  
   - Filter: Only rows where `Status` column is empty  
   - Credentials: Google Sheets OAuth2  
   - Output: New feedback entries

5. **Connect `When clicking 'Execute workflow'` and `Schedule Trigger` to both `Fetch Allowed Tags` and `Fetch New Feedbacks`.**

6. **Create Merge node `Merge Tags & Feedbacks`:**  
   - Mode: Choose Branch  
   - Connect Input 1 to `Fetch New Feedbacks`  
   - Connect Input 2 to `Fetch Allowed Tags`  
   - Output: Merged data with feedback and tags accessible

7. **Create Set node `Attach Tags Array`:**  
   - Add field `tags` (type string array)  
   - Value expression: `{{$items('Fetch Allowed Tags').map(i => i.json['Tags'])}}`  
   - Input: From `Merge Tags & Feedbacks` node

8. **Create SplitInBatches node `Process Feedbacks in Batches`:**  
   - Batch size: 10 (or configurable)  
   - Input: From `Attach Tags Array` node

9. **Create Code node `Aggregate Batch Items`:**  
   - JavaScript code to map batch items into one JSON object containing:  
     ```js
     const feedbackBatch = $input.all().map((item, index) => ({
       id: index,
       row_number: item.json.row_number,
       text: item.json.Feedbacks
     }));

     const tags = $input.first().json.tags;

     return {
       json: {
         feedbackBatch,
         tags
       }
     };
     ```  
   - Input: From main output of `Process Feedbacks in Batches`

10. **Create OpenAI node `Tag Feedbacks with AI`:**  
    - Model: GPT-4o-mini  
    - Credentials: OpenAI API key  
    - Messages (system): Use detailed system prompt specifying tagging rules, sentiment, emotion, multilingual support, JSON-only output format (copy the prompt from node configuration).  
    - Messages (user): Pass JSON string with keys `allowed_tags` and `feedbacks` from aggregated batch item.  
    - Input: From `Aggregate Batch Items`

11. **Create Code node `Split Batch Results`:**  
    - JavaScript code to parse AI response JSON and map to individual items:  
      ```js
      const response = JSON.parse($input.first().json.message.content);
      const results = Array.isArray(response) ? response : response.results || [];
      return results.map(result => ({
        json: {
          row_number: result.row_number,
          tags: result.tags || [],
          sentiment: result.sentiment || "",
          ai_tag_1: result.ai_tag_1 || "",
          ai_tag_2: result.ai_tag_2 || "",
          primary_emotion: result.primary_emotion || "",
          secondary_emotion: result.secondary_emotion || ""
        }
      }));
      ```  
    - Input: From `Tag Feedbacks with AI`

12. **Create Google Sheets node `Update Google Sheet (Tagged)`:**  
    - Operation: Update  
    - Document ID: Same Google Sheets URL  
    - Sheet Name: `Feedbacks`  
    - Matching column: `row_number`  
    - Columns to update: `Tag 1`, `Tag 2`, `Tag 3`, `AI Tag 1`, `AI Tag 2`, `Sentiment`, `Primary Emotion`, `Secondary Emotion`, `Status`, `Updated Date (N8N)`  
    - Mapping expressions:  
      - `Tag 1`: `={{ $json.tags[0] || '' }}`  
      - `Tag 2`: `={{ $json.tags[1] || '' }}`  
      - `Tag 3`: `={{ $json.tags[2] || '' }}`  
      - `Status`: `={{ $json.tags && $json.tags.length > 0 ? 'Updated' : 'Needs Review' }}`  
      - `AI Tag 1`: `={{ $json.ai_tag_1 }}`  
      - `AI Tag 2`: `={{ $json.ai_tag_2 }}`  
      - `Sentiment`: `={{ $json.sentiment }}`  
      - `Primary Emotion`: `={{ $json.primary_emotion }}`  
      - `Secondary Emotion`: `={{ $json.secondary_emotion }}`  
      - `Updated Date (N8N)`: `={{ new Date().toISOString().split('T')[0] }}`  
    - Credentials: Google Sheets OAuth2  
    - Input: From `Split Batch Results`

13. **Connect `Update Google Sheet (Tagged)` output back to `Process Feedbacks in Batches` to continue processing remaining batches.**

14. **Add Sticky Notes:**  
    - Add descriptive sticky notes at key positions to document purpose, usage, and configuration tips (content available from workflow notes).

15. **Set credentials for Google Sheets OAuth2 and OpenAI API with valid accounts.**

16. **Test the workflow manually and then enable schedule trigger for automation.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Duplicate the provided [Google Sheet template](https://docs.google.com/spreadsheets/d/1y7B3u5vgQLDidf-NdfPgAiBuQxP9Qa7RvdgsTPG14Fs/edit?usp=sharing) before using.                                                                                                | Setup instruction in sticky note and workflow description.                                              |
| The system prompt in the OpenAI node is critical to controlling tagging logic and should be adapted carefully to maintain the two-tier tagging system (allowed tags first, AI-generated tags fallback).                                                             | System prompt content in "Tag Feedbacks with AI" node.                                                   |
| Batch size can be adjusted in the SplitInBatches node to optimize performance and API cost. Default is 10 items per batch.                                                                                                                                          | Sticky Note6 explains batch processing rationale.                                                       |
| This workflow supports multilingual feedback, including Persian/Farsi, as instructed in the AI prompt.                                                                                                                                                             | Described in AI prompt in "Tag Feedbacks with AI" node.                                                 |
| Update your Google Sheets URL in all relevant Google Sheets nodes to ensure correct document access.                                                                                                                                                               | Parameter placeholder: `=GOOGLE_SHEETS_URL` in nodes Fetch Allowed Tags, Fetch New Feedbacks, Update.   |
| Created by: Parhum Khoshbakht, Product Manager & Leadership Coach — [LinkedIn](https://www.linkedin.com/in/parhumm/)                                                                                                                                               | Workflow author credit in Sticky Note11.                                                                |
| Consider extending this workflow to push tagged feedback results to other tools such as Notion, Slack, or Airtable for real-time stakeholder reporting.                                                                                                          | Suggested in Sticky Note1 (overview).                                                                    |
| Ensure your Google Sheets columns are named exactly as expected: Tags sheet with `Tags` column; Feedbacks sheet with `Feedbacks`, `Status`, `Tag 1`, `Tag 2`, `Tag 3`, `AI Tag 1`, `AI Tag 2`, `Sentiment`, `Primary Emotion`, `Secondary Emotion`, `Updated Date (N8N)`, `row_number`. | Critical for correct data mapping and update.                                                           |

---

**Disclaimer:**  
The provided content is exclusively derived from an n8n workflow automation configuration. It complies strictly with content policies and contains no illegal or protected material. All data processed is lawful and publicly accessible.