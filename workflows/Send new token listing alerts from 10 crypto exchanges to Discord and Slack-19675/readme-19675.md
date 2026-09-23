Send new token listing alerts from 10 crypto exchanges to Discord and Slack

https://n8nworkflows.xyz/workflows/send-new-token-listing-alerts-from-10-crypto-exchanges-to-discord-and-slack-19675


# Send new token listing alerts from 10 crypto exchanges to Discord and Slack

### 1. Workflow Overview

This workflow automates the detection and distribution of new token listings across ten cryptocurrency exchanges. It polls a public aggregation feed every five minutes, filters the results based on user-defined criteria (exchanges, listing type, keywords, and lookback windows), and broadcasts formatted notifications to Discord (using color-coded embeds) and optionally to Slack.

The logic is divided into five functional blocks:
- **1.1 Schedule & Configuration:** Initializes the run cadence and centralizes user parameters (webhooks, language, filters).
- **1.2 Data Ingestion & Guard:** Fetches the raw JSON feed from the public API and guards against empty or unreachable responses.
- **1.3 Item Processing & Filtering:** Breaks down the payload into individual listings, filters out previously processed items using execution memory, and applies exchange and keyword filters.
- **1.4 Field Preparation:** Normalizes and formats the payload fields (headline translation selection, symbol aggregation, and timestamp calculation).
- **1.5 Multi-Channel Delivery:** Evaluates listing types and optional switches to route formatted payloads to Discord webhooks (green for spot, orange for futures) and Slack incoming webhooks.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Schedule & Configuration
- **Overview:** Triggers the workflow execution on a fixed schedule and sets global runtime variables.
- **Nodes Involved:** `Every 5 minutes`, `Your settings`

- **Node Details:**
  - **Every 5 minutes**
    - *Type and technical role:* `n8n-nodes-base.scheduleTrigger` (Triggers execution based on time intervals).
    - *Configuration choices:* Interval configured to trigger every 5 minutes.
    - *Key expressions:* None.
    - *Input/Output:* No inputs; outputs to `Your settings`.
    - *Edge cases:* None.
  - **Your settings**
    - *Type and technical role:* `n8n-nodes-base.set` (Injects configuration parameters into the execution context).
    - *Configuration choices:* Manual mode with predefined string assignments for webhooks, filters, and language preferences.
    - *Key expressions:* Sets `discord_webhook_url`, `exchanges`, `listing_type`, `keyword`, `lookback_days` (default `"1"`), `language` (default `"en"`), `send_to_slack` (default `"false"`), and `slack_webhook_url`.
    - *Input/Output:* Input from `Every 5 minutes`; output to `Fetch new listings`.
    - *Edge cases:* Missing or invalid webhook URLs will cause downstream HTTP request failures.

---

#### Block 1.2: Data Ingestion & Guard
- **Overview:** Retrieves listing announcements from a public endpoint and terminates the execution gracefully if no data is returned.
- **Nodes Involved:** `Fetch new listings`, `Feed returned items?`, `Nothing new this run`

- **Node Details:**
  - **Fetch new listings**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (Executes an HTTP GET request to fetch remote JSON data).
    - *Configuration choices:* URL set to public API (`https://tokenearly.com/api/public/listings.json`), `neverError` option enabled to prevent workflow interruption on HTTP errors. Query parameters map user settings (`days`, `type`, `limit`).
    - *Key expressions:* `={{ $json.lookback_days }}`, `={{ $json.listing_type }}`.
    - *Input/Output:* Input from `Your settings`; output to `Feed returned items?`.
    - *Edge cases:* Network timeouts or API rate-limiting handled via `neverError` (passing error states to the next node).
  - **Feed returned items?**
    - *Type and technical role:* `n8n-nodes-base.if` (Conditional router).
    - *Configuration choices:* Evaluates whether the count of returned items is greater than zero.
    - *Key expressions:* `={{ $json.count }}` evaluated against `> 0`.
    - *Input/Output:* Input from `Fetch new listings`; outputs true branch to `Split into one item per listing`, false branch to `Nothing new this run`.
    - *Edge cases:* Payload missing the `count` property evaluates to false.
  - **Nothing new this run**
    - *Type and technical role:* `n8n-nodes-base.noOp` (Placeholder terminal node for empty data branches).
    - *Configuration choices:* None.
    - *Input/Output:* Input from `Feed returned items?` (false branch); no outputs.
    - *Edge cases:* None.

---

