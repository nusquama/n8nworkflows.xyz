Localize text into multiple languages with Google Gemini, Gmail, and Sheets

https://n8nworkflows.xyz/workflows/localize-text-into-multiple-languages-with-google-gemini--gmail--and-sheets-17694


# Localize text into multiple languages with Google Gemini, Gmail, and Sheets

### 1. Workflow Overview

This workflow is designed to automate the text localization process across multiple target languages simultaneously. Its primary purpose is to take source text and a comma-separated list of languages via an n8n form, utilize Google Gemini to contextually localize the content (preserving placeholders and brand names), log each translation to Google Sheets, and finally deliver all translations consolidated into a single email via Gmail.

The architecture relies on the following logical functional blocks:
- **1.1 Input Reception & Fan-Out:** Captures the localization parameters through an interactive form trigger and transforms the comma-separated language input into individual granular execution items.
- **1.2 Batch Processing & AI Localization:** Iterates through the language items in controlled batches, invoking the Google Gemini language model to perform context-aware localizations.
- **1.3 Data Persistence & Notification:** Formats the completed translations, appends each record to a designated Google Sheet, aggregates the final dataset, and dispatches a summary email to the requester.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Fan-Out
- **Overview:** This block collects localization parameters from a web form and parses the single comma-separated language string into discrete workflow items to allow per-language processing.
- **Nodes Involved:** 
  - `Localization Form`
  - `One Item Per Language`
- **Node Details:**
  - **Localization Form**
    - *Type & Role:* `n8n-nodes-base.formTrigger` (v2.6) - Serves as the webhook-based entry point generating a user-facing submission form.
    - *Configuration Choices:* Configured with custom button label ("Translate"), a custom success message ("Thanks! Your translations are being generated..."), and four specific form fields: `Your email` (email, required), `Content type` (dropdown with options: Marketing copy, Product UI text, Email, Support reply, General), `Languages` (text field, required), and `Text` (textarea, required).
    - *Expressions:* None.
    - *Connections:* Input: None (Trigger). Output: `One Item Per Language`.
    - *Edge Cases/Failures:* Form submission failures due to network issues; missing required fields handled natively by the form validation.
  - **One Item Per Language**
    - *Type & Role:* `n8n-nodes-base.code` (v2) - Executes JavaScript to parse the comma-separated language list and split it into multiple items.
    - *Configuration Choices:* Mode: `runOnceAllItems` (implicit via script output mapping). Iterates over `f.Languages`, trims whitespace, filters out empty entries, and maps each language into an isolated JSON payload containing email, content type, language, and text.
    - *Expressions:* Uses JavaScript `String($input.first().json.Languages).split(',')`.
    - *Connections:* Input: `Localization Form`. Output: `Loop Languages`.
    - *Edge Cases/Failures:* Malformed string inputs without commas are handled gracefully as a single item; empty string inputs result in an empty item array, halting downstream execution.

#### Block 1.2: Batch Processing & AI Localization
- **Overview:** This block manages rate limits by looping through items in batches, leveraging Google Gemini to contextually translate and localize the text for each target language, and saving individual records to storage.
- **Nodes Involved:**
  - `Loop Languages`
  - `Translate with Gemini`
  - `Google Gemini Chat Model`
  - `Shape Translation`
  - `Save Translation`
