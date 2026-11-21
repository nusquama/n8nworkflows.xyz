Build and review product video storyboards with forms and manual Cicadas handoff

https://n8nworkflows.xyz/workflows/build-and-review-product-video-storyboards-with-forms-and-manual-cicadas-handoff-20982


# Build and review product video storyboards with forms and manual Cicadas handoff

### 1. Workflow Overview

This workflow automates the collection and structuring of product video briefs. It takes user inputs via an n8n form, validates them against strict character and selection rules, generates a structured three-shot storyboard (hook, product detail, and closing frame), and provides a human-in-the-loop review interface. Upon approval, it renders copyable prompts and a Markdown handoff pack for manual video generation in external tools.

The logic is grouped into three main functional blocks:
- **1.1 Input Reception & Validation:** Collects raw product video brief details via an n8n Form Trigger and validates text lengths, required fields, and dropdown selections using a custom JavaScript Code node.
- **1.2 Storyboard Review & Routing:** Evaluates validation success using an If node. Invalid briefs display an error completion page; valid briefs present an interactive review form to approve or reject the draft plan.
- **1.3 Manual Handoff Generation:** Processes the review decision, rendering either a final completion page with per-shot copyable prompts, a Markdown handoff pack, and a direct link to Cicadas, or a revision notice.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Validation

##### Overview
This block initiates the process by exposing an interactive intake form to capture product video criteria. It then sanitizes and validates the submitted text lengths, formatting constraints, and selected options programmatically without calling external AI services.

##### Nodes Involved
- `Product video brief` (Form Trigger)
- `Prepare and validate brief` (Code)
- `Valid brief?` (If)

##### Node Details

- **Product video brief**
  - **Type and technical role:** `n8n-nodes-base.formTrigger` — Acts as the workflow entry point, hosting a web form to capture product attributes.
  - **Configuration choices:** Configured with response mode set to `lastNode`, path `cicadas-product-video-brief`, button label "Build draft storyboard", and attribution appending enabled. Defines 10 specific form fields (text, textarea, dropdowns) for attributes like product name, appearance, audience, benefit, context, CTA, restrictions, platform format, duration, and visual style.
  - **Key expressions or variables:** Outputs form fields as raw strings or option selections directly to subsequent nodes.
  - **Input and output connections:** Inputs: None (Trigger). Output: Connected to `Prepare and validate brief`.
  - **Version-specific requirements:** Uses type version `2.4`.
  - **Edge cases or potential failure types:** Unhandled HTML input (mitigated by downstream escaping) or missing mandatory fields (caught by native form validation).
  - **Sub-workflow reference:** None.

- **Prepare and validate brief**
  - **Type and technical role:** `n8n-nodes-base.code` — Executes a custom JavaScript function to normalize text, check character limits, validate allowed dropdown combinations, calculate shot timings, and build structured prompt objects and HTML review templates.
  - **Configuration choices:** Uses JavaScript to validate fields (`product`, `appearance`, `audience`, `benefit`, `context`, `cta`, `restrictions`) against defined length constraints and maps formats, styles, and durations.
  - **Key expressions or variables:** Consumes `$input.all()`. Returns a JSON object containing `valid` (boolean), `brief`, `shots`, `markdown`, and `reviewHtml`.
  - **Input and output connections:** Input: `Product video brief`. Output: Connected to `Valid brief?`.
  - **Version-specific requirements:** Uses type version `2`.
  - **Edge cases or potential failure types:** Type errors if non-string inputs bypass the form UI; safely handled via type coercion and fallback checks.
  - **Sub-workflow reference:** None.

- **Valid brief?**
  - **Type and technical role:** `n8n-nodes-base.if` — Routes the workflow branch based on whether the input validation passed.
  - **Configuration choices:** Evaluates condition version `2` with strict type validation, checking if `{{ $json.valid }}` equals `true`.
  - **Key expressions or variables:** `={{ $json.valid }}`
  - **Input and output connections:** Input: `Prepare and validate brief`. Outputs: True branch connects to `Review storyboard`; False branch connects to `Show validation feedback`.
  - **Version-specific requirements:** Uses type version `2.2`.
  - **Edge cases or potential failure types:** Evaluation failure if `valid` is undefined (defaults to false branch).
  - **Sub-workflow reference:** None.

---

#### 1.2 Storyboard Review & Routing

##### Overview
This block manages the conditional routing resulting from validation. If invalid, it displays an error page instructing the user to fix the input. If valid, it presents the generated storyboard within an interactive form for human review and approval.

##### Nodes Involved
- `Show validation feedback` (Form Completion)
- `Review storyboard` (Form)

##### Node Details

- **Show validation feedback**
  - **Type and technical role:** `n8n-nodes-base.form` — Renders a completion page detailing validation errors when a brief fails rule checks.
  - **Configuration choices:** Operation set to `completion`, response set via `showText` using the generated error HTML.
  - **Key expressions or variables:** `={{ $json.errorHtml }}`
  - **Input and output connections:** Input: `Valid brief?` (False branch). Output: None (Terminal node).
  - **Version-specific requirements:** Uses type version `2.4`.
  - **Edge cases or potential failure types:** None.
  - **Sub-workflow reference:** None.

