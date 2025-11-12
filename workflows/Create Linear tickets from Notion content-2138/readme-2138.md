Create Linear tickets from Notion content

https://n8nworkflows.xyz/workflows/create-linear-tickets-from-notion-content-2138


# Create Linear tickets from Notion content

### 1. Workflow Overview

This workflow automates the creation of Linear tickets from tasks defined within a Notion page. It is designed for teams that draft and organize feature requests or issues in Notion but track and manage work execution in Linear. The workflow extracts individual issues from Notion’s structured content blocks, processes and formats their details, assigns them to appropriate team members, creates corresponding issues in Linear, and links back the created issues into the original Notion blocks for traceability.

**Target use cases:**  
- Product teams defining features and tasks in Notion with rich content.  
- Engineering teams requiring all work items to be tracked in Linear.  
- Maintaining live bi-directional references between Notion and Linear.  
- Incremental imports avoiding duplicated issues when re-run.

**Logical Blocks:**  
- **1.1 Input Reception:** Receives user input via a form and fetches initial data from Linear and Notion.  
- **1.2 Data Filtering and Parsing:** Filters relevant Notion blocks and extracts their content, converting it to Markdown.  
- **1.3 Issue Title and Assignee Resolution:** Determines issue titles and assignees from content and optionally shortens titles via AI.  
- **1.4 Issue Description Preparation:** Aggregates and formats issue content into detailed descriptions.  
- **1.5 Linear Issue Creation and Notion Update:** Creates issues in Linear, retrieves their URLs, and updates Notion blocks to include back-links.  
- **1.6 Response Handling:** Sends responses to the user or halts with error messages when necessary.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block starts the workflow by receiving a form submission with the Notion page URL and the target Linear team name. It then fetches team and project details from Linear for the specified team.

- **Nodes Involved:**  
  - n8n Form Trigger  
  - Fetch Linear team details  
  - Team missing? (If condition)  
  - Respond with error1 (Error response if team not found)  
  - Set team ID  
  - Get issues (Retrieve Notion page content)

- **Node Details:**

  - **n8n Form Trigger**  
    - Type: Form Trigger  
    - Role: Entry point; collects Notion page URL and Linear team name.  
    - Configuration: Form with two required fields — Notion page URL (string), Linear team name (dropdown with predefined options).  
    - Input: User form submission.  
    - Output: JSON containing form data.  
    - Edge Cases: User submits invalid URL or team name not in dropdown.

  - **Fetch Linear team details**  
    - Type: GraphQL  
    - Role: Retrieves Linear team and member details by filtering teams based on the submitted team name.  
    - Configuration: Query filters teams containing the given name, retrieves team ID, members (id, name, email), and projects.  
    - Input: Team name from form trigger.  
    - Output: JSON data of matching teams.  
    - Credentials: HTTP Header Auth with Linear API token.  
    - Edge Cases: No matching team found, API errors, authentication failure, network timeout.

  - **Team missing?**  
    - Type: If  
    - Role: Checks if any team was found in the previous node (teams array length < 1).  
    - Output:  
      - True: Branch to Respond with error1.  
      - False: Branch to Set team ID.

  - **Respond with error1**  
    - Type: Respond to Webhook  
    - Role: Returns a JSON error message indicating the specified Linear team was not found.  
    - Input: Triggered if no team found.  
    - Output: JSON error response.  
    - Edge Cases: N/A.

  - **Set team ID**  
    - Type: Set  
    - Role: Extracts the first matching team ID from the fetched data for further use.  
    - Input: Linear team details JSON.  
    - Output: JSON with `team_id` field.  
    - Edge Cases: Empty data array; guarded by previous If node.

  - **Get issues**  
    - Type: Notion  
    - Role: Retrieves all content blocks from the specified Notion page URL.  
    - Configuration: Block ID set using the Notion page URL from form, fetchAll enabled.  
    - Credentials: Notion API integration.  
    - Output: Full block data from Notion page.  
    - Edge Cases: Notion page not shared with integration, network errors.  
    - On error: Continues with error output connected to Respond with error.

  - **Respond with error**  
    - Type: Respond to Webhook  
    - Role: Returns an error if Notion page content could not be fetched.  
    - Output: JSON error message.  

---

