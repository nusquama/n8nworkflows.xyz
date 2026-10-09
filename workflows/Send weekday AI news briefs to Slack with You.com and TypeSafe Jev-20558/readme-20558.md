Send weekday AI news briefs to Slack with You.com and TypeSafe Jev

https://n8nworkflows.xyz/workflows/send-weekday-ai-news-briefs-to-slack-with-you-com-and-typesafe-jev-20558


# Send weekday AI news briefs to Slack with You.com and TypeSafe Jev

### 1. Workflow Overview

This workflow automates the collection, evaluation, and summarization of artificial intelligence news from You.com, leveraging TypeSafe’s Jev model to filter for material developments, and finally delivering concise, cited morning briefs to a Slack channel every weekday.

The functional logic is structured into three main blocks:
- **1.1 Find the Latest News:** Initializes workflow execution parameters, iterates over target topics, queries You.com for recent articles, normalizes individual article records, and prevents duplicate processing across executions.
- **1.2 Judge What Matters:** Evaluates each article against custom team criteria using TypeSafe’s Jev API and filters out noise, retaining only significant developments.
- **1.3 Brief the Team in Slack:** Aggregates filtered articles by topic, uses You.com Research to synthesize brief summaries with citations, formats the payload for messaging, and publishes the updates to Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Find the Latest News
- **Overview:** This block establishes the operational schedule, defines tracking configurations, retrieves recent web articles per topic via You.com, and ensures that previously processed URLs are systematically ignored.
- **Nodes Involved:** 
  - `Every weekday at 8am`
  - `Config`
  - `One item per topic`
  - `You.com: search the latest news`
  - `One item per article`
  - `Prepare article`
  - `Skip articles already seen`
- **Node Details:**
  - **`Every weekday at 8am`**
    - *Type & Role:* Schedule Trigger (n8n-nodes-base.scheduleTrigger). Initiates workflow execution periodically.
    - *Configuration:* Interval-based schedule configured to run Monday through Friday at 08:00 based on the local instance timezone.
    - *Connections:* Input: None (Trigger); Output: `Config`.
    - *Edge Cases:* Timezone misconfigurations on the host server can shift execution times.
  - **`Config`**
    - *Type & Role:* Edit Fields / Set Node (n8n-nodes-base.set). Sets core constants and targeting criteria for the execution.
    - *Configuration:* Manual assignment of static array and string fields: `topics` (array of focus areas), `team` (description string), `caresAbout` (array of prioritization guidelines), and `materialityThreshold` (numeric threshold value).
    - *Key Expressions:* JSON arrays and strings defining the scope of the intelligence gathering.
    - *Connections:* Input: `Every weekday at 8am`; Output: `One item per topic`.
  - **`One item per topic`**
    - *Type & Role:* Item Lists / Split Out (n8n-nodes-base.splitOut). Flattens the array of topics into individual discrete items.
    - *Configuration:* Splits out the `topics` field, discarding unrelated fields.
    - *Connections:* Input: `Config`; Output: `You.com: search the latest news`.
  - **`You.com: search the latest news`**
    - *Type & Role:* Custom Node / You.com Search (@youdotcom-oss/n8n-nodes-youdotcom.youDotCom). Fetches live web search results from You.com.
    - *Configuration:* Operation set to `search`. Search options configured with a count of `15` results and freshness set to `day`.
    - *Key Expressions:* Query parameter evaluated as `={{ $json.topic + ' news' }}`.
    - *Connections:* Input: `One item per topic`; Output: `One item per article`.
    - *Edge Cases:* API authentication failures, rate limits, or empty result arrays if no news is indexed.
  - **`One item per article`**
    - *Type & Role:* Item Lists / Split Out (n8n-nodes-base.splitOut). Breaks down search results into individual article items.
    - *Configuration:* Splits the array located at `results.web`, designating the destination field as `article`.
    - *Connections:* Input: `You.com: search the latest news`; Output: `Prepare article`.
  - **`Prepare article`**
    - *Type & Role:* Edit Fields / Set Node (n8n-nodes-base.set). Normalizes article structures for subsequent processing steps.
    - *Configuration:* Manual field assignments mapping topic, title, description, URL, sanitized URL key, and publication age.
    - *Key Expressions:* Extracts references like `={{ $('One item per topic').item.json.topic }}` and handles optional properties using nullish coalescing (e.g., `={{ $json.article.description ?? '' }}`).
    - *Connections:* Input: `One item per article`; Output: `Skip articles already seen`.
  - **`Skip articles already seen`**
    - *Type & Role:* Remove Duplicates (n8n-nodes-base.removeDuplicates). Filters out articles processed in past executions.
    - *Configuration:* Operation set to `removeItemsSeenInPreviousExecutions`.
    - *Key Expressions:* Deduplication value evaluated against `={{ $json.urlKey }}`.
    - *Edge Cases:* Persistent execution history must be cleared prior to production deployment if testing resulted in false deduplication locks.

