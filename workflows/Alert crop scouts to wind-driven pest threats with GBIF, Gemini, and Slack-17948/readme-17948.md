Alert crop scouts to wind-driven pest threats with GBIF, Gemini, and Slack

https://n8nworkflows.xyz/workflows/alert-crop-scouts-to-wind-driven-pest-threats-with-gbif--gemini--and-slack-17948


# Alert crop scouts to wind-driven pest threats with GBIF, Gemini, and Slack

### 1. Workflow Overview

The "Alert crop scouts to wind-driven pest threats with GBIF, Gemini, and Slack" workflow is an automated agricultural biosecurity system. It runs daily to monitor regional pest migrations, cross-reference them with local wind vectors, utilize artificial intelligence to synthesize actionable intelligence, and notify field agricultural scouts of impending risks via chat.

The architecture is divided into the following functional blocks:
- **1.1 Trigger and Configuration Setup:** Initiates the automation on a fixed schedule and injects farm coordinates and operational search parameters.
- **1.2 Search and Filter Pest Occurrences:** Queries global biodiversity databases for recent invasive or introduced pest activities within a specified geographic radius, filtering out inert results.
- **1.3 Fetch Weather and Calculate Threat:** Pulls regional meteorological forecasts and computes spatial-vector relationships (wind direction, speed, and great-circle bearings) to identify genuine propagation threats.
- **1.4 Generate AI Alert and Notify Slack:** Leverages a Google Gemini Large Language Model agent to draft concise, prioritized advisory messages and dispatches them to a target Slack communication channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Configuration Setup
- **Overview:** This block establishes the operational cadence of the workflow and defines critical geographic, temporal, and communication variables required by downstream nodes.
- **Nodes Involved:** `Every Morning at 6am`, `Set Search Parameters`

- **Node Details:**
  - **Every Morning at 6am**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger)
    - *Configuration Choices:* Configured via a cron expression to fire daily at 06:00 AM (`0 6 * * *`).
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Output connects exclusively to `Set Search Parameters`.
    - *Version-Specific Requirements:* Type version 1.2.
    - *Edge Cases/Failure Types:* Execution failures occur only if the host scheduler service is offline or misconfigured in the instance settings.

  - **Set Search Parameters**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields / Set node)
    - *Configuration Choices:* Initializes key-value pairs representing farm coordinates (`farm_latitude`, `farm_longitude`), geographic search bounds (`radius_km`), lookback windows (`lookback_days`, `wind_lookback_days`), angular tolerances (`alignment_tolerance_deg`), and target communication destinations (`slack_channel`).
    - *Key Expressions or Variables:* Static strings and numerical values (Note: Latitude and Longitude values default to placeholders and must be cast to numeric inputs during manual setup).
    - *Input/Output Connections:* Input from `Every Morning at 6am`; output connects to `Fetch GBIF Pest Occurrences`.
    - *Version-Specific Requirements:* Type version 3.4.
    - *Edge Cases/Failure Types:* Type mismatch issues if latitude/longitude strings are parsed incorrectly downstream without numeric conversion.

---

#### 2.2 Search and Filter Pest Occurrences
- **Overview:** Queries the Global Biodiversity Information Facility (GBIF) API to discover recently recorded invasive or introduced pest occurrences near the farm's coordinates, subsequently validating whether any records were returned.
- **Nodes Involved:** `Fetch GBIF Pest Occurrences`, `Check for Nearby Pests`

