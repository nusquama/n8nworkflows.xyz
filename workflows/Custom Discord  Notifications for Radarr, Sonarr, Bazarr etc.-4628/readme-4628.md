Custom Discord  Notifications for Radarr, Sonarr, Bazarr etc.

https://n8nworkflows.xyz/workflows/custom-discord--notifications-for-radarr--sonarr--bazarr-etc--4628


# Custom Discord  Notifications for Radarr, Sonarr, Bazarr etc.

---

### 1. Workflow Overview

This n8n workflow is designed to receive webhook notifications from media management services—specifically Radarr, Sonarr, and Bazarr—and send customized, simplified notifications to a Discord channel. It targets users who self-host these services and want to tailor their notification appearance and content for Discord.

The workflow logically divides into these main blocks:

- **1.1 Input Reception:** A webhook node listens for incoming POST requests from Radarr, Sonarr, or Bazarr.

- **1.2 Source Identification and Routing:** A Switch node inspects the incoming payload to determine the source application and routes the flow accordingly.

- **1.3 Data Preparation and Enrichment:** For each source, specific Set and Code nodes extract, format, and prepare notification data (movie, series, or subtitle properties). Special cases like multiple episodes in Sonarr are handled by branching logic.

- **1.4 Description Translation:** Code nodes translate event type descriptions from their original service-specific terms into simplified, user-friendly text.

- **1.5 Notification Assembly:** A Code node creates a JSON object formatted for Discord webhook consumption, including embeds with fields like title, description, quality, size, subtitles, and release info.

- **1.6 Conditional Notification Output:** An Evaluation node optionally tests or logs the output, and the final HTTP Request node posts the notification to Discord.

- **1.7 Testing and Utilities:** Additional nodes provide test execution, JSON parsing, and filtering logic to aid workflow robustness and validation.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
Receives incoming HTTP POST requests at a specified endpoint from Radarr, Sonarr, or Bazarr.

**Nodes Involved:**  
- Webhook

**Node Details:**  

- **Webhook**  
  - Type: Webhook (HTTP Trigger)  
  - Configuration: Listens on path `/radarr` with POST method. No authentication or additional options configured.  
  - Inputs: External HTTP POST request from media services.  
  - Outputs: Passes received JSON payload downstream.  
  - Edge Cases: Incoming malformed JSON or wrong HTTP method will fail or be ignored. Needs external configuration in Radarr/Sonarr/Bazarr to point to this endpoint.  
  - Notes: The webhook path should be updated to match your n8n instance URL and exposed endpoint.

---

#### 2.2 Source Identification and Routing

**Overview:**  
Determines the origin service of the incoming webhook by inspecting payload fields and routes the flow into one of three branches: Bazarr, Radarr, or Sonarr.

**Nodes Involved:**  
- Switch

**Node Details:**

- **Switch**  
  - Type: Switch  
  - Configuration:  
    - Routes based on:  
      - If `$json.body.title` contains "Bazarr" → Bazarr branch  
      - If `$json.body.instanceName` equals "Radarr" → Radarr branch  
      - If `$json.body.instanceName` equals "Sonarr" → Sonarr branch  
  - Inputs: JSON from Webhook  
  - Outputs: 3 outputs, each connected to a respective Set properties node.  
  - Edge Cases: If none of the conditions match, no downstream flow occurs (no default output defined). Payload structure must be consistent for conditions to work.  
  - Version: 3.2 or above recommended for condition types used.

---

#### 2.3 Data Preparation and Enrichment

**Overview:**  
Extracts relevant data and formats notification content depending on source service and event type. It includes handling single and multiple episodes (Sonarr), movies (Radarr), and subtitles (Bazarr).

**Nodes Involved:**  
- Set Bazarr properties  
- Set Radarr properties  
- Set Sonarr properties  
- Set movie properties  
- Set series properties  
- Set subtitle properties  
- Split Out  
- Is multiple  
- Set movie fields  
- Set series fields  
- Set subtitle fields

**Node Details:**

- **Set Bazarr properties**  
  - Type: Set  
  - Role: Assigns notification color, avatar URL, username ("Bazarr"), and copies incoming body.  
  - Inputs: Switch output for Bazarr  
  - Outputs: To Set subtitle properties  
  - Edge Cases: Assumes payload has `body.message` with well-formed subtitle info.

