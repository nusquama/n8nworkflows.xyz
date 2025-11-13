Skip Tracing: Extract Phones & Emails from TruePeopleSearch with Zyte API

https://n8nworkflows.xyz/workflows/skip-tracing--extract-phones---emails-from-truepeoplesearch-with-zyte-api-6886


# Skip Tracing: Extract Phones & Emails from TruePeopleSearch with Zyte API

### 1. Workflow Overview

This n8n workflow automates skip tracing by extracting phone numbers, emails, and addresses from TruePeopleSearch.com using the Zyte API for web scraping. It targets use cases such as real estate, debt collection, and investigative research where enriched contact data is needed for entities by searching profiles and their relatives on TruePeopleSearch.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Filtering:** Receive input rows from a Google Sheet, filter out already processed entries.
- **1.2 Search Query and Profile Matching:** Make HTTP requests to Zyte API to load search results pages, then locate and extract the exact person’s profile URL.
- **1.3 Primary Profile Scraping:** Scrape the matched personal profile page for detailed info including phones, emails, age, name, and address.
- **1.4 Data Update and Error Handling:** Update Google Sheet rows with the extracted data or error statuses.
- **1.5 Fallback Relative Profile Scraping:** If no primary phone numbers found, scrape information from a relative’s profile as a fallback.
- **1.6 Relative Profile Data Extraction and Update:** Scrape relative’s profile page details and update sheets accordingly.

Sticky notes provide contextual guidance on each major step and Zyte API setup instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

**Overview:**  
Fetch rows from Google Sheets and filter out those that have already been processed or contain invalid URLs to avoid duplication.

**Nodes Involved:**  
- Webhook  
- Get row(s) in sheet  
- Add conditions to filter already processed rows  
- Limit number of rows to process

**Node Details:**

- **Webhook**  
  - Type: Webhook (Trigger)  
  - Receives incoming HTTP requests to start the workflow.  
  - No input connections, outputs to "Get row(s) in sheet".  
  - Possible failure: HTTP connection issues or unauthorized access.

- **Get row(s) in sheet**  
  - Type: Google Sheets - Read  
  - Reads rows from a specified Google Sheet (configured with OAuth2 credentials).  
  - Fetches data from a sheet named "Sheet1" in a specific document.  
  - Inputs from Webhook, outputs to filter node.  
  - Failure modes: Auth errors, sheet not found, API quota limits.

- **Add conditions to filter already processed rows**  
  - Type: Filter  
  - Filters rows where SearchURL is not empty, PersonURL is not "Not Found", and PersonURL does not contain "http" (indicating unprocessed or invalid rows).  
  - Input: Google Sheets rows  
  - Output: Filtered rows for processing  
  - Edge cases: Incorrect data formats or empty fields may skip valid rows.

- **Limit number of rows to process**  
  - Type: Limit  
  - Limits the number of rows processed per execution (default limit not explicitly set, so defaults apply).  
  - Input: Filtered rows  
  - Output: Limited rows to further processing  
  - Failure: None typical, but improper limits may cause workflow overload.

---

#### 2.2 Search Query and Profile Matching

**Overview:**  
Send HTTP request to Zyte API to get the HTML of the search results page for each row’s SearchURL. Then, parse the HTML to find the best matching profile URL based on first and last names.

**Nodes Involved:**  
- Make HTTP Request for SearchURL  
- Find the matching person profile from search results  
- If found  
- Update row in sheet with PersonURL  
- Update row in sheet with errors

**Node Details:**

- **Make HTTP Request for SearchURL**  
  - Type: HTTP Request  
  - Uses Zyte API with HTTP Basic Auth credentials to scrape the SearchURL page, requesting browser-rendered HTML to bypass bot protections.  
  - Input: Limited rows from previous block  
  - Output: HTML content for parsing  
  - Failures: HTTP errors, Zyte API key issues, network timeouts.

