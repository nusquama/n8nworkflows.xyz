Evaluations Metric: Answer Similarity

https://n8nworkflows.xyz/workflows/evaluations-metric--answer-similarity-4423


# Evaluations Metric: Answer Similarity

### 1. Workflow Overview

This workflow, titled **"Evaluations Metric: Answer Similarity"**, is designed to automate the evaluation of AI-generated answers by comparing them to ground truth answers using similarity scoring. It targets use cases involving closed-ended or fact-based questions where assessing the closeness of an AI response to expected answers is critical — such as in model validation, quality assurance, or consistency checks.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Triggering**: Receives inputs either from a Google Sheets dataset or via chat messages to initiate evaluation.
- **1.2 AI Processing and Response Generation**: Uses an AI Language Model (OpenAI GPT-4.1-mini) through an AI Agent to generate answers.
- **1.3 Data Preparation and Embedding Generation**: Extracts ground truth and AI answers, splits ground truth into items, and generates embeddings via OpenAI embeddings API.
- **1.4 Similarity Calculation**: Calculates cosine similarity scores between the AI answer embedding and ground truth embeddings.
- **1.5 Evaluation Reporting and Metric Update**: Updates the Google Sheets dataset with the evaluation results and sets metrics for further tracking.
- **1.6 No-Operation Handling**: Ensures workflow stability by handling cases where no evaluation proceeds.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Triggering

**Overview:**  
This block triggers the workflow either by fetching rows from a Google Sheet dataset or by receiving chat messages to start AI evaluation.

**Nodes Involved:**  
- When fetching a dataset row  
- When chat message received  
- Remap Input

**Node Details:**

- **When fetching a dataset row**  
  - Type: Evaluation Trigger  
  - Role: Initiates workflow on new/updated row in Google Sheet named "Similarity" within a specific spreadsheet.  
  - Config: Connected via Google Sheets OAuth2 credentials; listens to specific sheet and document IDs.  
  - Connections: Output → Remap Input  
  - Potential Failures: OAuth token expiration, sheet access permission errors, network errors.

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Webhook-based trigger for chat messages to start the AI agent processing.  
  - Config: Uses a webhook ID, no special options set.  
  - Connections: Output → AI Agent  
  - Failures: Webhook misconfiguration, latency, malformed chat input.

- **Remap Input**  
  - Type: Set Node  
  - Role: Maps incoming data field `input` into a unified variable `chatInput` for downstream nodes.  
  - Config: Assignment of `chatInput` = `{{ $json.input }}`.  
  - Connections: Output → AI Agent  
  - Failures: Expression evaluation errors if input field missing.

---

#### 2.2 AI Processing and Response Generation

**Overview:**  
This block involves the AI Agent utilizing the OpenAI GPT-4.1-mini language model to generate answers based on the input.

**Nodes Involved:**  
- OpenAI Chat Model1  
- AI Agent  
- Evaluation

**Node Details:**

- **OpenAI Chat Model1**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Executes GPT-4.1-mini model to generate AI responses.  
  - Config: Model set specifically to "gpt-4.1-mini" with default options; uses OpenAI API credentials.  
  - Connections: Output → AI Agent (ai_languageModel input)  
  - Failures: API key invalid, rate limits, model unavailability.

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Orchestrates AI model calls and handles prompt/response interaction.  
  - Config: Default options; connects to OpenAI Chat Model1 for language model backend.  
  - Connections: Input from Remap Input and When chat message received; Output → Evaluation  
  - Failures: Agent misconfiguration, timeout, malformed input.

- **Evaluation**  
  - Type: Evaluation Node  
  - Role: Determines if workflow should proceed with evaluation (checks if evaluating).  
  - Config: Operation set to "checkIfEvaluating".  
  - Connections: Output → Set Input Fields (if evaluating) or No Operation (if not).  
  - Failures: Logic errors in evaluation conditions.

---

#### 2.3 Data Preparation and Embedding Generation

