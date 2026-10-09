Monitor portfolio correlation and diversification with Google Sheets and Slack

https://n8nworkflows.xyz/workflows/monitor-portfolio-correlation-and-diversification-with-google-sheets-and-slack-20515


# Monitor portfolio correlation and diversification with Google Sheets and Slack

### 1. Workflow Overview

This workflow automates the monthly calculation and monitoring of portfolio diversification and asset correlations. It reads financial holdings from Google Sheets, fetches historical market pricing via an HTTP endpoint, calculates daily returns and Pearson correlation matrices, derives risk and diversification metrics, generates a visual chart, logs historical metrics back to Google Sheets, and publishes a summary report to Slack.

The workflow logic is divided into the following functional blocks:
- **1.1 Trigger and Portfolio Ingestion:** Initiates the monthly schedule and retrieves portfolio asset records from Google Sheets.
- **1.2 Validation and Weight Calculation:** Filters invalid rows and computes normalized portfolio weights and run identifiers.
- **1.3 Market Data Retrieval and Consolidation:** Requests 90 days of historical closing prices for each holding and aggregates the results.
- **1.4 Correlation and Diversification Computation:** Calculates daily returns, builds a Pearson correlation matrix, evaluates pairwise relationships, and computes overall diversification metrics and risk levels.
- **1.5 Persistence, Visualization, and Notification:** Formats and logs summary metrics and pairwise records to Google Sheets, generates a QuickChart bar chart, and dispatches a formatted diversification report to Slack.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Trigger and Portfolio Ingestion
- **Overview:** Starts the workflow on a monthly schedule and extracts raw asset holding entries from the designated Google Sheets spreadsheet.
- **Nodes Involved:** 
  - `Monthly Correlation Analysis Trigger`
  - `Load Portfolio Holdings`
- **Node Details:**
  - **Monthly Correlation Analysis Trigger**
    - *Type and Role:* `n8n-nodes-base.scheduleTrigger` (Trigger). Fires execution monthly at hour 9.
    - *Configuration:* Interval set to months at hour 9.
    - *Input/Output:* Output connects to `Load Portfolio Holdings`.
    - *Edge Cases:* Schedule misfires if the instance is offline during the trigger hour.
  - **Load Portfolio Holdings**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Data Retrieval). Reads rows from the "Portfolio Holdings" tab.
    - *Configuration:* Uses Google Sheets OAuth2 credential; document ID set to `19zJo3qyBTTKhBKCl4nE_HfnuPu41n62noy7xzJ1cFi0`, sheet name "Portfolio Holdings".
    - *Input/Output:* Input from `Monthly Correlation Analysis Trigger`, output to `Validate Portfolio Holdings`.
    - *Edge Cases:* API rate limits, missing sheet tabs, or invalid credentials.

#### Block 1.2: Validation and Weight Calculation
- **Overview:** Ensures each fetched holding contains valid parameters, filters out incomplete rows, and calculates individual asset portfolio weights.
- **Nodes Involved:**
  - `Validate Portfolio Holdings`
  - `Calculate Portfolio Weights`
- **Node Details:**
  - **Validate Portfolio Holdings**
    - *Type and Role:* `n8n-nodes-base.if` (Condition Router). Filters rows to ensure non-empty tickers and asset types with quantities greater than zero.
    - *Configuration:* Condition rules check that `ticker` is not empty, `quantity > 0`, and `asset_type` is not empty.
    - *Input/Output:* Input from `Load Portfolio Holdings`, output connects to `Calculate Portfolio Weights`.
    - *Edge Cases:* All items failing validation results in an empty stream.
  - **Calculate Portfolio Weights**
    - *Type and Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Computes total portfolio quantity, validates that at least two valid holdings exist, generates a unique `run_id`, and assigns normalized weights.
    - *Configuration:* Custom JavaScript block. Throws errors if valid items are fewer than two or total quantity is $\le 0$.
    - *Input/Output:* Input from `Validate Portfolio Holdings`, output connects to `Fetch Historical Market Data`.
    - *Edge Cases:* Throws runtime error if fewer than two valid holdings are processed.

#### Block 1.3: Market Data Retrieval and Consolidation
- **Overview:** Iterates through valid portfolio holdings to request historical pricing data and consolidates the responses into a structured dataset.
- **Nodes Involved:**
  - `Fetch Historical Market Data`
  - `Consolidate Historical Price Data`
