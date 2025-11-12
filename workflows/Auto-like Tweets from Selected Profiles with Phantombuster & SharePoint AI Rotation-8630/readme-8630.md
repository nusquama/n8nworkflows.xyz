Auto-like Tweets from Selected Profiles with Phantombuster & SharePoint AI Rotation

https://n8nworkflows.xyz/workflows/auto-like-tweets-from-selected-profiles-with-phantombuster---sharepoint-ai-rotation-8630


# Auto-like Tweets from Selected Profiles with Phantombuster & SharePoint AI Rotation

---

### 1. Workflow Overview

This workflow automates the process of liking tweets from selected Twitter profiles by integrating Phantombuster agents, Microsoft SharePoint storage, and AI-powered session cookie selection. It targets social media managers or automation specialists who want to systematically engage with tweets from curated lists of Twitter accounts while managing rate limits and avoiding duplicate likes.

The workflow is logically divided into four main blocks:

- **1.1 Session Cookie Selection and Environment Setup**  
  Downloads a list of Twitter session cookies from SharePoint and uses an AI node to select one cookie based on current time, ensuring cookie rotation and session management.

- **1.2 Tweet Scraping and Deduplication**  
  Uses a Phantombuster Profile Extractor Agent to scrape tweets from specified Twitter profiles, selects a random tweet, and checks if it has already been liked by consulting a SharePoint CSV list.

- **1.3 CSV Creation and Upload for Autoliking**  
  Generates a CSV file containing the selected tweet URL, uploads it to SharePoint, and triggers a Phantombuster Autolike Agent to like the tweet.

- **1.4 Scheduling and Rate Limiting**  
  Controls workflow execution frequency and pacing to respect API limits and avoid over-liking, using scheduled triggers and wait nodes.

---

### 2. Block-by-Block Analysis

#### 2.1 Session Cookie Selection and Environment Setup

- **Overview:**  
  Obtains session cookies stored on SharePoint, extracts valid cookies, and runs an AI-based selection logic to pick the appropriate session cookie according to the current Berlin time. Sets environment variables needed for downstream agents.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Available Session Cookies (SharePoint)  
  - Extract Cookies (Extract from File)  
  - OpenAI Chat Model2 (AI cookie selector)  
  - Select Cookie (LangChain AI agent)  
  - Set ENV Variables (Set node)  
  - Sticky Note4, Sticky Note5 (documentation)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow hourly at minute 5 to control execution frequency.  
    - Key Config: Interval set to hourly with trigger at 5 minutes past.  
    - Inputs: None  
    - Outputs: Get Available Session Cookies  
    - Edge Cases: Missed trigger or incorrect cron expression can disrupt timing.

  - **Get Available Session Cookies**  
    - Type: Microsoft SharePoint node  
    - Role: Downloads a text file containing session cookies from a defined SharePoint folder.  
    - Configuration: Uses OAuth2 credentials for SharePoint; targets specific site, folder, and file (`twitter_session_cookies.txt`).  
    - Inputs: Schedule Trigger  
    - Outputs: Extract Cookies  
    - Edge Cases: Authentication failure, file not found, access denied.

  - **Extract Cookies**  
    - Type: Extract From File (text operation)  
    - Role: Parses downloaded text file content to extract raw cookie strings.  
    - Inputs: Get Available Session Cookies  
    - Outputs: Select Cookie  
    - Edge Cases: Empty file, malformed content.

  - **OpenAI Chat Model2**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Supports cookie selection logic by providing AI inference capabilities.  
    - Configuration: Uses OpenAI GPT-4o model, no special options enabled.  
    - Inputs: None explicitly (used as a resource for Select Cookie)  
    - Outputs: Linked internally to Select Cookie node.  
    - Edge Cases: API limits, invalid API key.

  - **Select Cookie**  
    - Type: LangChain Agent  
    - Role: Implements a prompt-driven AI logic that selects one session cookie from the list based on current hour sliced evenly by number of cookies.  
    - Key Expressions: Uses dynamic Berlin timezone time expressions to calculate slice index.  
    - Inputs: Extract Cookies, OpenAI Chat Model2  
    - Outputs: Set ENV Variables  
    - Edge Cases: No cookies available, prompt parsing errors, AI service downtime.

  - **Set ENV Variables**  
    - Type: Set  
    - Role: Stores the selected session cookie and the configured hashtag as environment variables for use in subsequent nodes.  
    - Parameters: ENV_SEARCH_HASHTAGS (from parsed input), ENV_SESSION_COOKIE (from selected cookie JSON field).  
    - Inputs: Select Cookie  
    - Outputs: Get Profile Extractor Agent  
    - Edge Cases: Missing variables if prior nodes fail.