**Overview:**  
Prepares AI answer and ground truth data fields, splits multi-line ground truth into items, then generates vector embeddings for similarity comparison.

**Nodes Involved:**  
- Set Input Fields  
- Get Embeddings1 (for AI answer)  
- Get Embeddings (for ground truth items)  
- GroundTruth to Items  
- Remap Embeddings  
- Remap Embeddings1  
- Aggregate

**Node Details:**

- **Set Input Fields**  
  - Type: Set Node  
  - Role: Extracts `question`, `answer`, and splits `ground truth` from the dataset for embedding.  
  - Config:  
    - `question` = Input from first fetched dataset row's `input` field.  
    - `answer` = Current AI output.  
    - `groundTruth` = Splits `ground truth` string by newlines into an array.  
  - Connections: Output → Get Embeddings1 and GroundTruth to Items  
  - Failures: Missing fields, improper string splitting.

- **GroundTruth to Items**  
  - Type: Split Out Node  
  - Role: Converts the `groundTruth` array into separate items for individual embedding requests.  
  - Connections: Output → Get Embeddings

- **Get Embeddings1**  
  - Type: HTTP Request  
  - Role: Calls OpenAI embeddings API for the AI answer text with model `text-embedding-3-small`.  
  - Config: POST to `https://api.openai.com/v1/embeddings` with body input=`answer`.  
  - Connections: Output → Remap Embeddings  
  - Failures: API key errors, rate limits, malformed request.

- **Get Embeddings**  
  - Type: HTTP Request  
  - Role: Calls OpenAI embeddings API for each ground truth item text.  
  - Config: Same as Get Embeddings1 but input=`groundTruth` item.  
  - Connections: Output → Remap Embeddings1  
  - Failures: Same as above.

- **Remap Embeddings**  
  - Type: Set Node  
  - Role: Extracts embedding array from AI answer response JSON and assigns to field `answer`.  
  - Connections: Output → Create Embeddings Result

- **Remap Embeddings1**  
  - Type: Set Node  
  - Role: Extracts embedding array from ground truth response JSON and assigns to field `data`.  
  - Connections: Output → Aggregate

- **Aggregate**  
  - Type: Aggregate Node  
  - Role: Aggregates all split ground truth embeddings back into a single array under `groundTruth`.  
  - Connections: Output → Create Embeddings Result  
  - Failures: Empty input arrays, aggregation errors.

---

#### 2.4 Similarity Calculation

**Overview:**  
Calculates the cosine similarity between the AI answer embedding and each ground truth embedding, then computes the average similarity score.

**Nodes Involved:**  
- Create Embeddings Result  
- Calculate Similarity Score

**Node Details:**

- **Create Embeddings Result**  
  - Type: Merge Node  
  - Role: Combines AI answer embedding and aggregated ground truth embeddings into one item for scoring.  
  - Config: Mode "combine" by position (index alignment).  
  - Connections: Output → Calculate Similarity Score  
  - Failures: Misaligned merges if inputs missing.

- **Calculate Similarity Score**  
  - Type: Code Node (JavaScript)  
  - Role: Implements cosine similarity calculation for each ground truth embedding versus AI answer embedding; averages the scores.  
  - Key Code Snippet:  
    - Iterates over groundTruth embeddings, calculates cosine similarity with answer embedding.  
    - Returns average score as `score`.  
  - Connections: Output → Update Output  
  - Failures: Empty embeddings, division by zero if norms zero, runtime errors in code.

---

#### 2.5 Evaluation Reporting and Metric Update

**Overview:**  
Updates the original Google Sheet with the AI answer and similarity score, and sets the evaluation metric for tracking.

**Nodes Involved:**  
- Update Output  
- Update Metrics

**Node Details:**

- **Update Output**  
  - Type: Evaluation Node  
  - Role: Writes back the AI `answer` and calculated `score` into the Google Sheet row.  
  - Config: Uses same spreadsheet and sheet "Similarity" as source; updates `output` and `score` columns.  
  - Connections: Output → Update Metrics  
  - Failures: Google Sheets API errors, permission issues.