- **Node Details:**
  - **Fetch Historical Market Data**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (API Request). Calls a market-data endpoint to fetch historical close prices.
    - *Configuration:* POST method to `http://192.168.101.63:5678/webhook-test/portfolio-market-data` with a 30-second timeout. Sends JSON body containing `ticker`, `asset_type`, and `days: 90`.
    - *Input/Output:* Input from `Calculate Portfolio Weights`, output connects to `Consolidate Historical Price Data`.
    - *Edge Cases:* Endpoint downtime, timeouts, or malformed API responses.
  - **Consolidate Historical Price Data**
    - *Type and Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Aggregates individual price arrays, filters out invalid price entries, sorts by date, and formats the payload for matrix analysis.
    - *Configuration:* Custom JavaScript block. Requires at least two assets with valid prices.
    - *Input/Output:* Input from `Fetch Historical Market Data`, output connects to `Calculate Pairwise Correlations`.
    - *Edge Cases:* Insufficient overlapping data between tickers causes execution failure.

#### Block 1.4: Correlation and Diversification Computation
- **Overview:** Converts historical prices into daily returns, calculates Pearson correlation coefficients for all asset pairs, and derives overall portfolio risk and diversification scores.
- **Nodes Involved:**
  - `Calculate Pairwise Correlations`
  - `Calculate Diversification Metrics`
- **Node Details:**
  - **Calculate Pairwise Correlations**
    - *Type and Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Computes percentage daily returns, calculates pairwise Pearson correlation coefficients, and labels relationships as High, Moderate, or Low.
    - *Configuration:* Custom JavaScript block requiring at least 10 overlapping return observations per pair.
    - *Input/Output:* Input from `Consolidate Historical Price Data`, output connects to `Calculate Diversification Metrics`.
    - *Edge Cases:* Insufficient overlapping trading days throw runtime errors.
  - **Calculate Diversification Metrics**
    - *Type and Role:* `n8n-nodes-base.code` (JavaScript Data Transformation). Computes average correlation, maps it to a diversification score (0–100), determines overall risk level, and identifies highest/lowest correlated pairs.
    - *Configuration:* Custom JavaScript block.
    - *Input/Output:* Input from `Calculate Pairwise Correlations`, outputs fan out to `Build Correlation Chart Configuration`, `Prepare Diversification Summary Record`, and `Prepare Pairwise Correlation Records`.
    - *Edge Cases:* Empty pairwise arrays trigger error handling.

#### Block 1.5: Persistence, Visualization, and Notification
- **Overview:** Logs summary metrics and pairwise records to Google Sheets, builds a QuickChart visualization URL, requests the rendered chart image, and sends a formatted notification to Slack.
- **Nodes Involved:**
  - `Prepare Diversification Summary Record`
  - `Store Diversification Summary`
  - `Prepare Pairwise Correlation Records`
  - `Store Pairwise Correlations`
  - `Build Correlation Chart Configuration`
  - `Generate Correlation Chart`
  - `Prepare Slack Report`
  - `Send Diversification Report`
