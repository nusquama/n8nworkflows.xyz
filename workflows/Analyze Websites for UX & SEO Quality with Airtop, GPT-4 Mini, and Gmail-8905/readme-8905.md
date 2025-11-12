Analyze Websites for UX & SEO Quality with Airtop, GPT-4 Mini, and Gmail

https://n8nworkflows.xyz/workflows/analyze-websites-for-ux---seo-quality-with-airtop--gpt-4-mini--and-gmail-8905


# Analyze Websites for UX & SEO Quality with Airtop, GPT-4 Mini, and Gmail

### 1. Workflow Overview

This workflow automates the analysis of a website’s user experience (UX) and search engine optimization (SEO) metadata quality. It accepts a website URL via form submission, launches a remote browser session through Airtop, and uses AI-driven agents (powered by GPT-4 Mini) combined with simulated browser interactions to explore the website thoroughly. It then analyzes the site’s metadata (title, description, Open Graph image) for SEO quality, compiles a structured UX and SEO report, and sends the report via email through Gmail.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and URL Normalization:** Captures user input URL via form, normalizes it to ensure correct protocol prefix.
- **1.2 Airtop Browser Session Setup:** Creates a remote browser session and opens the target website.
- **1.3 AI-Driven Web Scraper Agent:** Uses GPT-4 Mini to control Airtop tools (click, type, query, load URL) to navigate and analyze multiple subpages for UX evaluation.
- **1.4 Metadata SEO Analysis:** Fetches raw HTML of the main page, extracts metadata, and evaluates SEO quality using AI.
- **1.5 Report Compilation:** Combines UX and SEO analysis into a final structured report.
- **1.6 Email Delivery:** Sends the final report via Gmail to a configured recipient.
- **1.7 Session Termination:** Properly closes the Airtop browser session to free resources.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and URL Normalization

**Overview:**  
Starts the workflow by receiving the website URL from a user-submitted form. It normalizes the URL to ensure it includes the `https://` prefix, which is critical for subsequent HTTP requests and browser navigation.

**Nodes Involved:**  
- On form submission  
- Set RAW data  
- Code in JavaScript  
- set input data

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Captures user input from a form titled "UX Website Analyst" with a required textarea field labeled "Website URL".  
  - *Config:* Form field is mandatory, placeholder examples provided (e.g., example.com).  
  - *Inputs:* None (trigger)  
  - *Outputs:* Passes form data as JSON.

- **Set RAW data**  
  - *Type:* Set  
  - *Role:* Extracts the raw URL string from the form JSON and assigns it to the variable `set_raw_data`.  
  - *Inputs:* From form trigger  
  - *Outputs:* JSON with `set_raw_data`.

- **Code in JavaScript**  
  - *Type:* Code  
  - *Role:* Normalizes the URL by adding `https://` if missing.  
  - *Key Logic:* Checks if URL starts with `http://` or `https://`. If not, prepends `https://`.  
  - *Inputs:* `set_raw_data` from previous node  
  - *Outputs:* JSON with `normalizedUrl`.

- **set input data**  
  - *Type:* Set  
  - *Role:* Stores normalized URL as `website_URL` and a placeholder for Airtop profile name (to be replaced by user).  
  - *Inputs:* From Code node  
  - *Outputs:* JSON with `website_URL` and `Airtop Profile Name (for sites that require authentication)`.

**Edge Cases / Failures:**  
- Missing or malformed URL input (form prevents empty, but malformed URLs may pass).  
- JavaScript code failure if unexpected input type.  
- User forgetting to replace Airtop profile placeholder.

---

#### 2.2 Airtop Browser Session Setup

**Overview:**  
Establishes a remote browser session via Airtop API, opens the target website URL, and retrieves the session and window IDs for downstream operations.

**Nodes Involved:**  
- Session  
- Window  
- Return IDs

**Node Details:**

- **Session**  
  - *Type:* Airtop node (session management)  
  - *Role:* Starts a new browser session using the Airtop profile name provided in input data.  
  - *Config:* Profile name is set dynamically from input data, timeout 5 minutes. Requires Airtop API credentials.  
  - *Inputs:* From `set input data`  
  - *Outputs:* Session details including `sessionId`.