- **Set Radarr properties**  
  - Type: Set  
  - Role: Assigns color, avatar URL, username ("Radarr"), and copies body.  
  - Inputs: Switch output for Radarr  
  - Outputs: To Set movie properties  
  - Edge Cases: Requires `body.movie` or `body.remoteMovie` for data extraction.

- **Set Sonarr properties**  
  - Type: Set  
  - Role: Assigns color, avatar URL, username ("Sonarr"), and copies body.  
  - Inputs: Switch output for Sonarr  
  - Outputs: To "Is multiple" node  
  - Edge Cases: Assumes payload may contain multiple episodes.

- **Is multiple**  
  - Type: If (Boolean check)  
  - Role: Checks if multiple episodes exist (`body.episodeFiles` array) for Sonarr.  
  - Inputs: Set Sonarr properties  
  - Outputs:  
    - True: Splits episodes out individually  
    - False: Continues with single episode  
  - Edge Cases: If `body.episodeFiles` missing or empty, flow goes to single episode path.

- **Split Out**  
  - Type: SplitOut  
  - Role: Splits array `body.episodeFiles` into separate items to handle each episode individually.  
  - Inputs: True output from Is multiple  
  - Outputs: Set series properties for each episode  
  - Edge Cases: Empty or large arrays may affect performance.

- **Set movie properties**  
  - Type: Set  
  - Role: Extracts and formats movie-related fields: title, description, quality, size (in GB), release name, poster image URL, subtitles list.  
  - Inputs: Set Radarr properties  
  - Outputs: Set movie fields  
  - Edge Cases: Some fields are conditional, e.g., fallback between `movie.year` and `remoteMovie.year`.

- **Set series properties**  
  - Type: Set  
  - Role: Extracts and formats series-related fields: title with season/episode, action, quality, size, release, poster, description, subtitles.  
  - Inputs: Split Out or single from Is multiple  
  - Outputs: Set series fields  
  - Edge Cases: Handles missing subtitles by defaulting to 'None'.

- **Set subtitle properties**  
  - Type: Set  
  - Role: Parses Bazarr subtitle message text to extract title, language, provider, score, description, and sets image to empty string.  
  - Inputs: Set Bazarr properties  
  - Outputs: Set subtitle fields  
  - Edge Cases: Regex extraction assumes well-structured message string; malformed messages may cause errors.

- **Set movie fields**  
  - Type: Code (JavaScript)  
  - Role: Constructs array `fields` for Discord embed from movie properties: Quality, Size, Subtitles (optional), Release (formatted in code block).  
  - Inputs: Set movie properties  
  - Outputs: Translate Radarr description  
  - Edge Cases: Skips subtitles field if empty string.

- **Set series fields**  
  - Type: Code (JavaScript)  
  - Role: Constructs array `fields` for Discord embed from series properties, abbreviates subtitle count display if multiple.  
  - Inputs: Set series properties  
  - Outputs: Translate Sonarr description  
  - Edge Cases: Subtitle display logic handles 'None' correctly.

- **Set subtitle fields**  
  - Type: Code (JavaScript)  
  - Role: Constructs array `fields` for Discord embed with Language and Score, both inline.  
  - Inputs: Set subtitle properties  
  - Outputs: Translate Bazarr description  
  - Edge Cases: Assumes properties exist as strings.

---

#### 2.4 Description Translation

**Overview:**  
Replaces technical or internal event type codes with human-readable descriptions for notification clarity.

**Nodes Involved:**  
- Translate Bazarr description  
- Translate Radarr description  
- Translate Sonarr description

**Node Details:**

- **Translate Bazarr description**  
  - Type: Code  
  - Role: Translates Bazarr event descriptions (e.g., "subtitles downloaded" → "Subtitles downloaded").  
  - Inputs: Set subtitle fields  
  - Outputs: Prepare notification JSON  
  - Edge Cases: Limited translation cases defined; unrecognized descriptions pass through unchanged.

- **Translate Radarr description**  
  - Type: Code  
  - Role: Maps Radarr event codes (e.g., "MovieFileDeleteupgrade" → "Movie upgraded") to user-friendly text.  
  - Inputs: Set movie fields  
  - Outputs: Prepare notification JSON  
  - Edge Cases: Other event descriptions pass through unchanged.