---

#### 2.2 Tweet Scraping and Deduplication

- **Overview:**  
  Scrapes tweets from a list of Twitter profiles by running a Phantombuster Profile Extractor Agent, extracts the tweet URLs, selects one at random, and verifies against a SharePoint CSV to avoid duplicate likes.

- **Nodes Involved:**  
  - Get Profile Extractor Agent (Phantombuster)  
  - Get List of Accounts (SharePoint)  
  - Launch Agent1 (Phantombuster)  
  - Wait3 (Wait node)  
  - HTTP Request, HTTP Request1 (Phantombuster API calls)  
  - Prepare Posts (Code)  
  - If Empty (If node)  
  - Download file (SharePoint)  
  - Extract from File (CSV extraction)  
  - Check if in List (Code)  
  - If (Conditional)  
  - Wait2 (Wait node)  
  - Get Random Post (Code)  
  - Sticky Note, Sticky Note6, Sticky Note1

- **Node Details:**

  - **Get Profile Extractor Agent**  
    - Type: Phantombuster API node  
    - Role: Retrieves metadata about the Profile Extractor Agent used to scrape tweets.  
    - Inputs: Set ENV Variables  
    - Outputs: Get List of Accounts  
    - Edge Cases: Agent not found, API errors.

  - **Get List of Accounts**  
    - Type: Microsoft SharePoint node  
    - Role: Downloads CSV file containing Twitter profiles to scrape.  
    - Inputs: Get Profile Extractor Agent  
    - Outputs: Launch Agent1  
    - Edge Cases: Authentication failure, file missing.

  - **Launch Agent1**  
    - Type: Phantombuster API node  
    - Role: Runs the Profile Extractor Agent with parameters including number of tweets per profile, feed type, filters, spreadsheet URL (downloaded profiles CSV), and session cookie.  
    - Inputs: Get List of Accounts  
    - Outputs: Wait3  
    - Edge Cases: Agent failure, invalid parameters, rate limits.

  - **Wait3**  
    - Type: Wait  
    - Role: Delays workflow for 30 seconds to allow agent completion.  
    - Inputs: Launch Agent1  
    - Outputs: HTTP Request  
    - Edge Cases: Timeout too short, causing incomplete data.

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Fetches updated agent data after execution, including output location metadata.  
    - Inputs: Wait3  
    - Outputs: HTTP Request1  
    - Edge Cases: Network errors, incorrect URLs.

  - **HTTP Request1**  
    - Type: HTTP Request  
    - Role: Downloads the filtered CSV result file from Phantombuster S3 storage.  
    - Inputs: HTTP Request  
    - Outputs: Prepare Posts  
    - Edge Cases: File not found, access denied.

  - **Prepare Posts**  
    - Type: Code  
    - Role: Parses CSV content, extracts tweet URLs into JSON objects.  
    - Inputs: HTTP Request1  
    - Outputs: If Empty  
    - Edge Cases: Malformed CSV, empty data.

  - **If Empty**  
    - Type: If  
    - Role: Checks if the prepared posts list is empty.  
    - Inputs: Prepare Posts  
    - Outputs: Wait2 (if not empty), else no output (stops workflow).  
    - Edge Cases: Logic errors in empty detection.

  - **Wait2**  
    - Type: Wait  
    - Role: Short wait before proceeding to random post selection.  
    - Inputs: If Empty (non-empty branch)  
    - Outputs: Get Random Post  
    - Edge Cases: None major.

  - **Get Random Post**  
    - Type: Code  
    - Role: Selects one tweet URL at random from the scraped tweets.  
    - Inputs: Wait2  
    - Outputs: Download file  
    - Edge Cases: Empty input, random selection bias.

  - **Download file**  
    - Type: Microsoft SharePoint node  
    - Role: Downloads CSV file listing tweets already liked to check for duplicates.  
    - Inputs: Get Random Post  
    - Outputs: Extract from File  
    - Edge Cases: File missing, access denied.

  - **Extract from File**  
    - Type: Extract From File (CSV)  
    - Role: Parses downloaded CSV to extract previously liked tweet URLs.  
    - Inputs: Download file  
    - Outputs: Check if in List  
    - Edge Cases: Malformed CSV.

  - **Check if in List**  
    - Type: Code  
    - Role: Compares the random tweet URL with URLs in the liked list to detect duplicates. Normalizes URLs for accurate comparison.  
    - Inputs: Extract from File and Get Random Post  
    - Outputs: If  
    - Edge Cases: URL normalization errors, missing fields.

  - **If**  
    - Type: If  
    - Role: Branches workflow depending on whether the random tweet is already liked (`true`) or not (`false`).  
    - Inputs: Check if in List  
    - Outputs:  
      - True: Wait2 (loops back to choose another random post)  
      - False: Prepare Updated Data  
    - Edge Cases: Logic errors in condition evaluation.

  - **Prepare Updated Data**  
    - Type: Code  
    - Role: Prepares updated CSV data including the new tweet URL to append to the liked list.  
    - Inputs: If (false branch)  
    - Outputs: Convert to File  
    - Edge Cases: Data merging errors.

  - **Convert to File**  
    - Type: Convert To File  
    - Role: Converts updated JSON data back to CSV file format for upload.  
    - Inputs: Prepare Updated Data  
    - Outputs: Update file  
    - Edge Cases: Conversion errors.

  - **Update file**  
    - Type: Microsoft SharePoint node  
    - Role: Updates the `twitter_posts_already_liked.csv` file on SharePoint with the new data.  
    - Inputs: Convert to File  
    - Outputs: Create CSV Binary  
    - Edge Cases: Access denied, write failures.

  - **Create CSV Binary**  
    - Type: Code  
    - Role: Creates a one-row CSV containing the selected tweet URL for the Autolike agent.  
    - Inputs: Update file  
    - Outputs: Upload CSV  
    - Edge Cases: Encoding errors.

