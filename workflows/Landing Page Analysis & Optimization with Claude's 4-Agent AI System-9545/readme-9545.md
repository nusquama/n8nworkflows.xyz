Landing Page Analysis & Optimization with Claude's 4-Agent AI System

https://n8nworkflows.xyz/workflows/landing-page-analysis---optimization-with-claude-s-4-agent-ai-system-9545


# Landing Page Analysis & Optimization with Claude's 4-Agent AI System

---

### 1. Workflow Overview

This workflow, titled **"Landing Page Analysis & Optimization with Claude's 4-Agent AI System,"** automates the process of auditing a landing page using a multi-agent AI system. Its primary purpose is to provide users with a comprehensive, AI-powered audit covering design, copywriting, SEO, and growth strategy aspects of their landing pages. The workflow is designed for marketers, conversion rate optimization specialists, and business owners seeking a rapid, data-driven evaluation and actionable recommendations to improve landing page performance and conversions.

The workflow logic is organized into the following functional blocks:

- **1.1 Form Input Reception:** Captures user-submitted landing page data including URL, industry, goals, and contact email.
- **1.2 Data Initialization:** Normalizes and stores user inputs as variables for downstream use.
- **1.3 Web Scraping & Content Extraction:** Fetches the HTML content of the landing page and extracts critical elements such as headings, CTAs, images, technology detection, and meta-information.
- **1.4 AI Multi-Agent Analysis:** Four AI agents independently analyze the landing page data from different expert perspectives:
  - Design Critic (UX/UI and visual design)
  - Copywriter (messaging and copy effectiveness)
  - SEO Specialist (technical SEO and performance)
  - Growth Strategist (synthesis and strategic recommendations)
- **1.5 Report Formatting and Email Delivery:** Aggregates AI outputs into a styled HTML email report and sends it to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Form Input Reception

**Overview:**  
Captures user input via a web form designed for landing page audit requests. This node serves as the workflow's trigger.

**Nodes Involved:**  
- Form Submission

**Node Details:**

- **Form Submission**  
  - Type: `formTrigger` (Webhook trigger node)  
  - Role: Initiates workflow on form submission.  
  - Configuration:  
    - Form titled "Free Landing Page Audit - By Evervise"  
    - Fields include Landing Page URL (required), Industry, Primary Goal (dropdown, required), Current Conversion Rate, Biggest Frustration, and Email (required).  
    - Form description promotes the audit as a $2,000 professional service powered by 4 AI specialists.  
  - Input: User HTTP POST form data  
  - Output: JSON with user-submitted fields.  
  - Edge Cases: Missing required fields result in no trigger; malformed URLs or emails might need validation (not handled here).  
  - Version: 2.3  

---

#### 1.2 Data Initialization

**Overview:**  
Assigns user inputs to workflow variables with default fallback values and records a timestamp for the analysis.

**Nodes Involved:**  
- Initialize Variables

**Node Details:**

- **Initialize Variables**  
  - Type: `set` node  
  - Role: Normalizes and assigns form data to variables with defaults:  
    - landing_page_url, industry (default 'General'), primary_goal, current_conversion (default 'Unknown'), biggest_frustration (default 'Not specified'), user_email, analysis_timestamp (ISO string).  
  - Input: JSON from Form Submission  
  - Output: JSON with standardized field names and values  
  - Edge Cases: Missing optional fields handled with defaults; timestamp ensures unique analysis reference.  
  - Version: 3.4  

---

#### 1.3 Web Scraping & Content Extraction

**Overview:**  
Fetches the landing page HTML and extracts key elements such as titles, meta tags, headings, CTAs, images, technology detection, and other SEO/UX relevant data for AI analysis.

**Nodes Involved:**  
- Fetch Page HTML  
- Extract Page Elements  
- Scrape single URL (alternative scraping via Apify)  
- Crawl4AI (alternative crawling with enhanced capabilities)  

**Node Details:**

- **Fetch Page HTML**  
  - Type: `httpRequest`  
  - Role: Requests the landing page HTML content using the URL variable.  
  - Config: 10s timeout, returns full response, continues on failure to avoid stopping workflow.  
  - Input: landing_page_url from Initialize Variables  
  - Output: Full HTTP response including HTML body  
  - Edge Cases: Timeout, 404 or other HTTP errors (handled by continueOnFail), invalid URLs.  
  - Version: 4.1  