- **Translate Sonarr description**  
  - Type: Code  
  - Role: Maps Sonarr event codes similarly (e.g., "EpisodeFileDeleteupgrade" → "Episode upgraded").  
  - Inputs: Set series fields  
  - Outputs: Prepare notification JSON  
  - Edge Cases: Limited cases defined.

---

#### 2.5 Notification Assembly

**Overview:**  
Builds the final JSON payload formatted for Discord webhook API, including embeds with all relevant fields and styling.

**Nodes Involved:**  
- Prepare notification JSON

**Node Details:**

- **Prepare notification JSON**  
  - Type: Code  
  - Role: Constructs Discord webhook JSON, setting username, avatar URL, and embed with title, description, author, thumbnail, color, and fields.  
  - Inputs: Translate * description nodes  
  - Outputs: Is Test node  
  - Edge Cases: Assumes all fields are present and correctly formatted. Missing fields may cause incomplete embeds.

---

#### 2.6 Conditional Notification Output and Testing

**Overview:**  
Optionally tests the generated notification against Google Sheets data and conditionally sends the notification to Discord.

**Nodes Involved:**  
- Is Test  
- Evaluation  
- Notify Discord  
- Run Tests  
- Test is enabled  
- Convert to JSON  
- Sticky Notes (for documentation)

**Node Details:**

- **Is Test**  
  - Type: Evaluation (checkIfEvaluating)  
  - Role: Determines if current run is a test to either log the output or send notification.  
  - Inputs: Prepare notification JSON  
  - Outputs:  
    - True: Evaluation node (logs or records output)  
    - False: Notify Discord node (sends message)  
  - Edge Cases: Requires proper evaluation setup.

- **Evaluation**  
  - Type: Evaluation  
  - Role: Sends notification data to Google Sheets for testing or logging purposes.  
  - Inputs: Is Test (true branch)  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Google API errors, auth failures.

- **Notify Discord**  
  - Type: HTTP Request  
  - Role: Sends POST request to Discord webhook URL with JSON body of notification.  
  - Inputs: Is Test (false branch)  
  - Configuration: Discord webhook URL set in node parameters. Retries enabled on failure.  
  - Edge Cases: HTTP errors, invalid webhook URL, rate limiting.

- **Run Tests**  
  - Type: Evaluation Trigger  
  - Role: Triggers the test evaluation based on spreadsheet data.  
  - Inputs: External or manual trigger  
  - Outputs: Test is enabled

- **Test is enabled**  
  - Type: Filter  
  - Role: Checks if test input exists to continue testing flow.  
  - Inputs: Run Tests  
  - Outputs: Convert to JSON

- **Convert to JSON**  
  - Type: Code  
  - Role: Parses test input JSON to proper format for Switch node.  
  - Inputs: Test is enabled  
  - Outputs: Switch (start of main flow)

- **Sticky Notes**  
  - Provide extensive documentation on event types, usage instructions, and output format.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                  |