---

#### 2.3 CSV Creation and Upload for Autoliking

- **Overview:**  
  Uploads the single-row CSV with the selected tweet URL to SharePoint and launches the Phantombuster Autolike Agent using that CSV and the selected session cookie.

- **Nodes Involved:**  
  - Upload CSV (Microsoft SharePoint)  
  - Get Autoliking Agent (Phantombuster)  
  - Launch AL Agent (Phantombuster)  
  - Wait1 (Wait node)  
  - Get Response (Phantombuster)  
  - Sticky Note2, Sticky Note7

- **Node Details:**

  - **Upload CSV**  
    - Type: Microsoft SharePoint node  
    - Role: Uploads the CSV `twitter_posts_to_like.csv` containing the tweet to like.  
    - Inputs: Create CSV Binary  
    - Outputs: Get Autoliking Agent  
    - Edge Cases: Upload failure, file overwrite conflicts.

  - **Get Autoliking Agent**  
    - Type: Phantombuster API node  
    - Role: Retrieves metadata for the Autolike Agent used to perform the liking action.  
    - Inputs: Upload CSV  
    - Outputs: Launch AL Agent  
    - Edge Cases: Agent not found.

  - **Launch AL Agent**  
    - Type: Phantombuster API node  
    - Role: Runs the Autolike Agent with parameters including the spreadsheet URL (uploaded CSV) and session cookie.  
    - Inputs: Get Autoliking Agent  
    - Outputs: Wait1  
    - Edge Cases: Agent failure, incorrect parameters.

  - **Wait1**  
    - Type: Wait  
    - Role: Waits 15 seconds after agent launch to allow operation to complete.  
    - Inputs: Launch AL Agent  
    - Outputs: Get Response  
    - Edge Cases: Insufficient wait time.

  - **Get Response**  
    - Type: Phantombuster API node  
    - Role: Retrieves output from the Autolike Agent run for monitoring or logging.  
    - Inputs: Wait1  
    - Outputs: None (end of flow)  
    - Edge Cases: API errors.