#### Block 1.3: Item Processing & Filtering
- **Overview:** Expands the item array, eliminates duplicate records across executions, and applies exchange and keyword filters.
- **Nodes Involved:** `Split into one item per listing`, `Only listings not seen before`, `Matches your exchange filter?`, `Matches your keyword filter?`, `Skipped by your filters`

- **Node Details:**
  - **Split into one item per listing**
    - *Type and technical role:* `n8n-nodes-base.splitOut` (Transforms an array of items into individual n8n items).
    - *Configuration choices:* Field to split out set to `items`.
    - *Input/Output:* Input from `Feed returned items?`; output to `Only listings not seen before`.
    - *Edge cases:* Missing `items` array causes execution failure.
  - **Only listings not seen before**
    - *Type and technical role:* `n8n-nodes-base.removeDuplicates` (Filters out items processed in previous runs).
    - *Configuration choices:* Operation set to remove items seen in previous executions.
    - *Key expressions:* Deduplication value uses `={{ $json.permalink }}`.
    - *Input/Output:* Input from `Split into one item per listing`; output to `Matches your exchange filter?`.
    - *Edge cases:* Duplicate keys or missing permalinks may bypass deduplication logic.
  - **Matches your exchange filter?**
    - *Type and technical role:* `n8n-nodes-base.if` (Conditional router).
    - *Configuration choices:* Checks if configured exchanges match the listing's exchange.
    - *Key expressions:* `={{ $('Your settings').first().json.exchanges || $json.exchange }}` contains `={{ $json.exchange }}`.
    - *Input/Output:* Input from `Only listings not seen before`; outputs true to `Matches your keyword filter?`, false to `Skipped by your filters`.
    - *Edge cases:* Case sensitivity mismatches between configured exchange IDs and feed data.
  - **Matches your keyword filter?**
    - *Type and technical role:* `n8n-nodes-base.if` (Conditional router).
    - *Configuration choices:* Checks if the combined titles in all languages contain the user's keyword.
    - *Key expressions:* `={{ ($json.title.en || '') + ' ' + ($json.title.zh || '') + ' ' + ($json.title.ko || '') }}` contains `={{ $('Your settings').first().json.keyword }}`.
    - *Input/Output:* Input from `Matches your exchange filter?`; outputs true to `Prepare alert fields`, false to `Skipped by your filters`.
    - *Edge cases:* Null or undefined title objects handled via fallback empty strings.
  - **Skipped by your filters**
    - *Type and technical role:* `n8n-nodes-base.noOp` (Log-collection branch terminal).
    - *Configuration choices:* None.
    - *Input/Output:* Inputs from `Matches your exchange filter?` and `Matches your keyword filter?` (false branches); no outputs.
    - *Edge cases:* None.

---

#### Block 1.4: Field Preparation
- **Overview:** Normalizes data fields, selects the correct headline language, calculates age, and prepares uniform payload structures.
- **Nodes Involved:** `Prepare alert fields`

- **Node Details:**
  - **Prepare alert fields**
    - *Type and technical role:* `n8n-nodes-base.set` (Transforms and assigns data properties).
    - *Configuration choices:* Manual assignment mode mapping localized titles, symbol strings, type labels, age calculations, and source links.
    - *Key expressions:* 
      - `headline`: `={{ $json.title[$('Your settings').first().json.language] || $json.title.en || $json.title.zh || $json.title.ko }}`
      - `symbols_text`: `={{ ($json.symbols || []).join(', ') }}`
      - `type_label`: `={{ $json.type === 'futures' ? 'Futures' : 'Spot' }}`
      - `minutes_old`: `={{ Math.max(0, Math.round((Date.now() - new Date($json.published_at).getTime()) / 60000)) }}`
      - `link`: `={{ $json.source_url || $json.permalink }}`
    - *Input/Output:* Input from `Matches your keyword filter?`; outputs to `Is it a futures listing?` and `Also send to Slack?`.
    - *Edge cases:* Invalid `published_at` date formats may return `NaN` for minutes calculation.

---

#### Block 1.5: Multi-Channel Delivery
- **Overview:** Routes formatted listings to Discord (distinguishing spot vs. futures embeds) and conditionally to Slack webhooks.
- **Nodes Involved:** `Is it a futures listing?`, `Also send to Slack?`, `Send futures alert to Discord`, `Send spot alert to Discord`, `Send to Slack`

