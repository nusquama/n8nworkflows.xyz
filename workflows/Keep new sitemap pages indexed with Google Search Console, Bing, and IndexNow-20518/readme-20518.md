Keep new sitemap pages indexed with Google Search Console, Bing, and IndexNow

https://n8nworkflows.xyz/workflows/keep-new-sitemap-pages-indexed-with-google-search-console--bing--and-indexnow-20518


# Keep new sitemap pages indexed with Google Search Console, Bing, and IndexNow

### 1. Workflow Overview

This workflow automates the daily discovery, filtering, prioritization, and submission of new web pages to major search engines and indexing systems. Its primary objective is to ensure fresh content is rapidly indexed without exhausting crawl budgets or API limits. 

The process is structured into the following functional blocks:
- **1.1 Schedule & Ingestion:** Triggers daily execution and downloads the primary sitemap or sitemap index.
- **1.2 Parsing & Deduplication:** Extracts individual URLs from XML and checks an n8n Data Table to filter out previously submitted links.
- **1.3 Classification & Routing:** Evaluates each new URL against deterministic rules to assign priority levels (Cornerstone/Homepage, Normal Content, or Low-Priority/Utility).
- **1.4 Cornerstone Submission:** Upserts high-priority URLs into tracking storage, batches them, and submits them in parallel to Bing Webmaster Tools, IndexNow, and Google Search Console (via sitemap resubmission).
- **1.5 Content Submission:** Upserts standard content URLs into tracking storage, batches them, and submits them to Bing and IndexNow.
- **1.6 Digest & Reporting:** Compiles success counts, submission metrics, and error statuses across all branches into a single consolidated Telegram message.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Ingestion
- **Overview:** Initiates the workflow daily at a set hour and fetches the root sitemap XML file from the target domain.
- **Nodes Involved:** `When Daily at 9am`, `Fetch Sitemap XML`.
- **Node Details:**
  - **When Daily at 9am**
    - *Type:* `n8n-nodes-base.scheduleTrigger` (v1.3)
    - *Technical Role:* Time-based entry point.
    - *Configuration:* Configured to fire once every day at 09:00.
    - *Connections:* Output connects to `Fetch Sitemap XML`.
    - *Edge Cases:* Server downtime or timezone shifts may delay execution.
  - **Fetch Sitemap XML**
    - *Type:* `n8n-nodes-base.httpRequest` (v4.4)
    - *Technical Role:* HTTP GET request to download sitemap data.
    - *Configuration:* Requests `https://YOUR_DOMAIN_ROOT/sitemap_index.xml` expecting a plain text/XML response.
    - *Connections:* Input from `When Daily at 9am`; outputs to `Parse Sitemap URLs` and `Compile Telegram Message`.
    - *Edge Cases:* DNS failures, HTTP 404/500 errors on the sitemap URL, or invalid XML formats.

#### 2.2 Parsing & Deduplication
- **Overview:** Converts raw sitemap XML into discrete URL items and queries an n8n Data Table to remove previously tracked URLs.
- **Nodes Involved:** `Parse Sitemap URLs`, `Verify URL Submission`.
- **Node Details:**
  - **Parse Sitemap URLs**
    - *Type:* `n8n-nodes-base.code` (v2)
    - *Technical Role:* JavaScript code node to parse XML and extract `<loc>` tags. Handles sitemap index files up to one level deep (max 20 child sitemaps).
    - *Expressions/Variables:* Processes `$input.first().json.data` / `body` and performs recursive helper requests for sub-sitemaps.
    - *Connections:* Input from `Fetch Sitemap XML`; output to `Verify URL Submission`.
    - *Edge Cases:* Malformed XML structures, timeouts on child sitemap fetches.
  - **Verify URL Submission**
    - *Type:* `n8n-nodes-base.dataTable` (v1.1)
    - *Technical Role:* Database lookup to filter out already submitted URLs.
    - *Configuration:* Matches rows where column `url` equals `{{ $json.url }}` using operation `rowNotExists`.
    - *Connections:* Input from `Parse Sitemap URLs`; output to `Determine URL Priority`.
    - *Edge Cases:* Data Table connectivity drops or misconfigured Table IDs.