- **Node Details:**
  - **Fetch GBIF Pest Occurrences**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration Choices:* Sends a GET request to the GBIF Occurrence Search endpoint utilizing query parameters for spatial filtering, establishment means, date ranges, and coordinate validation.
    - *Key Expressions or Variables:* 
      - `geoDistance`: `={{ $json.farm_latitude }},{{ $json.farm_longitude }},{{ $json.radius_km }}km`
      - `eventDate`: `={{ $now.minus({ days: $json.lookback_days }).toFormat('yyyy-MM-dd') }},{{ $now.toFormat('yyyy-MM-dd') }}`
    - *Input/Output Connections:* Input from `Set Search Parameters`; output connects to `Check for Nearby Pests`.
    - *Version-Specific Requirements:* Type version 4.2.
    - *Edge Cases/Failure Types:* External API downtime, rate-limiting, or invalid coordinate parameters resulting in HTTP 4xx/5xx errors.

  - **Check for Nearby Pests**
    - *Type and Technical Role:* `n8n-nodes-base.if` (If / Conditional Branching)
    - *Configuration Choices:* Evaluates whether the returned total occurrence count is greater than zero (`{{ $json.count }} > 0`).
    - *Key Expressions or Variables:* `={{ $json.count }}`
    - *Input/Output Connections:* Input from `Fetch GBIF Pest Occurrences`. True branch is intentionally left unlinked (terminates execution if zero pests are found); False branch routes to `Fetch Wind Data` *(Note: In n8n UI terms, index 0 is empty/True branch depending on mapping, but standard flow routes to wind fetch if occurrences exist).*
    - *Version-Specific Requirements:* Type version 2.2.
    - *Edge Cases/Failure Types:* Malformed JSON response structures causing conditional evaluation failures.

---

#### 2.3 Fetch Weather and Calculate Threat
- **Overview:** Pulls recent local wind speeds and directions from Open-Meteo, then executes custom JavaScript to cross-reference pest positions against local wind-vectors, calculating true vector-aligned migration threats.
- **Nodes Involved:** `Fetch Wind Data`, `Calculate Wind Threat Alignment`, `Check Wind Alignment Threat`

- **Node Details:**
  - **Fetch Wind Data**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request)
    - *Configuration Choices:* Queries the Open-Meteo API forecast endpoint for hourly wind history (`wind_speed_10m`, `wind_direction_10m`) based on farm coordinates and historical lookback configurations.
    - *Key Expressions or Variables:*
      - Latitude/Longitude: `={{ $('Set Search Parameters').first().json.farm_latitude }}` / `={{ $('Set Search Parameters').first().json.farm_longitude }}`
      - Past Days: `={{ $('Set Search Parameters').first().json.wind_lookback_days }}`
    - *Input/Output Connections:* Input from `Check for Nearby Pests`; output connects to `Calculate Wind Threat Alignment`.
    - *Version-Specific Requirements:* Type version 4.2.
    - *Edge Cases/Failure Types:* Timeouts or data gaps in meteorological records.

  - **Calculate Wind Threat Alignment**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Node)
    - *Configuration Choices:* Executes custom JavaScript processing arrays of GBIF occurrences and Open-Meteo wind vectors. Calculates circular means of wind direction, great-circle bearings, Haversine distances, and angular alignment differences.
    - *Key Expressions or Variables:* References upstream node data via `$('Set Search Parameters').first().json`, `$input.first().json.results`, and `$('Fetch Wind Data').first().json.hourly`.
    - *Input/Output Connections:* Input from `Fetch Wind Data`; output connects to `Check Wind Alignment Threat`.
    - *Version-Specific Requirements:* Type version 2.0.
    - *Edge Cases/Failure Types:* Unhandled division by zero during circular mean calculations if wind arrays are empty.

  - **Check Wind Alignment Threat**
    - *Type and Technical Role:* `n8n-nodes-base.if` (If / Conditional Branching)
    - *Configuration Choices:* Evaluates whether the calculated count of aligned threats exceeds zero (`{{ $json.aligned_threat_count }} > 0`).
    - *Key Expressions or Variables:* `={{ $json.aligned_threat_count }}`
    - *Input/Output Connections:* Input from `Calculate Wind Threat Alignment`. True branch is left unlinked (stops workflow if no wind-aligned vectors exist); False branch routes to `Scout Alert Generator Agent`.
    - *Version-Specific Requirements:* Type version 2.2.
    - *Edge Cases/Failure Types:* Type mismatch issues if count properties resolve as undefined.

