Track TikTok Ads Library with Apify, Google Sheets & Slack/Telegram Notifications

https://n8nworkflows.xyz/workflows/track-tiktok-ads-library-with-apify--google-sheets---slack-telegram-notifications-11615


# Track TikTok Ads Library with Apify, Google Sheets & Slack/Telegram Notifications

### 1. Workflow Overview

This workflow automates the monitoring of TikTok Ads Library data using Apify’s TikTok Ads Scraper, recording new ad creatives into Google Sheets, and sending notifications through Slack and Telegram when new ads are detected. It is designed to help marketers, competitive analysts, and social media managers track advertiser activity and creative trends over time.

The workflow is logically divided into the following functional blocks:

- **1.1 Parameter Initialization:** Set user-defined parameters such as target country, date range, advertiser keywords, business IDs, and ad limits.
- **1.2 Date Conversion:** Convert user-friendly date strings (DD/MM/YYYY) into Unix timestamps required by API calls.
- **1.3 Apify Request Preparation:** Build the JSON request body to run the Apify TikTok Ads Scraper actor with dynamic filtering.
- **1.4 Data Retrieval from Apify:** Invoke the Apify actor to scrape TikTok ads data based on parameters.
- **1.5 Data Preparation for Google Sheets:** Extract and safely prepare required fields (e.g., video URLs, usernames) for storage.
- **1.6 Existing Data Fetching:** Read existing ad IDs from Google Sheets to avoid duplicates.
- **1.7 Data Matching and Filtering:** Compare scraped ads with stored IDs to isolate new ads.
- **1.8 Data Appending and Counting:** Append new ads to Google Sheets and count new entries.
- **1.9 Notification Dispatch:** Send notifications with the count of new ads to Slack and Telegram channels.

---

### 2. Block-by-Block Analysis

#### 2.1 Parameter Initialization

- **Overview:** This block sets up all input parameters for the workflow, using default values and expressions to define date ranges and targeting filters.
- **Nodes Involved:** Set Parameters
- **Node Details:**

  - **Set Parameters**
    - Type: Set node (data assignment)
    - Configuration:
      - Defines parameters:
        - Ad target country (default "all")
        - Ad published date From (default: yesterday’s date, dynamically calculated)
        - Ad published date To (default: today’s date)
        - Advertiser name or keyword (empty by default)
        - adv_biz_ids (advertiser business IDs, default example value provided)
        - Ad limit (empty by default)
    - Expressions used to compute date defaults with current date minus one day and current date.
    - Input: none (triggered by Schedule Trigger)
    - Output: passes parameters as JSON forward
    - Potential failures: None expected unless expression evaluation fails (rare)
    - Notes: Sticky Note explains parameter setup, including ISO country codes and usage hints.

#### 2.2 Date Conversion

- **Overview:** Converts human-readable date strings into Unix timestamp format (milliseconds) required by the TikTok Ads API.
- **Nodes Involved:** Convert Dates to Unix
- **Node Details:**

  - **Convert Dates to Unix**
    - Type: Code node (JavaScript)
    - Configuration:
      - Parses date strings in DD/MM/YYYY format
      - Converts to JS Date objects and returns Unix timestamps
      - Returns a JSON object with original parameters plus start_time_unix and end_time_unix
    - Input: parameters JSON from Set Parameters
    - Output: enriched JSON including Unix timestamps
    - Edge cases:
      - Invalid or empty date strings throw an error
      - Malformed date format causes explicit error
    - Sticky Note: explains node purpose (date format transformation)

#### 2.3 Apify Request Preparation