#### 2.2 Data Filtering and Parsing

- **Overview:**  
  Filters the Notion page content to identify unimported, unchecked to_do blocks (tickets), then converts their contents into Markdown format suitable for Linear issue descriptions.

- **Nodes Involved:**  
  - Unimported, unchecked to_do blocks only (Filter)  
  - Loop Over Items (Split in Batches)  
  - Convert contents to Markdown (Code)  
  - Aggregate

- **Node Details:**

  - **Unimported, unchecked to_do blocks only**  
    - Type: Filter  
    - Role: Select only blocks which are:  
      - type `to_do`  
      - not already imported (text does not start with “[In Linear]”)  
      - unchecked (`checked` is false)  
    - Input: All blocks from Notion page.  
    - Output: Filtered list of to_do items eligible for import.  
    - Edge Cases: Incorrect block types, missing `to_do` field, text array empty.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes issues one by one to manage resource usage and API rate limits.  
    - Input: Filtered to_do blocks.  
    - Output: Individual items per batch.  
    - Edge Cases: Batch size defaults, potential loss if batches interrupted.

  - **Convert contents to Markdown**  
    - Type: Code  
    - Role: Converts various Notion block types (bulleted list, numbered list, toggle, to_do, images, videos, etc.) into Markdown strings with appropriate indentation reflecting content hierarchy.  
    - Key expressions: Uses nested loops to track parent block IDs to determine indentation.  
    - Input: Root content and individual blocks from Notion.  
    - Output: Markdown-formatted content strings for each block.  
    - Edge Cases: Unexpected block types, missing text fields, malformed content.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Combines all the Markdown lines from the block content into a single Markdown description for the issue.  
    - Input: Markdown lines from the code node.  
    - Output: Aggregated Markdown string array.  
    - Edge Cases: Empty content, aggregation failure.

---

#### 2.3 Issue Title and Assignee Resolution

- **Overview:**  
  Extracts the issue title and optionally the assignee from the first line of the to_do block text. If the title is too long, it is shortened using OpenAI GPT-4. The assignee is matched against team members fetched from Linear.

- **Nodes Involved:**  
  - Set assignee and title (Code)  
  - Shorten title (OpenAI LangChain node)  
  - Fetch Linear team details (already in 2.1, used here for members list)

- **Node Details:**

  - **Set assignee and title**  
    - Type: Code  
    - Role:  
      - Parses the first line of the to_do text for an optional assignee in square brackets (e.g., `[Omar] Task title`).  
      - Extracts assignee fragment and title.  
      - Constructs new content for updating the Notion block to include a link to the created Linear issue later.  
      - Matches assignee fragment with Linear team members (case-insensitive prefix match).  
    - Key expressions: Regex parsing `^(\\[[^\\]]*\\]\\s)?(.+)$` for assignee and title.  
    - Input: Single to_do block JSON, Linear team members list.  
    - Output: JSON enriched with `title`, `assignee_fragment`, `assignee` (member object), and `new_content` (for Notion update).  
    - Edge Cases: No assignee fragment, no matching member, multiple matching members (picks first), empty or malformed first line.

  - **Shorten title**  
    - Type: OpenAI LangChain  
    - Role: Calls GPT-4 to shorten the issue title if it exceeds 70 characters, with a max length of 150 chars output. If already short, returns original text.  
    - Configuration:  
      - Prompt instructs concise rephrasing, returns only the text.  
      - Model: GPT-4.  
    - Input: Title from previous node.  
    - Output: Possibly shortened title string.  
    - Credentials: OpenAI API key.  
    - Edge Cases: API rate limiting, network errors, unexpected model response.

---

#### 2.4 Issue Description Preparation

- **Overview:**  
  Prepares the final issue data payload including the title and a detailed description that includes a link back to the Notion block and the aggregated Markdown content. Also adds warnings if an assignee fragment was specified but no matching assignee was found.

- **Nodes Involved:**  
  - Prepare issue data (Set)

- **Node Details:**

  - **Prepare issue data**  
    - Type: Set  
    - Role:  
      - Sets `title` to either the original or shortened title depending on length.  
      - Composes the `description` string: includes a markdown link to the original Notion block (with URL constructed dynamically), a warning if assignee fragment was unmatched, and the aggregated Markdown content.  
    - Key expressions: Conditional expressions to pick title, build description string concatenating multiple sources.  
    - Input: Title, assignee info, aggregated Markdown.  
    - Output: JSON with `title` and `description` fields ready for Linear.  
    - Edge Cases: Missing or malformed URLs, empty Markdown content.