- **Find the matching person profile from search results**  
  - Type: Code (JavaScript)  
  - Parses the HTML response to identify profile cards with names and links.  
  - Uses regex to extract profile URLs and names.  
  - Searches for exact matches by first and last name, then first name only, then last name only, in that order.  
  - Returns the matched person profile URL or "Not Found".  
  - Input: HTML from Zyte API  
  - Output: `personUrl` string  
  - Edge cases: No matches found, malformed HTML, regex failures.

- **If found**  
  - Type: If  
  - Checks if `personUrl` contains "http" to verify a valid profile URL was found.  
  - Routes to update nodes accordingly.

- **Update row in sheet with PersonURL**  
  - Type: Google Sheets - Update  
  - Updates the Google Sheet row with the found `PersonURL` and marks Name as "Running..." to indicate processing.  
  - Input: If found (true branch)  
  - Output: Proceeds to "Make HTTP Request for PersonURL" node.

- **Update row in sheet with errors**  
  - Type: Google Sheets - Update  
  - Updates the row with error status if no valid person URL found.  
  - Input: If found (false branch)  
  - Output: Ends or error handling.

---

#### 2.3 Primary Profile Scraping

**Overview:**  
Scrape the matched personal profile page to extract detailed personal information such as name, age, phone numbers, emails, and current address using multiple code nodes with regex parsing.

**Nodes Involved:**  
- Make HTTP Request for PersonURL  
- Set html to scrape data  
- Name  
- Age  
- Primary Phone  
- Other numbers  
- Emails  
- Current Address  
- If phone numbers not found  
- Update Scraped Data in sheet  
- Update Errors in Google Sheets

**Node Details:**

- **Make HTTP Request for PersonURL**  
  - Type: HTTP Request  
  - Uses Zyte API with HTTP Basic Auth to scrape the matched person profile URL page.  
  - Input: PersonURL from previous block  
  - Output: Browser-rendered HTML  
  - Failures: HTTP errors, Zyte API issues, timeouts.

- **Set html to scrape data**  
  - Type: Set  
  - Assigns the browser HTML from the previous node into a new JSON field `html` for easier extraction downstream.  
  - Input: HTTP Request result  
  - Output: Passes to multiple scraping nodes.

- **Name**  
  - Type: Code  
  - Extracts the person’s name using a regex matching `<h1 class="oh1">...</h1>`.  
  - Returns `name` or "Not Found".  
  - Input: HTML from Set node.

- **Age**  
  - Type: Code  
  - Extracts age and birth year with regex matching `<span> Age X, Born Month Year </span>`.  
  - Returns combined string `ageWithDob` or "Not Found".  
  - Input: HTML.

- **Primary Phone**  
  - Type: Code  
  - Extracts the primary phone number from `<span itemprop="telephone">...</span>`.  
  - Returns `primaryPhone` or "Not Found".  
  - Input: HTML.

- **Other numbers**  
  - Type: Code  
  - Extracts phone numbers and their labels (e.g., cell, home) using regex with global matching, concatenating results.  
  - Returns a string with detailed numbers or "Not Found".  
  - Input: HTML.

- **Emails**  
  - Type: Code  
  - Extracts all email addresses using regex, filters out support emails, deduplicates, and joins with commas.  
  - Returns `emails`.  
  - Input: HTML.

- **Current Address**  
  - Type: Code  
  - Extracts current address by matching an `<a>` tag with `data-link-to-more="address"` attribute, strips HTML tags.  
  - Returns `currentAddress` or "Not Found".  
  - Input: HTML.

- **If phone numbers not found**  
  - Type: If  
  - Checks if "Other numbers" result is "Not Found". If true, triggers fallback scraping of relative’s profile.  
  - If false, proceeds to update scraped data.

- **Update Scraped Data in sheet**  
  - Type: Google Sheets - Update  
  - Updates the Google Sheet row with extracted primary profile data (name, age, phones, emails, address).  
  - Input: If phone numbers found (false branch).  
  - Output: Ends or waits for next input.