- **Update Metrics**  
  - Type: Evaluation Node  
  - Role: Sets the metric `score` in the evaluation system for further analysis or dashboards.  
  - Config: Metric name = "score", value = calculated score.  
  - Connections: None (terminal node)  
  - Failures: Evaluation system misconfiguration.

---

#### 2.6 No-Operation Handling

**Overview:**  
Handles cases where evaluation is skipped, ensuring smooth workflow without side effects.

**Nodes Involved:**  
- No Operation, do nothing

**Node Details:**

- **No Operation, do nothing**  
  - Type: NoOp Node  
  - Role: Acts as a sink to consume flow when evaluation condition is false.  
  - Connections: None  
  - Failures: None; safe fallback.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                          | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                                                      |
|----------------------------|----------------------------------|----------------------------------------|----------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| When fetching a dataset row | Evaluation Trigger                | Triggers on new/updated Google Sheet row | None                             | Remap Input                     | ## 1. Setup Your AI Workflow to Use Evaluations [Learn more about the Evaluations Trigger](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger) The Evaluations Trigger is a separate execution which does not affect your production workflow in any way. It is manually triggered and automatically pulled datasets from the assigned Google Sheet. |
| Remap Input                | Set                              | Maps input field to chatInput variable | When fetching a dataset row       | AI Agent                       |                                                                                                                                 |
| AI Agent                   | LangChain Agent                  | Executes AI processing with language model | Remap Input, When chat message received | Evaluation                     |                                                                                                                                 |
| When chat message received | LangChain Chat Trigger           | Starts workflow on chat message webhook | None                             | AI Agent                       |                                                                                                                                 |
| OpenAI Chat Model1         | LangChain OpenAI Chat Model      | Runs GPT-4.1-mini AI model for responses | None                             | AI Agent (ai_languageModel)    |                                                                                                                                 |
| Evaluation                 | Evaluation Node                  | Decides whether to evaluate or skip     | AI Agent                         | Set Input Fields, No Operation | ## 2. Answer Similarity: How similar is the AI response to the Ground Truth? [Learn more about the Evaluations Trigger](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger) For this evaluation, we want to do a simple check to see how similar the AI's response is to the ground truth. This test is most effective for closed-ended questions where there can be little to no deviation in the answers. |
| Set Input Fields           | Set                              | Extracts question, answer, and splits ground truth for processing | Evaluation                      | Get Embeddings1, GroundTruth to Items |                                                                                                                                 |
| Get Embeddings1            | HTTP Request                    | Obtains embedding vector for AI answer  | Set Input Fields                 | Remap Embeddings               |                                                                                                                                 |
| GroundTruth to Items       | Split Out                       | Splits ground truth text into array items | Set Input Fields                 | Get Embeddings                 |                                                                                                                                 |
| Get Embeddings             | HTTP Request                    | Obtains embedding vectors for each ground truth item | GroundTruth to Items             | Remap Embeddings1              |                                                                                                                                 |
| Remap Embeddings           | Set                              | Extracts AI answer embedding from API response | Get Embeddings1                 | Create Embeddings Result       |                                                                                                                                 |
| Remap Embeddings1          | Set                              | Extracts ground truth embedding from API response | Get Embeddings                  | Aggregate                     |                                                                                                                                 |
| Aggregate                  | Aggregate                       | Combines all ground truth embeddings into array | Remap Embeddings1               | Create Embeddings Result       |                                                                                                                                 |
| Create Embeddings Result   | Merge                          | Combines AI answer and ground truth embeddings for scoring | Remap Embeddings, Aggregate     | Calculate Similarity Score     |                                                                                                                                 |
| Calculate Similarity Score | Code                           | Calculates cosine similarity score between answer and ground truth embeddings | Create Embeddings Result         | Update Output                 | ## Try It Out! This n8n template demonstrates how to calculate the evaluation metric "Similarity" which measures LLM consistency with expected results. The scoring approach is adapted from https://github.com/explodinggradients/ragas/blob/main/ragas/src/ragas/metrics/_answer_similarity.py The scoring is most effective for closed-ended questions. |
| Update Output             | Evaluation                     | Updates Google Sheet with AI answer and similarity score | Calculate Similarity Score       | Update Metrics                |                                                                                                                                 |
| Update Metrics            | Evaluation                     | Sets evaluation metrics for tracking     | Update Output                   | None                         |                                                                                                                                 |
| No Operation, do nothing   | NoOp                           | Handles workflow stop conditions safely  | Evaluation                      | None                         |                                                                                                                                 |
| Sticky Note1              | Sticky Note                    | Documentation - Setup info                | None                           | None                         | ## 1. Setup Your AI Workflow to Use Evaluations [Learn more about the Evaluations Trigger](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger) |
| Sticky Note               | Sticky Note                    | Documentation - Answer Similarity concept | None                           | None                         | ## 2. Answer Similarity: How similar is the AI response to the Ground Truth? [Learn more about the Evaluations Trigger](https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger) |
| Sticky Note3              | Sticky Note                    | Documentation - Try it out, scoring approach, requirements, help links | None                           | None                         | ## Try It Out! This n8n template demonstrates how to calculate the evaluation metric "Similarity" which measures consistency. Scoring approach from https://github.com/explodinggradients/ragas/blob/main/ragas/src/ragas/metrics/_answer_similarity.py |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: "When fetching a dataset row"**  
   - Type: Evaluation Trigger  
   - Configure to monitor Google Sheet named "Similarity" in spreadsheet with ID `1YOnu2JJjlxd787AuYcg-wKbkjyjyZFgASYVV0jsij5Y`  
   - Use Google Sheets OAuth2 credentials  
   - Position: Left side (for example, X=-2040, Y=100)