- **Window**  
  - *Type:* Airtop node (window management)  
  - *Role:* Opens the target URL in the browser session. Optionally provides live view (debugging).  
  - *Config:* URL is the normalized `website_URL`. Requires Airtop credentials and session ID from prior node.  
  - *Inputs:* From `Session`  
  - *Outputs:* Window details including `windowId`.

- **Return IDs**  
  - *Type:* Set  
  - *Role:* Extracts and stores `sessionId` and `windowId` from prior nodes for use by AI agent nodes. Also sets a status message.  
  - *Inputs:* From `Window` and `Session`  
  - *Outputs:* JSON with `sessionId`, `windowId`, and status string.

**Edge Cases / Failures:**  
- Airtop API authentication errors.  
- Failure to start session or open the window (network issues, invalid profile).  
- Missing or incorrect URLs causing page load errors.

---

#### 2.3 AI-Driven Web Scraper Agent

**Overview:**  
The core logic that uses a GPT-4 Mini-powered Langchain agent to control the Airtop browser remotely. It simulates browsing actions (clicks, typing, queries, loading URLs) to explore the website, including multiple subpages, and evaluate UX aspects like navigation, design, accessibility, and trust signals.

**Nodes Involved:**  
- Webscraper (Langchain agent)  
- Click (Airtop interaction tool)  
- Query (Airtop extraction tool)  
- Load URL (Airtop window tool)  
- End session (Airtop session termination)  
- Type (Airtop interaction tool)  
- OpenAI Chat Model (GPT-4 Mini, AI model for the agent)

**Node Details:**

- **Webscraper**  
  - *Type:* Langchain agent node (AI agent controlling browser tools)  
  - *Role:* Orchestrates detailed UX analysis by instructing Airtop actions and interpreting page data.  
  - *Config:*  
    - Text prompt instructs detailed UX review: service content, navigation, design clarity, accessibility, trust signals, multiple subpages.  
    - Agent goals and usage instructions embedded in system message.  
    - Uses various Airtop tools to interact with the browser session, identified by `sessionId` and `windowId`.  
    - Retries enabled on failure with 5 seconds interval, max 20 iterations.  
  - *Inputs:* Receives `sessionId`, `windowId`, and `website_URL` from `Return IDs` and `set input data`.  
  - *Outputs:* JSON report of UX analysis.

- **Click / Query / Load URL / Type / End session**  
  - *Type:* Airtop interaction nodes  
  - *Role:* Perform specific browser interactions as commanded by the AI agent.  
  - *Config:* Use dynamic expressions to receive `sessionId` and `windowId` from AI agent context.  
  - *Inputs:* Called internally by `Webscraper` agent through Langchain tool interface.  
  - *Outputs:* Results passed back to AI agent for reasoning.  
  - *Notes:*  
    - Click: Clicks detailed elements on the page.  
    - Query: Asks questions or extracts info from current page.  
    - Load URL: Navigates to new URLs.  
    - Type: Types text into page elements (e.g., search boxes).  
    - End session: Closes browser session after analysis.

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI model node  
  - *Role:* Provides GPT-4 Mini AI capabilities for the Webscraper agent to analyze and plan browsing actions.  
  - *Config:* Uses `gpt-4.1-mini` model. Requires OpenAI API credentials.  
  - *Inputs:* Agent prompts from `Webscraper`.  
  - *Outputs:* AI responses for browser control.

**Edge Cases / Failures:**  
- AI decision errors or inability to find required info on website.  
- Airtop tool failures (e.g., element not found, navigation timeout).  
- API rate limits or network issues with Airtop or OpenAI.  
- Session or window ID desyncs or expirations.

---

#### 2.4 Metadata SEO Analysis

**Overview:**  
Fetches the raw HTML content of the main website URL, extracts critical SEO metadata (title, meta description, Open Graph image), and uses AI to evaluate their quality, strengths, weaknesses, and provides optimization suggestions.

**Nodes Involved:**  
- HTTP Request  
- HTML  
- OpenAI Chat Model3  
- Structured Output Parser  
- Metadata Analyze