- **Overview:** Constructs the JSON body required to run the Apify TikTok Ads Scraper actor, applying filters and optional limits dynamically.
- **Nodes Involved:** Build Apify Body
- **Node Details:**

  - **Build Apify Body**
    - Type: Code node (JavaScript)
    - Configuration:
      - Reads input parameters, including adv_biz_ids and ad limits
      - Constructs a URL with query parameters for region, start/end times, advertiser keywords, and business IDs
      - Builds a request body object with skipDetails=false and startUrls containing the constructed URL
      - Adds resultsLimit if specified and valid
      - Returns JSON containing original input plus a stringified customBody for Apify
    - Input: JSON with Unix timestamps and other params
    - Output: JSON with field `customBody` containing Apify actor input
    - Edge cases:
      - Empty or invalid limit ignored
      - Empty adv_biz_ids still included as empty parameter
    - Sticky Note: explains JSON body creation for Apify with result limit handling

#### 2.4 Data Retrieval from Apify

- **Overview:** Runs the Apify TikTok Ads Scraper actor with the prepared request body and retrieves the dataset of TikTok ad creatives.
- **Nodes Involved:** Get TT Ads through Apify
- **Node Details:**

  - **Get TT Ads through Apify**
    - Type: Apify node (custom integration)
    - Configuration:
      - Actor selected: silva95gustavo/tiktok-ads-scraper (Apify actor ID)
      - Operation: Run actor and get dataset
      - Uses `customBody` from previous node as input
      - Credentials: Apify API key required
    - Input: JSON with customBody string
    - Output: Array of ad creative objects from Apify dataset
    - Potential failures:
      - API authentication errors
      - Actor execution failure or timeout
      - Network errors
    - Sticky Note: describes node responsibility and credential setup

#### 2.5 Data Preparation for Google Sheets

- **Overview:** Extracts relevant fields from the Apify output for easier processing and Google Sheets insertion, handling missing or malformed data safely.
- **Nodes Involved:** Prepare Data for Sheets
- **Node Details:**

  - **Prepare Data for Sheets**
    - Type: Code node (JavaScript)
    - Configuration:
      - Iterates over all incoming items (ads)
      - Extracts first video URL, cover image URL, and TikTok username safely with fallback to empty strings
      - Returns enriched items including these fields for further use
    - Input: array of ads JSON from Apify node
    - Output: enriched array ready for Google Sheets
    - Edge cases:
      - Missing videos array or empty videos handled gracefully
      - Missing TikTok username handled gracefully
    - Sticky Note: explains safe extraction and preparation for Sheets

#### 2.6 Existing Data Fetching

- **Overview:** Reads the existing ad IDs stored in Google Sheets to identify which ads are already recorded.
- **Nodes Involved:** Read existing IDs, Collect ID list
- **Node Details:**

  - **Read existing IDs**
    - Type: Google Sheets node (read operation)
    - Configuration:
      - Reads column K (range K:K) from Sheet1 of a specific Google Sheets document
      - Credentials: Google Sheets OAuth2
    - Input: prepared ads data (from previous node)
    - Output: list of existing ad IDs in sheet column
    - Potential failures:
      - Authentication/permission errors with Google Sheets
      - Invalid document ID or sheet name
    - Sticky Note: explains checking existing IDs to avoid duplicates

  - **Collect ID list**
    - Type: Code node (JavaScript)
    - Configuration:
      - Extracts all ad IDs from the Google Sheets output
      - Converts IDs to strings to avoid type mismatches
      - Deduplicates IDs
      - Returns a single JSON item containing `existingIds` array
    - Input: items from Google Sheets read
    - Output: JSON with existingIds list
    - Edge cases:
      - Null or missing IDs filtered out
    - Sticky Note: explains ID collection for matching

#### 2.7 Data Matching and Filtering