- **Review storyboard**
  - **Type and technical role:** `n8n-nodes-base.form` — Presents an interactive review page containing the generated storyboard HTML and a decision dropdown.
  - **Configuration choices:** Operation set to `page`, using defined custom form fields with injected custom CSS (`pre { white-space: pre-wrap; ... }`). Includes an HTML display field and a dropdown field for the review decision.
  - **Key expressions or variables:** `={{ $json.reviewHtml }}`
  - **Input and output connections:** Input: `Valid brief?` (True branch). Output: Connected to `Build manual handoff`.
  - **Version-specific requirements:** Uses type version `2.4`.
  - **Edge cases or potential failure types:** User submitting without making a valid dropdown choice (mitigated by setting the field as required).
  - **Sub-workflow reference:** None.

---

#### 1.3 Manual Handoff Generation

##### Overview
This block processes the user's review decision. If approved, it compiles a styled HTML handoff page featuring copyable text areas for individual prompts and a complete Markdown project note pack, including a link to Cicadas. If revision is requested, it halts execution and shows a revision message.

##### Nodes Involved
- `Build manual handoff` (Code)
- `Show handoff or revision` (Form Completion)

##### Node Details

- **Build manual handoff**
  - **Type and technical role:** `n8n-nodes-base.code` — Executes a JavaScript function that evaluates the review decision, applies custom CSS styling, and constructs the final HTML response containing copyable prompts and Markdown documentation.
  - **Configuration choices:** Extracts the plan object from the upstream `Prepare and validate brief` node and the user decision from `Review storyboard`.
  - **Key expressions or variables:** `$('Prepare and validate brief').first().json`, `{{ $input.first().json.decision }}`
  - **Input and output connections:** Input: `Review storyboard`. Output: Connected to `Show handoff or revision`.
  - **Version-specific requirements:** Uses type version `2`.
  - **Edge cases or potential failure types:** Missing context if upstream nodes are skipped; handled by conditional checks returning a revision required message.
  - **Sub-workflow reference:** None.

- **Show handoff or revision**
  - **Type and technical role:** `n8n-nodes-base.form` — Renders the final HTML completion page containing copyable shot prompts, Markdown handoff notes, and external links.
  - **Configuration choices:** Operation set to `completion`, response mode set to `showText`.
  - **Key expressions or variables:** `={{ $json.html }}`
  - **Input and output connections:** Input: `Build manual handoff`. Output: None (Terminal node).
  - **Version-specific requirements:** Uses type version `2.4`.
  - **Edge cases or potential failure types:** Rendering errors if HTML payload is malformed.
  - **Sub-workflow reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Product video brief** | `n8n-nodes-base.formTrigger` | Collects product video brief metadata via web form | None (Trigger) | Prepare and validate brief | ## 1. Collect an approved brief<br>**No credentials or paid AI calls.**<br>Import into n8n, open Product video brief and test the complete workflow. Use the Production Form URL after publishing in your own instance.<br>Use non-sensitive inputs. Configure authentication if deploying a private team form; this template does not bundle credentials. |
