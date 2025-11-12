Automated Stock Analysis Reports with Technical & News Sentiment using GPT-4o

https://n8nworkflows.xyz/workflows/automated-stock-analysis-reports-with-technical---news-sentiment-using-gpt-4o-3790


# Automated Stock Analysis Reports with Technical & News Sentiment using GPT-4o

### 1. Workflow Overview

This workflow automates comprehensive weekly stock analysis by combining technical chart data and financial news sentiment, producing actionable investment insights in Hebrew with a right-to-left (RTL) formatted HTML email report. It targets investors, traders, and financial analysts seeking data-driven stock recommendations that integrate quantitative indicators with market sentiment.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives user input via a form or workflow trigger, capturing the stock ticker symbol and email.
- **1.2 News Sentiment Analysis:** Fetches and analyzes recent financial news sentiment from Alpha Vantage API.
- **1.3 Technical Data Collection:** Retrieves stock price history and technical indicators (Bollinger Bands, MACD) from Twelve Data API and chart images from Chart-img API.
- **1.4 Technical Analysis Processing:** Calculates support and resistance levels, Fibonacci retracements, and synthesizes technical data.
- **1.5 Visual Chart Analysis:** Uses GPT-4o to analyze chart images for candlestick patterns, RSI, EMA relations, and volume notes.
- **1.6 AI Agent Integration:** Combines technical and news analyses via an AI agent powered by GPT-4o to generate a structured JSON report with detailed insights and recommendations in Hebrew.
- **1.7 Text Refinement and HTML Generation:** Refines Hebrew text for professionalism, generates a styled RTL HTML email report, and adjusts colors dynamically based on sentiment.
- **1.8 Email Delivery:** Sends the final report via SMTP email to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Captures user input (stock ticker and email) via a form or workflow trigger to initiate the analysis.
- **Nodes Involved:**  
  - On form submission  
  - Workflow Input Trigger  
  - Set Stock Symbol and API Key

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point for user input via web form  
    - Config: Form fields for "Ticker symbol" (required) and "Email" (required)  
    - Outputs: Passes form data downstream  
    - Edge Cases: Missing or invalid ticker/email input; form submission errors

  - **Workflow Input Trigger**  
    - Type: Execute Workflow Trigger  
    - Role: Alternative entry point for programmatic workflow execution with inputs "ticker" and "chart_style"  
    - Outputs: Passes inputs downstream  
    - Edge Cases: Missing inputs or invalid ticker format

  - **Set Stock Symbol and API Key**  
    - Type: Set  
    - Role: Assigns variables for ticker symbol and Twelve Data API key for subsequent API calls  
    - Key Variables: `ticker` from input, `TwelveData_API_Key` (empty by default, to be set by user)  
    - Outputs: Passes variables downstream  
    - Edge Cases: Missing or invalid API key

---

#### 1.2 News Sentiment Analysis

- **Overview:** Fetches recent financial news related to the stock from Alpha Vantage, analyzes sentiment, identifies top articles and trending topics.
- **Nodes Involved:**  
  - Generate Variables For API  
  - Set Variables  
  - Get News Data  
  - Analyse API Input

- **Node Details:**

  - **Generate Variables For API**  
    - Type: Code  
    - Role: Generates the "wanted_date" parameter as yesterday's date in the required Alpha Vantage format (YYYYMMDDTHHMM)  
    - Outputs: JSON with `wanted_date`  
    - Edge Cases: Date calculation errors (unlikely)

  - **Set Variables**  
    - Type: Set  
    - Role: Sets variables `wantedDate`, `stockSymbol` (from input), and `apikey` (Alpha Vantage API key, to be configured)  
    - Edge Cases: Missing API key

  - **Get News Data**  
    - Type: HTTP Request  
    - Role: Calls Alpha Vantage NEWS_SENTIMENT API with ticker, date, and API key to fetch news feed  
    - Edge Cases: API rate limits, invalid API key, network errors, empty or malformed responses

  - **Analyse API Input**  
    - Type: Code  
    - Role: Processes news feed JSON to:  
      - Filter relevant articles for the stock  
      - Count sentiment categories (Bullish, Neutral, Bearish, etc.)  
      - Calculate weighted average sentiment score  
      - Identify top 5 influential articles sorted by impact  
      - Aggregate hot topics with article counts and average relevance  
      - Generate overall sentiment description and market outlook in Hebrew  
    - Outputs: Structured JSON with sentiment analysis, top articles, hot topics, and trends  
    - Edge Cases: Missing or incomplete news data, malformed article fields

