Send Daily 4K Bluray Preorder Updates from Blu-ray.com to Discord

https://n8nworkflows.xyz/workflows/send-daily-4k-bluray-preorder-updates-from-blu-ray-com-to-discord-6830


# Send Daily 4K Bluray Preorder Updates from Blu-ray.com to Discord

### 1. Workflow Overview

This workflow automates the daily retrieval and posting of new 4K Blu-ray preorder updates from Blu-ray.com into a Discord channel. It is designed to run automatically every night at 11 PM Eastern Time, scraping the Blu-ray.com new preorders page, filtering preorder items relevant to the current date, formatting these into a message, and sending the update to a Discord channel via webhook.

Logical blocks in the workflow are:

- **1.1 Trigger Block:** Initiates workflow execution either manually or via scheduled daily trigger.
- **1.2 Date Formatting Block:** Formats the current date according to the Eastern Time Zone to match the date format used on the Blu-ray.com site.
- **1.3 Web Scraping Block:** Downloads the HTML content of the Blu-ray.com new preorders page and extracts relevant HTML sections containing preorder links and dates.
- **1.4 Data Filtering Block:** Processes the extracted HTML snippets to isolate preorder items that correspond to today’s date.
- **1.5 Message Formatting Block:** Converts the filtered preorder items into a Markdown-formatted message with titles and URLs.
- **1.6 Discord Posting Block:** Sends the formatted message to a designated Discord channel using a webhook.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

- **Overview:** Starts the workflow either manually via a user click or automatically on a daily schedule at 11 PM ET.
- **Nodes Involved:** `When clicking ‘Execute workflow’`, `Schedule Trigger`
- **Node Details:**

  - **When clicking ‘Execute workflow’**
    - Type: Manual Trigger
    - Role: Allows manual initiation for testing or on-demand updates.
    - Configuration: No parameters.
    - Inputs: None
    - Outputs: Connected to `Format Todays Date`.
    - Edge Cases: None typical; manual workflow execution requires user action.

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Automatically triggers the workflow daily at 23:00 hours (11 PM).
    - Configuration: Trigger set to run every day at hour 23.
    - Inputs: None
    - Outputs: Connected to `Format Todays Date`.
    - Edge Cases: Timezone interpretation depends on n8n server settings; requires adjustment if server time differs from ET.
    - Sticky Note: Reminder to update timezone settings appropriately.

#### 2.2 Date Formatting Block

- **Overview:** Formats today’s date into a textual string that matches the Blu-ray.com date format for comparison.
- **Nodes Involved:** `Format Todays Date`
- **Node Details:**

  - **Format Todays Date**
    - Type: Code Node (JavaScript)
    - Role: Produces today’s date formatted as "Month Day, Year" in English (US), in the America/New_York timezone.
    - Configuration: Uses `toLocaleDateString` with options for year, month (long), day (2-digit), and fixed timezone America/New_York.
    - Input: From trigger nodes (`Schedule Trigger` or manual trigger).
    - Output: JSON with key `formattedDate` holding the formatted date string.
    - Edge Cases: If the n8n node environment lacks timezone support, formatting may fail or produce incorrect results.
    - Sticky Note: Advises user to update the timezone to match their desired locale.

#### 2.3 Web Scraping Block

- **Overview:** Retrieves the new preorder listings page and extracts relevant HTML content for further processing.
- **Nodes Involved:** `Scrape Page`, `Get Hyperlinks`
- **Node Details:**

  - **Scrape Page**
    - Type: HTTP Request
    - Role: Fetches the HTML content from the Blu-ray.com new preorders URL.
    - Configuration: GET request to `https://www.blu-ray.com/movies/movies.php?show=newpreorders`.
    - Input: From `Format Todays Date`.
    - Output: Raw HTML content of the webpage.
    - Edge Cases: Network errors, site downtime, or changes in page URL or structure may cause failure or incorrect data.
  
  - **Get Hyperlinks**
    - Type: HTML Extractor
    - Role: Parses the HTML response and extracts an array of HTML snippets containing either date headers or preorder entries.
    - Configuration: CSS selector targets `td[width=728]>h3` and `td[width=728]>div div` elements, extracting HTML content as strings.
    - Input: From `Scrape Page`.
    - Output: JSON array under key `links` containing extracted HTML snippets.
    - Edge Cases: Changes to site HTML structure or selectors may cause extraction failure or empty results.

#### 2.4 Data Filtering Block