---

#### 2.4 Generate AI Alert and Notify Slack
- **Overview:** Employs a Google Gemini language model agent constrained by custom system rules to translate structured threat data into a human-readable, prioritized scouting bulletin, and publishes the advisory directly to Slack.
- **Nodes Involved:** `Scout Alert Generator Agent`, `Gemini LLM Model`, `Send Slack Alert to Scouts`

- **Node Details:**
  - **Scout Alert Generator Agent**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent / LangChain Integration)
    - *Configuration Choices:* Configured as an autonomous agent using prompt-based guidelines to construct field alerts without fabricating unverified data.
    - *Key Expressions or Variables:* Passes serialized JSON data: `{{ JSON.stringify($json, null, 2) }}`.
    - *Input/Output Connections:* Input from `Check Wind Alignment Threat`; connected to `Gemini LLM Model` via AI language model connection; output connects to `Send Slack Alert to Scouts`.
    - *Version-Specific Requirements:* Type version 1.9. Requires LangChain sub-node dependencies.
    - *Edge Cases/Failure Types:* LLM hallucinations, token limit overruns, or failure to adhere to strict formatting boundaries.

  - **Gemini LLM Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (Google Gemini Chat Model)
    - *Configuration Choices:* Utilizes the `models/gemini-3.1-flash-lite` model identifier.
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Connects to `Scout Alert Generator Agent` via `ai_languageModel`.
    - *Version-Specific Requirements:* Type version 1.0. Requires valid Google PaLM/Gemini API credentials.
    - *Edge Cases/Failure Types:* Authentication failures, API quota exhaustion, or service outages.

  - **Send Slack Alert to Scouts**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Slack Node)
    - *Configuration Choices:* Posts message content to a specified Slack channel using channel name resolution.
    - *Key Expressions or Variables:*
      - Message Text: `={{ $json.output }}`
      - Channel ID: `={{ $('Set Search Parameters').first().json.slack_channel }}`
    - *Input/Output Connections:* Input from `Scout Alert Generator Agent`.
    - *Version-Specific Requirements:* Type version 2.3. Requires Slack OAuth2 or Bot Token credentials.
    - *Edge Cases/Failure Types:* Invalid channel names, missing bot permissions to post messages in public/private channels, or expired Slack tokens.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation & setup instructions | None | None | ## Migratory Pest Wind-Vector Tracker<br>### How it works<br><br>1. Automatically triggers every day at 6 AM with preset farm configuration.<br>2. Queries the GBIF API to find pest occurrences near the farm location.<br>3. Retrieves weather and wind vector data to calculate wind alignment threat.<br>4. Uses an AI agent with Google Gemini to draft a scout alert if conditions are met.<br>5. Notifies field scouts via Slack with the generated alert.<br><br>### Setup steps<br><br>- - [ ] Configure farm coordinates, radius, and lookback days in the Config set node.<br>- - [ ] Set up credentials for the Google Gemini chat model used by the AI agent.<br>- - [ ] Configure Slack credentials and destination channel for the field scout alerts.<br><br>### Customization<br><br>Adjust farm coordinates and search radius in the Config node or modify the lookback days and wind thresholds to suit specific crop requirements. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Block documentation | None | None | ## Trigger and configuration setup<br><br>Triggers the workflow daily at 6 AM and initializes the farm location and search radius configuration. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Block documentation | None | None | ## Search and filter pest occurrences<br><br>Searches GBIF for recent pest occurrences and checks if any pests were reported nearby. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Block documentation | None | None | ## Fetch weather and calculate threat<br><br>Fetches wind data, calculates wind alignment vector threat, and checks if wind is blowing toward the farm. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Block documentation | None | None | ## Generate AI alert and notify Slack<br><br>Drafts a scout alert using an AI agent powered by Gemini and sends the notification to Slack. |