---

#### 1.3 Technical Data Collection

- **Overview:** Retrieves stock price history and technical indicators (Bollinger Bands, MACD) from Twelve Data API and generates chart images from Chart-img API.
- **Nodes Involved:**  
  - Get Chart URL  
  - Download Chart  
  - Get Price History  
  - Get Bollinger Bands  
  - Get MACD

- **Node Details:**

  - **Get Chart URL**  
    - Type: HTTP Request  
    - Role: Posts to Chart-img API to generate a weekly candlestick chart image URL with volume, EMA(200), RSI indicators  
    - Config: Symbol formatted as NASDAQ:Ticker, style "candle", theme "light", interval "1W"  
    - Credentials: Chart-img API key via HTTP header authentication  
    - Outputs: JSON with chart image URL  
    - Edge Cases: Invalid API key, API downtime, malformed response

  - **Download Chart**  
    - Type: HTTP Request  
    - Role: Downloads the chart image from the URL obtained above  
    - Outputs: Chart image data and URL  
    - Edge Cases: URL invalid or expired, network errors

  - **Get Price History**  
    - Type: HTTP Request  
    - Role: Fetches daily historical price data (last 180 days) from Twelve Data API  
    - Query: symbol, interval=1day, outputsize=180, apikey  
    - Edge Cases: API rate limits, invalid API key, empty data

  - **Get Bollinger Bands**  
    - Type: HTTP Request  
    - Role: Fetches Bollinger Bands indicator data for the stock from Twelve Data API (1-day interval, 1 output)  
    - Edge Cases: API errors, missing data

  - **Get MACD**  
    - Type: HTTP Request  
    - Role: Fetches MACD indicator data for the stock from Twelve Data API (1-day interval, 1 output)  
    - Edge Cases: API errors, missing data

---

#### 1.4 Technical Analysis Processing

- **Overview:** Calculates support and resistance levels and Fibonacci retracements from price history; organizes and synthesizes all technical data into a structured summary.
- **Nodes Involved:**  
  - Calculate Support Resistance  
  - Merge  
  - Organizing Data

- **Node Details:**

  - **Calculate Support Resistance**  
    - Type: Code  
    - Role:  
      - Analyzes historical closing prices to identify local minima (support) and maxima (resistance) using a 5-day lookback window  
      - Calculates Fibonacci retracement levels between min and max prices  
      - Returns support, resistance, Fibonacci levels, current price, and data points count  
    - Edge Cases: Insufficient data points (<30), missing price data

  - **Merge**  
    - Type: Merge  
    - Role: Combines outputs from Calculate Support Resistance, Get Bollinger Bands, and Get MACD nodes into one data stream  
    - Edge Cases: Missing inputs from any source

  - **Organizing Data**  
    - Type: Code  
    - Role:  
      - Consolidates technical data from merged inputs  
      - Extracts Bollinger Bands and MACD values safely  
      - Analyzes technical indicators to generate bullish and bearish factors (e.g., price relative to Bollinger Bands, MACD crossover, proximity to support/resistance and Fibonacci levels)  
      - Produces a summary recommendation: "חיובית" (bullish), "שלילית" (bearish), or "נייטרלית" (neutral) based on factor counts  
    - Edge Cases: Missing or malformed indicator data, NaN values

