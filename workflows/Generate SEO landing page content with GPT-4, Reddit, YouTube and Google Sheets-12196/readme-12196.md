Generate SEO landing page content with GPT-4, Reddit, YouTube and Google Sheets

https://n8nworkflows.xyz/workflows/generate-seo-landing-page-content-with-gpt-4--reddit--youtube-and-google-sheets-12196


# Generate SEO landing page content with GPT-4, Reddit, YouTube and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *SEO Marketing Content Generator for AI and Engineering Education Services*  
**Title provided:** *Generate SEO landing page content with GPT-4, Reddit, YouTube and Google Sheets*

**Purpose:**  
Automates SEO landing page content generation by collecting multi-source audience and trend signals (Reddit, YouTube, NewsAPI), aggregating insights, and (optionally) using a LangChain AI Agent with GPT‑4o + SEO tools (SERP/Google Search, Wikipedia) to produce structured marketing content, convert it to HTML, and store results in Google Sheets.

**Primary use cases:**
- Rapid landing page drafting for education/AI/engineering services
- Trend & question mining from community + video + news sources
- Centralized content output tracking in Google Sheets

**Important execution note:**  
The node **“SEO Content Generator Agent” is disabled**. As shipped, the workflow will collect/aggregate research, but it will **not** generate content nor save to Sheets unless that node is enabled (and configured end-to-end).

### 1.1 Entry & Configuration
Manual trigger and a central configuration node define inputs (service type, audience, queries, etc.).

### 1.2 Multi-Source Research Collection
Parallel API pulls from Reddit, YouTube, and NewsAPI, then merges the results.

### 1.3 Insight Aggregation
A Code node summarizes merged research into structured “insights” (trends, keywords, sentiment, questions).

### 1.4 AI-Driven SEO Content Creation (currently disabled)
A LangChain Agent (GPT‑4o) uses the aggregated insights and can call tools (SERP API and Wikipedia) and enforces a structured JSON output schema.

### 1.5 Formatting & Persistence
Markdown is converted to HTML and appended/updated into a Google Sheet.

---

## 2. Block-by-Block Analysis

### Block 1 — Entry & Workflow Configuration
**Overview:** Initializes the workflow and defines all runtime parameters used by downstream research nodes (queries, subreddits, word count, etc.).

**Nodes involved:**
- When clicking 'Test workflow'
- Workflow Configuration

#### Node: When clicking 'Test workflow'
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — starts execution manually.
- **Configuration choices:** No parameters.
- **Inputs/outputs:**  
  - **Output →** Workflow Configuration
- **Edge cases:** None (only manual invocation).

#### Node: Workflow Configuration
- **Type / role:** Set (`n8n-nodes-base.set`) — defines constants/inputs for the run.
- **Key fields set (examples):**
  - `serviceType`: `"teaching"`
  - `targetAudience`: `"students"`
  - `industry`: `"AI and Engineering"`
  - `contentType`: `"landing page"`
  - `tone`: `"professional yet approachable"`
  - `wordCount`: `800`
  - `redditSubreddits`: `"artificial,MachineLearning,engineering,learnprogramming"`
  - `youtubeSearchQuery`: `"AI engineering education"`
  - `newsApiUrl`: `"https://newsapi.org/v2/everything"`
  - `newsApiQuery`: `"AI engineering education training"`
- **Expressions/variables:** These values are referenced later via `$json.<field>`.
- **Inputs/outputs:**
  - **Input ←** Manual Trigger
  - **Outputs →** Fetch Reddit Discussions, Fetch YouTube Videos, Fetch Industry News API (fan-out in parallel)
- **Edge cases / failure modes:**
  - Mis-typed or empty fields lead to weak/empty research results.
  - `redditSubreddits` is a comma-separated string; some Reddit node configs expect a single subreddit—behavior may vary depending on node implementation (verify in your n8n version).

---

### Block 2 — Multi-Source Research Collection
**Overview:** Pulls research data from three sources in parallel and merges them into a single combined stream for analysis.

**Nodes involved:**
- Fetch Reddit Discussions
- Fetch YouTube Videos
- Fetch Industry News API
- Combine Research Data

