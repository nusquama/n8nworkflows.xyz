Write SEO-ready research-backed articles with Gemini, Wikipedia and Gmail

https://n8nworkflows.xyz/workflows/write-seo-ready-research-backed-articles-with-gemini--wikipedia-and-gmail-17692


# Write SEO-ready research-backed articles with Gemini, Wikipedia and Gmail

### 1. Workflow Overview

This workflow automates the creation of research-backed, SEO-optimized articles by orchestrating a multi-agent AI pipeline. It collects user parameters via an interactive form, gathers and verifies facts using an AI research agent and Wikipedia, drafts and edits the content using Google Gemini models, logs the final structured output to Google Sheets, and delivers the finished article directly to the user via Gmail.

The workflow logic is categorized into the following functional blocks:
- **1.1 Input Reception:** Captures user requests via an n8n form trigger.
- **1.2 AI Research Phase:** Uses an autonomous AI agent backed by a Gemini language model and Wikipedia to gather and structure verified facts.
- **1.3 AI Writing & Editing Phase:** Sequential LLM chains draft the article strictly from the research notes, then optimize it for SEO.
- **1.4 Data Transformation & Delivery:** Extracts individual metadata elements (title, meta description, body, recipient email) using custom JavaScript, logs the entry into Google Sheets, and sends the final email.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** This block acts as the entry point of the workflow, presenting a structured form to the user to capture necessary parameters for article generation.
- **Nodes Involved:** 
  - `Article Brief Form`

- **Node Details:**
  - **Article Brief Form**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (v2.6) – Webhook-based form trigger that halts workflow execution until a user submits data.
    - *Configuration Choices:* Configured with custom fields (`Your email`, `Topic`, `Audience`, `Keyword`). Sets a custom submission response text confirming that the article is being processed.
    - *Key Expressions or Variables:* None (generates the initial data payload).
    - *Input and Output Connections:* Input: None (Workflow entry point). Output: `Research Agent`.
    - *Version-specific Requirements:* Requires n8n form trigger handling.
    - *Edge Cases / Potential Failure Types:* Incomplete submissions or invalid email formats handled by built-in form validation.

---

#### 2.2 AI Research Phase
- **Overview:** Evaluates the user's requested topic against live Wikipedia data to compile concise, verified research notes containing key points, metrics, and common misconceptions.
- **Nodes Involved:** 
  - `Research Agent`
  - `Research Model`
  - `Wikipedia`

