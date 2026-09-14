Parse and normalize resume PDFs with Telegram, OpenAI, and Gotenberg

https://n8nworkflows.xyz/workflows/parse-and-normalize-resume-pdfs-with-telegram--openai--and-gotenberg-17725


# Parse and normalize resume PDFs with Telegram, OpenAI, and Gotenberg

### 1. Workflow Overview

This workflow automates the intake, parsing, restructuring, and formatting of candidate resume PDFs received via a Telegram bot. It extracts unstructured resume text, leverages OpenAI language models to structure and validate the data against a strict JSON schema, transforms the data into modular HTML sections, and compiles it via a self-hosted Gotenberg instance into a polished PDF document sent directly back to the user on Telegram.

The workflow logic is divided into four functional blocks:
- **1.1 Telegram Intake & Auth:** Listens for incoming webhook events from Telegram, authenticates the sender chat ID, filters out command messages (`/start`), and downloads the attached resume document.
- **1.2 AI Resume Parsing:** Extracts the raw text from the downloaded PDF and passes it through an OpenAI-powered chain equipped with a structured output parser and an auto-fixing mechanism to guarantee strict schema compliance.
- **1.3 HTML Section Building:** Splits the structured JSON payload across parallel branches to generate formatted HTML blocks for personal details, technologies, employment history, education, projects, and volunteering, systematically merging them back into a single unified HTML string.
- **1.4 PDF Generation & Delivery:** Base64-encodes the final HTML payload, converts it to a binary HTML file, submits it to a Gotenberg rendering service to create a fresh PDF, and sends the resulting document back to the original Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Intake & Auth
**Overview:** This block acts as the entry point for the automation. It captures incoming messages from Telegram, validates the sender against an administrative whitelist, discards system commands, and retrieves the binary PDF file attached to valid requests.

- **Nodes Involved:** 
  - `Telegram trigger`
  - `Auth`
  - `No operation (unauthorized)`
  - `Check if start message`
  - `No operation (start message)`
  - `Get file`

- **Node Details:**
  - **Telegram trigger**
    - *Type and Technical Role:* `n8n-nodes-base.telegramTrigger` (Trigger)
    - *Configuration Choices:* Listens to `message` updates via webhook.
    - *Key Expressions:* None.
    - *Connections:* Input: None; Output: `Auth`.
    - *Version-specific Requirements:* Version 1.1.
    - *Edge Cases / Failures:* Webhook delivery failure if n8n instance URL changes or SSL certificates expire.
  - **Auth**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Evaluates whether the incoming chat ID matches the authorized threshold (default value: `0`).
    - *Key Expressions:* `={{ $json.message.chat.id }}` equals `0`.
    - *Connections:* Input: `Telegram trigger`; Output (True): `Check if start message`; Output (False): `No operation (unauthorized)`.
    - *Version-specific Requirements:* Version 2.
    - *Edge Cases / Failures:* Unauthorized users receive no feedback and terminate silently at the `No operation` node.
  - **No operation (unauthorized)**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Utility)
    - *Configuration Choices:* Empty pass-through node for terminating unauthorized execution paths.
    - *Connections:* Input: `Auth` (False); Output: None.
  - **Check if start message**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Verifies that the incoming message text is not equal to `/start`.
    - *Key Expressions:* `={{ $json.message.text }}` does not equal `/start`.
    - *Connections:* Input: `Auth` (True); Output (True): `Get file`; Output (False): `No operation (start message)`.
    - *Version-specific Requirements:* Version 2.
  - **No operation (start message)**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Utility)
    - *Configuration Choices:* Empty pass-through node for terminating `/start` command threads.
    - *Connections:* Input: `Check if start message` (False); Output: None.
  - **Get file**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Action)
    - *Configuration Choices:* Downloads file resources using the file ID extracted from the incoming message document.
    - *Key Expressions:* File ID: `={{ $json.message.document.file_id }}`.
    - *Connections:* Input: `Check if start message` (True); Output: `Extract text from PDF`.
    - *Version-specific Requirements:* Version 1.1.
    - *Edge Cases / Failures:* Fails if the uploaded file is missing, exceeds Telegram bot download limits, or lacks proper Telegram API credentials.

