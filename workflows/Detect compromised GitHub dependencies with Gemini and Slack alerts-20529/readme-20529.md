Detect compromised GitHub dependencies with Gemini and Slack alerts

https://n8nworkflows.xyz/workflows/detect-compromised-github-dependencies-with-gemini-and-slack-alerts-20529


# Detect compromised GitHub dependencies with Gemini and Slack alerts

### 1. Workflow Overview

This workflow automates the detection of compromised or vulnerable open-source dependencies (npm, PyPI, Maven, Go, Rust, etc.) across GitHub repositories. Running on a scheduled hourly basis, it queries the GitHub Advisory Database for recent malware and critical advisories, downloads software bill of materials (SBOMs) for user repositories, performs version-matching, uses Google Gemini to generate custom incident response plans, and alerts security teams via Slack while maintaining a de-duplication state in an n8n Data Table.

The system is organized into the following logical blocks:
- **1.1 Initialization & State Management:** Triggers the execution schedule, defines global scan parameters, and initializes/loads the persistent incident tracking data table.
- **1.2 Advisory Retrieval & Filtering:** Dynamically constructs query parameters based on lookback windows and severity configurations, fetches security advisories from the GitHub API, and indexes active threats.
- **1.3 Repository Enumeration & Dependency Matching:** Pulls repository lists, filters out unwanted targets (forks, archives, name matches), downloads repository dependency graphs (SBOMs), and cross-references dependencies with active advisory range rules while excluding previously reported alerts.
- **1.4 AI Incident Analysis:** Evaluates confirmed threats using a Large Language Model to structure technical risk assessments, remediation actions, and compromise checks.
- **1.5 Alerting, Logging, & Issue Tracking:** Dispatches formatted notifications to Slack, records processed incidents in the Data Table, and optionally generates automated issues in private GitHub repositories.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & State Management
- **Overview:** Sets up the execution cadence, establishes core operational parameters, and provisions or reads historical incident data to prevent duplicate alerting.
- **Nodes Involved:** 
  - `Every Hour`
  - `Settings`
  - `Create Incident Table`
  - `Load Incidents`
- **Node Details:**
  - **Every Hour** (`n8n-nodes-base.scheduleTrigger`)
    - *Type and Technical Role:* Trigger node.
    - *Configuration:* Interval set to every 1 hour.
    - *Input/Output:* No inputs; outputs execution payload to `Settings`.
    - *Edge Cases:* Missed triggers if n8n instance is offline.
  - **Settings** (`n8n-nodes-base.set`)
    - *Type and Technical Role:* Set node used as a centralized configuration store.
    - *Configuration:* Defines global variables (`github_owner`, `owner_type`, `lookback_days`, `include_critical_vulnerabilities`, `repo_name_filter`, `include_forks`, `max_repositories`, `create_github_issues`).
    - *Input/Output:* Input from `Every Hour`; outputs settings object to `Create Incident Table`.
    - *Edge Cases:* Invalid ownership types or incorrectly formatted target strings.
  - **Create Incident Table** (`n8n-nodes-base.dataTable`)
    - *Type and Technical Role:* Data Table management node.
    - *Configuration:* Creates table `supply_chain_incidents` with schema (`key`, `ghsa_id`, `advisory_type`, `severity`, `repo`, `ecosystem`, `package`, `version`, `fixed_version`, `confirmed`, `detected_at`, `status`, `summary`) if it does not exist (`executeOnce: true`).
    - *Input/Output:* Input from `Settings`; outputs to `Load Incidents`.
    - *Edge Cases:* Database locking or permissions errors.
  - **Load Incidents** (`n8n-nodes-base.dataTable`)
    - *Type and Technical Role:* Data Table row retrieval node.
    - *Configuration:* Retrieves all existing rows from `supply_chain_incidents` (`executeOnce: true`, `alwaysOutputData: true`).
    - *Input/Output:* Input from `Create Incident Table`; outputs historical records to `Build Advisory Queries`.

#### 2.2 Advisory Retrieval & Filtering
- **Overview:** Determines the time horizon for threat scanning, fetches relevant malware and critical vulnerabilities directly from the GitHub Advisory Database, and filters out withdrawn or invalid records.
- **Nodes Involved:**
  - `Build Advisory Queries`
  - `Fetch Advisories`
  - `Index Advisories`
  - `Any Advisories?`