- **Extract Page Elements**  
  - Type: `code` (JavaScript)  
  - Role: Parses HTML to extract:  
    - Page title, meta description  
    - H1 and H2 headings (limited to first 5 or 10)  
    - CTA buttons and links with typical CTA text  
    - Form counts  
    - Word count estimate (text content only)  
    - Image count and alt attribute presence  
    - Mobile viewport tag  
    - Detects use of React, Bootstrap, Tailwind, Google Analytics, Facebook Pixel  
    - Provides a 3000-char HTML preview snippet  
  - Input: HTML response from Fetch Page HTML  
  - Output: Structured JSON with extracted page data  
  - Edge Cases: Malformed HTML, missing tags, scripts/styles stripped before word count  
  - Version: 2  

- **Scrape single URL (Apify)**  
  - Type: Apify node  
  - Role: Alternative scraper to fetch and process landing page content robustly via Apify platform.  
  - Input: landing_page_url  
  - Output: Scraped page data for Extract Page Elements (used optionally)  
  - Edge Cases: Apify API limits or errors, authentication required  
  - Version: 1  

- **Crawl4AI**  
  - Type: HTTP Request  
  - Role: Alternative advanced crawler with headless browser config and multi-page crawl (max 10 pages, depth 2).  
  - Requires HTTP Header Auth credential.  
  - Input: landing_page_url  
  - Output: Crawl results for analysis (connected to Extract Page Elements)  
  - Edge Cases: API errors, authentication errors, slow response  
  - Version: 4.2  

---

#### 1.4 AI Multi-Agent Analysis

**Overview:**  
Four specialized AI agents analyze the extracted landing page data independently, focusing on design, copywriting, SEO, and growth strategy, producing scored, detailed feedback and recommendations.

**Nodes Involved:**  
- Design Critic Model  
- Agent 1: Design Critic  
- Copywriter Model  
- Agent 2: Copywriter  
- SEO Specialist Model  
- Agent 3: SEO & Technical  
- Growth Strategist Model  
- Agent 4: Growth Strategist  

**Node Details:**

- **Design Critic Model**  
  - Type: LangChain Anthropic LM Chat node  
  - Role: Provides the language model backend for Agent 1  
  - Config: Model "claude-sonnet-4-5-20250929" with temperature 0.4 and topP 0.9  
  - Credential: Anthropic API  
  - Input: Prompt from Agent 1 node  
  - Output: AI response to design critique prompt  
  - Edge Cases: API errors, rate limits  

- **Agent 1: Design Critic**  
  - Type: LangChain Agent  
  - Role: UX/UI expert evaluating visual hierarchy, CTAs, whitespace, color, mobile, trust signals, with scoring and detailed recommendations.  
  - Key Expressions: Uses extracted page elements and initialized variables to build prompt, requesting structured output with scores and actionable insights.  
  - Input: Extract Page Elements output + Initialize Variables  
  - Output: Design analysis text  
  - Edge Cases: Incomplete page data may reduce analysis quality  

- **Copywriter Model**  
  - Type: LangChain Anthropic LM Chat node  
  - Role: Language model for Agent 2  
  - Config: Same model with temperature 0.5, topP 0.9  
  - Credential: Anthropic API  

- **Agent 2: Copywriter**  
  - Type: LangChain Agent  
  - Role: Assesses headline impact, value proposition clarity, benefits vs. features, emotional triggers, CTA copy, and overall messaging.  
  - Input: Extracted page copy data + variables  
  - Output: Copywriting analysis with scoring and rewrite suggestions  

- **SEO Specialist Model**  
  - Type: LangChain Anthropic LM Chat node  
  - Role: Language model for Agent 3  
  - Config: Temperature 0.3, topP 0.85  
  - Credential: Anthropic API  

- **Agent 3: SEO & Technical**  
  - Type: LangChain Agent  
  - Role: Reviews SEO fundamentals, meta tags, headings, image optimization, mobile readiness, analytics presence, and technical health.  
  - Input: Page elements and variables  
  - Output: SEO & technical analysis with scores and recommendations  

- **Growth Strategist Model**  
  - Type: LangChain Anthropic LM Chat node  
  - Role: Language model for Agent 4  
  - Config: Temperature 0.6, topP 0.95  
  - Credential: Anthropic API  