#### 2.2 AI Resume Parsing
**Overview:** This block extracts raw textual content from the downloaded PDF binary and processes it using an OpenAI language model chain. It enforces a strict schema using a structured parser combined with an auto-fixing mechanism to handle and correct any structural JSON anomalies.

- **Nodes Involved:**
  - `Extract text from PDF`
  - `Parse resume data`
  - `OpenAI Chat Model`
  - `Auto-fixing Output Parser`
  - `OpenAI Chat Model1`
  - `Structured Output Parser`
  - `Set parsed fileds`

- **Node Details:**
  - **Extract text from PDF**
    - *Type and Technical Role:* `n8n-nodes-base.extractFromFile` (Data Transformation)
    - *Configuration Choices:* Operation set to `pdf` text extraction.
    - *Connections:* Input: `Get file`; Output: `Parse resume data`.
    - *Edge Cases / Failures:* Fails on password-protected PDFs or scanned documents that contain no selectable character layers.
  - **Parse resume data**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI / LangChain Chain)
    - *Configuration Choices:* System prompt instructing the model to extract and return unified JSON data without hallucinations.
    - *Key Expressions:* Prompt: `={{ $json.text }}`.
    - *Connections:* Input: `Extract text from PDF` and `OpenAI Chat Model` (AI Model) and `Auto-fixing Output Parser` (AI Output Parser); Output: `Set parsed fileds`.
    - *Version-specific Requirements:* Version 1.3.
  - **OpenAI Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model)
    - *Configuration Choices:* Uses `gpt-4-turbo-preview`, temperature set to `0`, response format configured to `json_object`.
    - *Connections:* Output: Connected to `Parse resume data` (AI Language Model).
  - **Auto-fixing Output Parser**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserAutofixing` (AI Parser)
    - *Connections:* Input: `OpenAI Chat Model1` (AI Model) and `Structured Output Parser` (AI Output Parser); Output: Connected to `Parse resume data` (AI Output Parser).
  - **OpenAI Chat Model1**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model)
    - *Configuration Choices:* Temperature set to `0`.
    - *Connections:* Output: Connected to `Auto-fixing Output Parser` (AI Language Model).
  - **Structured Output Parser**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Parser)
    - *Configuration Choices:* Configured with a comprehensive JSON schema enforcing types for personal info, employment history, education, projects, volunteering, programming languages, and foreign languages.
    - *Connections:* Output: Connected to `Auto-fixing Output Parser` (AI Output Parser).
  - **Set parsed fileds**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Keeps incoming validated JSON payload and duplicates it across all downstream formatting outputs.
    - *Connections:* Input: `Parse resume data`; Output: `Personal info`, `Technologies`, `Convert employment history to HTML`, `Convert education to HTML`, `Convert projects to HTML`, and `Convert volunteering to HTML`.
    - *Version-specific Requirements:* Version 3.2.

#### 2.3 HTML Section Building
**Overview:** This block fans out the structured JSON object into six parallel conversion paths. Code nodes loop through individual array items (such as jobs, education entries, and projects) to generate clean HTML markup, which is then concatenated with headers and systematically merged back into a unified data payload.

- **Nodes Involved:**
  - `Personal info`
  - `Technologies`
  - `Convert employment history to HTML`
  - `Employment history`
  - `Convert education to HTML`
  - `Education`
  - `Convert projects to HTML`
  - `Projects`
  - `Convert volunteering to HTML`
  - `Volunteering`
  - `Merge education and employment history`
  - `Merge projects and volunteering`
  - `Merge personal info and technologies`
  - `Merge other data`
  - `Merge all`

- **Node Details:**
  - **Personal info**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Assigns an HTML string to the `personal_info` field using dot notation referencing resume attributes (name, address, email, GitHub).
    - *Key Expressions:* Evaluates `{{ $json.personal_info.name }}`, `{{ $json.personal_info.address }}`, etc.
    - *Connections:* Input: `Set parsed fileds`; Output: `Merge personal info and technologies`.
  - **Technologies**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Builds an HTML block for programming languages, tools, and methodologies arrays joined by comma separators.
    - *Key Expressions:* Uses `.join(', ')` on respective array properties.
    - *Connections:* Input: `Set parsed fileds`; Output: `Merge personal info and technologies`.
  - **Convert employment history to HTML**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Custom Code)
    - *Configuration Choices:* JavaScript loop processing `employment_history` items and nested responsibilities into formatted HTML strings.
    - *Key Expressions:* Processes `$input.item.json.employment_history`.
    - *Connections:* Input: `Set parsed fileds`; Output: `Employment history`.
  - **Employment history**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Wraps code output with an `Employment history` header.
    - *Key Expressions:* `={{ $json["htmlOutput"] }}`.
    - *Connections:* Input: `Convert employment history to HTML`; Output: `Merge education and employment history`.
  - **Convert education to HTML**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Custom Code)
    - *Configuration Choices:* JavaScript loop converting educational background entries into HTML blocks.
    - *Connections:* Input: `Set parsed fileds`; Output: `Education`.
  - **Education**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Wraps code output with an `Education` header.
    - *Connections:* Input: `Convert education to HTML`; Output: `Merge education and employment history`.
  - **Convert projects to HTML**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Custom Code)
    - *Configuration Choices:* JavaScript loop parsing project arrays, descriptions, and associated technologies into HTML.
    - *Connections:* Input: `Set parsed fileds`; Output: `Projects`.
  - **Projects**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Wraps code output with a `Projects` header.
    - *Connections:* Input: `Convert projects to HTML`; Output: `Merge projects and volunteering`.
  - **Convert volunteering to HTML**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Custom Code)
    - *Configuration Choices:* JavaScript loop transforming volunteering activities into HTML blocks.
    - *Connections:* Input: `Set parsed fileds`; Output: `Volunteering`.
  - **Volunteering**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Wraps code output with a `Volunteering` header.
    - *Connections:* Input: `Convert volunteering to HTML`; Output: `Merge projects and volunteering`.
  - **Merge education and employment history**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data Transformation)
    - *Configuration Choices:* Mode set to `combine`, combination mode set to `multiplex`.
    - *Connections:* Input: `Employment history`, `Education`; Output: `Merge other data`.
  - **Merge projects and volunteering**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data Transformation)
    - *Configuration Choices:* Mode set to `combine`, combination mode set to `multiplex`.
    - *Connections:* Input: `Projects`, `Volunteering`; Output: `Merge other data`.
  - **Merge personal info and technologies**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data Transformation)
    - *Configuration Choices:* Mode set to `combine`, combination mode set to `multiplex`.
    - *Connections:* Input: `Personal info`, `Technologies`; Output: `Merge all`.
  - **Merge other data**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data Transformation)
    - *Configuration Choices:* Mode set to `combine`, combination mode set to `multiplex`.
    - *Connections:* Input: `Merge education and employment history`, `Merge projects and volunteering`; Output: `Merge all`.
  - **Merge all**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data Transformation)
    - *Configuration Choices:* Mode set to `combine`, combination mode set to `multiplex`.
    - *Connections:* Input: `Merge personal info and technologies`, `Merge other data`; Output: `Set final data`.

#### 2.4 PDF Generation & Delivery
**Overview:** This block consolidates all HTML blocks into a single document string, encodes it into base64, packages it as a binary HTML file, and submits it to a Gotenberg server to render a professional PDF delivered back to the candidate via Telegram.

- **Nodes Involved:**
  - `Set final data`
  - `Convert raw to base64`
  - `Convert to HTML`
  - `Generate plain PDF doc`
  - `Send PDF to the user`

- **Node Details:**
  - **Set final data**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Concatenates all section attributes (`personal_info`, `employment_history`, `education`, `projects`, `volunteering`, `technologies`) into a single string under the property `output`.
    - *Key Expressions:* `={{ $json.personal_info }} ... {{ $json.technologies }}`.
    - *Connections:* Input: `Merge all`; Output: `Convert raw to base64`.
  - **Convert raw to base64**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Custom Code)
    - *Configuration Choices:* Converts the combined HTML string to a base64 encoded buffer.
    - *Key Expressions:* `Buffer.from($json.output).toString('base64')`.
    - *Connections:* Input: `Set final data`; Output: `Convert to HTML`.
  - **Convert to HTML**
    - *Type and Technical Role:* `n8n-nodes-base.convertToFile` (Data Transformation)
    - *Configuration Choices:* Converts binary data into a file named `index.html` with mime type `text/html`.
    - *Connections:* Input: `Convert raw to base64`; Output: `Generate plain PDF doc`.
  - **Generate plain PDF doc**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Action / HTTP Request)
    - *Configuration Choices:* Sends a POST request using `multipart/form-data` to the Gotenberg Chromium conversion endpoint (`http://gotenberg:3000/forms/chromium/convert/html`), configured to expect a file in the response.
    - *Connections:* Input: `Convert to HTML`; Output: `Send PDF to the user`.
    - *Edge Cases / Failures:* Timeouts or connection failures if the local Gotenberg container is offline or unreachable.
  - **Send PDF to the user**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Action)
    - *Configuration Choices:* Sends a document (`sendDocument`) using binary data back to the originating chat ID.
    - *Key Expressions:* Chat ID: `={{ $('Telegram trigger').item.json["message"]["chat"]["id"] }}`; Filename: `={{ $('Set parsed fileds').item.json["personal_info"]["name"].toLowerCase().replace(' ', '-') }}.pdf`.
    - *Connections:* Input: `Generate plain PDF doc`; Output: None.
    - *Edge Cases / Failures:* Fails if the candidate blocked the bot or if file size limits imposed by Telegram are exceeded.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| OpenAI Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | AI Model | None | Parse resume data | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Convert education to HTML | `n8n-nodes-base.code` | Custom Code | Set parsed fileds | Education | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Auto-fixing Output Parser | `@n8n/n8n-nodes-langchain.outputParserAutofixing` | AI Parser | OpenAI Chat Model1, Structured Output Parser | Parse resume data | 2️⃣ AI Resume Parsing<br><br>**Extract text from PDF** pulls the raw text out of the resume... |
