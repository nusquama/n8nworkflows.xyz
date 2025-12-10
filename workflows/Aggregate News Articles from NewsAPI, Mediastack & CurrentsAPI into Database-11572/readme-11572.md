Aggregate News Articles from NewsAPI, Mediastack & CurrentsAPI into Database

https://n8nworkflows.xyz/workflows/aggregate-news-articles-from-newsapi--mediastack---currentsapi-into-database-11572


# Aggregate News Articles from NewsAPI, Mediastack & CurrentsAPI into Database

### 1. Workflow Overview

This workflow automates the aggregation of news articles from three distinct news APIs: NewsAPI.org, Mediastack, and CurrentsAPI. Its primary purpose is to collect categorized news articles on a scheduled basis, normalize their data into a unified format, and store them in a NocoDB database for further use in content pipelines, research, or editorial processes.

The workflow is logically split into the following blocks:

- **1.1 Scheduled Triggers**: Initiate data fetching from each news API on configurable schedules.
- **1.2 NewsAPI.org Top Headlines Processing**: Fetches top headlines, processes, normalizes, and stores them.
- **1.3 NewsAPI.org Category-Based Processing**: Iterates over predefined categories, fetches related articles, normalizes, and stores them.
- **1.4 Mediastack Category-Based Processing**: Iterates over Mediastack categories, fetches corresponding news, normalizes, and stores them.
- **1.5 CurrentsAPI Latest News Processing**: Fetches latest news from CurrentsAPI, normalizes, and stores the data.
- **1.6 Data Normalization and Storage**: Contains Set nodes to normalize API responses into a consistent schema, followed by NocoDB nodes to persist the data.
- **1.7 Utility Nodes**: Includes split nodes to handle batch processing and code nodes to iterate categories.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Triggers

**Overview:**  
These nodes trigger the workflow execution at configured times to fetch news from different APIs automatically.

**Nodes Involved:**  
- MediaStack (Schedule Trigger)  
- CurrentsAPI (Schedule Trigger)  
- NewsAPI - Top Headlines (Schedule Trigger)  
- NewsAPI - Categories (Schedule Trigger)

**Node Details:**

- **MediaStack**  
  - Type: Schedule Trigger  
  - Configuration: Runs every 2 days at 06:05 AM.  
  - Connections: Triggers `mediastack categories` node.  
  - Failure modes: Scheduler misconfiguration or disabled scheduler results in no execution.

- **CurrentsAPI**  
  - Type: Schedule Trigger  
  - Configuration: Runs daily at 06:10 AM.  
  - Connections: Triggers `currentsapi config` node.

- **NewsAPI - Top Headlines**  
  - Type: Schedule Trigger  
  - Configuration: Runs daily at 06:00 AM.  
  - Connections: Triggers `call newsapi.org - Top Headlines`.

- **NewsAPI - Categories**  
  - Type: Schedule Trigger  
  - Configuration: Runs daily at 06:01 AM.  
  - Connections: Triggers `newsapi.org categories`.

---

#### 2.2 NewsAPI.org Top Headlines Processing

**Overview:**  
Fetches top US headlines from NewsAPI.org, normalizes the articles, and stores them in the database.

**Nodes Involved:**  
- call newsapi.org - Top Headlines (HTTP Request)  
- Split Out 1 (Split Out)  
- Set Newsapi.org top headlines (Set)  
- Add NewsAPI item (NocoDB)  

**Node Details:**

- **call newsapi.org - Top Headlines**  
  - Type: HTTP Request  
  - Configuration: GET request to `https://newsapi.org/v2/top-headlines?country=us&apiKey=API_KEY`.  
  - Notes: Must replace `API_KEY` with valid NewsAPI key.  
  - Output: JSON containing `articles` array.  
  - On error: Continues workflow without halting.  
  - Max retries: 2 with 5 sec wait between.  

