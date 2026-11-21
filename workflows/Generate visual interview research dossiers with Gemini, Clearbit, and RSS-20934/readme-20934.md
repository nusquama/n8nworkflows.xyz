Generate visual interview research dossiers with Gemini, Clearbit, and RSS

https://n8nworkflows.xyz/workflows/generate-visual-interview-research-dossiers-with-gemini--clearbit--and-rss-20934


# Generate visual interview research dossiers with Gemini, Clearbit, and RSS

### 1. Workflow Overview

This workflow is designed to automate pre-interview research by collecting inputs for a target company and role, gathering live intelligence from multiple web sources, synthesizing the data via Google Gemini, and rendering a print-ready, downloadable HTML dossier complete with quantitative charts and architecture diagrams.

The logical execution is grouped into the following functional blocks:
- **1.1 Input Reception & Domain Resolution:** Captures user inputs via an n8n Form trigger, resolves the official company domain using Clearbit, and generates base visual asset URLs (logos and screenshots).
- **1.2 Parallel Data Scraping:** Scrapes the official website text (via Jina Reader), recent news headlines (via Google News RSS), and community discussions (via Hacker News Algolia API) simultaneously.
- **1.3 AI Synthesis & Report Generation:** Aggregates all scraped data into a unified context, feeds it to Google Gemini with a structured JSON output schema, and transforms the resulting JSON into a styled HTML document containing QuickChart radar charts and Mermaid diagrams.
- **1.4 Output Delivery:** Displays the final dossier in the browser via an n8n Form completion page (with auto-print functionality) and converts the document into a downloadable HTML file.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Domain Resolution
This block captures the initial candidate or interviewer requirements and establishes a clean corporate domain and associated visual assets.

- **Interview Research Form**
  - **Type & Technical Role:** `n8n-nodes-base.formTrigger` (Trigger)
  - **Configuration:** Configured with four fields: Company Name (required), Company Website URL (optional), Target Role (required), and Job Description (optional text area).
  - **Key Expressions:** None (static form configuration).
  - **Connections:** Output connects to *Auto-Discover Company URL*.
  - **Version Requirements:** Type Version 2.2.
  - **Edge Cases / Failure Types:** Incomplete required fields will prevent form submission.

- **Auto-Discover Company URL**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Action)
  - **Configuration:** Queries the Clearbit Autocomplete API (`https://autocomplete.clearbit.com/v1/companies/suggest?query=...`) with a 10-second timeout. `continueOnFail` is enabled.
  - **Key Expressions:** `={{ encodeURIComponent($json['Company Name']) }}`
  - **Connections:** Input from *Interview Research Form*; output connects to *Resolve URL and Visual Assets*.
  - **Version Requirements:** Type Version 4.2.
  - **Edge Cases / Failure Types:** Network timeouts or API unavailability handled via `continueOnFail`.

- **Resolve URL and Visual Assets**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Action)
  - **Configuration:** Executes custom JavaScript to parse Clearbit results, falling back to manual URLs or domain-name slugs if Clearbit returns no hits. Generates fallback logo URLs and WordPress MShot screenshot URLs.
  - **Key Expressions:** Reads form inputs and HTTP response arrays.
  - **Connections:** Input from *Auto-Discover Company URL*; outputs fan out to *Scrape Website (Jina Reader)*, *Fetch 6M News (Google RSS)*, and *Projects and Tech Pulse (HN API)*.
  - **Version Requirements:** Type Version 2.
  - **Edge Cases / Failure Types:** Malformed domain inputs or missing search results invoke fallback naming heuristics (`slug.com`).

---

#### 2.2 Parallel Data Scraping
This block triggers three parallel data collection pipelines to gather primary website content, recent press, and developer community sentiment.

- **Scrape Website (Jina Reader)**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Action)
  - **Configuration:** Fetches parsed markdown text of the target website via Jina Reader (`https://r.jina.ai/`) with a 15-second timeout. `continueOnFail` is enabled.
  - **Key Expressions:** `={{ $json.company_url }}`
  - **Connections:** Input from *Resolve URL and Visual Assets*; output connects to input index 0 of *Merge Research Streams*.
  - **Version Requirements:** Type Version 4.2.
  - **Edge Cases / Failure Types:** Unreachable websites or blocking firewalls return errors caught by `continueOnFail`.

- **Fetch 6M News (Google RSS)**
  - **Type & Technical Role:** `n8n-nodes-base.rssFeedRead` (Action)
  - **Configuration:** Reads Google News RSS search results for the company over a 6-month window. `continueOnFail` is enabled.
  - **Key Expressions:** `={{ encodeURIComponent($json.company_name + ' when:6m') }}`
  - **Connections:** Input from *Resolve URL and Visual Assets*; output connects to input index 1 of *Merge Research Streams*.
  - **Version Requirements:** Type Version 1.1.
  - **Edge Cases / Failure Types:** Rate limiting or empty RSS payloads handled by `continueOnFail`.