- **Update Errors in Google Sheets**  
  - Type: Google Sheets - Update  
  - Updates Google Sheet row to indicate an error if HTTP request for person profile fails or scraping issues occur.  
  - Input: From HTTP Request for PersonURL node on error.

---

#### 2.4 Fallback Relative Profile Scraping

**Overview:**  
If no phone numbers are found in the primary profile, the workflow scrapes the profile of a relative as a fallback to gather contact info.

**Nodes Involved:**  
- RelativeProfile  
- Update Scraped data with RelativeURL into Sheets  
- Check if RelativeURL exists  
- Make HTTP Request for RelativeURL  
- Set relativeprofile page html to scrape data  
- RelativeName  
- RelativeAge  
- RelativePrimaryPhone  
- Relative Other numbers  
- RelativeEmails  
- Relative Current Address  
- Their RelativeProfile  
- Update Relative scraped data to sheets  
- Update errors Google Sheets  
- Update row in sheet with http error  
- If phone numbers not found (false branch)

**Node Details:**

- **RelativeProfile**  
  - Type: Code  
  - Extracts relative’s profile URL from the primary profile HTML.  
  - Returns `relativeProfileUrl` or "Not Found".  
  - Input: Primary profile HTML.

- **Update Scraped data with RelativeURL into Sheets**  
  - Type: Google Sheets - Update  
  - Updates the sheet row with the relative's profile URL and marks "Relative Name" as "Running...".  
  - Input: RelativeProfile output.

- **Check if RelativeURL exists**  
  - Type: If  
  - Checks if `relativeProfileUrl` is not "Not Found".  
  - True branch triggers HTTP request to scrape relative profile.

- **Make HTTP Request for RelativeURL**  
  - Type: HTTP Request  
  - Uses Zyte API to scrape the relative’s profile page.  
  - Input: RelativeURL from previous node.  
  - Output: HTML content or error.

- **Set relativeprofile page html to scrape data**  
  - Type: Set  
  - Sets the browser HTML from relative profile HTTP request into `html` field for scraping.

- **RelativeName, RelativeAge, RelativePrimaryPhone, Relative Other numbers, RelativeEmails, Relative Current Address**  
  - Type: Code (each)  
  - Each extracts specific details from the relative’s profile HTML using regex similar to the primary profile nodes.  
  - Return respective fields or "Not Found".

- **Their RelativeProfile**  
  - Type: Code  
  - Extracts a further relative profile URL from the relative’s profile page if available.  
  - Returns `relativeProfileUrl`.

- **Update Relative scraped data to sheets**  
  - Type: Google Sheets - Update  
  - Updates the sheet with all extracted relative profile details including name, age, phones, emails, and addresses.  
  - Input: Aggregated relative profile data.

- **Update errors Google Sheets**  
  - Type: Google Sheets - Update  
  - Updates sheet with error info if HTTP request for relative profile fails or scraping errors occur.

- **Update row in sheet with http error**  
  - Type: Google Sheets - Update  
  - Specifically updates row indicating HTTP error with relative profile scraping.

---

### 3. Summary Table

