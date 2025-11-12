Automatic Jest Test Generation for GitHub PRs with Dual AI Review

https://n8nworkflows.xyz/workflows/automatic-jest-test-generation-for-github-prs-with-dual-ai-review-4013


# Automatic Jest Test Generation for GitHub PRs with Dual AI Review

---

### 1. Workflow Overview

This workflow automates the generation and review of Jest unit tests for React/TypeScript code changes in GitHub pull requests (PRs). It listens to PR events, analyzes the diff of changed `.tsx` files, generates Jest tests via an AI agent, refines those tests through a second AI review step, and finally posts suggested test code as comments on the PR. This supports developers by providing automated, high-quality unit test suggestions without blocking PR workflows.

The workflow is logically divided into these core blocks:

- **1.1 Webhook Event Reception:** Captures GitHub pull request open/update events.
- **1.2 PR Metadata and Diff Retrieval:** Extracts PR details and fetches the unified diff of changed files.
- **1.3 Diff Processing and File Filtering:** Splits the diff into file-specific chunks and filters for `.tsx` files.
- **1.4 Fetch Full File Contents:** Retrieves the latest file contents for each changed `.tsx` file to provide context.
- **1.5 AI Test Generation:** Uses an AI agent to generate Jest tests focused on the diff hunks.
- **1.6 AI Test Review:** A second AI agent reviews and improves the generated tests for style and edge cases.
- **1.7 Comment Construction and Posting:** Formats the AI-reviewed tests and posts them as comments on the PR.

---

### 2. Block-by-Block Analysis

#### 1.1 Webhook Event Reception

**Overview:**  
Listens for GitHub pull request events (`opened` and `synchronize`) to trigger the workflow.

**Nodes Involved:**  
- `Webhook`

**Node Details:**  
- **`Webhook`**  
  - Type: HTTP Webhook  
  - Listens on `/webhook/github/pr-events` for PR open or update events  
  - Input: Incoming HTTP payload from GitHub webhook  
  - Output: Passes GitHub PR event data downstream  
  - Failure modes: Network issues, missed webhook delivery, invalid event format  
  - No special version requirements  

---

#### 1.2 PR Metadata and Diff Retrieval

**Overview:**  
Retrieves detailed PR data, extracts diff URL and repo info, fetches the raw diff text.

**Nodes Involved:**  
- `GH - Get PR`  
- `Extrat API URL, owner and repo`  
- `GET .diff file`  
- `Merge info needed`  
- `Merge`

**Node Details:**  
- **`GH - Get PR`**  
  - Type: GitHub node  
  - Action: Get Pull Request details (owner, repo, pull_number, merge_commit_sha)  
  - Inputs: PR number and repo info from webhook  
  - Outputs: Full PR metadata including `diff_url`  
  - Retries enabled for robustness  
  - Potential failures: Auth errors, rate limits, invalid PR number  
  
- **`Extrat API URL, owner and repo`**  
  - Type: Code node (JavaScript)  
  - Extracts `diff_url`, `owner`, `repo`, and `merge_commit_sha` from PR data  
  - Key expressions parse webhook and PR response JSON  
  - Outputs these values for downstream HTTP request and API calls  
  - Failure mode: Parsing errors if expected fields missing  
  
- **`GET .diff file`**  
  - Type: HTTP Request  
  - GET request to the extracted `diff_url` to fetch unified diff text of PR changes  
  - Outputs raw diff body text  
  - Edge cases: HTTP errors, network timeouts, invalid URLs  
  
- **`Merge info needed`**  
  - Type: Merge node  
  - Merges outputs from previous nodes to consolidate data for next steps  
  - Ensures data synchronization for multiple inputs  
  
- **`Merge`**  
  - Type: Merge node  
  - Combines data streams before test generation steps to unify PR info and diffs  

---

#### 1.3 Diff Processing and File Filtering

**Overview:**  
Splits the unified diff into separate file diffs and filters to keep only React `.tsx` files.

**Nodes Involved:**  
- `Splits text into array`  
- `FIlter for components`  
- `Merge info needed`  
- `Get File name`

