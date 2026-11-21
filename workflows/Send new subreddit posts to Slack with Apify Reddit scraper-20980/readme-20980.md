Send new subreddit posts to Slack with Apify Reddit scraper

https://n8nworkflows.xyz/workflows/send-new-subreddit-posts-to-slack-with-apify-reddit-scraper-20980


# Send new subreddit posts to Slack with Apify Reddit scraper

### 1. Workflow Overview

This workflow is designed to automate the collection of new Reddit posts from targeted subreddits and deliver them to a designated Slack channel on an hourly schedule. It serves as a modern replacement for deprecated Reddit RSS feeds and avoids the complexities of maintaining a custom Reddit API application with OAuth requirements.

The logical processing is divided into two sequential functional blocks:
- **1.1 Fetch New Posts:** Manages execution timing, sets configuration parameters (subreddits, lookback window, post limits), and invokes the Apify Reddit Scraper Actor to retrieve recent platform data.
- **1.2 Deliver Once:** Filters raw scraper outputs to retain only valid post items, eliminates duplicate threads using execution history, and formats/dispatches the resulting data to Slack.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Fetch New Posts
- **Overview:** Initializes the hourly execution cycle, defines target monitoring criteria, and retrieves raw data from Reddit via the Apify platform.
- **Nodes Involved:** `Every hour`, `Settings`, `Get new subreddit posts`.

##### Node Details:
- **Every hour**
  - **Type and Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger Node). Initiates the workflow run automatically every hour.
  - **Configuration Choices:** Interval set to 1 hour (`hoursInterval: 1`).
  - **Key Expressions or Variables:** None.
  - **Input / Output:** No input connections; outputs to the `Settings` node.
  - **Version Requirements:** Version 1.2.
  - **Edge Cases / Failure Types:** Missed triggers if the n8n instance experiences downtime during the scheduled interval.

- **Settings**
  - **Type and Technical Role:** `n8n-nodes-base.set` (Data Transformation Node). Establishes environment variables and execution parameters for the scraper.
  - **Configuration Choices:** Assigns string and numeric values to variables (`subreddits`, `lookback`, `maxPosts`).
  - **Key Expressions or Variables:** 
    - `subreddits`: `"SaaS, startups, Entrepreneur"`
    - `lookback`: `"70 minutes"`
    - `maxPosts`: `100`
  - **Input / Output:** Input from `Every hour`; outputs to `Get new subreddit posts`.
  - **Version Requirements:** Version 3.4.
  - **Edge Cases / Failure Types:** Invalid data types if modified manually.

- **Get new subreddit posts**
  - **Type and Technical Role:** `@apify/n8n-nodes-apify.apify` (Action Node). Executes the external Apify Actor (`Reddit Scraper`) and waits for dataset results.
  - **Configuration Choices:** 
    - Resource: `Actors`
    - Operation: `Run actor and get dataset`
    - Memory: `512` MB
    - Max Total Charge: `$1`
    - Actor Source: `store` (`harshmaur/reddit-scraper`)
  - **Key Expressions or Variables:** 
    - `customBody`: Uses JavaScript string manipulation to parse the comma-separated subreddit string into an array, clean whitespace, and format properties (`subredditUrls`, `postedAfter`, `maxPostsCount`, `crawlCommentsPerPost: false`, `includeNSFW: false`).
  - **Input / Output:** Input from `Settings`; outputs to `Keep posts only`.
  - **Version Requirements:** Version 1.
  - **Edge Cases / Failure Types:** Apify API authentication errors, insufficient Apify account credits, or timeouts during heavy scraping operations. Requires the `@apify/n8n-nodes-apify` community node.
  - **Credentials Required:** `apifyApi` (Apify Account).

---

#### Block 1.2: Deliver Once
- **Overview:** Validates scraped payloads, suppresses duplicate notifications using historical execution states, and structures individual threads for delivery to a Slack channel.
- **Nodes Involved:** `Keep posts only`, `Skip posts already sent`, `Post to Slack`.

##### Node Details:
- **Keep posts only**
  - **Type and Technical Role:** `n8n-nodes-base.filter` (Flow Control Node). Filters out unwanted items (such as comments or metadata) returned by the scraper.
  - **Configuration Choices:** Combinator set to `and` with a strict evaluation rule.
  - **Key Expressions or Variables:** 
    - Left Value: `={{ $json.dataType }}`
    - Operator: `equals`
    - Right Value: `post`
  - **Input / Output:** Input from `Get new subreddit posts`; outputs to `Skip posts already sent`.
  - **Version Requirements:** Version 2.2.
  - **Edge Cases / Failure Types:** Schema changes in the Apify actor output causing `dataType` evaluation to fail.