---

#### 1.5 Visual Chart Analysis

- **Overview:** Uses GPT-4o to analyze the downloaded chart image for visual patterns, RSI, EMA relation, volume notes, and price zones.
- **Nodes Involved:**  
  - First Technical Analysis  
  - Set Variable

- **Node Details:**

  - **First Technical Analysis**  
    - Type: OpenAI GPT-4o (image analysis)  
    - Role: Analyzes weekly candlestick chart image with volume, EMA, RSI panel  
    - Output: Structured JSON with RSI numeric value and state, RSI divergence, trend direction, candlestick patterns (max 3), EMA relation, volume notes, and visually inferred support/resistance price zones  
    - Edge Cases: Image analysis failures, unclear chart visuals, API errors

  - **Set Variable**  
    - Type: Set  
    - Role: Stores the JSON output from the visual analysis for downstream use  
    - Edge Cases: Missing or invalid JSON output

---

#### 1.6 AI Agent Integration

- **Overview:** Combines technical and news sentiment analyses using an AI agent powered by GPT-4o to produce a detailed, structured stock analysis report in Hebrew with investment recommendations.
- **Nodes Involved:**  
  - Window Buffer Memory  
  - AI Agent  
  - Structured Output Parser  
  - Technical Analysis Tool (Sub-Workflow)  
  - Trends Analysis Tool (Sub-Workflow)  
  - Think  
  - Refine Text  
  - Merge-2  
  - Warp as JSON for GPT  
  - ChatGPT 4o  
  - Set Final Response

- **Node Details:**

  - **Window Buffer Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains session context for AI agent interactions using a fixed session key  
    - Edge Cases: Memory overflow or session key conflicts

  - **AI Agent**  
    - Type: LangChain Agent (GPT-4o)  
    - Role:  
      - Receives ticker symbol input  
      - Calls two sub-workflows/tools: "technical_analysis" and "trends_analysis" with the ticker  
      - Merges and analyzes combined data  
      - Generates a structured JSON report with keys for stock symbol, analysis date, recommendation class and text, sentiment counts, technical analysis paragraphs, top articles (translated to Hebrew), and hot topics  
      - Uses "Think" tool to verify recommendation confidence  
      - Enforces Hebrew language and RTL formatting  
    - Key Expressions: Uses templated system message with detailed instructions for output format and translation rules  
    - Edge Cases: AI model errors, incomplete data from sub-workflows, translation inaccuracies

  - **Structured Output Parser**  
    - Type: LangChain Output Parser Structured  
    - Role: Parses AI agent's JSON output to ensure it matches the expected schema for HTML template filling  
    - Edge Cases: Parsing errors, malformed JSON

  - **Technical Analysis Tool** (Sub-Workflow)  
    - Type: LangChain Tool Workflow  
    - Role: Performs detailed technical analysis on the ticker symbol, invoked by AI Agent  
    - Edge Cases: Sub-workflow failures, missing inputs

  - **Trends Analysis Tool** (Sub-Workflow)  
    - Type: LangChain Tool Workflow  
    - Role: Performs news sentiment analysis on the ticker symbol, invoked by AI Agent  
    - Edge Cases: Sub-workflow failures, missing inputs

  - **Think**  
    - Type: LangChain Tool Think  
    - Role: Used by AI Agent to verify recommendation confidence and reasoning  
    - Edge Cases: Think tool timeout or failure

  - **Refine Text**  
    - Type: OpenAI GPT-4o  
    - Role: Refines Hebrew text fields "technicalAnalysis" and "recommendationText" for professional style and correct terminology (e.g., "רצועות בולינגר")  
    - Edge Cases: API errors, text formatting issues

  - **Merge-2**  
    - Type: Merge  
    - Role: Combines refined text and news/technical data for final processing  
    - Edge Cases: Missing inputs

  - **Warp as JSON for GPT**  
    - Type: Code  
    - Role: Formats combined JSON data as a pretty-printed JSON string wrapped in markdown code block for input to ChatGPT 4o node  
    - Edge Cases: JSON serialization errors

  - **ChatGPT 4o**  
    - Type: OpenAI GPT-4o  
    - Role: Senior technical analyst role to synthesize visual and quantitative data into five titled sections: Quick Stats, Candles and EMA, RSI, Indicator Synthesis, and Actionable Takeaway, strictly data-driven and concise in Hebrew  
    - Edge Cases: API errors, incomplete input data

  - **Set Final Response**  
    - Type: Set  
    - Role: Stores final textual response and chart image URL for downstream use  
    - Edge Cases: Missing or malformed input