#### Node: Fetch Reddit Discussions
- **Type / role:** Reddit (`n8n-nodes-base.reddit`) — searches posts.
- **Configuration choices:**
  - Operation: `search`
  - `subreddit`: `={{ $json.redditSubreddits }}`
  - `keyword`: `={{ $json.industry }} {{ $json.serviceType }}`
  - `limit`: `20`
- **Inputs/outputs:**
  - **Input ←** Workflow Configuration
  - **Output →** Combine Research Data (input 0)
- **Credentials:** Requires Reddit API credentials configured in n8n (not included in JSON).
- **Edge cases / failure modes:**
  - Auth/OAuth errors, rate limits.
  - If multiple subreddits in one string aren’t supported, results may be empty or only the first subreddit may be used.

#### Node: Fetch YouTube Videos
- **Type / role:** YouTube (`n8n-nodes-base.youTube`) — searches videos.
- **Configuration choices:**
  - Resource: `video`
  - Filter query `q`: `={{ $json.youtubeSearchQuery }}`
  - `limit`: `20`
- **Inputs/outputs:**
  - **Input ←** Workflow Configuration
  - **Output →** Combine Research Data (input 1)
- **Credentials:** Requires YouTube Data API credentials (typically Google OAuth2 or API key depending on node config).
- **Edge cases / failure modes:**
  - Quota exceeded, auth errors.
  - Regional restrictions or empty query results.

#### Node: Fetch Industry News API
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls NewsAPI.
- **Configuration choices:**
  - URL: `={{ $json.newsApiUrl }}` (defaults to `https://newsapi.org/v2/everything`)
  - Query parameters:
    - `q`: `={{ $json.newsApiQuery }}`
    - `apiKey`: placeholder `"<__PLACEHOLDER_VALUE__NewsAPI Key__>"`
    - `pageSize`: `20`
    - `sortBy`: `relevancy`
  - Sends query string: enabled
- **Inputs/outputs:**
  - **Input ←** Workflow Configuration
  - **Output →** (Not connected in the provided graph)
- **Critical wiring issue:** This node is **not connected** to the Merge node, so its results are **not included** in the combined dataset.
- **Edge cases / failure modes:**
  - 401/403 if API key missing/invalid.
  - Response shape: NewsAPI returns `{ articles: [...] }` usually; downstream code expects per-item “article-like” fields (see Code node). You typically need to **split `articles` into items** before merging.

#### Node: Combine Research Data
- **Type / role:** Merge (`n8n-nodes-base.merge`) — combines streams.
- **Configuration choices:**
  - Mode: `combine`
  - Combine by: `combineAll`
- **Inputs/outputs:**
  - **Inputs ←** Fetch Reddit Discussions (0), Fetch YouTube Videos (1)
  - **Output →** Aggregate Insights
- **Edge cases / failure modes:**
  - With `combineAll`, output cardinality can be non-intuitive; verify how your n8n version pairs/combines items.
  - If one branch returns 0 items, combined output may be empty.

---

### Block 3 — Insight Aggregation
**Overview:** Converts raw mixed-source items into a single structured `insights` object (keywords, trends, questions, sentiment, source counts, summary).

**Nodes involved:**
- Aggregate Insights

#### Node: Aggregate Insights
- **Type / role:** Code (`n8n-nodes-base.code`) — custom JavaScript aggregation.
- **Key logic (interpreted):**
  - Reads all incoming items: `const items = $input.all();`
  - Attempts to detect source type by checking fields:
    - Reddit: `subreddit` or `reddit_url`
    - YouTube: `videoId` or `youtube_url` or `channelTitle`
    - News: `article` or `news_url` or `publishedAt`
  - Extracts `title` and `content` per source and:
    - Builds `topKeywords` frequency map (filters words length > 4, strips punctuation)
    - Extracts question-like sentences (simple `?` matching and wh-word detection)
    - Performs very simple sentiment scoring via keyword presence
    - Tracks `popularTopics` based on titles
  - Produces:
    - `insights.keyTrends` (top 5 keywords)
    - `insights.topKeywords` (top 20 with frequency)
    - `insights.popularTopics` (top 10 topics)
    - `insights.commonQuestions` (up to 15 unique)
    - `insights.sourceBreakdown`, `insights.sentimentAnalysis`
    - `insights.summary` text
- **Inputs/outputs:**
  - **Input ←** Combine Research Data
  - **Output →** SEO Content Generator Agent