- **Node Details:**
  - **Build Advisory Queries** (`n8n-nodes-base.code`)
    - *Type and Technical Role:* JavaScript code evaluation node.
    - *Configuration:* Calculates ISO date thresholds using `lookback_days` from the `Settings` node and constructs query URLs for malware and critical advisories.
    - *Input/Output:* Input from `Load Incidents`; outputs query objects to `Fetch Advisories`.
  - **Fetch Advisories** (`n8n-nodes-base.httpRequest`)
    - *Type and Technical Role:* HTTP request node utilizing pagination.
    - *Configuration:* Uses GitHub API (`https://api.github.com/advisories`) with predefined GitHub credentials, checking pagination links (`rel="next"` up to 30 pages).
    - *Input/Output:* Input from `Build Advisory Queries`; outputs raw advisory JSON items to `Index Advisories`.
    - *Edge Cases:* Rate limiting by GitHub API (`429 Too Many Requests`), invalid tokens, or network timeouts.
  - **Index Advisories** (`n8n-nodes-base.code`)
    - *Type and Technical Role:* JavaScript code evaluation node.
    - *Configuration:* Normalizes advisory structures into a map, discarding withdrawn or invalid items and extracting version range rules.
    - *Input/Output:* Input from `Fetch Advisories`; outputs indexed summary lists to `Any Advisories?`.
  - **Any Advisories?** (`n8n-nodes-base.if`)
    - *Type and Technical Role:* Conditional branching node.
    - *Configuration:* Evaluates if `count > 0`.
    - *Input/Output:* Input from `Index Advisories`; outputs to `List Repositories` if advisories exist.

#### 2.3 Repository Enumeration & Dependency Matching
- **Overview:** Enumerates repositories associated with the configured user or organization, downloads their dependency graphs (SBOMs), and cross-checks installed versions against known vulnerability version ranges.
- **Nodes Involved:**
  - `List Repositories`
  - `Filter Repositories`
  - `Limit Repositories`
  - `Get Dependency Graph`
  - `Match Dependencies`
- **Node Details:**
  - **List Repositories** (`n8n-nodes-base.httpRequest`)
    - *Type and Technical Role:* HTTP request node utilizing pagination.
    - *Configuration:* Conditionally queries user repositories or organization repositories based on `owner_type` settings with GitHub API credentials.
    - *Input/Output:* Input from `Any Advisories?`; outputs repository lists to `Filter Repositories`.
    - *Edge Cases:* Pagination limits, missing permissions on enterprise or organizational boundaries.
  - **Filter Repositories** (`n8n-nodes-base.filter`)
    - *Type and Technical Role:* Data filtering node.
    - *Configuration:* Removes archived repositories, disabled items, forks (unless `include_forks` is true), and repositories failing `repo_name_filter` regex.
    - *Input/Output:* Input from `List Repositories`; outputs filtered list to `Limit Repositories`.
  - **Limit Repositories** (`n8n-nodes-base.limit`)
    - *Type and Technical Role:* Item limitation node.
    - *Configuration:* Restricts output items to `max_repositories` set in global settings.
    - *Input/Output:* Input from `Filter Repositories`; outputs limited items to `Get Dependency Graph`.
  - **Get Dependency Graph** (`n8n-nodes-base.httpRequest`)
    - *Type and Technical Role:* HTTP request node with batch processing configuration.
    - *Configuration:* Requests SBOM endpoints (`/repos/{owner}/{repo}/dependency-graph/sbom`) with a batch size of 5 and 500ms intervals. Sets error handling to `continueRegularOutput`.
    - *Input/Output:* Input from `Limit Repositories`; outputs SBOM responses to `Match Dependencies`.
    - *Edge Cases:* Repositories without dependency graph enabled return `404` or `422`, handled via regular output continuation.
  - **Match Dependencies** (`n8n-nodes-base.code`)
    - *Type and Technical Role:* JavaScript code evaluation node.
    - *Configuration:* Parses package URLs (`purl`), matches version segments using custom semantic version comparison logic against vulnerable ranges, cross-references against history loaded from the `supply_chain_incidents` table, and compiles affected groups.
    - *Input/Output:* Input from `Get Dependency Graph` (and reads from `Index Advisories`, `Limit Repositories`, `Load Incidents`); outputs grouped threat payloads to `AI Incident Responder`.

#### 2.4 AI Incident Analysis
- **Overview:** Passes vulnerability and impacted repository details to an LLM chain to generate structured operational guidance, technical summaries, and triage steps.
- **Nodes Involved:**
  - `AI Incident Responder`
  - `Gemini`
  - `Response Plan Parser`