#### 2.3 Classification & Routing
- **Overview:** Categorizes new URLs based on path patterns and routes them into respective priority branches.
- **Nodes Involved:** `Determine URL Priority`, `Route by Priority Level`, `Bypass Low Priority URLs`.
- **Node Details:**
  - **Determine URL Priority**
    - *Type:* `n8n-nodes-base.code` (v2)
    - *Technical Role:* Deterministic URL classifier assigning priority scores (`2` for cornerstone/homepage, `1` for normal content, `0` for utility/archive).
    - *Connections:* Input from `Verify URL Submission`; output to `Route by Priority Level`.
    - *Edge Cases:* Non-standard URL structures resulting in misclassification.
  - **Route by Priority Level**
    - *Type:* `n8n-nodes-base.switch` (v3.4)
    - *Technical Role:* Conditional router splitting items based on the assigned priority score.
    - *Configuration:* Routes `priority >= 2` to output 0, `priority >= 1` to output 1, and default/lower priorities to output 2.
    - *Connections:* Input from `Determine URL Priority`; outputs to `Note Cornerstone Submission`, `Record Content Submission`, and `Bypass Low Priority URLs`.
  - **Bypass Low Priority URLs**
    - *Type:* `n8n-nodes-base.noOp` (v1)
    - *Technical Role:* Termination point for low-value or utility pages.
    - *Connections:* Input from `Route by Priority Level`; no downstream nodes.

#### 2.4 Cornerstone Submission
- **Overview:** Records high-priority URLs in tracking storage, aggregates them, and pushes them concurrently to Bing, IndexNow, and Google Search Console.
- **Nodes Involved:** `Note Cornerstone Submission`, `Gather Cornerstone URLs`, `Post Cornerstones to Bing`, `Post Cornerstones to IndexNow`, `Resubmit Sitemap to Google API`, `Combine Cornerstone Submissions`.
- **Node Details:**
  - **Note Cornerstone Submission**
    - *Type:* `n8n-nodes-base.dataTable` (v1.1)
    - *Technical Role:* Upserts URL metadata (`url`, `kind`, `priority`, `first_seen`) into the tracking table.
    - *Connections:* Input from `Route by Priority Level`; output to `Gather Cornerstone URLs`.
  - **Gather Cornerstone URLs**
    - *Type:* `n8n-nodes-base.aggregate` (v1)
    - *Technical Role:* Collects individual items into a single array under `url_list`.
    - *Connections:* Input from `Note Cornerstone Submission`; outputs to Bing, IndexNow, and Google API nodes.
  - **Post Cornerstones to Bing**
    - *Type:* `n8n-nodes-base.httpRequest` (v4.4)
    - *Technical Role:* POST request to Bing Webmaster Tools API batch submission endpoint.
    - *Configuration:* `onError: continueErrorOutput`. Sends JSON body containing `siteUrl` and a sliced array of up to 40 URLs.
    - *Credentials:* `YOUR_BING_API_KEY`.
    - *Connections:* Input from `Gather Cornerstone URLs`; output to `Combine Cornerstone Submissions`.
  - **Post Cornerstones to IndexNow**
    - *Type:* `n8n-nodes-base.httpRequest` (v4.4)
    - *Technical Role:* POST request to IndexNow submission API endpoint (`api.indexnow.org`).
    - *Configuration:* `onError: continueErrorOutput`. Submits up to 10,000 URLs with domain host, key, and key location.
    - *Connections:* Input from `Gather Cornerstone URLs`; output to `Combine Cornerstone Submissions`.
  - **Resubmit Sitemap to Google API**
    - *Type:* `n8n-nodes-base.httpRequest` (v4.4)
    - *Technical Role:* PUT request to Google Search Console API for sitemap resubmission.
    - *Configuration:* `onError: continueErrorOutput`. Uses predefined Google API credentials.
    - *Credentials:* `googleApi` (Requires `https://www.googleapis.com/auth/webmasters` scope).
    - *Connections:* Input from `Gather Cornerstone URLs`; output to `Combine Cornerstone Submissions`.
  - **Combine Cornerstone Submissions**
    - *Type:* `n8n-nodes-base.merge` (v3.2)
    - *Technical Role:* Merges responses from all three API calls into a single execution stream.
    - *Configuration:* Append mode with 3 inputs.
    - *Connections:* Inputs from Bing, IndexNow, and Google API nodes; output to `Compile Telegram Message`.

