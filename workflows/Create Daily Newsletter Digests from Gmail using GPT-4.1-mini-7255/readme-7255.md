Create Daily Newsletter Digests from Gmail using GPT-4.1-mini

https://n8nworkflows.xyz/workflows/create-daily-newsletter-digests-from-gmail-using-gpt-4-1-mini-7255


# Create Daily Newsletter Digests from Gmail using GPT-4.1-mini

### 1. Workflow Overview

This workflow automates the creation and sending of a daily newsletter digest compiled from Gmail messages using GPT-4.1-mini. It targets users who want to receive a concise, well-formatted summary of newsletters or emails from specified senders, aggregated and summarized by an AI language model, and sent as an HTML email digest.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow once daily at a specified hour.
- **1.2 Gmail Message Retrieval**: Queries Gmail for all relevant messages from specified senders received since the previous day.
- **1.3 Batch Processing Loop**: Processes each message individually in batches to handle large volumes efficiently.
- **1.4 Detailed Message Fetch and Parsing**: Retrieves full message content and extracts HTML, sender, subject, and formatted date.
- **1.5 AI Summarization**: Uses GPT-4.1-mini to analyze individual emails and generate JSON-formatted topic summaries.
- **1.6 Topics Aggregation and Formatting**: Merges all topic summaries from the batch, formats them into a clean HTML email template.
- **1.7 Email Sending**: Sends the assembled digest email to the configured recipient.

Supporting the main flow are utility nodes for data cleaning, HTML extraction, and notes to explain workflow steps.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview**: Starts the workflow once daily at 16:00 (4 PM).
- **Nodes Involved**: `Schedule Trigger`
- **Node Details**:
  - Type: `Schedule Trigger`
  - Configuration: Triggers workflow daily at hour 16.
  - Inputs: None (entry point).
  - Outputs: Starts the chain to fetch Gmail messages.
  - Edge cases: Workflow will not run if n8n instance is offline at trigger time.

#### 1.2 Gmail Message Retrieval

- **Overview**: Queries Gmail and retrieves all messages matching specific sender filters from the last 24 hours.
- **Nodes Involved**: `Get many messages`
- **Node Details**:
  - Type: `Gmail` node, operation `getAll`
  - Configuration:
    - Query filter includes multiple sender addresses, excludes certain strings, and limits to messages after yesterday’s date.
    - Returns all matched messages (`returnAll: true`).
    - Uses OAuth2 Gmail credentials.
  - Inputs: Trigger from `Schedule Trigger`.
  - Outputs: List of message metadata (IDs).
  - Edge cases:
    - Gmail API rate limits.
    - No messages found returns empty array.
    - Auth errors if token expired.

#### 1.3 Batch Processing Loop

- **Overview**: Processes each Gmail message individually to avoid processing overload.
- **Nodes Involved**: `Loop Over Items`
- **Node Details**:
  - Type: `SplitInBatches`
  - Configuration: Default batch size, iterates one message at a time.
  - Inputs: Message list from `Get many messages`.
  - Outputs: Batches single message ID to next nodes.
  - Edge cases: If no messages, loop does not run.

#### 1.4 Detailed Message Fetch and Parsing

- **Overview**: Retrieves full content of each message, extracts HTML body, sender, subject, and reformats the date.
- **Nodes Involved**: `Get a message`, `Get message data`, `Clean`
- **Node Details**:
  - **Get a message**
    - Type: `Gmail` node, operation `get`
    - Configuration: Fetches full message by ID from batch.
    - Inputs: Batch message ID from `Loop Over Items`.
    - Outputs: Full message JSON.
    - Edge cases: Message may be deleted between list and fetch; API errors possible.
  - **Get message data**
    - Type: `Code` node (JavaScript)
    - Configuration:
      - Extracts HTML content from the payload, including multipart emails.
      - Cleans and extracts sender’s name (removes email, quotes).
      - Converts date to `DD.MM.YYYY` format.
      - Extracts subject.
    - Inputs: Full message JSON.
    - Outputs: Object with html, subject, from, date.
    - Edge cases: Missing HTML parts, malformed dates.
  - **Clean**
    - Type: `Code` node
    - Configuration:
      - Converts date from `DD.MM.YYYY` to `MM.DD`.
      - Filters out items missing HTML or date.
      - Passes cleaned data forward.
    - Inputs: Output from `Get message data`.
    - Outputs: Filtered and cleaned message data.
    - Edge cases: Null or missing fields filtered out.