| Node Name                             | Node Type                | Functional Role                                             | Input Node(s)                                | Output Node(s)                               | Sticky Note                                                                                                      |
|-------------------------------------|--------------------------|-------------------------------------------------------------|----------------------------------------------|----------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Webhook                             | Webhook                  | Start workflow on HTTP request                              | None                                         | Get row(s) in sheet                          | # Step 1: Search & Match Entity \nPulls data from Google Sheet, searches truepeoplesearch.com by address, and matches entity by name. |
| Get row(s) in sheet                 | Google Sheets             | Read rows from Google Sheet                                 | Webhook                                      | Add conditions to filter already processed rows |                                                                                                                  |
| Add conditions to filter already processed rows | Filter                   | Filter out rows already processed or invalid                | Get row(s) in sheet                          | Limit number of rows to process              |                                                                                                                  |
| Limit number of rows to process     | Limit                    | Limit number of rows processed                              | Add conditions to filter already processed rows | Make HTTP Request for SearchURL              |                                                                                                                  |
| Make HTTP Request for SearchURL     | HTTP Request             | Scrape search result page via Zyte API                      | Limit number of rows to process               | Find the matching person profile from search results, Update Sheet on Errors | # Zyte API Setup Guide\n- Visit zyte.com\n- Sign up with email\n- Use Zyte API Playground to get API key...        |
| Find the matching person profile from search results | Code                     | Parse search results HTML to find matching profile URL      | Make HTTP Request for SearchURL               | If found, Update Sheet on Errors             |                                                                                                                  |
| If found                           | If                       | Check if person profile URL found                            | Find the matching person profile from search results | Update row in sheet with PersonURL, Update row in sheet with errors |                                                                                                                  |
| Update row in sheet with PersonURL  | Google Sheets             | Update sheet with found profile URL                          | If found (true branch)                        | Make HTTP Request for PersonURL               |                                                                                                                  |
| Update row in sheet with errors     | Google Sheets             | Update sheet on no match or errors                           | If found (false branch)                       | None                                         |                                                                                                                  |
| Make HTTP Request for PersonURL     | HTTP Request             | Scrape person profile page via Zyte API                     | Update row in sheet with PersonURL            | Set html to scrape data, Update Errors in Google Sheets |                                                                                                                  |
| Set html to scrape data             | Set                      | Assign HTML to variable for scraping                         | Make HTTP Request for PersonURL               | Name, Age, Primary Phone, Other numbers, Emails, Current Address |                                                                                                                  |
| Name                               | Code                     | Extract name from profile HTML                               | Set html to scrape data                       | Age                                          |                                                                                                                  |
| Age                                | Code                     | Extract age and birth year from profile HTML                | Name                                          | Primary Phone                                 |                                                                                                                  |
| Primary Phone                      | Code                     | Extract primary phone number                                 | Age                                           | Other numbers                                 |                                                                                                                  |
| Other numbers                      | Code                     | Extract all phone numbers and labels                         | Primary Phone                                 | Emails                                        |                                                                                                                  |
| Emails                             | Code                     | Extract emails, filter and deduplicate                       | Other numbers                                 | Current Address                               |                                                                                                                  |
| Current Address                   | Code                     | Extract current address                                      | Emails                                        | If phone numbers not found                     |                                                                                                                  |
| If phone numbers not found          | If                       | Check if phone numbers found, trigger fallback if not       | Current Address                               | RelativeProfile, Update Scraped Data in sheet | # Step 3: Fallback to Relative  \nIf no phone found, scrapes personal info from a relative’s profile page as fallback. |
| Update Scraped Data in sheet        | Google Sheets             | Update sheet with scraped primary profile data              | If phone numbers not found (false branch)    | None                                         |                                                                                                                  |
| Update Errors in Google Sheets      | Google Sheets             | Update sheet on errors during person profile scraping       | Make HTTP Request for PersonURL (on error)   | None                                         |                                                                                                                  |
| RelativeProfile                    | Code                     | Extract relative profile URL from primary profile HTML      | If phone numbers not found (true branch)     | Update Scraped data with RelativeURL into Sheets |                                                                                                                  |
| Update Scraped data with RelativeURL into Sheets | Google Sheets             | Update sheet with relative profile URL                      | RelativeProfile                               | Check if RelativeURL exists                    |                                                                                                                  |
| Check if RelativeURL exists         | If                       | Check if relative profile URL exists                         | Update Scraped data with RelativeURL into Sheets | Make HTTP Request for RelativeURL, Update errors Google Sheets |                                                                                                                  |
| Make HTTP Request for RelativeURL   | HTTP Request             | Scrape relative profile page via Zyte API                   | Check if RelativeURL exists (true branch)    | Set relativeprofile page html to scrape data, Update row in sheet with http error |                                                                                                                  |
| Update errors Google Sheets         | Google Sheets             | Update sheet on errors during relative profile scraping     | Check if RelativeURL exists (false branch)   | None                                         |                                                                                                                  |
| Set relativeprofile page html to scrape data | Set                      | Assign relative profile HTML to variable                     | Make HTTP Request for RelativeURL             | RelativeName                                  |                                                                                                                  |
| RelativeName                      | Code                     | Extract name from relative profile HTML                      | Set relativeprofile page html to scrape data | RelativeAge                                   |                                                                                                                  |
| RelativeAge                       | Code                     | Extract age and birth year from relative profile HTML       | RelativeName                                  | RelativePrimaryPhone                           |                                                                                                                  |
| RelativePrimaryPhone              | Code                     | Extract relative’s primary phone number                      | RelativeAge                                   | Relative Other numbers                         |                                                                                                                  |
| Relative Other numbers            | Code                     | Extract all phone numbers and labels from relative profile  | RelativePrimaryPhone                          | RelativeEmails                                 |                                                                                                                  |
| RelativeEmails                   | Code                     | Extract emails from relative profile                         | Relative Other numbers                        | Relative Current Address                       |                                                                                                                  |
| Relative Current Address         | Code                     | Extract current address from relative profile               | RelativeEmails                                | Their RelativeProfile                          |                                                                                                                  |
| Their RelativeProfile            | Code                     | Extract further relative profile URL from relative profile  | Relative Current Address                      | Update Relative scraped data to sheets         |                                                                                                                  |
| Update Relative scraped data to sheets | Google Sheets             | Update sheet with all extracted relative profile details    | Their RelativeProfile                         | None                                         |                                                                                                                  |
| Update row in sheet with http error | Google Sheets             | Update sheet row with HTTP error status for relative profile | Make HTTP Request for RelativeURL (on error) | None                                         |                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `5e78fc90-9461-4b44-bc64-8177c975d778` (or custom)  
   - Purpose: Entry point to trigger workflow.

