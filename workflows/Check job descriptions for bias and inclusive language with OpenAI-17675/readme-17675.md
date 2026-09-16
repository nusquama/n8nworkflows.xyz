Check job descriptions for bias and inclusive language with OpenAI

https://n8nworkflows.xyz/workflows/check-job-descriptions-for-bias-and-inclusive-language-with-openai-17675


# Check job descriptions for bias and inclusive language with OpenAI

### 1. Workflow Overview

The "Job Description Bias and Inclusive Language Checker" workflow is designed to audit job postings for exclusionary wording, gender-coding bias, and potential barriers before publication. It serves recruiters, hiring managers, and HR teams by providing an immediate, objective second opinion on inclusivity.

The logical flow is organized into two primary operational blocks:
- **1.1 Input Reception, Validation, and Preparation:** Collects user submissions via an n8n form, normalizes fields, validates text length against a minimum threshold to avoid wasted processing, and truncates overly long inputs to respect model context constraints.
- **1.2 AI Analysis and Report Generation:** Leverages an OpenAI chat model configured with DEI-aware prompts and structured output parsing to evaluate the text, builds a styled HTML response containing inclusivity scores and flagged phrases, and presents the final report to the user directly within the form completion interface.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception, Validation, and Preparation
This block handles user intake, sanitizes raw form variables, enforces character count limitations, and routes submissions either to processing or error handling.

- **Nodes Involved:** 
  - `Form Trigger`
  - `Map Form Fields`
  - `Long Enough?`
  - `Too Short - Error Response`
  - `Prepare Input`

- **Node Details:**
  - **Form Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (v2.2) – Acts as the entry point, rendering an interactive web form for users.
    - *Configuration Choices:* Configured with three fields: "Job Description Text" (textarea, required), "Role Title", and "Industry". Attribution text is disabled.
    - *Key Expressions or Variables:* None (triggers on form submission payload).
    - *Input and Output Connections:* Input: None (Webhook/Trigger). Output: Connects to `Map Form Fields`.
    - *Version-specific Requirements:* Requires v2.2+ for modern form UI handling.
    - *Edge Cases / Potential Failure Types:* Webhook registration failures or unreachable production URLs if the workflow is inactive.

  - **Map Form Fields**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Normalizes and cleans form payload keys into predictable camelCase properties.
    - *Configuration Choices:* Uses custom JavaScript to extract, trim, and set default values for `jdText`, `roleTitle`, and `industry`.
    - *Key Expressions or Variables:* `{{ $input.first().json }}`
    - *Input and Output Connections:* Input: `Form Trigger`. Output: Connects to `Long Enough?`.
    - *Edge Cases / Potential Failure Types:* JavaScript runtime errors if expected keys are missing from the upstream payload.

  - **Long Enough?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2.2) – Evaluates text length to branch execution based on validation rules.
    - *Configuration Choices:* Checks if `{{ $json.jdText.length }}` is greater than or equal to `150` characters.
    - *Key Expressions or Variables:* `{{ $json.jdText.length }}`
    - *Input and Output Connections:* Input: `Map Form Fields`. Output: True branch connects to `Prepare Input`; False branch connects to `Too Short - Error Response`.

  - **Too Short - Error Response**
    - *Type and Technical Role:* `n8n-nodes-base.form` (v1) – Returns an HTML completion response directly to the user.
    - *Configuration Choices:* Operation set to `completion` with `respondWith` set to `showText`, rendering a styled warning message prompting the user to submit at least 150 characters.
    - *Input and Output Connections:* Input: `Long Enough?` (False branch). Output: None (Terminal node).

  - **Prepare Input**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Prepares and constrains input payloads before model consumption.
    - *Configuration Choices:* Checks if text exceeds 12,000 characters. If so, slices the string to the 12,000-character limit and flags `truncated: true`.
    - *Key Expressions or Variables:* `{{ $input.first().json }}`
    - *Input and Output Connections:* Input: `Long Enough?` (True branch). Output: Connects to `Analyze Bias`.

---

#### 2.2 AI Analysis and Report Generation
This block configures the language model connection, enforces structured output schemas, executes the DEI evaluation prompt, compiles the results into an HTML report, and displays the final output to the user.

- **Nodes Involved:**
  - `OpenAI Chat Model`
  - `Bias Report Schema`
  - `Auto-fixing Output Parser`
  - `Analyze Bias`
  - `Build HTML Report`
  - `Send Report`