| OpenAI Chat Model1 | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | AI Model | None | Auto-fixing Output Parser | 2️⃣ AI Resume Parsing<br><br>**Extract text from PDF** pulls the raw text out of the resume... |
| Structured Output Parser | `@n8n/n8n-nodes-langchain.outputParserStructured` | AI Parser | None | Auto-fixing Output Parser | 2️⃣ AI Resume Parsing<br><br>**Extract text from PDF** pulls the raw text out of the resume... |
| Convert employment history to HTML | `n8n-nodes-base.code` | Custom Code | Set parsed fileds | Employment history | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Convert projects to HTML | `n8n-nodes-base.code` | Custom Code | Set parsed fileds | Projects | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Convert volunteering to HTML | `n8n-nodes-base.code` | Custom Code | Set parsed fileds | Volunteering | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Telegram trigger | `n8n-nodes-base.telegramTrigger` | Trigger | None | Auth | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Telegram trigger | `n8n-nodes-base.telegramTrigger` | Trigger | None | Auth | 1️⃣ Telegram Intake & Auth<br><br>**Telegram trigger** fires on every incoming message, and **Auth** checks the sender's chat id... |
| Auth | `n8n-nodes-base.if` | Flow Control | Telegram trigger | Check if start message, No operation (unauthorized) | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Auth | `n8n-nodes-base.if` | Flow Control | Telegram trigger | Check if start message, No operation (unauthorized) | 1️⃣ Telegram Intake & Auth<br><br>**Telegram trigger** fires on every incoming message, and **Auth** checks the sender's chat id... |
| No operation (unauthorized) | `n8n-nodes-base.noOp` | Utility | Auth | None | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| No operation (unauthorized) | `n8n-nodes-base.noOp` | Utility | Auth | None | 1️⃣ Telegram Intake & Auth<br><br>**Telegram trigger** fires on every incoming message, and **Auth** checks the sender's chat id... |
| Check if start message | `n8n-nodes-base.if` | Flow Control | Auth | Get file, No operation (start message) | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Check if start message | `n8n-nodes-base.if` | Flow Control | Auth | Get file, No operation (start message) | 1️⃣ Telegram Intake & Auth<br><br>**Telegram trigger** fires on every incoming message, and **Auth** checks the sender's chat id... |
| No operation (start message) | `n8n-nodes-base.noOp` | Utility | Check if start message | None | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| No operation (start message) | `n8n-nodes-base.noOp` | Utility | Check if start message | None | 1️⃣ Telegram Intake & Auth<br><br>**Telegram trigger** fires on every incoming message, and **Auth** checks the sender's chat id... |
| Get file | `n8n-nodes-base.telegram` | Action | Check if start message | Extract text from PDF | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Get file | `n8n-nodes-base.telegram` | Action | Check if start message | Extract text from PDF | 1️⃣ Telegram Intake & Auth<br><br>**Telegram trigger** fires on every incoming message, and **Auth** checks the sender's chat id... |
| Extract text from PDF | `n8n-nodes-base.extractFromFile` | Data Transformation | Get file | Parse resume data | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Extract text from PDF | `n8n-nodes-base.extractFromFile` | Data Transformation | Get file | Parse resume data | 2️⃣ AI Resume Parsing<br><br>**Extract text from PDF** pulls the raw text out of the resume... |
| Set parsed fileds | `n8n-nodes-base.set` | Data Transformation | Parse resume data | Convert employment history to HTML, Convert education to HTML, Convert projects to HTML, Personal info, Convert volunteering to HTML, Technologies | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Set parsed fileds | `n8n-nodes-base.set` | Data Transformation | Parse resume data | Convert employment history to HTML, Convert education to HTML, Convert projects to HTML, Personal info, Convert volunteering to HTML, Technologies | 2️⃣ AI Resume Parsing<br><br>**Extract text from PDF** pulls the raw text out of the resume... |
| Personal info | `n8n-nodes-base.set` | Data Transformation | Set parsed fileds | Merge personal info and technologies | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Personal info | `n8n-nodes-base.set` | Data Transformation | Set parsed fileds | Merge personal info and technologies | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Technologies | `n8n-nodes-base.set` | Data Transformation | Set parsed fileds | Merge personal info and technologies | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Technologies | `n8n-nodes-base.set` | Data Transformation | Set parsed fileds | Merge personal info and technologies | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Employment history | `n8n-nodes-base.set` | Data Transformation | Convert employment history to HTML | Merge education and employment history | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Education | `n8n-nodes-base.set` | Data Transformation | Convert education to HTML | Merge education and employment history | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Projects | `n8n-nodes-base.set` | Data Transformation | Convert projects to HTML | Merge projects and volunteering | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Volunteering | `n8n-nodes-base.set` | Data Transformation | Convert volunteering to HTML | Merge projects and volunteering | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Merge education and employment history | `n8n-nodes-base.merge` | Data Transformation | Employment history, Education | Merge other data | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Merge projects and volunteering | `n8n-nodes-base.merge` | Data Transformation | Projects, Volunteering | Merge other data | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Merge personal info and technologies | `n8n-nodes-base.merge` | Data Transformation | Personal info, Technologies | Merge all | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Merge all | `n8n-nodes-base.merge` | Data Transformation | Merge personal info and technologies, Merge other data | Set final data | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Merge all | `n8n-nodes-base.merge` | Data Transformation | Merge personal info and technologies, Merge other data | Set final data | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |
| Set final data | `n8n-nodes-base.set` | Data Transformation | Merge all | Convert raw to base64 | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Set final data | `n8n-nodes-base.set` | Data Transformation | Merge all | Convert raw to base64 | 4️⃣ PDF Generation & Delivery<br><br>**Set final data** concatenates every formatted section into one HTML document... |
| Convert raw to base64 | `n8n-nodes-base.code` | Custom Code | Set final data | Convert to HTML | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Convert raw to base64 | `n8n-nodes-base.code` | Custom Code | Set final data | Convert to HTML | 4️⃣ PDF Generation & Delivery<br><br>**Set final data** concatenates every formatted section into one HTML document... |
| Convert to HTML | `n8n-nodes-base.convertToFile` | Data Transformation | Convert raw to base64 | Generate plain PDF doc | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Convert to HTML | `n8n-nodes-base.convertToFile` | Data Transformation | Convert raw to base64 | Generate plain PDF doc | 4️⃣ PDF Generation & Delivery<br><br>**Set final data** concatenates every formatted section into one HTML document... |
| Generate plain PDF doc | `n8n-nodes-base.httpRequest` | Action | Convert to HTML | Send PDF to the user | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Generate plain PDF doc | `n8n-nodes-base.httpRequest` | Action | Convert to HTML | Send PDF to the user | 4️⃣ PDF Generation & Delivery<br><br>**Set final data** concatenates every formatted section into one HTML document... |
| Send PDF to the user | `n8n-nodes-base.telegram` | Action | Generate plain PDF doc | None | 📄 Resume Data Extraction with Telegram and Gotenberg<br><br>Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF... |
| Send PDF to the user | `n8n-nodes-base.telegram` | Action | Generate plain PDF doc | None | 4️⃣ PDF Generation & Delivery<br><br>**Set final data** concatenates every formatted section into one HTML document... |
| Parse resume data | `@n8n/n8n-nodes-langchain.chainLlm` | AI / LangChain Chain | Extract text from PDF, OpenAI Chat Model, Auto-fixing Output Parser | Set parsed fileds | 2️⃣ AI Resume Parsing<br><br>**Extract text from PDF** pulls the raw text out of the resume... |
| Merge other data | `n8n-nodes-base.merge` | Data Transformation | Merge education and employment history, Merge projects and volunteering | Merge all | 3️⃣ HTML Section Building<br><br>Six parallel branches format each section of the resume... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Telegram Intake & Auth Setup
1. Create a **Telegram Trigger** node (`n8n-nodes-base.telegramTrigger`). Set `Updates` to `message`. Configure valid Telegram Bot credentials.
2. Create an **IF** node named `Auth` (`n8n-nodes-base.if`). Add a condition checking if `{{ $json.message.chat.id }}` equals your target authorized user chat ID (replace default `0`).
3. Connect `Telegram Trigger` output `0` to `Auth` input `0`.
4. Create a **No Operation** node named `No operation (unauthorized)` (`n8n-nodes-base.noOp`). Connect `Auth` output `1` (False) to it.
5. Create an **IF** node named `Check if start message` (`n8n-nodes-base.if`). Add a condition where `{{ $json.message.text }}` does not equal `/start`.
6. Connect `Auth` output `0` (True) to `Check if start message`.
7. Create a **No Operation** node named `No operation (start message)` (`n8n-nodes-base.noOp`). Connect `Check if start message` output `1` (False) to it.
8. Create a **Telegram** node named `Get file` (`n8n-nodes-base.telegram`). Set resource to `file` and configure parameter `File ID` to `={{ $json.message.document.file_id }}` using the same Telegram credentials.
9. Connect `Check if start message` output `0` (True) to `Get file`.

