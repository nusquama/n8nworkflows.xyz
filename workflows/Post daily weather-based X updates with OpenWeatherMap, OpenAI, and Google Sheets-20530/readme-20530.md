Post daily weather-based X updates with OpenWeatherMap, OpenAI, and Google Sheets

https://n8nworkflows.xyz/workflows/post-daily-weather-based-x-updates-with-openweathermap--openai--and-google-sheets-20530


# Post daily weather-based X updates with OpenWeatherMap, OpenAI, and Google Sheets

### 1. Workflow Overview

This workflow automates the generation and publishing of weather-adapted social media updates on X (formerly Twitter). It runs on a daily schedule, fetches current meteorological data, categorizes it into a marketing bucket, assigns a tailored campaign angle, uses an LLM to generate an engaging post, publishes it, and logs the details to Google Sheets.

The logical execution is grouped into three main blocks:
- **1.1 Schedule and Data Ingestion:** Triggers the workflow daily at 07:00 and retrieves current weather details for a configured city from OpenWeatherMap.
- **1.2 Weather Classification and Routing:** Processes the raw weather data into a standardized temperature and condition, categorizes it into a marketing bucket (`rain`, `hot`, `cold`, or `pleasant`), and routes the execution down a specific path to inject a matching promotional angle.
- **1.3 Content Generation, Publishing, and Logging:** Connects to OpenAI via LangChain to draft a short post matching the weather and promotional angle, publishes the result to X, and records the interaction data in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule and Data Ingestion
- **Overview:** Initializes the daily execution cycle and pulls up-to-date meteorological information for the target location.
- **Nodes Involved:** `Every Day 07:00`, `Check Today's Weather`
- **Node Details:**
  - **Every Day 07:00**
    - *Type and Technical Role:* `ScheduleTrigger` (Version 1.2) - Initiates the workflow based on a time interval.
    - *Configuration:* Set to trigger daily at hour 7.
    - *Key Expressions:* None.
    - *Connections:* Output connects to `Check Today's Weather`.
    - *Edge Cases:* Timezone offsets relative to the n8n instance configuration.
  - **Check Today's Weather**
    - *Type and Technical Role:* `OpenWeatherMap` (Version 1) - Fetches current weather data from the OpenWeatherMap API.
    - *Configuration:* Configured for city "Tokyo" with English language output. Requires valid OpenWeatherMap credentials.
    - *Key Expressions:* City name parameter statically set to `"Tokyo"`.
    - *Input/Output:* Input from `Every Day 07:00`; output connects to `Classify Weather Bucket`.
    - *Edge Cases:* API rate limits, network connectivity issues, or invalid city name strings resulting in API 404/500 errors.

#### 2.2 Weather Classification and Routing
- **Overview:** Evaluates temperature and weather conditions to determine a marketing bucket, then branches the execution flow into one of four specific campaign angles.
- **Nodes Involved:** `Classify Weather Bucket`, `Route By Weather`, `☔ Rainy Day Angle`, `🥵 Hot Day Angle`, `🥶 Cold Day Angle`, `😊 Pleasant Day Angle`
- **Node Details:**
  - **Classify Weather Bucket**
    - *Type and Technical Role:* `Code` (Version 2) - Executes custom JavaScript to normalize Kelvin temperatures to Celsius and apply threshold logic.
    - *Configuration:* JavaScript processing block extracting temperature, weather condition, and description.
    - *Key Expressions:* Accesses `$input.first().json` to read OpenWeatherMap payloads, computes `tempC = Math.round(temp - 273.15)`, and outputs a JSON object containing `city`, `tempC`, `condition`, `description`, `bucket`, and `date`.
    - *Input/Output:* Input from `Check Today's Weather`; output connects to `Route By Weather`.
    - *Edge Cases:* Missing or null properties in the OpenWeatherMap JSON payload handled via optional chaining (`?.`).
  - **Route By Weather**
    - *Type and Technical Role:* `Switch` (Version 3.2) - Splits execution paths based on evaluated rule conditions.
    - *Configuration:* Evaluates `$json.bucket` against strings: `"rain"`, `"hot"`, and `"cold"`, with a fallback output designated as `"Pleasant"`.
    - *KeyExpressions:* `={{ $json.bucket }}`
    - *Input/Output:* Input from `Classify Weather Bucket`; outputs connect to their respective campaign angle nodes.
    - *Edge Cases:* Unmatched string values routed automatically to the fallback branch.
  - **☔ Rainy Day Angle**
    - *Type and Technical Role:* `Set` (Version 3.4) - Appends static marketing copy to the workflow payload.
    - *Configuration:* Assigns a string value to the `angle` property while including all other incoming fields.
    - *Key Expressions:* Static string describing rainy day offers.
    - *Input/Output:* Input from `Route By Weather` (Rain output); output connects to `Write Today's Post`.
  - **🥵 Hot Day Angle**
    - *Type and Technical Role:* `Set` (Version 3.4) - Appends static marketing copy to the workflow payload.
    - *Configuration:* Assigns a string value to the `angle` property while including all other incoming fields.
    - *Key Expressions:* Static string describing hot day offers.
    - *Input/Output:* Input from `Route By Weather` (Hot output); output connects to `Write Today's Post`.
  - **🥶 Cold Day Angle**
    - *Type and Technical Role:* `Set` (Version 3.4) - Appends static marketing copy to the workflow payload.
    - *Configuration:* Assigns a string value to the `angle` property while including all other incoming fields.
    - *Key Expressions:* Static string describing cold day offers.
    - *Input/Output:* Input from `Route By Weather` (Cold output); output connects to `Write Today's Post`.
  - **😊 Pleasant Day Angle**
    - *Type and Technical Role:* `Set` (Version 3.4) - Appends static marketing copy to the workflow payload.
    - *Configuration:* Assigns a string value to the `angle` property while including all other incoming fields.
    - *Key Expressions:* Static string describing pleasant day offers.
    - *Input/Output:* Input from `Route By Weather` (Pleasant fallback output); output connects to `Write Today's Post`.