#### 2.5 Content Submission
- **Overview:** Records standard content URLs, aggregates them, and submits them in parallel to Bing and IndexNow.
- **Nodes Involved:** `Record Content Submission`, `Gather Content URLs`, `Post Content to Bing`, `Post Content to IndexNow`, `Combine Content Submissions`.
- **Node Details:**
  - **Record Content Submission**
    - *Type:* `n8n-nodes-base.dataTable` (v1.1)
    - *Technical Role:* Upserts standard content URLs into the tracking table.
    - *Connections:* Input from `Route by Priority Level`; output to `Gather Content URLs`.
  - **Gather Content URLs**
    - *Type:* `n8n-nodes-base.aggregate` (v1)
    - *Technical Role:* Aggregates content URL items into a single array (`url_list`).
    - *Connections:* Input from `Record Content Submission`; outputs to Bing and IndexNow HTTP request nodes.
  - **Post Content to Bing**
    - *Type:* `n8n-nodes-base.httpRequest` (v4.4)
    - *Technical Role:* POST request to Bing Webmaster Tools API for content batch submission.
    - *Configuration:* `onError: continueErrorOutput`. Batches up to 40 URLs.
    - *Connections:* Input from `Gather Content URLs`; output to `Combine Content Submissions`.
  - **Post Content to IndexNow**
    - *Type:* `n8n-nodes-base.httpRequest` (v4.4)
    - *Technical Role:* POST request to IndexNow API for content batch submission.
    - *Configuration:* `onError: continueErrorOutput`. Batches up to 10,000 URLs.
    - *Connections:* Input from `Gather Content URLs`; output to `Combine Content Submissions`.
  - **Combine Content Submissions**
    - *Type:* `n8n-nodes-base.merge` (v3.2)
    - *Technical Role:* Merges API responses from Bing and IndexNow content submissions.
    - *Configuration:* Append mode with 2 inputs.
    - *Connections:* Inputs from Bing and IndexNow content nodes; output to `Compile Telegram Message`.