| Every Morning at 6am | n8n-nodes-base.scheduleTrigger | Daily execution timer | None | Set Search Parameters | ## Trigger and configuration setup<br><br>Triggers the workflow daily at 6 AM and initializes the farm location and search radius configuration. |
| Set Search Parameters | n8n-nodes-base.set | Defines variables, coordinates, and thresholds | Every Morning at 6am | Fetch GBIF Pest Occurrences | ## Trigger and configuration setup<br><br>Triggers the workflow daily at 6 AM and initializes the farm location and search radius configuration. |
| Fetch GBIF Pest Occurrences | n8n-nodes-base.httpRequest | Queries GBIF API for invasive/introduced pests | Set Search Parameters | Check for Nearby Pests | ## Search and filter pest occurrences<br><br>Searches GBIF for recent pest occurrences and checks if any pests were reported nearby. |
| Check for Nearby Pests | n8n-nodes-base.if | Filters workflow execution based on pest presence | Fetch GBIF Pest Occurrences | Fetch Wind Data | ## Search and filter pest occurrences<br><br>Searches GBIF for recent pest occurrences and checks if any pests were reported nearby. |
| Fetch Wind Data | n8n-nodes-base.httpRequest | Fetches hourly wind history from Open-Meteo | Check for Nearby Pests | Calculate Wind Threat Alignment | ## Fetch weather and calculate threat<br><br>Fetches wind data, calculates wind alignment vector threat, and checks if wind is blowing toward the farm. |
| Calculate Wind Threat Alignment | n8n-nodes-base.code | Computes vector bearings and wind-pest alignment | Fetch Wind Data | Check Wind Alignment Threat | ## Fetch weather and calculate threat<br><br>Fetches wind data, calculates wind alignment vector threat, and checks if wind is blowing toward the farm. |
| Check Wind Alignment Threat | n8n-nodes-base.if | Evaluates if wind-aligned threats exist | Calculate Wind Threat Alignment | Scout Alert Generator Agent | ## Fetch weather and calculate threat<br><br>Fetches wind data, calculates wind alignment vector threat, and checks if wind is blowing toward the farm. |
| Scout Alert Generator Agent | @n8n/n8n-nodes-langchain.agent | AI Agent generating structured scout alerts | Check Wind Alignment Threat | Send Slack Alert to Scouts | ## Generate AI alert and notify Slack<br><br>Drafts a scout alert using an AI agent powered by Gemini and sends the notification to Slack. |
| Gemini LLM Model | @n8n/n8n-nodes-lmChatGoogleGemini | Provides LLM backend for AI agent | None | Scout Alert Generator Agent | ## Generate AI alert and notify Slack<br><br>Drafts a scout alert using an AI agent powered by Gemini and sends the notification to Slack. |
| Send Slack Alert to Scouts | n8n-nodes-base.slack | Dispatches notification message to Slack channel | Scout Alert Generator Agent | None | ## Generate AI alert and notify Slack<br><br>Drafts a scout alert using an AI agent powered by Gemini and sends the notification to Slack. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**
   - Create a **Schedule Trigger** node named `Every Morning at 6am`.
   - Set trigger interval rule to Cron Expression: `0 6 * * *`.

2. **Create Configuration Node:**
   - Create an **Edit Fields (Set)** node named `Set Search Parameters`.
   - Add string assignments for `farm_latitude` and `farm_longitude` (insert numeric coordinate values).
   - Add number assignments for `radius_km` (`100`), `lookback_days` (`2`), `wind_lookback_days` (`2`), and `alignment_tolerance_deg` (`45`).
   - Add a string assignment for `slack_channel` (e.g., `#general`).
   - Connect `Every Morning at 6am` to `Set Search Parameters`.