#### 2.3 Content Generation, Publishing, and Logging
- **Overview:** Synthesizes the weather data and marketing angle through an LLM to craft a concise promotional post, publishes it directly to X, and creates an archival record in Google Sheets.
- **Nodes Involved:** `Write Today's Post`, `OpenAI Chat Model`, `Publish On X`, `Log Published Post`
- **Node Details:**
  - **Write Today's Post**
    - *Type and Technical Role:* `chainLlm` (Version 1.5) - Executes a LangChain text generation chain using an attached chat model.
    - *Configuration:* Prompt defines a tweet structure constraint (max 260 characters, specific tone, single emoji, max two hashtags). System message establishes the persona of a neighborhood café voice.
    - *Key Expressions:* 
      - Prompt text: `=Today's weather in {{ $json.city }}: {{ $json.description }}, {{ $json.tempC }}°C.\n\nCampaign angle for today:\n{{ $json.angle }}\n\nWrite ONE social media post...`
    - *Input/Output:* Main input from any of the four campaign angle nodes; model linked via `ai_languageModel` input from `OpenAI Chat Model`. Output connects to `Publish On X`.
    - *Edge Cases:* LLM output exceeding character limits or failing to respect emoji constraints.
  - **OpenAI Chat Model**
    - *Type and Technical Role:* `lmChatOpenAi` (Version 1.2) - Provides the underlying language model configuration for the LangChain node.
    - *Configuration:* Uses model `gpt-4o-mini`. Requires valid OpenAI credentials.
    - *Input/Output:* Connects to `Write Today's Post` via `ai_languageModel`.
    - *Edge Cases:* OpenAI API outages, token limit exhaustion, or authentication failures.
  - **Publish On X**
    - *Type and Technical Role:* `twitter` (Version 2) - Publishes a status update to the authenticated X (Twitter) account.
    - *Configuration:* Posts text content mapped from the upstream generation step. Requires valid X (Twitter) OAuth/API credentials.
    - *Key Expressions:* `={{ $json.text }}`
    - *Input/Output:* Input from `Write Today's Post`; output connects to `Log Published Post`.
    - *Edge Cases:* Duplicate content flags by X, rate-limiting, or invalid OAuth scopes.
  - **Log Published Post**
    - *Type and Technical Role:* `googleSheets` (Version 4.5) - Appends a row of execution metadata into a spreadsheet.
    - *Configuration:* Operation set to append. Mapped columns: `date`, `post`, `temp`, `bucket`, `weather`. Requires valid Google Sheets credentials.
    - *Key Expressions:* 
      - `date`: `={{ $('Classify Weather Bucket').first().json.date }}`
      - `post`: `={{ $('Write Today\'s Post').first().json.text }}`
      - `temp`: `={{ $('Classify Weather Bucket').first().json.tempC }}`
      - `bucket`: `={{ $('Classify Weather Bucket').first().json.bucket }}`
      - `weather`: `={{ $('Classify Weather Bucket').first().json.description }}`
    - *Input/Output:* Input from `Publish On X`. Terminal node.
    - *Edge Cases:* Spreadsheet structure mismatch, missing column headers, or insufficient permission scopes on the Google Sheets account.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every Day 07:00 | scheduleTrigger | Triggers the workflow daily at 07:00 | None | Check Today's Weather | ## 🌦️ Weather-Adaptive Social Media Promoter<br><br>Every morning this workflow checks the **local weather** and publishes a social post whose product angle matches it. A café would push iced drinks on hot days, delivery on rainy days and terrace seats on pleasant days — automatically.<br><br>### 🔧 Setup<br>1. Connect **OpenWeatherMap** (free API key), **OpenAI**, **X (Twitter)** and **Google Sheets** credentials.<br>2. Set your city in **Check Today's Weather**.<br>3. Edit the four **campaign angle** nodes (☔ 🥵 🥶 😊) — replace the sample café products with your own offers.<br>4. Point **Log Published Post** at a sheet with headers: `date, weather, temp, bucket, post`.<br>5. Activate — it posts daily at **07:00**.<br><br>### 💡 Why it's useful<br>Weather is the cheapest personalization signal there is. Same product, different pitch every day — with zero manual work. |