- **Overview:** Processes the extracted HTML snippets to collect preorder items only for the current date.
- **Nodes Involved:** `Filter Todays Items`
- **Node Details:**

  - **Filter Todays Items**
    - Type: Code Node (JavaScript)
    - Role: Iterates over the `links` array; recognizes date headers by validating if a string is a date; if the current date matches today’s formatted date, collects the preorder items under that date.
    - Configuration: JavaScript function `isValidDate` checks if a string is a valid date. Uses the date header to assign current active date and accumulates preorder items belonging to today.
    - Input: Receives `links` array from `Get Hyperlinks` and `formattedDate` from `Format Todays Date`.
    - Output: JSON object with `items` array containing preorder HTML snippets for today.
    - Edge Cases: If date parsing fails, or page structure changes, filtering may include wrong items or none.
    - Sticky Note: Optional note on adjusting message formatting downstream.

#### 2.5 Message Formatting Block

- **Overview:** Converts filtered preorder items into a Markdown message with clickable links for Discord.
- **Nodes Involved:** `Format Message`
- **Node Details:**

  - **Format Message**
    - Type: Code Node (JavaScript)
    - Role: Parses each preorder item’s HTML snippet to extract `href` and `title` attributes and formats them as Markdown links.
    - Configuration: Custom JavaScript function `extractTitleAndHref` extracts URL and title from the HTML string.
    - Input: JSON array `items` from `Filter Todays Items`.
    - Output: JSON object with single key `message` containing the formatted string starting with "*New 4k Preorders Today!*".
    - Edge Cases: Malformed HTML snippets or missing attributes may cause extraction errors or incomplete messages.
    - Sticky Note: Optional note about adjusting message formatting.

#### 2.6 Discord Posting Block

- **Overview:** Sends the formatted message into a Discord channel via webhook.
- **Nodes Involved:** `Post to Discord`
- **Node Details:**

  - **Post to Discord**
    - Type: Discord Node
    - Role: Posts the message content to Discord channel using webhook authentication.
    - Configuration: Message content bound dynamically to the output of `Format Message` node (`{{$json.message}}`).
    - Credentials: Uses stored Discord Webhook API credentials.
    - Input: From `Format Message`.
    - Output: None (terminal node).
    - Edge Cases: Webhook invalidation, network errors, Discord API rate limits, or malformed message content.
    - Sticky Note: Reminder to connect to correct Discord webhook.

---

### 3. Summary Table

| Node Name                 | Node Type             | Functional Role                         | Input Node(s)                   | Output Node(s)           | Sticky Note                                                                                 |
|---------------------------|-----------------------|---------------------------------------|--------------------------------|--------------------------|---------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger        | Manual start of workflow               | None                           | Format Todays Date        |                                                                                             |
| Schedule Trigger          | Schedule Trigger      | Automatic daily start at 11 PM ET      | None                           | Format Todays Date        | Note: This is scheduled to run every day at 11pm.                                          |
| Format Todays Date        | Code                  | Format current date in ET timezone     | When clicking ‘Execute workflow’, Schedule Trigger | Scrape Page             | To Do: Update this to be set to your correct TimeZone.                                     |
| Scrape Page               | HTTP Request          | Download Blu-ray.com preorder page     | Format Todays Date             | Get Hyperlinks            |                                                                                             |
| Get Hyperlinks            | HTML Extractor        | Extract preorder links and date headers | Scrape Page                   | Filter Todays Items       |                                                                                             |
| Filter Todays Items       | Code                  | Filter preorder items for today’s date | Get Hyperlinks                | Format Message            | Optional: Adjust the formatting of the message that gets posted to Discord.                 |
| Format Message            | Code                  | Format preorder items into Markdown    | Filter Todays Items            | Post to Discord           | Optional: Adjust the formatting of the message that gets posted to Discord.                 |
| Post to Discord           | Discord               | Send message to Discord via webhook    | Format Message                | None                     | To Do: Connect to your Discord channel webhook here:                                       |
| Sticky Note               | Sticky Note           | Reminder to update timezone             | None                         | None                     | To Do: Update this to be set to your correct TimeZone.                                     |
| Sticky Note1              | Sticky Note           | Reminder to connect Discord webhook     | None                         | None                     | To Do: Connect to your Discord channel webhook here:                                       |
| Sticky Note2              | Sticky Note           | Optional message formatting note        | None                         | None                     | Optional: Adjust the formatting of the message that gets posted to Discord.                 |
| Sticky Note3              | Sticky Note           | Schedule reminder                       | None                         | None                     | Note: This is scheduled to run every day at 11pm.                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: Manual Trigger
   - Name: `When clicking ‘Execute workflow’`
   - No parameters.

2. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Name: `Schedule Trigger`
   - Parameters:
     - Rule: Interval trigger at hour 23 (11 PM daily).
   - Connect both `When clicking ‘Execute workflow’` and `Schedule Trigger` outputs to the next node (`Format Todays Date`).

3. **Create Code Node to Format Today’s Date**
   - Type: Code (JavaScript)
   - Name: `Format Todays Date`
   - Parameters:
     - JavaScript code:
       ```js
       const today = new Date();
       const formatted = today.toLocaleDateString('en-US', {
         year: 'numeric',
         month: 'long',
         day: '2-digit',
         timeZone: 'America/New_York'
       });
       return [{ json: { formattedDate: formatted } }];
       ```
   - Connect inputs from both triggers.
   - Important: Set timezone to `America/New_York` or adjust as needed.

4. **Create HTTP Request Node to Scrape Blu-ray.com**
   - Type: HTTP Request
   - Name: `Scrape Page`
   - Parameters:
     - HTTP Method: GET
     - URL: `https://www.blu-ray.com/movies/movies.php?show=newpreorders`
     - No authentication or special headers.
   - Connect input from `Format Todays Date`.

5. **Create HTML Extract Node to Get Links**
   - Type: HTML Extract
   - Name: `Get Hyperlinks`
   - Parameters:
     - Operation: extractHtmlContent
     - Extraction Values:
       - Key: `links`
       - CSS Selector: `td[width=728]>h3, td[width=728]>div div`
       - Return Array: true
       - Return Value: html
   - Connect input from `Scrape Page`.

6. **Create Code Node to Filter Today’s Items**
   - Type: Code (JavaScript)
   - Name: `Filter Todays Items`
   - Parameters:
     - JavaScript code:
       ```js
       function isValidDate(str) {
         const date = new Date(str);
         return !isNaN(date.getTime());
       }

       let activeDate = "";
       let preOrders = [];

       for (const item of $input.first().json.links) {
         if (isValidDate(item)) {
           activeDate = item;
         } else {
           if (activeDate === $('Format Todays Date').first().json.formattedDate) {
             preOrders.push(item.substring(item.indexOf("]") + 1));
           }
         }
       }

       return [{
         json: {
           items: preOrders
         }
       }];
       ```
   - Connect input from `Get Hyperlinks`.

7. **Create Code Node to Format Message**
   - Type: Code (JavaScript)
   - Name: `Format Message`
   - Parameters:
     - JavaScript code:
       ```js
       function extractTitleAndHref(htmlString) {
         const hrefStart = htmlString.indexOf('href="') + 6;
         const hrefEnd = htmlString.indexOf('"', hrefStart);
         const href = htmlString.substring(hrefStart, hrefEnd);

         const titleStart = htmlString.indexOf('title="') + 7;
         const titleEnd = htmlString.indexOf('"', titleStart);
         const title = htmlString.substring(titleStart, titleEnd);

         return { title, href };
       }

       let message = "*New 4k Preorders Today!*\n";

       for (const link of $input.first().json.items) {
         let anchor = extractTitleAndHref(link);
         message += `[${anchor.title}](${anchor.href})\n`;
       }

       return [{ json: { message: message } }];
       ```
   - Connect input from `Filter Todays Items`.

8. **Create Discord Node to Post Message**
   - Type: Discord
   - Name: `Post to Discord`
   - Parameters:
     - Content: `={{ $json.message }}`
     - Authentication: Webhook
   - Credentials:
     - Connect your Discord webhook credentials (OAuth2 or webhook URL).
   - Connect input from `Format Message`.

9. **Add Sticky Notes (Optional but Recommended)**
   - Add notes near `Format Todays Date` about timezone adjustment.
   - Add note near `Post to Discord` to remind connecting correct webhook.
   - Add optional note near `Filter Todays Items` or `Format Message` about message formatting customization.
   - Add note near `Schedule Trigger` about scheduled run time.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                     |
|----------------------------------------------------------------------------------------------|----------------------------------------------------|
| This workflow posts daily updates at 11 PM ET; adjust schedule trigger if running in another timezone. | Scheduling and timezone configuration               |
| Discord webhook credentials must be preconfigured in n8n credentials manager for the Discord node. | Discord webhook setup                               |
| Blu-ray.com page structure may change, requiring updates to CSS selectors in `Get Hyperlinks`. | HTML extraction maintenance                         |
| JavaScript date parsing depends on consistent date formats on source site; errors may cause no items posted. | Data filtering reliability                          |
| Optional formatting of Discord message can be customized in the `Format Message` code node. | Message appearance customization                     |

---

**Disclaimer:**  
The text provided is generated exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.