- **Edge cases / failure modes:**
  - If upstream items don’t contain expected fields, source detection may classify as `unknown`, reducing usefulness.
  - NewsAPI data is not included (not merged) and even if merged, would likely require “articles array → items” transformation to match expectations.
  - Simple keyword/sentiment heuristics can be noisy (e.g., “issue” in neutral contexts).
  - Large input volumes can increase execution time; still typically fine at 40 items.

---

### Block 4 — AI-Driven SEO Content Creation (Disabled)
**Overview:** Uses a LangChain Agent with GPT‑4o, optionally calling SERP (Google Search) and Wikipedia tools, to generate a structured SEO landing page payload.

**Nodes involved:**
- SEO Content Generator Agent (disabled)
- OpenAI GPT-4
- Google Search for SEO Research
- Wikipedia Research
- Structured Content Output

#### Node: SEO Content Generator Agent (disabled)
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates LLM + tools + output parser.
- **Status:** **Disabled** (`disabled: true`). Downstream nodes will not run.
- **Prompt construction:**
  - Main text prompt injects:
    - Service/Audience/Industry/ContentType
    - `insights.summary`
    - `insights.keyTrends`
    - First 5 of `insights.commonQuestions`
  - System message instructs:
    - SEO-focused, data-driven landing page writing
    - Use Reddit/YouTube/News/Wikipedia/Google Search insights
    - ~800 words, professional approachable tone, conversion CTAs
    - Must return data matching output parser schema
- **Inputs/outputs:**
  - **Main input ←** Aggregate Insights
  - **Main output →** Convert to HTML
  - **AI connections:**
    - Language model input from **OpenAI GPT-4**
    - Tool connections from **Google Search for SEO Research** and **Wikipedia Research**
    - Output parser from **Structured Content Output**
- **Edge cases / failure modes:**
  - If disabled: no content generation; workflow effectively stops after aggregation.
  - If enabled: tool credentials missing (SERP, Wikipedia not required, OpenAI required) can cause agent failures.
  - Output parser failures if the model returns invalid JSON or misses required fields.

#### Node: OpenAI GPT-4
- **Type / role:** OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) — provides `gpt-4o`.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Temperature: `0.7`
- **Connections:**
  - **AI languageModel →** SEO Content Generator Agent
- **Credentials:** OpenAI API credential required.
- **Edge cases:**
  - Rate limits, invalid key, model access restrictions.
  - Higher temperature increases risk of schema deviations (mitigated by output parser, but not guaranteed).

#### Node: Google Search for SEO Research
- **Type / role:** SERP API tool (`@n8n/n8n-nodes-langchain.toolSerpApi`) — enables the agent to run search queries.
- **Configuration choices:** Default options (none specified).
- **Connections:**
  - **AI tool →** SEO Content Generator Agent
- **Credentials:** Typically requires a SerpAPI key configured in n8n.
- **Edge cases:**
  - Missing key, quota limits, tool timeouts.

#### Node: Wikipedia Research
- **Type / role:** Wikipedia tool (`@n8n/n8n-nodes-langchain.toolWikipedia`) — allows factual lookups.
- **Connections:**
  - **AI tool →** SEO Content Generator Agent
- **Edge cases:**
  - Ambiguous queries, irrelevant pages, rate limits (rare).

#### Node: Structured Content Output
- **Type / role:** Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces schema.
- **Schema (high-level):**
  - `headline` (string)
  - `subheadline` (string)
  - `metaTitle` (string)
  - `metaDescription` (string)
  - `keywords` (string[])
  - `mainContent` (string, markdown)
  - `callToAction` (string)
  - `targetKeywords` (string[])
- **Connections:**
  - **AI outputParser →** SEO Content Generator Agent
- **Edge cases:**
  - If the LLM returns invalid JSON or wrong types, parsing fails and the agent errors.

---

### Block 5 — Formatting & Storage
**Overview:** Converts the generated markdown body to HTML and stores the final record in Google Sheets.

**Nodes involved:**
- Convert to HTML
- Save to Google Sheets

#### Node: Convert to HTML
- **Type / role:** Markdown (`n8n-nodes-base.markdown`) — converts markdown to HTML.
- **Configuration choices:**
  - Mode: `markdownToHtml`
  - Markdown input: `={{ $json.mainContent }}`