---

#### 1.7 Text Refinement and HTML Generation

- **Overview:** Generates a fully styled, responsive HTML email report in Hebrew with RTL layout, dynamically adjusts colors based on sentiment, and cleans up the HTML content.
- **Nodes Involved:**  
  - Generate HTML  
  - Adjust HTML Colors

- **Node Details:**

  - **Generate HTML**  
    - Type: HTML  
    - Role: Uses a detailed HTML template with inline CSS for RTL Hebrew layout, embedding data from AI Agent output such as stock symbol, analysis date, recommendation, technical analysis, sentiment charts, top articles, and hot topics  
    - Edge Cases: Missing data fields, template rendering errors

  - **Adjust HTML Colors**  
    - Type: Code  
    - Role:  
      - Dynamically updates colors of recommendation titles, sentiment bars, article tags, and buttons based on sentiment classification (positive, neutral, negative)  
      - Removes topics with only one article to reduce clutter  
      - Removes undefined or placeholder articles from the HTML  
      - Logs debug information for sentiment classification  
    - Edge Cases: HTML parsing errors, regex mismatches, incomplete data

---

#### 1.8 Email Delivery

- **Overview:** Sends the generated HTML report via SMTP email to the user-provided email address.
- **Nodes Involved:**  
  - Send Stock Analysis