2. **Create "Remap Input" Set Node**  
   - Add assignment: `chatInput = {{$json.input}}`  
   - Connect output of "When fetching a dataset row" → "Remap Input"  
   - Position: X=-1820, Y=100

3. **Create Trigger Node: "When chat message received"**  
   - Type: LangChain Chat Trigger  
   - Set webhook ID as appropriate  
   - Position: X=-1820, Y=300

4. **Create "OpenAI Chat Model1" Node**  
   - Type: LangChain OpenAI Chat Model  
   - Select model: "gpt-4.1-mini"  
   - Use OpenAI API credentials  
   - Position: X=-1512, Y=420

5. **Create "AI Agent" Node**  
   - Type: LangChain Agent  
   - Connect "Remap Input" output → "AI Agent" input  
   - Connect "When chat message received" output → "AI Agent" input  
   - Connect "OpenAI Chat Model1" node to AI Agent’s `ai_languageModel` input  
   - Position: X=-1600, Y=200

6. **Create "Evaluation" Node**  
   - Type: Evaluation  
   - Set operation to "checkIfEvaluating"  
   - Connect output of "AI Agent" → "Evaluation"  
   - Position: X=-1180, Y=200

7. **Create "Set Input Fields" Node**  
   - Type: Set  
   - Create assignments:  
     - `question` = `{{$('When fetching a dataset row').first().json.input}}`  
     - `answer` = `{{$json.output}}`  
     - `groundTruth` = `{{$('When fetching a dataset row').first().json['ground truth'].split('\n')}}`  
   - Connect first output of "Evaluation" (evaluating = true) → "Set Input Fields"  
   - Position: X=-960, Y=100

8. **Create "No Operation, do nothing" Node**  
   - Type: NoOp (No Operation)  
   - Connect second output of "Evaluation" (evaluating = false) → NoOp  
   - Position: X=-960, Y=300

9. **Create "Get Embeddings1" HTTP Request Node**  
   - Method: POST to `https://api.openai.com/v1/embeddings`  
   - Body parameters:  
     - `input` = `{{$json.answer}}`  
     - `model` = `"text-embedding-3-small"`  
     - `encoding_format` = `"float"`  
   - Authentication: OpenAI API credentials  
   - Connect output of "Set Input Fields" → "Get Embeddings1"  
   - Position: X=-520, Y=100