- **Skip posts already sent**
  - **Type and Technical Role:** `n8n-nodes-base.removeDuplicates` (Data Processing Node). Compares current item identifiers against past executions to prevent duplicate messaging.
  - **Configuration Choices:** 
    - Operation: `removeItemsSeenInPreviousExecutions`
  - **Key Expressions or Variables:** 
    - Dedupe Value: `={{ $json.id }}`
  - **Input / Output:** Input from `Keep posts only`; outputs to `Post to Slack`.
  - **Version Requirements:** Version 2.
  - **Edge Cases / Failure Types:** Loss of execution history storage within the n8n database, which could result in re-sending previously processed posts.

- **Post to Slack**
  - **Type and Technical Role:** `n8n-nodes-base.slack` (Messaging Action Node). Sends formatted post metadata as a text message to a specific Slack workspace channel.
  - **Configuration Choices:** 
    - Resource: `message`
    - Operation: `post`
    - Message Type: `text`
    - Unfurling: Disabled (`false`)
    - Include Link to Workflow: Disabled (`false`)
  - **Key Expressions or Variables:** 
    - `text`: Constructs a Markdown message string containing escaped HTML entities for safety, coupled with dynamic fields (`$json.postUrl`, `$json.title`, `$json.communityName`, `$json.authorName`, `$json.upVotes`, `$json.commentsCount`).
  - **Input / Output:** Input from `Skip posts already sent`; terminal node.
  - **Version Requirements:** Version 2.3.
  - **Edge Cases / Failure Types:** Slack API rate limits, revoked OAuth tokens, or missing permissions for the bot to write to the selected channel.
  - **Credentials Required:** `slackApi` (Slack Account).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Overview** | `n8n-nodes-base.stickyNote` | Visual documentation and architecture notes | *None* | *None* | ## Send new subreddit posts to Slack every hour (Reddit RSS replacement)<br><br>Reddit RSS feeds stop on 2026-11-13. This workflow does the same job as an **RSS Read** node on `reddit.com/r/<name>/new/.rss`: every hour it posts each new thread from your subreddits to Slack. It uses the Reddit Scraper Actor on Apify, so you need no Reddit app, OAuth or API approval.<br><br>### How it works<br>1. **Every hour** starts the run.<br>2. **Settings** holds your subreddits and the look-back window.<br>3. **Get new subreddit posts** runs the Apify Actor with `subredditUrls` and `postedAfter: "70 minutes"`.<br>4. **Skip posts already sent** drops any post an earlier run already delivered (the 10-minute overlap means nothing slips between runs).<br>5. **Post to Slack** sends title, link, subreddit, author and score.<br><br>### Setup<br>1. Add the **Apify** node (`@apify/n8n-nodes-apify`, a verified community node) if your n8n does not have it yet.<br>2. Create an **Apify API** credential with the token from [Apify Console → Settings → API & Integrations](https://console.apify.com/settings/integrations). The free Apify plan includes monthly credit.<br>3. Connect Slack and pick a channel in **Post to Slack**.<br>4. Edit `subreddits` in **Settings** (comma-separated names, e.g. `SaaS, startups`).<br><br>### Customization tips<br>- Need fewer pings? Run every 6 hours and set `lookback` a little longer than the interval, e.g. `7 hours`.<br>- Swap Slack for Discord, Teams, email or a database: every post carries `title`, `postUrl`, `body`, `communityName`, `authorName`, `createdAt`, `upVotes`, `commentsCount`.<br>- Results cost $2 per 1,000 on Apify (less on paid Apify plans). The Apify node caps each run at $1. |