- **Split Out 1**  
  - Type: Split Out  
  - Purpose: Splits the `articles` array from the previous node into individual items for processing.  
  - Input: Response from `call newsapi.org - Top Headlines`.  
  - Output: Single article items passed to the next node.

- **Set Newsapi.org top headlines**  
  - Type: Set  
  - Configuration: Maps article fields to a normalized schema with keys like `title`, `aggregator` ("newsapi.org"), `category` ("top headlines"), `publisher`, `summary`, `author`, `sources`, `content`, `images`, and `publish_date`.  
  - Input: Individual articles from `Split Out 1`.

- **Add NewsAPI item**  
  - Type: NocoDB  
  - Configuration: Creates new records in NocoDB with the normalized fields, sets `status` to "new".  
  - Credentials: Requires configured NocoDB API token credential.  
  - Input: Normalized data from `Set Newsapi.org top headlines`.

**Edge Cases:**  
- API key invalid or rate limit exceeded results in empty or error response, but workflow continues.  
- Missing article fields handled by default empty values.  
- Network timeouts or HTTP errors retried twice before continuing.

---

#### 2.3 NewsAPI.org Category-Based Processing

**Overview:**  
Iterates over a predefined set of categories, fetches top headlines per category from NewsAPI.org, normalizes, and stores them.

**Nodes Involved:**  
- newsapi.org categories (Set)  
- Itemize Newsapi Categories (Code)  
- Loop Over NewsAPI (Split In Batches)  
- call newsapi.org - categories (HTTP Request)  
- Split Out (Split Out)  
- Set Newsapi.org categories (Set)  
- Add NewsAPI item1 (NocoDB)  
- Wait1 (Wait)

**Node Details:**

- **newsapi.org categories**  
  - Type: Set  
  - Configuration: Defines `article_limit` (20) and array of categories: ["general","business","entertainment","health","science","sports","technology"].  
  - Output: Passes categories array downstream.

- **Itemize Newsapi Categories**  
  - Type: Code  
  - Purpose: Converts categories array into individual JSON items `{ category: cat }` for iteration.  
  - Input: From `newsapi.org categories`.  
  - Error Handling: Throws error if input is not an array.

- **Loop Over NewsAPI**  
  - Type: Split In Batches  
  - Purpose: Processes each category item sequentially.  
  - On completion: Triggers `call newsapi.org - categories`.

- **call newsapi.org - categories**  
  - Type: HTTP Request  
  - Configuration: Calls `https://newsapi.org/v2/top-headlines?category={{ $json.category }}&apiKey=API_KEY`.  
  - Notes: Requires valid API key replacement.  
  - On error: Continues without stopping workflow.  
  - Retries: Max 2 with 5 sec delay.

- **Split Out**  
  - Type: Split Out  
  - Purpose: Splits the `articles` array from API response into individual articles.

- **Set Newsapi.org categories**  
  - Type: Set  
  - Configuration: Maps fields similar to the top headlines block, with `aggregator` set to "newsapi.org" and `category` set dynamically from current category.

- **Add NewsAPI item1**  
  - Type: NocoDB  
  - Purpose: Creates new database records with normalized data and status "new".  
  - Credential: NocoDB API token needed.

- **Wait1**  
  - Type: Wait  
  - Purpose: Introduces delay or synchronization after adding items.  
  - Connections: Loops back to `Loop Over NewsAPI` for next batch.

**Edge Cases:**  
- Invalid categories or empty API response may yield no articles.  
- API rate limits may affect execution; retry logic helps mitigate.  
- If `categories` array is misconfigured or empty, the code node throws an error to avoid silent failures.

---

#### 2.4 Mediastack Category-Based Processing

**Overview:**  
Fetches news articles from Mediastack API for predefined categories, normalizes, and stores them in the database.

**Nodes Involved:**  
- mediastack categories (Set)  
- Itemize Mediastack Categories (Code)  
- Loop Over Mediastack (Split In Batches)  
- call mediastack (HTTP Request)  
- Split Out 2 (Split Out)  
- Set Mediastack (Set)  
- Add Mediastack item (NocoDB)  
- Wait (Wait)