- **Node Details:**

  - **Send Stock Analysis**  
    - Type: Email Send  
    - Role: Sends the final HTML email with subject including stock symbol and analysis date, from a configured sender email  
    - Parameters: Uses SMTP credentials configured by user (host, port, username, password)  
    - Edge Cases: SMTP authentication failures, email delivery errors, spam filtering

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                                    | Input Node(s)                        | Output Node(s)                   | Sticky Note                                                                                                      |
|----------------------------|----------------------------------|---------------------------------------------------|------------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------|
| On form submission          | Form Trigger                     | Receives user input (ticker, email)                | -                                  | AI Agent                        |                                                                                                                 |
| Workflow Input Trigger      | Execute Workflow Trigger         | Alternative input trigger with ticker and chart_style | -                                  | Set Stock Symbol and API Key    |                                                                                                                 |
| Set Stock Symbol and API Key| Set                             | Sets ticker and Twelve Data API key                 | Workflow Input Trigger              | Get Chart URL, Get Price History, Get Bollinger Bands, Get MACD | Sticky Note19: Replace TwelveData API Key                                                                        |
| Generate Variables For API  | Code                            | Generates yesterday's date for Alpha Vantage API   | Schedule Trigger1                   | Set Variables                  |                                                                                                                 |
| Set Variables              | Set                             | Sets variables for news API call (wantedDate, stockSymbol, apikey) | Generate Variables For API          | Get News Data                  |                                                                                                                 |
| Get News Data              | HTTP Request                    | Fetches news sentiment data from Alpha Vantage     | Set Variables                      | Analyse API Input              |                                                                                                                 |
| Analyse API Input          | Code                            | Processes news data to extract sentiment and topics | Get News Data                      | Merge-2                       |                                                                                                                 |
| Get Chart URL              | HTTP Request                    | Requests chart image URL from Chart-img API         | Set Stock Symbol and API Key       | Download Chart                | Sticky Note17: Replace Chart-img API Key                                                                        |
| Download Chart             | HTTP Request                    | Downloads chart image                                | Get Chart URL                     | First Technical Analysis      |                                                                                                                 |
| Get Price History          | HTTP Request                    | Fetches historical price data from Twelve Data API  | Set Stock Symbol and API Key       | Calculate Support Resistance  | Sticky Note19: Replace TwelveData API Key                                                                        |
| Get Bollinger Bands        | HTTP Request                    | Fetches Bollinger Bands indicator data              | Set Stock Symbol and API Key       | Merge                        | Sticky Note19: Replace TwelveData API Key                                                                        |
| Get MACD                  | HTTP Request                    | Fetches MACD indicator data                          | Set Stock Symbol and API Key       | Merge                        | Sticky Note19: Replace TwelveData API Key                                                                        |
| Calculate Support Resistance| Code                            | Calculates support, resistance, Fibonacci levels    | Get Price History                 | Merge                        |                                                                                                                 |
| Merge                     | Merge                           | Combines technical indicator data                    | Calculate Support Resistance, Get Bollinger Bands, Get MACD | Organizing Data              |                                                                                                                 |
| Organizing Data           | Code                            | Synthesizes technical data and generates summary    | Merge                            | Merge-2                      |                                                                                                                 |
| First Technical Analysis  | OpenAI GPT-4o (Image Analysis)  | Analyzes chart image for visual technical indicators | Download Chart                   | Set Variable                 | Sticky Note8: Replace OpenAI Credentials                                                                         |
| Set Variable              | Set                             | Stores visual analysis JSON output                   | First Technical Analysis          | Merge-2                      |                                                                                                                 |
| Merge-2                   | Merge                           | Combines news sentiment and technical analysis data | Organizing Data, Analyse API Input, Set Variable | Warp as JSON for GPT          |                                                                                                                 |
| Warp as JSON for GPT      | Code                            | Formats combined JSON as markdown-wrapped string    | Merge-2                         | ChatGPT 4o                   |                                                                                                                 |
| ChatGPT 4o                | OpenAI GPT-4o                   | Synthesizes final detailed analysis report          | Warp as JSON for GPT             | Set Final Response           | Sticky Note15: Replace OpenAI Credentials                                                                        |
| Set Final Response        | Set                             | Stores final textual response and chart image URL   | ChatGPT 4o                     | Generate HTML                |                                                                                                                 |
| AI Agent                  | LangChain Agent (GPT-4o)        | Core AI agent combining technical and news analysis | On form submission, Window Buffer Memory, Technical Analysis Tool, Trends Analysis Tool | Refine Text                 | Sticky Note: AI Agent powered by GPT-4o for combined stock analysis in Hebrew                                    |
| Window Buffer Memory      | LangChain Memory Buffer         | Maintains session context for AI agent               | -                              | AI Agent                    |                                                                                                                 |
| Technical Analysis Tool   | LangChain Tool Workflow         | Sub-workflow for detailed technical analysis         | AI Agent (tool call)            | AI Agent                    | Sticky Note1: Technical Analysis Tool combining multiple APIs and GPT-4o                                        |
| Trends Analysis Tool      | LangChain Tool Workflow         | Sub-workflow for news sentiment analysis              | AI Agent (tool call)            | AI Agent                    | Sticky Note2: Trends Analysis Tool using Alpha Vantage news API                                                |
| Think                     | LangChain Tool Think            | Verification tool for AI agent recommendations        | AI Agent (ai_tool)              | AI Agent                    |                                                                                                                 |
| Refine Text               | OpenAI GPT-4o                  | Refines Hebrew text for professionalism               | AI Agent                      | Generate HTML               | Sticky Note14: Replace OpenAI Credentials                                                                       |
| Generate HTML             | HTML                           | Generates styled RTL Hebrew HTML email report         | Refine Text                   | Adjust HTML Colors          |                                                                                                                 |
| Adjust HTML Colors        | Code                           | Dynamically adjusts colors and cleans HTML content    | Generate HTML                 | Send Stock Analysis         | Sticky Note13: Replace SMTP Credentials                                                                         |
| Send Stock Analysis       | Email Send                     | Sends final email report to user                       | Adjust HTML Colors            | -                           |                                                                                                                 |
| Schedule Trigger1         | Schedule Trigger               | Triggers workflow on schedule (weekly)                 | -                              | Generate Variables For API  |                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Input Nodes:**
   - Add a **Form Trigger** node named "On form submission" with fields:  
     - "Ticker symbol:" (text, required)  
     - "Email:" (email, required)  
     Configure response message: "Success! Check your inbox (or spam folder) for your analysis report."
   - Add an **Execute Workflow Trigger** node named "Workflow Input Trigger" with inputs:  
     - "ticker" (string)  
     - "chart_style" (string, optional, deprecated)