- **Node Details:**
  - **OpenAI Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.2) – LangChain integration node that provides the underlying OpenAI model.
    - *Configuration Choices:* Uses model `gpt-4.1-mini` with a low temperature of `0.2` for deterministic, focused analytical responses.
    - *Input and Output Connections:* Output: Connected via AI sub-nodes to `Analyze Bias` and `Auto-fixing Output Parser`.
    - *Edge Cases / Potential Failure Types:* Invalid API credentials, rate limits, or API outages from OpenAI.

  - **Bias Report Schema**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3) – Defines the JSON schema expected from the language model.
    - *Configuration Choices:* Provides an explicit JSON schema example detailing `inclusivity_score`, `overall_summary`, `gender_coding` details, `flags` array (with phrase, category, severity, explanation, and suggested rewrite), and `readability_note`.
    - *Input and Output Connections:* Output: Connects to `Auto-fixing Output Parser`.

  - **Auto-fixing Output Parser**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserAutofixing` (v1) – Automatically attempts to correct malformed JSON outputs returned by the language model.
    - *Input and Output Connections:* Input: Connected to `OpenAI Chat Model` and `Bias Report Schema`. Output: Connects to `Analyze Bias`.

  - **Analyze Bias**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (v1.7) – Executes the core language model chain using a DEI-aware prompt.
    - *Configuration Choices:* Uses a detailed system prompt referencing established gender-decoder research (Gaucher, Friesen & Kay). Evaluates input role title, industry, and job description text. Integrates output parsing.
    - *Key Expressions or Variables:* `{{ $json.roleTitle }}`, `{{ $json.industry }}`, `{{ $json.jdText }}`
    - *Input and Output Connections:* Input: `Prepare Input`, `OpenAI Chat Model`, and `Auto-fixing Output Parser`. Output: Connects to `Build HTML Report`.
    - *Edge Cases / Potential Failure Types:* Model hallucinations failing output schema validation despite auto-fixing.

  - **Build HTML Report**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) – Transforms the structured JSON AI response into a fully styled, responsive HTML document.
    - *Configuration Choices:* JavaScript code dynamically calculates color codes based on severity levels and inclusivity scores, builds visual category chips for gender-coded words, and constructs HTML markup blocks for all flagged phrases.
    - *Key Expressions or Variables:* `{{ $input.first().json.output }}`
    - *Input and Output Connections:* Input: `Analyze Bias`. Output: Connects to `Send Report`.

  - **Send Report**
    - *Type and Technical Role:* `n8n-nodes-base.form` (v1) – Renders the completed analysis report on the form completion page.
    - *Configuration Choices:* Operation set to `completion` with `respondWith` set to `showText`.
    - *Key Expressions or Variables:* `{{ $json.html }}`
    - *Input and Output Connections:* Input: `Build HTML Report`. Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Form Trigger | `n8n-nodes-base.formTrigger` | Collects job description text and metadata via web form | None | Map Form Fields | ## 1. Input, Validation and Prep<br>Collects the job description through a form, checks it has enough text to analyze, and trims anything too long before sending it to the AI. Short submissions get a friendly error message instead of wasting an AI call. |
| Map Form Fields | `n8n-nodes-base.code` | Normalizes and cleans raw form data fields | Form Trigger | Long Enough? | ## 1. Input, Validation and Prep<br>Collects the job description through a form, checks it has enough text to analyze, and trims anything too long before sending it to the AI. Short submissions get a friendly error message instead of wasting an AI call. |
| Long Enough? | `n8n-nodes-base.if` | Validates that submission meets minimum length requirements | Map Form Fields | Prepare Input, Too Short - Error Response | ## 1. Input, Validation and Prep<br>Collects the job description through a form, checks it has enough text to analyze, and trims anything too long before sending it to the AI. Short submissions get a friendly error message instead of wasting an AI call. |
| Too Short - Error Response | `n8n-nodes-base.form` | Displays error message for submissions under 150 characters | Long Enough? | None | ## 1. Input, Validation and Prep<br>Collects the job description through a form, checks it has enough text to analyze, and trims anything too long before sending it to the AI. Short submissions get a friendly error message instead of wasting an AI call. |
| Prepare Input | `n8n-nodes-base.code` | Truncates text exceeding token safety limits | Long Enough? | Analyze Bias | ## 1. Input, Validation and Prep<br>Collects the job description through a form, checks it has enough text to analyze, and trims anything too long before sending it to the AI. Short submissions get a friendly error message instead of wasting an AI call. |
| OpenAI Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Provides the language model backend | None | Analyze Bias, Auto-fixing Output Parser | ## 2. AI Analysis and Report<br>The AI scores inclusivity, reads gender coding, and flags discouraging phrases with suggested rewrites. The results are turned into a formatted report and shown back on the same page. |
| Bias Report Schema | `@n8n/n8n-nodes-langchain.outputParserStructured` | Defines expected JSON schema structure | None | Auto-fixing Output Parser | ## 2. AI Analysis and Report<br>The AI scores inclusivity, reads gender coding, and flags discouraging phrases with suggested rewrites. The results are turned into a formatted report and shown back on the same page. |
| Auto-fixing Output Parser | `@n8n/n8n-nodes-langchain.outputParserAutofixing` | Corrects malformed model outputs | OpenAI Chat Model, Bias Report Schema | Analyze Bias | ## 2. AI Analysis and Report<br>The AI scores inclusivity, reads gender coding, and flags discouraging phrases with suggested rewrites. The results are turned into a formatted report and shown back on the same page. |
| Analyze Bias | `@n8n/n8n-nodes-langchain.chainLlm` | Executes DEI analysis prompt against model | Prepare Input, OpenAI Chat Model, Auto-fixing Output Parser | Build HTML Report | ## 2. AI Analysis and Report<br>The AI scores inclusivity, reads gender coding, and flags discouraging phrases with suggested rewrites. The results are turned into a formatted report and shown back on the same page. |
| Build HTML Report | `n8n-nodes-base.code` | Formats analysis JSON into a responsive HTML report | Analyze Bias | Send Report | ## 2. AI Analysis and Report<br>The AI scores inclusivity, reads gender coding, and flags discouraging phrases with suggested rewrites. The results are turned into a formatted report and shown back on the same page. |
| Send Report | `n8n-nodes-base.form` | Displays final HTML report on form completion page | Build HTML Report | None | ## 2. AI Analysis and Report<br>The AI scores inclusivity, reads gender coding, and flags discouraging phrases with suggested rewrites. The results are turned into a formatted report and shown back on the same page. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**
   - Type: `n8n-nodes-base.formTrigger`
   - Configuration: Set form title to `Job Description Bias & Inclusive Language Checker`. Add three fields: `Job Description Text` (textarea, required), `Role Title (optional, improves accuracy)`, and `Industry (optional, e.g. Tech, Healthcare, Construction)`. Set button label to `Check My Job Description`.

2. **Create Map Form Fields Node:**
   - Type: `n8n-nodes-base.code`
   - Connection: Connect `Form Trigger` output to this node.
   - Configuration: Insert JavaScript to normalize form keys into `jdText`, `roleTitle`, and `industry`.

3. **Create Long Enough? Node:**
   - Type: `n8n-nodes-base.if`
   - Connection: Connect `Map Form Fields` output to this node.
   - Configuration: Set condition type to number operation `gte` (greater than or equal). Left value: `={{ $json.jdText.length }}`. Right value: `150`.

4. **Create Too Short - Error Response Node:**
   - Type: `n8n-nodes-base.form`
   - Connection: Connect the `false` (bottom) output of `Long Enough?`.
   - Configuration: Operation: `completion`, Respond With: `showText`. Provide an HTML error template informing the user that submissions must be at least 150 characters.

5. **Create Prepare Input Node:**
   - Type: `n8n-nodes-base.code`
   - Connection: Connect the `true` (top) output of `Long Enough?`.
   - Configuration: Insert JavaScript to check if `text.length > 12000`, truncate to 12,000 characters if true, and return the payload with a `truncated` boolean flag.

6. **Create OpenAI Chat Model Node:**
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
   - Configuration: Select model `gpt-4.1-mini` and set temperature to `0.2`. Configure valid OpenAI credentials.

7. **Create Bias Report Schema Node:**
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`
   - Configuration: Provide the JSON schema defining `inclusivity_score`, `overall_summary`, `gender_coding` object, `flags` array, and `readability_note`.