**Node Details:**  
- **`Splits text into array`**  
  - Type: Code (JavaScript)  
  - Splits raw diff text by `"diff --git"` delimiter into array of file diffs  
  - Output: Array of file-specific diff chunks  
  - Edge cases: Unexpected diff format could cause split errors  
  
- **`FIlter for components`**  
  - Type: Code (JavaScript)  
  - Filters diff chunks to retain only those where the file path ends with `.tsx`  
  - Uses regex `/\.tsx$/` on file path extracted from diff header  
  - Outputs filtered array  
  - Edge cases: Missing file path, malformed diff chunks  
  
- **`Merge info needed`**  
  - Type: Merge node  
  - Used to combine filtered diff chunks with metadata for subsequent file content fetching  
  
- **`Get File name`**  
  - Type: Code (JavaScript)  
  - Extracts the exact file name/path from each filtered diff chunk for API calls  
  - Output: file names for fetching content  
  - Edge cases: Incorrect extraction if diff chunk malformed  

---

#### 1.4 Fetch Full File Contents

**Overview:**  
Fetches the latest contents of each changed `.tsx` file from GitHub to provide full context for test generation.

**Nodes Involved:**  
- `Get Files`  
- `Get Repo`

**Node Details:**  
- **`Get Files`**  
  - Type: GitHub Tool node  
  - Fetches blob/file content of the specified file at the PR’s merge commit SHA or latest branch commit  
  - Input: Repository info and file path from previous nodes  
  - Output: Raw file content  
  - Failure modes: File missing, permissions error, rate limits  
  
- **`Get Repo`**  
  - Type: GitHub Tool node  
  - Retrieves repository metadata if needed for context or API calls  
  - Output: Repo info objects  
  - Generally stable; auth errors possible  

---

#### 1.5 AI Test Generation

**Overview:**  
Generates Jest unit tests focusing on the behaviors changed in the diff hunks using an AI language model.

**Nodes Involved:**  
- `o3-mini` (OpenAI chat model)  
- `Structured Output Parser`  
- `Test Maker`

**Node Details:**  
- **`o3-mini`**  
  - Type: LangChain OpenAI Chat model node  
  - Configured with OpenAI API credentials  
  - Role: Provides base AI language model for test generation  
  - Input: Prompt with diff hunks and file context  
  - Output: Raw AI-generated test code  
  - Failure: API quota, network, response parsing  
  
- **`Structured Output Parser`**  
  - Type: LangChain structured output parser  
  - Parses AI raw output into structured JSON or code blocks for downstream use  
  - Ensures output matches expected Jest test format  
  
- **`Test Maker`**  
  - Type: LangChain AI Agent node  
  - Orchestrates prompt, input from GitHub files and diff, and OpenAI model to generate Jest tests  
  - Receives inputs from `o3-mini` and `Get Files` nodes  
  - Output: Initial Jest test suite code focused on changed behavior  
  - Edge cases: Incomplete or incoherent AI output, prompt failures  

---

#### 1.6 AI Test Review

**Overview:**  
Refines the initially generated Jest tests by reviewing for readability, edge cases, and naming conventions with a second AI pass.

**Nodes Involved:**  
- `gpt4.1-mini` (OpenAI chat model)  
- `Structured Output Parser1`  
- `Code Reviewer`

**Node Details:**  
- **`gpt4.1-mini`**  
  - Type: LangChain OpenAI Chat model node  
  - Uses a stronger or different OpenAI model for review pass  
  - Input: Initial Jest tests and file content  
  - Output: Polished test suite code  
  - Failure modes: API quota, network issues  
  
- **`Structured Output Parser1`**  
  - Parses refined AI output into structured code blocks or JSON  
  - Ensures correct formatting for comment posting  
  
- **`Code Reviewer`**  
  - LangChain AI Agent node  
  - Coordinates review prompt, inputs from `gpt4.1-mini` and GitHub repo/files  
  - Output: Final polished Jest test code  

---

#### 1.7 Comment Construction and Posting

**Overview:**  
Wraps the polished test code in TypeScript code fences and posts it as a comment on the PR's GitHub issue thread.

**Nodes Involved:**  
- `Prepare for POST`  
- `POST Comments`