10. **Create "GroundTruth to Items" Node**  
    - Type: Split Out  
    - Field to split: `groundTruth`  
    - Connect output of "Set Input Fields" → "GroundTruth to Items"  
    - Position: X=-720, Y=300

11. **Create "Get Embeddings" HTTP Request Node**  
    - Same configuration as "Get Embeddings1" but `input` = `{{$json.groundTruth}}` (one item)  
    - Connect output of "GroundTruth to Items" → "Get Embeddings"  
    - Position: X=-520, Y=300

12. **Create "Remap Embeddings" Set Node**  
    - Assign `answer` = `{{$json.data[0].embedding}}` from "Get Embeddings1" response  
    - Connect output of "Get Embeddings1" → "Remap Embeddings"  
    - Position: X=-320, Y=100

13. **Create "Remap Embeddings1" Set Node**  
    - Assign `data` = `{{$json.data[0].embedding}}` from "Get Embeddings" response  
    - Connect output of "Get Embeddings" → "Remap Embeddings1"  
    - Position: X=-320, Y=300

14. **Create "Aggregate" Node**  
    - Aggregate mode: aggregate all item data into field `groundTruth`  
    - Connect output of "Remap Embeddings1" → "Aggregate"  
    - Position: X=-120, Y=300

15. **Create "Create Embeddings Result" Merge Node**  
    - Mode: Combine by position (index)  
    - Connect output of "Remap Embeddings" → input 1  
    - Connect output of "Aggregate" → input 2  
    - Position: X=80, Y=300

16. **Create "Calculate Similarity Score" Code Node**  
    - Paste JavaScript code to compute cosine similarity between `answer` and each item in `groundTruth`, output average score.  
    - Connect output of "Create Embeddings Result" → "Calculate Similarity Score"  
    - Position: X=280, Y=300

17. **Create "Update Output" Evaluation Node**  
    - Operation: Update outputs  
    - Update values:  
      - `output` = `{{$('Set Input Fields').item.json.answer}}`  
      - `score` = `{{$json.score}}`  
    - Sheet: same as trigger  
    - Connect output of "Calculate Similarity Score" → "Update Output"  
    - Position: X=520, Y=300

18. **Create "Update Metrics" Evaluation Node**  
    - Operation: setMetrics  
    - Metric: `score` = `{{$json.score}}`  
    - Connect output of "Update Output" → "Update Metrics"  
    - Position: X=700, Y=300

19. **Add Sticky Notes for Documentation**  
    - Add notes describing setup, answer similarity purpose, and usage instructions as per original workflow content.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Setup Your AI Workflow to Use Evaluations. The Evaluations Trigger is a separate execution which does not affect your production workflow in any way. It is manually triggered and automatically pulls datasets from the assigned Google Sheet. | https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger |
| Answer Similarity: How similar is the AI response to the Ground Truth? This evaluation works best for closed-ended questions where there can be little to no deviation in answers. | https://docs.n8n.io/integrations/builtin/?utm_source=n8n_app&utm_medium=node_settings_modal-credential_link&utm_campaign=n8n-nodes-base.evaluationTrigger |
| This template demonstrates calculating the evaluation metric "Similarity" adapted from [ragas Answer Similarity metric](https://github.com/explodinggradients/ragas/blob/main/ragas/src/ragas/metrics/_answer_similarity.py). The scoring approach uses cosine similarity between embeddings of AI answers and ground truth. | https://github.com/explodinggradients/ragas/blob/main/ragas/src/ragas/metrics/_answer_similarity.py |
| Need Help? Join the n8n [Discord](https://discord.com/invite/XPKeKXeB7d) or ask in the [Forum](https://community.n8n.io/)! | https://discord.com/invite/XPKeKXeB7d, https://community.n8n.io/ |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.