- **Projects and Tech Pulse (HN API)**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (Action)
  - **Configuration:** Queries the Hacker News Algolia API for stories matching the company name (`https://hn.algolia.com/api/v1/search?query=...&tags=story&hitsPerPage=6`). `continueOnFail` is enabled.
  - **Key Expressions:** `={{ encodeURIComponent($json.company_name) }}`
  - **Connections:** Input from *Resolve URL and Visual Assets*; output connects to input index 2 of *Merge Research Streams*.
  - **Version Requirements:** Type Version 4.2.
  - **Edge Cases / Failure Types:** API rate limits or lack of search hits handled via `continueOnFail`.

- **Merge Research Streams**
  - **Type & Technical Role:** `n8n-nodes-base.merge` (Action)
  - **Configuration:** Combines the three parallel scraping inputs using numberInputs mode (3 inputs).
  - **Key Expressions:** None.
  - **Connections:** Inputs from *Scrape Website*, *Fetch 6M News*, and *Projects and Tech Pulse*; output connects to *Build Context with Source URLs*.
  - **Version Requirements:** Type Version 3.
  - **Edge Cases / Failure Types:** Incomplete branch failures are bypassed due to upstream `continueOnFail` settings.

- **Build Context with Source URLs**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Action)
  - **Configuration:** JavaScript node that normalizes and formats website text, news snippets (with citation indices), and Hacker News threads into a unified aggregated research payload.
  - **Key Expressions:** Reads data from preceding nodes via n8n data accessor methods (`$()`).
  - **Connections:** Input from *Merge Research Streams*; output connects to *Synthesize Dossier (LLM Chain)*.
  - **Version Requirements:** Type Version 2.
  - **Edge Cases / Failure Types:** Missing sub-properties are safely wrapped in try-catch blocks to prevent execution halts.

---

#### 2.3 AI Synthesis & Report Generation
This block leverages Google Gemini to process the unstructured research context into structured intelligence, followed by code-based rendering of the final HTML document.