| Check Today's Weather | openWeatherMap | Fetches current weather data for the configured city | Every Day 07:00 | Classify Weather Bucket | ## 🌦️ Weather-Adaptive Social Media Promoter<br><br>Every morning this workflow checks the **local weather** and publishes a social post whose product angle matches it. A café would push iced drinks on hot days, delivery on rainy days and terrace seats on pleasant days — automatically.<br><br>### 🔧 Setup<br>1. Connect **OpenWeatherMap** (free API key), **OpenAI**, **X (Twitter)** and **Google Sheets** credentials.<br>2. Set your city in **Check Today's Weather**.<br>3. Edit the four **campaign angle** nodes (☔ 🥵 🥶 😊) — replace the sample café products with your own offers.<br>4. Point **Log Published Post** at a sheet with headers: `date, weather, temp, bucket, post`.<br>5. Activate — it posts daily at **07:00**.<br><br>### 💡 Why it's useful<br>Weather is the cheapest personalization signal there is. Same product, different pitch every day — with zero manual work. |
| Classify Weather Bucket | code | Calculates Celsius temperature and classifies condition into a bucket | Check Today's Weather | Route By Weather | ### 1️⃣ Read the sky<br>Fetch current weather and classify it into one of four marketing buckets: `rain / hot / cold / pleasant`. |
| Route By Weather | switch | Routes execution to a branch based on the weather bucket | Classify Weather Bucket | ☔ Rainy Day Angle,<br>🥵 Hot Day Angle,<br>🥶 Cold Day Angle,<br>😊 Pleasant Day Angle | ### 2️⃣ Pick the campaign angle<br>One branch per weather bucket. **Edit these four nodes** — this is where your products and promos live. |
| ☔ Rainy Day Angle | set | Injects promotional copy for rainy conditions | Route By Weather | Write Today's Post | ### 2️⃣ Pick the campaign angle<br>One branch per weather bucket. **Edit these four nodes** — this is where your products and promos live. |
| 🥵 Hot Day Angle | set | Injects promotional copy for hot conditions | Route By Weather | Write Today's Post | ### 2️⃣ Pick the campaign angle<br>One branch per weather bucket. **Edit these four nodes** — this is where your products and promos live. |
| 🥶 Cold Day Angle | set | Injects promotional copy for cold conditions | Route By Weather | Write Today's Post | ### 2️⃣ Pick the campaign angle<br>One branch per weather bucket. **Edit these four nodes** — this is where your products and promos live. |
| 😊 Pleasant Day Angle | set | Injects promotional copy for pleasant conditions | Route By Weather | Write Today's Post | ### 2️⃣ Pick the campaign angle<br>One branch per weather bucket. **Edit these four nodes** — this is where your products and promos live. |
| Write Today's Post | chainLlm | Generates a short social media post using an LLM prompt | ☔ Rainy Day Angle,<br>🥵 Hot Day Angle,<br>🥶 Cold Day Angle,<br>😊 Pleasant Day Angle | Publish On X | ### 3️⃣ Write, publish & log<br>The LLM writes a short post in your brand voice, it goes out on X, and everything is logged to Sheets so you can compare engagement per weather type later. |
| OpenAI Chat Model | lmChatOpenAi | Provides the gpt-4o-mini language model backend | None | Write Today's Post | ### 3️⃣ Write, publish & log<br>The LLM writes a short post in your brand voice, it goes out on X, and everything is logged to Sheets so you can compare engagement per weather type later. |
| Publish On X | twitter | Publishes the generated post text to X (Twitter) | Write Today's Post | Log Published Post | ### 3️⃣ Write, publish & log<br>The LLM writes a short post in your brand voice, it goes out on X, and everything is logged to Sheets so you can compare engagement per weather type later. |
| Log Published Post | googleSheets | Appends execution metadata and post text to Google Sheets | Publish On X | None | ### 3️⃣ Write, publish & log<br>The LLM writes a short post in your brand voice, it goes out on X, and everything is logged to Sheets so you can compare engagement per weather type later. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node named `Every Day 07:00`.
   - Set the interval rule to trigger daily at hour `7`.