#### Step 2: AI Parsing Setup
10. Create an **Extract From File** node named `Extract text from PDF` (`n8n-nodes-base.extractFromFile`). Set operation to `pdf`.
11. Connect `Get file` to `Extract text from PDF`.
12. Create an **Advanced AI Chat Model** node named `OpenAI Chat Model` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`). Set model to `gpt-4-turbo-preview`, temperature to `0`, and response format to `json_object`. Configure OpenAI API credentials.
13. Create a **Structured Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserStructured`). Paste the JSON schema definition covering `personal_info`, `employment_history`, `education`, `projects`, `volunteering`, `programming_languages`, and `foreign_languages`.
14. Create an **Advanced AI Chat Model** node named `OpenAI Chat Model1` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`). Set temperature to `0` and provide OpenAI API credentials.
15. Create an **Auto-fixing Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserAutofixing`). Connect `OpenAI Chat Model1` to its `ai_languageModel` input and `Structured Output Parser` to its `ai_outputParser` input.
16. Create an **AI Basic LLM Chain** node named `Parse resume data` (`@n8n/n8n-nodes-langchain.chainLlm`). Set prompt to `={{ $json.text }}` and include the system message instructing data extraction without making things up.
17. Connect `OpenAI Chat Model` to `Parse resume data` (`ai_languageModel`), `Auto-fixing Output Parser` to `Parse resume data` (`ai_outputParser`), and `Extract text from PDF` to `Parse resume data` (`main`).
18. Create a **Set** node named `Set parsed fileds` (`n8n-nodes-base.set`). Connect `Parse resume data` to it.

