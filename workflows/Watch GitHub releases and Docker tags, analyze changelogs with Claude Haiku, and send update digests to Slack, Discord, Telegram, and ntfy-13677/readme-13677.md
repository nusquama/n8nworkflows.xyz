Watch GitHub releases and Docker tags, analyze changelogs with Claude Haiku, and send update digests to Slack, Discord, Telegram, and ntfy

https://n8nworkflows.xyz/workflows/watch-github-releases-and-docker-tags--analyze-changelogs-with-claude-haiku--and-send-update-digests-to-slack--discord--telegram--and-ntfy-13677


# Watch GitHub releases and Docker tags, analyze changelogs with Claude Haiku, and send update digests to Slack, Discord, Telegram, and ntfy

This document provides a technical analysis of the **GitHub Release Watcher** n8n workflow. This system automates the monitoring of software releases, utilizes AI for intelligent changelog summarization, and dispatches multi-channel alerts based on urgency.

---

### 1. Workflow Overview

The workflow is designed to monitor a list of software repositories (GitHub) and container images (Docker Hub/GHCR). It determines if a new version has been released by comparing current tags against a persistent internal state. If a new release is detected, it uses the Claude Haiku LLM to analyze the changelog for breaking changes, security vulnerabilities, and general impact, then sends a formatted report to various messaging platforms.

#### Logical Blocks:
*   **1.1 Configuration & Watchlist Generation:** Initializes global settings and aggregates repositories from manual lists and optional `docker-compose.yml` auto-detection.
*   **1.2 Release Detection:** Queries GitHub and Docker APIs to fetch the latest stable tags while filtering out pre-releases.
*   **1.3 Version Comparison:** Compares fetched tags against `staticData`. Only new versions proceed to the AI pipeline.
*   **1.4 AI Analysis:** Claude Haiku processes the raw changelogs into structured JSON containing summaries, urgency ratings, and security flags.
*   **1.5 Routing & Persistence:** Determines if an alert should be sent immediately (critical) or batched. Logs the release into a PostgreSQL database.
*   **1.6 Delivery:** Formats and dispatches messages to Discord, Telegram, Slack, and ntfy.

---

### 2. Block-by-Block Analysis

#### 2.1 Configuration & Watchlist
This block sets the environment and defines what to monitor.
*   **Nodes:** `Schedule Trigger`, `Configure Watcher`, `Build Repo Watchlist`, `Auto-Detect Enabled?`, `Has Compose URL?`, `Fetch Compose File`, `Parse and Map Compose`, `Merge Manual + Auto`, `Deduplicate Watchlist`.
*   **Configuration:** The `Configure Watcher` node acts as a central hub for API tokens, webhook URLs, and operational modes (e.g., `test_mode`).
*   **Logic:** It supports "Auto-Detection" by fetching a `docker-compose.yml` via URL or text, parsing the images via Regex, and mapping them to known GitHub repositories.
*   **Failure Modes:** Invalid Compose YAML, failed HTTP fetch for the remote compose file, or unauthorized GitHub tokens.

#### 2.2 Release Detection & Comparison
Fetches data and decides if the workflow should continue.
*   **Nodes:** `Source Router`, `Fetch Latest Release`, `Fetch Registry Tags`, `Extract Release Data`, `Extract Registry Data`, `Merge All Sources`, `Compare Versions`, `Has Updates?`.
*   **Logic:** GitHub uses the `/releases/latest` endpoint. Docker/GHCR fetch tags and filter for "stable" patterns (excluding `-rc`, `-beta`, etc.). The `Compare Versions` node uses `n8n` workflow static data (`$getWorkflowStaticData('global')`) to store and compare version strings.
*   **Edge Cases:** Rate limiting on GitHub (handled by an error check in `Extract Release Data`), repositories with no releases, or non-semantic versioning.

#### 2.3 AI Processing
Analyzes the content of the release.
*   **Nodes:** `Prep Changelog for AI`, `Analyze Release`, `Claude Haiku`, `Parse AI Response`.
*   **Logic:** The changelog is truncated to ~1500 characters to manage token costs while preserving "Breaking Changes" sections. The `Analyze Release` node (ChainLlm) instructs Claude Haiku to return a specific JSON schema.
*   **Variables:** `{{ $json.changelogTruncated }}`, `{{ $json.label }}`.
*   **Failure Modes:** LLM hallucinations (mitigated by JSON parsing logic in `Parse AI Response`), Anthropic API timeouts, or credit exhaustion.

#### 2.4 Routing, Storage & Delivery
Formats and sends the notifications.
*   **Nodes:** `Apply Alert Rules`, `Urgency Router`, `Prep DB Record`, `Save to DB`, `Format Instant Alert`, `Format Digest`, `Channel Router`, `Send Discord`, `Send Telegram`, `Send Slack`, `Send ntfy`, `Update Stored Versions`.
*   **Logic:**
    *   **Urgency:** "Critical" or "Security" updates bypass the daily digest and are sent immediately via `Format Instant Alert`.
    *   **Digest:** Standard updates are aggregated via `Format Digest`.
    *   **Database:** Generates an SQL `INSERT` statement for PostgreSQL to keep a historical log.
    *   **Version Update:** The final node updates the `staticData` so the next run ignores these versions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule | Workflow entry point | - | Configure Watcher | Runs daily at 8 AM |