8. **Create Auto-fixing Output Parser Node:**
   - Type: `@n8n/n8n-nodes-langchain.outputParserAutofixing`
   - Connections: Connect `OpenAI Chat Model` (AI language model connection) and `Bias Report Schema` (AI output parser connection) to this node.

9. **Create Analyze Bias Node:**
   - Type: `@n8n/n8n-nodes-langchain.chainLlm`
   - Connections: Connect `Prepare Input` (Main input), `OpenAI Chat Model` (AI language model connection), and `Auto-fixing Output Parser` (AI output parser connection).
   - Configuration: Set prompt type to define. Paste the DEI-aware system prompt evaluating role title, industry, and `{{ $json.jdText }}` against bias principles.

10. **Create Build HTML Report Node:**
    - Type: `n8n-nodes-base.code`
    - Connection: Connect `Analyze Bias` output to this node.
    - Configuration: Insert JavaScript parsing `{{ $input.first().json.output }}` and generating a fully styled, responsive HTML report containing score breakdowns, gender coding chips, and flagged phrase cards.

11. **Create Send Report Node:**
    - Type: `n8n-nodes-base.form`
    - Connection: Connect `Build HTML Report` output to this node.
    - Configuration: Operation: `completion`, Respond With: `showText`, Response Text: `={{ $json.html }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Designed for recruiters, hiring managers, and HR teams needing a fast second opinion on job posts before publication. | General Workflow Purpose |
| Based on established gender-decoder research by Gaucher, Friesen & Kay and inclusive-hiring best practices. | DEI Analytical Framework |