3. **Create GBIF API Request Node:**
   - Create an **HTTP Request** node named `Fetch GBIF Pest Occurrences`.
   - Set Method to `GET`, URL to `https://api.gbif.org/v1/occurrence/search`.
   - Add Query Parameters:
     - `geoDistance`: `={{ $json.farm_latitude }},{{ $json.farm_longitude }},{{ $json.radius_km }}km`
     - `establishmentMeans`: `INTRODUCED`
     - `establishmentMeans`: `INVASIVE`
     - `eventDate`: `={{ $now.minus({ days: $json.lookback_days }).toFormat('yyyy-MM-dd') }},{{ $now.toFormat('yyyy-MM-dd') }}`
     - `hasCoordinate`: `true`
     - `limit`: `50`
   - Connect `Set Search Parameters` to `Fetch GBIF Pest Occurrences`.

4. **Create Pest Verification Condition:**
   - Create an **If** node named `Check for Nearby Pests`.
   - Set condition check: Left Value `={{ $json.count }}`, Operator `Larger Than`, Right Value `0`.
   - Connect `Fetch GBIF Pest Occurrences` to `Check for Nearby Pests`.

5. **Create Open-Meteo Weather Request Node:**
   - Create an **HTTP Request** node named `Fetch Wind Data`.
   - Set Method to `GET`, URL to `https://api.open-meteo.com/v1/forecast`.
   - Add Query Parameters:
     - `latitude`: `={{ $('Set Search Parameters').first().json.farm_latitude }}`
     - `longitude`: `={{ $('Set Search Parameters').first().json.farm_longitude }}`
     - `hourly`: `wind_speed_10m,wind_direction_10m`
     - `past_days`: `={{ $('Set Search Parameters').first().json.wind_lookback_days }}`
     - `forecast_days`: `1`
     - `timezone`: `auto`
   - Connect the negative/false output of `Check for Nearby Pests` to `Fetch Wind Data`.

6. **Create Threat Calculation Code Node:**
   - Create a **Code** node named `Calculate Wind Threat Alignment`.
   - Paste the provided JavaScript code block handling circular means, great-circle bearings, haversine distances, and threat filtering.
   - Connect `Fetch Wind Data` to `Calculate Wind Threat Alignment`.

7. **Create Threat Verification Condition:**
   - Create an **If** node named `Check Wind Alignment Threat`.
   - Set condition check: Left Value `={{ $json.aligned_threat_count }}`, Operator `Larger Than`, Right Value `0`.
   - Connect `Calculate Wind Threat Alignment` to `Check Wind Alignment Threat`.

8. **Create AI Agent and Model Nodes:**
   - Create an **AI Agent** node named `Scout Alert Generator Agent`. Set prompt type to `Define` and paste the biosecurity analyst prompt configuration containing `{{ JSON.stringify($json, null, 2) }}`.
   - Create a **Google Gemini Chat Model** node named `Gemini LLM Model`. Select model name `models/gemini-3.1-flash-lite`. Configure Google PaLM/Gemini API credentials.
   - Connect `Gemini LLM Model` output to the `ai_languageModel` input of `Scout Alert Generator Agent`.
   - Connect the negative/false branch output of `Check Wind Alignment Threat` to `Scout Alert Generator Agent`.

9. **Create Slack Notification Node:**
   - Create a **Slack** node named `Send Slack Alert to Scouts`.
   - Configure resource to Post Message, selecting destination by channel name.
   - Set Message to `={{ $json.output }}` and Channel ID expression to `={{ $('Set Search Parameters').first().json.slack_channel }}`.
   - Configure valid Slack OAuth2 or Bot credentials.
   - Connect `Scout Alert Generator Agent` to `Send Slack Alert to Scouts`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Video walkthrough and demonstration | [Workflow Video Link](https://youtu.be/3ExkAWTyEXU?si=R7bAUzHqf1_LpHnT) |