- **Node Details:**
  - **AI Incident Responder** (`@n8n/n8n-nodes-langchain.chainLlm`)
    - *Type and Technical Role:* Advanced AI Chain node.
    - *Configuration:* Configured with specific system prompt rules regarding malware triage, output constraints, and retries.
    - *Input/Output:* Input from `Match Dependencies` and `Gemini` (Language Model); outputs structured text payloads to `Format Alert`.
    - *Edge Cases:* API errors, JSON schema violations, or rate limits on the Gemini endpoint.
  - **Gemini** (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)
    - *Type and Technical Role:* Chat model integration node.
    - *Configuration:* Uses model `models/gemini-3.8-flash`.
    - *Input/Output:* Connects directly to `AI Incident Responder`.
  - **Response Plan Parser** (`@n8n/n8n-nodes-langchain.outputParserStructured`)
    - *Type and Technical Role:* Structured output parser node.
    - *Configuration:* Enforces strict JSON schema validation containing `headline`, `what_happened`, `immediate_steps`, `how_to_fix`, and `check_for_compromise`.
    - *Input/Output:* Connects to `AI Incident Responder`.

#### 2.5 Alerting, Logging, & Issue Tracking
- **Overview:** Transforms AI and matching outputs into formatted Slack messages, persists incident states back to the Data Table, and optionally provisions remediation issues on private GitHub repositories.
- **Nodes Involved:**
  - `Format Alert`
  - `Slack Supply-Chain Alert`
  - `Prepare Incident Rows`
  - `Save Incidents`
  - `Plan GitHub Issues`
  - `Create GitHub Issue`
- **Node Details:**
  - **Format Alert** (`n8n-nodes-base.code`)
    - *Type and Technical Role:* JavaScript code evaluation node.
    - *Configuration:* Formats an HTML-escaped Slack block message string mixing threat characteristics, AI remediation, and affected packages.
    - *Input/Output:* Input from `AI Incident Responder`; outputs formatted alert payloads to `Slack Supply-Chain Alert`, `Prepare Incident Rows`, and `Plan GitHub Issues`.
  - **Slack Supply-Chain Alert** (`n8n-nodes-base.slack`)
    - *Type and Technical Role:* Messaging integration node.
    - *Configuration:* Posts messages to the `#security-alerts` channel using Slack credentials.
    - *Input/Output:* Input from `Format Alert`.
    - *Edge Cases:* Missing channel authorization or invalid token configuration.
  - **Prepare Incident Rows** (`n8n-nodes-base.code`)
    - *Type and Technical Role:* JavaScript code evaluation node.
    - *Configuration:* Flattens affected groups into individual rows matching the Data Table schema.
    - *Input/Output:* Input from `Format Alert`; outputs row objects to `Save Incidents`.
  - **Save Incidents** (`n8n-nodes-base.dataTable`)
    - *Type and Technical Role:* Data Table persistence node.
    - *Configuration:* Performs an upsert operation on the `supply_chain_incidents` table matching on the unique identifier `key`.
    - *Input/Output:* Input from `Prepare Incident Rows`.
  - **Plan GitHub Issues** (`n8n-nodes-base.code`)
    - *Type and Technical Role:* JavaScript code evaluation node.
    - *Configuration:* Drafts Markdown-safe issue payloads for private repositories when `create_github_issues` is enabled.
    - *Input/Output:* Input from `Format Alert`; outputs issue definitions to `Create GitHub Issue`.
  - **Create GitHub Issue** (`n8n-nodes-base.github`)
    - *Type and Technical Role:* GitHub integration node.
    - *Configuration:* Creates issues under specific repository targets using GitHub API credentials.
    - *Input/Output:* Input from `Plan GitHub Issues`.
    - *Edge Cases:* Missing `Issues: write` permission on token scopes.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Main Note | n8n-nodes-base.stickyNote | Workflow overview and instructions | None | None | Detect compromised open-source packages in GitHub repositories with Gemini and Slack... |