2. **Add Google Sheets Node (Get row(s) in sheet)**  
   - Operation: Read rows from Google Sheet  
   - Document ID: Your target Google Sheet ID  
   - Sheet Name: e.g., "Sheet1"  
   - Credentials: Google Sheets OAuth2 account  
   - Connect from Webhook.

3. **Add Filter Node (Add conditions to filter already processed rows)**  
   - Conditions:  
     - `SearchURL` not empty  
     - `PersonURL` not equal to "Not Found"  
     - `PersonURL` does not contain "http"  
   - Connect from Google Sheets node.

4. **Add Limit Node (Limit number of rows to process)**  
   - Default limit or set desired max rows per run  
   - Connect from Filter node.

5. **Add HTTP Request Node (Make HTTP Request for SearchURL)**  
   - URL: `https://api.zyte.com/v1/extract`  
   - Method: POST  
   - Body (JSON): `{ "url": "{{ $json.SearchURL }}", "browserHtml": true }`  
   - Authentication: HTTP Basic Auth with Zyte API user ID as username, blank password  
   - Header: Content-Type: application/json  
   - Connect from Limit node.

6. **Add Code Node (Find the matching person profile from search results)**  
   - JavaScript to parse HTML and find profile URL by matching first and last names in sheet data  
   - Input: HTTP request output  
   - Output: `personUrl` string  
   - Connect from HTTP Request for SearchURL.

7. **Add If Node (If found)**  
   - Condition: Check if `personUrl` contains "http"  
   - Connect from Code node.

8. **Add Google Sheets Node (Update row in sheet with PersonURL)**  
   - Operation: Update row  
   - Columns to update: `PersonURL` with found URL, `Name` set to "Running..."  
   - Matching column: `row_number`  
   - Connect from If node (true branch).

9. **Add Google Sheets Node (Update row in sheet with errors)**  
   - Operation: Update row  
   - Columns: `PersonURL` set to error or "Not Found"  
   - Connect from If node (false branch).