2. **Add Weather Retrieval:**
   - Add an **OpenWeatherMap** node named `Check Today's Weather`.
   - Configure the city name (e.g., `Tokyo`) and language (`en`). Connect credentials for OpenWeatherMap.
   - Connect `Every Day 07:00` output to this node.
3. **Add Weather Classification Logic:**
   - Add a **Code** node named `Classify Weather Bucket`.
   - Paste the JavaScript block to compute Celsius temperature, analyze weather condition strings, and return an object containing `city`, `tempC`, `condition`, `description`, `bucket`, and `date`.
   - Connect `Check Today's Weather` output to this node.
4. **Configure Routing:**
   - Add a **Switch** node named `Route By Weather`.
   - Set up three rules matching conditions where `{{ $json.bucket }}` equals `rain`, `hot`, and `cold`. Set the fallback output to handle `pleasant`.
   - Connect `Classify Weather Bucket` output to this node.
5. **Set Campaign Angles:**
   - Add four **Set** nodes named:
     - `☔ Rainy Day Angle` (connected to Switch output 1)
     - `🥵 Hot Day Angle` (connected to Switch output 2)
     - `🥶 Cold Day Angle` (connected to Switch output 3)
     - `😊 Pleasant Day Angle` (connected to Switch fallback output)
   - Configure each node to include other fields and add a string assignment named `angle` containing relevant promotional copy for each condition.
6. **Configure AI Generation:**
   - Add an **Advanced AI -> LangChain** text generation chain node named `Write Today's Post`.
   - Set the prompt type to define and configure the system message and prompt text utilizing variables like `{{ $json.city }}`, `{{ $json.description }}`, `{{ $json.tempC }}`, and `{{ $json.angle }}`.
   - Add an **OpenAI Chat Model** sub-node, configure it to use `gpt-4o-mini`, and link it to `Write Today's Post` via the `ai_languageModel` connection. Connect credentials for OpenAI.
   - Connect all four **Set** nodes to the main input of `Write Today's Post`.
7. **Configure Publishing:**
   - Add a **Twitter (X)** node named `Publish On X`.
   - Map the text parameter to `={{ $json.text }}`. Connect credentials for X (Twitter).
   - Connect `Write Today's Post` output to this node.
8. **Configure Logging:**
   - Add a **Google Sheets** node named `Log Published Post`.
   - Set the operation to `append`, and select your target document and sheet name. Connect credentials for Google Sheets.
   - Map the columns using expressions pointing back to upstream nodes:
     - `date`: `={{ $('Classify Weather Bucket').first().json.date }}`
     - `weather`: `={{ $('Classify Weather Bucket').first().json.description }}`
     - `temp`: `={{ $('Classify Weather Bucket').first().json.tempC }}`
     - `bucket`: `={{ $('Classify Weather Bucket').first().json.bucket }}`
     - `post`: `={{ $('Write Today\'s Post').first().json.text }}`
   - Connect `Publish On X` output to this terminal node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Weather-Adaptive Social Media Promoter Template | Reference implementation pattern for context-aware automation workflows using n8n, OpenWeatherMap, OpenAI, and X. |