#### Step 3: HTML Building & Merging Setup
19. Create two **Set** nodes named `Personal info` and `Technologies` (`n8n-nodes-base.set`), configuring their string values using HTML tags and mustache expressions matching `personal_info` properties and `.join(', ')` arrays respectively. Connect both to `Set parsed fileds`.
20. Create four **Code** nodes named `Convert employment history to HTML`, `Convert education to HTML`, `Convert projects to HTML`, and `Convert volunteering to HTML` (`n8n-nodes-base.code`), setting mode to `runOnceForEachItem` and inserting respective JavaScript conversion functions to transform arrays into HTML text. Connect all four to `Set parsed fileds`.
21. Create four **Set** nodes named `Employment history`, `Education`, `Projects`, and `Volunteering` (`n8n-nodes-base.set`), mapping their values to `={{ $json["htmlOutput"] }}` with section header formatting. Connect each code node to its corresponding set node.
22. Create **Merge** nodes (`n8n-nodes-base.merge`) configured in `combine` mode with `multiplex` combination:
    - `Merge education and employment history`: Inputs from `Employment history` (0) and `Education` (1).
    - `Merge projects and volunteering`: Inputs from `Projects` (0) and `Volunteering` (1).
    - `Merge personal info and technologies`: Inputs from `Personal info` (0) and `Technologies` (1).