- **Node Details:**
  - **Prepare Diversification Summary Record**
    - *Type and Role:* `n8n-nodes-base.code` (Data Shaping). Structures portfolio-level summary attributes for spreadsheet insertion.
    - *Input/Output:* Input from `Calculate Diversification Metrics`, output connects to `Store Diversification Summary`.
  - **Store Diversification Summary**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Data Persistence). Appends portfolio summary metrics to the "Diversification Log" tab.
    - *Configuration:* Spreadsheet ID `19zJo3qyBTTKhBKCl4nE_HfnuPu41n62noy7xzJ1cFi0`, operation `append`.
    - *Input/Output:* Input from `Prepare Diversification Summary Record`, output connects to `Prepare Slack Report`.
  - **Prepare Pairwise Correlation Records**
    - *Type and Role:* `n8n-nodes-base.code` (Data Shaping). Flattens pairwise correlation arrays into individual items for batch database or sheet logging.
    - *Input/Output:* Input from `Calculate Diversification Metrics`, output connects to `Store Pairwise Correlations`.
  - **Store Pairwise Correlations**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Data Persistence). Appends individual pair correlation logs to the "Pairwise Correlations" tab.
    - *Configuration:* Spreadsheet ID `19zJo3qyBTTKhBKCl4nE_HfnuPu41n62noy7xzJ1cFi0`, operation `append`.
    - *Input/Output:* Input from `Prepare Pairwise Correlation Records`, outputs none.
  - **Build Correlation Chart Configuration**
    - *Type and Role:* `n8n-nodes-base.code` (Data Shaping). Constructs a Chart.js configuration JSON object for QuickChart.
    - *Input/Output:* Input from `Calculate Diversification Metrics`, output connects to `Generate Correlation Chart`.
  - **Generate Correlation Chart**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (API Request). Requests rendered PNG chart image from QuickChart.io using the encoded chart config URL.
    - *Configuration:* GET method using dynamic expression `={{ $json.quickchart_url }}` with a 30-second timeout.
    - *Input/Output:* Input from `Build Correlation Chart Configuration`, outputs none.
  - **Prepare Slack Report**
    - *Type and Role:* `n8n-nodes-base.code` (Data Shaping). Formats diversification metrics into a readable multi-line plaintext notification message.
    - *Input/Output:* Input from `Store Diversification Summary`, output connects to `Send Diversification Report`.
  - **Send Diversification Report**
    - *Type and Role:* `n8n-nodes-base.slack` (Messaging Integration). Posts the formatted diversification report to the configured Slack channel.
    - *Configuration:* Slack OAuth2 credential, destination channel ID `C0B1LNY15GW`.
    - *Input/Output:* Input from `Prepare Slack Report`, outputs none.
    - *Edge Cases:* Missing channel permissions or invalid Slack credentials.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Monthly Correlation Analysis Trigger | n8n-nodes-base.scheduleTrigger | Fires workflow monthly | None | Load Portfolio Holdings | # Portfolio Correlation Matrix and Diversification Monitor<br><br>This workflow runs a scheduled portfolio correlation analysis using the holdings stored in Google Sheets. It validates the portfolio, calculates holding weights, retrieves historical market prices, consolidates the data, and calculates pairwise correlations from daily returns. The workflow then derives an average correlation and diversification score, identifies the strongest and weakest relationships, generates a correlation visualization, and stores both summary and pairwise results for historical tracking. A concise diversification report is finally sent to Slack.<br><br>### Setup Steps<br><br>1. Configure the scheduled trigger to run the portfolio correlation analysis monthly at the required execution time.<br>2. Configure the Google Sheets credential and connect the Portfolio Holdings sheet containing ticker, quantity, and asset_type fields.<br>3. Configure the production market-data API endpoint and authentication required to retrieve approximately 90 days of historical daily closing prices for each portfolio holding.<br>4. Verify that the historical price response provides ticker information and dated closing prices, and ensure the workflow receives at least two assets with sufficient overlapping observations.<br>5. Configure the Diversification Log and Pairwise Correlations Google Sheets sheets with the required columns for portfolio-level and pairwise historical tracking.<br>6. Configure the Slack credential and destination channel, then validate that the notification contains the diversification score, average correlation, risk level, strongest correlation, weakest correlation, and historical period. |
