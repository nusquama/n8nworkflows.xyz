Compare personal grocery inflation to US CPI with Gemini and Telegram

https://n8nworkflows.xyz/workflows/compare-personal-grocery-inflation-to-us-cpi-with-gemini-and-telegram-20532


# Compare personal grocery inflation to US CPI with Gemini and Telegram

### 1. Workflow Overview

This workflow processes grocery receipt photos submitted via an n8n Form or a Telegram Bot, extracts line items using Google Gemini, matches products against a historical database, calculates a personal spending-weighted inflation rate, and compares it with the official US Consumer Price Index (CPI) pulled from the US Bureau of Labor Statistics (BLS) API. Finally, it generates a human-friendly economist report, optionally renders an audio voice note via ElevenLabs, and returns the results to the user's origin channel.

The logic is grouped into five functional blocks:
- **1.1 Intake & Initialization:** Receives triggers from the Web Form or Telegram, injects runtime configuration settings, and initializes/loads the persistent n8n Data Table.
- **1.2 Receipt Ingestion & Parsing:** Normalizes inputs into structured datasets, processes unparsed or user-uploaded photos through Google Gemini LLM chains, and handles sample households.
- **1.3 Product Matching & Pricing:** Compares incoming items against known product keys, leverages an AI agent to assign normalized keys to new entries, prices items, and upserts them to the database.
- **1.4 Mathematical Computation & Official Indexing:** Fetches live official CPI data from the BLS API and calculates a Laspeyres-style personal inflation rate based on spending weights across months.
- **1.5 Reporting, Voice Synthesis & Delivery:** Synthesizes an economist's commentary via Gemini, generates an optional audio summary via ElevenLabs, and routes the completed HTML or Telegram output back to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake & Initialization
- **Overview:** Sets up core runtime variables, accepts incoming webhook requests from either a web form or a Telegram chat, and ensures the target data table for receipt rows exists.
- **Nodes Involved:** `Your receipts`, `Telegram receipts`, `Settings`, `Create receipt lines table`, `Load saved lines`, `Gather receipts`, `What came in?`.
- **Node Details:**
  - **Your receipts** (`n8n-nodes-base.formTrigger`):
    - *Role:* Web form entry point.
    - *Configuration:* Collects options (sample household vs. upload), receipt image files, and household name.
    - *Connections:* Inbound trigger; Output connects to `Settings`.
  - **Telegram receipts** (`n8n-nodes-base.telegramTrigger`):
    - *Role:* Telegram bot entry point.
    - *Configuration:* Listens for messages, downloads attachments, and handles image sizes.
    - *Connections:* Inbound trigger; Output connects to `Settings`.
  - **Settings** (`n8n-nodes-base.set`):
    - *Role:* Establishes global workflow variables.
    - *Configuration:* Sets `official_source` ("BLS"), `official_series_id` ("CUUR0000SAF11"), currency symbol, narrator voice parameters, and ElevenLabs credentials/models.
    - *Connections:* Input from `Your receipts` or `Telegram receipts`; Output to `Create receipt lines table`.
  - **Create receipt lines table** (`n8n-nodes-base.dataTable`):
    - *Role:* Database schema provisioning.
    - *Configuration:* Creates `receipt_economist_lines` if missing, defining columns like `line_id`, `household_id`, `item_key`, `unit_price`, and `needs_review`.
    - *Connections:* Input from `Settings`; Output to `Load saved lines`.
  - **Load saved lines** (`n8n-nodes-base.dataTable`):
    - *Role:* State retrieval.
    - *Configuration:* Retrieves all existing rows from `receipt_economist_lines`.
    - *Connections:* Input from `Create receipt lines table`; Output to `Gather receipts`.
  - **Gather receipts** (`n8n-nodes-base.code`):
    - *Role:* Payload normalization and routing logic.
    - *Configuration:* Javascript switch distinguishing between sample household mode, photo uploads, reports, and help commands.
    - *Connections:* Input from `Load saved lines`; Output to `What came in?`.
  - **What came in?** (`n8n-nodes-base.switch`):
    - *Role:* Multibranch workflow router.
    - *Configuration:* Evaluates `$json.kind` to route to photos, sample receipts, reports, or help instructions.
    - *Connections:* Input from `Gather receipts`; Outputs connect to `Read receipt photo`, `Find lines to match`, `Fetch official index`, and `Build report`.