- **Agent 4: Growth Strategist**  
  - Type: LangChain Agent  
  - Role: Synthesizes all agent analyses into a strategic report including an overall grade (A+ to F), scorecard, top 5 prioritized actions, estimated conversion lift, and high-level business recommendations.  
  - Input: Outputs from Agents 1, 2, 3 + Initialize Variables  
  - Output: Comprehensive strategic analysis  

- **Flow Logic:**  
  - Extract Page Elements triggers Agent 1: Design Critic  
  - Agent 1 triggers Agent 2: Copywriter  
  - Agent 2 triggers Agent 3: SEO & Technical  
  - Agents 2 and 3 also connected to their respective models  
  - Agent 3 triggers Agent 4: Growth Strategist  
  - Agent 4 triggers email formatting  

- Edge Cases: API failures, timeout, incomplete data, inconsistent AI output format.

---

#### 1.5 Report Formatting and Email Delivery

**Overview:**  
Combines all AI analyses into a professional HTML email with branding, score badges, detailed sections, and calls to action, then sends the report to the user.

**Nodes Involved:**  
- Format Email Report  
- Send Email Report  

**Node Details:**

- **Format Email Report**  
  - Type: `code` (JavaScript)  
  - Role: Generates a styled, responsive HTML email embedding:  
    - Landing page grade and score with colored badge  
    - Sections for strategic overview, design, copy, SEO analyses  
    - Page statistics summary  
    - Calls to action linking to paid services and booking  
    - Footer with company info and links  
  - Uses current date, colors based on grade, and extracts data from agent outputs and variables.  
  - Output: JSON with email_html, email_subject, user_email, grade, score, url, timestamp  
  - Edge Cases: Missing AI outputs may lead to incomplete sections; HTML must render correctly across clients.  

- **Send Email Report**  
  - Type: Gmail node  
  - Role: Sends the email to the user email address with subject and HTML content from formatting node.  
  - Credential: Gmail OAuth2 (user "Sofia Wagner")  
  - Input: Outputs from formatting node  
  - Edge Cases: Gmail API limits, email deliverability, invalid addresses  

---

### 3. Summary Table