2. **Set Initial Variables:**
   - Add a **Set** node "Set Stock Symbol and API Key" connected to "Workflow Input Trigger" or "On form submission" (depending on entry).  
   - Assign variables:  
     - `ticker` = input ticker  
     - `TwelveData_API_Key` = your Twelve Data API key (string)

3. **Technical Data Collection:**
   - Add **HTTP Request** node "Get Chart URL":  
     - POST to `https://api.chart-img.com/v2/tradingview/advanced-chart/storage`  
     - JSON body with style "candle", theme "light", interval "1W", symbol "NASDAQ:{{ticker}}"  
     - Include studies: Volume, EMA(200), RSI  
     - Use HTTP Header Auth with Chart-img API key  
   - Connect to **HTTP Request** node "Download Chart" to download the image from the URL returned.

   - Add **HTTP Request** nodes for Twelve Data API:  
     - "Get Price History": GET `https://api.twelvedata.com/time_series` with parameters: symbol={{ticker}}, interval=1day, outputsize=180, apikey  
     - "Get Bollinger Bands": GET `https://api.twelvedata.com/bbands` with symbol, interval=1day, outputsize=1, apikey  
     - "Get MACD": GET `https://api.twelvedata.com/macd` with symbol, interval=1day, outputsize=1, apikey

4. **Calculate Support and Resistance:**
   - Add a **Code** node "Calculate Support Resistance" that:  
     - Processes price history to find Fibonacci levels and local support/resistance points  
     - Returns structured JSON with these levels and current price

5. **Merge Technical Data:**
   - Add a **Merge** node to combine outputs of "Calculate Support Resistance", "Get Bollinger Bands", and "Get MACD".

6. **Organize Technical Data:**
   - Add a **Code** node "Organizing Data" that:  
     - Consolidates merged technical data  
     - Analyzes Bollinger Bands, MACD, support/resistance, Fibonacci levels  
     - Generates bullish/bearish factors and a summary recommendation

7. **Visual Chart Analysis:**
   - Add an **OpenAI GPT-4o** node "First Technical Analysis" configured for image analysis:  
     - Input: base64-encoded chart image from "Download Chart"  
     - Prompt instructs to analyze weekly candlestick chart with volume, EMA, RSI panel  
     - Output: structured JSON with RSI, trend, candlestick patterns, EMA relation, volume notes, price zones  
   - Add a **Set** node "Set Variable" to store this JSON output.

8. **News Sentiment Analysis:**
   - Add a **Schedule Trigger** node "Schedule Trigger1" to run weekly or as desired.
   - Add a **Code** node "Generate Variables For API" to generate yesterday's date in Alpha Vantage format.
   - Add a **Set** node "Set Variables" to assign `wantedDate`, `stockSymbol`, and Alpha Vantage `apikey`.
   - Add an **HTTP Request** node "Get News Data" to call Alpha Vantage NEWS_SENTIMENT API with ticker, date, and API key.
   - Add a **Code** node "Analyse API Input" to process news data: filter relevant articles, calculate sentiment counts and scores, identify top articles and hot topics, generate overall sentiment and market outlook.

