Find competitor keyword gaps and log opportunities to Notion with DataForSEO

https://n8nworkflows.xyz/workflows/find-competitor-keyword-gaps-and-log-opportunities-to-notion-with-dataforseo-13508


# Find competitor keyword gaps and log opportunities to Notion with DataForSEO

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow performs a **ranked keyword gap analysis** between *your domain/page* and a *competitor domain/page* using **DataForSEO Labs API**. It collects the top organic ranked keywords for both targets, finds keywords your competitor ranks for **that you do not**, and logs those “opportunity” keywords into a **Notion database** with key SEO metrics.

**Target use cases:**
- Content planning based on competitor keyword coverage
- Quick discovery of missing keywords for SEO expansion
- Creating a Notion-based backlog for content/SEO tasks

### 1.1 Trigger & Orchestration
Manual run entry point, then sequential API calls and processing.

### 1.2 Fetch “My site” ranked keywords (baseline set)
Calls DataForSEO to fetch your ranked keywords and extracts them into an array for later comparison.

### 1.3 Fetch competitor ranked keywords
Calls DataForSEO for competitor keywords, aligned to the same language/location as your site.

### 1.4 Gap detection & Notion logging
Splits competitor keyword list into individual items, filters out keywords already present in your set, and creates Notion database pages for the remaining opportunities.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & workflow start

**Overview:** Starts the workflow manually and kicks off the first DataForSEO call.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Sticky Note (global description & setup)

#### Node: **When clicking ‘Execute workflow’**
- **Type / role:** Manual Trigger (entry point)
- **Configuration:** No parameters; runs when user clicks *Execute workflow* in n8n.
- **Connections:**
  - **Output →** Get my ranked keywords
- **Edge cases / failures:** None (except if workflow requires credentials later and they’re missing).

#### Node: **Sticky Note**
- **Type / role:** Sticky Note (documentation)
- **Content highlights:**
  - Explains overall flow (DataForSEO → compare → Notion)
  - Includes setup link: https://app.dataforseo.com/api-access
  - Suggests a Notion AI prompt for turning gaps into a content strategy
- **Applies to:** Whole workflow (documentation only; no runtime effect)

---

### Block 2 — Fetch and prepare “My site” keyword list

**Overview:** Pulls your top ranked keywords from DataForSEO and converts them into an array (`myKeywords`) used for gap comparison.

**Nodes involved:**
- Get my ranked keywords
- Edit fields (myKeywords)
- Sticky Note1

#### Node: **Get my ranked keywords**
- **Type / role:** DataForSEO Labs API node (`get-ranked-keywords`)
- **Configuration (interpreted):**
  - **Operation:** `get-ranked-keywords`
  - **Target (`target_any`):** `https://dataforseo.com/apis/serp-api` (your site/page to analyze)
  - **Location:** `united states`
  - **Language:** `english`
  - **Offset:** set to `"="` (this looks unintended; typically offset is a number like `0`)
- **Credentials:** DataForSEO account (login/password-based for API)
- **Output:** A DataForSEO response structure (notably `tasks[0].result[0].items` list of ranked keyword objects)
- **Connections:**
  - **Output →** Edit fields (myKeywords)
- **Edge cases / failures:**
  - Invalid credentials / authorization failure
  - Target URL not accepted or wrong format for DataForSEO “target_any”
  - Quota/credits exceeded (DataForSEO)
  - If response contains no `tasks[0].result[0].items`, downstream mapping may fail

#### Node: **Edit fields (myKeywords)**
- **Type / role:** Set node (field shaping / transformation)
- **Configuration (interpreted):**
  - Creates a new field `myKeywords` (type: array)
  - Value expression:
    - `{{ $('Get my ranked keywords').item.json.tasks[0].result[0].items.map(item => item.keyword_data.keyword) }}`
  - This builds an array of keyword strings from your ranked keywords.
- **Connections:**
  - **Input ←** Get my ranked keywords
  - **Output →** Get my competitor’s ranked keywords
- **Edge cases / failures:**
  - If `items` is missing or not an array, `.map(...)` throws an expression/runtime error
  - If any item lacks `keyword_data.keyword`, you may get `undefined` values in `myKeywords`