**Node Details:**  
- **`Prepare for POST`**  
  - Type: Code (JavaScript)  
  - Constructs the JSON payload for GitHub API comment creation  
  - Wraps tests in markdown code fences:  
    ```ts  
    // generated tests...  
    ```  
  - Output: `{ "body": "<tests>" }` JSON object for POST request  
  
- **`POST Comments`**  
  - Type: HTTP Request node  
  - Sends POST request to GitHub API endpoint:  
    `https://api.github.com/repos/{owner}/{repo}/issues/{pull_number}/comments`  
  - Uses GitHub token with permission to post comments  
  - Edge cases: Auth failures, API rate limits, network errors

---

### 3. Summary Table

| Node Name                 | Node Type                               | Functional Role                         | Input Node(s)                  | Output Node(s)                 | Sticky Note                |
|---------------------------|---------------------------------------|---------------------------------------|-------------------------------|-------------------------------|----------------------------|
| Webhook                   | HTTP Webhook                          | Entry point; receives PR events       |                               | Extrat API URL, owner and repo|                            |
| Extrat API URL, owner and repo | Code (JS)                         | Extracts diff_url, owner, repo info   | Webhook                       | GET .diff file, Merge info needed, Merge |                      |
| GH - Get PR               | GitHub                                | Retrieves PR metadata                  |                               |                               |                            |
| GET .diff file            | HTTP Request                         | Fetches raw unified diff text          | Extrat API URL, owner and repo | Splits text into array        |                            |
| Splits text into array    | Code (JS)                            | Splits diff by file                    | GET .diff file                | FIlter for components         |                            |
| FIlter for components     | Code (JS)                            | Keeps only `.tsx` file diffs           | Splits text into array        | Merge info needed             |                            |
| Merge info needed         | Merge                                | Combines filtered diffs and metadata  | Extrat API URL..., FIlter for components | Get File name               |                            |
| Get File name             | Code (JS)                            | Extracts file name/path from diff     | Merge info needed             | Test Maker                   |                            |
| Get Files                 | GitHub Tool                         | Fetches file contents for each file   |                               | Test Maker, Code Reviewer     |                            |
| Get Repo                  | GitHub Tool                         | Retrieves repo metadata                |                               | Test Maker, Code Reviewer     |                            |
| o3-mini                   | LangChain OpenAI Chat                | AI model for generating tests         |                               | Test Maker                   |                            |
| Structured Output Parser  | LangChain Output Parser              | Parses AI test generation output      | o3-mini                      | Test Maker                   |                            |
| Test Maker                | LangChain AI Agent                   | Generates Jest tests based on diffs   | o3-mini, Get Files, Get Repo  | Code Reviewer                |                            |
| gpt4.1-mini               | LangChain OpenAI Chat                | AI model for reviewing tests          |                               | Code Reviewer                |                            |
| Structured Output Parser1 | LangChain Output Parser              | Parses AI reviewed test output        | gpt4.1-mini                  | Code Reviewer                |                            |
| Code Reviewer             | LangChain AI Agent                   | Reviews and refines generated tests   | gpt4.1-mini, Get Files, Get Repo | Merge                      |                            |
| Merge                     | Merge                                | Merges final outputs for posting      | Extrat API URL..., Code Reviewer | Prepare for POST            |                            |
| Prepare for POST          | Code (JS)                            | Builds JSON payload for PR comment    | Merge                        | POST Comments                |                            |
| POST Comments             | HTTP Request                         | Posts generated tests as PR comments  | Prepare for POST             |                               |                            |
| 2nd Node for Testing      | Code (JS)                            | (Unused/test node)                     |                               |                               |                            |
| Sticky Note               | Sticky Note                         | (No content)                          |                               |                               |                            |
| Sticky Note1              | Sticky Note                         | (No content)                          |                               |                               |                            |
| Sticky Note2              | Sticky Note                         | (No content)                          |                               |                               |                            |
| Sticky Note3              | Sticky Note                         | (No content)                          |                               |                               |                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: HTTP Webhook  
   - Path: `/webhook/github/pr-events`  
   - Events: Configure GitHub webhook to trigger on `pull_request.opened` and `pull_request.synchronize`

2. **Create GitHub Node "GH - Get PR"**  
   - Action: Get Pull Request  
   - Inputs: `owner`, `repo`, `pull_number` extracted from webhook payload  
   - Enable retry on failure