| Load Portfolio Holdings | n8n-nodes-base.googleSheets | Reads holdings from sheet | Monthly Correlation Analysis Trigger | Validate Portfolio Holdings | ## Analysis Trigger and Portfolio Input<br>This section starts the scheduled portfolio correlation analysis and retrieves the current holdings from Google Sheets. The incoming records are validated to ensure each holding has a valid ticker, positive quantity, and asset type before the workflow continues with portfolio calculations. |
| Validate Portfolio Holdings | n8n-nodes-base.if | Filters out invalid holdings | Load Portfolio Holdings | Calculate Portfolio Weights | ## Analysis Trigger and Portfolio Input<br>This section starts the scheduled portfolio correlation analysis and retrieves the current holdings from Google Sheets. The incoming records are validated to ensure each holding has a valid ticker, positive quantity, and asset type before the workflow continues with portfolio calculations. |
| Calculate Portfolio Weights | n8n-nodes-base.code | Computes portfolio weights | Validate Portfolio Holdings | Fetch Historical Market Data | ## Portfolio Weight and Market Data Preparation<br><br>This section converts the validated holdings into portfolio weights and retrieves historical price information for each asset. The market data is then consolidated into a single structured dataset, ensuring that all assets have usable historical observations for the correlation analysis. |
| Fetch Historical Market Data | n8n-nodes-base.httpRequest | Fetches historical prices | Calculate Portfolio Weights | Consolidate Historical Price Data | ## Portfolio Weight and Market Data Preparation<br><br>This section converts the validated holdings into portfolio weights and retrieves historical price information for each asset. The market data is then consolidated into a single structured dataset, ensuring that all assets have usable historical observations for the correlation analysis. |
| Consolidate Historical Price Data | n8n-nodes-base.code | Aggregates price datasets | Fetch Historical Market Data | Calculate Pairwise Correlations | ## Portfolio Weight and Market Data Preparation<br><br>This section converts the validated holdings into portfolio weights and retrieves historical price information for each asset. The market data is then consolidated into a single structured dataset, ensuring that all assets have usable historical observations for the correlation analysis. |
| Calculate Pairwise Correlations | n8n-nodes-base.code | Calculates correlation matrix | Consolidate Historical Price Data | Calculate Diversification Metrics | ## Correlation and Diversification Analysis<br><br>This section calculates daily returns and Pearson correlation coefficients for every unique holding pair. It then derives the portfolio average correlation and converts it into a diversification score from 0 to 100, while identifying the highest and lowest correlated pairs and assigning an overall risk classification. |
| Calculate Diversification Metrics | n8n-nodes-base.code | Computes overall risk metrics | Calculate Pairwise Correlations | Build Correlation Chart Configuration,<br>Prepare Diversification Summary Record,<br>Prepare Pairwise Correlation Records | ## Correlation and Diversification Analysis<br><br>This section calculates daily returns and Pearson correlation coefficients for every unique holding pair. It then derives the portfolio average correlation and converts it into a diversification score from 0 to 100, while identifying the highest and lowest correlated pairs and assigning an overall risk classification. |
| Build Correlation Chart Configuration | n8n-nodes-base.code | Builds QuickChart config | Calculate Diversification Metrics | Generate Correlation Chart | ## Correlation Visualization<br><br>This section converts the calculated pairwise correlation results into a chart configuration and generates a visual representation of the relationships between portfolio holdings. The resulting chart provides an easy way to identify strongly correlated assets and visually assess the portfolio's diversification characteristics. |
| Generate Correlation Chart | n8n-nodes-base.httpRequest | Requests rendered chart | Build Correlation Chart Configuration | None | ## Correlation Visualization<br><br>This section converts the calculated pairwise correlation results into a chart configuration and generates a visual representation of the relationships between portfolio holdings. The resulting chart provides an easy way to identify strongly correlated assets and visually assess the portfolio's diversification characteristics. |
| Prepare Diversification Summary Record | n8n-nodes-base.code | Prepares summary log payload | Calculate Diversification Metrics | Store Diversification Summary | ## Diversification Reporting and Persistence<br>This section prepares and stores the portfolio-level diversification results, then formats the key metrics into a concise Slack report. It ensures the analysis is both persistently logged for historical tracking and communicated to the team. |
| Store Diversification Summary | n8n-nodes-base.googleSheets | Logs summary to Google Sheets | Prepare Diversification Summary Record | Prepare Slack Report | ## Diversification Reporting and Persistence<br>This section prepares and stores the portfolio-level diversification results, then formats the key metrics into a concise Slack report. It ensures the analysis is both persistently logged for historical tracking and communicated to the team. |
| Prepare Slack Report | n8n-nodes-base.code | Formats Slack notification | Store Diversification Summary | Send Diversification Report | ## Diversification Reporting and Persistence<br>This section prepares and stores the portfolio-level diversification results, then formats the key metrics into a concise Slack report. It ensures the analysis is both persistently logged for historical tracking and communicated to the team. |
| Send Diversification Report | n8n-nodes-base.slack | Posts message to Slack | Prepare Slack Report | None | ## Diversification Reporting and Persistence<br>This section prepares and stores the portfolio-level diversification results, then formats the key metrics into a concise Slack report. It ensures the analysis is both persistently logged for historical tracking and communicated to the team. |
| Prepare Pairwise Correlation Records | n8n-nodes-base.code | Flattens pair records | Calculate Diversification Metrics | Store Pairwise Correlations | ## Pairwise Correlation Persistence<br>This section transforms the calculated pairwise correlations into structured records and stores them in Google Sheets. It preserves each asset relationship, correlation value, absolute correlation, and relationship classification for historical analysis and portfolio diversification tracking. |
| Store Pairwise Correlations | n8n-nodes-base.googleSheets | Logs pair records to sheets | Prepare Pairwise Correlation Records | None | ## Pairwise Correlation Persistence<br>This section transforms the calculated pairwise correlations into structured records and stores them in Google Sheets. It preserves each asset relationship, correlation value, absolute correlation, and relationship classification for historical analysis and portfolio diversification tracking. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node named `Monthly Correlation Analysis Trigger`.
   - Set interval trigger rules to run monthly at hour 9.