|-------------------------|---------------------|-----------------------------------------------|------------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| Webhook                 | Webhook             | Receives incoming POST requests                | External HTTP Request         | Switch                       |                                                                                              |
| Switch                  | Switch              | Routes flow to Bazarr, Radarr, or Sonarr branch | Webhook                      | Set Bazarr properties; Set Radarr properties; Set Sonarr properties |                                                                                              |
| Set Bazarr properties   | Set                 | Assigns Bazarr notification base properties    | Switch (Bazarr output)        | Set subtitle properties       |                                                                                              |
| Set subtitle properties | Set                 | Parses Bazarr message for subtitle details     | Set Bazarr properties         | Set subtitle fields           |                                                                                              |
| Set subtitle fields     | Code                | Builds Discord embed fields for subtitles      | Set subtitle properties       | Translate Bazarr description  |                                                                                              |
| Translate Bazarr description | Code           | Translates Bazarr event text                    | Set subtitle fields           | Prepare notification JSON     |                                                                                              |
| Set Radarr properties   | Set                 | Assigns Radarr notification base properties    | Switch (Radarr output)        | Set movie properties          |                                                                                              |
| Set movie properties    | Set                 | Extracts and formats Radarr movie event details| Set Radarr properties         | Set movie fields              |                                                                                              |
| Set movie fields        | Code                | Builds Discord embed fields for movies          | Set movie properties          | Translate Radarr description  |                                                                                              |
| Translate Radarr description | Code           | Translates Radarr event text                     | Set movie fields              | Prepare notification JSON     |                                                                                              |
| Set Sonarr properties   | Set                 | Assigns Sonarr notification base properties    | Switch (Sonarr output)        | Is multiple                  |                                                                                              |
| Is multiple             | If                  | Checks if multiple episodes present in Sonarr  | Set Sonarr properties         | Split Out (true), Set series properties (false) |                                                                                              |
| Split Out               | SplitOut            | Splits multiple episodes into separate items   | Is multiple (true)            | Set series properties         |                                                                                              |
| Set series properties   | Set                 | Extracts and formats Sonarr series event details| Split Out or Is multiple (false) | Set series fields           |                                                                                              |
| Set series fields       | Code                | Builds Discord embed fields for series          | Set series properties         | Translate Sonarr description  |                                                                                              |
| Translate Sonarr description | Code            | Translates Sonarr event text                      | Set series fields             | Prepare notification JSON     |                                                                                              |
| Prepare notification JSON | Code               | Constructs final Discord webhook payload         | Translate * description nodes | Is Test                      | Sticky Note4: "Output Discord format of notification with fields including title, description, etc." |
| Is Test                 | Evaluation          | Determines test mode to send or log notification | Prepare notification JSON     | Evaluation (test), Notify Discord (production) |                                                                                              |
| Evaluation              | Evaluation          | Sends notification data to Google Sheets for testing | Is Test (true)             |                              |                                                                                              |
| Notify Discord          | HTTP Request        | Sends notification to Discord webhook           | Is Test (false)               |                              |                                                                                              |
| Run Tests               | Evaluation Trigger  | Triggers test evaluations                        | External or manual trigger    | Test is enabled              |                                                                                              |
| Test is enabled         | Filter              | Checks if test input exists                       | Run Tests                    | Convert to JSON              |                                                                                              |
| Convert to JSON         | Code                | Parses test JSON input                            | Test is enabled              | Switch                      |                                                                                              |
| Sticky Note             | Sticky Note         | Documentation on Bazarr events                    |                              |                              | "Bazarr events: subtitle downloaded, subtitle upgraded"                                      |
| Sticky Note1            | Sticky Note         | Documentation on Radarr events                    |                              |                              | "Radarr events: File Import, Movie File Delete, Movie File Delete for Upgrade, Manual Interaction Required" |
| Sticky Note2            | Sticky Note         | Documentation on Sonarr events                    |                              |                              | "Sonarr events: File Import, Movie File Delete, Movie File Delete for Upgrade, Manual Interaction Required" |
| Sticky Note3            | Sticky Note         | Overall usage instructions and configuration notes |                            |                              | Extensive workflow instructions and setup notes                                              |
| Sticky Note4            | Sticky Note         | Output format explanation                         |                              |                              | "Discord format of notification"                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `radarr` (or your chosen path)  
   - No authentication  
   - This node receives incoming notifications from Radarr, Sonarr, Bazarr.

2. **Add Switch Node for Routing**  
   - Connect Webhook output to Switch input  
   - Create 3 outputs with conditions:  
     - Output "Bazarr": `$json.body.title` contains "Bazarr"  
     - Output "Radarr": `$json.body.instanceName` equals "Radarr"  
     - Output "Sonarr": `$json.body.instanceName` equals "Sonarr"  

3. **Set Bazarr Properties (Set Node)**  
   - Inputs: Switch Bazarr output  
   - Assign:  
     - color: 16777215 (white)  
     - avatar: Bazarr logo URL  
     - username: "Bazarr"  
     - body: `{{$json.body}}` (copy entire payload)

4. **Set Subtitle Properties (Set Node)**  
   - Inputs: Set Bazarr properties  
   - Extract fields from `body.message` using regex: title, language, provider, score, description  
   - Set image to empty string  

5. **Set Subtitle Fields (Code Node)**  
   - Inputs: Set subtitle properties  
   - Create `fields` array with Language and Score as inline fields

6. **Translate Bazarr Description (Code Node)**  
   - Inputs: Set subtitle fields  
   - Translate known description strings (e.g., "subtitles downloaded" → "Subtitles downloaded")

7. **Set Radarr Properties (Set Node)**  
   - Inputs: Switch Radarr output  
   - Assign:  
     - color: 16761392  
     - avatar: Radarr logo URL  
     - username: "Radarr"  
     - body: `{{$json.body}}`

