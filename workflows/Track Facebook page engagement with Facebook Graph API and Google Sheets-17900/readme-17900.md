Track Facebook page engagement with Facebook Graph API and Google Sheets

https://n8nworkflows.xyz/workflows/track-facebook-page-engagement-with-facebook-graph-api-and-google-sheets-17900


# Track Facebook page engagement with Facebook Graph API and Google Sheets

### 1. Workflow Overview

The purpose of this workflow is to automate the tracking of a Facebook Page’s recent post performance by pulling engagement metrics—specifically likes, comments, and shares—via the Facebook Graph API and synchronizing them with a Google Sheets spreadsheet. Target use cases include social media analytics, automated reporting, and maintaining historical engagement records without manual data entry.

The workflow logic is divided into the following functional blocks:
- **1.1 Input Reception & Configuration:** Initializes the workflow execution either manually or via a daily schedule and sets up global runtime variables.
- **1.2 Data Collection & Parsing:** Connects to the Facebook Graph API to fetch the authenticated page details, retrieves the recent feed, and splits the feed array into individual post items.
- **1.3 Parallel Engagement Extraction:** Fans out each post into three parallel branches to fetch comments, reactions, and shares independently, formatting each metric payload into a standardized schema.
- **1.4 Aggregation & Storage:** Merges the parallel metric streams back into a unified record per post, upserts the data into Google Sheets, and applies a controlled pause to respect API rate limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Configuration
- **Overview:** This block acts as the entry point for the workflow, allowing execution on-demand or automatically on a schedule, while centralizing configuration parameters such as API versions and rate limits.
- **Nodes Involved:** `Manually Run Workflow`, `Daily Schedule (Optional)`, `Configuration`
- **Node Details:**
  - **Manually Run Workflow**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Trigger Node). Initiates workflow execution manually on user request.
    - *Configuration Choices:* Default configuration.
    - *Input/Output Connections:* No inputs; outputs to `Configuration`.
    - *Edge Cases/Failure Types:* User authentication or n8n editor connectivity issues.
  - **Daily Schedule (Optional)**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node). Automatically triggers the workflow execution at a set schedule (configured to run daily at 08:00).
    - *Configuration Choices:* Rule configured with an interval triggered at hour 8.
    - *Input/Output Connections:* No inputs; outputs to `Configuration`.
    - *Edge Cases/Failure Types:* Timezone misalignment or server downtime during the scheduled trigger window.
  - **Configuration**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation/Assignment Node). Defines global static variables used across subsequent API calls.
    - *Configuration Choices:* Assigns `max_posts` (number: 3), `graph_api_version` (string: `v21.0`), and `wait_seconds_between_posts` (number: 2).
    - *Key Expressions/Variables:* None (static data assignment).
    - *Input/Output Connections:* Inputs from `Manually Run Workflow` or `Daily Schedule (Optional)`; outputs to `Get Facebook Page Info`.
    - *Edge Cases/Failure Types:* Expression evaluation issues if downstream nodes reference misnamed variable keys.

#### 1.2 Data Collection & Parsing
- **Overview:** This block authenticates against the Facebook Graph API, requests the target page's recent posts based on configuration limits, and breaks down the feed array into individual item streams for granular processing.
- **Nodes Involved:** `Get Facebook Page Info`, `Get Recent Page Posts`, `Split Out Each Post`
- **Node Details:**
  - **Get Facebook Page Info**
    - *Type and Technical Role:* `n8n-nodes-base.facebookGraphApi` (API Integration Node). Fetches basic profile information for the authenticated Facebook entity.
    - *Configuration Choices:* Queries the `me` node using the Graph API version sourced from the configuration block.
    - *Key Expressions/Variables:* `={{ $('Configuration').item.json.graph_api_version }}`
    - *Input/Output Connections:* Input from `Configuration`; outputs to `Get Recent Page Posts`.
    - *Edge Cases/Failure Types:* Expired Page access tokens, insufficient page permissions, or invalid Graph API version strings.
  - **Get Recent Page Posts**
    - *Type and Technical Role:* `n8n-nodes-base.facebookGraphApi` (API Integration Node). Retrieves the feed items associated with the retrieved Facebook Page ID.
    - *Configuration Choices:* Queries `={{ $json.id }}/feed` requesting fields `id`, `message`, and `created_time`, constrained by a query parameter `limit` linked to the configuration variable.
    - *Key Expressions/Variables:* Node path targets `={{ $json.id }}/feed`; limit parameter set to `={{ $('Configuration').item.json.max_posts }}`.
    - *Input/Output Connections:* Input from `Get Facebook Page Info`; outputs to `Split Out Each Post`.
    - *Edge Cases/Failure Types:* Empty feed responses if the page has no posts, or hitting Graph API pagination/rate limits.
  - **Split Out Each Post**
    - *Type and Technical Role:* `n8n-nodes-base.splitOut` (Flow Control Node). Takes the array of posts (`data`) and splits them into individual execution items.
    - *Configuration Choices:* Field to split out set to `data`.
    - *Input/Output Connections:* Input from `Get Recent Page Posts`; outputs simultaneously to `Fetch Post Comments`, `Fetch Post Reactions`, and `Fetch Post Shares`.
    - *Edge Cases/Failure Types:* Failure if the incoming `data` property is null or not formatted as an array.