#### Node: **Sticky Note1**
- **Type / role:** Sticky Note (documentation)
- **Content:** Guidance for configuring the “my ranked keywords” pull (target, location, language, parameters)
- **Applies to nodes:** Get my ranked keywords (and generally this block)

---

### Block 3 — Fetch competitor ranked keywords

**Overview:** Pulls competitor ranked keywords using the same language and location returned from the “my site” DataForSEO call, ensuring comparable datasets.

**Nodes involved:**
- Get my competitor’s ranked keywords
- Sticky Note2

#### Node: **Get my competitor’s ranked keywords**
- **Type / role:** DataForSEO Labs API node (`get-ranked-keywords`)
- **Configuration (interpreted):**
  - **Operation:** `get-ranked-keywords`
  - **Target (`target_any`):** `https://serpapi.com/` (competitor site/page)
  - **Offset:** `0`
  - **Language:** `{{ $('Get my ranked keywords').item.json.tasks[0].data.language_name }}`
  - **Location:** `{{ $('Get my ranked keywords').item.json.tasks[0].data.location_name }}`
  - This “inherits” geo/language settings from the first call to keep analysis consistent.
- **Credentials:** DataForSEO account (same credential as prior node)
- **Connections:**
  - **Input ←** Edit fields (myKeywords)
  - **Output →** Split out (competitor keywords)
- **Edge cases / failures:**
  - If the first DataForSEO call doesn’t return `tasks[0].data.language_name` or `location_name`, expressions resolve to null/empty and the API call may fail or behave unexpectedly
  - Same DataForSEO API failure modes as earlier (quota/auth/invalid target)

#### Node: **Sticky Note2**
- **Type / role:** Sticky Note (documentation)
- **Content:** Guidance for configuring competitor target and parameters
- **Applies to nodes:** Get my competitor’s ranked keywords

---

### Block 4 — Gap detection & Notion logging

**Overview:** Converts competitor keyword list into individual items, filters out keywords already present in `myKeywords`, and writes remaining opportunities to Notion as database pages.