- **Node Details:**
  - **Loop Languages**
    - *Type & Role:* `n8n-nodes-base.splitInBatches` (v3) - Controls iteration over the generated language items in manageable batch sizes.
    - *Configuration Choices:* Batch size set to `3`.
    - *Expressions:* None.
    - *Connections:* Input: `One Item Per Language`, `Save Translation` (loopback). Output: Output 0 (Done) -> `Build Email Body`; Output 1 (Loop) -> `Translate with Gemini`.
    - *Edge Cases/Failures:* Infinite loops are prevented by the built-in counter logic of the splitInBatches node.
  - **Translate with Gemini**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chainLlm` (v1.9) - Executes an LLM chain prompting the AI model to act as a professional localizer.
    - *Configuration Choices:* Prompt type set to define. Injected prompt instructs the model to translate content based on `{{ $json.contentType }}` and `{{ $json.language }}` while keeping placeholders like `{name}` or `%s` and brand names intact.
    - *Expressions:* Uses `{{ $json.contentType }}`, `{{ $json.language }}`, and `{{ $json.text }}`.
    - *Connections:* Input: `Loop Languages`. Output: `Shape Translation`. AI Model input connected from `Google Gemini Chat Model`.
    - *Edge Cases/Failures:* API rate-limiting errors or upstream credential expiration on the Gemini API.
  - **Google Gemini Chat Model**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (v1.1) - Sub-node providing the language model backend for the LLM chain.
    - *Configuration Choices:* Model name: `models/gemini-3.1-flash-lite`. Temperature: `0.3`. Credentials required: Google Gemini (PaLM) API account.
    - *Expressions:* None.
    - *Connections:* Output (`ai_languageModel`) -> `Translate with Gemini`.
    - *Edge Cases/Failures:* Invalid API credentials or deprecated model names.
  - **Shape Translation**
    - *Type & Role:* `n8n-nodes-base.code` (v2) - Formats the LLM output and correlates it back with the original loop iteration data.
    - *Configuration Choices:* Mode: `runOnceForEachItem`. Merges data from the `Loop Languages` node scope with the LLM text output.
    - *Expressions:* Uses `$('Loop Languages').item.json` and `$json.text`.
    - *Connections:* Input: `Translate with Gemini`. Output: `Save Translation`.
    - *Edge Cases/Failures:* Missing `.text` property in the LLM response is safely coerced using `String($json.text || '').trim()`.
  - **Save Translation**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (v4.7) - Appends localized rows to a Google Sheets spreadsheet.
    - *Configuration Choices:* Operation set to `append`. Document and Sheet configured to target the "Translations" sheet. Credentials required: Google Sheets OAuth2 account.
    - *Expressions:* None.
    - *Connections:* Input: `Shape Translation`. Output: `Loop Languages` (loopback).
    - *Edge Cases/Failures:* Sheet permission errors, missing target sheet/tab names, or API quota limitations.

#### Block 1.3: Data Persistence & Notification
- **Overview:** Compiles all processed translation records after the loop concludes and dispatches a consolidated plain-text summary email to the requestor.
- **Nodes Involved:**
  - `Build Email Body`
  - `Email All Translations`
- **Node Details:**
  - **Build Email Body**
    - *Type & Role:* `n8n-nodes-base.code` (v2) - Aggregates all items from the `Shape Translation` node into a single structured email payload.
    - *Configuration Choices:* Mode: `runOnceAllItems`. Maps all rows, extracts email and translation blocks, and formats them into a single string.
    - *Expressions:* Uses `$('Shape Translation').all()` and `$('Localization Form').first()`.
    - *Connections:* Input: `Loop Languages` (Output 0 / Done). Output: `Email All Translations`.
    - *Edge Cases/Failures:* Empty row arrays fall back to the initial form submission email address.
  - **Email All Translations**
    - *Type & Role:* `n8n-nodes-base.gmail` (v2.2) - Sends the consolidated translations via Gmail.
    - *Configuration Choices:* Email Type: `text`. Send to, message body, and subject fields populated dynamically. Credentials required: Gmail OAuth2 account.
    - *Expressions:* 
      - To: `={{ $json.email }}`
      - Subject: `={{ "Your translations (" + $json.count + " languages)" }}`
      - Message: `={{ $json.body }}`
    - *Connections:* Input: `Build Email Body`. Output: None (Terminal node).
    - *Edge Cases/Failures:* Invalid recipient email formats, OAuth2 token revocation, or Gmail API sending limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Overview Sticky** | `n8n-nodes-base.stickyNote` | Documentation placeholder outlining overall architecture, setup steps, and tips. | None | None | ## Localize text into many languages at once with Gemini<br><br>### How it works<br>Machine translation is often too literal for marketing or UI text. This template localizes properly and does it for every language in one submission. Paste your text in the built-in form and list the target languages, and a Code node fans the request out into one item per language. A Loop Over Items node then processes them in small batches so you stay under the rate limit, and for each language a Basic LLM Chain with Google Gemini translates the text as a localizer would - matching tone, adapting idioms and units, keeping placeholders like {name} intact and leaving brand names alone. Every translation is saved to a Google Sheet, and when the loop finishes all of them are combined into a single email so you get one tidy message instead of one per language.<br><br>### Setup<br>1. Connect Google Gemini (PaLM) API, Google Sheets and Gmail.<br>2. Point the Save Translation node at your spreadsheet (tab: Translations).<br>3. Open the form URL, paste text and list languages separated by commas.<br><br>### Customization tips<br>Translate an uploaded file, or push each language to its own sheet or CMS locale. |
| **S1** | `n8n-nodes-base.stickyNote` | Groups the form submission and item-splitting logic. | None | None | ## 1. Submit & fan out<br>The form takes text and a list of languages; a Code node creates one item per language. |
| **Localization Form** | `n8n-nodes-base.formTrigger` | Captures localization form input parameters. | None | One Item Per Language | ## 1. Submit & fan out<br>The form takes text and a list of languages; a Code node creates one item per language. |
| **One Item Per Language** | `n8n-nodes-base.code` | Parses comma-separated languages into individual items. | Localization Form | Loop Languages | ## 1. Submit & fan out<br>The form takes text and a list of languages; a Code node creates one item per language. |
| **S2** | `n8n-nodes-base.stickyNote` | Groups the batch loop, AI translation, formatting, and saving logic. | None | None | ## 2. Loop & translate (Basic LLM Chain)<br>Process the languages in batches to respect the rate limit. Gemini localizes each one and it is saved to Sheets. |
| **Loop Languages** | `n8n-nodes-base.splitInBatches` | Processes target language items in controlled batches. | One Item Per Language, Save Translation | Build Email Body, Translate with Gemini | ## 2. Loop & translate (Basic LLM Chain)<br>Process the languages in batches to respect the rate limit. Gemini localizes each one and it is saved to Sheets. |
| **Translate with Gemini** | `@n8n/n8n-nodes-langchain.chainLlm` | Performs contextual localization via LLM chain. | Loop Languages | Shape Translation | ## 2. Loop & translate (Basic LLM Chain)<br>Process the languages in batches to respect the rate limit. Gemini localizes each one and it is saved to Sheets. |
| **Google Gemini Chat Model** | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Supplies the Gemini LLM configuration and credentials. | None | Translate with Gemini | ## 2. Loop & translate (Basic LLM Chain)<br>Process the languages in batches to respect the rate limit. Gemini localizes each one and it is saved to Sheets. |
| **Shape Translation** | `n8n-nodes-base.code` | Formats the AI translation output and combines with original data. | Translate with Gemini | Save Translation | ## 2. Loop & translate (Basic LLM Chain)<br>Process the languages in batches to respect the rate limit. Gemini localizes each one and it is saved to Sheets. |
| **Save Translation** | `n8n-nodes-base.googleSheets` | Appends individual translations to Google Sheets. | Shape Translation | Loop Languages | ## 2. Loop & translate (Basic LLM Chain)<br>Process the languages in batches to respect the rate limit. Gemini localizes each one and it is saved to Sheets. |
| **S3** | `n8n-nodes-base.stickyNote` | Groups the email aggregation and notification logic. | None | None | ## 3. Combine & email<br>When the loop is done, all translations are combined into one email. |
| **Build Email Body** | `n8n-nodes-base.code` | Compiles all translations into a single summary payload. | Loop Languages | Email All Translations | ## 3. Combine & email<br>When the loop is done, all translations are combined into one email. |
| **Email All Translations** | `n8n-nodes-base.gmail` | Sends the final aggregated email via Gmail. | Build Email Body | None | ## 3. Combine & email<br>When the loop is done, all translations are combined into one email. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Input Form:**
   - Add a **Form Trigger** node named `Localization Form` (Type version 2.6).
   - Configure options: Set button label to `Translate`, form title to `Localize my text`, description to `Paste text and pick languages to get faithful, tone-matched translations by email.`
   - Add form fields:
     - Field 1: Type `email`, Label `Your email`, Required: true.
     - Field 2: Type `dropdown`, Label `Content type`, Options: `Marketing copy`, `Product UI text`, `Email`, `Support reply`, `General`, Required: true.
     - Field 3: Type `text`, Label `Languages`, Placeholder `e.g. Japanese, Spanish, German`, Required: true.
     - Field 4: Type `textarea`, Label `Text`, Placeholder `Paste the text to translate...`, Required: true.
   - Configure success response text: `Thanks! Your translations are being generated and will arrive by email shortly.`

2. **Add the Language Parser Node:**
   - Create a **Code** node named `One Item Per Language` (Type version 2).
   - Connect `Localization Form` main output to `One Item Per Language`.
   - Set mode to run once across all items, and insert the following JavaScript code:
     ```javascript
     const f = $input.first().json;
     const langs = String(f.Languages || '').split(',').map(function (s) { return s.trim(); }).filter(Boolean);
     return langs.map(function (l) { return { json: { email: f['Your email'], contentType: f['Content type'], language: l, text: f.Text } }; });
     ```

3. **Configure Batch Looping:**
   - Add a **Split In Batches** node named `Loop Languages` (Type version 3).
   - Connect `One Item Per Language` main output to `Loop Languages`.
   - Set Batch Size parameter to `3`.

4. **Setup the AI Chain and Model:**
   - Add a **Basic LLM Chain** node named `Translate with Gemini` (Type version 1.9).
   - Connect the loop iteration output (Output 1) of `Loop Languages` to `Translate with Gemini`.
   - Set Prompt Type to `Define` and paste the following text into the Text parameter:
     ```text
     You are a professional localizer, not a literal translator. Translate the {{ $json.contentType }} below into {{ $json.language }}. Match the tone and intent, adapt idioms and units naturally for that locale, keep any placeholders like {name} or %s unchanged, and do not translate brand names. Output only the translation.

     Text:
     {{ $json.text }}
     ```
   - Add a **Google Gemini Chat Model** node named `Google Gemini Chat Model` (Type version 1.1).
   - Configure model name to `models/gemini-3.1-flash-lite` and set temperature to `0.3`.
   - Configure credentials for Google Gemini (PaLM) API account.
   - Connect the `ai_languageModel` output of `Google Gemini Chat Model` to the corresponding input of `Translate with Gemini`.

5. **Format and Save Translations:**
   - Add a **Code** node named `Shape Translation` (Type version 2).
   - Connect `Translate with Gemini` main output to `Shape Translation`.
   - Set mode to `runOnceForEachItem` and insert the following JavaScript code:
     ```javascript
     const src = $('Loop Languages').item.json;
     return { email: src.email, contentType: src.contentType, language: src.language, original: src.text, translation: String($json.text || '').trim() };
     ```
   - Add a **Google Sheets** node named `Save Translation` (Type version 4.7).
   - Connect `Shape Translation` main output to `Save Translation`.
   - Configure operation to `append`, select your Google Sheets document, and select the sheet/tab named `Translations`.
   - Configure credentials for Google Sheets OAuth2 account.
   - Connect the main output of `Save Translation` back to the input of `Loop Languages` to form the batch iteration loop.

6. **Compile and Send Notification Email:**
   - Add a **Code** node named `Build Email Body` (Type version 2).
   - Connect the completion output (Output 0 / Done) of `Loop Languages` to `Build Email Body`.
   - Set mode to run once across all items, and insert the following JavaScript code:
     ```javascript
     const rows = $('Shape Translation').all().map(function (i) { return i.json; });
     const email = (rows[0] && rows[0].email) || $('Localization Form').first().json['Your email'];
     const blocks = rows.map(function (r) { return r.language + ':\n' + r.translation; }).join('\n\n');
     return [{ json: { email: email, count: rows.length, body: 'Here are your translations.\n\n' + blocks } }];
     ```
   - Add a **Gmail** node named `Email All Translations` (Type version 2.2).
   - Connect `Build Email Body` main output to `Email All Translations`.
   - Set Resource to `Message` and Operation to `Send`.
   - Configure parameters:
     - Send To: `={{ $json.email }}`
     - Subject: `={{ "Your translations (" + $json.count + " languages)" }}`
     - Message: `={{ $json.body }}`
     - Email Type: `text`
   - Configure credentials for Gmail OAuth2 account.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow built via n8n automated assistant | Internal builder metadata and schema definition |