**Node Details:**

- **mediastack categories**  
  - Type: Set  
  - Configuration: Sets `article_limit` (15) and array of categories: ["general","business","entertainment","health","science","sports","technology"].  
  - Output: Categories array for iteration.

- **Itemize Mediastack Categories**  
  - Type: Code  
  - Function: Converts the categories array into individual JSON items to iterate.  
  - Error Handling: Throws error if input is not an array.

- **Loop Over Mediastack**  
  - Type: Split In Batches  
  - Processes each category sequentially.

- **call mediastack**  
  - Type: HTTP Request  
  - Configuration: GET request to `http://api.mediastack.com/v1/news` with JSON query parameters including `access_key` (to be updated), `categories`, `languages` ("en"), `sort` ("published_desc"), and `limit` (dynamic from `mediastack categories`).  
  - Notes: On error, continues workflow.  
  - Retries: Max 2 with 5 sec delay.

- **Split Out 2**  
  - Type: Split Out  
  - Splits `data` array from Mediastack response into individual articles.

- **Set Mediastack**  
  - Type: Set  
  - Maps Mediastack article fields to normalized keys: `title`, `aggregator` ("mediastack"), `category`, `publisher`, `summary`, `author`, `sources`, `content`, `images`, `publish_date`.

- **Add Mediastack item**  
  - Type: NocoDB  
  - Creates new records with normalized data, status "new".  
  - Requires NocoDB API token.

- **Wait**  
  - Type: Wait  
  - Used to pace the workflow execution after DB inserts.

**Edge Cases:**  
- API key missing or invalid results in no data; node continues without halting workflow.  
- JSON query must be updated with a valid access key before running.  
- Article fields missing or malformed handled gracefully by Set node.

---

#### 2.5 CurrentsAPI Latest News Processing

**Overview:**  
Fetches latest news from CurrentsAPI, normalizes the response, and inserts it into the database.

**Nodes Involved:**  
- currentsapi config (Set)  
- call currentsapi (HTTP Request)  
- Split Out 3 (Split Out)  
- Set CurrentsAPI (Set)  
- Add CurrentsAPI item (NocoDB)

**Node Details:**

- **currentsapi config**  
  - Type: Set  
  - Configuration: Sets `article_limit` (15) and `category` ("latest-news").  
  - Output: Configuration for API call.

- **call currentsapi**  
  - Type: HTTP Request  
  - Configuration: GET request to `https://api.currentsapi.services/v1/latest-news` with query parameters: `apiKey` (to be updated), `language` ("en"), and `limit` (from `article_limit`).  
  - On error: Continues workflow.  
  - No retries configured.

- **Split Out 3**  
  - Type: Split Out  
  - Splits `news` array from response into individual articles.

- **Set CurrentsAPI**  
  - Type: Set  
  - Maps article fields (`title`, `aggregator` set to "currentsapi", `category`, `publisher`, `summary`, `author`, `sources`, `content`, `images`, `publish_date`).  
  - Note: Combines first two elements of `category` array for category field.

- **Add CurrentsAPI item**  
  - Type: NocoDB  
  - Inserts normalized article into database with status "new".  
  - Uses NocoDB API token credential.

**Edge Cases:**  
- API key must be replaced before running.  
- If API returns empty or malformed data, node continues without error.  
- If `category` field is empty or has fewer than two elements, string concatenation may result in partial or malformed category.

---

#### 2.6 Utility and Supporting Nodes

**Overview:**  
Nodes responsible for splitting arrays, iterating batches, and providing configuration data.

**Nodes Involved:**  
- Split Out, Split Out 1, Split Out 2, Split Out 3 (Split Out)  
- Loop Over NewsAPI, Loop Over Mediastack (Split In Batches)  
- Itemize Newsapi Categories, Itemize Mediastack Categories (Code)  
- mediastack categories, newsapi.org categories, currentsapi config (Set)  