#### 1.3 Parallel Engagement Extraction
- **Overview:** For each individual post item, this block concurrently fetches interaction metrics (comments, reactions, and shares) from the Graph API and structures the output data into clean, uniform schema formats.
- **Nodes Involved:** `Fetch Post Comments`, `Format Comment Count`, `Fetch Post Reactions`, `Format Like Count`, `Fetch Post Shares`, `Format Share Count`
- **Node Details:**
  - **Fetch Post Comments**
    - *Type and Technical Role:* `n8n-nodes-base.facebookGraphApi` (API Integration Node). Queries comment totals for a specific post ID.
    - *Configuration Choices:* Node targets `={{ $json.id }}` with edge `comments`. Requests field `id` with query parameters order set to `reverse_chronological`, summary set to `true`, and filter set to `stream`.
    - *Key Expressions/Variables:* Node: `={{ $json.id }}`; API Version: `={{ $('Configuration').item.json.graph_api_version }}`.
    - *Input/Output Connections:* Input from `Split Out Each Post`; outputs to `Format Comment Count`.
    - *Edge Cases/Failure Types:* Post comments disabled or API permission restriction errors.
  - **Format Comment Count**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation Node). Extracts comment summary metrics and pairs them with the originating post ID.
    - *Configuration Choices:* Assigns `post_id` and `comment_count`.
    - *Key Expressions/Variables:* `post_id` = `={{ $('Split Out Each Post').item.json.id }}`, `comment_count` = `={{ $json.summary?.total_count ?? 0 }}`.
    - *Input/Output Connections:* Input from `Fetch Post Comments`; outputs to `Combine Engagement Metrics` (Input 0).
    - *Edge Cases/Failure Types:* Missing summary object falling back to default `0` value.
  - **Fetch Post Reactions**
    - *Type and Technical Role:* `n8n-nodes-base.facebookGraphApi` (API Integration Node). Queries reaction/like totals for a specific post ID.
    - *Configuration Choices:* Node targets `={{ $json.id }}` with edge `reactions`. Requests field `type` with query parameters order set to `reverse_chronological` and summary set to `true`.
    - *Key Expressions/Variables:* Node: `={{ $json.id }}`; API Version: `={{ $('Configuration').item.json.graph_api_version }}`.
    - *Input/Output Connections:* Input from `Split Out Each Post`; outputs to `Format Like Count`.
    - *Edge Cases/Failure Types:* API rate limits or restricted post visibility.
  - **Format Like Count**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation Node). Extracts reaction summary totals and assigns them to the post ID.
    - *Configuration Choices:* Assigns `post_id` and `like_count`.
    - *Key Expressions/Variables:* `post_id` = `={{ $('Split Out Each Post').item.json.id }}`, `like_count` = `={{ $json.summary?.total_count ?? 0 }}`.
    - *Input/Output Connections:* Input from `Fetch Post Reactions`; outputs to `Combine Engagement Metrics` (Input 1).
    - *Edge Cases/Failure Types:* Missing summary object falling back to default `0` value.
  - **Fetch Post Shares**
    - *Type and Technical Role:* `n8n-nodes-base.facebookGraphApi` (API Integration Node). Queries share metrics and general post metadata.
    - *Configuration Choices:* Node targets `={{ $json.id }}` requesting fields `id`, `message`, `created_time`, and `shares` with query parameter order set to `reverse_chronological`.
    - *Key Expressions/Variables:* Node: `={{ $json.id }}`; API Version: `={{ $('Configuration').item.json.graph_api_version }}`.
    - *Input/Output Connections:* Input from `Split Out Each Post`; outputs to `Format Share Count`.
    - *Edge Cases/Failure Types:* Shares field omitted by API if zero shares exist.
  - **Format Share Count**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation Node). Parses share metrics, post message text, and associates them with the post ID.
    - *Configuration Choices:* Assigns `post_id`, `post_message`, and `share_count`.
    - *Key Expressions/Variables:* `post_id` = `={{ $('Split Out Each Post').item.json.id }}`, `post_message` = `={{ $('Split Out Each Post').item.json.message }}`, `share_count` = `={{ $json.shares?.count ?? 0 }}`.
    - *Input/Output Connections:* Input from `Fetch Post Shares`; outputs to `Combine Engagement Metrics` (Input 2).
    - *Edge Cases/Failure Types:* Unhandled undefined properties on posts without a message text string.