#### 2.2 Judge What Matters
- **Overview:** Evaluates individual articles using TypeSafe’s Jev API to score development veracity, topical relevance, and team materiality, filtering out noise.
- **Nodes Involved:**
  - `Jev: judge each article`
  - `Keep only material news`
  - `Material article details`
- **Node Details:**
  - **`Jev: judge each article`**
    - *Type & Role:* HTTP Request (n8n-nodes-base.httpRequest). Interfaces with the external Jev AI evaluation endpoint.
    - *Configuration:* POST request to `https://api.typesafe.ai/v1/systemone` with a 30-second timeout. Uses generic HTTP Header Authentication (`Authorization: Bearer <API_KEY>`). Error handling configured to continue regular output upon failure.
    - *Key Expressions:* Constructed JSON request body dynamically injects the model name, watch states, and specific multi-question classification instructions (reports new development, about topic, materiality score).
    - *Connections:* Input: `Skip articles already seen`; Output: `Keep only material news`.
    - *Edge Cases:* API timeouts, invalid bearer tokens, or structural changes to the third-party evaluation schema.
  - **`Keep only material news`**
    - *Type & Role:* Filter (n8n-nodes-base.filter). Retains only articles meeting predefined qualification thresholds.
    - *Configuration:* Complex condition check requiring:
      - `reports_new_development` noul score $\ge 0.8$
      - `about_topic` noul score $\ge 0.8$
      - `materiality` score $\ge$ the `materialityThreshold` defined in the Config node.
    - *Connections:* Input: `Jev: judge each article`; Output: `Material article details`.
  - **`Material article details`**
    - *Type & Role:* Edit Fields / Set Node (n8n-nodes-base.set). Restructures verified items to carry forward essential identifiers.
    - *Configuration:* Manual assignments preserving `topic`, `title`, and `url`.
    - *Key Expressions:* References previous node scopes using `$('Prepare article').item.json...`.
    - *Connections:* Input: `Keep only material news`; Output: `Group articles by topic`.

#### 2.3 Brief the Team in Slack
- **Overview:** Groups filtered articles by subject, generates a synthesized research summary with citations via You.com, formats the output for Markdown compatibility, and publishes the brief to Slack.
- **Nodes Involved:**
  - `Group articles by topic`
  - `You.com: research what changed`
  - `Format Slack brief`
  - `Post brief to Slack`