**Node Details:**

- **Split Out Nodes**  
  - Purpose: Take an array field from JSON and output each element as a separate item for further processing.

- **Loop Over Nodes**  
  - Purpose: Process a batch of items one at a time, useful for rate limiting or API constraints.

- **Itemize Categories (Code)**  
  - Purpose: Converts an array of categories into individual JSON objects for iteration.  
  - Throws errors if input is not an array (helps detect misconfiguration).

- **Category Config Set Nodes**  
  - Purpose: Define categories and article limits as parameters for API calls in a centralized manner.

---

#### 2.7 Sticky Notes

**Overview:**  
Provide documentation, instructions, API usage limits, and configuration tips directly inside the workflow for user reference.

**Nodes Involved:**  
- Sticky Note (main project overview)  
- Sticky Note7 (Mediastack info)  
- Sticky Note8 (NewsAPI info)  
- Sticky Note10 (CurrentsAPI info)  
- Sticky Note12 (NewsAPI Categories)

**Contents Highlights:**  
- API limits and plans for each news source.  
- Required updates before running (API keys, DB credentials).  
- Scheduling instructions (all schedulers disabled by default).  
- Category lists used for API queries.  
- Advice for replacing NocoDB with other DBs if desired.

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                              | Input Node(s)                    | Output Node(s)                        | Sticky Note                                                              |
|--------------------------------|-----------------------|----------------------------------------------|---------------------------------|-------------------------------------|---------------------------------------------------------------------------|
| MediaStack                     | Schedule Trigger      | Triggers Mediastack news aggregation          | None                            | mediastack categories               |                                                                           |
| mediastack categories          | Set                   | Defines Mediastack categories & limits        | MediaStack                     | Itemize Mediastack Categories       |                                                                           |
| Itemize Mediastack Categories  | Code                  | Converts category array to individual items   | mediastack categories          | Loop Over Mediastack                |                                                                           |
| Loop Over Mediastack           | Split In Batches      | Processes Mediastack categories sequentially | Itemize Mediastack Categories  | call mediastack                    |                                                                           |
| call mediastack                | HTTP Request          | Fetches news for Mediastack category           | Loop Over Mediastack           | Split Out 2                       | "UPDATE ACCESS_KEY" note present                                          |
| Split Out 2                   | Split Out             | Splits Mediastack response into articles      | call mediastack                | Set Mediastack                    |                                                                           |
| Set Mediastack                | Set                   | Normalizes Mediastack article fields           | Split Out 2                   | Add Mediastack item               |                                                                           |
| Add Mediastack item            | NocoDB                | Inserts Mediastack articles into DB            | Set Mediastack                | Wait                            |                                                                           |
| Wait                         | Wait                  | Synchronizes pacing after DB insert             | Add Mediastack item            | Loop Over Mediastack              |                                                                           |
| CurrentsAPI                   | Schedule Trigger      | Triggers CurrentsAPI news aggregation           | None                          | currentsapi config               |                                                                           |
| currentsapi config            | Set                   | Sets CurrentsAPI limits and category            | CurrentsAPI                   | call currentsapi                |                                                                           |
| call currentsapi             | HTTP Request          | Fetches latest news from CurrentsAPI             | currentsapi config            | Split Out 3                    | "ADD API KEY" note present                                               |
| Split Out 3                  | Split Out             | Splits CurrentsAPI news array                    | call currentsapi              | Set CurrentsAPI                |                                                                           |
| Set CurrentsAPI              | Set                   | Normalizes CurrentsAPI article fields            | Split Out 3                  | Add CurrentsAPI item           |                                                                           |
| Add CurrentsAPI item          | NocoDB                | Inserts CurrentsAPI articles into DB              | Set CurrentsAPI              | None                          |                                                                           |
| NewsAPI - Top Headlines       | Schedule Trigger      | Triggers NewsAPI top headlines fetch             | None                          | call newsapi.org - Top Headlines |                                                                           |
| call newsapi.org - Top Headlines | HTTP Request          | Fetches US top headlines from NewsAPI             | NewsAPI - Top Headlines       | Split Out 1                   | "ADD API KEY TO URL" note present                                        |
| Split Out 1                  | Split Out             | Splits NewsAPI top headlines into articles        | call newsapi.org - Top Headlines | Set Newsapi.org top headlines |                                                                           |
| Set Newsapi.org top headlines | Set                   | Normalizes NewsAPI top headlines article fields   | Split Out 1                  | Add NewsAPI item              |                                                                           |
| Add NewsAPI item              | NocoDB                | Inserts NewsAPI top headlines into DB              | Set Newsapi.org top headlines | None                          |                                                                           |
| NewsAPI - Categories          | Schedule Trigger      | Triggers NewsAPI category-based fetch              | None                          | newsapi.org categories        |                                                                           |
| newsapi.org categories        | Set                   | Defines NewsAPI categories & article limit          | NewsAPI - Categories          | Itemize Newsapi Categories    |                                                                           |
| Itemize Newsapi Categories    | Code                  | Converts NewsAPI categories array to items          | newsapi.org categories        | Loop Over NewsAPI             |                                                                           |
| Loop Over NewsAPI             | Split In Batches      | Processes NewsAPI categories sequentially            | Itemize Newsapi Categories    | call newsapi.org - categories |                                                                           |
| call newsapi.org - categories | HTTP Request          | Fetches NewsAPI articles by category                  | Loop Over NewsAPI             | Split Out                    | "ADD API KEY TO URL" note present                                        |
| Split Out                    | Split Out             | Splits NewsAPI category response into articles       | call newsapi.org - categories | Set Newsapi.org categories    |                                                                           |
| Set Newsapi.org categories    | Set                   | Normalizes NewsAPI category article fields             | Split Out                    | Add NewsAPI item1            |                                                                           |
| Add NewsAPI item1             | NocoDB                | Inserts NewsAPI category articles into DB              | Set Newsapi.org categories    | Wait1                       |                                                                           |
| Wait1                        | Wait                  | Synchronizes pacing after DB insert                     | Add NewsAPI item1             | Loop Over NewsAPI            |                                                                           |
| Sticky Note                   | Sticky Note           | Main workflow overview and instructions               | None                          | None                        | Contains detailed setup and usage instructions                          |
| Sticky Note7                  | Sticky Note           | Mediastack API info and categories                       | None                          | None                        | Contains Mediastack API limits and categories link                     |
| Sticky Note8                  | Sticky Note           | NewsAPI.org Top Headlines info and usage                  | None                          | None                        | Contains NewsAPI limits and link                                      |
| Sticky Note10                 | Sticky Note           | CurrentsAPI info and usage                                 | None                          | None                        | Contains CurrentsAPI limits and link                                  |
| Sticky Note12                 | Sticky Note           | NewsAPI.org categories detail                              | None                          | None                        | Lists categories and usage details for NewsAPI                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Nodes:**  
   - Create four Schedule Trigger nodes named:  
     - "MediaStack": Set interval to every 2 days at 06:05 AM.  
     - "CurrentsAPI": Set daily trigger at 06:10 AM.  
     - "NewsAPI - Top Headlines": Set daily trigger at 06:00 AM.  
     - "NewsAPI - Categories": Set daily trigger at 06:01 AM.