| Step 1 Note | n8n-nodes-base.stickyNote | Explains settings and incident log table | None | None | 1. Settings & incident log - Edit Settings once... |
| Step 2 Note | n8n-nodes-base.stickyNote | Explains recent advisories pull logic | None | None | 2. Recent advisories - Malware and critical advisories from the last 7 days... |
| Step 3 Note | n8n-nodes-base.stickyNote | Explains dependency graphs & matching logic | None | None | 3. Dependency graphs & matching - Downloads each repository's SBOM... |
| Step 4 Note | n8n-nodes-base.stickyNote | Explains AI response plan generation | None | None | 4. AI response plan - One Gemini call per new advisory... |
| Step 5 Note | n8n-nodes-base.stickyNote | Explains alerts, logs, and issues | None | None | 5. Alert, log & issues - Slack alert, incident log... |
| Every Hour | n8n-nodes-base.scheduleTrigger | Triggers the workflow hourly | None | Settings | |
| Settings | n8n-nodes-base.set | Defines global workflow configuration | Every Hour | Create Incident Table | |
| Create Incident Table | n8n-nodes-base.dataTable | Creates the incidents data table if missing | Settings | Load Incidents | |
| Load Incidents | n8n-nodes-base.dataTable | Loads known past incidents for de-duplication | Create Incident Table | Build Advisory Queries | |
| Build Advisory Queries | n8n-nodes-base.code | Constructs GitHub Advisory API query URLs | Load Incidents | Fetch Advisories | |
| Fetch Advisories | n8n-nodes-base.httpRequest | Fetches advisories from GitHub API | Build Advisory Queries | Index Advisories | |
| Index Advisories | n8n-nodes-base.code | Parses and indexes fetched advisories | Fetch Advisories | Any Advisories? | |
| Any Advisories? | n8n-nodes-base.if | Checks if any advisories exist | Index Advisories | List Repositories | |
| List Repositories | n8n-nodes-base.httpRequest | Retrieves repositories for target user or org | Any Advisories? | Filter Repositories | |
| Filter Repositories | n8n-nodes-base.filter | Filters out forks, archives, and ignored repos | List Repositories | Limit Repositories | |
| Limit Repositories | n8n-nodes-base.limit | Limits the maximum repositories scanned | Filter Repositories | Get Dependency Graph | |
| Get Dependency Graph | n8n-nodes-base.httpRequest | Downloads SBOMs for repositories | Limit Repositories | Match Dependencies | |
| Match Dependencies | n8n-nodes-base.code | Matches dependencies with advisory ranges | Get Dependency Graph, Load Incidents, Limit Repositories, Index Advisories | AI Incident Responder | |
| AI Incident Responder | @n8n/n8n-nodes-langchain.chainLlm | Generates AI incident response plans | Match Dependencies, Gemini, Response Plan Parser | Format Alert | |
| Gemini | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides the Gemini chat model | None | AI Incident Responder | |
| Response Plan Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured output format on AI | None | AI Incident Responder | |
| Format Alert | n8n-nodes-base.code | Formats Slack messages and tables | AI Incident Responder, Match Dependencies | Slack Supply-Chain Alert, Prepare Incident Rows, Plan GitHub Issues | |
| Slack Supply-Chain Alert | n8n-nodes-base.slack | Sends security alerts to Slack channel | Format Alert | None | |
| Prepare Incident Rows | n8n-nodes-base.code | Prepares rows for Data Table logging | Format Alert | Save Incidents | |
| Save Incidents | n8n-nodes-base.dataTable | Saves incidents to prevent duplicate alerts | Prepare Incident Rows | None | |
| Plan GitHub Issues | n8n-nodes-base.code | Drafts issues for private repositories | Format Alert | Create GitHub Issue | |
| Create GitHub Issue | n8n-nodes-base.github | Creates GitHub issues for private repos | Plan GitHub Issues | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**
   - Create a **Schedule Trigger** node named `Every Hour`.
   - Set the interval rule to trigger every `1` hour.
2. **Create Configuration Node:**
   - Add a **Set** node named `Settings`.
   - Configure string/boolean assignments: `github_owner` (string, e.g., `"your-org"`), `owner_type` (string, `"org"` or `"user"`), `lookback_days` (number, `7`), `include_critical_vulnerabilities` (boolean, `true`), `repo_name_filter` (string, `""`), `include_forks` (boolean, `false`), `max_repositories` (number, `100`), and `create_github_issues` (boolean, `false`).
   - Connect `Every Hour` to `Settings`.