#### 2.6 Digest & Reporting
- **Overview:** Compiles execution statistics, submission counts, and API response codes into a formatted summary message sent via Telegram.
- **Nodes Involved:** `Compile Telegram Message`, `Dispatch Telegram Digest`.
- **Node Details:**
  - **Compile Telegram Message**
    - *Type:* `n8n-nodes-base.code` (v2)
    - *Technical Role:* JavaScript code node that safely evaluates upstream node outputs using `try/catch` wrappers and constructs a markdown digest string.
    - *Connections:* Inputs from `Fetch Sitemap XML`, `Combine Cornerstone Submissions`, and `Combine Content Submissions`; output to `Dispatch Telegram Digest`.
    - *Edge Cases:* Missing upstream nodes or empty branches handled via robust fallback checks.
  - **Dispatch Telegram Digest**
    - *Type:* `n8n-nodes-base.telegram` (v1.2)
    - *Technical Role:* Sends the compiled markdown digest to the configured Telegram chat ID.
    - *Configuration:* Uses expression `{{ $json.digest }}` for text content.
    - *Credentials:* Telegram Bot Token (`YOUR_TELEGRAM_CHAT_ID`).
    - *Connections:* Input from `Compile Telegram Message`; no downstream nodes.
    - *Edge Cases:* Invalid chat ID or revoked bot token.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Daily at 9am | scheduleTrigger | Time-based entry point | None | Fetch Sitemap XML | ## Keep new pages indexed on Google, Bing and IndexNow<br><br>### How it works<br><br>This workflow runs daily, fetches the site sitemap, parses it into individual URLs, and checks which URLs have not already been submitted. It classifies each new URL by priority, skips low-priority pages, and sends higher-priority URLs to Bing, IndexNow, and for cornerstone pages also resubmits the sitemap to Google Search Console. It then builds and sends a Telegram digest summarizing the run and indexing submissions.<br><br>### Setup steps<br><br>- Set the schedule trigger time and update the sitemap URL in the Fetch Sitemap node to your domain’s sitemap or sitemap index.<br>- Configure the Data Table nodes used to track submitted URLs so the workflow can avoid duplicate submissions.<br>- Add credentials or API keys for Bing Webmaster URL submission, IndexNow, and Google Search Console sitemap submission.<br>- Configure the IndexNow key and make sure the key file is hosted and accessible on the target domain if required.<br>- Connect the Telegram bot credentials and set the target chat ID for the digest message.<br><br>### Customization<br><br>Adjust the URL priority rules in the Classify URL Priority code node, change which branches submit to Google/Bing/IndexNow, or tune the daily schedule and Telegram digest format. |