- **Node Details:**
  - **Is it a futures listing?**
    - *Type and technical role:* `n8n-nodes-base.if` (Conditional router).
    - *Configuration choices:* Strict string comparison on listing type.
    - *Key expressions:* `={{ $json.type }}` equals `futures`.
    - *Input/Output:* Input from `Prepare alert fields`; outputs true to `Send futures alert to Discord`, false to `Send spot alert to Discord`.
    - *Edge cases:* Case mismatch on listing types.
  - **Also send to Slack?**
    - *Type and technical role:* `n8n-nodes-base.if` (Conditional router).
    - *Configuration choices:* Evaluates whether the Slack toggle is enabled in settings.
    - *Key expressions:* `={{ $('Your settings').first().json.send_to_slack }}` equals `true`.
    - *Input/Output:* Input from `Prepare alert fields`; outputs true to `Send to Slack`, false to an empty branch.
    - *Edge cases:* Boolean values passed as strings.
  - **Send futures alert to Discord**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (POSTs JSON payload to Discord webhook).
    - *Configuration choices:* HTTP POST method, custom JSON body with orange embed color (`15965202`).
    - *Key expressions:* Dynamic URL derived from `={{ $('Your settings').first().json.discord_webhook_url }}`; JSON stringification of embed structure.
    - *Input/Output:* Input from `Is it a futures listing?` (true branch); no outputs.
    - *Edge cases:* Webhook rate limiting (HTTP 429) from Discord.
  - **Send spot alert to Discord**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (POSTs JSON payload to Discord webhook).
    - *Configuration choices:* HTTP POST method, custom JSON body with green embed color (`3066993`).
    - *Key expressions:* Dynamic URL from settings; JSON stringification of embed structure.
    - *Input/Output:* Input from `Is it a futures listing?` (false branch); no outputs.
    - *Edge cases:* Webhook rate limiting (HTTP 429) from Discord.
  - **Send to Slack**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (POSTs form-urlencoded or JSON payload to Slack webhook).
    - *Configuration choices:* HTTP POST method, body parameters configured with Slack mrkdwn text.
    - *Key expressions:* Dynamic URL from `={{ $('Your settings').first().json.slack_webhook_url }}`; conditional prefix based on listing type.
    - *Input/Output:* Input from `Also send to Slack?` (true branch); no outputs.
    - *Edge cases:* Invalid Slack webhook URL configuration.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note** | `n8n-nodes-base.stickyNote` | Workflow documentation and configuration guide | None | None | ## Crypto exchange listing alerts for Discord, no account needed<br><br>Watches 10 crypto exchanges and posts every new token listing to a Discord channel as a color-coded embed, with an optional Slack copy.<br><br>### How it works<br><br>1. **Every 5 minutes** starts the run.<br>2. **Your settings** holds everything you might want to change: the Discord webhook, exchanges, spot or futures, a keyword filter, lookback window, headline language, and the Slack switch.<br>3. **Fetch new listings** calls a free, public, read-only JSON feed. No account, no API key, no token.<br>4. **Feed returned items?** ends an empty or failed run quietly instead of throwing.<br>5. **Only listings not seen before** remembers each permalink across executions, so a restart never re-announces a listing.<br>6. Two filters narrow the stream to the exchanges and keywords you care about.<br>7. **Prepare alert fields** picks the headline in your language and builds the fields both channels reuse.<br>8. Spot listings post as a green embed, futures listings as an orange one. Slack stays off until you enable it.<br><br>### Setup<br><br>1. In Discord open your channel settings, then Integrations, then Webhooks, and copy a webhook URL.<br>2. Open **Your settings** and paste it into `discord_webhook_url`.<br>3. Optional: set `send_to_slack` to `true` and paste a Slack incoming webhook URL into `slack_webhook_url`.<br>4. Save, then activate the workflow. No n8n credential is required anywhere.<br><br>### Customization tips<br><br>Narrow the feed with `exchanges` (`binance,okx`) or `listing_type` (`spot` or `futures`). Raise `lookback_days` up to 30 to backfill on the first run. Titles arrive in three languages, so set `language` to `en`, `zh` or `ko`. |