---

#### 2.4 Scheduling and Rate Limiting

- **Overview:**  
  Controls workflow pacing using scheduled triggers and wait nodes to limit the number of likes per day, avoiding rate limits and API throttling.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Wait1  
  - Wait2  
  - Wait3  
  - Sticky Note8

- **Node Details:**

  - **Schedule Trigger**  
    - As previously described, initiates the workflow hourly.

  - **Wait Nodes (Wait1, Wait2, Wait3)**  
    - Introduce delays between key steps:  
      - Wait3: 30 seconds after launching the profile extractor agent  
      - Wait2: short delay before selecting a random post again if duplicate detected  
      - Wait1: 15 seconds after launching Autolike Agent  
    - These delays help spread operations evenly to meet daily limits (~1000 likes/day).  
    - Edge Cases: Incorrect timing may cause unintended rate limits or API errors.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                                          | Input Node(s)                        | Output Node(s)                   | Sticky Note                                                                                                                  |
|---------------------------|--------------------------------|----------------------------------------------------------|------------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger               | Starts workflow hourly to control execution frequency    | None                               | Get Available Session Cookies     | ### 4) Rate Limiting & Scheduling  \n• Controls workflow pacing and throttling                                               |
| Get Available Session Cookies | Microsoft SharePoint          | Downloads session cookies file                            | Schedule Trigger                   | Extract Cookies                  |                                                                                                                              |
| Extract Cookies           | Extract From File (text)       | Extracts raw session cookies from text file               | Get Available Session Cookies      | Select Cookie                   |                                                                                                                              |
| OpenAI Chat Model2        | LangChain OpenAI Chat Model    | Provides AI model for cookie selection                    | None                              | Linked internally to Select Cookie|                                                                                                                              |
| Select Cookie             | LangChain Agent                | Selects session cookie based on current time             | Extract Cookies, OpenAI Chat Model2| Set ENV Variables               | ### 1) Hashtag & Cookie Selection  \n• Downloads cookie list → AI selects one based on time slice                            |
| Set ENV Variables         | Set                           | Stores selected cookie and hashtag as environment variables| Select Cookie                     | Get Profile Extractor Agent      |                                                                                                                              |
| Get Profile Extractor Agent| Phantombuster API             | Retrieves profile extractor agent metadata                | Set ENV Variables                 | Get List of Accounts             |                                                                                                                              |
| Get List of Accounts      | Microsoft SharePoint          | Downloads CSV of Twitter profiles to scrape               | Get Profile Extractor Agent       | Launch Agent1                   |                                                                                                                              |
| Launch Agent1             | Phantombuster API             | Runs profile extractor agent to scrape tweets             | Get List of Accounts              | Wait3                          |                                                                                                                              |
| Wait3                     | Wait                         | Waits 30 seconds for agent completion                      | Launch Agent1                    | HTTP Request                   |                                                                                                                              |
| HTTP Request              | HTTP Request                 | Fetches agent output metadata                              | Wait3                           | HTTP Request1                  |                                                                                                                              |
| HTTP Request1             | HTTP Request                 | Downloads filtered CSV result from Phantombuster S3       | HTTP Request                   | Prepare Posts                  |                                                                                                                              |
| Prepare Posts             | Code                         | Parses tweets CSV into JSON objects                        | HTTP Request1                  | If Empty                      |                                                                                                                              |
| If Empty                 | If                           | Checks if scraped tweets list is empty                     | Prepare Posts                  | Wait2 (if not empty)           |                                                                                                                              |
| Wait2                     | Wait                         | Short wait before random selection                         | If Empty                      | Get Random Post               |                                                                                                                              |
| Get Random Post           | Code                         | Selects one tweet URL at random                            | Wait2                         | Download file                 |                                                                                                                              |
| Download file             | Microsoft SharePoint          | Downloads CSV of already liked tweets                      | Get Random Post               | Extract from File             |                                                                                                                              |
| Extract from File         | Extract From File (CSV)       | Parses already liked tweets CSV                            | Download file                 | Check if in List              |                                                                                                                              |
| Check if in List          | Code                         | Detects if selected tweet is duplicate                     | Extract from File, Get Random Post| If                           |                                                                                                                              |
| If                       | If                           | Branches based on duplicate check                          | Check if in List              | Wait2 (duplicate), Prepare Updated Data (unique) |                                                                                                                              |
| Prepare Updated Data      | Code                         | Prepares updated liked tweets list including new tweet    | If (false branch)             | Convert to File               |                                                                                                                              |
| Convert to File           | Convert To File               | Converts updated liked list JSON to CSV                    | Prepare Updated Data           | Update file                  |                                                                                                                              |
| Update file               | Microsoft SharePoint          | Updates liked tweets CSV on SharePoint                     | Convert to File              | Create CSV Binary            |                                                                                                                              |
| Create CSV Binary         | Code                         | Creates CSV with single tweet URL                          | Update file                  | Upload CSV                  |                                                                                                                              |
| Upload CSV                | Microsoft SharePoint          | Uploads single-row CSV for Autolike agent                  | Create CSV Binary            | Get Autoliking Agent         | ### 3) CSV Upload & Auto-like Launch  \n• Uploads CSV and triggers Autolike Agent                                             |
| Get Autoliking Agent      | Phantombuster API             | Retrieves Autolike Agent metadata                          | Upload CSV                  | Launch AL Agent             |                                                                                                                              |
| Launch AL Agent           | Phantombuster API             | Runs Autolike Agent with CSV and session cookie            | Get Autoliking Agent        | Wait1                      |                                                                                                                              |
| Wait1                     | Wait                         | Waits 15 seconds after launching Autolike Agent            | Launch AL Agent             | Get Response               |                                                                                                                              |
| Get Response              | Phantombuster API             | Retrieves Autolike Agent output                            | Wait1                      | None                       |                                                                                                                              |
| Sticky Note               | Sticky Note                  | Documentation on Twitter Post scraping                     | None                       | None                       | ## Get Twitter Posts By Scraping defined Profiles                                                                            |
| Sticky Note1              | Sticky Note                  | Documentation on random post selection and deduplication  | None                       | None                       | ## Get Random Post, Check if Already Liked and Upload to SharePoint                                                          |
| Sticky Note2              | Sticky Note                  | Documentation on Autoliking agent launch                   | None                       | None                       | ## Launch Autoliking Agent to Like Tweet                                                                                      |
| Sticky Note4              | Sticky Note                  | Documentation on session cookie retrieval                  | None                       | None                       | ## Start Workflow and Retrieve Session Cookie                                                                                 |
| Sticky Note5              | Sticky Note                  | Notes on hashtag & cookie selection                         | None                       | None                       | ### 1) Hashtag & Cookie Selection  \n• Downloads cookie list → AI picks one based on time slice                              |
| Sticky Note6              | Sticky Note                  | Notes on tweet scraping & duplicate check                   | None                       | None                       | ### 2) Tweet Scraping & Deduplication  \n• Runs Phantombuster Agent → gets tweets → deduplication                            |
| Sticky Note7              | Sticky Note                  | Notes on CSV upload and autolike agent trigger             | None                       | None                       | ### 3) CSV Upload & Auto-like Launch  \n• Creates CSV with tweet URL, uploads, triggers Autolike                             |
| Sticky Note8              | Sticky Note                  | Notes on rate limiting and scheduling                       | None                       | None                       | ### 4) Rate Limiting & Scheduling  \n• Scheduled trigger plus wait nodes throttle likes to ~1000/day                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to hourly with trigger at 5 minutes past each hour.

