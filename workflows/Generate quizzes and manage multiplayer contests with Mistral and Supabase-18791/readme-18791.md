Generate quizzes and manage multiplayer contests with Mistral and Supabase

https://n8nworkflows.xyz/workflows/generate-quizzes-and-manage-multiplayer-contests-with-mistral-and-supabase-18791


# Generate quizzes and manage multiplayer contests with Mistral and Supabase

### 1. Workflow Overview

This workflow powers MindArena’s quiz automation backend, handling single-player quiz generation, quiz retrieval, AI-powered performance analysis, and multiplayer contest room management through distinct webhook entry points.

The workflow logic is divided into five core functional blocks:
- **1.1 AI Quiz Generation:** Receives quiz parameters via POST, calls Mistral Cloud to generate questions, stores the master record in Supabase, and iterates over questions to persist them individually.
- **1.2 Quiz Retrieval:** Accepts a quiz ID query parameter, queries Supabase for the quiz metadata and associated questions, and formats them into a single JSON response.
- **1.3 AI Performance Feedback:** Receives student results, invokes an AI Agent powered by Mistral Cloud to analyze performance, parses or falls back to standard JSON feedback, and returns structured tutoring notes.
- **1.4 Arena Management Router:** Acts as a centralized single-endpoint dispatcher (`/webhook/arena-management`) that uses a Switch node to route requests based on the `action` parameter.
- **1.5 Arena Sub-Actions (Rooms, Contests, and Results):** Handles 7 distinct multiplayer operations: `createRoom`, `joinRoom`, `getRoom`, `startContest`, `getContest`, `submitResult`, and `getLeaderboard`.

---

### 2. Block-by-Block Analysis