10. **Add HTTP Request Node (Make HTTP Request for PersonURL)**  
    - Same Zyte API setup as before  
    - URL: `personUrl` from previous update node  
    - Connect from Update row in sheet with PersonURL.

11. **Add Set Node (Set html to scrape data)**  
    - Assign field `html` to `browserHtml` from previous HTTP request  
    - Connect from HTTP Request for PersonURL.

12. **Create multiple Code Nodes for scraping:**  
    - Name: Extract `<h1 class="oh1">` for name  
    - Age: Extract age and DOB from span  
    - Primary Phone: Extract primary phone number  
    - Other numbers: Extract other phone numbers with labels  
    - Emails: Extract and deduplicate emails ignoring support emails  
    - Current Address: Extract address from `<a>` tag with `data-link-to-more="address"`  
    - Connect all sequentially from Set node.

13. **Add If Node (If phone numbers not found)**  
    - Condition: Check if "Other numbers" equals "Not Found"  
    - Connect from Current Address node.

14. **Add Google Sheets Node (Update Scraped Data in sheet)**  
    - Update row with extracted profile data (name, age, phones, emails, address)  
    - Connect from If node (false branch).

15. **Add Google Sheets Node (Update Errors in Google Sheets)**  
    - Update row with error info if HTTP request failed for PersonURL  
    - Connect from HTTP Request for PersonURL (on error).

16. **Add Code Node (RelativeProfile)**  
    - Extract relative profile URL from primary profile HTML  
    - Connect from If phone numbers not found (true branch).

17. **Add Google Sheets Node (Update Scraped data with RelativeURL into Sheets)**  
    - Update `RelativeURL` column and mark `Relative Name` as "Running..."  
    - Connect from RelativeProfile.

18. **Add If Node (Check if RelativeURL exists)**  
    - Condition: `RelativeURL` not equal to "Not Found"  
    - Connect from previous update.

19. **Add HTTP Request Node (Make HTTP Request for RelativeURL)**  
    - Zyte API POST request for relative profile URL  
    - Connect from If node (true branch).

20. **Add Google Sheets Node (Update errors Google Sheets)**  
    - Update errors if relative profile URL not found  
    - Connect from If node (false branch).

21. **Add Set Node (Set relativeprofile page html to scrape data)**  
    - Assign `html` field from Zyte API response  
    - Connect from HTTP Request for RelativeURL.

22. **Add multiple Code Nodes for relative profile scraping:**  
    - RelativeName, RelativeAge, RelativePrimaryPhone, Relative Other numbers, RelativeEmails, Relative Current Address  
    - Connect sequentially from Set relativeprofile page html to scrape data.

23. **Add Code Node (Their RelativeProfile)**  
    - Extract further relative profile URL if any  
    - Connect from Relative Current Address.

24. **Add Google Sheets Node (Update Relative scraped data to sheets)**  
    - Update sheet with all relative profile data fields  
    - Connect from Their RelativeProfile.

25. **Add Google Sheets Node (Update row in sheet with http error)**  
    - Update row with error if HTTP request for relative profile fails  
    - Connect from HTTP Request for RelativeURL (on error).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                   | Context or Link                                           |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Zyte API Setup Guide: Visit zyte.com, sign up, use API Playground to get your unique user ID for HTTP Basic Auth; this allows bypassing captchas and bot protections for scraping.               | Zyte API: http://zyte.com                                  |
| Step 1: Search & Match Entity - Pull data from Google Sheet, search truepeoplesearch.com by address, match entity by name.                                                                      | Workflow sticky note                                       |
| Step 2: Scrape Matched Profile - Extract full personal details including phones, emails, and address from the matched profile page.                                                            | Workflow sticky note                                       |
| Step 3: Fallback to Relative - If no phone found in primary profile, scrape a relative’s profile as fallback to extract contact info.                                                           | Workflow sticky note                                       |

---

This documentation provides a detailed, structured understanding of the skip tracing workflow to extract phones and emails from TruePeopleSearch using Zyte API, enabling reproduction, modifications, and error anticipation.