- **Inputs/outputs:**
  - **Input ←** SEO Content Generator Agent
  - **Output →** Save to Google Sheets
- **Edge cases:**
  - If `mainContent` is missing (agent output malformed), conversion outputs empty or errors.
  - HTML may include elements your publishing stack sanitizes; consider post-processing.

#### Node: Save to Google Sheets
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — persists output.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Document ID: placeholder `"<__PLACEHOLDER_VALUE__Google Sheets Document ID__>"`
  - Sheet name: `"SEO Content"`
  - Columns mapping: auto-map input data
- **Inputs/outputs:**
  - **Input ←** Convert to HTML
  - No downstream outputs.
- **Credentials:** Google Sheets OAuth2 credential with write access.
- **Edge cases / failure modes:**
  - Missing/invalid document ID or sheet name.
  - Auto-mapping works best when sheet headers match JSON keys; otherwise values may not land in intended columns.
  - `appendOrUpdate` typically needs a defined key column to “update” deterministically; otherwise it may behave like append.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Test workflow' | manualTrigger | Manual entry point | — | Workflow Configuration | ## How It Works\nThis workflow automates SEO content creation by aggregating multi-source research and generating optimized articles. Designed for content marketers, SEO specialists, and digital agencies, it solves the time-consuming challenge of researching trending topics and crafting search-optimized content at scale. The system pulls discussions from Reddit, videos from YouTube, and industry news via APIs, then combines this data into comprehensive insights. An AI agent analyzes aggregated research, performs Google Search SEO analysis, consults Wikipedia for accuracy, and generates structured, SEO-optimized HTML content. The final output saves automatically to Google Sheets for easy management and publishing workflows. |
| Workflow Configuration | set | Defines run parameters (queries, tone, etc.) | When clicking 'Test workflow' | Fetch Reddit Discussions; Fetch YouTube Videos; Fetch Industry News API | ## Setup Steps\n1. Configure Reddit, YouTube, and industry news API credentials in fetch nodes\n2. Add OpenAI API key for GPT-4 agent and Google API key for search analysis\n3. Connect Google Sheets with write permissions for content storage |
| Fetch Reddit Discussions | reddit | Collect Reddit posts/questions | Workflow Configuration | Combine Research Data | ## Multi-Source Research Collection\n\n**What:** Fetches Reddit discussions, YouTube videos, and industry news API data, then combines all research  \n\n**Why:** Single-source content lacks depth and misses trending angles. |
| Fetch YouTube Videos | youTube | Collect YouTube trend signals | Workflow Configuration | Combine Research Data | ## Multi-Source Research Collection\n\n**What:** Fetches Reddit discussions, YouTube videos, and industry news API data, then combines all research  \n\n**Why:** Single-source content lacks depth and misses trending angles. |
| Fetch Industry News API | httpRequest | Collect industry news (NewsAPI) | Workflow Configuration | — | ## Multi-Source Research Collection\n\n**What:** Fetches Reddit discussions, YouTube videos, and industry news API data, then combines all research  \n\n**Why:** Single-source content lacks depth and misses trending angles. |
| Combine Research Data | merge | Merge research streams | Fetch Reddit Discussions; Fetch YouTube Videos | Aggregate Insights | ## Multi-Source Research Collection\n\n**What:** Fetches Reddit discussions, YouTube videos, and industry news API data, then combines all research  \n\n**Why:** Single-source content lacks depth and misses trending angles. |
| Aggregate Insights | code | Summarize and structure insights | Combine Research Data | SEO Content Generator Agent | ## AI-Driven SEO Content Creation\n\n**What:** AI agent processes research, runs Google Search SEO analysis \n\n**Why:** AI automation delivers SEO-optimized |
| SEO Content Generator Agent | langchain agent | Orchestrates GPT + tools; generates structured SEO content | Aggregate Insights | Convert to HTML | ## AI-Driven SEO Content Creation\n\n**What:** AI agent processes research, runs Google Search SEO analysis \n\n**Why:** AI automation delivers SEO-optimized |
| OpenAI GPT-4 | OpenAI chat model | LLM for generation (gpt-4o) | — (AI connection) | SEO Content Generator Agent (AI languageModel) | ## AI-Driven SEO Content Creation\n\n**What:** AI agent processes research, runs Google Search SEO analysis \n\n**Why:** AI automation delivers SEO-optimized |
| Google Search for SEO Research | SerpAPI tool | Tool for SERP/SEO research | — (AI connection) | SEO Content Generator Agent (AI tool) | ## AI-Driven SEO Content Creation\n\n**What:** AI agent processes research, runs Google Search SEO analysis \n\n**Why:** AI automation delivers SEO-optimized |
| Wikipedia Research | Wikipedia tool | Tool for background/factual checks | — (AI connection) | SEO Content Generator Agent (AI tool) | ## AI-Driven SEO Content Creation\n\n**What:** AI agent processes research, runs Google Search SEO analysis \n\n**Why:** AI automation delivers SEO-optimized |
| Structured Content Output | structured output parser | Enforces JSON schema for content output | — (AI connection) | SEO Content Generator Agent (AI outputParser) | ## AI-Driven SEO Content Creation\n\n**What:** AI agent processes research, runs Google Search SEO analysis \n\n**Why:** AI automation delivers SEO-optimized |
| Convert to HTML | markdown | Converts generated markdown to HTML | SEO Content Generator Agent | Save to Google Sheets | ## Response Coordination & Notifications\n\n**What:**\nAutomatically schedules maintenance or inspection teams and sends tailored email \n\n**Why:**\nFast, coordinated action reduces secondary damage. |
| Save to Google Sheets | googleSheets | Persists outputs to Google Sheets | Convert to HTML | — | ## Response Coordination & Notifications\n\n**What:**\nAutomatically schedules maintenance or inspection teams and sends tailored email \n\n**Why:**\nFast, coordinated action reduces secondary damage. |