#### 2.1 AI Quiz Generation
- **Overview:** Receives topic, difficulty, and question count requests, uses a Mistral language model via an LLM Chain to generate structured JSON questions, validates the output, and writes the batch to Supabase.
- **Nodes Involved:** `Webhook1`, `Create Quiz`, `Basic LLM Chain`, `Mistral Cloud Chat Model2`, `Code in JavaScript1`, `If`, `Limit`, `Loop Over Items`, `Create Question`, `Respond to Webhook4`, `Respond to Webhook5`.
- **Node Details:**
  - **Webhook1**
    - *Type/Role:* `n8n-nodes-base.webhook` (Trigger) — Listens for incoming POST requests at path `generate-quiz`.
    - *Config:* HTTP Method: POST, Response Mode: Response Node.
    - *Expressions:* Extracts `topic`, `difficulty`, and `number_of_questions` from `{{ $json.body }}`.
    - *Connections:* Input: None; Output: `Create Quiz`.
    - *Edge Cases:* Missing or malformed JSON payloads will cause downstream validation errors.
  - **Create Quiz**
    - *Type/Role:* `n8n-nodes-base.supabase` (Database Operation) — Inserts the initial quiz record into the `quizzes` table.
    - *Config:* Operation: Insert/Create. Table ID: `quizzes`.
    - *Expressions:* `topic`, `number_of_questions`, and `difficulty` mapped from `Webhook1`.
    - *Connections:* Input: `Webhook1`; Output: `Basic LLM Chain`.
    - *Edge Cases:* Supabase connection failure, invalid table schema, or constraint violations.
  - **Basic LLM Chain**
    - *Type/Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Processing) — Formats prompts enforcing strict JSON output for quiz generation.
    - *Config:* Prompt Type: Define. Prompts request a specific number of questions with 4 options and 1 correct answer without markdown.
    - *Connections:* Input: `Create Quiz`, `Mistral Cloud Chat Model2`; Output: `Code in JavaScript1`.
    - *Version Requirements:* LangChain node version 1.9+.
    - *Edge Cases:* LLM returning markdown formatting (` ```json `) or invalid JSON syntax.
  - **Mistral Cloud Chat Model2**
    - *Type/Role:* `@n8n/n8n-nodes-langchain.lmChatMistralCloud` (AI Model) — Provides the Mistral Large model (`mistral-large-latest`) integration.
    - *Config:* Model selection: `mistral-large-latest`.
    - *Connections:* Input: None; Output: `Basic LLM Chain` (AI model link).
    - *Credentials:* Requires `Mistral Cloud account`.
  - **Code in JavaScript1**
    - *Type/Role:* `n8n-nodes-base.code` (Data Transformation) — Parses raw LLM text output into discrete JSON items per question, injecting parent webhook metadata.
    - *Config:* JavaScript execution mode.
    - *Connections:* Input: `Basic LLM Chain`; Output: `If`.
    - *Edge Cases:* Parsing failure if LLM output is not valid JSON.
  - **If**
    - *Type/Role:* `n8n-nodes-base.if` (Flow Control) — Verifies if questions exist and the item array length is greater than 0.
    - *Config:* Evaluates existence of `question` and length > 0.
    - *Connections:* Input: `Code in JavaScript1`; Outputs: `Limit` (true branch), `Respond to Webhook4` (false branch).
  - **Limit**
    - *Type/Role:* `n8n-nodes-base.limit` (Data Shaping) — Truncates the question array to match the requested `number_of_questions`.
    - *Config:* Max items: `={{ $json.number_of_questions }}`.
    - *Connections:* Input: `If` (true); Output: `Loop Over Items`.
  - **Loop Over Items**
    - *Type/Role:* `n8n-nodes-base.splitInBatches` (Flow Control) — Iterates through questions one by one for sequential database insertion.
    - *Connections:* Input: `Limit`; Outputs: `Respond to Webhook5` (when loop finishes), `Create Question` (loop iteration).
  - **Create Question**
    - *Type/Role:* `n8n-nodes-base.supabase` (Database Operation) — Inserts an individual question linked to the created quiz ID into the `questions` table.
    - *Config:* Operation: Create. Table ID: `questions`. Field mapping includes `quiz_id`, `question`, options A-D, `correct_answer`, and `explanation`.
    - *Connections:* Input: `Loop Over Items`; Output: `Loop Over Items` (loops back).
  - **Respond to Webhook4**
    - *Type/Role:* `n8n-nodes-base.respondToWebhook` (Response Node) — Returns a failure JSON message when no questions are generated.
    - *Config:* Response body: `{"success": false, "message": "No questions were generated."}`.
    - *Connections:* Input: `If` (false); Output: None.
  - **Respond to Webhook5**
    - *Type/Role:* `n8n-nodes-base.respondToWebhook` (Response Node) — Returns a success payload containing the generated `quiz_id`.
    - *Config:* Response body references `{{ $('Create Quiz').first().json.id }}`.
    - *Connections:* Input: `Loop Over Items` (completion); Output: None.

---

#### 2.2 Quiz Retrieval
- **Overview:** Fetches quiz details and its corresponding questions from Supabase based on a provided quiz ID query parameter, returning a combined JSON payload.
- **Nodes Involved:** `Webhook`, `Get a row`, `Get many rows`, `Code in JavaScript`, `Respond to Webhook`.
- **Node Details:**
  - **Webhook**
    - *Type/Role:* `n8n-nodes-base.webhook` (Trigger) — Listens for requests on path `get-quiz`.
    - *Config:* Response Mode: Response Node.
    - *Connections:* Input: None; Output: `Get a row`.
  - **Get a row**
    - *Type/Role:* `n8n-nodes-base.supabase` (Database Operation) — Retrieves the quiz metadata record from the `quizzes` table using query parameters.
    - *Config:* Operation: Get. Table ID: `quizzes`. Filter condition: `id` equals `={{ $json.query.id }}`.
    - *Connections:* Input: `Webhook`; Output: `Get many rows`.
    - *Edge Cases:* Quiz ID not found in database.
  - **Get many rows**
    - *Type/Role:* `n8n-nodes-base.supabase` (Database Operation) — Retrieves all questions associated with the quiz ID from the `questions` table.
    - *Config:* Operation: Get All. Table ID: `questions`. Filter: `quiz_id` equals `={{ $json.id }}`. Return All: True.
    - *Connections:* Input: `Get a row`; Output: `Code in JavaScript`.
  - **Code in JavaScript**
    - *Type/Role:* `n8n-nodes-base.code` (Data Transformation) — Aggregates quiz details and maps all retrieved question records into an organized array.
    - *Connections:* Input: `Get many rows`; Output: `Respond to Webhook`.
  - **Respond to Webhook**
    - *Type/Role:* `n8n-nodes-base.respondToWebhook` (Response Node) — Returns the unified quiz and questions JSON object to the client.
    - *Connections:* Input: `Code in JavaScript`; Output: None.

---

#### 2.3 AI Performance Feedback
- **Overview:** Evaluates student test metrics using an AI Agent powered by Mistral Cloud, parses the structured response, and applies a fallback mechanism if JSON parsing fails.
- **Nodes Involved:** `Webhook - AI Feedback`, `AI Agent1`, `Mistral Cloud Chat Model1`, `Parse Structured Feedback`, `Respond Webhook`.
- **Node Details:**
  - **Webhook - AI Feedback**
    - *Type/Role:* `n8n-nodes-base.webhook` (Trigger) — Listens for POST requests at path `ai-feedback`.
    - *Config:* HTTP Method: POST. Response Mode: Response Node.
    - *Connections:* Input: None; Output: `AI Agent1`.
  - **AI Agent1**
    - *Type/Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent) — Processes student score, percentage, topic, and difficulty to generate pedagogical feedback.
    - *Config:* Prompt type: Define. Enforces a strict 4-field JSON output schema (`overall_performance`, `strengths`, `areas_to_improve`, `practical_suggestion`) without markdown.
    - *Connections:* Input: `Webhook - AI Feedback`, `Mistral Cloud Chat Model1`; Output: `Parse Structured Feedback`.
    - *Version Requirements:* LangChain agent node version 3.1+.
  - **Mistral Cloud Chat Model1**
    - *Type/Role:* `@n8n/n8n-nodes-langchain.lmChatMistralCloud` (AI Model) — Supplies the Mistral Cloud chat engine for the tutor agent.
    - *Connections:* Input: None; Output: `AI Agent1` (AI model link).
    - *Credentials:* Requires `Mistral Cloud account`.
  - **Parse Structured Feedback**
    - *Type/Role:* `n8n-nodes-base.code` (Data Transformation) — Sanitizes AI text outputs, strips markdown blocks, parses JSON, and falls back to default performance messages if parsing throws an error.
    - *Connections:* Input: `AI Agent1`; Output: `Respond Webhook`.
    - *Edge Cases:* Malformed JSON returned by the AI agent triggers the robust fallback object.
  - **Respond Webhook**
    - *Type/Role:* `n8n-nodes-base.respondToWebhook` (Response Node) — Stringifies and returns the structured feedback payload.
    - *Config:* Response body: `={{ JSON.stringify($json) }}`.
    - *Connections:* Input: `Parse Structured Feedback`; Output: None.

---

#### 2.4 Arena Management Router
- **Overview:** Receives all multiplayer requests at a single webhook endpoint and dispatches execution paths based on the `action` property in the request body.
- **Nodes Involved:** `Webhook - Arena Mgmt1`, `Switch - Action Router2`.
- **Node Details:**
  - **Webhook - Arena Mgmt1**
    - *Type/Role:* `n8n-nodes-base.webhook` (Trigger) — Listens for POST requests at path `arena-management`.
    - *Config:* HTTP Method: POST. Response Mode: Response Node.
    - *Connections:* Input: None; Output: `Switch - Action Router2`.
  - **Switch - Action Router2**
    - *Type/Role:* `n8n-nodes-base.switch` (Flow Control) — Routes incoming requests based on `{{ $json.body.action }}`.
    - *Config:* 7 routing rules evaluating string equality: `createRoom`, `joinRoom`, `getRoom`, `startContest`, `getContest`, `submitResult`, and `getLeaderboard`.
    - *Connections:* Input: `Webhook - Arena Mgmt1`; Outputs: Corresponding prepare nodes for each action branch.

---

#### 2.5 Arena Sub-Actions (Rooms, Contests, and Results)
- **Overview:** Executes specific multiplayer game logic including room creation, user joining, state retrieval, quiz synchronization, answer scoring, and leaderboard calculation.
- **Nodes Involved:** `CreateRoom - Prepare1`, `Create Quiz2`, `Basic LLM Chain2`, `Mistral Cloud Chat Model4`, `CreateRoom - Parse Questions1`, `Limit2`, `Create Question3`, `CreateRoom - Questions Complete`, `CreateRoom - Insert Room1`, `CreateRoom - Prepare Host1`, `CreateRoom - Add Host1`, `CreateRoom - Response1`, `Respond - createRoom1`, `JoinRoom - Prepare`, `JoinRoom - Get Room`, `JoinRoom - Check Room`, `JoinRoom - Get Players`, `JoinRoom - Validate Player`, `JoinRoom - Add Player`, `JoinRoom - Response`, `Respond - joinRoom`, `GetRoom - Prepare`, `GetRoom - Get Room`, `GetRoom - Players`, `GetRoom - Response`, `Respond - getRoom`, `StartContest - Prepare`, `StartContest - Get Room`, `StartContest - Authorize and Update`, `StartContest - Update Room`, `StartContest - Response`, `Respond - startContest`, `GetContest - Prepare`, `GetContest - Room`, `GetContest - Questions`, `GetContest - Secure Response`, `Respond - getContest`, `SubmitResult - Prepare`, `SubmitResult - Get Room`, `SubmitResult - Get Questions`, `SubmitResult - Score Server Side`, `SubmitResult - Insert Result`, `SubmitResult - Response`, `Respond - submitResult`, `Leaderboard - Prepare`, `Leaderboard - Results`, `Leaderboard - Rank`, `Respond - getLeaderboard`.
- **Node Details:**
  - **CreateRoom - Prepare1**
    - *Type/Role:* `n8n-nodes-base.code` (Data Validation & Generation) — Validates room creation payload parameters, enforces constraints, and generates a random 8-character alphanumeric room code.
    - *Connections:* Input: `Switch - Action Router2`; Output: `Create Quiz2`.
    - *Edge Cases:* Missing mandatory parameters (`host_id`, `topic`, etc.) or `max_players` outside the 2–10 range trigger errors.
  - **Create Quiz2**
    - *Type/Role:* `n8n-nodes-base.supabase` (Database Operation) — Inserts the new quiz record tied to the room into Supabase.
    - *Connections:* Input: `CreateRoom - Prepare1`; Output: `Basic LLM Chain2`.
  - **Basic LLM Chain2**
    - *Type/Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Processing) — Requests AI-generated questions matching room specifications.
    - *Connections:* Input: `Create Quiz2`, `Mistral Cloud Chat Model4`; Output: `CreateRoom - Parse Questions1`.
  - **Mistral Cloud Chat Model4**
    - *Type/Role:* `@n8n/n8n-nodes-langchain.lmChatMistralCloud` (AI Model) — Chat model integration for room quiz generation.
    - *Connections:* Input: None; Output: `Basic LLM Chain2`.
    - *Credentials:* Requires `Mistral Cloud account`.
  - **CreateRoom - Parse Questions1**
    - *Type/Role:* `n8n-nodes-base.code` (Data Transformation) — Parses raw AI text into valid question arrays and maps question numbers.
    - *Connections:* Input: `Basic LLM Chain2`; Output: `Limit2`.
  - **Limit2**
    - *Type/Role:* `n8n-nodes-base.limit` (Data Shaping) — Caps the generated questions to the requested count.
    - *Connections:* Input: `CreateRoom - Parse Questions1`; Output: `Create Question3`.
  - **Create Question3**
    - *Type/Role:* `n8n-nodes-base.supabase` (Database Operation) — Inserts generated questions into the `questions` table.
    - *Connections:* Input: `Limit2`; Output: `CreateRoom - Questions Complete`.
  - **CreateRoom - Questions Complete**
    - *Type/Role:* `n8n-nodes-base.code` (Flow Aggregation) — Confirms question insertion completion.
    - *Connections:* Input: `Create Question3`; Output: `CreateRoom - Insert Room1`.
  - **CreateRoom - Insert Room1**
    - *Type/Role:* `n8n-nodes-base.supabase` (Database Operation) — Creates the game room record in the `rooms` table with status set to `waiting`.
    - *Connections:* Input: `CreateRoom - Questions Complete`; Output: `CreateRoom - Prepare Host1`.
  - **CreateRoom - Prepare Host1**
    - *Type/Role:* `n8n-nodes-base.code` (Data Transformation) — Formats host profile data for registration.
    - *Connections:* Input: `CreateRoom - Insert Room1`; Output: `CreateRoom - Add Host1`.
  - **CreateRoom - Add Host1**
    - *Type/Role:* `n8n-nodes-base.supabase` (Database Operation) — Inserts the host into the `room_players` table with `is_host: true`.
    - *Connections:* Input: `CreateRoom - Prepare Host1`; Output: `CreateRoom - Response1`.
  - **CreateRoom - Response1** & **Respond - createRoom1**
    - *Type/Role:* `n8n-nodes-base.code` & `n8n-nodes-base.respondToWebhook` (Response Nodes) — Formats and returns the successful room creation payload containing room metadata and player lists.
    - *Connections:* Input sequence: `CreateRoom - Add Host1` → `CreateRoom - Response1` → `Respond - createRoom1`.
  - **JoinRoom - Prepare** to **Respond - joinRoom**
    - *Type/Role:* Validation, database lookup (`rooms`, `room_players`), capacity checks, and player insertion.
    - *Nodes:* `JoinRoom - Prepare`, `JoinRoom - Get Room`, `JoinRoom - Check Room`, `JoinRoom - Get Players`, `JoinRoom - Validate Player`, `JoinRoom - Add Player`, `JoinRoom - Response`, `Respond - joinRoom`.
    - *Edge Cases:* Throws errors if the room is not found, status is not waiting, the room is full, or the user has already joined.
  - **GetRoom - Prepare** to **Respond - getRoom**
    - *Type/Role:* Room details and active player enumeration.
    - *Nodes:* `GetRoom - Prepare`, `GetRoom - Get Room`, `GetRoom - Players`, `GetRoom - Response`, `Respond - getRoom`.
    - *Edge Cases:* Missing `room_code` or `room_id`.
  - **StartContest - Prepare** to **Respond - startContest**
    - *Type/Role:* Host authorization check, quiz association verification, and room status update (`started`).
    - *Nodes:* `StartContest - Prepare`, `StartContest - Get Room`, `StartContest - Authorize and Update`, `StartContest - Update Room`, `StartContest - Response`, `Respond - startContest`.
    - *Edge Cases:* Non-host attempting to start the game, or missing `quiz_id`.
  - **GetContest - Prepare** to **Respond - getContest**
    - *Type/Role:* Contest state verification, active question retrieval, and stripping correct answers before delivery to clients.
    - *Nodes:* `GetContest - Prepare`, `GetContest - Room`, `GetContest - Questions`, `GetContest - Secure Response`, `Respond - getContest`.
    - *Edge Cases:* Attempting to fetch contest questions before the contest status is set to `started`.
  - **SubmitResult - Prepare** to **Respond - submitResult**
    - *Type/Role:* Submission validation, server-side answer grading against correct database values, score computation, and results persistence.
    - *Nodes:* `SubmitResult - Prepare`, `SubmitResult - Get Room`, `SubmitResult - Get Questions`, `SubmitResult - Score Server Side`, `SubmitResult - Insert Result`, `SubmitResult - Response`, `Respond - submitResult`.
    - *Edge Cases:* Submissions when contest is inactive, invalid completion times, or malformed answer payloads.
  - **Leaderboard - Prepare** to **Respond - getLeaderboard**
    - *Type/Role:* Result aggregation, sorting by score (descending) with completion time as a tie-breaker, and rank assignment.
    - *Nodes:* `Leaderboard - Prepare`, `Leaderboard - Results`, `Leaderboard - Rank`, `Respond - getLeaderboard`.
    - *Edge Cases:* Missing `room_id`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note - AI Feedback` | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## AI Performance Feedback... |
| `Webhook - AI Feedback` | `n8n-nodes-base.webhook` | Trigger webhook | None | `AI Agent1` | ## AI Performance Analysis... |
| `Parse Structured Feedback` | `n8n-nodes-base.code` | Sanitize and parse AI response | `AI Agent1` | `Respond Webhook` | ## Parse & Respond... |
| `Respond Webhook` | `n8n-nodes-base.respondToWebhook` | Return HTTP response | `Parse Structured Feedback` | None | ## Parse & Respond... |
| `AI Agent1` | `@n8n/n8n-nodes-langchain.agent` | AI tutoring agent | `Webhook - AI Feedback`, `Mistral Cloud Chat Model1` | `Parse Structured Feedback` | ## AI Performance Analysis... |
| `Mistral Cloud Chat Model1` | `@n8n/n8n-nodes-langchain.lmChatMistralCloud` | LLM integration | None | `AI Agent1` | ## AI Performance Analysis... |
| `Sticky Note - Setup` | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## MindArena Management... |
| `JoinRoom - Prepare` | `n8n-nodes-base.code` | Validate input payload | `Switch - Action Router2` | `JoinRoom - Get Room` | ## Join Room... |
| `JoinRoom - Get Room` | `n8n-nodes-base.supabase` | Query room by code | `JoinRoom - Prepare` | `JoinRoom - Check Room` | ## Join Room... |
| `JoinRoom - Check Room` | `n8n-nodes-base.code` | Validate room status & players | `JoinRoom - Get Room` | `JoinRoom - Get Players` | ## Join Room... |
| `JoinRoom - Add Player` | `n8n-nodes-base.supabase` | Insert player record | `JoinRoom - Validate Player` | `JoinRoom - Response` | ## Join Room... |
| `JoinRoom - Response` | `n8n-nodes-base.code` | Format success message | `JoinRoom - Add Player` | `Respond - joinRoom` | ## Join Room... |
| `Respond - joinRoom` | `n8n-nodes-base.respondToWebhook` | Return HTTP response | `JoinRoom - Response` | None | ## Join Room... |
| `GetRoom - Prepare` | `n8n-nodes-base.code` | Validate room identifier | `Switch - Action Router2` | `GetRoom - Get Room` | ## Get Room... |
| `GetRoom - Get Room` | `n8n-nodes-base.supabase` | Retrieve room details | `GetRoom - Prepare` | `GetRoom - Players` | ## Get Room... |
| `GetRoom - Players` | `n8n-nodes-base.supabase` | Retrieve room players | `GetRoom - Get Room` | `GetRoom - Response` | ## Get Room... |
| `GetRoom - Response` | `n8n-nodes-base.code` | Combine room & players | `GetRoom - Players` | `Respond - getRoom` | ## Get Room... |
| `Respond - getRoom` | `n8n-nodes-base.respondToWebhook` | Return HTTP response | `GetRoom - Response` | None | ## Get Room... |
| `StartContest - Prepare` | `n8n-nodes-base.code` | Validate host request | `Switch - Action Router2` | `StartContest - Get Room` | ## Start Contest... |
| `StartContest - Get Room` | `n8n-nodes-base.supabase` | Retrieve room status | `StartContest - Prepare` | `StartContest - Authorize and Update` | ## Start Contest... |
| `StartContest - Authorize and Update` | `n8n-nodes-base.code` | Authorize host & quiz | `StartContest - Get Room` | `StartContest - Update Room` | ## Start Contest... |
| `StartContest - Response` | `n8n-nodes-base.code` | Format start response | `StartContest - Update Room` | `Respond - startContest` | ## Start Contest... |
| `Respond - startContest` | `n8n-nodes-base.respondToWebhook` | Return HTTP response | `StartContest - Response` | None | ## Start Contest... |
| `GetContest - Prepare` | `n8n-nodes-base.code` | Validate room ID | `Switch - Action Router2` | `GetContest - Room` | ## Get Contest... |
| `GetContest - Room` | `n8n-nodes-base.supabase` | Retrieve active room | `GetContest - Prepare` | `GetContest - Questions` | ## Get Contest... |
| `GetContest - Questions` | `n8n-nodes-base.supabase` | Retrieve contest questions | `GetContest - Room` | `GetContest - Secure Response` | ## Get Contest... |
| `GetContest - Secure Response` | `n8n-nodes-base.code` | Filter out correct answers | `GetContest - Questions` | `Respond - getContest` | ## Get Contest... |
| `Respond - getContest` | `n8n-nodes-base.respondToWebhook` | Return HTTP response | `GetContest - Secure Response` | None | ## Get Contest... |
| `SubmitResult - Prepare` | `n8n-nodes-base.code` | Validate submission payload | `Switch - Action Router2` | `SubmitResult - Get Room` | ## Submit Result... |
| `SubmitResult - Get Questions` | `n8n-nodes-base.supabase` | Retrieve quiz questions | `SubmitResult - Get Room` | `SubmitResult - Score Server Side` | ## Submit Result... |
| `SubmitResult - Response` | `n8n-nodes-base.code` | Format result payload | `SubmitResult - Insert Result` | `Respond - submitResult` | ## Submit Result... |
| `Respond - submitResult` | `n8n-nodes-base.respondToWebhook` | Return HTTP response | `SubmitResult - Response` | None | ## Submit Result... |
| `Leaderboard - Prepare` | `n8n-nodes-base.code` | Validate room ID | `Switch - Action Router2` | `Leaderboard - Results` | ## Get Leaderboard... |
| `Leaderboard - Results` | `n8n-nodes-base.supabase` | Fetch contest results | `Leaderboard - Prepare` | `Leaderboard - Rank` | ## Get Leaderboard... |
| `Leaderboard - Rank` | `n8n-nodes-base.code` | Sort and rank players | `Leaderboard - Results` | `Respond - getLeaderboard` | ## Get Leaderboard... |
| `Respond - getLeaderboard` | `n8n-nodes-base.respondToWebhook` | Return HTTP response | `Leaderboard - Rank` | None | ## Get Leaderboard... |
| `SubmitResult - Get Room` | `n8n-nodes-base.supabase` | Retrieve room status | `SubmitResult - Prepare` | `SubmitResult - Get Questions` | ## Submit Result... |
| `SubmitResult - Score Server Side` | `n8n-nodes-base.code` | Grade answers server-side | `SubmitResult - Get Questions` | `SubmitResult - Insert Result` | ## Submit Result... |
| `SubmitResult - Insert Result` | `n8n-nodes-base.supabase` | Save score in Supabase | `SubmitResult - Score Server Side` | `SubmitResult - Response` | ## Submit Result... |
| `Webhook - Arena Mgmt1` | `n8n-nodes-base.webhook` | Trigger arena webhook | None | `Switch - Action Router2` | ## Webhook + Router... |
| `StartContest - Update Room` | `n8n-nodes-base.supabase` | Update room status | `StartContest - Authorize and Update` | `StartContest - Response` | ## Start Contest... |
| `Switch - Action Router2` | `n8n-nodes-base.switch` | Route actions | `Webhook - Arena Mgmt1` | Multiple prepare nodes | ## Webhook + Router... |
| `Basic LLM Chain` | `@n8n/n8n-nodes-langchain.chainLlm` | Generate quiz questions | `Create Quiz`, `Mistral Cloud Chat Model2` | `Code in JavaScript1` | ## Generate & Process... |
| `Loop Over Items` | `n8n-nodes-base.splitInBatches` | Iterate question items | `Limit`, `Create Question` | `Respond to Webhook5`, `Create Question` | ## Save & Return Result... |
| `Create Question` | `n8n-nodes-base.supabase` | Save question record | `Loop Over Items` | `Loop Over Items` | ## Save & Return Result... |
| `If` | `n8n-nodes-base.if` | Validate question generation | `Code in JavaScript1` | `Limit`, `Respond to Webhook4` | ## Validate, Limit & Respond... |
| `Create Quiz` | `n8n-nodes-base.supabase` | Insert quiz record | `Webhook1` | `Basic LLM Chain` | ## Save & Return Result... |
| `Webhook1` | `n8n-nodes-base.webhook` | Trigger quiz generation | None | `Create Quiz` | ## 1. Receive & Prepare... |
| `Code in JavaScript1` | `n8n-nodes-base.code` | Process LLM output | `Basic LLM Chain` | `If` | ## Generate & Process... |
| `Limit` | `n8n-nodes-base.limit` | Limit question count | `If` | `Loop Over Items` | ## Validate, Limit & Respond... |
| `Mistral Cloud Chat Model2` | `@n8n/n8n-nodes-langchain.lmChatMistralCloud` | LLM integration | None | `Basic LLM Chain` | ## Generate & Process... |
| `Respond to Webhook4` | `n8n-nodes-base.respondToWebhook` | Return failure response | `If` | None | ## Validate, Limit & Respond... |
| `Respond to Webhook5` | `n8n-nodes-base.respondToWebhook` | Return success response | `Loop Over Items` | None | ## Save & Return Result... |
| `CreateRoom - Prepare1` | `n8n-nodes-base.code` | Validate room parameters | `Switch - Action Router2` | `Create Quiz2` | ## Create Room... |
| `Create Quiz2` | `n8n-nodes-base.supabase` | Insert room quiz | `CreateRoom - Prepare1` | `Basic LLM Chain2` | ## Create Room... |
| `Basic LLM Chain2` | `@n8n/n8n-nodes-langchain.chainLlm` | Generate room quiz questions | `Create Quiz2`, `Mistral Cloud Chat Model4` | `CreateRoom - Parse Questions1` | ## Create Room... |
| `Mistral Cloud Chat Model4` | `@n8n/n8n-nodes-langchain.lmChatMistralCloud` | LLM integration | None | `Basic LLM Chain2` | ## Create Room... |
| `CreateRoom - Parse Questions1` | `n8n-nodes-base.code` | Parse AI question output | `Basic LLM Chain2` | `Limit2` | ## Create Room... |
| `Limit2` | `n8n-nodes-base.limit` | Limit question array | `CreateRoom - Parse Questions1` | `Create Question3` | ## Create Room... |
| `Create Question3` | `n8n-nodes-base.supabase` | Save question record | `Limit2` | `CreateRoom - Questions Complete` | ## Create Room... |
| `CreateRoom - Insert Room1` | `n8n-nodes-base.supabase` | Insert room record | `CreateRoom - Questions Complete` | `CreateRoom - Prepare Host1` | ## Create Room... |
| `CreateRoom - Prepare Host1` | `n8n-nodes-base.code` | Format host registration | `CreateRoom - Insert Room1` | `CreateRoom - Add Host1` | ## Create Room... |
| `CreateRoom - Add Host1` | `n8n-nodes-base.supabase` | Add host as player | `CreateRoom - Prepare Host1` | `CreateRoom - Response1` | ## Create Room... |
| `CreateRoom - Response1` | `n8n-nodes-base.code` | Format room response | `CreateRoom - Add Host1` | `Respond - createRoom1` | ## Create Room... |
| `Respond - createRoom1` | `n8n-nodes-base.respondToWebhook` | Return HTTP response | `CreateRoom - Response1` | None | ## Create Room... |
| `CreateRoom - Questions Complete` | `n8n-nodes-base.code` | Confirm questions saved | `Create Question3` | `CreateRoom - Insert Room1` | ## Create Room... |
| `JoinRoom - Get Players` | `n8n-nodes-base.supabase` | Retrieve room players | `JoinRoom - Check Room` | `JoinRoom - Validate Player` | ## Join Room... |
| `JoinRoom - Validate Player` | `n8n-nodes-base.code` | Check room capacity | `JoinRoom - Get Players` | `JoinRoom - Add Player` | ## Join Room... |
| `Webhook` | `n8n-nodes-base.webhook` | Trigger quiz retrieval | None | `Get a row` | ## Get Quiz... |
| `Respond to Webhook` | `n8n-nodes-base.respondToWebhook` | Return HTTP response | `Code in JavaScript` | None | ## Combine & Format... |
| `Get many rows` | `n8n-nodes-base.supabase` | Fetch related questions | `Get a row` | `Code in JavaScript` | ## Get Quiz... |
| `Get a row` | `n8n-nodes-base.supabase` | Fetch quiz record | `Webhook` | `Get many rows` | ## Get Quiz... |
| `Code in JavaScript` | `n8n-nodes-base.code` | Combine quiz & questions | `Get many rows` | `Respond to Webhook` | ## Combine & Format... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Credentials and Database Tables
1. **Supabase:** Create a Supabase project and set up the following tables with appropriate columns:
   - `quizzes`: `id` (uuid/serial), `topic` (text), `difficulty` (text), `number_of_questions` (int).
   - `questions`: `id` (serial), `quiz_id` (foreign key), `question` (text), `option_a` (text), `option_b` (text), `option_c` (text), `option_d` (text), `correct_answer` (text), `explanation` (text).
   - `rooms`: `id` (serial/uuid), `room_code` (text), `host_id` (text), `quiz_id` (foreign key), `topic` (text), `difficulty` (text), `num_questions` (int), `max_players` (int), `status` (text).
   - `room_players`: `id` (serial), `room_id` (foreign key), `user_id` (text), `player_name` (text), `is_host` (boolean), `joined_at` (timestamp).
   - `contest_results`: `id` (serial), `room_id` (foreign key), `user_id` (text), `player_name` (text), `score` (int), `total_questions` (int), `percentage` (numeric), `completion_time_seconds` (numeric), `submitted_at` (timestamp).
2. **Credentials:** Set up a Supabase API credential (`supabaseApi`) and a Mistral Cloud API credential (`mistralCloudApi`) inside n8n.

#### Step 2: Build the AI Quiz Generation Flow
1. **Webhook1:** Create a Webhook node, set HTTP Method to `POST`, path to `generate-quiz`, and Response Mode to `Response Node`.
2. **Create Quiz:** Add a Supabase node (Operation: `Create`, Table: `quizzes`). Map fields: `topic` = `={{ $json.body.topic }}`, `number_of_questions` = `={{ $json.body.number_of_questions }}`, `difficulty` = `={{ $json.body.difficulty }}`.
3. **Basic LLM Chain & Mistral Cloud Chat Model2:** Add an LLM Chain node connected to a Mistral Cloud Chat Model node (`mistral-large-latest`). Set the prompt to demand valid JSON without markdown containing the requested number of questions.
4. **Code in JavaScript1:** Add a Code node to parse the LLM text output into an array of question objects, attaching parent webhook parameters.
5. **If:** Add an If node to check if valid questions exist (`{{ $json.question }}`).
6. **Limit & Loop Over Items:** Connect the true branch of the If node to a Limit node (`maxItems: ={{ $json.number_of_questions }}`), followed by a Split In Batches (`Loop Over Items`) node.
7. **Create Question:** Inside the loop, add a Supabase node (Operation: `Create`, Table: `questions`) mapping `quiz_id` and question fields. Loop back to `Loop Over Items`.
8. **Respond to Webhook4 & 5:** Connect the `If` false branch to a Respond to Webhook node returning a failure message, and connect the loop completion output to a Respond to Webhook node returning the generated `quiz_id`.

#### Step 3: Build the Quiz Retrieval Flow
1. **Webhook:** Create a Webhook node on path `get-quiz`.
2. **Get a row:** Add a Supabase node (Operation: `Get`, Table: `quizzes`) filtered by `id` = `={{ $json.query.id }}`.
3. **Get many rows:** Add a Supabase node (Operation: `Get All`, Table: `questions`) filtered by `quiz_id` = `={{ $json.id }}`.
4. **Code in JavaScript & Respond to Webhook:** Combine the retrieved quiz and questions array into a unified JSON object and respond via webhook.

#### Step 4: Build the AI Performance Feedback Flow
1. **Webhook - AI Feedback:** Create a Webhook node on path `ai-feedback` (POST).
2. **AI Agent1 & Mistral Cloud Chat Model1:** Add an AI Agent node connected to a Mistral Chat Model. Configure the prompt to evaluate student score/percentage and return strict JSON with `overall_performance`, `strengths`, `areas_to_improve`, and `practical_suggestion`.
3. **Parse Structured Feedback & Respond Webhook:** Add a Code node to sanitize the AI response and implement a default fallback message upon JSON parse failure. Connect to a Respond to Webhook node.

#### Step 5: Build the Arena Management Flow & Router
1. **Webhook - Arena Mgmt1:** Create a Webhook node on path `arena-management` (POST).
2. **Switch - Action Router2:** Add a Switch node evaluating `={{ $json.body.action }}` across 7 distinct routing rules (`createRoom`, `joinRoom`, `getRoom`, `startContest`, `getContest`, `submitResult`, `getLeaderboard`).
3. **Build Sub-Action Branches:**
   - **Create Room Branch:** Connect `createRoom` to `CreateRoom - Prepare1` (validates parameters and generates an 8-character `room_code`), `Create Quiz2`, `Basic LLM Chain2` with `Mistral Cloud Chat Model4`, `CreateRoom - Parse Questions1`, `Limit2`, `Create Question3`, `CreateRoom - Questions Complete`, `CreateRoom - Insert Room1`, `CreateRoom - Prepare Host1`, `CreateRoom - Add Host1`, `CreateRoom - Response1`, and `Respond - createRoom1`.
   - **Join Room Branch:** Connect `joinRoom` through `JoinRoom - Prepare`, `JoinRoom - Get Room`, `JoinRoom - Check Room`, `JoinRoom - Get Players`, `JoinRoom - Validate Player`, `JoinRoom - Add Player`, `JoinRoom - Response`, to `Respond - joinRoom`.
   - **Get Room Branch:** Connect `getRoom` through `GetRoom - Prepare`, `GetRoom - Get Room`, `GetRoom - Players`, `GetRoom - Response`, to `Respond - getRoom`.
   - **Start Contest Branch:** Connect `startContest` through `StartContest - Prepare`, `StartContest - Get Room`, `StartContest - Authorize and Update`, `StartContest - Update Room`, `StartContest - Response`, to `Respond - startContest`.
   - **Get Contest Branch:** Connect `getContest` through `GetContest - Prepare`, `GetContest - Room`, `GetContest - Questions`, `GetContest - Secure Response`, to `Respond - getContest`.
   - **Submit Result Branch:** Connect `submitResult` through `SubmitResult - Prepare`, `SubmitResult - Get Room`, `SubmitResult - Get Questions`, `SubmitResult - Score Server Side`, `SubmitResult - Insert Result`, `SubmitResult - Response`, to `Respond - submitResult`.
   - **Get Leaderboard Branch:** Connect `getLeaderboard` through `Leaderboard - Prepare`, `Leaderboard - Results`, `Leaderboard - Rank`, to `Respond - getLeaderboard`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Need Help or Customization? Digital Biz Tech can help tailor this workflow to your business. We offer free setup support, including credential configuration and deployment. | Contact: [rajeet.nair@digitalbiz.tech](mailto:rajeet.nair@digitalbiz.tech) |
| Digital Biz Tech Website | [https://www.digitalbiz.tech](https://www.digitalbiz.tech) |
| Digital Biz Tech LinkedIn | [https://www.linkedin.com/company/digital-biz-tech/](https://www.linkedin.com/company/digital-biz-tech/) |
| Developer: JESWIN MADONA M | LinkedIn: [https://www.linkedin.com/in/jeswinmadona/](https://www.linkedin.com/in/jeswinmadona) |