- **Overview:** Merges new scraped ads data with existing IDs and filters out ads already present in Google Sheets, isolating truly new ads.
- **Nodes Involved:** Attach existing ids, Filter new creatives
- **Node Details:**

  - **Attach existing ids**
    - Type: Merge node (combine mode)
    - Configuration:
      - Combines the existing IDs object with the list of scraped ads
      - Uses “combine all” to join datasets for filtering
    - Input: existingIds from Collect ID list and prepared ads from Prepare Data for Sheets
    - Output: combined data for filtering
    - Sticky Note: explains matching creatives with Google Sheets entries

  - **Filter new creatives**
    - Type: Code node (JavaScript)
    - Configuration:
      - Extracts existing IDs
      - Iterates over scraped ads to filter only those whose adId is not in existing IDs
      - Ensures duplicates within current run are excluded
      - Returns array of new ads as separate items
    - Input: combined data
    - Output: filtered new ads only
    - Edge cases:
      - Ads without adId are ignored
      - Ad IDs cast to string to avoid mismatch
    - Sticky Note: explains filtering new creatives

#### 2.8 Data Appending and Counting

- **Overview:** Appends or updates new ads in Google Sheets, then counts the number of new ads found.
- **Nodes Involved:** Append or update row in sheet, Count new ads
- **Node Details:**

  - **Append or update row in sheet**
    - Type: Google Sheets node (appendOrUpdate operation)
    - Configuration:
      - Document and sheet specified (same as reading node)
      - Columns mapped explicitly with formulas for cover image (Excel IMAGE formula)
      - Matching column: adId to avoid duplicates
      - Appends new ads or updates existing matching rows
      - Credentials: Google Sheets OAuth2
    - Input: new ads filtered from previous node
    - Output: operation result
    - Edge cases:
      - Column order and names must stay consistent to avoid data corruption
      - Google Sheets API rate limits or quota errors possible
    - Sticky Note: warns about keeping column order and names in sync

  - **Count new ads**
    - Type: Code node (JavaScript)
    - Configuration:
      - Counts number of input items (new ads)
      - Returns JSON with newCount field
    - Input: new ads from filter node
    - Output: JSON with count of new ads
    - Sticky Note: explains counting for notification message

#### 2.9 Notification Dispatch