#### 1.4 Aggregation & Storage
- **Overview:** Consolidates the parallel metric branches into a single row structure per post, updates or appends the dataset to Google Sheets, and throttles execution speed to comply with rate limits.
- **Nodes Involved:** `Combine Engagement Metrics`, `Save Engagement to Sheet`, `Rate Limit Pause`
- **Node Details:**
  - **Combine Engagement Metrics**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data Merging Node). Recombines the three parallel branches (`comment_count`, `like_count`, `share_count`) into a unified item.
    - *Configuration Choices:* Mode set to `combine` using `combineByPosition` across `3` number inputs.
    - *Input/Output Connections:* Inputs from `Format Comment Count`, `Format Like Count`, and `Format Share Count`; outputs to `Save Engagement to Sheet`.
    - *Edge Cases/Failure Types:* Branch data misalignment if item counts differ across parallel processing branches.
  - **Save Engagement to Sheet**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Destination Integration Node). Appends or updates rows inside the connected Google Sheet based on matching criteria.
    - *Configuration Choices:* Operation set to `appendOrUpdate`. Document ID configured to "Count Facebook Engagement" and Sheet Name set to "Sheet1". Uses Google Sheets OAuth2 API credentials.
    - *Input/Output Connections:* Input from `Combine Engagement Metrics`; outputs to `Rate Limit Pause`.
    - *Edge Cases/Failure Types:* Authentication revocation, spreadsheet permission errors, or missing header matching keys during updates.
  - **Rate Limit Pause**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Flow Control Node). Delays execution between iterations to prevent hitting Facebook API rate limits.
    - *Configuration Choices:* Wait amount configured via expression referencing the configuration variable.
    - *Key Expressions/Variables:* `={{ $('Configuration').item.json.wait_seconds_between_posts }}`
    - *Input/Output Connections:* Input from `Save Engagement to Sheet`; no downstream output.
    - *Edge Cases/Failure Types:* Execution timeout if wait durations accumulate excessively across large datasets.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Manually Run Workflow | n8n-nodes-base.manualTrigger | Trigger workflow manually | None | Configuration | # Track Facebook page engagement to Google Sheets<br># 📥 [Open full documentation on Notion](https://automatisation.notion.site/Course-Track-Facebook-page-engagement-to-Google-Sheets-3a03d6550fd98157b27bf6177ca7a660)<br><br>## How it works<br>This workflow tracks engagement (likes, comments, shares) for your Facebook Page's most recent posts and logs one consolidated row per post to Google Sheets.<br><br>It starts from a manual trigger or an optional daily schedule, reads your settings from the Configuration node, fetches your Page feed, and splits it into individual posts. For each post, it fetches comments, reactions and shares in parallel, combines the three metrics into a single row, and upserts it into your Google Sheet using the Post ID as a unique key so re-runs update existing rows instead of duplicating them. A short wait between posts protects you from Facebook API rate limits.<br><br>## Setup<br>1. Create a Facebook Graph API credential in n8n using a Page access token from the Graph API Explorer.<br>2. Connect your Google Sheets account.<br>3. Open "Save Engagement to Sheet" and pick your spreadsheet + tab (columns: POST ID, POST, LIKES, COMMENTS, SHARES).<br>4. Adjust values in "Configuration" (max posts to analyze, Graph API version, pause between posts).<br>5. Run manually to test, or enable the schedule trigger for daily tracking.<br><br>## Requirements<br>- A Facebook Page you manage plus a Graph API Explorer access token<br>- Google Sheets account connected in n8n<br>- A spreadsheet with columns: POST ID, POST, LIKES, COMMENTS, SHARES<br><br>## Customization<br>- Change `max_posts` to analyze more or fewer recent posts.<br>- Adjust `wait_seconds_between_posts` if you hit Facebook rate limits.<br>- Swap the manual trigger for the schedule trigger to automate daily tracking.<br>- Add more metrics (e.g. reach, impressions) as extra parallel branches feeding the same Merge node.<br><br>Need help customizing?<br>Contact me for consulting and support : [Linkedin](https://www.linkedin.com/in/doctor-firass/)<br><br># MY NEW YOUTUBE CHANNEL<br>👉 [Subscribe to my new YouTube channel](https://www.youtube.com/@DrFiras_AI). Here I'll share videos and Shorts with practical tutorials and FREE templates for n8n.<br><br>[![The AI Doctor](https://www.dr-firas.com/the-ai-doctor.png)](https://www.youtube.com/@DrFiras_AI)<br>## Trigger & Configuration<br>Start manually or on a daily schedule, then set your variables once in Configuration. |