2. **Create Category Configurations:**  
   - Create "mediastack categories" Set node:  
     - Assign `article_limit` = 15 (number).  
     - Assign `categories` = ["general","business","entertainment","health","science","sports","technology"] (array).
   - Create "newsapi.org categories" Set node:  
     - Assign `article_limit` = 20 (number).  
     - Assign `categories` = ["general","business","entertainment","health","science","sports","technology"] (array).
   - Create "currentsapi config" Set node:  
     - Assign `article_limit` = 15 (number).  
     - Assign `category` = "latest-news" (string).

3. **Create Code Nodes for Category Itemization:**  
   - Create "Itemize Mediastack Categories" Code node with JS:  
     ```javascript
     const categories = $json.categories;
     if (!Array.isArray(categories)) throw new Error("Expected categories array");
     return categories.map(cat => ({ json: { category: cat } }));
     ```
   - Create "Itemize Newsapi Categories" Code node with the same JS code (adjust variable names if needed).

4. **Setup Mediastack News Flow:**  
   - Connect "MediaStack" → "mediastack categories" → "Itemize Mediastack Categories" → "Loop Over Mediastack" (Split In Batches).  
   - Create "call mediastack" HTTP Request node:  
     - Method: GET  
     - URL: `http://api.mediastack.com/v1/news`  
     - Query Parameters (JSON):  
       ```json
       {
         "access_key": "ACCESS_KEY",
         "categories": "{{ $json.category }}",
         "languages": "en",
         "sort": "published_desc",
         "limit": "{{ $('mediastack categories').item.json.article_limit }}"
       }
       ```  
     - Replace `ACCESS_KEY` with your actual Mediastack API key.  
     - Set "Retry On Fail" enabled, max 2 tries, 5 seconds between tries.  
     - On error: Continue workflow.  
   - Connect "Loop Over Mediastack" → "call mediastack" → "Split Out 2" (field: `data`).  
   - Create "Set Mediastack" node to normalize fields:  
     - Map fields like `title`, `aggregator`="mediastack", `category`, `publisher`, etc. using expressions referencing `$json.data`.  
   - Connect "Split Out 2" → "Set Mediastack" → "Add Mediastack item".  
   - Create "Add Mediastack item" NocoDB node:  
     - Operation: Create  
     - Map all normalized fields accordingly.  
     - Use your NocoDB API token credentials.  
   - Connect "Add Mediastack item" → "Wait" (empty Wait node) → back to "Loop Over Mediastack" for pacing.