- **Overview:** Conditional block to send notifications via Slack and Telegram only if new ads were found.
- **Nodes Involved:** Any new ads?, Send a message (Slack), Send a text message (Telegram)
- **Node Details:**

  - **Any new ads?**
    - Type: If node
    - Configuration:
      - Condition: JSON field newCount > 0 (strict number comparison)
    - Input: count from previous node
    - Output: yes/no branch
    - Sticky Note: explains condition check before notification

  - **Send a message (Slack)**
    - Type: Slack node
    - Configuration:
      - Sends a message to a specified Slack channel (channelId can be set)
      - Message text includes number of new ads and link to Google Sheets
      - Webhook used for authentication
    - Input: newCount from count node (only if newCount > 0)
    - Output: Slack API response
    - Potential failures:
      - Invalid webhook or channel ID
      - Slack API rate limits
    - Sticky Note: explains Slack notification setup

  - **Send a text message (Telegram)**
    - Type: Telegram node
    - Configuration:
      - Sends a text message to a specified chat ID
      - Message text includes newCount and link to Google Sheets
      - Telegram API credentials required
    - Input: newCount from count node (only if newCount > 0)
    - Output: Telegram API response
    - Potential failures:
      - Invalid chat ID or token
      - Telegram rate limits
    - Sticky Note: explains Telegram notification setup

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                              | Input Node(s)            | Output Node(s)                      | Sticky Note                                                                                                 |
|---------------------------|-------------------------|----------------------------------------------|--------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger        | Starts workflow on schedule                    | None                     | Set Parameters                    |                                                                                                             |
| Set Parameters            | Set                     | Define parameters for scraping                 | Schedule Trigger          | Convert Dates to Unix              | ## Set your parameters...                                                                                   |
| Convert Dates to Unix     | Code                    | Convert dates from DD/MM/YYYY to Unix          | Set Parameters            | Build Apify Body                  | Transforms date format from DD/MM/YYYY to Unix timestamp                                                    |
| Build Apify Body          | Code                    | Build JSON request body for Apify actor        | Convert Dates to Unix     | Get TT Ads through Apify          | Creates JSON body for Apify if you set a result limit                                                       |
| Get TT Ads through Apify  | Apify                   | Run Apify actor and fetch TikTok ads data      | Build Apify Body          | Prepare Data for Sheets           | This node gets data from the Apify scraper. Add your credentials and choose an actor                         |
| Prepare Data for Sheets   | Code                    | Extract and prepare data for Google Sheets     | Get TT Ads through Apify  | Read existing IDs, Attach existing ids | Safely extract video data and prepare for Google Sheets                                                     |
| Read existing IDs         | Google Sheets           | Read existing ad IDs from Google Sheets        | Prepare Data for Sheets   | Collect ID list                   | Check which creative IDs already exist in Google Sheets so we only send a notification when new ones are found |
| Collect ID list           | Code                    | Collect and deduplicate existing IDs            | Read existing IDs         | Attach existing ids               | Collect IDs for future matching                                                                             |
| Attach existing ids       | Merge                   | Combine existing IDs with new ads data          | Collect ID list, Prepare Data for Sheets | Filter new creatives           | Match creatives from the response with those already stored in Google Sheets                                |
| Filter new creatives      | Code                    | Filter only new ads not in existing IDs         | Attach existing ids       | Append or update row in sheet, Count new ads | Match creatives from the API response with those already stored in Google Sheets                           |
| Append or update row in sheet | Google Sheets       | Append or update new ads in Google Sheets       | Filter new creatives      |                               | Add creatives to Google Sheets. Be careful to keep the column order and names in sync.                      |
| Count new ads             | Code                    | Count how many new ads were found                | Filter new creatives      | Any new ads?                     | Count how many new creatives were found for a more compact message. Use {{$json.newCount}} in the Slack or Telegram message. |
| Any new ads?              | If                      | Check if there are new ads to notify             | Count new ads             | Send a text message, Send a message | Check if there are any new ads before sending notifications                                                |
| Send a message            | Slack                   | Send Slack notification about new creatives     | Any new ads?              |                               | Send Slack notification about new creatives                                                                |
| Send a text message       | Telegram                | Send Telegram notification about new creatives  | Any new ads?              |                               | Send Telegram notification about new creatives                                                             |
| Sticky Notes (various)    | Sticky Note             | Documentation and explanations                    | N/A                      | N/A                             | Various explanatory notes on workflow sections and nodes                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set it to run at desired intervals (e.g., daily).

2. **Add a Set node named "Set Parameters"**  
   - Create parameters:  
     - Ad target country (string, default "all")  
     - Ad published date From (string, default: `={{ $now.minus({ days: 1 }).toFormat('dd/MM/yyyy') }}`)  
     - Ad published date To (string, default: `={{ $now.toFormat('dd/MM/yyyy') }}`)  
     - Advertiser name or keyword (string, default empty)  
     - adv_biz_ids (string, default "6920170086569345793" or empty)  
     - Ad limit (string, default empty)  
   - Connect Schedule Trigger → Set Parameters

3. **Add a Code node named "Convert Dates to Unix"**  
   - Use JavaScript to parse DD/MM/YYYY dates from input parameters and convert to Unix timestamps (milliseconds).  
   - Return original inputs plus `start_time_unix` and `end_time_unix`.  
   - Connect Set Parameters → Convert Dates to Unix

4. **Add a Code node named "Build Apify Body"**  
   - Construct a JSON body with `startUrls` containing a URL built from input parameters, including conditional adv_biz_ids and resultsLimit.  
   - Stringify the body and add as `customBody` field.  
   - Connect Convert Dates to Unix → Build Apify Body

5. **Add an Apify node named "Get TT Ads through Apify"**  
   - Select the TikTok Ads Scraper actor (ID: AovGexsTSmlalAFzp)  
   - Operation: Run actor and get dataset  
   - Use `customBody` from previous node as input  
   - Add Apify API credentials  
   - Connect Build Apify Body → Get TT Ads through Apify