**Node Details:**

- **HTTP Request**  
  - *Type:* HTTP Request  
  - *Role:* Performs a GET request to the normalized website URL to retrieve raw HTML content.  
  - *Inputs:* From `Output` node (which receives after session termination)  
  - *Outputs:* Raw page HTML.

- **HTML**  
  - *Type:* HTML Extract  
  - *Role:* Extracts metadata fields using CSS selectors:  
    - Title from `<head><title>` tag  
    - Description from `<meta name="description">` content attribute  
    - Open Graph image from `<meta property="og:image">` content attribute  
  - *Inputs:* Raw HTML from HTTP Request  
  - *Outputs:* Extracted metadata JSON.

- **OpenAI Chat Model3**  
  - *Type:* Langchain OpenAI model  
  - *Role:* Receives extracted metadata and prompts AI to analyze SEO quality and provide structured feedback.  
  - *Config:* Uses GPT-4 Mini model, with a prompt instructing detailed SEO metadata evaluation (presence, quality, strengths, weaknesses, recommendations).  
  - *Inputs:* Metadata JSON from HTML node  
  - *Outputs:* AI-generated textual analysis.

- **Structured Output Parser**  
  - *Type:* Langchain structured output parser  
  - *Role:* Parses AI textual output into structured JSON following a defined schema describing status, strengths, weaknesses, and recommendations for each metadata element.  
  - *Inputs:* AI text from OpenAI Chat Model3  
  - *Outputs:* Structured JSON with SEO metadata analysis.

- **Metadata Analyze**  
  - *Type:* Langchain chain node  
  - *Role:* Combines the parsing step with the AI textual analysis in one chain.  
  - *Inputs:* Extracted metadata from HTML  
  - *Outputs:* Structured metadata SEO review JSON.

**Edge Cases / Failures:**  
- HTTP request failure (site down, invalid URL).  
- HTML extraction misses metadata if tags are non-standard or absent.  
- AI misinterpretation or malformed output not matching JSON schema (parser has auto-fix enabled).  
- OpenAI API errors or rate limits.

---

#### 2.5 Report Compilation

**Overview:**  
Combines UX analysis from the Webscraper agent with SEO metadata analysis into a human-readable final report string, formatted for email delivery.

**Nodes Involved:**  
- Edit Fields

**Node Details:**

- **Edit Fields**  
  - *Type:* Set  
  - *Role:* Constructs a comprehensive report string (`final_raport`) including:  
    - Website URL  
    - Metadata analysis summary (title, description, overall) with strengths, weaknesses, recommendations  
    - UX website analysis output from the Webscraper agent  
  - *Inputs:*  
    - Metadata analysis output from `Metadata Analyze`  
    - Raw metadata from `HTML`  
    - UX analysis output from `Webscraper`  
  - *Outputs:* JSON with `final_raport` string ready for email.

**Edge Cases / Failures:**  
- Missing or malformed input data from previous nodes.  
- Expression evaluation errors if references are broken.

---

#### 2.6 Email Delivery

**Overview:**  
Sends the compiled report via Gmail to a configured recipient email address.

**Nodes Involved:**  
- Send a message

**Node Details:**

- **Send a message**  
  - *Type:* Gmail node  
  - *Role:* Emails the `final_raport` text to the specified recipient.  
  - *Config:*  
    - `sendTo` is a placeholder `[PLACE YOUR EMAIL ADDRESS]` that must be replaced.  
    - Subject includes the analyzed website URL.  
    - Uses Gmail OAuth2 credentials.  
  - *Inputs:* From `Edit Fields`  
  - *Outputs:* Email send confirmation.

**Edge Cases / Failures:**  
- Missing or invalid Gmail credentials.  
- Incorrect recipient email.  
- Gmail API rate limits or network errors.

---

#### 2.7 Session Termination

**Overview:**  
Ensures the Airtop browser session is properly closed after analysis to free resources.

**Nodes Involved:**  
- Terminate a session  
- Output

**Node Details:**