8. **Set Movie Properties (Set Node)**  
   - Inputs: Set Radarr properties  
   - Extract and assign: title, description, quality, size (GB), release, image (poster URL), subtitles

9. **Set Movie Fields (Code Node)**  
   - Inputs: Set movie properties  
   - Build `fields` array with Quality, Size, optional Subtitles, Release (formatted)

10. **Translate Radarr Description (Code Node)**  
    - Inputs: Set movie fields  
    - Map eventType or deleteReason codes to friendly text

11. **Set Sonarr Properties (Set Node)**  
    - Inputs: Switch Sonarr output  
    - Assign:  
      - color: 52479  
      - avatar: Sonarr logo URL  
      - username: "Sonarr"  
      - body: `{{$json.body}}`

12. **Is Multiple (If Node)**  
    - Inputs: Set Sonarr properties  
    - Condition: Check if `body.episodeFiles` array exists

13. **Split Out (SplitOut Node)**  
    - Inputs: Is multiple true output  
    - Split `body.episodeFiles` into individual items

14. **Set Series Properties (Set Node)**  
    - Inputs: Split Out output and Is multiple false output  
    - Extract and assign: title (with SxxExx), action, quality, size (GB), release, image, description, subtitles

15. **Set Series Fields (Code Node)**  
    - Inputs: Set series properties  
    - Build `fields` array with Quality, Size, Subtitles (abbreviated if multiple), Release

16. **Translate Sonarr Description (Code Node)**  
    - Inputs: Set series fields  
    - Translate event descriptions to friendly text

17. **Prepare Notification JSON (Code Node)**  
    - Inputs: Translate * description nodes (Bazarr, Radarr, Sonarr)  
    - Construct Discord webhook JSON with username, avatar, embed (title, description, author, thumbnail, color, fields)

18. **Is Test (Evaluation Node)**  
    - Inputs: Prepare notification JSON  
    - Configure to check if this is a test run or production

19. **Evaluation (Evaluation Node)**  
    - Inputs: Is Test true output  
    - Configure Google Sheets OAuth2 credentials  
    - Send notification data to Google Sheets for logging/testing

20. **Notify Discord (HTTP Request Node)**  
    - Inputs: Is Test false output  
    - Method: POST  
    - URL: Discord webhook URL  
    - Body: JSON from Prepare notification JSON node  
    - Enable retry on failure

21. **Run Tests (Evaluation Trigger Node)**  
    - Configure to trigger test executions as needed

22. **Test is enabled (Filter Node)**  
    - Checks if test input exists, forwards accordingly

23. **Convert to JSON (Code Node)**  
    - Parses JSON test input before feeding to Switch

24. **Add Sticky Notes**  
    - Add documentation notes about event types, setup instructions, and output format as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Custom Radarr / Sonarr / Bazarr notifications allow users hosting these media managers to tailor notification appearance and content for Discord. Only selected event types are tested for simplicity. Configuration involves setting the webhook URL in the media manager and the Discord webhook URL in the workflow. Testing uses Google Sheets evaluation nodes.                | Sticky Note3 content                                                                                              |
| Bazarr events supported: subtitle downloaded, subtitle upgraded. No event types in Bazarr; uses simple JSON webhook with "jsons://" protocol instead of "https://".                                                                                                                                                                                               | Sticky Note                                                                                                       |
| Radarr and Sonarr events supported include File Import, Movie/Episode File Delete, Delete for Upgrade, and Manual Interaction Required.                                                                                                                                                                                                                           | Sticky Note1 and Sticky Note2                                                                                      |
| Output Discord notification format includes: Title (with year and episode), action description, thumbnail image, quality, size, subtitle count, and release name.                                                                                                                                                                                                | Sticky Note4                                                                                                       |
| For testing and logging, Google Sheets OAuth2 credentials are used to store or evaluate notification outputs. Ensure proper credential setup in n8n with permissions to edit the specified spreadsheet.                                                                                                                                                              | Google Sheets OAuth2 credential references in Evaluation nodes                                                    |
| Discord webhook URL must be updated with your actual webhook URL. Retry on failure is enabled to handle transient network issues or Discord rate limits.                                                                                                                                                                                                       | Notify Discord node settings                                                                                       |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.

---