2. **Create Microsoft SharePoint node "Get Available Session Cookies"**  
   - Operation: Download file  
   - Select SharePoint site, folder, and file `twitter_session_cookies.txt`  
   - Connect input from Schedule Trigger.

3. **Create "Extract Cookies" node (Extract From File)**  
   - Operation: Text extraction to parse cookies  
   - Connect input from "Get Available Session Cookies".

4. **Create OpenAI Chat Model node "OpenAI Chat Model2"**  
   - Model: GPT-4o-latest  
   - No additional options.  
   - No direct input connection; will be linked within AI agent node.

5. **Create LangChain Agent node "Select Cookie"**  
   - Prompt: Implement time-slice logic to select one session cookie based on current Berlin hour and number of cookies (2 to 4).  
   - Input: Output of "Extract Cookies" and OpenAI Chat Model2 as AI resource.  
   - Output: Connect to "Set ENV Variables".

6. **Create Set node "Set ENV Variables"**  
   - Define variables:  
     - `ENV_SEARCH_HASHTAGS` (from parsed input or static value)  
     - `ENV_SESSION_COOKIE` (from `session_cookie` field output of Select Cookie)  
   - Connect input from Select Cookie.

7. **Create Phantombuster node "Get Profile Extractor Agent"**  
   - Operation: Get agent metadata  
   - Agent ID: Profile Extractor Agent ID (8659007661393374)  
   - Connect input from Set ENV Variables.