2. **Add Portfolio Ingestion:**
   - Add a **Google Sheets** node named `Load Portfolio Holdings`.
   - Configure credential (OAuth2), document ID (`19zJo3qyBTTKhBKCl4nE_HfnuPu41n62noy7xzJ1cFi0`), and select the `Portfolio Holdings` sheet.
   - Connect `Monthly Correlation Analysis Trigger` to `Load Portfolio Holdings`.

3. **Validate Incoming Holdings:**
   - Add an **If** node named `Validate Portfolio Holdings`.
   - Set conditions to verify `ticker` is not empty, `quantity > 0`, and `asset_type` is not empty.
   - Connect `Load Portfolio Holdings` to `Validate Portfolio Holdings`.

4. **Calculate Portfolio Weights:**
   - Add a **Code** node named `Calculate Portfolio Weights`.
   - Paste JavaScript to filter valid items, compute total quantities, create a `run_id`, and calculate individual asset portfolio weights.
   - Connect `Validate Portfolio Holdings` (true branch) to `Calculate Portfolio Weights`.

5. **Fetch Historical Market Data:**
   - Add an **HTTP Request** node named `Fetch Historical Market Data`.
   - Method: `POST`, URL: `http://192.168.101.63:5678/webhook-test/portfolio-market-data`.
   - Body Parameters (JSON): `={{ { ticker: $json.ticker, asset_type: $json.asset_type, days: 90 } }`. Timeout: `30000`.
   - Connect `Calculate Portfolio Weights` to `Fetch Historical Market Data`.

6. **Consolidate Historical Prices:**
   - Add a **Code** node named `Consolidate Historical Price Data`.
   - Add script logic to aggregate price arrays and validate that at least two assets have valid price histories.
   - Connect `Fetch Historical Market Data` to `Consolidate Historical Price Data`.

7. **Calculate Correlations:**
   - Add a **Code** node named `Calculate Pairwise Correlations`.
   - Implement daily return series extraction and Pearson correlation calculations for asset pairs.
   - Connect `Consolidate Historical Price Data` to `Calculate Pairwise Correlations`.

8. **Calculate Diversification Metrics:**
   - Add a **Code** node named `Calculate Diversification Metrics`.
   - Implement logic to compute average correlation, diversification score (0–100), risk levels, and identify highest/lowest pairs.
   - Connect `Calculate Pairwise Correlations` to `Calculate Diversification Metrics`.

9. **Branch 1: Visualization Pipeline:**
   - Add a **Code** node named `Build Correlation Chart Configuration` to format Chart.js JSON and generate a QuickChart URL. Connect `Calculate Diversification Metrics` here.
   - Add an **HTTP Request** node named `Generate Correlation Chart` (GET method using `={{ $json.quickchart_url }}`). Connect `Build Correlation Chart Configuration` to this node.

10. **Branch 2: Summary Logging and Slack Notification:**
    - Add a **Code** node named `Prepare Diversification Summary Record` to shape the summary payload. Connect `Calculate Diversification Metrics` here.
    - Add a **Google Sheets** node named `Store Diversification Summary`. Operation: `append`, sheet name: `Diversification Log`, mapped to summary fields (`run_id`, `calculated_at`, `holding_count`, `average_correlation`, `diversification_score`, `risk_level`). Connect summary prep to this node.
    - Add a **Code** node named `Prepare Slack Report` to generate multi-line plaintext notification text. Connect `Store Diversification Summary` here.
    - Add a **Slack** node named `Send Diversification Report`. Configure Slack OAuth2 credential and select channel `C0B1LNY15GW` with message text `={{ $json.slack_message }}`. Connect Slack preparation to this node.

11. **Branch 3: Pairwise Correlation Persistence:**
    - Add a **Code** node named `Prepare Pairwise Correlation Records` to flatten pairwise arrays. Connect `Calculate Diversification Metrics` here.
    - Add a **Google Sheets** node named `Store Pairwise Correlations`. Operation: `append`, sheet name: `Pairwise Correlations`, mapped to pairwise fields (`run_id`, `ticker_a`, `ticker_b`, `correlation`, `absolute_correlation`, `relationship`). Connect pairwise record preparation to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Professional implementation and customization support for n8n workflows, integrations, and financial analysis. | [Contact WeblineIndia](https://www.weblineindia.com/contact-us.html) |