- **Terminate a session**  
  - *Type:* Airtop node (session management)  
  - *Role:* Closes the Airtop session identified by `sessionId` from `Return IDs`.  
  - *Inputs:* From `Webscraper` (after AI agent finishes)  
  - *Outputs:* Confirmation of termination.

- **Output**  
  - *Type:* Set  
  - *Role:* Sets a status message and triggers the HTTP Request to fetch HTML for SEO analysis.  
  - *Inputs:* From `Terminate a session`  
  - *Outputs:* Passes control to HTTP Request node.

**Edge Cases / Failures:**  
- Failure to terminate session causing resource leakage.  
- Session ID mismatch or expired session.

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                                  | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                                                       |
|---------------------|----------------------------------|-------------------------------------------------|--------------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger                     | Captures user website URL input                   | —                        | Set RAW data               | Trigger: Starts workflow, captures URL input, requires "Website URL" field.                                                      |
| Set RAW data         | Set                             | Stores raw URL from form                          | On form submission       | Code in JavaScript          | Trigger block note applies here as well.                                                                                         |
| Code in JavaScript   | Code                            | Normalizes URL by adding https:// if missing     | Set RAW data             | set input data              | Trigger block note applies here as well.                                                                                         |
| set input data       | Set                             | Prepares normalized URL and Airtop profile name | Code in JavaScript       | Session                    | Create Airtop Session: Replace Airtop profile placeholder.                                                                        |
| Session             | Airtop                          | Starts Airtop browser session                     | set input data           | Window                     | Create Airtop Session block note applies here.                                                                                   |
| Window              | Airtop                          | Opens the target URL in browser                   | Session                  | Return IDs                 | Create Airtop Session block note applies here.                                                                                   |
| Return IDs          | Set                             | Extracts sessionId and windowId                   | Window                   | Webscraper                 | Create Airtop Session block note applies here.                                                                                   |
| Webscraper          | Langchain Agent                 | AI-driven UX website analysis                     | Return IDs               | Terminate a session, End session, Load URL, Click, Query, Type | Web Scraper: Core AI agent controlling browser actions, uses GPT-4 Mini.                                                        |
| Click               | Airtop interaction tool          | Clicks elements in browser                         | Called by Webscraper     | Webscraper                 | Web Scraper block note applies.                                                                                                 |
| Query               | Airtop extraction tool           | Queries info from current page                     | Called by Webscraper     | Webscraper                 | Web Scraper block note applies.                                                                                                 |
| Load URL            | Airtop window tool               | Navigates to URLs                                  | Called by Webscraper     | Webscraper                 | Web Scraper block note applies.                                                                                                 |
| Type                | Airtop interaction tool          | Types text into page elements                      | Called by Webscraper     | Webscraper                 | Web Scraper block note applies.                                                                                                 |
| End session          | Airtop session termination       | Ends Airtop session (called internally)           | Called by Webscraper     | Webscraper                 | Web Scraper block note applies.                                                                                                 |
| Terminate a session  | Airtop session termination       | Ensures session closed after scraping             | Webscraper               | Output                     | Terminate Airtop Session: Frees resources, passes sessionId.                                                                     |
| Output               | Set                             | Sets status message and triggers HTTP Request     | Terminate a session      | HTTP Request               | —                                                                                                                                |
| HTTP Request         | HTTP Request                    | Fetches raw HTML of website                        | Output                   | HTML                       | Metadata & Analysis: Fetch HTML for SEO metadata extraction.                                                                    |
| HTML                 | HTML Extract                   | Extracts title, description, and og:image         | HTTP Request             | Metadata Analyze            | Metadata & Analysis block note applies.                                                                                        |
| Metadata Analyze     | Langchain Chain LLM             | Analyzes SEO metadata via GPT-4 Mini AI           | HTML                     | Edit Fields                | Metadata & Analysis block note applies.                                                                                        |
| Structured Output Parser | Langchain Output Parser        | Parses AI metadata analysis output into JSON      | OpenAI Chat Model2       | Metadata Analyze           | Metadata & Analysis block note applies.                                                                                        |
| OpenAI Chat Model    | Langchain OpenAI Chat Model    | Provides GPT-4 Mini AI for Webscraper agent       | Webscraper               | Webscraper                 | Web Scraper block note applies.                                                                                                |
| OpenAI Chat Model2   | Langchain OpenAI Chat Model    | Provides GPT-4 Mini AI for parsing metadata analysis | Metadata Analyze         | Structured Output Parser    | Metadata & Analysis block note applies.                                                                                        |
| OpenAI Chat Model3   | Langchain OpenAI Chat Model    | Provides GPT-4 Mini AI for metadata evaluation    | HTML                     | Metadata Analyze           | Metadata & Analysis block note applies.                                                                                        |
| Edit Fields          | Set                             | Combines UX and SEO results into final report     | Metadata Analyze, Webscraper | Send a message           | Make Final Report: Prepares email content.                                                                                     |
| Send a message       | Gmail                          | Sends final report email                           | Edit Fields              | —                          | Send Answer: Replace email placeholder, configure Gmail OAuth2 credentials.                                                     |
| Sticky Note          | Sticky Note                    | See positions inline                               | —                        | —                          | See detailed notes section below for all sticky notes.                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node**:  
   - Type: Form Trigger  
   - Form Title: "UX Website Analyst"  
   - Form Fields: One textarea labeled "Website URL", required, with placeholder "example.com"  
   - No additional options.