8. **Create Microsoft SharePoint node "Get List of Accounts"**  
   - Operation: Download or update file (depending on use case)  
   - Select SharePoint site, folder, file `twitter_profiles_to_scrape.csv`  
   - Connect input from "Get Profile Extractor Agent".

9. **Create Phantombuster node "Launch Agent1"**  
   - Operation: Launch agent  
   - Agent ID: Profile Extractor Agent ID  
   - Parameters (JSON):  
     ```json
     {
       "numberOfTweetsPerProfile": 20,
       "feedType": "tweet",
       "filters": "tweetLink",
       "spreadsheetUrl": "{{$('Get List of Accounts').item.json['@content.downloadUrl']}}",
       "sessionCookie": "{{$('Set ENV Variables').item.json.ENV_SESSION_COOKIE}}"
     }
     ```  
   - Connect input from "Get List of Accounts".

10. **Create Wait node "Wait3"**  
    - Duration: 30 seconds  
    - Connect input from "Launch Agent1".

11. **Create HTTP Request node "HTTP Request"**  
    - Method: GET  
    - URL: `https://api.phantombuster.com/api/v2/agents/fetch?id=8659007661393374`  
    - Authentication: Phantombuster API credentials  
    - Connect input from "Wait3".

12. **Create HTTP Request node "HTTP Request1"**  
    - Method: GET  
    - URL built dynamically from previous HTTP Request output for filtered CSV result download.  
    - Authentication: Phantombuster API credentials  
    - Connect input from "HTTP Request".

13. **Create Code node "Prepare Posts"**  
    - JavaScript: Parse CSV data lines into JSON array of `{tweetUrl}` objects.  
    - Connect input from "HTTP Request1".

14. **Create If node "If Empty"**  
    - Condition: Check if prepared posts are empty (`$json.isEmpty() == true`)  
    - Connect input from "Prepare Posts".  
    - True branch: stops workflow (no output).  
    - False branch: connects to "Wait2".

15. **Create Wait node "Wait2"**  
    - No duration defined (optional short wait).  
    - Connect input from False branch of "If Empty".

16. **Create Code node "Get Random Post"**  
    - JavaScript: Selects one random tweet URL from input list.  
    - Connect input from "Wait2".

17. **Create Microsoft SharePoint node "Download file"**  
    - Operation: Download file  
    - Select SharePoint site, folder, file `twitter_posts_already_liked.csv`  
    - Connect input from "Get Random Post".

18. **Create Extract From File node "Extract from File"**  
    - Operation: CSV extraction with header row.  
    - Connect input from "Download file".