23. Create a **Merge** node named `Merge other data` (`n8n-nodes-base.merge`) with inputs from `Merge education and employment history` (0) and `Merge projects and volunteering` (1).
24. Create a **Merge** node named `Merge all` (`n8n-nodes-base.merge`) with inputs from `Merge personal info and technologies` (0) and `Merge other data` (1).

#### Step 4: PDF Compilation & Delivery Setup
25. Create a **Set** node named `Set final data` (`n8n-nodes-base.set`) concatenating all section strings into an `output` field. Connect `Merge all` to it.
26. Create a **Code** node named `Convert raw to base64` (`n8n-nodes-base.code`) using `Buffer.from($json.output).toString('base64')`. Connect `Set final data` to it.
27. Create a **Convert to File** node named `Convert to HTML` (`n8n-nodes-base.convertToFile`) with operation `toBinary`, source property `encoded`, filename `index.html`, and mime type `text/html`. Connect `Convert raw to base64` to it.
28. Create an **HTTP Request** node named `Generate plain PDF doc` (`n8n-nodes-base.httpRequest`). Set method to `POST`, URL to `http://gotenberg:3000/forms/chromium/convert/html`, content type to `multipart/form-data`, body parameters to send binary property `data` under parameter name `files`, and response format to `file`. Connect `Convert to HTML` to it.
29. Create a **Telegram** node named `Send PDF to the user` (`n8n-nodes-base.telegram`). Set resource to `message`, operation to `sendDocument`, enable binary data, set chat ID expression to `={{ $('Telegram trigger').item.json["message"]["chat"]["id"] }}`, and filename expression to `={{ $('Set parsed fileds').item.json["personal_info"]["name"].toLowerCase().replace(' ', '-') }}.pdf`. Connect `Generate plain PDF doc` to it.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Resume Data Extraction with Telegram and Gotenberg Overview | Turns any resume PDF sent to a Telegram bot into a clean, reformatted PDF processed via OpenAI and Gotenberg. |
| Gotenberg Endpoint Configuration | Default local endpoint URL: `http://gotenberg:3000/forms/chromium/convert/html`. Update if hosted remotely. |
| PDF Document Constraints | Assumes submitted documents are text-selectable PDF resumes. Scanned image-only PDFs will fail text extraction. |