- **Synthesize Dossier (LLM Chain)**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.chainLlm` (Action)
  - **Configuration:** Instructs the LLM to act as an executive interview strategist, consuming scraped intelligence and outputting a rigid JSON schema containing scores, metrics, Mermaid diagram syntax, and citation logs.
  - **KeyExpressions:** Interpolates company name, resolved URL, target role, job description, and aggregated research markdown.
  - **Connections:** Input from *Build Context with Source URLs*; connected to *Google Gemini Chat Model* (AI language model); output connects to *Build Visual PDF Dossier*.
  - **Version Requirements:** Type Version 1.4.
  - **Edge Cases / Failure Types:** LLM output drift or invalid JSON formatting handled gracefully downstream.

- **Google Gemini Chat Model**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (AI Model)
  - **Configuration:** Uses model `models/gemini-3.1-flash-lite` with a low temperature (`0.1`) to ensure deterministic structure compliance.
  - **Credentials Required:** Google Gemini (PaLM) API account (`googlePalmApi`).
  - **Connections:** Connected to *Synthesize Dossier (LLM Chain)* via the `ai_languageModel` input.
  - **Version Requirements:** Type Version 1.

- **Build Visual PDF Dossier**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Action)
  - **Configuration:** Parses the LLM's JSON output, builds QuickChart radar chart configurations, encodes Mermaid diagram definitions into Base64 for rendering via Mermaid.ink, and constructs a fully responsive, print-optimized HTML report.
  - **Key Expressions:** Reads raw text from LLM output and base properties from resolution nodes.
  - **Connections:** Input from *Synthesize Dossier (LLM Chain)*; outputs fan out to *Display and Auto-Print PDF in Browser* and *Create Downloadable Dossier File*.
  - **Version Requirements:** Type Version 2.
  - **Edge Cases / Failure Types:** Malformed LLM JSON triggers a robust fallback structural payload.

---

#### 2.4 Output Delivery
This block presents the compiled report to the user in the browser and creates a downloadable file asset.

- **Display and Auto-Print PDF in Browser**
  - **Type & Technical Role:** `n8n-nodes-base.form` (Action)
  - **Configuration:** Operation set to `completion`. Displays a completion page displaying the generated HTML document which auto-triggers the browser print dialog. `continueOnFail` is enabled.
  - **Key Expressions:** `={{ $json.company_name }}` and `={{ $json.printable_html_document }}`.
  - **Connections:** Input from *Build Visual PDF Dossier*.
  - **Version Requirements:** Type Version 1.
  - **Edge Cases / Failure Types:** Browser popup blockers or restrictive iframe policies might suppress automated print execution.

- **Create Downloadable Dossier File**
  - **Type & Technical Role:** `n8n-nodes-base.convertToFile` (Action)
  - **Configuration:** Converts the HTML text property (`printable_html_document`) into a downloadable file format with a dynamically generated file name.
  - **Key Expressions:** `={{ $json.file_name }}.html`.
  - **Connections:** Input from *Build Visual PDF Dossier*.
  - **Version Requirements:** Type Version 1.1.
  - **Edge Cases / Failure Types:** Binary data handling limitations in environment memory (mitigated by n8n binary configuration settings).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note - Setup | `n8n-nodes-base.stickyNote` | Documentation & Credential guidelines | None | None | ## Setup Instructions (Visual and Citation Edition)<br>1. Single API Key: Connect your Google Gemini Chat Model credential.<br>2. Zero-Cost Visual Engines Included:<br>   - Logos and Website Snapshots: Clearbit + WordPress MShots<br>   - Architecture Diagrams: Mermaid.ink SVG Renderer<br>   - Graphical Radar Charts: QuickChart.io Engine<br>   - Proofs and Citations Table: Clickable URLs from Google News RSS and Hacker News |
| Sticky Note - Discovery | `n8n-nodes-base.stickyNote` | Documentation for discovery and scraping | None | None | ## 1. Auto Domain Discovery and Parallel Scraping<br>Resolves the official domain and logo, then scrapes the official site, Google News RSS (with URLs), and Hacker News threads simultaneously. |
| Sticky Note - Gemini and PDF | `n8n-nodes-base.stickyNote` | Documentation for AI synthesis and PDF generation | None | None | ## 2. Gemini Visual Synthesis and Rich PDF Generator<br>Generates quantitative scores, a Mermaid ecosystem diagram, verified source links, and compiles a print-ready visual PDF dossier. |
| Interview Research Form | `n8n-nodes-base.formTrigger` | Collects company and role details | None | Auto-Discover Company URL | |
| Auto-Discover Company URL | `n8n-nodes-base.httpRequest` | Discovers domain via Clearbit | Interview Research Form | Resolve URL and Visual Assets | |
| Resolve URL and Visual Assets | `n8n-nodes-base.code` | Normalizes URLs and builds asset links | Auto-Discover Company URL | Scrape Website, Fetch 6M News, Projects and Tech Pulse | |
| Scrape Website (Jina Reader) | `n8n-nodes-base.httpRequest` | Scrapes official website text | Resolve URL and Visual Assets | Merge Research Streams | |
| Fetch 6M News (Google RSS) | `n8n-nodes-base.rssFeedRead` | Fetches recent news via RSS | Resolve URL and Visual Assets | Merge Research Streams | |
| Projects and Tech Pulse (HN API) | `n8n-nodes-base.httpRequest` | Fetches tech discussions via HN API | Resolve URL and Visual Assets | Merge Research Streams | |
| Merge Research Streams | `n8n-nodes-base.merge` | Merges three scraping branches | Scrape Website, Fetch 6M News, Projects and Tech Pulse | Build Context with Source URLs | |
| Build Context with Source URLs | `n8n-nodes-base.code` | Constructs aggregated research payload | Merge Research Streams | Synthesize Dossier (LLM Chain) | |
| Synthesize Dossier (LLM Chain) | `@n8n/n8n-nodes-langchain.chainLlm` | Orchestrates LLM JSON structured generation | Build Context with Source URLs, Google Gemini Chat Model | Build Visual PDF Dossier | |
| Google Gemini Chat Model | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides underlying LLM capabilities | None | Synthesize Dossier (LLM Chain) | |
| Build Visual PDF Dossier | `n8n-nodes-base.code` | Builds HTML report, charts, and diagrams | Synthesize Dossier (LLM Chain) | Display and Auto-Print PDF in Browser, Create Downloadable Dossier File | |
| Display and Auto-Print PDF in Browser | `n8n-nodes-base.form` | Displays form completion page with print trigger | Build Visual PDF Dossier | None | |
| Create Downloadable Dossier File | `n8n-nodes-base.convertToFile` | Converts HTML string into downloadable file | Build Visual PDF Dossier | None | |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these step-by-step instructions:

1. **Create the Trigger Node:**
   - Add a **Form Trigger** node (`n8n-nodes-base.formTrigger`).
   - Configure the form title as `AI Visual Company Research and Interview Dossier`.
   - Add form fields:
     - `Company Name` (Type: Text, Required: true, Placeholder: `e.g. Stripe, Anthropic, Vercel`)
     - `Company Website URL (Optional)` (Type: Text, Required: false, Placeholder: `Leave blank to auto-discover, or enter https://...`)
     - `Target Role` (Type: Text, Required: true, Placeholder: `e.g. Senior AI Engineer`)
     - `Job Description (Optional)` (Type: Textarea, Required: false, Placeholder: `Paste the JD here, or leave blank for general prep...`)