9. **Merge News and Technical Data:**
   - Add a **Merge** node "Merge-2" to combine outputs from "Organizing Data", "Analyse API Input", and "Set Variable" (visual analysis).

10. **AI Agent Integration:**
    - Add a **LangChain Memory Buffer Window** node "Window Buffer Memory" with a fixed session key.
    - Add a **LangChain Agent** node "AI Agent" configured with:  
      - Input: ticker symbol from form or trigger  
      - Calls two sub-workflows/tools: "technical_analysis" and "trends_analysis" (configured separately)  
      - Uses GPT-4o with detailed system prompt to combine data and produce structured JSON report in Hebrew with recommendations  
      - Uses "Think" tool for recommendation verification  
    - Add **LangChain Output Parser Structured** node "Structured Output Parser" to validate AI output.

11. **Sub-Workflows Setup:**
    - **Technical Analysis Tool:** Workflow that performs technical analysis on ticker symbol, using APIs and GPT-4o for visual pattern recognition.  
    - **Trends Analysis Tool:** Workflow that performs news sentiment analysis on ticker symbol using Alpha Vantage API.

12. **Refine Text:**
    - Add an **OpenAI GPT-4o** node "Refine Text" to polish Hebrew text fields "technicalAnalysis" and "recommendationText" for professionalism and correct terminology.

13. **Generate Final HTML:**
    - Add an **HTML** node "Generate HTML" with the provided RTL Hebrew email template, embedding AI Agent output fields.
    - Add a **Code** node "Adjust HTML Colors" to dynamically update colors based on sentiment, remove topics with only one article, and clean undefined articles.

14. **Send Email:**
    - Add an **Email Send** node "Send Stock Analysis" configured with SMTP credentials:  
      - Host (e.g., smtp.gmail.com), Port (465 or 587), Username, Password  
      - From email: e.g., "Elay's AI Assistant <elayguez@gmail.com>"  
      - To email: from form input  
      - Subject: includes stock symbol and analysis date  
      - HTML body: from "Adjust HTML Colors"

15. **Connect Nodes:**
    - Connect nodes following the logical flow described above, ensuring data dependencies are respected.

16. **Credential Setup:**
    - Configure credentials for:  
      - OpenAI API (GPT-4o)  
      - Chart-img API (HTTP Header Auth)  
      - Twelve Data API Key  
      - Alpha Vantage API Key  
      - SMTP Email

17. **Activate Workflow:**
    - Test with sample ticker and email input.  
    - Monitor logs for errors or API limits.  
    - Adjust parameters or API keys as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This report is generated automatically and does not constitute investment advice. Consult a licensed advisor before investing. | Important legal disclaimer in workflow description                                               |
| Built by [Elay Guez](https://www.linkedin.com/in/elay-g)                                              | Project credit                                                                                   |
| Setup instructions and required credentials detailed in Sticky Note and workflow description           | Workflow setup guidance                                                                          |
| The HTML email template is fully RTL and styled for Hebrew, including dynamic color adjustments based on sentiment | Email formatting and styling details                                                            |
| Uses multiple APIs: Chart-img, Twelve Data, Alpha Vantage, OpenAI GPT-4o                                | External integrations                                                                            |
| Sub-workflows "technical_analysis" and "trends_analysis" must be imported and configured separately    | Modular workflow design                                                                          |
| For best results, ensure API keys have sufficient quota and SMTP credentials allow sending emails       | Operational considerations                                                                      |

---

This structured documentation enables advanced users and AI agents to fully understand, reproduce, and customize the Automated Stock Analysis Reports workflow with GPT-4o, ensuring robust integration of technical and news sentiment analyses into actionable Hebrew-language investment insights.