5. **Setup CurrentsAPI News Flow:**  
   - Connect "CurrentsAPI" → "currentsapi config" → "call currentsapi".  
   - Create "call currentsapi" HTTP Request node:  
     - Method: GET  
     - URL: `https://api.currentsapi.services/v1/latest-news`  
     - Query Parameters:  
       - apiKey = "API_KEY" (replace with your CurrentsAPI key)  
       - language = "en"  
       - limit = `{{$json.article_limit}}`  
     - On error: Continue workflow.  
   - Connect "call currentsapi" → "Split Out 3" (field: `news`).  
   - Create "Set CurrentsAPI" node to normalize fields:  
     - Map fields using `$json` with keys like `title`, `aggregator` = "currentsapi", `category` (combine first two categories if array), `publisher`, etc.  
   - Connect "Split Out 3" → "Set CurrentsAPI" → "Add CurrentsAPI item".  
   - Create "Add CurrentsAPI item" NocoDB node as above.

6. **Setup NewsAPI.org Top Headlines Flow:**  
   - Connect "NewsAPI - Top Headlines" → "call newsapi.org - Top Headlines".  
   - Create "call newsapi.org - Top Headlines" HTTP Request node:  
     - Method: GET  
     - URL: `https://newsapi.org/v2/top-headlines?country=us&apiKey=API_KEY` (replace `API_KEY`).  
     - Retry and error handling same as Mediastack.  
   - Connect to "Split Out 1" (field: `articles`).  
   - Create "Set Newsapi.org top headlines" node:  
     - Normalize fields similarly to others, set `aggregator`="newsapi.org", `category`="top headlines".  
   - Connect "Split Out 1" → "Set Newsapi.org top headlines" → "Add NewsAPI item".  
   - Create "Add NewsAPI item" NocoDB node.