**Nodes involved:**
- Split out (competitor keywords)
- Filter (My site doesn't have keyword)
- Create a database page
- Sticky Note3

#### Node: **Split out (competitor keywords)**
- **Type / role:** Split Out (itemization of an array field)
- **Configuration (interpreted):**
  - **Field to split out:** `tasks[0].result[0].items`
  - Produces **one n8n item per competitor keyword entry**.
- **Connections:**
  - **Input ←** Get my competitor’s ranked keywords
  - **Output →** Filter (My site doesn't have keyword)
- **Edge cases / failures:**
  - If competitor response has no `tasks[0].result[0].items` or it’s empty, nothing proceeds to filter/logging
  - If the field is not an array, Split Out will fail

#### Node: **Filter (My site doesn't have keyword)**
- **Type / role:** Filter (remove keywords already present on your site list)
- **Configuration (interpreted):**
  - Condition: **`myKeywords` NOT CONTAINS competitor keyword**
  - Left value:
    - `{{ $('Edit fields (myKeywords)').item.json.myKeywords }}`
  - Right value:
    - `{{ $json.keyword_data.keyword }}`
  - Uses “strict” type validation and array operation `notContains`.
- **Connections:**
  - **Input ←** Split out (competitor keywords)
  - **Output →** Create a database page (for items that pass filter)
- **Edge cases / failures:**
  - If `myKeywords` is not an array (missing or wrong type), filter may error or never match
  - Keyword casing differences (e.g., “Shoes” vs “shoes”) will be treated as different keywords because matching is case-sensitive (per node options)
  - Duplicate competitor keywords could create duplicate Notion pages (no deduplication step)

#### Node: **Create a database page**
- **Type / role:** Notion node — Create Database Page (write opportunity rows)
- **Configuration (interpreted):**
  - **Resource:** Database Page
  - **Database:** “Competitor” (Notion database ID `2f6a8ae5-ccb4-8020-9cec-def2626d321a`)
  - **Properties mapped:**
    - **Competitor’s Ranked Keyword (title):**
      - `{{ $('Split out (competitor keywords)').item.json.keyword_data.keyword }}`
    - **Keyword’s Search Volume (number):**
      - `{{ $('Split out (competitor keywords)').item.json.keyword_data.keyword_info.search_volume }}`
    - **Competitor’s Position (number):**
      - `{{ $('Split out (competitor keywords)').item.json.ranked_serp_element.serp_item.rank_group }}`
    - **Competitor’s URL (url):**
      - `{{ $('Split out (competitor keywords)').item.json.ranked_serp_element.serp_item.url }}`
      - `ignoreIfEmpty: true`
    - **Keyword’s Competition (number):**
      - `{{ $('Split out (competitor keywords)').item.json.keyword_data.keyword_info.competition }}`
- **Credentials:** Notion API credential (“Notion account”)
- **Connections:**
  - **Input ←** Filter (My site doesn't have keyword)
  - **Output:** None (end)
- **Edge cases / failures:**
  - Notion auth issues, revoked access, or missing database permissions
  - Property type mismatch (e.g., the Notion property isn’t actually a Number/URL/Title)
  - Rate limiting by Notion if many pages are created quickly
  - If `ranked_serp_element.serp_item.url` is empty/undefined, it will be skipped due to `ignoreIfEmpty`

#### Node: **Sticky Note3**
- **Type / role:** Sticky Note (documentation)
- **Content:** Recommends Notion database fields and example expressions.
- **Important note:** The example mappings shown in the sticky note appear **partially swapped/incorrect** (it lists rank_group under “Keyword Competition” and URL under “Competitor Position”). The actual node configuration is correct: rank_group → position, competition → competition, url → URL.
- **Applies to nodes:** Split out / Filter / Create a database page

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Get my ranked keywords | This workflow uses the DataForSEO APIs to perform a ranked keyword gap analysis between your website and a competitor. It pulls the top 100 organic keywords for both domains, identifies keywords your competitor ranks for but you don’t, and saves your keyword opportunities into a Notion database.  / Once the data is stored, you can use Notion AI to turn this keyword gap data into an actionable content plan. Recommended prompt: “Analyze the keyword gap between my website’s page {{your URL}} and competitor website’s page and build a content strategy for me.” / How it works + Setup steps (includes https://app.dataforseo.com/api-access). |
| Get my ranked keywords | DataForSEO Labs API | Fetch ranked keywords for your site/page | When clicking ‘Execute workflow’ | Edit fields (myKeywords) | ## Get my ranked keywords / Create a DataForSEO connection and set up Your target website/page, location, language and additional parameters if needed. |
| Edit fields (myKeywords) | Set | Build array of your keywords for comparison | Get my ranked keywords | Get my competitor’s ranked keywords | ## Get my ranked keywords / Create a DataForSEO connection and set up Your target website/page, location, language and additional parameters if needed. |
| Get my competitor’s ranked keywords | DataForSEO Labs API | Fetch ranked keywords for competitor | Edit fields (myKeywords) | Split out (competitor keywords) | ## Get my competitor’s ranked keywords / Select a DataForSEO connection, set up a Competitor website/page, and specify additional parameters if needed. |
| Split out (competitor keywords) | Split Out | Turn competitor keyword array into individual items | Get my competitor’s ranked keywords | Filter (My site doesn't have keyword) | ## Find keyword gaps and log the opportunities in Notion / Create a Notion connection... (note: sticky note’s sample field mappings appear partially swapped vs actual node config). |
| Filter (My site doesn't have keyword) | Filter | Keep only competitor keywords not in your keyword array | Split out (competitor keywords) | Create a database page | ## Find keyword gaps and log the opportunities in Notion / Create a Notion connection... (note: sticky note’s sample field mappings appear partially swapped vs actual node config). |
| Create a database page | Notion | Write keyword gap opportunities into Notion database | Filter (My site doesn't have keyword) | — | ## Find keyword gaps and log the opportunities in Notion / Create a Notion connection... (note: sticky note’s sample field mappings appear partially swapped vs actual node config). |
| Sticky Note | Sticky Note | Documentation | — | — | This workflow uses the DataForSEO APIs... (includes https://app.dataforseo.com/api-access). |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## Get my ranked keywords / Create a DataForSEO connection... |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## Get my competitor’s ranked keywords / Select a DataForSEO connection... |
| Sticky Note3 | Sticky Note | Documentation | — | — | ## Find keyword gaps and log the opportunities in Notion / Notion DB field suggestions + expressions (partially swapped). |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add node: Manual Trigger**
   - Name: `When clicking ‘Execute workflow’`

3. **Add node: DataForSEO Labs API**
   - Name: `Get my ranked keywords`
   - Credentials: create/select **DataForSEO API** credential using your API login/password (from https://app.dataforseo.com/api-access)
   - Operation: **get-ranked-keywords**
   - Set:
     - **target_any:** your domain/page (example in workflow: `https://dataforseo.com/apis/serp-api`)
     - **location_name:** `united states`
     - **language_name:** `english`
     - **offset:** set to `0` (recommended; the provided workflow shows `"="`, which may cause issues)

4. **Add node: Set**
   - Name: `Edit fields (myKeywords)`
   - Add field:
     - Name: `myKeywords`
     - Type: **Array**
     - Value (expression):
       - `$('Get my ranked keywords').item.json.tasks[0].result[0].items.map(item => item.keyword_data.keyword)`
   - This node should output the original item plus `myKeywords`.

5. **Add node: DataForSEO Labs API**
   - Name: `Get my competitor’s ranked keywords`
   - Credentials: same DataForSEO credential
   - Operation: **get-ranked-keywords**
   - Set:
     - **target_any:** competitor domain/page (example: `https://serpapi.com/`)
     - **offset:** `0`
     - **language_name** (expression):
       - `$('Get my ranked keywords').item.json.tasks[0].data.language_name`
     - **location_name** (expression):
       - `$('Get my ranked keywords').item.json.tasks[0].data.location_name`

6. **Add node: Split Out**
   - Name: `Split out (competitor keywords)`
   - Field to split out:
     - `tasks[0].result[0].items`

7. **Add node: Filter**
   - Name: `Filter (My site doesn't have keyword)`
   - Condition (Array → notContains):
     - **Left value (expression):**
       - `$('Edit fields (myKeywords)').item.json.myKeywords`
     - **Right value (expression):**
       - `$json.keyword_data.keyword`
   - Keep case sensitivity in mind (default behavior here is case-sensitive).

8. **Add node: Notion**
   - Name: `Create a database page`
   - Credentials: create/select a **Notion API** credential (OAuth/token depending on your n8n setup)
   - Resource: **Database Page**
   - Database: select your target database (create one if needed) with these properties:
     - **Competitor’s Ranked Keyword** (Title)
     - **Keyword’s Search Volume** (Number)
     - **Keyword’s Competition** (Number)
     - **Competitor’s Position** (Number)
     - **Competitor’s URL** (URL)
   - Map properties using expressions:
     - Title: `$('Split out (competitor keywords)').item.json.keyword_data.keyword`
     - Search Volume: `$('Split out (competitor keywords)').item.json.keyword_data.keyword_info.search_volume`
     - Competition: `$('Split out (competitor keywords)').item.json.keyword_data.keyword_info.competition`
     - Position: `$('Split out (competitor keywords)').item.json.ranked_serp_element.serp_item.rank_group`
     - URL: `$('Split out (competitor keywords)').item.json.ranked_serp_element.serp_item.url` (enable “ignore if empty” if available)

9. **Connect nodes in this order:**
   1) Manual Trigger → 2) Get my ranked keywords → 3) Edit fields (myKeywords) → 4) Get my competitor’s ranked keywords → 5) Split out → 6) Filter → 7) Notion Create page

10. (Optional but recommended) **Add safeguards**
   - Ensure `offset` is numeric
   - Add a deduplication step before Notion (to avoid repeated pages)
   - Add error handling or rate limiting if the result set is large

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| DataForSEO API access requires API login/password and sufficient credits. | https://app.dataforseo.com/api-access |
| Suggested Notion AI prompt to turn stored gaps into a content strategy: “Analyze the keyword gap between my website’s page {{your URL}} and competitor website’s page and build a content strategy for me.” | Mentioned in workflow sticky note |
| Sticky note mapping examples appear partially swapped compared to the actual Notion node mapping (rank vs competition vs URL). Use the Notion node’s current expressions as the source of truth. | Applies to the “Find keyword gaps and log…” note block |