- **Node Details:**
  - **Research Agent**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (v3.1) – LangChain autonomous agent that orchestrates LLM reasoning with external tools.
    - *Configuration Choices:* Utilizes a defined prompt utilizing `{{ $json.Topic }}` and `{{ $json.Audience }}` from the form trigger. Includes a strict system message instructing the agent to verify facts with Wikipedia.
    - *Key Expressions or Variables:* `=Research the topic "{{ $json.Topic }}" for an audience of {{ $json.Audience }}. Use the Wikipedia tool to gather accurate facts...`
    - *Input and Output Connections:* Input: `Article Brief Form`. Output: `Write Draft with Gemini`. AI Model Input: `Research Model`. AI Tool Input: `Wikipedia`.
    - *Edge Cases / Potential Failure Types:* Hallucination risks mitigated by enforced Wikipedia lookups; API timeout or rate-limit errors from Gemini or Wikipedia.

  - **Research Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (v1.1) – Language model provider for the research agent.
    - *Configuration Choices:* Model set to `models/gemini-3.1-flash-lite` with a lower temperature (`0.2`) for factual accuracy.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Output (AI Language Model): Connected to `Research Agent`.
    - *Credentials Required:* `Google Gemini(PaLM) Api account`.

  - **Wikipedia**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.toolWikipedia` (v1) – LangChain tool allowing the agent to query Wikipedia articles.
    - *Configuration Choices:* Default parameters.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Output (AI Tool): Connected to `Research Agent`.

---

#### 2.3 AI Writing & Editing Phase
- **Overview:** Takes the structured research notes and processes them sequentially through two specialized LLM chains: first, drafting a ~500-word article without introducing outside facts, and second, optimizing the draft for SEO and incorporating target keywords.
- **Nodes Involved:** 
  - `Write Draft with Gemini`
  - `Writer Model`
  - `Edit and Optimise with Gemini`
  - `Editor Model`

- **Node Details:**
  - **Write Draft with Gemini**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (v1.9) – Basic LLM chain handling the primary drafting phase.
    - *Configuration Choices:* Instructs the model to write strictly using the research notes provided by the previous agent step.
    - *Key Expressions or Variables:* `={{ $('Article Brief Form').item.json.Audience }}`, `={{ $('Article Brief Form').item.json.Topic }}`, `={{ $json.output }}`.
    - *Input and Output Connections:* Input: `Research Agent`. Output: `Edit and Optimise with Gemini`. AI Model Input: `Writer Model`.

  - **Writer Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (v1.1) – Language model provider for the writer chain.
    - *Configuration Choices:* Model set to `models/gemini-3.1-flash-lite` with a higher temperature (`0.7`) to encourage fluent, natural writing style.
    - *Credentials Required:* `Google Gemini(PaLM) Api account`.

  - **Edit and Optimise with Gemini**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (v1.9) – Basic LLM chain acting as an SEO editor.
    - *Configuration Choices:* Refines phrasing, injects the user-provided SEO keyword naturally, and formats explicit output lines for the title (`TITLE:`) and meta description (`META:`).
    - *Key Expressions or Variables:* `={{ $('Article Brief Form').item.json.Keyword }}`, `={{ $json.text }}`.
    - *Input and Output Connections:* Input: `Write Draft with Gemini`. Output: `Split Article, Title and Meta`. AI Model Input: `Editor Model`.

  - **Editor Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (v1.1) – Language model provider for the editor chain.
    - *Configuration Choices:* Model set to `models/gemini-3.1-flash-lite` with a moderate temperature (`0.4`).
    - *Credentials Required:* `Google Gemini(PaLM) Api account`.

---

#### 2.4 Data Transformation & Delivery
- **Overview:** Parses the editor output to segregate structured article components, saves the finalized entry into a Google Sheets tracking log, and emails the complete text to the requester.
- **Nodes Involved:** 
  - `Split Article, Title and Meta`
  - `Log Article`
  - `Email the Article`

- **Node Details:**
  - **Split Article, Title and Meta**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Executes custom JavaScript to parse regular expressions from the AI editor's output text.
    - *Configuration Choices:* Runs once per item. Uses RegEx to extract lines prefixed with `TITLE:` and `META:`, strips them from the main body, and aggregates form parameters and processed text into a unified return object (`email`, `topic`, `title`, `meta`, `article`).
    - *Key Expressions or Variables:* `String($json.text || '')`, `$('Article Brief Form').item.json`.
    - *Input and Output Connections:* Input: `Edit and Optimise with Gemini`. Output: `Log Article`.
    - *Edge Cases / Potential Failure Types:* If the LLM fails to output `TITLE:` or `META:` explicitly, matching variables default to empty strings.

  - **Log Article**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.7) – Appends structured data rows to a connected Google Spreadsheet.
    - *Configuration Choices:* Operation set to `append`. Uses list mode to target a specific document and sheet name (cached as "Articles").
    - *Credentials Required:* `Google Sheets OAuth2 Api`.
    - *Edge Cases / Potential Failure Types:* Spreadsheet schema mismatches or permission errors on the connected Google account.

  - **Email the Article**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (v2.2) – Sends the finished article via the Gmail API.
    - *Configuration Choices:* Plain text email delivery format configured using expressions mapping directly to the parsed code node output.
    - *Key Expressions or Variables:* 
      - Send To: `={{ $("Split Article, Title and Meta").item.json.email }}`
      - Subject: `={{ "Your article: " + $("Split Article, Title and Meta").item.json.title }}`
      - Message: `={{ "TITLE: " + $("Split Article, Title and Meta").item.json.title + "\nMETA: " + $("Split Article, Title and Meta").item.json.meta + "\n\n" + $("Split Article, Title and Meta").item.json.article }}`
    - *Input and Output Connections:* Input: `Log Article`. Output: None (Terminal node).
    - *Credentials Required:* `Gmail OAuth2`.
    - *Edge Cases / Potential Failure Types:* Invalid recipient email addresses or rate limiting enforced by Google Workspace SMTP/API limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Article Brief Form** | `n8n-nodes-base.formTrigger` | Captures user inputs (email, topic, audience, keyword) via an interactive form. | None | Research Agent | ## Turn a topic into a research-backed article with a multi-agent AI pipeline<br><br>### How it works<br>Instead of one prompt doing everything, this template splits the job across three specialised AI steps, like a real editorial team. Submit a topic and audience in the built-in form. First an AI Agent researches the topic, using the Wikipedia tool to gather and verify facts, and produces structured notes. Then a Basic LLM Chain (the writer) turns only those notes into a friendly 500-word draft, so it cannot invent facts. Finally a second Basic LLM Chain (the SEO editor) tightens the draft, works in your keyword, and adds an SEO title and meta description. The finished piece is logged to Google Sheets and emailed to you.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Gmail.<br>2. Point the Log Article node at your spreadsheet (tab: Articles).<br>3. Open the form URL and submit a topic.<br><br>### Customization tips<br>Add a fact-check agent, or publish straight to WordPress or a Google Doc. |
| **Research Agent** | `@n8n/n8n-nodes-langchain.agent` | Orchestrates factual research using Gemini and Wikipedia. | Article Brief Form | Write Draft with Gemini | ## 1. Research Agent<br>An AI Agent gathers and verifies facts with the Wikipedia tool and writes structured notes. |
| **Research Model** | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides the LLM backend for the Research Agent. | None | Research Agent | ## 1. Research Agent<br>An AI Agent gathers and verifies facts with the Wikipedia tool and writes structured notes. |
| **Wikipedia** | `@n8n/n8n-nodes-langchain.toolWikipedia` | Provides Wikipedia search capabilities as a tool to the Research Agent. | None | Research Agent | ## 1. Research Agent<br>An AI Agent gathers and verifies facts with the Wikipedia tool and writes structured notes. |
| **Write Draft with Gemini** | `@n8n/n8n-nodes-langchain.chainLlm` | Drafts a ~500-word article strictly based on the research notes. | Research Agent | Edit and Optimise with Gemini | ## 2. Writer & SEO editor (Basic LLM Chains)<br>One chain writes the draft from the notes only; a second chain edits it and adds an SEO title and meta. |
| **Writer Model** | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides the LLM backend for the draft writing chain. | None | Write Draft with Gemini | ## 2. Writer & SEO editor (Basic LLM Chains)<br>One chain writes the draft from the notes only; a second chain edits it and adds an SEO title and meta. |
| **Edit and Optimise with Gemini** | `@n8n/n8n-nodes-langchain.chainLlm` | Optimizes the draft for SEO, inserts keywords, and structures titles/meta descriptions. | Write Draft with Gemini | Split Article, Title and Meta | ## 2. Writer & SEO editor (Basic LLM Chains)<br>One chain writes the draft from the notes only; a second chain edits it and adds an SEO title and meta. |
| **Editor Model** | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides the LLM backend for the SEO editor chain. | None | Edit and Optimise with Gemini | ## 2. Writer & SEO editor (Basic LLM Chains)<br>One chain writes the draft from the notes only; a second chain edits it and adds an SEO title and meta. |
| **Split Article, Title and Meta** | `n8n-nodes-base.code` | Parses the editor output to separate title, meta description, body, and email variables. | Edit and Optimise with Gemini | Log Article | ## 3. Log & deliver<br>Split out the title and meta, log the article to Sheets and email the finished piece. |
| **Log Article** | `n8n-nodes-base.googleSheets` | Appends finalized article details to a Google Sheets document. | Split Article, Title and Meta | Email the Article | ## 3. Log & deliver<br>Split out the title and meta, log the article to Sheets and email the finished piece. |
| **Email the Article** | `n8n-nodes-base.gmail` | Sends the compiled article and metadata via Gmail to the requester. | Log Article | None | ## 3. Log & deliver<br>Split out the title and meta, log the article to Sheets and email the finished piece. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Form Trigger Node:**
   - Add an **On form submission** node (`n8n-nodes-base.formTrigger`).
   - Set form title to `Topic to article` and button label to `Write my article`.
   - Add four form fields:
     1. Type: `Email`, Label: `Your email`, Required: `true`.
     2. Type: `Text`, Label: `Topic`, Placeholder: `e.g. Benefits of intermittent fasting`, Required: `true`.
     3. Type: `Text`, Label: `Audience`, Placeholder: `e.g. Busy beginners`, Required: `true`.
     4. Type: `Text`, Label: `Keyword`, Placeholder: `e.g. intermittent fasting for beginners`, Required: `false`.
   - Set success response text to confirm email delivery.

2. **Create the Research Agent Block:**
   - Add an **AI Agent** node (`@n8n/n8n-nodes-langchain.agent`). Connect its main input to the **Article Brief Form** node.
   - Configure prompt type to *Define* and set the text expression to research the topic using `{{ $json.Topic }}` and `{{ $json.Audience }}`. Add a system message instructing the agent to verify facts with Wikipedia.
   - Add a **Google Gemini Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`), set model to `models/gemini-3.1-flash-lite`, temperature to `0.2`, and link it to the agent's AI Language Model input. Configure your Google Gemini API credentials.
   - Add a **Wikipedia** tool node (`@n8n/n8n-nodes-langchain.toolWikipedia`) and link it to the agent's AI Tool input.