Additional sticky note (general):
- **Prerequisites / Use Cases / Customization / Benefits** note exists and is broadly applicable to the workflow (see Section 5).

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node **Manual Trigger** named **“When clicking 'Test workflow'”**.

2) **Add Configuration Node**
   1. Add node **Set** named **“Workflow Configuration”**.
   2. Enable “Include Other Fields” (so downstream data isn’t discarded).
   3. Add fields:
      - `serviceType` (string): `teaching`
      - `targetAudience` (string): `students`
      - `industry` (string): `AI and Engineering`
      - `contentType` (string): `landing page`
      - `tone` (string): `professional yet approachable`
      - `wordCount` (number): `800`
      - `redditSubreddits` (string): `artificial,MachineLearning,engineering,learnprogramming`
      - `youtubeSearchQuery` (string): `AI engineering education`
      - `newsApiUrl` (string): `https://newsapi.org/v2/everything`
      - `newsApiQuery` (string): `AI engineering education training`
   4. Connect **Manual Trigger → Workflow Configuration**.

3) **Add Research Fetch Nodes (parallel fan-out)**
   1. Add **Reddit** node named **“Fetch Reddit Discussions”**:
      - Operation: **Search**
      - Subreddit: `={{ $json.redditSubreddits }}`
      - Keyword: `={{ $json.industry }} {{ $json.serviceType }}`
      - Limit: `20`
      - Configure Reddit credentials (OAuth/app) in n8n.
   2. Add **YouTube** node named **“Fetch YouTube Videos”**:
      - Resource: **Video**
      - Query `q`: `={{ $json.youtubeSearchQuery }}`
      - Limit: `20`
      - Configure YouTube credentials (Google/YouTube Data API) in n8n.
   3. Add **HTTP Request** node named **“Fetch Industry News API”**:
      - Method: GET
      - URL: `={{ $json.newsApiUrl }}`
      - Send Query Parameters: on
      - Parameters:
        - `q`: `={{ $json.newsApiQuery }}`
        - `apiKey`: your NewsAPI key
        - `pageSize`: `20`
        - `sortBy`: `relevancy`
   4. Connect **Workflow Configuration →** each of the three fetch nodes.

4) **Normalize NewsAPI output (recommended to make it usable)**
   - Add a node **Item Lists** (or **Code**) after **Fetch Industry News API** to split `articles` into items.
   - Example approach:
     - Use **Item Lists → Split Out Items** on field `articles`
   - This is necessary because the provided workflow does not do this, and the aggregation code expects item-level objects.