---

#### 2.5 Linear Issue Creation and Notion Update

- **Overview:**  
  Creates issues in Linear using the prepared data, fetches the created issue’s URL, and updates the original Notion to_do block to link back to the Linear issue.

- **Nodes Involved:**  
  - Create linear issue (Linear node)  
  - Get issue URL (GraphQL)  
  - Add link to Notion block (HTTP Request)  
  - Loop Over Items (Split in Batches continuation)

- **Node Details:**

  - **Create linear issue**  
    - Type: Linear node  
    - Role: Creates a new issue in Linear with title, team ID, assignee ID, and description.  
    - Configuration: Uses credentials for Linear API.  
    - Input: Title, description, team ID, assignee ID.  
    - Output: Created issue metadata including issue ID.  
    - Edge Cases: API errors, validation errors, assignee ID missing.

  - **Get issue URL**  
    - Type: GraphQL  
    - Role: Retrieves the URL of the newly created Linear issue using its ID.  
    - Configuration: Query issues by ID, fetching only the URL.  
    - Input: Issue ID from the previous node.  
    - Output: Issue URL JSON.  
    - Edge Cases: API failure, invalid issue ID.

  - **Add link to Notion block**  
    - Type: HTTP Request  
    - Role: Updates the original Notion to_do block by PATCHing the `to_do` property to prepend a link to the Linear issue URL, marking it as imported.  
    - Configuration: PATCH to Notion API `blocks/{block_id}` endpoint, JSON body uses `new_content` with dynamic replacement of `$url` by the issue URL.  
    - Credentials: Notion API integration with appropriate scopes.  
    - Input: Block ID from Loop Over Items, new content with Linear issue link.  
    - Output: HTTP response from Notion API.  
    - Edge Cases: Notion API rate limits, permission errors, malformed JSON body.

  - **Loop Over Items** (continuation)  
    - After updating Notion, loops back to process next item until done.

---

#### 2.6 Response Handling

- **Overview:**  
  Provides user feedback on success or failure of the workflow execution.

- **Nodes Involved:**  
  - Respond with error  
  - Respond with error1

- **Node Details:**

  - **Respond with error**  
    - Returns error JSON if Notion page content could not be fetched.

  - **Respond with error1**  
    - Returns error JSON if Linear team not found.

---

### 3. Summary Table