| Fetch Sitemap XML | httpRequest | Download sitemap XML | When Daily at 9am | Parse Sitemap URLs, Compile Telegram Message | ## Schedule and fetch sitemap<br><br>Starts the workflow once per day and downloads the configured sitemap that will be inspected for new URLs. |
| Parse Sitemap URLs | code | Parse XML into discrete URLs | Fetch Sitemap XML | Verify URL Submission | ## Parse and deduplicate URLs<br><br>Converts the sitemap XML into individual URL items and checks the tracking table so previously submitted URLs are not processed again. |
| Verify URL Submission | dataTable | Filter out previously submitted URLs | Parse Sitemap URLs | Determine URL Priority | ## Parse and deduplicate URLs<br><br>Converts the sitemap XML into individual URL items and checks the tracking table so previously submitted URLs are not processed again. |
| Determine URL Priority | code | Assign priority tiers based on URL structure | Verify URL Submission | Route by Priority Level | ## Classify and route URLs<br><br>Assigns each new URL a deterministic priority and branches the workflow into cornerstone, normal content, or low-priority handling paths. |
| Route by Priority Level | switch | Conditional branch router | Determine URL Priority | Note Cornerstone Submission, Record Content Submission, Bypass Low Priority URLs | ## Classify and route URLs<br><br>Assigns each new URL a deterministic priority and branches the workflow into cornerstone, normal content, or low-priority handling paths. |
| Bypass Low Priority URLs | noOp | Terminal point for low-priority links | Route by Priority Level | None | ## Skip low priority URLs<br><br>Terminates the low-priority branch without submitting those URLs to any indexing service. |
| Note Cornerstone Submission | dataTable | Upsert cornerstone URLs into tracking table | Route by Priority Level | Gather Cornerstone URLs | ## Prepare cornerstone URLs<br><br>Marks cornerstone URLs as submitted in the tracking table and aggregates them into a batch for indexing API calls. |
| Gather Cornerstone URLs | aggregate | Aggregate cornerstone items into array | Note Cornerstone Submission | Post Cornerstones to Bing, Post Cornerstones to IndexNow, Resubmit Sitemap to Google API | ## Prepare cornerstone URLs<br><br>Marks cornerstone URLs as submitted in the tracking table and aggregates them into a batch for indexing API calls. |
| Post Cornerstones to Bing | httpRequest | Submit cornerstone batch to Bing | Gather Cornerstone URLs | Combine Cornerstone Submissions | ## Submit cornerstone indexing<br><br>Submits cornerstone URL batches to Bing and IndexNow, resubmits the sitemap to Google Search Console, and merges the resulting API responses. |
| Post Cornerstones to IndexNow | httpRequest | Push cornerstone batch to IndexNow | Gather Cornerstone URLs | Combine Cornerstone Submissions | ## Submit cornerstone indexing<br><br>Submits cornerstone URL batches to Bing and IndexNow, resubmits the sitemap to Google Search Console, and merges the resulting API responses. |
| Resubmit Sitemap to Google API | httpRequest | Resubmit sitemap XML to Search Console | Gather Cornerstone URLs | Combine Cornerstone Submissions | ## Submit cornerstone indexing<br><br>Submits cornerstone URL batches to Bing and IndexNow, resubmits the sitemap to Google Search Console, and merges the resulting API responses. |
| Combine Cornerstone Submissions | merge | Merge cornerstone API response streams | Post Cornerstones to Bing, Post Cornerstones to IndexNow, Resubmit Sitemap to Google API | Compile Telegram Message | ## Submit cornerstone indexing<br><br>Submits cornerstone URL batches to Bing and IndexNow, resubmits the sitemap to Google Search Console, and merges the resulting API responses. |
| Record Content Submission | dataTable | Upsert standard content URLs into tracking table | Route by Priority Level | Gather Content URLs | ## Prepare content URLs<br><br>Marks standard content URLs as submitted and aggregates them into a batch for the normal content indexing path. |
| Gather Content URLs | aggregate | Aggregate content items into array | Record Content Submission | Post Content to Bing, Post Content to IndexNow | ## Prepare content URLs<br><br>Marks standard content URLs as submitted and aggregates them into a batch for the normal content indexing path. |
| Post Content to Bing | httpRequest | Submit content batch to Bing | Gather Content URLs | Combine Content Submissions | ## Submit content indexing<br><br>Sends standard content URL batches to Bing and IndexNow, then merges the returned submission results. |
| Post Content to IndexNow | httpRequest | Push content batch to IndexNow | Gather Content URLs | Combine Content Submissions | ## Submit content indexing<br><br>Sends standard content URL batches to Bing and IndexNow, then merges the returned submission results. |
| Combine Content Submissions | merge | Merge content API response streams | Post Content to Bing, Post Content to IndexNow | Compile Telegram Message | ## Submit content indexing<br><br>Sends standard content URL batches to Bing and IndexNow, then merges the returned submission results. |
| Compile Telegram Message | code | Compile summary digest string | Fetch Sitemap XML, Combine Cornerstone Submissions, Combine Content Submissions | Dispatch Telegram Digest | ## Send Telegram report<br><br>Builds a single digest for the run using sitemap and submission counts, then sends the report to Telegram. |
| Dispatch Telegram Digest | telegram | Send execution report via Telegram | Compile Telegram Message | None | ## Send Telegram report<br><br>Builds a single digest for the run using sitemap and submission counts, then sends the report to Telegram. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger:** Add a `Schedule Trigger` node (`When Daily at 9am`), set interval to 1 day, and configure it to trigger at `09:00`.
2. **Add Sitemap Fetcher:** Create an `HTTP Request` node (`Fetch Sitemap XML`), set method to `GET`, URL to `https://YOUR_DOMAIN_ROOT/sitemap_index.xml`, and response format to `Text`. Connect `When Daily at 9am` to it.
3. **Add Sitemap Parser:** Create a `Code` node (`Parse Sitemap URLs`), set mode to `Run Once for All Items`, and paste the JavaScript sitemap parsing logic handling XML `<loc>` extraction and child indexes. Connect `Fetch Sitemap XML` output to this node.
4. **Setup Deduplication Table:** Create a `Data Table` node (`Verify URL Submission`), set resource to `Row`, operation to `Row Not Exists`, select your Data Table ID, and configure a filter where column `url` equals `{{ $json.url }}`. Connect `Parse Sitemap URLs` to this node.
5. **Add Priority Classifier:** Create a `Code` node (`Determine URL Priority`), set mode to `Run Once for All Items`, and paste the deterministic rule classifier script that assigns priority values (`2`, `1`, `0`). Connect `Verify URL Submission` to it.
6. **Add Priority Switch Router:** Create a `Switch` node (`Route by Priority Level`), configure rules to evaluate `{{ $json.priority }}` (Output 0 for `>= 2`, Output 1 for `>= 1`, Output 2 for default/utility). Connect `Determine URL Priority` to it.
7. **Configure Low-Priority Branch:** Create a `NoOp` node (`Bypass Low Priority URLs`) and connect Output 2 of the Switch node to it.
8. **Configure Cornerstone Branch:**
   - Create a `Data Table` node (`Note Cornerstone Submission`), set resource to `Row`, operation to `Upsert`, and map columns (`url`, `kind`, `priority`, `first_seen`). Connect Switch Output 0 to this node.
   - Create an `Aggregate` node (`Gather Cornerstone URLs`), set aggregation mode to `Aggregate All Item Data` with destination field `url_list`. Connect `Note Cornerstone Submission` to it.
   - Create an `HTTP Request` node (`Post Cornerstones to Bing`), set method to `POST`, URL to `https://ssl.bing.com/webmaster/api.svc/json/SubmitUrlBatch?apikey=YOUR_BING_API_KEY`, and enable `onError: continueErrorOutput`. Map JSON body to batch up to 40 URLs. Configure Bing API credentials.
   - Create an `HTTP Request` node (`Post Cornerstones to IndexNow`), set method to `POST`, URL to `https://api.indexnow.org/indexnow`, and enable `onError: continueErrorOutput`. Map JSON body for IndexNow submission.
   - Create an `HTTP Request` node (`Resubmit Sitemap to Google API`), set method to `PUT`, URL to Google Search Console sitemap endpoint, enable `onError: continueErrorOutput`, and configure predefined Google API credentials with the `https://www.googleapis.com/auth/webmasters` scope.
   - Create a `Merge` node (`Combine Cornerstone Submissions`), set mode to `Append` with 3 inputs, and connect the responses from Bing, IndexNow, and Google API nodes.