3. **Create Code Node "Extrat API URL, owner and repo"**  
   - JavaScript extracts `diff_url`, `owner`, `repo`, `merge_commit_sha` from PR payload  
   - Outputs these values for subsequent nodes

4. **Create HTTP Request Node "GET .diff file"**  
   - Method: GET  
   - URL: Use extracted `diff_url`  
   - Response Format: Text (unified diff)  

5. **Create Code Node "Splits text into array"**  
   - JavaScript splits diff text on `"diff --git"` string into array of diffs  

6. **Create Code Node "FIlter for components"**  
   - Filters array to keep only diff chunks for files ending with `.tsx` (using regex)  

7. **Create Merge Node "Merge info needed"**  
   - Merge outputs from `Extrat API URL, owner and repo` and filtered diffs  

8. **Create Code Node "Get File name"**  
   - Extract file path/name from each diff chunk for API calls  

9. **Create GitHub Tool Node "Get Files"**  
   - Fetch file contents for each `.tsx` file at PR commit SHA  

10. **Create GitHub Tool Node "Get Repo"**  
    - Retrieve repository metadata  

11. **Create LangChain OpenAI Chat Node "o3-mini"**  
    - Configure with OpenAI credentials  
    - Model: e.g., `gpt-3.5-turbo` or similar for test generation  

12. **Create LangChain Output Parser Node "Structured Output Parser"**  
    - Parses AI output structure from `o3-mini`  

13. **Create LangChain AI Agent Node "Test Maker"**  
    - Inputs: AI language model node, GitHub file contents, and repo info  
    - Prompt: `"Generate Jest unit tests covering only the behaviors changed in these diff hunks."`  

14. **Create LangChain OpenAI Chat Node "gpt4.1-mini"**  
    - Configure with OpenAI credentials  
    - Model: e.g., `gpt-4` or advanced for review pass  

15. **Create LangChain Output Parser Node "Structured Output Parser1"**  
    - Parses AI output from `gpt4.1-mini`  

16. **Create LangChain AI Agent Node "Code Reviewer"**  
    - Inputs: AI language model node, initial test code, file contents, repo info  
    - Prompt: `"Review and improve these tests for readability, edge-cases, and naming conventions."`  

17. **Create Merge Node "Merge"**  
    - Combine outputs from `Extrat API URL, owner and repo` and `Code Reviewer` for posting  

18. **Create Code Node "Prepare for POST"**  
    - Wrap reviewed tests in TypeScript markdown code fences  
    - Build JSON object `{ "body": "<tests>" }` for GitHub API  

19. **Create HTTP Request Node "POST Comments"**  
    - Method: POST  
    - URL: `https://api.github.com/repos/{owner}/{repo}/issues/{pull_number}/comments`  
    - Body: Use output from `Prepare for POST`  
    - Auth: GitHub token with `repo` permissions for commenting  

20. **Connect nodes according to the flow:**  
    - `Webhook` → `Extrat API URL, owner and repo` → `GET .diff file` → `Splits text into array` → `FIlter for components` → `Merge info needed` → `Get File name` → `Test Maker`  
    - `Get Files` and `Get Repo` feed into `Test Maker` and `Code Reviewer`  
    - `o3-mini` feeds into `Test Maker`  
    - `gpt4.1-mini` feeds into `Code Reviewer`  
    - `Code Reviewer` output merges and flows to `Prepare for POST` → `POST Comments`

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow uses a two-stage AI approach mimicking Test-Driven Development and code review processes.  | Design rationale section                                                                                 |
| GitHub token used should be scoped minimally to read PRs and post comments only.                     | Security best practice                                                                                   |
| The workflow operates asynchronously, ensuring developers are not blocked by test generation delays.| Non-blocking PR comments design                                                                          |
| Mermaid diagram illustrating node connections helps visualize workflow logic.                        | Provided in workflow description                                                                         |
| For detailed AI prompt tuning or LangChain integration, see n8n documentation and LangChain guides.| https://docs.n8n.io/nodes/automation/langchain/                                                       |

---

This document fully describes the workflow, enabling reproduction, modification, and troubleshooting by technical users and automation agents.