| Node Name                         | Node Type                    | Functional Role                                    | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                              |
|----------------------------------|------------------------------|---------------------------------------------------|--------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------|
| n8n Form Trigger                 | Form Trigger                 | Receives Notion page URL and Linear team name     | N/A                            | Fetch Linear team details      | # Try me out<br>1. In the form trigger node, enter the names of your Linear team(s) to display on the form <br>2. Make sure your Notion page is formatted according to the [spec](https://www.notion.so/n8n/Template-for-design-review-automatic-Linear-import-8848dd09892341969faedd1313eea586?pvs=4) and shared with your Notion integration<br>2. Click the 'test workflow' button below |
| Fetch Linear team details        | GraphQL                     | Fetches Linear team and member details              | n8n Form Trigger               | Team missing?                 | ### Get the issues to create from Notion (and load Linear team details)                                |
| Team missing?                   | If                          | Checks if team exists                               | Fetch Linear team details      | Respond with error1 / Set team ID |                                                                                                        |
| Respond with error1              | Respond to Webhook          | Returns error if team not found                      | Team missing? (true branch)    | N/A                           |                                                                                                        |
| Set team ID                    | Set                         | Sets team ID for use in later steps                  | Team missing? (false branch)   | Get issues                    |                                                                                                        |
| Get issues                     | Notion                      | Retrieves all blocks from Notion page                | Set team ID                   | Unimported, unchecked to_do blocks only / Respond with error |                                                                                                        |
| Respond with error              | Respond to Webhook          | Returns error if Notion page content fetch failed    | Get issues (error output)      | N/A                           |                                                                                                        |
| Unimported, unchecked to_do blocks only | Filter                    | Filters unimported, unchecked to_do blocks           | Get issues                    | Loop Over Items               |                                                                                                        |
| Loop Over Items               | SplitInBatches              | Processes each to_do block individually               | Unimported, unchecked to_do blocks only / Add link to Notion block | Set assignee and title / null |                                                                                                        |
| Set assignee and title          | Code                        | Parses title and assignee from to_do block            | Loop Over Items               | Shorten title                 | ### Figure out issue assignee and title (shortening if necessary)                                      |
| Shorten title                  | OpenAI LangChain            | Shortens long titles via GPT-4                        | Set assignee and title        | Get issue contents            |                                                                                                        |
| Get issue contents             | Notion                      | Retrieves all nested content blocks for current issue | Shorten title                 | Set page URL                  |                                                                                                        |
| Set page URL                   | Set                         | Sets Notion page URL and root block content          | Get issue contents            | Convert contents to Markdown  |                                                                                                        |
| Convert contents to Markdown   | Code                        | Converts Notion blocks to Markdown format             | Set page URL                  | Aggregate                    | ### Compose issue description                                                                          |
| Aggregate                     | Aggregate                   | Aggregates Markdown lines into single description     | Convert contents to Markdown   | Prepare issue data            |                                                                                                        |
| Prepare issue data             | Set                         | Prepares title and description for Linear issue       | Aggregate                    | Create linear issue           | ### Create issue and add link to it in Notion                                                          |
| Create linear issue            | Linear                      | Creates issue in Linear with prepared data             | Prepare issue data            | Get issue URL                |                                                                                                        |
| Get issue URL                 | GraphQL                     | Fetches URL of created Linear issue                    | Create linear issue           | Add link to Notion block     |                                                                                                        |
| Add link to Notion block       | HTTP Request                | Updates Notion to_do block with Linear issue link      | Get issue URL                 | Loop Over Items              |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a **Form Trigger** node  
- Name: `n8n Form Trigger`  
- Parameters:  
  - Path: unique webhook path (e.g., `5a631d63-f899-4967-acad-69924674e96a`)  
  - Form fields:  
    - Notion page URL (required, text)  
    - Linear team name (required, dropdown with values: AI, Adore, Payday, NODES)  
  - Response mode: response node  
  - Form description: Link to Notion formatting spec  

**Step 2:** Add a **GraphQL** node to fetch Linear team details  
- Name: `Fetch Linear team details`  
- Parameters:  
  - Query:  
    ```graphql
    query GetTeamsAndProjects {
      teams(filter: {name: {contains: "{{ $json['Linear team name'] }}"}}) {
        nodes {
          id
          name
          members {
            nodes {
              id
              name
              email
            }
          }
          projects {
            nodes {
              id
              name
              description
            }
          }
        }
      }
    }
    ```  
  - Endpoint: `https://api.linear.app/graphql`  
  - Request method: GET  
  - Authentication: HTTP Header Auth (Linear API token)  

**Step 3:** Add an **If** node named `Team missing?`  
- Condition: Check if `{{$json.data.teams?.nodes?.length}} < 1` (boolean true)  
- True branch leads to error response node.  
- False branch leads to next step.

**Step 4:** Add **Respond to Webhook** node named `Respond with error1`  
- Response body: JSON string indicating team not found (using team name from form input).

**Step 5:** Add a **Set** node named `Set team ID`  
- Assign `team_id` from first team node ID in the GraphQL response.

**Step 6:** Add a **Notion** node named `Get issues`  
- Operation: Get All Blocks  
- Block ID: use Notion page URL from form trigger as URL mode input  
- Return all: true  
- Credential: Notion API integration with read permissions for your workspace  

**Step 7:** Add **Respond to Webhook** node named `Respond with error`  
- Response body: JSON error indicating Notion content fetch failure.

**Step 8:** Add a **Filter** node named `Unimported, unchecked to_do blocks only`  
- Conditions:  
  - `type` equals `to_do`  
  - Text does not start with `[In Linear]`  
  - `to_do.checked` is false  

**Step 9:** Add a **SplitInBatches** node named `Loop Over Items`  
- Use default batch size (1) for sequential processing.