6. **Add a Code node named "Prepare Data for Sheets"**  
   - Extract first video URL, cover image URL, and TikTok username safely from each ad item  
   - Return enriched items  
   - Connect Get TT Ads through Apify → Prepare Data for Sheets

7. **Add a Google Sheets node named "Read existing IDs"**  
   - Operation: Read data  
   - Spreadsheet ID: your TikTok Ads library research spreadsheet  
   - Sheet name: Sheet1 (gid=0)  
   - Range: column K (K:K) to read existing ad IDs  
   - Add Google Sheets OAuth2 credentials  
   - Connect Prepare Data for Sheets → Read existing IDs

8. **Add a Code node named "Collect ID list"**  
   - Extract ad IDs from Google Sheets output, cast to strings, remove duplicates  
   - Return as `existingIds` array inside one JSON object  
   - Connect Read existing IDs → Collect ID list

9. **Add a Merge node named "Attach existing ids"**  
   - Mode: Combine  
   - Combine By: Combine All  
   - Connect Collect ID list (to input 1) and Prepare Data for Sheets (input 2) → Attach existing ids

10. **Add a Code node named "Filter new creatives"**  
    - Use existingIds and scraped ads to filter only new ads not present in existingIds  
    - Return new ads as separate items  
    - Connect Attach existing ids → Filter new creatives

11. **Add a Google Sheets node named "Append or update row in sheet"**  
    - Operation: Append or Update  
    - Spreadsheet ID and Sheet as before  
    - Map columns explicitly: adId, adName, paidBy, videos (videoUrl), startURL, targeting, tiktokUser (tiktokUsername), Impressions, regionStats, advertiserID, coverImageURL (with IMAGE formula), advertiserName  
    - Match rows by adId  
    - Connect Filter new creatives → Append or update row in sheet

12. **Add a Code node named "Count new ads"**  
    - Count input items (new ads) and output JSON with `newCount` property  
    - Connect Filter new creatives → Count new ads

13. **Add an If node named "Any new ads?"**  
    - Condition: `{{$json.newCount}} > 0`  
    - Connect Count new ads → Any new ads?

14. **Add a Slack node named "Send a message"**  
    - Use Slack webhook or credentials  
    - Message text: "Hello! {{$json.newCount}} of TikTok ads were found today! You can see full list here — [link to Google Sheets]"  
    - Connect Any new ads? (true) → Send a message

15. **Add a Telegram node named "Send a text message"**  
    - Configure with Telegram API credentials and chat ID  
    - Message text same as Slack message  
    - Connect Any new ads? (true) → Send a text message

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow is built by Kirill Khatkevich. Connect on LinkedIn: https://www.linkedin.com/in/kirill-khatkevich/                                              | Author credit                                                                                   |
| Apify TikTok Ads Scraper actor used: https://console.apify.com/actors/AovGexsTSmlalAFzp/input                                                                    | Apify actor documentation and setup guide                                                      |
| Google Sheets document used for storing ads data: https://docs.google.com/spreadsheets/d/15mF_dwFrTTAaEALIo0xpRHc3_eKUKBFucmBv5lZTzUA/edit?usp=sharing         | Spreadsheet where ads are recorded                                                             |
| Instructions for setting parameters include use of ISO 3166 country codes for "Ad target country"                                                              | Parameter setup detail                                                                          |
| Slack and Telegram notifications include a direct link to the Google Sheets document for quick access                                                         | Notification message content                                                                    |
| Keep Google Sheets column order and header names consistent to avoid data corruption                                                                           | Important maintenance note                                                                     |
| The workflow ensures no duplicate ads are notified or added to Google Sheets by matching adId fields                                                           | Duplicate prevention strategy                                                                  |
| Date format expected is DD/MM/YYYY for input parameters, with conversion to Unix timestamps for API calls                                                     | Date handling detail                                                                           |

---

This completes the detailed analysis, documentation, and reproduction instructions for the "Track TikTok Ads Library with Apify, Google Sheets & Slack/Telegram Notifications" workflow.