#### 1.5 AI Summarization

- **Overview**: Sends the cleaned message content to GPT-4.1-mini model for summarization into JSON topics.
- **Nodes Involved**: `Message a model`
- **Node Details**:
  - Type: `OpenAI` node (LangChain integration), model `gpt-4.1-mini`
  - Configuration:
    - Prompt instructs model to analyze email newsletters, produce JSON array of topic objects each with title, description, subject, from, and date.
    - Includes conditional instructions for specific sender addresses (custom logic).
    - Uses OpenAI credentials.
  - Inputs: Cleaned message data.
  - Outputs: JSON with topics array.
  - Edge cases:
    - API timeouts or rate limits.
    - Invalid JSON responses.
    - Model misinterpretation of instructions.

#### 1.6 Topics Aggregation and Formatting

- **Overview**: Merges all topic arrays from multiple summaries into a single array, then formats into a styled HTML email template.
- **Nodes Involved**: `Merge`, `Create template`
- **Node Details**:
  - **Merge**
    - Type: `Code` node
    - Configuration:
      - Flattens all topic arrays from batch items into one combined array.
      - Handles missing or null topics safely.
    - Inputs: Summaries from `Message a model` processed in batches.
    - Outputs: Single JSON object with unified `topics` array.
    - Edge cases: Empty input arrays.
  - **Create template**
    - Type: `Code` node
    - Configuration:
      - Iterates over topics to build an HTML block for each.
      - Escapes HTML special characters.
      - Formats a complete HTML document with inline styles for email.
      - Outputs `htmlBody` field for sending.
    - Inputs: Merged topics array.
    - Outputs: Final HTML email body.
    - Edge cases: No topics leads to empty output (workflow stop).

#### 1.7 Email Sending

- **Overview**: Sends the final formatted HTML digest email to the configured recipient.
- **Nodes Involved**: `Send a message`
- **Node Details**:
  - Type: `Gmail` node, operation `send`
  - Configuration:
    - Destination email is hardcoded (`your-email@example.com`).
    - Subject is static (`Your-subject`).
    - Message body is the previously generated HTML.
    - Uses Gmail OAuth2 credentials.
  - Inputs: HTML email body from `Create template`.
  - Outputs: None (end node).
  - Edge cases:
    - Sending failures due to auth or quota.
    - Email formatting issues if HTML malformed.

---

### 3. Summary Table

| Node Name          | Node Type                     | Functional Role                     | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                                         |
|--------------------|-------------------------------|-----------------------------------|----------------------|----------------------|---------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger               | Starts workflow daily             | None                 | Get many messages     |                                                                                                                     |
| Get many messages   | Gmail                         | Retrieves emails from Gmail       | Schedule Trigger     | Loop Over Items       |                                                                                                                     |
| Loop Over Items     | SplitInBatches                | Iterates over each message        | Get many messages    | Merge, Get a message  | ## Iterates over each message                                                                                       |
| Get a message       | Gmail                         | Fetches full message content      | Loop Over Items      | Get message data      |                                                                                                                     |
| Get message data    | Code                          | Extracts HTML, sender, subject, date | Get a message       | Clean                 | ## Clean up the text and forms the final message                                                                    |
| Clean              | Code                          | Cleans and reformats data         | Get message data     | Message a model       | ## Clean up the text and forms the final message                                                                    |
| Message a model     | OpenAI (LangChain OpenAI)      | Summarize email content to JSON   | Clean                | Loop Over Items       |                                                                                                                     |
| Merge              | Code                          | Merges all topic arrays           | Loop Over Items      | Create template       |                                                                                                                     |
| Create template     | Code                          | Formats topics into HTML email    | Merge                | Send a message        |                                                                                                                     |
| Send a message      | Gmail                         | Sends final digest email          | Create template      | None                  |                                                                                                                     |
| Sticky Note        | Sticky Note                   | Workflow usage explanation        | None                 | None                  | ## Try this out!\nSend a number to your Telegram bot (e.g., 2) and get a neatly formatted digest of all Gmail newsletters received since that date. Each email is summarized by an LLM into concise topics, merged into a single Telegram message, automatically split into chunks to fit Telegram limits, and safely formatted as HTML. |
| Sticky Note1       | Sticky Note                   | Explains batch iteration          | None                 | None                  | ## Iterates over each message                                                                                        |
| Sticky Note2       | Sticky Note                   | Explains data cleaning            | None                 | None                  | ## Clean up the text and forms the final message                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Schedule Trigger` node:**
   - Type: Schedule Trigger.
   - Configure trigger to run daily at 16:00 (4 PM).
   - No credentials required.