2. **Create Set node "Set RAW data"**:  
   - Assign variable `set_raw_data` to `={{ $json["Website URL"] }}`

3. **Create Code node "Code in JavaScript"**:  
   - Paste JavaScript to normalize URL:  
     ```js
     function normalizeUrl(url) {
       if (!/^https?:\/\//i.test(url)) {
         return "https://" + url;
       }
       return url;
     }

     const inputUrl = $json["set_raw_data"];
     const outputUrl = normalizeUrl(inputUrl);

     return [
       {
         json: {
           normalizedUrl: outputUrl
         }
       }
     ];
     ```
  
4. **Create Set node "set input data"**:  
   - Assign `website_URL` = `={{ $json.normalizedUrl }}`  
   - Assign `Airtop Profile Name (for sites that require authentication)` = `[PLACE YOUR AIRTOP PROFILE NAME]` (to be replaced)

5. **Create Airtop node "Session"**:  
   - Operation: Start session  
   - Profile Name: `={{ $json["Airtop Profile Name (for sites that require authentication)"] }}`  
   - Timeout Minutes: 5  
   - Set Airtop API credentials.

6. **Create Airtop node "Window"**:  
   - Operation: Open window  
   - URL: `={{ $('set input data').item.json.website_URL }}`  
   - Enable live view for debugging (optional)  
   - Use Airtop API credentials.

7. **Create Set node "Return IDs"**:  
   - Assign `sessionId` = `={{ $('Session').item.json.sessionId }}`  
   - Assign `windowId` = `={{ $('Window').item.json.data.windowId }}`  
   - Assign `output` = "Session and window created successfully"

8. **Create Langchain OpenAI Chat Model "OpenAI Chat Model"**:  
   - Model: `gpt-4.1-mini`  
   - Set OpenAI API credentials.

9. **Create Airtop interaction nodes: "Click", "Query", "Load URL", "Type", "End session"**:  
   - Each node uses dynamic expressions to get `sessionId` and `windowId` from AI agent context.  
   - Set required parameters and descriptions as per use case.  
   - Use Airtop API credentials.

10. **Create Langchain Agent node "Webscraper"**:  
    - Text prompt: detailed UX evaluation instructions including multi-subsite navigation and analysis.  
    - System message: agent goals and tool usage instructions for Airtop tools.  
    - Set parameters for max iterations (20), retry on failure, wait between tries.  
    - Configure AI language model input linked to "OpenAI Chat Model".  
    - Set AI tools: connect to Airtop nodes above for interaction.  
    - Inputs for `sessionId`, `windowId`, and `website_URL` from "Return IDs" and "set input data".

11. **Create Airtop node "Terminate a session"**:  
    - Operation: Terminate session  
    - `sessionId` from `Return IDs`

12. **Create Set node "Output"**:  
    - Assign `output` = `={{ $('Webscraper').item.json.output }}` (or status message)  
    - Connect from "Terminate a session" to trigger next block.