| **Sticky Note1** | `n8n-nodes-base.stickyNote` | Visual grouping for polling block | None | None | ## 1. Poll the public feed<br>A schedule trigger, your settings in one place, one HTTP request to a free read-only endpoint, and a guard so an empty response ends the run quietly. No credential is needed here. |
| **Sticky Note2** | `n8n-nodes-base.stickyNote` | Visual grouping for item processing block | None | None | ## 2. One item per listing, new ones only<br>Split Out expands the array. Remove Duplicates stores every permalink already handled. The two filters keep only the exchanges and keywords you chose in settings. |
| **Sticky Note3** | `n8n-nodes-base.stickyNote` | Visual grouping for field preparation block | None | None | ## 3. Build the fields and pick a branch<br>Compose the headline, symbols and link once, then decide where this listing goes. Spot and futures get different embed colors; Slack stays off until you enable it. |
| **Sticky Note4** | `n8n-nodes-base.stickyNote` | Visual grouping for delivery block | None | None | ## 4. Deliver<br>Three webhook posts, same fields. Discord gets an embed, Slack gets mrkdwn text. Swap any of them for Telegram, Gmail or a database without touching anything upstream. |
| **Every 5 minutes** | `n8n-nodes-base.scheduleTrigger` | Triggers workflow every 5 minutes | None | Your settings | |
| **Your settings** | `n8n-nodes-base.set` | Houses configuration variables | Every 5 minutes | Fetch new listings | |
| **Fetch new listings** | `n8n-nodes-base.httpRequest` | Calls the public token listing feed | Your settings | Feed returned items? | |
| **Feed returned items?** | `n8n-nodes-base.if` | Validates non-empty feed response | Fetch new listings | Split into one item per listing, Nothing new this run | |
| **Nothing new this run** | `n8n-nodes-base.noOp` | Terminal node for empty responses | Feed returned items? | None | |
| **Split into one item per listing** | `n8n-nodes-base.splitOut` | Flattens listings array | Feed returned items? | Only listings not seen before | |
| **Only listings not seen before** | `n8n-nodes-base.removeDuplicates` | Prevents duplicate announcements | Split into one item per listing | Matches your exchange filter? | |
| **Matches your exchange filter?** | `n8n-nodes-base.if` | Filters items by exchange ID | Only listings not seen before | Matches your keyword filter?, Skipped by your filters | |
| **Matches your keyword filter?** | `n8n-nodes-base.if` | Filters items by title keyword | Matches your exchange filter? | Prepare alert fields, Skipped by your filters | |
| **Skipped by your filters** | `n8n-nodes-base.noOp` | Terminal log node for filtered out items | Matches your exchange filter?, Matches your keyword filter? | None | |
| **Prepare alert fields** | `n8n-nodes-base.set` | Normalizes and computes alert attributes | Matches your keyword filter? | Is it a futures listing?, Also send to Slack? | |
| **Is it a futures listing?** | `n8n-nodes-base.if` | Splits route based on listing type | Prepare alert fields | Send futures alert to Discord, Send spot alert to Discord | |
| **Also send to Slack?** | `n8n-nodes-base.if` | Conditional router for Slack delivery | Prepare alert fields | Send to Slack | |
| **Send futures alert to Discord** | `n8n-nodes-base.httpRequest` | Posts orange embed to Discord webhook | Is it a futures listing? | None | |
| **Send spot alert to Discord** | `n8n-nodes-base.httpRequest` | Posts green embed to Discord webhook | Is it a futures listing? | None | |
| **Send to Slack** | `n8n-nodes-base.httpRequest` | Posts formatted markdown to Slack webhook | Also send to Slack? | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n:

1. **Create the Schedule Trigger:**
   - Add a **Schedule Trigger** node named `Every 5 minutes`.
   - Set the interval field to `Minutes` and interval value to `5`.

2. **Create the Settings Node:**
   - Add a **Set** node named `Your settings`.
   - Configure the manual assignments with the following string fields:
     - `discord_webhook_url`: `YOUR_DISCORD_WEBHOOK_URL`
     - `exchanges`: `""`
     - `listing_type`: `""`
     - `keyword`: `""`
     - `lookback_days`: `"1"`
     - `language`: `"en"`
     - `send_to_slack`: `"false"`
     - `slack_webhook_url`: `""`
   - Enable `Include Other Fields`.
   - *Connection:* Connect `Every 5 minutes` output to `Your settings`.

3. **Create the HTTP Request Node:**
   - Add an **HTTP Request** node named `Fetch new listings`.
   - Set Method to `GET`, URL to `https://tokenearly.com/api/public/listings.json`.
   - Under Options, enable `neverError` (Response -> Response -> neverError: true).
   - Add query parameters:
     - Name: `days`, Value: `={{ $json.lookback_days }}`
     - Name: `type`, Value: `={{ $json.listing_type }}`
     - Name: `limit`, Value: `100`
   - *Connection:* Connect `Your settings` output to `Fetch new listings`.