**Step 10:** Add a **Code** node named `Set assignee and title`  
- Runs once per item (mode: runOnceForEachItem)  
- Logic:  
  - Extracts first line text from to_do block.  
  - Parses optional assignee in square brackets and title.  
  - Matches assignee fragment to Linear team members (case-insensitive prefix).  
  - Prepares new content for Notion update with placeholder `$url`.  

**Step 11:** Add an **OpenAI LangChain** node named `Shorten title`  
- Model: GPT-4  
- Prompt: Instruct to shorten text if longer than 70 chars to max 150 chars, else return original.  
- Input: Title from previous code node.  
- Credential: OpenAI API key.

**Step 12:** Add a **Notion** node named `Get issue contents`  
- Operation: Get All Blocks nested under current to_do block ID (dynamic from item).  
- Credential: Notion API integration.  

**Step 13:** Add a **Set** node named `Set page URL`  
- Assign:  
  - `page_url`: base Notion page URL (remove query params)  
  - `root_content`: content of the root block (from "Set assignee and title")  
  - `root_id`: current block ID  

**Step 14:** Add a **Code** node named `Convert contents to Markdown`  
- Converts all nested Notion blocks into Markdown format with indentation reflecting block hierarchy.  
- Supports block types: bulleted list, numbered list, toggle, to_do, image, video, and paragraphs with links.

**Step 15:** Add an **Aggregate** node named `Aggregate`  
- Aggregates all Markdown lines into a single array for description.

**Step 16:** Add a **Set** node named `Prepare issue data`  
- Assigns:  
  - `title`: use shortened title if original longer than 70 chars, else original  
  - `description`: markdown including link to original Notion block (constructed URL), warning if assignee fragment unmatched, and aggregated Markdown content.

**Step 17:** Add a **Linear** node named `Create linear issue`  
- Parameters:  
  - Title: from `title` field  
  - Team ID: from `team_id`  
  - Additional fields: Assignee ID (if available), Description  
- Credential: Linear API credentials.

**Step 18:** Add a **GraphQL** node named `Get issue URL`  
- Query the created issue’s URL by issue ID returned from previous node.

**Step 19:** Add an **HTTP Request** node named `Add link to Notion block`  
- PATCH request to Notion API `blocks/{block_id}` endpoint  
- Body: JSON updating `to_do` content with a prepended link to the Linear issue URL (replacing `$url` in prepared new_content).  
- Header includes `Notion-Version: 2022-06-28`  
- Credential: Notion API integration.

**Step 20:** Connect `Add link to Notion block` output back to `Loop Over Items` to continue processing all issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                                                                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| The Notion page must be formatted according to the specified template for defining issues, including use of to_do blocks with optional assignee syntax in square brackets `[Assignee] Title`.                                                                                                                          | [Notion Template Example](https://www.notion.so/Template-for-importing-Notion-blocks-as-issues-in-Linear-8848dd09892341969faedd1313eea586?pvs=21)                                                                                                |
| The workflow uses GPT-4 to shorten long titles, requiring a valid OpenAI API key with access to GPT-4.                                                                                                                                                                                                                 | OpenAI API documentation                                                                                                                                                                                                                         |
| The Linear API token used must have permissions to query teams, members, and create issues.                                                                                                                                                                                                                            | Linear API docs: https://developers.linear.app/docs/graphql/getting-started                                                                                                                                                                      |
| Notion API integration must have read and write permissions on the target workspace and pages.                                                                                                                                                                                                                        | Notion API docs: https://developers.notion.com/reference/patch-block                                                                                                                                                                             |
| The workflow avoids duplicate imports by checking for the `[In Linear]` prefix in to_do block text.                                                                                                                                                                                                                   | Internal logic detail                                                                                                                                                                                                                            |
| For incremental imports, ensure the Notion page is updated exclusively via the workflow to keep link consistency.                                                                                                                                                                                                     | Operational best practice                                                                                                                                                                                                                        |
| The workflow includes error handling to respond with user-friendly messages if the Linear team is not found or if the Notion page is not accessible by the integration.                                                                                                                                              | Workflow robustness improvement                                                                                                                                                                                                                  |

---

This comprehensive reference document enables advanced users and AI agents to fully understand, reproduce, and adapt the "Create Linear tickets from Notion content" workflow in n8n.