2. **Create `Get many messages` Gmail node:**
   - Operation: getAll.
   - Filters: 
     - Query with multiple sender addresses using OR.
     - Exclude specific strings.
     - Use dynamic date filter: `after:{{ $now.minus({ days: 1 }).toFormat('yyyy/MM/dd') }}`.
   - Return all results.
   - Connect output of `Schedule Trigger` to this node.
   - Set Gmail OAuth2 credentials.

3. **Create `Loop Over Items` node:**
   - Type: SplitInBatches.
   - Default batch size (1 message per batch).
   - Connect output of `Get many messages` to input.
   - This node allows iterative processing of each message.

4. **Create `Get a message` Gmail node:**
   - Operation: get.
   - Message ID: use expression `={{ $json.id }}` from batch item.
   - Connect output of `Loop Over Items` to this node.
   - Use same Gmail OAuth2 credentials.

5. **Create `Get message data` Code node:**
   - Use provided JavaScript to extract HTML from the email payload, parse sender’s name, subject, and convert date to `DD.MM.YYYY`.
   - Connect output of `Get a message` to this node.

6. **Create `Clean` Code node:**
   - Use provided JavaScript code to:
     - Convert `DD.MM.YYYY` date to `MM.DD`.
     - Filter out entries without HTML or date.
     - Pass relevant data forward.
   - Connect output of `Get message data` to this node.

7. **Create `Message a model` OpenAI node:**
   - Model: `gpt-4.1-mini`.
   - Set prompt with instructions to summarize emails into JSON topics array.
   - Use expressions to pass `subject`, `from`, `date`, and `html` from input JSON.
   - Enable JSON output parsing.
   - Connect output of `Clean` node to this node.
   - Configure OpenAI API credentials.

8. **Connect output of `Message a model` back to `Loop Over Items` node:**
   - This allows processing the next message after summarizing current one.

9. **Create `Merge` Code node:**
   - Use provided JavaScript to flatten all topic arrays from batch items into one combined array.
   - Connect output of `Loop Over Items` (first output) to this node.

10. **Create `Create template` Code node:**
    - Use provided JavaScript to convert merged topics array into a styled HTML email.
    - Connect output of `Merge` node to this node.

11. **Create `Send a message` Gmail node:**
    - Operation: send.
    - Recipient email: set to your desired email (e.g., `your-email@example.com`).
    - Subject: set statically (e.g., `Your-subject`).
    - Message body: use expression `={{ $json.htmlBody }}` from `Create template`.
    - Connect output of `Create template` to this node.
    - Use Gmail OAuth2 credentials.

12. **Add Sticky Notes for documentation:**
    - Add notes to explain iteration, data cleaning, and general workflow usage as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Try this out! Send a number to your Telegram bot (e.g., 2) and get a neatly formatted digest of all Gmail newsletters received since that date. Each email is summarized by an LLM into concise topics, merged into a single Telegram message, automatically split into chunks to fit Telegram limits, and safely formatted as HTML. | Usage explanation from Sticky Note node                         |
| The workflow requires valid Gmail OAuth2 credentials and OpenAI API credentials for GPT-4.1-mini.                                                               | Credential setup                                                  |
| The HTML extraction handles multipart MIME messages safely, including nested parts.                                                                              | Code node `Get message data` implementation                      |
| The date formats are localized to `MM.DD` in the digest for readability.                                                                                         | Code node `Clean` implementation                                 |
| The prompt sent to GPT-4.1-mini includes instructions for language, JSON formatting, and conditional content handling based on sender address.                   | OpenAI node prompt details                                       |

---

**Disclaimer:**  
The text analyzed and described here originates exclusively from an automation workflow created with n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.