| Configure Watcher | Set | Global settings | Schedule Trigger | Watchlist / Auto-Detect | Open to set URLs and tokens |
| Build Repo Watchlist | Code | Manual repo list | Configure Watcher | Merge Manual + Auto | Add/remove repos to monitor |
| Auto-Detect Enabled? | If | Logic switch | Configure Watcher | Has Compose URL? / Skip | Enable/disable compose scanning |
| Parse and Map Compose | Code | YAML parser | Fetch Compose / Has URL? | Merge Manual + Auto | Extracts images from docker-compose |
| Source Router | Switch | API routing | Deduplicate Watchlist | GitHub Fetch / Reg. Fetch | Splits GitHub vs Docker Hub/GHCR |
| Fetch Latest Release | HTTP Request | Data acquisition | Source Router | Extract Release Data | GitHub Releases API |
| Fetch Registry Tags | HTTP Request | Data acquisition | Source Router | Extract Registry Data | Container tag APIs |
| Compare Versions | Code | Change detection | Merge All Sources | Has Updates? | Checks tags against static data |
| Claude Haiku | Anthropic Model | AI Brain | Analyze Release | Analyze Release | Uses Claude Haiku 4.5/2025 |
| Analyze Release | ChainLlm | AI Prompting | Prep Changelog | Parse AI Response | Summarizes and rates urgency |
| Save to DB | Postgres | Data persistence | Prep DB Record | - | Saves release_history table |
| Urgency Router | Switch | Alert prioritization | Apply Alert Rules | Instant / Digest | Critical/Security vs Daily Batch |
| Format Digest | Code | Multi-report builder | Urgency Router | Channel Router | Aggregates all found updates |
| Channel Router | Switch | Multi-channel dispatch | Format nodes | Send nodes / Update Stored | Routes to enabled channels |
| Update Stored Versions | Code | State persistence | Channel Router | - | Updates workflow static data |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:** Create a new n8n workflow and configure a **Schedule Trigger** (e.g., `0 8 * * *`).
2.  **Configuration Node:** Add a **Set** node (`Configure Watcher`). Define strings for `github_token`, `discord_webhook_url`, `telegram_chat_id`, `ntfy_topic`, and a boolean `test_mode`.
3.  **Watchlist Construction:**
    *   Create a **Code** node to return an array of objects containing `owner`, `repo`, `label`, and `source`.
    *   (Optional) Create an **HTTP Request** node to fetch a `docker-compose.yml` and a **Code** node to regex-match images.
    *   Use a **Merge** and **Code** node to deduplicate the manual and auto-detected lists.
4.  **Fetch Logic:**
    *   Use a **Switch** node to route by `source`.
    *   For GitHub: **HTTP Request** to `https://api.github.com/repos/{{owner}}/{{repo}}/releases/latest`.
    *   For Docker: **HTTP Request** to Docker Hub/GHCR tag endpoints.
5.  **Comparison Logic:**
    *   Use a **Merge** node to collect all results.
    *   Use a **Code** node accessing `$getWorkflowStaticData('global')`. If `test_mode` is false, only return items where the tag differs from the stored value.
6.  **AI Integration:**
    *   Add a **Basic LLM Chain** node.
    *   Connect an **Anthropic Chat Model** node (configured for `claude-haiku`).
    *   Set the prompt to request a JSON response including `summary`, `breaking`, `urgency`, and `security`.
7.  **Routing & Dispatch:**
    *   Use a **Code** node to parse the LLM JSON.
    *   Use a **Switch** node to split `critical` updates from others.
    *   Create **Code** nodes for "Digest" (aggregating items) and "Instant" formatting.
    *   Use a **Switch** node with "All matching outputs" to trigger multiple **HTTP Request** (Discord, ntfy), **Slack**, and **Telegram** nodes based on enabled settings.
8.  **Finalize:** Add a final **Code** node to update the `staticData` values with the new tags.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| GitHub Token Setup Guide | [https://www.nxsi.io/guides/github-personal-access-token](https://www.nxsi.io/guides/github-personal-access-token) |
| Anthropic API Key Setup | [https://www.nxsi.io/guides/anthropic-api-key](https://www.nxsi.io/guides/anthropic-api-key) |
| Discord Webhook Guide | [https://www.nxsi.io/guides/discord-webhook](https://www.nxsi.io/guides/discord-webhook) |
| Telegram Bot Setup | [https://www.nxsi.io/guides/telegram-bot](https://www.nxsi.io/guides/telegram-bot) |
| ntfy Push Notifications | [https://ntfy.sh](https://ntfy.sh) |
| Workflow Credits | Created by nxsi.io |
| Operating Cost | Estimated ~$0.01-$0.03 per run for 10-15 repositories. |