| **Section fetch** | `n8n-nodes-base.stickyNote` | Visual grouping for fetching logic | *None* | *None* | ## 1. Fetch new posts<br>Schedule, your settings, and one Apify run per hour. |
| **Section deliver** | `n8n-nodes-base.stickyNote` | Visual grouping for filtering and delivery | *None* | *None* | ## 2. Deliver once<br>Drop repeats from earlier runs, then post to Slack. |
| **Every hour** | `n8n-nodes-base.scheduleTrigger` | Triggers execution on an hourly interval | *None* | Settings | ## 1. Fetch new posts<br>Schedule, your settings, and one Apify run per hour. |
| **Settings** | `n8n-nodes-base.set` | Defines configuration variables for scraping | Every hour | Get new subreddit posts | ## 1. Fetch new posts<br>Schedule, your settings, and one Apify run per hour. |
| **Get new subreddit posts** | `@apify/n8n-nodes-apify.apify` | Scrapes target subreddits via Apify Actor | Settings | Keep posts only | ## 1. Fetch new posts<br>Schedule, your settings, and one Apify run per hour. |
| **Keep posts only** | `n8n-nodes-base.filter` | Filters out non-post items from dataset | Get new subreddit posts | Skip posts already sent | ## 2. Deliver once<br>Drop repeats from earlier runs, then post to Slack. |
| **Skip posts already sent** | `n8n-nodes-base.removeDuplicates` | Prevents duplicate messaging via execution history | Keep posts only | Post to Slack | ## 2. Deliver once<br>Drop repeats from earlier runs, then post to Slack. |
| **Post to Slack** | `n8n-nodes-base.slack` | Posts formatted thread data to Slack | Skip posts already sent | *None* | ## 2. Deliver once<br>Drop repeats from earlier runs, then post to Slack. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create and configure the Trigger Node:**
   - Add a **Schedule Trigger** node.
   - Name it `Every hour`.
   - Set interval to run every `1` hour.

2. **Create and configure the Settings Node:**
   - Add a **Set (Edit Fields)** node and connect it to `Every hour`.
   - Name it `Settings`.
   - Add three string/number assignments:
     - Name: `subreddits` | Type: `String` | Value: `SaaS, startups, Entrepreneur`
     - Name: `lookback` | Type: `String` | Value: `70 minutes`
     - Name: `maxPosts` | Type: `Number` | Value: `100`

3. **Create and configure the Apify Node:**
   - Install the community node package `@apify/n8n-nodes-apify` if not already present.
   - Add an **Apify** node and connect it downstream from `Settings`.
   - Name it `Get new subreddit posts`.
   - Set Resource to `Actors`, Operation to `Run actor and get dataset`.
   - Select Actor ID `harshmaur/reddit-scraper` (Reddit Scraper - Posts, Comments, Search & Subreddits).
   - Configure Memory to `512` and Max Total Charge to `$1`.
   - Populate the Custom Body expression:
     ```javascript
     ={{ JSON.stringify({ subredditUrls: $json.subreddits.split(',').map(s => s.trim()).filter(Boolean), postedAfter: $json.lookback, maxPostsCount: $json.maxPosts, crawlCommentsPerPost: false, includeNSFW: false }, null, 2) }}
     ```
   - Connect valid **Apify API** credentials (`apifyApi`).

4. **Create and configure the Filter Node:**
   - Add a **Filter** node and connect it to `Get new subreddit posts`.
   - Name it `Keep posts only`.
   - Set condition logic to evaluate if `={{ $json.dataType }}` equals `post`.

5. **Create and configure the Duplicate Removal Node:**
   - Add a **Remove Duplicates** node and connect it to `Keep posts only`.
   - Name it `Skip posts already sent`.
   - Set operation to `Remove items seen in previous executions`.
   - Set Dedupe Value to `={{ $json.id }}`.

6. **Create and configure the Slack Node:**
   - Add a **Slack** node and connect it to `Skip posts already sent`.
   - Name it `Post to Slack`.
   - Set Resource to `Message`, Operation to `Post`.
   - Set Message Type to `Text`.
   - In advanced options, ensure Link Unfurling is disabled.
   - Populate the Text field using the following expression:
     ```text
     =*<{{ $json.postUrl }}|{{ $json.title.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;') }}>*
     {{ $json.communityName }} · u/{{ $json.authorName }} · {{ $json.upVotes }} upvotes · {{ $json.commentsCount }} comments
     ```
   - Connect valid **Slack API** credentials (`slackApi`) and select your target workspace channel.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Console Settings & Integrations | [Apify Console API & Integrations](https://console.apify.com/settings/integrations) |
| Apify Actor Documentation for Reddit Scraper | [Apify Actor Listing](https://console.apify.com/actors/9sHOY9RzPYGjmTHo8/input) |