| Node Name             | Node Type                           | Functional Role                        | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                                  |
|-----------------------|-----------------------------------|-------------------------------------|---------------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview     | stickyNote                        | Provides high-level workflow summary| -                         | -                          | 📋 WORKFLOW OVERVIEW: AI Landing Page Analyzer & Optimizer; performance, funnel, optimization ideas                           |
| Form Submission       | formTrigger                      | User input form trigger               | -                         | Initialize Variables        | 📝 FORM INTAKE: User submits landing page audit; design tips and customization options                                         |
| Initialize Variables  | set                              | Normalize and assign input variables | Form Submission            | Fetch Page HTML             |                                                                                                                              |
| Fetch Page HTML       | httpRequest                      | Fetch landing page HTML               | Initialize Variables       | Extract Page Elements       | 🌐 WEB SCRAPING: Fetch HTML, extract elements, detect tech; enhancement ideas                                                  |
| Scrape single URL     | Apify                           | Alternative page scraping             | Initialize Variables       | Extract Page Elements       | 🌐 WEB SCRAPING (alternative): robust scraping via Apify                                                                      |
| Crawl4AI              | httpRequest                     | Alternative crawling with browser    | Initialize Variables       | Extract Page Elements       | 🌐 WEB SCRAPING (alternative): advanced crawler; auth required                                                                |
| Extract Page Elements | code                            | Parse HTML & extract key elements    | Fetch Page HTML / Scrape single URL / Crawl4AI | Agent 1: Design Critic      |                                                                                                                              |
| Design Critic Model   | lmChatAnthropic (LangChain)     | Provides LM for Design Critic agent  | Agent 1: Design Critic     | Agent 1: Design Critic      | 🤖 AGENT 1 INFO: UX/UI analysis expert; focus areas and model settings                                                        |
| Agent 1: Design Critic| langchain.agent                 | UX/UI design analysis                 | Extract Page Elements      | Agent 2: Copywriter         |                                                                                                                              |
| Copywriter Model      | lmChatAnthropic (LangChain)     | Provides LM for Copywriter agent     | Agent 2: Copywriter        | Agent 2: Copywriter         | 🤖 AGENT 2 INFO: Messaging & persuasion expert; focus areas and model settings                                                |
| Agent 2: Copywriter   | langchain.agent                 | Copywriting & messaging analysis     | Agent 1: Design Critic     | Agent 3: SEO & Technical    |                                                                                                                              |
| SEO Specialist Model  | lmChatAnthropic (LangChain)     | Provides LM for SEO agent            | Agent 3: SEO & Technical   | Agent 3: SEO & Technical    | 🤖 AGENT 3 INFO: SEO specialist; technical analysis focus and model settings                                                  |
| Agent 3: SEO & Technical| langchain.agent               | SEO and technical site analysis      | Agent 2: Copywriter        | Agent 4: Growth Strategist  |                                                                                                                              |
| Growth Strategist Model| lmChatAnthropic (LangChain)     | Provides LM for Growth Strategist    | Agent 4: Growth Strategist | Agent 4: Growth Strategist  | 🤖 AGENT 4 INFO: Synthesis & strategy expert; role and model settings                                                        |
| Agent 4: Growth Strategist| langchain.agent             | Synthesizes all analyses into strategy| Agent 3: SEO & Technical   | Format Email Report         |                                                                                                                              |
| Format Email Report   | code                            | Generates final HTML email report    | Agent 4: Growth Strategist | Send Email Report           | 📧 EMAIL DELIVERY: Beautiful HTML report; includes grades, analyses, stats, CTAs, and company branding                         |
| Send Email Report     | gmail                           | Sends report email to user           | Format Email Report        | -                          |                                                                                                                              |
| Form Section          | stickyNote                      | Notes on form intake best practices  | -                         | -                          | 📝 FORM INTAKE sticky note (see Form Submission)                                                                              |
| Scraping Section      | stickyNote                      | Notes on scraping and extraction     | -                         | -                          | 🌐 WEB SCRAPING sticky note (see Fetch Page HTML and Extract Page Elements)                                                   |
| Agent 1 Info          | stickyNote                      | Design Critic agent info and focus   | -                         | -                          | 🤖 AGENT 1 INFO sticky note                                                                                                   |
| Agent 2 Info          | stickyNote                      | Copywriter agent info and focus      | -                         | -                          | 🤖 AGENT 2 INFO sticky note                                                                                                   |
| Agent 3 Info          | stickyNote                      | SEO Specialist agent info and focus | -                         | -                          | 🤖 AGENT 3 INFO sticky note                                                                                                   |
| Agent 4 Info          | stickyNote                      | Growth Strategist agent info         | -                         | -                          | 🤖 AGENT 4 INFO sticky note                                                                                                   |
| Email Section         | stickyNote                      | Email report delivery notes          | -                         | -                          | 📧 EMAIL DELIVERY sticky note                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Submission node:**  
   - Type: `formTrigger`  
   - Configure webhook ID (e.g., "lp-analyzer-form-001")  
   - Set form title: "Free Landing Page Audit - By Evervise"  
   - Add fields:  
     - Landing Page URL (text, required)  
     - Industry (text, optional)  
     - Primary Goal (dropdown, required) with options: Generate Leads, Drive Sales, Get Sign-ups, Book Appointments, Download Resource, Other  
     - Current Conversion Rate (text, optional)  
     - Biggest Frustration (textarea, optional)  
     - Your Email (email, required)  
   - Add form description accordingly.

2. **Add Initialize Variables node:**  
   - Type: `set`  
   - Create variables with expressions referencing form fields:  
     - `landing_page_url` = `{{$json["Landing Page URL"]}}`  
     - `industry` = `{{$json["What's your industry?"] || 'General'}}`  
     - `primary_goal` = `{{$json["Primary goal of this page"]}}`  
     - `current_conversion` = `{{$json["Current conversion rate (if known)"] || 'Unknown'}}`  
     - `biggest_frustration` = `{{$json["Biggest frustration with current page"] || 'Not specified'}}`  
     - `user_email` = `{{$json["Your Email"]}}`  
     - `analysis_timestamp` = `{{ new Date().toISOString() }}`  

3. **Add Fetch Page HTML node:**  
   - Type: `httpRequest`  
   - URL: `={{$json.landing_page_url}}`  
   - Timeout: 10000 ms  
   - Set "Full Response" to true, "Never Error" to true  
   - Enable "Continue On Fail"  
   
4. **Add Extract Page Elements node:**  
   - Type: `code` (JavaScript)  
   - Use provided JS code to extract title, meta, headings, CTAs, forms, images, technology, etc.  
   - Input: Output of Fetch Page HTML node  