7. **Setup NewsAPI.org Category-Based Flow:**  
   - Connect "NewsAPI - Categories" → "newsapi.org categories" → "Itemize Newsapi Categories" → "Loop Over NewsAPI" (Split In Batches).  
   - Create "call newsapi.org - categories" HTTP Request node:  
     - Method: GET  
     - URL: `https://newsapi.org/v2/top-headlines?category={{ $json.category }}&apiKey=API_KEY` (replace API_KEY).  
     - Retry and error handling configured like above.  
   - Connect "Loop Over NewsAPI" → "call newsapi.org - categories" → "Split Out" (field: `articles`).  
   - Create "Set Newsapi.org categories" node:  
     - Normalize fields with dynamic category.  
   - Connect "Split Out" → "Set Newsapi.org categories" → "Add NewsAPI item1".  
   - Create "Add NewsAPI item1" NocoDB node.  
   - Connect "Add NewsAPI item1" → "Wait1" wait node → back to "Loop Over NewsAPI".

8. **Create Sticky Notes:**  
   - Add Sticky Notes in appropriate places containing:  
     - API limits and plans for each provider.  
     - Instructions for API key replacement, DB credentials, scheduling notes.  
     - Category lists and usage hints.

9. **Credentials Setup:**  
   - Configure NocoDB API token credential with your token.  
   - Replace all placeholder API keys in HTTP Request nodes with your valid keys.

10. **Enable Schedulers:**  
    - By default, schedulers are disabled; enable all four to activate automatic runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow pulls news articles from NewsAPI, Mediastack, and CurrentsAPI on a scheduled basis. Each provider’s results are normalized into a consistent schema, then written into your database (NocoDB by default). Use case: automated aggregation of categorized news for content pipelines, research agents, or editorial queues.                            | Main Sticky Note in workflow                                                                         |
| Before running, update all API keys in HTTP Request nodes and configure your NocoDB API token credential. Ensure your database table has the required fields (source_category, title, summary, author, sources, content, images, publisher_date, etc.).                                                                                                      | Main Sticky Note                                                                                     |
| Scheduling nodes are disabled by default. Enable all four schedulers to allow automatic execution. You may adjust run times but all must be enabled for full functionality.                                                                                                                                                                                | Main Sticky Note                                                                                     |
| Mediastack Free Plan allows 100 calls/month. Categories supported: General, Business, Entertainment, Health, Science, Sports, Technology.                                                                                                                                                                                                                   | Sticky Note7 [Mediastack](https://www.mediastack.com)                                               |
| NewsAPI Free Plan allows 100 requests/day. Categories: business, entertainment, general, health, science, sports, technology. Returns ~20 top headlines per request.                                                                                                                                                                                         | Sticky Note8 [NewsAPI.org](https://newsapi.org/)                                                    |
| CurrentsAPI Free Plan allows 20 requests/day. Supports latest news.                                                                                                                                                                                                                                                                                          | Sticky Note10 [CurrentsAPI](https://currentsapi.services/en)                                       |
| NewsAPI.org categories: business, entertainment, general, health, science, sports, technology.                                                                                                                                                                                                                                                              | Sticky Note12                                                                                       |
| To adapt to other databases (Google Sheets, Airtable, etc.), replace NocoDB nodes with equivalent “create row” or insert operations. Set nodes already provide normalized fields ready for use.                                                                                                                                                            | Main Sticky Note                                                                                     |
| API rate limits may cause partial data retrieval; built-in retry logic helps but monitor your API usage quotas.                                                                                                                                                                                                                                              | Implied from retry settings and error handling in HTTP nodes                                        |
| The workflow uses batch splitting to avoid overwhelming APIs and to pace database inserts via Wait nodes.                                                                                                                                                                                                                                                  | Observed from Split In Batches and Wait nodes                                                      |

---

**Disclaimer:** The provided text and workflow are generated exclusively from an automated n8n workflow export. All data handled respects current content policies and contains no illegal, offensive, or protected content. All processed data are legal and public.