19. **Create Code node "Check if in List"**  
    - JavaScript: Normalizes URLs and checks if selected random post exists in liked list.  
    - Connect input from "Extract from File" and "Get Random Post" (both inputs used in code).  
    - Output connects to "If" node.

20. **Create If node "If"**  
    - Condition: If `isDuplicate` is true  
    - True branch: loops back to "Wait2" (to select another random post)  
    - False branch: proceeds to "Prepare Updated Data".

21. **Create Code node "Prepare Updated Data"**  
    - JavaScript: Combines existing liked posts plus new random post into updated list.  
    - Connect input from False branch of "If".

22. **Create Convert To File node "Convert to File"**  
    - Converts JSON list to CSV format.  
    - Connect input from "Prepare Updated Data".

23. **Create Microsoft SharePoint node "Update file"**  
    - Operation: Update file  
    - Select SharePoint site, folder, file `twitter_posts_already_liked.csv`  
    - Use "Convert to File" output as content.  
    - Connect input from "Convert to File".

24. **Create Code node "Create CSV Binary"**  
    - JavaScript: Creates single-row CSV with selected tweet URL for autolike.  
    - Connect input from "Update file".

25. **Create Microsoft SharePoint node "Upload CSV"**  
    - Operation: Upload file  
    - Select SharePoint site, folder, file name `twitter_posts_to_like.csv`  
    - Use "Create CSV Binary" output as content.  
    - Connect input from "Create CSV Binary".

26. **Create Phantombuster node "Get Autoliking Agent"**  
    - Operation: Get agent metadata  
    - Agent ID: Autolike Agent ID (275516550112020)  
    - Connect input from "Upload CSV".

27. **Create Phantombuster node "Launch AL Agent"**  
    - Operation: Launch agent  
    - Agent ID: Autolike Agent ID  
    - Parameters (JSON):  
      ```json
      {
        "spreadsheetUrl": "{{$('Upload CSV').item.json['@content.downloadUrl']}}",
        "sessionCookie": "{{$('Set ENV Variables').first().json.ENV_SESSION_COOKIE}}"
      }
      ```  
    - Connect input from "Get Autoliking Agent".

28. **Create Wait node "Wait1"**  
    - Duration: 15 seconds  
    - Connect input from "Launch AL Agent".

29. **Create Phantombuster node "Get Response"**  
    - Operation: Get agent output  
    - Agent ID: Autolike Agent ID  
    - Connect input from "Wait1".

30. **Add Sticky Notes**  
    - Add descriptive sticky notes at relevant points to document each major block and sub-step as per original.

31. **Configure Credentials**  
    - Microsoft SharePoint OAuth2 credentials for SharePoint nodes.  
    - Phantombuster API credentials for all Phantombuster nodes.  
    - OpenAI API credentials for LangChain nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The AI cookie selection uses a GPT-4o-powered LangChain agent with a prompt that divides session cookies by current Berlin hour to balance usage. | Cookie rotation logic is key to avoid session expiration or bans.                                      |
| Phantombuster agents are used for Twitter profile scraping and autoliking, facilitating scalable automation.   | Requires valid Phantombuster API access and agent setup with proper configuration.                      |
| Microsoft SharePoint is used as a storage backend for session cookies, profiles list, and tracking liked tweets. | OAuth2 credentials must be set and SharePoint folder/file IDs updated if structure changes.             |
| The workflow is designed to respect rate limits by using scheduled triggers and wait nodes to throttle actions.| Adjust wait durations or schedule interval to fit your Phantombuster plan limits and Twitter rules.    |
| Sticky notes throughout the workflow provide clear documentation blocks and usage instructions.                | Useful for onboarding and maintenance; keep notes updated with any workflow changes.                    |
| For detailed Phantombuster API documentation and agent setup, visit https://phantombuster.com/api             | External resource for agent IDs, parameters, and troubleshooting.                                      |

---

**Disclaimer:**  
The provided content originates exclusively from an n8n automation workflow. It fully complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.