5. *(Optional)* Add alternative scraping nodes:  
   - Apify Scrape single URL node (configured with Apify credentials)  
   - Crawl4AI HTTP Request node with necessary authentication and POST JSON body for crawling  

6. **Add Design Critic Model node:**  
   - Type: `langchain.lmChatAnthropic`  
   - Model: "claude-sonnet-4-5-20250929"  
   - Temperature: 0.4, topP: 0.9  
   - Credential: Anthropic API  

7. **Add Agent 1: Design Critic node:**  
   - Type: `langchain.agent`  
   - Build prompt using extracted elements and initialized variables (see node for exact template)  
   - System message defines UX/UI expert role and output structure  

8. **Add Copywriter Model node:**  
   - Similar to Design Critic Model, with temperature 0.5  

9. **Add Agent 2: Copywriter node:**  
   - Type: `langchain.agent`  
   - Build prompt focusing on copywriting and messaging analysis  

10. **Add SEO Specialist Model node:**  
    - Type: `langchain.lmChatAnthropic`  
    - Temperature 0.3, topP 0.85  

11. **Add Agent 3: SEO & Technical node:**  
    - Type: `langchain.agent`  
    - Prompt for SEO and technical analysis  

12. **Add Growth Strategist Model node:**  
    - Temperature 0.6, topP 0.95  

13. **Add Agent 4: Growth Strategist node:**  
    - Type: `langchain.agent`  
    - Synthesizes all previous agents’ outputs and variables  
    - Outputs overall grade, scorecard, priorities, lift estimates, and recommendations  

14. **Connect nodes in sequence:**  
    - Form Submission → Initialize Variables → Fetch Page HTML → Extract Page Elements → Agent 1: Design Critic → Agent 2: Copywriter → Agent 3: SEO & Technical → Agent 4: Growth Strategist → Format Email Report → Send Email Report  
    - Attach respective LM model nodes to their agents' AI languageModel input.

15. **Add Format Email Report node:**  
    - Type: `code`  
    - Implement provided JavaScript to generate HTML email with embedded analyses, grades, stats, CTAs, and branding.  
    - Input: Outputs from all agents and Initialize Variables  

16. **Add Send Email Report node:**  
    - Type: `gmail`  
    - Configure OAuth2 credentials for Gmail (e.g., user "Sofia Wagner")  
    - Send to `user_email` with subject and HTML body from Format Email Report  

17. **Add Sticky Notes** for documentation inside n8n canvas:  
    - Workflow Overview  
    - Form Section  
    - Scraping Section  
    - Agent 1-4 Info  
    - Email Section  

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Performance: Average completion 60-90 seconds, ~ $0.15-0.25 cost per analysis                                   | Workflow Overview sticky note                                                                    |
| Conversion Funnel: Free audit report → Paid tiers for fixes, testing, full optimization                        | Workflow Overview sticky note                                                                    |
| Optimization Ideas: Add screenshot capture (UrlBox API), visual mockups, A/B testing, tracking implementation  | Workflow Overview sticky note                                                                    |
| Form Design Tips: Keep simple, use dropdowns, make email required only                                       | Form Section sticky note                                                                         |
| Scraping Enhancements: Add screenshot capture, CSS extraction, speed checks, broken link scanning              | Scraping Section sticky note                                                                     |
| Agent 1: Focus on UX/UI design, visual hierarchy, CTAs, mobile responsiveness                                 | Agent 1 Info sticky note                                                                         |
| Agent 2: Focus on messaging, headline effectiveness, emotional triggers, CTA copy                              | Agent 2 Info sticky note                                                                         |
| Agent 3: Focus on SEO fundamentals, meta tags, headings, image alt tags, mobile readiness                      | Agent 3 Info sticky note                                                                         |
| Agent 4: Synthesizes analyses, assigns grade, prioritizes improvements, estimates conversion lift             | Agent 4 Info sticky note                                                                         |
| Email Report: Stylish HTML with clear grade badge, analysis sections, page stats, CTAs, company brand links    | Email Section sticky note                                                                        |
| Website for services and contact: https://evervise.com                                                        | Footer in email report                                                                           |
| CTA Link in email: https://evervise.com/landing-page-optimization                                            | Included in email report CTA box                                                                 |
| The workflow respects content policies and handles only legal and public data                                 | Disclaimer                                                                                       |

---

This documentation enables advanced users and AI agents to fully understand, reproduce, and extend the Landing Page Analysis & Optimization workflow, anticipating potential issues and integration points.