3. **Setup Data Tables (Incident Log):**
   - Add a **Data Table** node named `Create Incident Table`.
   - Set resource to `Table`, operation to `Create`, table name to `supply_chain_incidents`, and define columns (`key`, `ghsa_id`, `advisory_type`, `severity`, `repo`, `ecosystem`, `package`, `version`, `fixed_version`, `confirmed`, `detected_at`, `status`, `summary`). Enable `Execute Once`.
   - Connect `Settings` to `Create Incident Table`.
   - Add another **Data Table** node named `Load Incidents`. Set resource to `Row`, operation to `Get`, return all rows, and select `supply_chain_incidents`. Enable `Execute Once` and `Always Output Data`.
   - Connect `Create Incident Table` to `Load Incidents`.
4. **Build Advisory Retrieval Branch:**
   - Add a **Code** node named `Build Advisory Queries`. Insert code that creates query URLs using `lookback_days` and severity settings. Enable `Execute Once`. Connect `Load Incidents` to it.
   - Add an **HTTP Request** node named `Fetch Advisories`. Set URL expression `={{ $json.url }}`, method `GET`, enable pagination (response contains next link), set headers (`Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`), and configure predefined `githubApi` credentials. Connect `Build Advisory Queries` to it.
   - Add a **Code** node named `Index Advisories` to filter out withdrawn advisories and map vulnerability rules. Connect `Fetch Advisories` to it.
   - Add an **If** node named `Any Advisories?` to evaluate if `{{ $json.count > 0 }}`. Connect `Index Advisories` to it.
5. **Enumerate Repositories and Fetch Dependency Graphs:**
   - Add an **HTTP Request** node named `List Repositories`. Set URL conditionally based on `owner_type` (`user` vs `org`), enable GitHub API credentials, and pagination. Connect `Any Advisories?` (True branch) to it.
   - Add a **Filter** node named `Filter Repositories`. Filter out archived, disabled, and unwanted forks/names. Connect `List Repositories` to it.
   - Add a **Limit** node named `Limit Repositories` with max items set to `={{ $('Settings').first().json.max_repositories }}`. Connect `Filter Repositories` to it.
   - Add an **HTTP Request** node named `Get Dependency Graph`. Set URL to `=https://api.github.com/repos/{{ $json.full_name }}/dependency-graph/sbom`, enable GitHub API credentials, configure batching (batch size 5, interval 500ms), and set error handling to continue regular output. Connect `Limit Repositories` to it.
6. **Execute Dependency Matching:**
   - Add a **Code** node named `Match Dependencies`. Add matching logic that parses `purl`, compares semantic versions against vulnerable ranges, and filters out already known incidents. Connect `Get Dependency Graph` to it.
7. **Configure AI Incident Responder (LangChain):**
   - Add a **Google Gemini Chat Model** node named `Gemini`. Select model `models/gemini-3.8-flash` and configure Google Gemini API credentials.
   - Add a **Structured Output Parser** node named `Response Plan Parser` with a JSON schema defining `headline`, `what_happened`, `immediate_steps`, `how_to_fix`, and `check_for_compromise`.
   - Add an **Advanced AI Chain** node (`AI Incident Responder`). Set prompt text targeting security supply-chain triage rules. Connect `Gemini` to the AI Language Model input, `Response Plan Parser` to the AI Output Parser input, and `Match Dependencies` to the main input.
8. **Configure Output Handlers:**
   - Add a **Code** node named `Format Alert` to format the Slack message payload and combine AI response data. Connect `AI Incident Responder` to it.
   - Add a **Slack** node named `Slack Supply-Chain Alert`. Set message text to `={{ $json.slack_text }}`, select channel `#security-alerts`, and connect Slack credentials. Connect `Format Alert` to it.
   - Add a **Code** node named `Prepare Incident Rows` to flatten incident groups. Connect `Format Alert` to it.
   - Add a **Data Table** node named `Save Incidents`. Set resource to `Row`, operation to `Upsert`, table name `supply_chain_incidents`, matching columns on `key`. Connect `Prepare Incident Rows` to it.
   - Add a **Code** node named `Plan GitHub Issues` to draft issues for private repositories conditionally. Connect `Format Alert` to it.
   - Add a **GitHub** node named `Create GitHub Issue`. Set resource to `Issue`, operation to `Create`, repository, title, and body from expressions, using GitHub API credentials. Connect `Plan GitHub Issues` to it.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Gemini API keys can be obtained freely from AI Studio. | [Google AI Studio](https://aistudio.google.com) |
| GitHub personal access tokens require appropriate scopes (`repo` or fine-grained `Contents: read`, `Metadata: read`, and `Issues: write`). | [GitHub Developer Documentation](https://docs.github.com/en/authentication/keeping-your-account-and-secure/managing-your-personal-access-tokens) |