9. **Configure Content Branch:**
   - Create a `Data Table` node (`Record Content Submission`), set resource to `Row`, operation to `Upsert`, and map columns. Connect Switch Output 1 to this node.
   - Create an `Aggregate` node (`Gather Content URLs`), set destination field to `url_list`. Connect `Record Content Submission` to it.
   - Create an `HTTP Request` node (`Post Content to Bing`), set method to `POST`, URL to Bing batch API with `YOUR_BING_API_KEY`, and enable `onError: continueErrorOutput`.
   - Create an `HTTP Request` node (`Post Content to IndexNow`), set method to `POST`, URL to IndexNow API, and enable `onError: continueErrorOutput`.
   - Create a `Merge` node (`Combine Content Submissions`), set mode to `Append` with 2 inputs, and connect the responses from Bing and IndexNow content nodes.
10. **Configure Reporting Branch:**
    - Create a `Code` node (`Compile Telegram Message`), set mode to `Run Once for All Items`, and paste the digest compilation script utilizing `try/catch` wrappers and node references. Connect `Fetch Sitemap XML`, `Combine Cornerstone Submissions`, and `Combine Content Submissions` to this node.
    - Create a `Telegram` node (`Dispatch Telegram Digest`), set text expression to `{{ $json.digest }}`, and configure your Telegram bot credentials and `YOUR_TELEGRAM_CHAT_ID`. Connect `Compile Telegram Message` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Official n8n Documentation | [n8n Docs](https://docs.n8n.io/) |
| Bing Webmaster API Reference | [Bing Webmaster API Docs](https://www.bing.com/webmasters/help/webmaster-apis-3165bc44) |
| IndexNow Protocol Specification | [IndexNow Documentation](https://www.indexnow.org/) |
| Google Search Console API Reference | [Google Search Console API Docs](https://developers.google.com/webmaster-tools) |