- **Node Details:**
  - **`Group articles by topic`**
    - *Type & Role:* Summarize (n8n-nodes-base.summarize). Groups aggregated articles by their category topic.
    - *Configuration:* Fields to split by set to `topic`. Fields to summarize configured to append `title` and `url` arrays.
    - *Connections:* Input: `Material article details`; Output: `You.com: research what changed`.
  - **`You.com: research what changed`**
    - *Type & Role:* Custom Node / You.com Research (@youdotcom-oss/n8n-nodes-youdotcom.youDotCom). Synthesizes a structured summary from grouped news references.
    - *Configuration:* Operation set to `research` with `researchEffort` set to `lite`.
    - *Key Expressions:* Dynamically constructs research prompts incorporating mapped titles and URLs from the preceding aggregation step.
    - *Connections:* Input: `Group articles by topic`; Output: `Format Slack brief`.
    - *Edge Cases:* Failures in research synthesis if appended URL lists exceed token bounds or if the API returns malformed source citations.
  - **`Format Slack brief`**
    - *Type & Role:* Edit Fields / Set Node (n8n-nodes-base.set). Transforms raw research text and citations into clean Slack-compatible Markdown.
    - *Configuration:* Manual string transformation mapping `text`.
    - *Key Expressions:* Executes inline JavaScript routines to sanitize Markdown heading syntax, bold tags, and translate citation indices (e.g., `[[1]]`) into clickable Slack link tags (`<url|[1]>`).
    - *Connections:* Input: `You.com: research what changed`; Output: `Post brief to Slack`.
  - **`Post brief to Slack`**
    - *Type & Role:* Slack Node (n8n-nodes-base.slack). Publishes the formatted markdown payload directly to the communication workspace.
    - *Configuration:* Select mode set to `channel`. Target channel ID configured by name (`ai-news`). Link unfurling disabled (`unfurl_links: false`, `unfurl_media: false`).
    - *Key Expressions:* Message text evaluated as `={{ $json.text }}`.
    - *Connections:* Input: `Format Slack brief`; Output: None (Terminal Node).
    - *Edge Cases:* Missing Slack credentials, insufficient bot permissions to post to target channels, or channel name misconfigurations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Every weekday at 8am` | `n8n-nodes-base.scheduleTrigger` | Triggers workflow execution at 8 AM on weekdays. | None | `Config` | ## 1. Find the latest news<br><br>Search each topic's past day, one article per item, and skip anything seen in earlier runs. |
| `Config` | `n8n-nodes-base.set` | Defines core configuration variables and team context. | `Every weekday at 8am` | `One item per topic` | ## 1. Find the latest news<br><br>Search each topic's past day, one article per item, and skip anything seen in earlier runs. |
| `One item per topic` | `n8n-nodes-base.splitOut` | Iterates through configured topic array items. | `Config` | `You.com: search the latest news` | ## 1. Find the latest news<br><br>Search each topic's past day, one article per item, and skip anything seen in earlier runs. |
| `You.com: search the latest news` | `@youdotcom-oss/n8n-nodes-youdotcom.youDotCom` | Queries You.com for recent news articles by topic. | `One item per topic` | `One item per article` | ## 1. Find the latest news<br><br>Search each topic's past day, one article per item, and skip anything seen in earlier runs. |
| `One item per article` | `n8n-nodes-base.splitOut` | Extracts individual articles from web search results. | `You.com: search the latest news` | `Prepare article` | ## 1. Find the latest news<br><br>Search each topic's past day, one article per item, and skip anything seen in earlier runs. |
| `Prepare article` | `n8n-nodes-base.set` | Normalizes metadata fields for individual articles. | `One item per article` | `Skip articles already seen` | ## 1. Find the latest news<br><br>Search each topic's past day, one article per item, and skip anything seen in earlier runs. |
| `Skip articles already seen` | `n8n-nodes-base.removeDuplicates` | Prevents duplicate processing of previously seen URLs. | `Prepare article` | `Jev: judge each article` | ## 1. Find the latest news<br><br>Search each topic's past day, one article per item, and skip anything seen in earlier runs.<br><br>**Test runs mark articles as seen.** Before your first real run, set this node to *Clear Deduplication History*, execute it once, then switch it back. |
| `Jev: judge each article` | `n8n-nodes-base.httpRequest` | Sends article attributes to TypeSafe Jev API for scoring. | `Skip articles already seen` | `Keep only material news` | ## 2. Judge what matters<br><br>Jev scores each article. Only real, material developments pass. |
| `Keep only material news` | `n8n-nodes-base.filter` | Filters out non-material or irrelevant news items. | `Jev: judge each article` | `Material article details` | ## 2. Judge what matters<br><br>Jev scores each article. Only real, material developments pass. |
| `Material article details` | `n8n-nodes-base.set` | Formats filtered article details for grouping. | `Keep only material news` | `Group articles by topic` | ## 2. Judge what matters<br><br>Jev scores each article. Only real, material developments pass. |
| `Group articles by topic` | `n8n-nodes-base.summarize` | Aggregates material articles and links by subject category. | `Material article details` | `You.com: research what changed` | ## 3. Brief the team in Slack<br><br>Group by topic, then You.com Research writes one short cited brief per topic. |
| `You.com: research what changed` | `@youdotcom-oss/n8n-nodes-youdotcom.youDotCom` | Synthesizes a cited research brief per topic. | `Group articles by topic` | `Format Slack brief` | ## 3. Brief the team in Slack<br><br>Group by topic, then You.com Research writes one short cited brief per topic. |
| `Format Slack brief` | `n8n-nodes-base.set` | Converts synthesized briefs into Slack-compatible Markdown. | `You.com: research what changed` | `Post brief to Slack` | ## 3. Brief the team in Slack<br><br>Group by topic, then You.com Research writes one short cited brief per topic. |
| `Post brief to Slack` | `n8n-nodes-base.slack` | Publishes the final morning brief to a Slack channel. | `Format Slack brief` | None | ## 3. Brief the team in Slack<br><br>Group by topic, then You.com Research writes one short cited brief per topic. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node named `Every weekday at 8am` (Type Version 1.3).
   - Set interval rule configuration to trigger at hour `8` on days `[1, 2, 3, 4, 5]` (Monday–Friday).

2. **Setup Configuration:**
   - Add an **Edit Fields (Set)** node named `Config`.
   - Add assignments: `topics` (Array: `["AI model releases", "AI agent tooling", "AI regulation"]`), `team` (String: description of team), `caresAbout` (Array: tracking criteria), and `materialityThreshold` (Number: `1.3`).

3. **Split Topics:**
   - Add a **Split Out** node named `One item per topic`. Configure `fieldToSplitOut` to `topics` and set destination field to `topic`.

4. **Incorporate You.com Search:**
   - Add a **You.com** node named `You.com: search the latest news` (Install `@youdotcom-oss/n8n-nodes-youdotcom`).
   - Set operation to `search`, query expression to `={{ $json.topic + ' news' }}`, count to `15`, and freshness to `day`. Link You.com API credentials.

5. **Split Articles:**
   - Add a **Split Out** node named `One item per article`. Set `fieldToSplitOut` to `results.web` and destination field to `article`.

6. **Normalize Article Data:**
   - Add an **Edit Fields (Set)** node named `Prepare article`. Map fields: `topic`, `title`, `description`, `url`, `urlKey` (`={{ $json.article.url.split('?')[0].split('#')[0] }}`), and `publishedAt`.

7. **Deduplication Check:**
   - Add a **Remove Duplicates** node named `Skip articles already seen`. Set operation to `removeItemsSeenInPreviousExecutions` with dedupe value `={{ $json.urlKey }}`.

8. **AI Scoring via Jev:**
   - Add an **HTTP Request** node named `Jev: judge each article`.
   - Set Method to `POST`, URL to `https://api.typesafe.ai/v1/systemone`, timeout to `30000`, and set authentication to Generic Credential Type -> HTTP Header Auth (`Authorization: Bearer <YOUR_TYPESAFE_API_KEY>`).
   - Configure JSON body parameter with evaluation logic payloads evaluating development veracity, topical relevance, and team materiality. Set Error Handling to continue regular output.