13. **Create HTTP Request node**:  
    - Method: GET  
    - URL: `={{ $('set input data').item.json.website_URL }}`

14. **Create HTML Extract node "HTML"**:  
    - Extract:  
      - Title: CSS selector `head > title`  
      - Description: attribute `content` from `meta[name="description"]`  
      - og_image: attribute `content` from `meta[property="og:image"]`

15. **Create Langchain OpenAI Chat Model "OpenAI Chat Model3"**:  
    - Model: `gpt-4.1-mini`  
    - OpenAI credentials  
    - Configured with metadata evaluation prompt.

16. **Create Langchain Output Parser "Structured Output Parser"**:  
    - JSON schema example to parse AI output with metadata fields.

17. **Create Langchain Chain LLM node "Metadata Analyze"**:  
    - Chain includes "OpenAI Chat Model3" and "Structured Output Parser"  
    - Input: output from "HTML"

18. **Create Set node "Edit Fields"**:  
    - Compose `final_raport` string combining:  
      - Website URL  
      - Metadata analysis results (title, description, summary)  
      - UX analysis from `Webscraper` output

19. **Create Gmail node "Send a message"**:  
    - To: `[PLACE YOUR EMAIL ADDRESS]` (replace)  
    - Subject: `New raport about {{ $('set input data').item.json.website_URL }}`  
    - Message: `={{ $json.final_raport }}`  
    - Connect Gmail OAuth2 credentials.

20. **Connect all nodes as per the workflow:**

    - Form Trigger → Set RAW data → Code in JavaScript → set input data → Session → Window → Return IDs → Webscraper → Terminate a session → Output → HTTP Request → HTML → Metadata Analyze → Edit Fields → Send a message.

21. **Add Sticky Notes** (optional but recommended) at positions as per original to explain each block.

22. **Replace placeholders:**  
    - Airtop profile name in "set input data" node.  
    - Recipient email in "Send a message" node.  
    - Set all credentials in corresponding nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow analyzes a website’s UX and SEO by combining browser automation (Airtop), AI-driven browsing control (GPT-4 Mini), and metadata SEO analysis, sending a detailed report by email.                                                                                                                                                                                                                                                                                                                | Workflow purpose summary                                                                         |
| Ensure to replace placeholders: `[PLACE YOUR AIRTOP PROFILE NAME]` and `[PLACE YOUR EMAIL ADDRESS]` before running.                                                                                                                                                                                                                                                                                                                                                                                          | Critical setup instructions                                                                    |
| Airtop credentials are mandatory for remote browser automation; OpenAI API key is required for AI analysis; Gmail OAuth2 credentials are needed for email sending.                                                                                                                                                                                                                                                                                                                                           | Credential requirements                                                                        |
| The workflow normalizes input URLs by adding `https://` if missing to avoid HTTP request or navigation failures.                                                                                                                                                                                                                                                                                                                                                                                             | Input validation note                                                                          |
| Always terminate Airtop sessions after use to avoid resource leaks and unnecessary active browser instances.                                                                                                                                                                                                                                                                                                                                                                                                 | Resource management best practice                                                             |
| UX analysis is driven by an AI agent that uses Airtop tools such as Click, Type, Query, and Load URL, simulating user interactions for thorough multi-subpage exploration.                                                                                                                                                                                                                                                                                                                                   | AI agent and Airtop interaction explanation                                                 |
| Metadata SEO analysis extracts key elements (title, description, og:image) and uses AI to assess quality, strengths, weaknesses, and actionable recommendations in structured JSON.                                                                                                                                                                                                                                                                                                                            | SEO metadata analysis process                                                                 |
| Final report merges UX and SEO insights into a formatted text string sent by Gmail to a configured email.                                                                                                                                                                                                                                                                                                                                                                                                    | Report delivery summary                                                                       |
| Video tutorial for workflow setup and use: https://youtu.be/OmaADzfHrBM                                                                                                                                                                                                                                                                                                                                                                                                                                         | Multimedia setup resource                                                                     |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow built with n8n, complying fully with content policies and using only legal, public data.