| Configuration | n8n-nodes-base.set | Store global variables | Manually Run Workflow, Daily Schedule (Optional) | Get Facebook Page Info | # Track Facebook page engagement to Google Sheets<br># 📥 [Open full documentation on Notion](https://automatisation.notion.site/Course-Track-Facebook-page-engagement-to-Google-Sheets-3a03d6550fd98157b27bf6177ca7a660)<br><br>## How it works<br>This workflow tracks engagement (likes, comments, shares) for your Facebook Page's most recent posts and logs one consolidated row per post to Google Sheets.<br><br>It starts from a manual trigger or an optional daily schedule, reads your settings from the Configuration node, fetches your Page feed, and splits it into individual posts. For each post, it fetches comments, reactions and shares in parallel, combines the three metrics into a single row, and upserts it into your Google Sheet using the Post ID as a unique key so re-runs update existing rows instead of duplicating them. A short wait between posts protects you from Facebook API rate limits.<br><br>## Setup<br>1. Create a Facebook Graph API credential in n8n using a Page access token from the Graph API Explorer.<br>2. Connect your Google Sheets account.<br>3. Open "Save Engagement to Sheet" and pick your spreadsheet + tab (columns: POST ID, POST, LIKES, COMMENTS, SHARES).<br>4. Adjust values in "Configuration" (max posts to analyze, Graph API version, pause between posts).<br>5. Run manually to test, or enable the schedule trigger for daily tracking.<br><br>## Requirements<br>- A Facebook Page you manage plus a Graph API Explorer access token<br>- Google Sheets account connected in n8n<br>- A spreadsheet with columns: POST ID, POST, LIKES, COMMENTS, SHARES<br><br>## Customization<br>- Change `max_posts` to analyze more or fewer recent posts.<br>- Adjust `wait_seconds_between_posts` if you hit Facebook rate limits.<br>- Swap the manual trigger for the schedule trigger to automate daily tracking.<br>- Add more metrics (e.g. reach, impressions) as extra parallel branches feeding the same Merge node.<br><br>Need help customizing?<br>Contact me for consulting and support : [Linkedin](https://www.linkedin.com/in/doctor-firass/)<br><br># MY NEW YOUTUBE CHANNEL<br>👉 [Subscribe to my new YouTube channel](https://www.youtube.com/@DrFiras_AI). Here I'll share videos and Shorts with practical tutorials and FREE templates for n8n.<br><br>[![The AI Doctor](https://www.dr-firas.com/the-ai-doctor.png)](https://www.youtube.com/@DrFiras_AI)<br>## Trigger & Configuration<br>Start manually or on a daily schedule, then set your variables once in Configuration. |
| Daily Schedule (Optional) | n8n-nodes-base.scheduleTrigger | Trigger workflow on daily schedule | None | Configuration | # Track Facebook page engagement to Google Sheets<br># 📥 [Open full documentation on Notion](https://automatisation.notion.site/Course-Track-Facebook-page-engagement-to-Google-Sheets-3a03d6550fd98157b27bf6177ca7a660)<br><br>## How it works<br>This workflow tracks engagement (likes, comments, shares) for your Facebook Page's most recent posts and logs one consolidated row per post to Google Sheets.<br><br>It starts from a manual trigger or an optional daily schedule, reads your settings from the Configuration node, fetches your Page feed, and splits it into individual posts. For each post, it fetches comments, reactions and shares in parallel, combines the three metrics into a single row, and upserts it into your Google Sheet using the Post ID as a unique key so re-runs update existing rows instead of duplicating them. A short wait between posts protects you from Facebook API rate limits.<br><br>## Setup<br>1. Create a Facebook Graph API credential in n8n using a Page access token from the Graph API Explorer.<br>2. Connect your Google Sheets account.<br>3. Open "Save Engagement to Sheet" and pick your spreadsheet + tab (columns: POST ID, POST, LIKES, COMMENTS, SHARES).<br>4. Adjust values in "Configuration" (max posts to analyze, Graph API version, pause between posts).<br>5. Run manually to test, or enable the schedule trigger for daily tracking.<br><br>## Requirements<br>- A Facebook Page you manage plus a Graph API Explorer access token<br>- Google Sheets account connected in n8n<br>- A spreadsheet with columns: POST ID, POST, LIKES, COMMENTS, SHARES<br><br>## Customization<br>- Change `max_posts` to analyze more or fewer recent posts.<br>- Adjust `wait_seconds_between_posts` if you hit Facebook rate limits.<br>- Swap the manual trigger for the schedule trigger to automate daily tracking.<br>- Add more metrics (e.g. reach, impressions) as extra parallel branches feeding the same Merge node.<br><br>Need help customizing?<br>Contact me for consulting and support : [Linkedin](https://www.linkedin.com/in/doctor-firass/)<br><br># MY NEW YOUTUBE CHANNEL<br>👉 [Subscribe to my new YouTube channel](https://www.youtube.com/@DrFiras_AI). Here I'll share videos and Shorts with practical tutorials and FREE templates for n8n.<br><br>[![The AI Doctor](https://www.dr-firas.com/the-ai-doctor.png)](https://www.youtube.com/@DrFiras_AI)<br>## Trigger & Configuration<br>Start manually or on a daily schedule, then set your variables once in Configuration. |
| Get Facebook Page Info | n8n-nodes-base.facebookGraphApi | Fetch page details | Configuration | Get Recent Page Posts | # Track Facebook page engagement to Google Sheets<br># 📥 [Open full documentation on Notion](https://automatisation.notion.site/Course-Track-Facebook-page-engagement-to-Google-Sheets-3a03d6550fd98157b27bf6177ca7a660)<br><br>## How it works<br>This workflow tracks engagement (likes, comments, shares) for your Facebook Page's most recent posts and logs one consolidated row per post to Google Sheets.<br><br>It starts from a manual trigger or an optional daily schedule, reads your settings from the Configuration node, fetches your Page feed, and splits it into individual posts. For each post, it fetches comments, reactions and shares in parallel, combines the three metrics into a single row, and upserts it into your Google Sheet using the Post ID as a unique key so re-runs update existing rows instead of duplicating them. A short wait between posts protects you from Facebook API rate limits.<br><br>## Setup<br>1. Create a Facebook Graph API credential in n8n using a Page access token from the Graph API Explorer.<br>2. Connect your Google Sheets account.<br>3. Open "Save Engagement to Sheet" and pick your spreadsheet + tab (columns: POST ID, POST, LIKES, COMMENTS, SHARES).<br>4. Adjust values in "Configuration" (max posts to analyze, Graph API version, pause between posts).<br>5. Run manually to test, or enable the schedule trigger for daily tracking.<br><br>## Requirements<br>- A Facebook Page you manage plus a Graph API Explorer access token<br>- Google Sheets account connected in n8n<br>- A spreadsheet with columns: POST ID, POST, LIKES, COMMENTS, SHARES<br><br>## Customization<br>- Change `max_posts` to analyze more or fewer recent posts.<br>- Adjust `wait_seconds_between_posts` if you hit Facebook rate limits.<br>- Swap the manual trigger for the schedule trigger to automate daily tracking.<br>- Add more metrics (e.g. reach, impressions) as extra parallel branches feeding the same Merge node.<br><br>Need help customizing?<br>Contact me for consulting and support : [Linkedin](https://www.linkedin.com/in/doctor-firass/)<br><br># MY NEW YOUTUBE CHANNEL<br>👉 [Subscribe to my new YouTube channel](https://www.youtube.com/@DrFiras_AI). Here I'll share videos and Shorts with practical tutorials and FREE templates for n8n.<br><br>[![The AI Doctor](https://www.dr-firas.com/the-ai-doctor.png)](https://www.youtube.com/@DrFiras_AI)<br>## Collect page & feed data<br>Fetch page info, get the recent feed, split into individual posts. |
| Get Recent Page Posts | n8n-nodes-base.facebookGraphApi | Fetch recent feed items | Get Facebook Page Info | Split Out Each Post | # Track Facebook page engagement to Google Sheets<br># 📥 [Open full documentation on Notion](https://automatisation.notion.site/Course-Track-Facebook-page-engagement-to-Google-Sheets-3a03d6550fd98157b27bf6177ca7a660)<br><br>## How it works<br>This workflow tracks engagement (likes, comments, shares) for your Facebook Page's most recent posts and logs one consolidated row per post to Google Sheets.<br><br>It starts from a manual trigger or an optional daily schedule, reads your settings from the Configuration node, fetches your Page feed, and splits it into individual posts. For each post, it fetches comments, reactions and shares in parallel, combines the three metrics into a single row, and upserts it into your Google Sheet using the Post ID as a unique key so re-runs update existing rows instead of duplicating them. A short wait between posts protects you from Facebook API rate limits.<br><br>## Setup<br>1. Create a Facebook Graph API credential in n8n using a Page access token from the Graph API Explorer.<br>2. Connect your Google Sheets account.<br>3. Open "Save Engagement to Sheet" and pick your spreadsheet + tab (columns: POST ID, POST, LIKES, COMMENTS, SHARES).<br>4. Adjust values in "Configuration" (max posts to analyze, Graph API version, pause between posts).<br>5. Run manually to test, or enable the schedule trigger for daily tracking.<br><br>## Requirements<br>- A Facebook Page you manage plus a Graph API Explorer access token<br>- Google Sheets account connected in n8n<br>- A spreadsheet with columns: POST ID, POST, LIKES, COMMENTS, SHARES<br><br>## Customization<br>- Change `max_posts` to analyze more or fewer recent posts.<br>- Adjust `wait_seconds_between_posts` if you hit Facebook rate limits.<br>- Swap the manual trigger for the schedule trigger to automate daily tracking.<br>- Add more metrics (e.g. reach, impressions) as extra parallel branches feeding the same Merge node.<br><br>Need help customizing?<br>Contact me for consulting and support : [Linkedin](https://www.linkedin.com/in/doctor-firass/)<br><br># MY NEW YOUTUBE CHANNEL<br>👉 [Subscribe to my new YouTube channel](https://www.youtube.com/@DrFiras_AI). Here I'll share videos and Shorts with practical tutorials and FREE templates for n8n.<br><br>[![The AI Doctor](https://www.dr-firas.com/the-ai-doctor.png)](https://www.youtube.com/@DrFiras_AI)<br>## Collect page & feed data<br>Fetch page info, get the recent feed, split into individual posts. |
| Split Out Each Post | n8n-nodes-base.splitOut | Split feed array into individual posts | Get Recent Page Posts | Fetch Post Comments, Fetch Post Reactions, Fetch Post Shares | # Track Facebook page engagement to Google Sheets<br># 📥 [Open full documentation on Notion](https://automatisation.notion.site/Course-Track-Facebook-page-engagement-to-Google-Sheets-3a03d6550fd98157b27bf6177ca7a660)<br><br>## How it works<br>This workflow tracks engagement (likes, comments, shares) for your Facebook Page's most recent posts and logs one consolidated row per post to Google Sheets.<br><br>It starts from a manual trigger or an optional daily schedule, reads your settings from the Configuration node, fetches your Page feed, and splits it into individual posts. For each post, it fetches comments, reactions and shares in parallel, combines the three metrics into a single row, and upserts it into your Google Sheet using the Post ID as a unique key so re-runs update existing rows instead of duplicating them. A short wait between posts protects you from Facebook API rate limits.<br><br>## Setup<br>1. Create a Facebook Graph API credential in n8n using a Page access token from the Graph API Explorer.<br>2. Connect your Google Sheets account.<br>3. Open "Save Engagement to Sheet" and pick your spreadsheet + tab (columns: POST ID, POST, LIKES, COMMENTS, SHARES).<br>4. Adjust values in "Configuration" (max posts to analyze, Graph API version, pause between posts).<br>5. Run manually to test, or enable the schedule trigger for daily tracking.<br><br>## Requirements<br>- A Facebook Page you manage plus a Graph API Explorer access token<br>- Google Sheets account connected in n8n<br>- A spreadsheet with columns: POST ID, POST, LIKES, COMMENTS, SHARES<br><br>## Customization<br>- Change `max_posts` to analyze more or fewer recent posts.<br>- Adjust `wait_seconds_between_posts` if you hit Facebook rate limits.<br>- Swap the manual trigger for the schedule trigger to automate daily tracking.<br>- Add more metrics (e.g. reach, impressions) as extra parallel branches feeding the same Merge node.<br><br>Need help customizing?<br>Contact me for consulting and support : [Linkedin](https://www.linkedin.com/in/doctor-firass/)<br><br># MY NEW YOUTUBE CHANNEL<br>👉 [Subscribe to my new YouTube channel](https://www.youtube.com/@DrFiras_AI). Here I'll share videos and Shorts with practical tutorials and FREE templates for n8n.<br><br>[![The AI Doctor](https://www.dr-firas.com/the-ai-doctor.png)](https://www.youtube.com/@DrFiras_AI)<br>## Collect page & feed data<br>Fetch page info, get the recent feed, split into individual posts. |
| Fetch Post Comments | n8n-nodes-base.facebookGraphApi | Fetch comment metrics per post | Split Out Each Post | Format Comment Count | ## Fetch engagement metrics<br>Three parallel branches fetch comments, reactions and shares for each post. |
| Format Comment Count | n8n-nodes-base.set | Format comment count payload | Fetch Post Comments | Combine Engagement Metrics | ## Fetch engagement metrics<br>Three parallel branches fetch comments, reactions and shares for each post. |
| Fetch Post Reactions | n8n-nodes-base.facebookGraphApi | Fetch reaction metrics per post | Split Out Each Post | Format Like Count | ## Fetch engagement metrics<br>Three parallel branches fetch comments, reactions and shares for each post. |
| Format Like Count | n8n-nodes-base.set | Format like count payload | Fetch Post Reactions | Combine Engagement Metrics | ## Fetch engagement metrics<br>Three parallel branches fetch comments, reactions and shares for each post. |
| Fetch Post Shares | n8n-nodes-base.facebookGraphApi | Fetch share metrics per post | Split Out Each Post | Format Share Count | ## Fetch engagement metrics<br>Three parallel branches fetch comments, reactions and shares for each post. |
| Format Share Count | n8n-nodes-base.set | Format share count and message payload | Fetch Post Shares | Combine Engagement Metrics | ## Fetch engagement metrics<br>Three parallel branches fetch comments, reactions and shares for each post. |
| Combine Engagement Metrics | n8n-nodes-base.merge | Merge parallel metrics streams | Format Comment Count, Format Like Count, Format Share Count | Save Engagement to Sheet | ## Combine & store results<br>Merge the three metrics into one row per post and upsert it into Google Sheets, then pause to respect rate limits. |
| Save Engagement to Sheet | n8n-nodes-base.googleSheets | Upsert engagement metrics to Google Sheets | Combine Engagement Metrics | Rate Limit Pause | ## Combine & store results<br>Merge the three metrics into one row per post and upsert it into Google Sheets, then pause to respect rate limits. |
| Rate Limit Pause | n8n-nodes-base.wait | Throttle execution between posts | Save Engagement to Sheet | None | ## Combine & store results<br>Merge the three metrics into one row per post and upsert it into Google Sheets, then pause to respect rate limits. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** (`n8n-nodes-base.manualTrigger`) node named `Manually Run Workflow`.
   - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`) node named `Daily Schedule (Optional)` and configure its interval trigger at hour `8`.

2. **Create the Configuration Node:**
   - Add a **Set** (`n8n-nodes-base.set`) node named `Configuration`.
   - Connect both `Manually Run Workflow` and `Daily Schedule (Optional)` to `Configuration`.
   - Configure assignments:
     - `max_posts` (Number): `3`
     - `graph_api_version` (String): `v21.0`
     - `wait_seconds_between_posts` (Number): `2`

3. **Fetch Page Info and Feed:**
   - Add a **Facebook Graph API** (`n8n-nodes-base.facebookGraphApi`) node named `Get Facebook Page Info`. Connect `Configuration` output to this node. Set node parameter to `me` and set Graph API Version to `={{ $('Configuration').item.json.graph_api_version }}`.
   - Add a second **Facebook Graph API** node named `Get Recent Page Posts`. Connect `Get Facebook Page Info` output to this node.
     - Set node parameter to `={{ $json.id }}/feed`.
     - Set Graph API Version to `={{ $('Configuration').item.json.graph_api_version }}`.
     - Under Options, add fields: `id`, `message`, `created_time`.
     - Add query parameter: Name `limit`, Value `={{ $('Configuration').item.json.max_posts }}`.

4. **Split Posts:**
   - Add a **Split Out** (`n8n-nodes-base.splitOut`) node named `Split Out Each Post`. Connect `Get Recent Page Posts` to it.
   - Set the `Field to Split Out` parameter to `data`.

5. **Create Parallel Engagement Branches:**
   - **Comments Branch:**
     - Add a **Facebook Graph API** node named `Fetch Post Comments`. Connect from `Split Out Each Post`. Set node to `={{ $json.id }}`, edge to `comments`, Graph API Version to `={{ $('Configuration').item.json.graph_api_version }}`, add field `id`, and add query parameters: `order` = `reverse_chronological`, `summary` = `true`, `filter` = `stream`.
     - Add a **Set** node named `Format Comment Count`. Connect from `Fetch Post Comments`. Assign `post_id` = `={{ $('Split Out Each Post').item.json.id }}` and `comment_count` = `={{ $json.summary?.total_count ?? 0 }}`.
   - **Reactions Branch:**
     - Add a **Facebook Graph API** node named `Fetch Post Reactions`. Connect from `Split Out Each Post`. Set node to `={{ $json.id }}`, edge to `reactions`, Graph API Version to `={{ $('Configuration').item.json.graph_api_version }}`, add field `type`, and add query parameters: `order` = `reverse_chronological`, `summary` = `true`.
     - Add a **Set** node named `Format Like Count`. Connect from `Fetch Post Reactions`. Assign `post_id` = `={{ $('Split Out Each Post').item.json.id }}` and `like_count` = `={{ $json.summary?.total_count ?? 0 }}`.
   - **Shares Branch:**
     - Add a **Facebook Graph API** node named `Fetch Post Shares`. Connect from `Split Out Each Post`. Set node to `={{ $json.id }}`, Graph API Version to `={{ $('Configuration').item.json.graph_api_version }}`, add fields `id`, `message`, `created_time`, `shares`, and add query parameter: `order` = `reverse_chronological`.
     - Add a **Set** node named `Format Share Count`. Connect from `Fetch Post Shares`. Assign `post_id` = `={{ $('Split Out Each Post').item.json.id }}`, `post_message` = `={{ $('Split Out Each Post').item.json.message }}`, and `share_count` = `={{ $json.shares?.count ?? 0 }}`.

6. **Merge and Store Metrics:**
   - Add a **Merge** (`n8n-nodes-base.merge`) node named `Combine Engagement Metrics`.
     - Set Mode to `combine`, Combine By to `combineByPosition`, and Number Inputs to `3`.
     - Connect `Format Comment Count` to input 0, `Format Like Count` to input 1, and `Format Share Count` to input 2.
   - Add a **Google Sheets** (`n8n-nodes-base.googleSheets`) node named `Save Engagement to Sheet`.
     - Connect `Combine Engagement Metrics` to it.
     - Configure Operation to `appendOrUpdate`.
     - Configure Document ID and Sheet Name (e.g., spreadsheet "Count Facebook Engagement", Sheet "Sheet1").
     - Authenticate using a valid Google Sheets OAuth2 credential.
   - Add a **Wait** (`n8n-nodes-base.wait`) node named `Rate Limit Pause`.
     - Connect `Save Engagement to Sheet` to it.
     - Set Amount to `={{ $('Configuration').item.json.wait_seconds_between_posts }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Open full documentation on Notion | [Notion Documentation](https://automatisation.notion.site/Course-Track-Facebook-page-engagement-to-Google-Sheets-3a03d6550fd98157b27bf6177ca7a660) |
| Contact me for consulting and support | [LinkedIn Profile](https://www.linkedin.com/in/doctor-firass/) |
| Subscribe to my new YouTube channel for practical tutorials and FREE templates | [YouTube Channel (@DrFiras_AI)](https://www.youtube.com/@DrFiras_AI) |
| Branding Asset (The AI Doctor) | [Logo Image Link](https://www.dr-firas.com/the-ai-doctor.png) |