| **Prepare and validate brief** | `n8n-nodes-base.code` | Validates brief inputs and builds storyboard prompts programmatically | Product video brief | Valid brief? | ## 2. Validate and review<br>Checks required text, length and allowed options. Builds three shots: hook, detail, closing. **Fixed planning rules, not AI analysis.**<br>The review form lets you approve or request revision. Invalid or declined briefs never show the generation handoff. Timings are edit guidance, not supported model durations. |
| **Valid brief?** | `n8n-nodes-base.if` | Routes workflow based on input validation results | Prepare and validate brief | Review storyboard, Show validation feedback | ## 2. Validate and review<br>Checks required text, length and allowed options. Builds three shots: hook, detail, closing. **Fixed planning rules, not AI analysis.**<br>The review form lets you approve or request revision. Invalid or declined briefs never show the generation handoff. Timings are edit guidance, not supported model durations. |
| **Show validation feedback** | `n8n-nodes-base.form` | Renders form completion page listing validation errors | Valid brief? | None | ## 2. Validate and review<br>Checks required text, length and allowed options. Builds three shots: hook, detail, closing. **Fixed planning rules, not AI analysis.**<br>The review form lets you approve or request revision. Invalid or declined briefs never show the generation handoff. Timings are edit guidance, not supported model durations. |
| **Review storyboard** | `n8n-nodes-base.form` | Displays storyboard plan and requests human review decision | Valid brief? | Build manual handoff | ## 2. Validate and review<br>Checks required text, length and allowed options. Builds three shots: hook, detail, closing. **Fixed planning rules, not AI analysis.**<br>The review form lets you approve or request revision. Invalid or declined briefs never show the generation handoff. Timings are edit guidance, not supported model durations. |
| **Build manual handoff** | `n8n-nodes-base.code` | Compiles copyable prompts, Markdown pack, and responsive HTML layout | Review storyboard | Show handoff or revision | ## 3. Copy prompts and generate manually<br>The ending page offers separate prompts and a complete Markdown pack. Open [Cicadas](https://cicadas.ai/) separately and check available tools, limits and pricing before generating.<br>**No API calls, auto-prefill, generation, download, assembly or publishing.** The brief is not sent to Cicadas. n8n hosting/execution fees may apply. Supplied product claims require human review. |
| **Show handoff or revision** | `n8n-nodes-base.form` | Displays final handoff page with copyable prompts or revision notice | Build manual handoff | None | ## 3. Copy prompts and generate manually<br>The ending page offers separate prompts and a complete Markdown pack. Open [Cicadas](https://cicadas.ai/) separately and check available tools, limits and pricing before generating.<br>**No API calls, auto-prefill, generation, download, assembly or publishing.** The brief is not sent to Cicadas. n8n hosting/execution fees may apply. Supplied product claims require human review. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Point:**
   - Add a **Form Trigger** node named `Product video brief`.
   - Set the path parameter to `cicadas-product-video-brief`, button label to `Build draft storyboard`, and enable attribution appending.
   - Set the form title to `Plan a product video` and form description to `Turn your approved product facts into a three-shot draft for manual generation in Cicadas. Rule-based preparation only: no AI calls, no API key and no automatic video generation. Use non-sensitive information. The next page lets you approve or revise the plan.`.
   - Add the following 10 fields to the form:
     1. `product` (Text, Label: `Product name`, Required: `true`)
     2. `appearance` (Textarea, Label: `Product appearance`, Required: `true`)
     3. `audience` (Text, Label: `Audience`, Required: `true`)
     4. `benefit` (Textarea, Label: `Approved benefit`, Required: `true`)
     5. `context` (Textarea, Label: `Scene and intended use`, Required: `true`)
     6. `cta` (Text, Label: `Call to action`, Required: `true`)
     7. `restrictions` (Textarea, Label: `Things to avoid`, Required: `false`)
     8. `platform` (Dropdown, Label: `Video format`, Options: `Vertical social (9:16)`, `Landscape video (16:9)`, `Square feed (1:1)`, Required: `true`)
     9. `duration` (Dropdown, Label: `Planned edit length`, Options: `15 seconds`, `20 seconds`, `30 seconds`, Required: `true`)
    10. `style` (Dropdown, Label: `Visual style`, Options: `Clean studio`, `Everyday lifestyle`, `Bold editorial`, Required: `true`)

2. **Add Validation Logic:**
   - Create a **Code** node named `Prepare and validate brief`.
   - Paste the validation and storyboard generation JavaScript logic that normalizes inputs, checks field length limits, validates dropdown choices, calculates shot cut timings, and returns an object containing `valid`, `brief`, `shots`, `markdown`, and `reviewHtml`.
   - Connect `Product video brief` output to `Prepare and validate brief`.

3. **Add Conditional Routing:**
   - Create an **If** node named `Valid brief?`.
   - Set condition version to `2` with strict type validation, evaluating `={{ $json.valid }}` equal to `true`.
   - Connect `Prepare and validate brief` output to `Valid brief?`.

4. **Add Error Handling Path:**
   - Create a **Form** node named `Show validation feedback` (Operation: `completion`, Response: `showText`).
   - Set the response text expression to `={{ $json.errorHtml }}`.
   - Connect the false branch output of `Valid brief?` to `Show validation feedback`.

5. **Add Review Path:**
   - Create a **Form** node named `Review storyboard` (Operation: `page`, Define Form: `fields`).
   - Set form title to `Review your storyboard`, button label to `Confirm review decision`, and form description to `Review the complete draft before preparing the copyable handoff. Approval here does not spend credits or generate a video.`.
   - Add an HTML field with value `={{ $json.reviewHtml }}` and a dropdown field named `decision` (Label: `Review decision`, Options: `Approve the draft and show the manual handoff`, `Revise the brief before generation`, Required: `true`).
   - Connect the true branch output of `Valid brief?` to `Review storyboard`.

6. **Add Handoff Generation Logic:**
   - Create a **Code** node named `Build manual handoff`.
   - Paste the JavaScript function that reads the plan from the upstream validation node and the review decision, rendering the styled HTML output containing copyable text areas and Markdown text.
   - Connect `Review storyboard` output to `Build manual handoff`.

7. **Add Final Completion Output:**
   - Create a **Form** node named `Show handoff or revision` (Operation: `completion`, Response: `showText`).
   - Set the response text expression to `={{ $json.html }}`.
   - Connect `Build manual handoff` output to `Show handoff or revision`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Manual handoff: open Cicadas and copy the approved prompts yourself. No API credentials required. | [Cicadas](https://cicadas.ai/) |