3. **Create the Writer Chain Block:**
   - Add a **Basic LLM Chain** node (`@n8n/n8n-nodes-langchain.chainLlm`). Connect its main input to the output of the **Research Agent**.
   - Configure prompt type to *Define* and set the instruction text to write a ~500-word article using only the research notes (`{{ $json.output }}`), targeted to `{{ $('Article Brief Form').item.json.Audience }}` on `{{ $('Article Brief Form').item.json.Topic }}`.
   - Add a second **Google Gemini Chat Model** node, set model to `models/gemini-3.1-flash-lite`, temperature to `0.7`, and connect it to the writer chain's AI Language Model input using the same Gemini credentials.

4. **Create the Editor Chain Block:**
   - Add another **Basic LLM Chain** node. Connect its main input to the output of the **Write Draft with Gemini** node.
   - Configure prompt type to *Define* and set instruction text instructing the model to act as an SEO editor, incorporating keyword `{{ $('Article Brief Form').item.json.Keyword }}` and formatting lines starting with `TITLE:` and `META:`.
   - Add a third **Google Gemini Chat Model** node, set model to `models/gemini-3.1-flash-lite`, temperature to `0.4`, and connect it to the editor chain's AI Language Model input.

5. **Create the Code Parsing Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Connect its main input to **Edit and Optimise with Gemini**.
   - Set mode to *Run Once for Each Item*.
   - Insert JavaScript code to extract regex matches for `TITLE:` and `META:`, strip them from the body, and map properties (`email`, `topic`, `title`, `meta`, `article`) back into an output object referencing `$('Article Brief Form').item.json`.

6. **Create the Google Sheets Logging Node:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`). Connect its main input to the **Code** node.
   - Configure operation to *Append*. Select your target document ID and sheet name (e.g., `Articles`) using list selection mode. Configure Google Sheets OAuth2 credentials.

7. **Create the Gmail Delivery Node:**
   - Add a **Gmail** node (`n8n-nodes-base.gmail`). Connect its main input to the **Log Article** node.
   - Configure parameters:
     - *Send To:* `={{ $("Split Article, Title and Meta").item.json.email }}`
     - *Subject:* `={{ "Your article: " + $("Split Article, Title and Meta").item.json.title }}`
     - *Message:* `={{ "TITLE: " + $("Split Article, Title and Meta").item.json.title + "\nMETA: " + $("Split Article, Title and Meta").item.json.meta + "\n\n" + $("Split Article, Title and Meta").item.json.article }}`
     - *Email Type:* `Text`
   - Configure Gmail OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Multi-agent editorial pipeline architecture separating research, drafting, and SEO optimization. | Template design concept mirroring editorial teams. |