9. **Filter Qualified Content:**
   - Add a **Filter** node named `Keep only material news`.
   - Set conditions to verify `reports_new_development` $\ge 0.8$, `about_topic` $\ge 0.8$, and `materiality` score $\ge$ `materialityThreshold`.

10. **Refine Qualified Articles:**
    - Add an **Edit Fields (Set)** node named `Material article details` to preserve `topic`, `title`, and `url`.

11. **Group by Topic:**
    - Add a **Summarize** node named `Group articles by topic`. Split by `topic`, appending `title` and `url` values.

12. **AI Research Synthesis:**
    - Add a **You.com** node named `You.com: research what changed`. Set operation to `research`, effort to `lite`, and map the prompt to summarize grouped developments with exact names and URLs. Link You.com credentials.

13. **Format for Messaging:**
    - Add an **Edit Fields (Set)** node named `Format Slack brief` to construct the final text payload mapping Markdown elements and translating reference arrays to Slack links.

14. **Publish to Slack:**
    - Add a **Slack** node named `Post brief to Slack`. Connect Slack OAuth2 credentials, select channel resource, set target channel name to `ai-news`, and disable link unfurling. Connect the message parameter to `={{ $json.text }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Verified You.com integration package | Available via n8n community nodes (@youdotcom-oss/n8n-nodes-youdotcom) |
| TypeSafe Jev model endpoints | Documentation and API keys managed at [TypeSafe AI](https://api.typesafe.ai) |
| Deduplication history maintenance | Clear deduplication history manually before initial production execution to prevent historical locks |