2. **Add Domain Discovery:**
   - Add an **HTTP Request** node named `Auto-Discover Company URL`.
   - Set method to `GET`, URL to `https://autocomplete.clearbit.com/v1/companies/suggest?query={{ encodeURIComponent($json['Company Name']) }}`.
   - Set timeout to `10000` ms and enable `Continue On Fail`.
   - Connect *Interview Research Form* to this node.

3. **Add URL & Asset Resolution Logic:**
   - Add a **Code** node named `Resolve URL and Visual Assets` (JavaScript mode).
   - Paste logic to extract form inputs, check Clearbit response hits for valid domains, construct fallback domain slugs (`.com`), and generate WordPress MShots screenshot URLs (`https://s.wordpress.com/mshots/v1/...`).
   - Connect *Auto-Discover Company URL* to this node.

4. **Set Up Parallel Scraping Branches:**
   - **Branch A:** Add an **HTTP Request** node named `Scrape Website (Jina Reader)`. Set URL to `https://r.jina.ai/{{ $json.company_url }}`, timeout `15000` ms, enable `Continue On Fail`.
   - **Branch B:** Add a **RSS Feed Read** node named `Fetch 6M News (Google RSS)`. Set URL to `https://news.google.com/rss/search?q={{ encodeURIComponent($json.company_name + ' when:6m') }}&hl=en-US&gl=US&ceid=US:en`, enable `Continue On Fail`.
   - **Branch C:** Add an **HTTP Request** node named `Projects and Tech Pulse (HN API)`. Set URL to `https://hn.algolia.com/api/v1/search?query={{ encodeURIComponent($json.company_name) }}&tags=story&hitsPerPage=6`, enable `Continue On Fail`.
   - Connect *Resolve URL and Visual Assets* to all three nodes simultaneously.

5. **Merge and Aggregate Research Data:**
   - Add a **Merge** node named `Merge Research Streams` configured for `3` number inputs. Connect outputs from the three scraping nodes into inputs `0`, `1`, and `2` respectively.
   - Add a **Code** node named `Build Context with Source URLs`. Write JavaScript to format website text snippets, news titles with source URLs, and Hacker News discussion metrics into a single aggregated research object. Connect *Merge Research Streams* to this node.

6. **Configure AI Synthesis:**
   - Add an **Advanced AI -> Chain: LLM** node named `Synthesize Dossier (LLM Chain)`.
   - Configure prompt to instruct the LLM to output *only* a valid JSON object matching the requested schema (`about_the_company`, `quantitative_ratings`, `mermaid_ecosystem_diagram`, `verified_evidence_log`, etc.).
   - Add a **Google Gemini Chat Model** sub-node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`). Set model name to `models/gemini-3.1-flash-lite`, temperature to `0.1`, and connect valid Google Gemini (PaLM) API credentials.
   - Connect the AI model node to the `ai_languageModel` input of `Synthesize Dossier (LLM Chain)`. Connect *Build Context with Source URLs* to its main input.

7. **Build Visual PDF Report:**
   - Add a **Code** node named `Build Visual PDF Dossier` (JavaScript mode).
   - Write code to parse the JSON response from the LLM, construct QuickChart radar chart JSON configurations, encode Mermaid diagram definitions in Base64 for Mermaid.ink, and assemble a complete, styled HTML document string (`printable_html_document`) including inline CSS and an auto-print JavaScript hook (`window.print()`).
   - Connect *Synthesize Dossier (LLM Chain)* to this node.

8. **Configure Output Delivery:**
   - Add a **Form** node (`n8n-nodes-base.form`) named `Display and Auto-Print PDF in Browser`. Set operation to `completion`, completion title to `Visual Dossier Ready: {{ $json.company_name }}`, and completion message to `={{ $json.printable_html_document }}`. Enable `Continue On Fail`.
   - Add a **Convert to File** node named `Create Downloadable Dossier File`. Set operation to `toText`, source property to `printable_html_document`, and file name expression to `={{ $json.file_name }}.html`.
   - Connect *Build Visual PDF Dossier* output to both output delivery nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Logos and Website Snapshots | Clearbit (Autocomplete & Logo API) and WordPress MShots (`https://s.wordpress.com/mshots/v1/`) |
| Architecture Diagrams | Mermaid.ink SVG Renderer (`https://mermaid.ink/svg/`) |
| Graphical Radar Charts | QuickChart.io Engine (`https://quickchart.io/chart`) |
| Official Website Scraping | Jina Reader (`https://r.jina.ai/`) |
| Tech Community Signals | Hacker News Algolia API (`https://hn.algolia.com/api/v1/search`) |