4. **Create the Response Guard Logic:**
   - Add an **If** node named `Feed returned items?`.
   - Set condition: Left Value `={{ $json.count }}`, Operator `Greater Than` (number), Right Value `0`.
   - Add a **No-Op** node named `Nothing new this run`.
   - *Connections:* Connect `Fetch new listings` to `Feed returned items?`. Connect the `false` output of `Feed returned items?` to `Nothing new this run`.

5. **Create the Item Processing & Deduplication Nodes:**
   - Add a **Split Out** node named `Split into one item per listing`. Set `Field to Split Out` to `items`.
     - *Connection:* Connect true output of `Feed returned items?` to this node.
   - Add a **Remove Duplicates** node named `Only listings not seen before`.
     - Set Logic to `Remove items with already seen key values`, Operation to `Remove items seen in previous executions`.
     - Set Dedupe Value to `={{ $json.permalink }}`.
     - *Connection:* Connect `Split into one item per listing` output to this node.

6. **Create the Filtering Nodes:**
   - Add an **If** node named `Matches your exchange filter?`.
     - Set condition: Left Value `={{ $('Your settings').first().json.exchanges || $json.exchange }}`, Operator `Contains`, Right Value `={{ $json.exchange }}`.
     - *Connection:* Connect `Only listings not seen before` output to this node.
   - Add an **If** node named `Matches your keyword filter?`.
     - Set condition: Left Value `={{ ($json.title.en || '') + ' ' + ($json.title.zh || '') + ' ' + ($json.title.ko || '') }}`, Operator `Contains`, Right Value `={{ $('Your settings').first().json.keyword }}`.
     - *Connection:* Connect the `true` output of `Matches your exchange filter?` to this node.
   - Add a **No-Op** node named `Skipped by your filters`.
     - *Connections:* Connect the `false` output of both `Matches your exchange filter?` and `Matches your keyword filter?` to this node.

7. **Create the Field Preparation Node:**
   - Add a **Set** node named `Prepare alert fields`.
   - Configure manual assignments:
     - `headline`: `={{ $json.title[$('Your settings').first().json.language] || $json.title.en || $json.title.zh || $json.title.ko }}`
     - `symbols_text`: `={{ ($json.symbols || []).join(', ') }}`
     - `type_label`: `={{ $json.type === 'futures' ? 'Futures' : 'Spot' }}`
     - `minutes_old`: `={{ Math.max(0, Math.round((Date.now() - new Date($json.published_at).getTime()) / 60000)) }}`
     - `link`: `={{ $json.source_url || $json.permalink }}`
   - *Connection:* Connect the `true` output of `Matches your keyword filter?` to `Prepare alert fields`.

8. **Create the Delivery Routing and Webhook Nodes:**
   - Add an **If** node named `Is it a futures listing?`.
     - Set condition: Left Value `={{ $json.type }}`, Operator `Equals` (string, case-sensitive), Right Value `futures`.
     - *Connection:* Connect `Prepare alert fields` output to this node.
   - Add an **If** node named `Also send to Slack?`.
     - Set condition: Left Value `={{ $('Your settings').first().json.send_to_slack }}`, Operator `Equals` (string, case-sensitive), Right Value `true`.
     - *Connection:* Connect `Prepare alert fields` output to this node as an additional output connection.
   - Add an **HTTP Request** node named `Send futures alert to Discord`:
     - Method: `POST`, URL: `={{ $('Your settings').first().json.discord_webhook_url }}`
     - Specify Body: `JSON`, JSON Body: construct using the embed schema with color `15965202` and title referencing `$json.exchange_name + ' · Futures listing'`.
     - *Connection:* Connect true output of `Is it a futures listing?` here.
   - Add an **HTTP Request** node named `Send spot alert to Discord`:
     - Method: `POST`, URL: `={{ $('Your settings').first().json.discord_webhook_url }}`
     - Specify Body: `JSON`, JSON Body: construct using the embed schema with color `3066993` and title referencing `$json.exchange_name + ' · Spot listing'`.
     - *Connection:* Connect false output of `Is it a futures listing?` here.
   - Add an **HTTP Request** node named `Send to Slack`:
     - Method: `POST`, URL: `={{ $('Your settings').first().json.slack_webhook_url }}`
     - Specify Body: `Using Body Parameters`, Parameter Name: `text`, Value: formatted Slack mrkdwn string.
     - *Connection:* Connect true output of `Also send to Slack?` here.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Public API Data Source | [Tokenearly Public API](https://tokenearly.com/api/public/listings.json) |
| Authentication Requirement | None required (Public, read-only JSON feed) |