5) **Merge Research Streams**
   1. Add **Merge** node named **“Combine Research Data”**:
      - Mode: **Combine**
      - Combine By: **Combine All**
   2. Connect:
      - **Fetch Reddit Discussions → Combine Research Data (Input 0)**
      - **Fetch YouTube Videos → Combine Research Data (Input 1)**
   3. If you want news included, either:
      - Add a second Merge to merge in the news branch, or
      - Convert Merge mode to something that supports 3-way merging via chaining (Reddit+YouTube → Merge1; Merge1+News → Merge2).

6) **Aggregate Insights**
   1. Add **Code** node named **“Aggregate Insights”** and paste the aggregation logic (adapt as needed).
   2. Connect **Combine Research Data → Aggregate Insights**.

7) **Create AI Agent (LangChain)**
   1. Add **LangChain Agent** node named **“SEO Content Generator Agent”**.
   2. Ensure the node is **enabled**.
   3. Set:
      - Prompt type: “Define” (custom prompt)
      - Main prompt text using expressions:
        - Service/Audience/Industry/ContentType
        - `{{$json.insights.summary}}`
        - `{{$json.insights.keyTrends.join(", ")}}`
        - `{{$json.insights.commonQuestions.slice(0, 5).join("\n")}}`
      - System message with your SEO instructions (as in the workflow).
      - Enable “Has Output Parser”.
   4. Connect **Aggregate Insights → SEO Content Generator Agent**.

8) **Add LLM**
   1. Add **OpenAI Chat Model** node named **“OpenAI GPT-4”**:
      - Model: **gpt-4o**
      - Temperature: **0.7**
      - Configure **OpenAI API credentials** in n8n.
   2. Connect **OpenAI GPT-4 (AI languageModel) → SEO Content Generator Agent**.

9) **Add Agent Tools**
   1. Add **SerpAPI Tool** node named **“Google Search for SEO Research”**:
      - Configure SerpAPI credentials/key in n8n.
      - Connect **(AI tool) → SEO Content Generator Agent**.
   2. Add **Wikipedia Tool** node named **“Wikipedia Research”**:
      - Connect **(AI tool) → SEO Content Generator Agent**.

10) **Add Structured Output Parser**
   1. Add **Structured Output Parser** node named **“Structured Content Output”**.
   2. Set schema (manual) with fields: `headline`, `subheadline`, `metaTitle`, `metaDescription`, `keywords[]`, `mainContent` (markdown), `callToAction`, `targetKeywords[]`.
   3. Connect **Structured Content Output (AI outputParser) → SEO Content Generator Agent**.

11) **Convert Markdown to HTML**
   1. Add **Markdown** node named **“Convert to HTML”**:
      - Mode: Markdown to HTML
      - Markdown: `={{ $json.mainContent }}`
   2. Connect **SEO Content Generator Agent → Convert to HTML**.

12) **Save to Google Sheets**
   1. Add **Google Sheets** node named **“Save to Google Sheets”**:
      - Operation: **Append or Update**
      - Document ID: your spreadsheet ID
      - Sheet name: `SEO Content`
      - Mapping: Auto-map (or define explicit mapping to columns)
      - Configure **Google Sheets OAuth2** credentials with write scope.
   2. Connect **Convert to HTML → Save to Google Sheets**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Prerequisites\nReddit API access, YouTube Data API key, industry news API\n## Use Cases\nBlog content automation, competitive content analysis, trending topic research\n## Customization\nModify research sources, adjust AI prompts for brand voice, customize SEO parameters\n## Benefits\n10x faster content production, multi-platform research coverage | Applies to the overall workflow (Sticky Note content). |
| ## Setup Steps\n1. Configure Reddit, YouTube, and industry news API credentials in fetch nodes\n2. Add OpenAI API key for GPT-4 agent and Google API key for search analysis\n3. Connect Google Sheets with write permissions for content storage | Applies to initial setup and credentials (Sticky Note content). |
| **Data integration gap:** NewsAPI branch is not connected to the merge, and NewsAPI responses typically require splitting `articles[]` into items before aggregation. | Recommended fix before relying on “industry news” insights. |
| **Execution gap:** “SEO Content Generator Agent” is disabled, so content generation + Sheets persistence won’t run until enabled. | Enable the node and verify tool + parser wiring. |
| Sticky note “Response Coordination & Notifications” content appears unrelated to SEO workflow. | Treat as leftover/boilerplate commentary; does not match implemented nodes (no email/notifications exist). |