#### 2.2 Receipt Ingestion & Parsing
- **Overview:** Extracts raw line-item text, store names, and dates from uploaded receipt images utilizing a structured Google Gemini vision chain.
- **Nodes Involved:** `Read receipt photo`, `Receipt format`, `Gemini (reads & matches)`.
- **Node Details:**
  - **Read receipt photo** (`@n8n/n8n-nodes-langchain.chainLlm`):
    - *Role:* Vision LLM node for receipt transcription.
    - *Configuration:* Uses structured output parsing, `maxTries: 2`, and passes binary image keys (`data`).
    - *Connections:* Input from `What came in?` (Photos branch); Output to `Find lines to match`.
  - **Receipt format** (`@n8n/n8n-nodes-langchain.outputParserStructured`):
    - *Role:* JSON Schema enforcement for OCR data.
    - *Configuration:* Enforces schema containing `store`, `purchase_date`, and `lines` array (`raw_text`, `quantity`, `line_total`).
    - *Connections:* Connected as AI output parser to `Read receipt photo`.
  - **Gemini (reads & matches)** (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`):
    - *Role:* LLM backend model provider.
    - *Configuration:* Uses model `models/gemini-3.8-flash` with a temperature of `0`.
    - *Connections:* Connected as AI language model provider to `Read receipt photo` and `Match new lines to products`.

#### 2.3 Product Matching & Pricing
- **Overview:** Maps unclassified receipt lines against previously learned product definitions or invokes an AI agent to assign standardized keys, before calculating unit prices and persisting data.
- **Nodes Involved:** `Find lines to match`, `Anything new to match?`, `Match new lines to products`, `Search saved products`, `Matches format`, `Price matched lines`, `Save receipt lines`.
- **Node Details:**
  - **Find lines to match** (`n8n-nodes-base.code`):
    - *Role:* Deduplication and diff check.
    - *Configuration:* Compares incoming raw texts against known mapped rows loaded from the database.
    - *Connections:* Input from `Read receipt photo` and `What came in?` (Sample receipts branch); Output to `Anything new to match?`.
  - **Anything new to match?** (`n8n-nodes-base.switch`):
    - *Role:* Branching on whether unrecognized lines exist.
    - *Configuration:* Evaluates `unmatched_count`.
    - *Connections:* Input from `Find lines to match`; Outputs connect to `Match new lines to products`, `Price matched lines`, and `Build report`.
  - **Match new lines to products** (`@n8n/n8n-nodes-langchain.agent`):
    - *Role:* Autonomous AI agent for item categorization and key assignment.
    - *Configuration:* Iterative agent using system instructions to normalize product names into snake_case keys.
    - *Connections:* Input from `Anything new to match?`; Output to `Price matched lines`.
  - **Search saved products** (`n8n-nodes-base.dataTableTool`):
    - *Role:* Tool for the AI agent to search historical entries in the Data Table.
    - *Configuration:* Queries `receipt_economist_lines` via `ilike` filters on labels or item keys.
    - *Connections:* Connected as a tool to `Match new lines to products`.
  - **Matches format** (`@n8n/n8n-nodes-langchain.outputParserStructured`):
    - *Role:* Schema enforcement for AI product mapping.
    - *Configuration:* Enforces array schema containing `raw_text`, `item_key`, `label`, `category`, and `needs_review`.
    - *Connections:* Connected as AI output parser to `Match new lines to products`.
  - **Price matched lines** (`n8n-nodes-base.code`):
    - *Role:* Financial calculations per line.
    - *Configuration:* Computes `unit_price` (`line_total / quantity`) and flags review statuses.
    - *Connections:* Input from `Match new lines to products` and `Anything new to match?`; Output to `Save receipt lines`.
  - **Save receipt lines** (`n8n-nodes-base.dataTable`):
    - *Role:* Database upsert operation.
    - *Configuration:* Upserts structured line items into `receipt_economist_lines` using `line_id` as the match condition.
    - *Connections:* Input from `Price matched lines`; Output to `Fetch official index`.

#### 2.4 Mathematical Computation & Official Indexing
- **Overview:** Pulls macro inflation statistics from the US BLS public API and computes personal Laspeyres price indices.
- **Nodes Involved:** `Fetch official index`, `Calculate personal inflation`.
- **Node Details:**
  - **Fetch official index** (`n8n-nodes-base.httpRequest`):
    - *Role:* External API client for macroeconomic benchmarks.
    - *Configuration:* POST request to `https://api.bls.gov/publicAPI/v1/timeseries/data/` sending dynamic series IDs and year ranges.
    - *Connections:* Input from `Save receipt lines` and `What came in?` (Report branch); Output to `Calculate personal inflation`.
  - **Calculate personal inflation** (`n8n-nodes-base.code`):
    - *Role:* Econometric calculation engine.
    - *Configuration:* Computes spending-weighted price changes between the two most recent months containing receipts, alongside category breakouts and BLS index delta comparisons.
    - *Connections:* Input from `Fetch official index`; Output to `Write the economist's note`.

#### 2.5 Reporting, Voice Synthesis & Delivery
- **Overview:** Converts quantitative calculations into structured text narratives, synthesizes audio output via ElevenLabs, and routes the final HTML page or Telegram message.
- **Nodes Involved:** `Write the economist's note`, `Gemini (writes)`, `Note format`, `Voice note wanted?`, `Speak the note`, `Build report`, `Reply where they asked`, `Show the report`, `Telegram: send voice note`, `Telegram: send report`.
- **Node Details:**
  - **Write the economist's note** (`@n8n/n8n-nodes-langchain.chainLlm`):
    - *Role:* Narrative generation LLM node.
    - *Configuration:* Generates qualitative insights, driver lists, and voice note scripts based on pre-calculated numbers.
    - *Connections:* Input from `Calculate personal inflation`; Output to `Voice note wanted?`.
  - **Gemini (writes)** (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`):
    - *Role:* LLM provider for the writing chain.
    - *Configuration:* Uses model `models/gemini-3.8-flash` with a temperature of `0.7`.
    - *Connections:* Connected as AI language model provider to `Write the economist's note` and `Note format`.
  - **Note format** (`@n8n/n8n-nodes-langchain.outputParserStructured`):
    - *Role:* Schema enforcement for the economist note.
    - *Configuration:* Enforces schema containing `headline`, `verdict`, `drivers`, `tip`, `method_note`, and `voice_note`.
    - *Connections:* Connected as AI output parser to `Write the economist's note`.
  - **Voice note wanted?** (`n8n-nodes-base.if`):
    - *Role:* Conditional check for audio generation.
    - *Configuration:* Evaluates whether the channel requests a voice note and if an ElevenLabs voice ID is configured.
    - *Connections:* Input from `Write the economist's note`; Outputs connect to `Speak the note` and `Build report`.
  - **Speak the note** (`n8n-nodes-base.httpRequest`):
    - *Role:* Text-to-speech API client.
    - *Configuration:* POST request to ElevenLabs TTS API (`/v1/text-to-speech/{voice_id}`) with generic Header Auth (`xi-api-key`), outputting binary audio data.
    - *Connections:* Input from `Voice note wanted?`; Output to `Build report`.
  - **Build report** (`n8n-nodes-base.code`):
    - *Role:* Presentation builder.
    - *Configuration:* Assembles responsive HTML documents for web form outputs and formatted markdown strings for Telegram payloads, embedding audio binary attachments if applicable.
    - *Connections:* Input from `Speak the note`, `Voice note wanted?`, and `What came in?` (Help/Report branches); Output to `Reply where they asked`.
  - **Reply where they asked** (`n8n-nodes-base.switch`):
    - *Role:* Delivery channel router.
    - *Configuration:* Routes payloads based on `channel` (`form` vs `telegram`) and `has_audio` status.
    - *Connections:* Input from `Build report`; Outputs connect to `Show the report`, `Telegram: send voice note`, and `Telegram: send report`.
  - **Show the report** (`n8n-nodes-base.form`):
    - *Role:* Web form completion response node.
    - *Configuration:* Responds to form submissions by rendering HTML output.
    - *Connections:* Input from `Reply where they asked`.
  - **Telegram: send voice note** (`n8n-nodes-base.telegram`):
    - *Role:* Telegram audio delivery node.
    - *Configuration:* Sends binary MP3 audio attachments using `sendAudio`.
    - *Connections:* Input from `Reply where they asked`; Output to `Telegram: send report`.
  - **Telegram: send report** (`n8n-nodes-base.telegram`):
    - *Role:* Telegram text delivery node.
    - *Configuration:* Sends formatted HTML message text using `sendMessage`.
    - *Connections:* Input from `Reply where they asked` and `Telegram: send voice note`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Note: About this template | `n8n-nodes-base.stickyNote` | Documentation and setup guide | None | None | ## Track your personal grocery inflation from receipts... *(See template canvas)* |
| Note: Intake | `n8n-nodes-base.stickyNote` | Documentation for intake block | None | None | ## 1. Receipts come in... |
| Note: Reading and matching | `n8n-nodes-base.stickyNote` | Documentation for OCR and matching block | None | None | ## 2. Reading and matching... |
| Note: The maths | `n8n-nodes-base.stickyNote` | Documentation for calculation block | None | None | ## 3. The economist's maths... |
| Note: Report | `n8n-nodes-base.stickyNote` | Documentation for reporting and TTS block | None | None | ## 4. The report... |
| Your receipts | `n8n-nodes-base.formTrigger` | Web form entry point | None | Settings | |
| Telegram receipts | `n8n-nodes-base.telegramTrigger` | Telegram bot entry point | None | Settings | |
| Settings | `n8n-nodes-base.set` | Define global workflow variables | Your receipts, Telegram receipts | Create receipt lines table | |
| Create receipt lines table | `n8n-nodes-base.dataTable` | Provision n8n Data Table | Settings | Load saved lines | |
| Load saved lines | `n8n-nodes-base.dataTable` | Load historical receipt records | Create receipt lines table | Gather receipts | |
| Gather receipts | `n8n-nodes-base.code` | Normalize inbound triggers & payload | Load saved lines | What came in? | |
| What came in? | `n8n-nodes-base.switch` | Route based on trigger type | Gather receipts | Read receipt photo, Find lines to match, Fetch official index, Build report | |
| Read receipt photo | `@n8n/n8n-nodes-langchain.chainLlm` | OCR extraction from receipt image | What came in? | Find lines to match | |
| Gemini (reads & matches) | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM backend for OCR and matching | None | Read receipt photo, Match new lines to products | |
| Receipt format | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforce JSON schema for OCR | Read receipt photo | Read receipt photo | |
| Find lines to match | `n8n-nodes-base.code` | Diff incoming lines against known keys | Read receipt photo, What came in? | Anything new to match? | |
| Anything new to match? | `n8n-nodes-base.switch` | Route based on presence of new lines | Find lines to match | Match new lines to products, Price matched lines, Build report | |
| Match new lines to products | `@n8n/n8n-nodes-langchain.agent` | Autonomous AI agent for categorization | Anything new to match? | Price matched lines | |
| Search saved products | `n8n-nodes-base.dataTableTool` | Data table tool for AI agent | Match new lines to products | Match new lines to products | |
| Matches format | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforce JSON schema for AI matches | Match new lines to products | Match new lines to products | |
| Price matched lines | `n8n-nodes-base.code` | Compute unit prices and totals | Match new lines to products, Anything new to match? | Save receipt lines | |
| Save receipt lines | `n8n-nodes-base.dataTable` | Upsert line items to Data Table | Price matched lines | Fetch official index | |
| Fetch official index | `n8n-nodes-base.httpRequest` | Fetch live US BLS CPI time series | Save receipt lines, What came in? | Calculate personal inflation | |
| Calculate personal inflation | `n8n-nodes-base.code` | Calculate weighted personal inflation index | Fetch official index | Write the economist's note | |
| Write the economist's note | `@n8n/n8n-nodes-langchain.chainLlm` | Generate qualitative insights and speech script | Calculate personal inflation | Voice note wanted? | |
| Gemini (writes) | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | LLM backend for narrative generation | None | Write the economist's note, Note format | |
| Note format | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforce JSON schema for economist note | Write the economist's note | Write the economist's note | |
| Voice note wanted? | `n8n-nodes-base.if` | Check if audio generation is required | Write the economist's note | Speak the note, Build report | |
| Speak the note | `n8n-nodes-base.httpRequest` | ElevenLabs Text-to-Speech API call | Voice note wanted? | Build report | |
| Build report | `n8n-nodes-base.code` | Build HTML pages and Telegram text layouts | Speak the note, Voice note wanted?, What came in? | Reply where they asked | |
| Reply where they asked | `n8n-nodes-base.switch` | Route reply to Web form or Telegram | Build report | Show the report, Telegram: send voice note, Telegram: send report | |
| Show the report | `n8n-nodes-base.form` | Render HTML completion page | Reply where they asked | None | |
| Telegram: send voice note | `n8n-nodes-base.telegram` | Send audio voice note via Telegram | Reply where they asked | Telegram: send report | |
| Telegram: send report | `n8n-nodes-base.telegram` | Send text report via Telegram | Reply where they asked, Telegram: send voice note | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Trigger Nodes:**
   - Add a **Form Trigger** node (`Your receipts`). Set the form title to `"Your Inflation Isn't Their Inflation"`, add a dropdown field (`"Where should the receipts come from?"` with options `"Try the sample household"` and `"Use the photos I upload below"`), a multi-file upload field accepting `.jpg, .jpeg, .png, .webp`, and a text input for `"Household name"`.
   - Add a **Telegram Trigger** node (`Telegram receipts`). Set updates to `message` with image download enabled.

2. **Add Settings & Database Initialization:**
   - Add a **Set** node (`Settings`). Configure manual assignments: `official_source` = `BLS`, `official_series_id` = `CUUR0000SAF11`, `official_label` = `Official US inflation: food at home`, `currency_symbol` = `$`, `narrator_voice` = a descriptive prompt string, `elevenlabs_voice_id` = your voice ID, `elevenlabs_model` = `eleven_multilingual_v2`.
   - Add a **Data Table** node (`Create receipt lines table`). Set resource to `table`, operation to `create`, table name to `receipt_economist_lines`, and define columns (`line_id`, `household_id`, `receipt_id`, `purchase_date`, `month`, `store`, `raw_text`, `item_key`, `label`, `category`, `quantity`, `line_total`, `unit_price`, `needs_review`) with `createIfNotExists` enabled.
   - Add a **Data Table** node (`Load saved lines`). Set resource to `row`, operation to `get`, and retrieve all rows from `receipt_economist_lines`.

3. **Add Intake Router & Switch:**
   - Add a **Code** node (`Gather receipts`). Paste the JavaScript payload normalization code handling sample household generation and multi-channel context mapping.
   - Add a **Switch** node (`What came in?`). Set 4 output routes checking `$json.kind` equal to `photo`, `receipt`, `report`, and `help`.

4. **Add OCR & Vision Parsing:**
   - Add a **Basic LLM Chain** node (`Read receipt photo`). Connect it to a **Google Gemini Chat Model** credential (`Gemini (reads & matches)` using `models/gemini-3.8-flash`). Configure prompt extraction for store name, date, and line items.
   - Attach a **Structured Output Parser** (`Receipt format`) enforcing the object schema with `store`, `purchase_date`, and `lines` array.

5. **Add Product Matching & Pricing Pipeline:**
   - Add a **Code** node (`Find lines to match`) to compile readable receipts and identify unmapped raw strings.
   - Add a **Switch** node (`Anything new to match?`) evaluating `unmatched_count`.
   - Add an **AI Agent** node (`Match new lines to products`). Provide system instructions for product normalization, connect it to the Gemini Chat Model, and attach the **Structured Output Parser** (`Matches format`).
   - Attach a **Data Table Tool** (`Search saved products`) to the AI agent configured to query `receipt_economist_lines` via `ilike` filters.
   - Add a **Code** node (`Price matched lines`) to compute unit prices and map final attributes.
   - Add a **Data Table** node (`Save receipt lines`). Set operation to `upsert` on `receipt_economist_lines` matching by `line_id`.

6. **Add Macroeconomic Calculation & BLS Request:**
   - Add an **HTTP Request** node (`Fetch official index`). Method: `POST`, URL: `https://api.bls.gov/publicAPI/v1/timeseries/data/`, sending a JSON body with the configured `official_series_id` and calculated year range.
   - Add a **Code** node (`Calculate personal inflation`) to compute Laspeyres-style weighted price indices and merge with official CPI data.

7. **Add Report Synthesis, TTS & Delivery:**
   - Add a **Basic LLM Chain** node (`Write the economist's note`) connected to the Gemini Chat Model and a **Structured Output Parser** (`Note format`) enforcing schema with `headline`, `verdict`, `drivers`, `tip`, `method_note`, and `voice_note`.
   - Add an **If** node (`Voice note wanted?`) checking reply style preferences and ElevenLabs credential presence.
   - Add an **HTTP Request** node (`Speak the note`). Method: `POST`, URL: `https://api.elevenlabs.io/v1/text-to-speech/{voice_id}?output_format=mp3_44100_64`, with HTTP Header Auth (`xi-api-key`).
   - Add a **Code** node (`Build report`) to format responsive HTML layouts and Telegram text payloads.
   - Add a **Switch** node (`Reply where they asked`) routing between web form and Telegram channels.
   - Add a **Form** node (`Show the report`) set to completion response with HTML output, and **Telegram** nodes (`Telegram: send voice note`, `Telegram: send report`) configured for audio and message dispatch.

8. **Establish Connections:**
   - Wire nodes strictly adhering to the dependency sequence outlined in Section 2 and Section 3. Ensure AI model and parser nodes bind correctly to their respective parent LangChain nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Creator Attribution | Built by Alicia Graham, economist and n8n Ambassador. |
| Google Gemini API Studio | Free API key generation for Gemini nodes: [Google AI Studio](https://aistudio.google.com) |
| US Bureau of Labor Statistics API | Public CPI Time Series data source (no API key required for v1 endpoints): [BLS Public Data API](https://www.bls.gov/developers/) |
| ElevenLabs Text-to-Speech API | Optional narration engine documentation: [ElevenLabs API Docs](https://elevenlabs.io/docs) |