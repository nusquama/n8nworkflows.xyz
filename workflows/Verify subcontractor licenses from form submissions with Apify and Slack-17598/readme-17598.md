Verify subcontractor licenses from form submissions with Apify and Slack

https://n8nworkflows.xyz/workflows/verify-subcontractor-licenses-from-form-submissions-with-apify-and-slack-17598


# Verify subcontractor licenses from form submissions with Apify and Slack

### 1. Workflow Overview

This workflow automates the collection, validation, and verification of subcontractor licensing data submitted via an onboarding form. Its primary purpose is to ensure that subcontractors possess active, valid licenses issued to their exact business names by relevant state licensing boards before they are reviewed by compliance personnel. 

The architecture implements a "fail-closed" security and compliance model: any incomplete submission, unsupported US state, or API lookup failure results in a manual review flag rather than an automated pass. 

The workflow is categorized into the following logical blocks:
- **1.1 Input Reception & Normalization:** Captures form submissions, normalizes inconsistent field namings, standardizes US state inputs from full names to two-letter codes, and validates mandatory fields.
- **1.2 Routing & Feasibility Check:** Evaluates whether the submission originates from a supported US state and maps it to the corresponding Apify actor configuration.
- **1.3 Licensing Lookup & Decision Engine:** Executes the targeted Apify actor to scrape state licensing boards, compares the official records against the subcontractor's claims, and outputs a strict verdict (`pass`, `review`, `fail`, or `manual-review`).
- **1.4 Audit Trail & Notification:** Formats a structured record of the verification event and conditionally dispatches the results to downstream audit targets and team communication channels.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization

##### Overview
This block initiates the process by hosting an interactive onboarding intake form, capturing submissions, and transforming raw inputs into a clean, predictable schema while verifying the presence of all required parameters.

##### Nodes Involved
- `Subcontractor submits the form`
- `Validate the submission`

##### Node Details

###### Subcontractor submits the form
- **Type and Technical Role:** `n8n-nodes-base.formTrigger` (v2.2) — Acts as the primary web-facing entry point, rendering an integrated HTML intake form.
- **Configuration Choices:** Configured with form title "Subcontractor onboarding", response mode set to output the last node results, and mandatory fields defined for Company, Contact name, Email, State, and License number.
- **Key Expressions or Variables:** N/A (Trigger node)
- **Input and Output Connections:** 
  - Inputs: None (Webhook-driven entry point)
  - Outputs: Connected to `Validate the submission`
- **Version-specific Requirements:** None.
- **Edge Cases or Potential Failure Types:** Users submitting malformed emails or leaving required fields blank directly on the form UI (prevented client-side by required field flags).

###### Validate the submission
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) — Normalizes varying key formats (e.g., `companyName` vs `businessName`), standardizes full US state names to 2-letter ISO codes using a comprehensive mapping dictionary, and evaluates completeness.
- **Configuration Choices:** Executes custom JavaScript designed to fail closed (`valid: false` if data is incomplete).
- **Key Expressions or Variables:** `$input.first().json`
- **Input and Output Connections:** 
  - Inputs: Connected from `Subcontractor submits the form`
  - Outputs: Connected to `Route by state`
- **Version-specific Requirements:** JavaScript execution environment enabled.
- **Edge Cases or Potential Failure Types:** Unrecognized state inputs default to an empty string, appending a descriptive error message to the `missing` array.

---

#### 2.2 Routing & Feasibility Check

##### Overview
This block checks whether the normalized state is supported by integrated Apify actors, mapping supported requests to their corresponding scrapers and input schemas, or routing unsupported entries directly to manual review queues.

##### Nodes Involved
- `Route by state`
- `Can we check it?`
- `Send it to a human`

##### Node Details

###### Route by state
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) — Evaluates the normalized state against an internal dictionary of 17 supported US states (`AL`, `AR`, `CA`, `CT`, `FL`, `MA`, `MI`, `MN`, `NC`, `NM`, `NV`, `OR`, `SC`, `TN`, `TX`, `VA`, `WA`) and injects the corresponding Apify Actor ID and input parameters.
- **Configuration Choices:** Maps state codes to specific Scrapebench actors and query fields (e.g., `businessName`, `lastName`, or `query`).
- **Key Expressions or Variables:** `$input.all()`
- **Input and Output Connections:** 
  - Inputs: Connected from `Validate the submission`
  - Outputs: Connected to `Can we check it?`
- **Version-specific Requirements:** None.
- **Edge Cases or Potential Failure Types:** Submissions from non-covered states bypass automated scraping and are flagged as `verdict: 'manual-review'`.

###### Can we check it?
- **Type and Technical Role:** `n8n-nodes-base.if` (v2) — Conditional gatekeeper that splits the workflow path depending on whether the state is covered and the payload is fully valid.
- **Configuration Choices:** Evaluates boolean expression `{{ $json.covered }}`.
- **Key Expressions or Variables:** `{{ $json.covered }}`
- **Input and Output Connections:** 
  - Inputs: Connected from `Route by state`
  - Outputs: 
    - True branch (Output Index 0) connected to `Look up the licence`
    - False branch (Output Index 1) connected to `Send it to a human`
- **Version-specific Requirements:** None.
- **Edge Cases or Potential Failure Types:** Boolean evaluation failure if upstream properties are missing (mitigated by prior validation steps).

###### Send it to a human
- **Type and Technical Role:** `n8n-nodes-base.set` (v3.4) — Standardizes uncheckable or incomplete submissions by assigning a formatted summary string while retaining all other payload fields.
- **Configuration Choices:** Assigns a `summary` property containing the reason for manual intervention.
- **Key Expressions or Variables:** `=Needs a human: {{ $json.reason }}`
- **Input and Output Connections:** 
  - Inputs: Connected from `Can we check it?` (False branch)
  - Outputs: Connected to `Write the audit row`
- **Version-specific Requirements:** None.
- **Edge Cases or Potential Failure Types:** None.

---

#### 2.3 Licensing Lookup & Decision Engine

##### Overview
This block performs live scraping of state licensing boards via Apify, cross-references official records with submitted subcontractor details, and applies strict compliance logic to determine the final verification verdict.

##### Nodes Involved
- `Look up the licence`
- `Decide the verdict`

##### Node Details

###### Look up the licence
- **Type and Technical Role:** `@apify/n8n-nodes-apify.apify` (v1) — Community integration node that invokes specific Apify actors to scrape state licensing boards and retrieve matching datasets.
- **Configuration Choices:** 
  - Resource: `Actors`
  - Operation: `Run actor and get dataset`
  - Actor ID: Dynamic evaluation from `{{ $json.actorId }}`
  - Custom Body: Dynamic JSON payload from `{{ $json.actorInput }}`
  - Error Handling: Configured with `onError: continueRegularOutput` to prevent workflow halts on API failures.
- **Key Expressions or Variables:** `{{ $json.actorId }}`, `{{ $json.actorInput }}`
- **Input and Output Connections:** 
  - Inputs: Connected from `Can we check it?` (True branch)
  - Outputs: Connected to `Decide the verdict`
- **Version-specific Requirements:** Requires the `@apify/n8n-nodes-apify` community node installed on a self-hosted n8n instance. Requires valid Apify API credentials.
- **Edge Cases or Potential Failure Types:** Apify rate limits, actor timeouts, invalid input parameters, or scraper selector failures. Mitigated by `continueRegularOutput` error handling.

###### Decide the verdict
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) — Compares the claimed license data against the returned state board dataset using alphanumeric normalization and regex patterns.
- **Configuration Choices:** Implements conditional compliance logic to assign one of four verdicts (`pass`, `review`, `fail`, `manual-review`):
  - Fails closed if the Apify lookup returns an execution error.
  - Returns `fail` if no matching records are found or if license status matches negative terms (`inactive`, `expired`, `suspended`, `revoked`, etc.).
  - Returns `review` (borrowed license pattern) if the license is active but registered to a different entity name than claimed.
  - Returns `pass` only if an active license matches both the number and the company name.
- **Key Expressions or Variables:** `$('Validate the submission').first().json`, `$input.all()`
- **Input and Output Connections:** 
  - Inputs: Connected from `Look up the licence`
  - Outputs: Connected to `Write the audit row`
- **Version-specific Requirements:** JavaScript execution environment enabled.
- **Edge Cases or Potential Failure Types:** Variations in business naming conventions requiring fuzzy string matching logic.

---

#### 2.4 Audit Trail & Notification

##### Overview
This block normalizes verification outputs into a structured audit schema suitable for database logging or sheet appending, and conditionally pushes notification summaries to communication platforms.

##### Nodes Involved
- `Write the audit row`
- `Post the verdict to Slack`

##### Node Details

###### Write the audit row
- **Type and Technical Role:** `n8n-nodes-base.code` (v2) — Formats the finalized verification object into a clean audit schema containing timestamps, raw claims, board evidence, and final verdicts.
- **Configuration Choices:** Maps properties from upstream evaluation steps into a uniform JSON array. Designed as a code node to allow immediate out-of-the-box execution without external sheet/database dependencies.
- **Key Expressions or Variables:** `$input.all()`
- **Input and Output Connections:** 
  - Inputs: Connected from `Decide the verdict` and `Send it to a human`
  - Outputs: Connected to `Post the verdict to Slack`
- **Version-specific Requirements:** None.
- **Edge Cases or Potential Failure Types:** Downstream integrations (e.g., Google Sheets, Airtable) must match the output keys if this node is swapped out.

###### Post the verdict to Slack
- **Type and Technical Role:** `n8n-nodes-base.slack` (v2.3) — Posts the verification verdict summary directly to a designated Slack channel.
- **Configuration Choices:** 
  - Resource: `channel`
  - Select: `channel`
  - Channel ID: `#onboarding`
  - Text: `={{ $json.summary }}`
  - Node Status: Disabled by default (`disabled: true`) to ensure template execution without pre-configured Slack credentials.
- **Key Expressions or Variables:** `={{ $json.summary }}`
- **Input and Output Connections:** 
  - Inputs: Connected from `Write the audit row`
  - Outputs: None (Terminal node)
- **Version-specific Requirements:** Requires valid Slack OAuth2 or Bot Token credentials and an active workspace connection.
- **Edge Cases or Potential Failure Types:** Invalid channel IDs or expired bot tokens will cause message delivery failure.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Subcontractor submits the form | `n8n-nodes-base.formTrigger` | Renders and captures onboarding form submissions. | None | Validate the submission | ## Verify a subcontractor's licence the moment they submit your onboarding form<br><br>![Workflow overview](https://scrapebench.dev/img/n8n/subcontractor-onboarding-gate/canvas.png)<br><br>> **Self-hosted only.** This template uses the `@apify/n8n-nodes-apify` community node, which cannot be installed on n8n Cloud.<br><br>### Who's it for<br>General contractors onboarding subs, property managers approving vendors, marketplaces admitting trade accounts, franchise ops teams.<br><br>The problem it fixes is not that nobody checks licences - it is that the check is done by hand by whoever happens to process the form. It is inconsistent, it gets skipped under load, and it leaves no record of *when* it was checked or what the board said at the time. When it matters, months later, there is nothing to point at.<br><br>### What it does<br>A subcontractor submits the built-in n8n form. Before anyone opens the submission, the claimed licence number is checked against that state's licensing board and the submission comes back with one of four verdicts:<br><br>- `pass` - an active licence exists and it is held by the company that claimed it<br>- `review` - the licence is active but registered to a **different** name. Borrowed licence numbers are a real pattern, so this never auto-passes<br>- `fail` - the licence is expired, revoked, suspended, or there is no record of it at all<br>- `manual-review` - the form was incomplete, the state is not covered, or the lookup itself failed<br><br>**The gate fails closed.** Any error, timeout or ambiguity becomes `manual-review`, never a pass. A compliance tool that guesses in your favour when it breaks is a liability, not a control.<br><br>Every submission also produces an audit row - what was claimed, what the board held, the verdict, and the timestamp.<br><br>Covers AL, AR, CA, CT, FL, MA, MI, MN, NC, NM, NV, OR, SC, TN, TX, VA and WA. Submissions from other states are labelled `manual-review` and passed to a person - never rejected, and never quietly approved.<br><br>### How to set up<br>1. Install the `@apify/n8n-nodes-apify` community node.<br>2. Add your Apify API key as an Apify credential on the **Look up the licence** node.<br>3. Activate the workflow to get the production form URL, and use that as your intake form. Swap the trigger for a Webhook node if you already collect submissions in Typeform, Jotform or your own site.<br>4. Replace **Write the audit row** with Google Sheets -> Append row, Airtable, or a database insert. It ships as a Code node so the template runs on import without a credential.<br>5. Add a Slack credential to **Post the verdict to Slack**, pick a channel, and enable the node.<br><br>### Requirements<br>- An Apify account and API token (about $0.004 per check)<br>- Self-hosted n8n (community node)<br>- Optional: Slack, and a sheet or table for the audit trail<br><br>### How to customize<br>- **Different intake form:** swap the Form Trigger for a Webhook node. The validator accepts several field spellings and a full state name as well as a 2-letter code.<br>- **Stricter matching:** `review` currently covers any name that doesn't match. Tighten or loosen the comparison in **Decide the verdict**.<br>- **Add a state:** one row in the `ACTORS` map in **Route by state**.<br>- **Route the verdicts:** add a Switch after **Decide the verdict** to send `pass` straight to your CRM and hold the rest.<br><br>## 1. The intake form<br>n8n's own Form Trigger, so the template is self-contained - no third-party form account needed.<br><br>Activate the workflow to get the public form URL.<br><br>Already collecting submissions elsewhere? Swap this for a **Webhook** node. The next node accepts<br>several field spellings, so most existing payloads work unchanged. |
| Validate the submission | `n8n-nodes-base.code` | Normalizes field names, maps state names to codes, and validates completeness. | Subcontractor submits the form | Route by state | ## Verify a subcontractor's licence the moment they submit your onboarding form<br><br>![Workflow overview](https://scrapebench.dev/img/n8n/subcontractor-onboarding-gate/canvas.png)<br><br>> **Self-hosted only.** This template uses the `@apify/n8n-nodes-apify` community node, which cannot be installed on n8n Cloud.<br><br>### Who's it for<br>General contractors onboarding subs, property managers approving vendors, marketplaces admitting trade accounts, franchise ops teams.<br><br>The problem it fixes is not that nobody checks licences - it is that the check is done by hand by whoever happens to process the form. It is inconsistent, it gets skipped under load, and it leaves no record of *when* it was checked or what the board said at the time. When it matters, months later, there is nothing to point at.<br><br>### What it does<br>A subcontractor submits the built-in n8n form. Before anyone opens the submission, the claimed licence number is checked against that state's licensing board and the submission comes back with one of four verdicts:<br><br>- `pass` - an active licence exists and it is held by the company that claimed it<br>- `review` - the licence is active but registered to a **different** name. Borrowed licence numbers are a real pattern, so this never auto-passes<br>- `fail` - the licence is expired, revoked, suspended, or there is no record of it at all<br>- `manual-review` - the form was incomplete, the state is not covered, or the lookup itself failed<br><br>**The gate fails closed.** Any error, timeout or ambiguity becomes `manual-review`, never a pass. A compliance tool that guesses in your favour when it breaks is a liability, not a control.<br><br>Every submission also produces an audit row - what was claimed, what the board held, the verdict, and the timestamp.<br><br>Covers AL, AR, CA, CT, FL, MA, MI, MN, NC, NM, NV, OR, SC, TN, TX, VA and WA. Submissions from other states are labelled `manual-review` and passed to a person - never rejected, and never quietly approved.<br><br>### How to set up<br>1. Install the `@apify/n8n-nodes-apify` community node.<br>2. Add your Apify API key as an Apify credential on the **Look up the licence** node.<br>3. Activate the workflow to get the production form URL, and use that as your intake form. Swap the trigger for a Webhook node if you already collect submissions in Typeform, Jotform or your own site.<br>4. Replace **Write the audit row** with Google Sheets -> Append row, Airtable, or a database insert. It ships as a Code node so the template runs on import without a credential.<br>5. Add a Slack credential to **Post the verdict to Slack**, pick a channel, and enable the node.<br><br>### Requirements<br>- An Apify account and API token (about $0.004 per check)<br>- Self-hosted n8n (community node)<br>- Optional: Slack, and a sheet or table for the audit trail<br><br>### How to customize<br>- **Different intake form:** swap the Form Trigger for a Webhook node. The validator accepts several field spellings and a full state name as well as a 2-letter code.<br>- **Stricter matching:** `review` currently covers any name that doesn't match. Tighten or loosen the comparison in **Decide the verdict**.<br>- **Add a state:** one row in the `ACTORS` map in **Route by state**.<br>- **Route the verdicts:** add a Switch after **Decide the verdict** to send `pass` straight to your CRM and hold the rest.<br><br>## 2. Normalise it<br>People type \"California\" into a box that wants \"CA\", and forms disagree about whether the field is<br>`license_number` or `licenseNumber`.<br><br>This node accepts both, and marks anything missing a required field as `manual-review` rather than<br>guessing at it. |
| Route by state | `n8n-nodes-base.code` | Maps supported states to specific Apify actors and query inputs. | Validate the submission | Can we check it? | ## Verify a subcontractor's licence the moment they submit your onboarding form<br><br>![Workflow overview](https://scrapebench.dev/img/n8n/subcontractor-onboarding-gate/canvas.png)<br><br>> **Self-hosted only.** This template uses the `@apify/n8n-nodes-apify` community node, which cannot be installed on n8n Cloud.<br><br>### Who's it for<br>General contractors onboarding subs, property managers approving vendors, marketplaces admitting trade accounts, franchise ops teams.<br><br>The problem it fixes is not that nobody checks licences - it is that the check is done by hand by whoever happens to process the form. It is inconsistent, it gets skipped under load, and it leaves no record of *when* it was checked or what the board said at the time. When it matters, months later, there is nothing to point at.<br><br>### What it does<br>A subcontractor submits the built-in n8n form. Before anyone opens the submission, the claimed licence number is checked against that state's licensing board and the submission comes back with one of four verdicts:<br><br>- `pass` - an active licence exists and it is held by the company that claimed it<br>- `review` - the licence is active but registered to a **different** name. Borrowed licence numbers are a real pattern, so this never auto-passes<br>- `fail` - the licence is expired, revoked, suspended, or there is no record of it at all<br>- `manual-review` - the form was incomplete, the state is not covered, or the lookup itself failed<br><br>**The gate fails closed.** Any error, timeout or ambiguity becomes `manual-review`, never a pass. A compliance tool that guesses in your favour when it breaks is a liability, not a control.<br><br>Every submission also produces an audit row - what was claimed, what the board held, the verdict, and the timestamp.<br><br>Covers AL, AR, CA, CT, FL, MA, MI, MN, NC, NM, NV, OR, SC, TN, TX, VA and WA. Submissions from other states are labelled `manual-review` and passed to a person - never rejected, and never quietly approved.<br><br>### How to set up<br>1. Install the `@apify/n8n-nodes-apify` community node.<br>2. Add your Apify API key as an Apify credential on the **Look up the licence** node.<br>3. Activate the workflow to get the production form URL, and use that as your intake form. Swap the trigger for a Webhook node if you already collect submissions in Typeform, Jotform or your own site.<br>4. Replace **Write the audit row** with Google Sheets -> Append row, Airtable, or a database insert. It ships as a Code node so the template runs on import without a credential.<br>5. Add a Slack credential to **Post the verdict to Slack**, pick a channel, and enable the node.<br><br>### Requirements<br>- An Apify account and API token (about $0.004 per check)<br>- Self-hosted n8n (community node)<br>- Optional: Slack, and a sheet or table for the audit trail<br><br>### How to customize<br>- **Different intake form:** swap the Form Trigger for a Webhook node. The validator accepts several field spellings and a full state name as well as a 2-letter code.<br>- **Stricter matching:** `review` currently covers any name that doesn't match. Tighten or loosen the comparison in **Decide the verdict**.<br>- **Add a state:** one row in the `ACTORS` map in **Route by state**.<br>- **Route the verdicts:** add a Switch after **Decide the verdict** to send `pass` straight to your CRM and hold the rest. |
| Can we check it? | `n8n-nodes-base.if` | Evaluates if the submission is covered and valid. | Route by state | Look up the licence, Send it to a human | ## Verify a subcontractor's licence the moment they submit your onboarding form<br><br>![Workflow overview](https://scrapebench.dev/img/n8n/subcontractor-onboarding-gate/canvas.png)<br><br>> **Self-hosted only.** This template uses the `@apify/n8n-nodes-apify` community node, which cannot be installed on n8n Cloud.<br><br>### Who's it for<br>General contractors onboarding subs, property managers approving vendors, marketplaces admitting trade accounts, franchise ops teams.<br><br>The problem it fixes is not that nobody checks licences - it is that the check is done by hand by whoever happens to process the form. It is inconsistent, it gets skipped under load, and it leaves no record of *when* it was checked or what the board said at the time. When it matters, months later, there is nothing to point at.<br><br>### What it does<br>A subcontractor submits the built-in n8n form. Before anyone opens the submission, the claimed licence number is checked against that state's licensing board and the submission comes back with one of four verdicts:<br><br>- `pass` - an active licence exists and it is held by the company that claimed it<br>- `review` - the licence is active but registered to a **different** name. Borrowed licence numbers are a real pattern, so this never auto-passes<br>- `fail` - the licence is expired, revoked, suspended, or there is no record of it at all<br>- `manual-review` - the form was incomplete, the state is not covered, or the lookup itself failed<br><br>**The gate fails closed.** Any error, timeout or ambiguity becomes `manual-review`, never a pass. A compliance tool that guesses in your favour when it breaks is a liability, not a control.<br><br>Every submission also produces an audit row - what was claimed, what the board held, the verdict, and the timestamp.<br><br>Covers AL, AR, CA, CT, FL, MA, MI, MN, NC, NM, NV, OR, SC, TN, TX, VA and WA. Submissions from other states are labelled `manual-review` and passed to a person - never rejected, and never quietly approved.<br><br>### How to set up<br>1. Install the `@apify/n8n-nodes-apify` community node.<br>2. Add your Apify API key as an Apify credential on the **Look up the licence** node.<br>3. Activate the workflow to get the production form URL, and use that as your intake form. Swap the trigger for a Webhook node if you already collect submissions in Typeform, Jotform or your own site.<br>4. Replace **Write the audit row** with Google Sheets -> Append row, Airtable, or a database insert. It ships as a Code node so the template runs on import without a credential.<br>5. Add a Slack credential to **Post the verdict to Slack**, pick a channel, and enable the node.<br><br>### Requirements<br>- An Apify account and API token (about $0.004 per check)<br>- Self-hosted n8n (community node)<br>- Optional: Slack, and a sheet or table for the audit trail<br><br>### How to customize<br>- **Different intake form:** swap the Form Trigger for a Webhook node. The validator accepts several field spellings and a full state name as well as a 2-letter code.<br>- **Stricter matching:** `review` currently covers any name that doesn't match. Tighten or loosen the comparison in **Decide the verdict**.<br>- **Add a state:** one row in the `ACTORS` map in **Route by state**.<br>- **Route the verdicts:** add a Switch after **Decide the verdict** to send `pass` straight to your CRM and hold the rest. |
| Look up the licence | `@apify/n8n-nodes-apify.apify` | Executes state board scrapers via Apify. | Can we check it? | Decide the verdict | ## Verify a subcontractor's licence the moment they submit your onboarding form<br><br>![Workflow overview](https://scrapebench.dev/img/n8n/subcontractor-onboarding-gate/canvas.png)<br><br>> **Self-hosted only.** This template uses the `@apify/n8n-nodes-apify` community node, which cannot be installed on n8n Cloud.<br><br>### Who's it for<br>General contractors onboarding subs, property managers approving vendors, marketplaces admitting trade accounts, franchise ops teams.<br><br>The problem it fixes is not that nobody checks licences - it is that the check is done by hand by whoever happens to process the form. It is inconsistent, it gets skipped under load, and it leaves no record of *when* it was checked or what the board said at the time. When it matters, months later, there is nothing to point at.<br><br>### What it does<br>A subcontractor submits the built-in n8n form. Before anyone opens the submission, the claimed licence number is checked against that state's licensing board and the submission comes back with one of four verdicts:<br><br>- `pass` - an active licence exists and it is held by the company that claimed it<br>- `review` - the licence is active but registered to a **different** name. Borrowed licence numbers are a real pattern, so this never auto-passes<br>- `fail` - the licence is expired, revoked, suspended, or there is no record of it at all<br>- `manual-review` - the form was incomplete, the state is not covered, or the lookup itself failed<br><br>**The gate fails closed.** Any error, timeout or ambiguity becomes `manual-review`, never a pass. A compliance tool that guesses in your favour when it breaks is a liability, not a control.<br><br>Every submission also produces an audit row - what was claimed, what the board held, the verdict, and the timestamp.<br><br>Covers AL, AR, CA, CT, FL, MA, MI, MN, NC, NM, NV, OR, SC, TN, TX, VA and WA. Submissions from other states are labelled `manual-review` and passed to a person - never rejected, and never quietly approved.<br><br>### How to set up<br>1. Install the `@apify/n8n-nodes-apify` community node.<br>2. Add your Apify API key as an Apify credential on the **Look up the licence** node.<br>3. Activate the workflow to get the production form URL, and use that as your intake form. Swap the trigger for a Webhook node if you already collect submissions in Typeform, Jotform or your own site.<br>4. Replace **Write the audit row** with Google Sheets -> Append row, Airtable, or a database insert. It ships as a Code node so the template runs on import without a credential.<br>5. Add a Slack credential to **Post the verdict to Slack**, pick a channel, and enable the node.<br><br>### Requirements<br>- An Apify account and API token (about $0.004 per check)<br>- Self-hosted n8n (community node)<br>- Optional: Slack, and a sheet or table for the audit trail<br><br>### How to customize<br>- **Different intake form:** swap the Form Trigger for a Webhook node. The validator accepts several field spellings and a full state name as well as a 2-letter code.<br>- **Stricter matching:** `review` currently covers any name that doesn't match. Tighten or loosen the comparison in **Decide the verdict**.<br>- **Add a state:** one row in the `ACTORS` map in **Route by state**.<br>- **Route the verdicts:** add a Switch after **Decide the verdict** to send `pass` straight to your CRM and hold the rest. |
| Send it to a human | `n8n-nodes-base.set` | Assigns summary text for uncheckable or unsupported states. | Can we check it? | Write the audit row | ## Verify a subcontractor's licence the moment they submit your onboarding form<br><br>![Workflow overview](https://scrapebench.dev/img/n8n/subcontractor-onboarding-gate/canvas.png)<br><br>> **Self-hosted only.** This template uses the `@apify/n8n-nodes-apify` community node, which cannot be installed on n8n Cloud.<br><br>### Who's it for<br>General contractors onboarding subs, property managers approving vendors, marketplaces admitting trade accounts, franchise ops teams.<br><br>The problem it fixes is not that nobody checks licences - it is that the check is done by hand by whoever happens to process the form. It is inconsistent, it gets skipped under load, and it leaves no record of *when* it was checked or what the board said at the time. When it matters, months later, there is nothing to point at.<br><br>### What it does<br>A subcontractor submits the built-in n8n form. Before anyone opens the submission, the claimed licence number is checked against that state's licensing board and the submission comes back with one of four verdicts:<br><br>- `pass` - an active licence exists and it is held by the company that claimed it<br>- `review` - the licence is active but registered to a **different** name. Borrowed licence numbers are a real pattern, so this never auto-passes<br>- `fail` - the licence is expired, revoked, suspended, or there is no record of it at all<br>- `manual-review` - the form was incomplete, the state is not covered, or the lookup itself failed<br><br>**The gate fails closed.** Any error, timeout or ambiguity becomes `manual-review`, never a pass. A compliance tool that guesses in your favour when it breaks is a liability, not a control.<br><br>Every submission also produces an audit row - what was claimed, what the board held, the verdict, and the timestamp.<br><br>Covers AL, AR, CA, CT, FL, MA, MI, MN, NC, NM, NV, OR, SC, TN, TX, VA and WA. Submissions from other states are labelled `manual-review` and passed to a person - never rejected, and never quietly approved.<br><br>### How to set up<br>1. Install the `@apify/n8n-nodes-apify` community node.<br>2. Add your Apify API key as an Apify credential on the **Look up the licence** node.<br>3. Activate the workflow to get the production form URL, and use that as your intake form. Swap the trigger for a Webhook node if you already collect submissions in Typeform, Jotform or your own site.<br>4. Replace **Write the audit row** with Google Sheets -> Append row, Airtable, or a database insert. It ships as a Code node so the template runs on import without a credential.<br>5. Add a Slack credential to **Post the verdict to Slack**, pick a channel, and enable the node.<br><br>### Requirements<br>- An Apify account and API token (about $0.004 per check)<br>- Self-hosted n8n (community node)<br>- Optional: Slack, and a sheet or table for the audit trail<br><br>### How to customize<br>- **Different intake form:** swap the Form Trigger for a Webhook node. The validator accepts several field spellings and a full state name as well as a 2-letter code.<br>- **Stricter matching:** `review` currently covers any name that doesn't match. Tighten or loosen the comparison in **Decide the verdict**.<br>- **Add a state:** one row in the `ACTORS` map in **Route by state**.<br>- **Route the verdicts:** add a Switch after **Decide the verdict** to send `pass` straight to your CRM and hold the rest.<br><br>## 3. Anything we cannot check goes to a person<br>Not one of the 17 covered states, or an incomplete submission.<br><br>Never auto-rejected and never auto-approved. \"We didn't check\" is its own answer. |
| Decide the verdict | `n8n-nodes-base.code` | Compares claimed subcontractor data against scraped state board results. | Look up the licence | Write the audit row | ## Verify a subcontractor's licence the moment they submit your onboarding form<br><br>![Workflow overview](https://scrapebench.dev/img/n8n/subcontractor-onboarding-gate/canvas.png)<br><br>> **Self-hosted only.** This template uses the `@apify/n8n-nodes-apify` community node, which cannot be installed on n8n Cloud.<br><br>### Who's it for<br>General contractors onboarding subs, property managers approving vendors, marketplaces admitting trade accounts, franchise ops teams.<br><br>The problem it fixes is not that nobody checks licences - it is that the check is done by hand by whoever happens to process the form. It is inconsistent, it gets skipped under load, and it leaves no record of *when* it was checked or what the board said at the time. When it matters, months later, there is nothing to point at.<br><br>### What it does<br>A subcontractor submits the built-in n8n form. Before anyone opens the submission, the claimed licence number is checked against that state's licensing board and the submission comes back with one of four verdicts:<br><br>- `pass` - an active licence exists and it is held by the company that claimed it<br>- `review` - the licence is active but registered to a **different** name. Borrowed licence numbers are a real pattern, so this never auto-passes<br>- `fail` - the licence is expired, revoked, suspended, or there is no record of it at all<br>- `manual-review` - the form was incomplete, the state is not covered, or the lookup itself failed<br><br>**The gate fails closed.** Any error, timeout or ambiguity becomes `manual-review`, never a pass. A compliance tool that guesses in your favour when it breaks is a liability, not a control.<br><br>Every submission also produces an audit row - what was claimed, what the board held, the verdict, and the timestamp.<br><br>Covers AL, AR, CA, CT, FL, MA, MI, MN, NC, NM, NV, OR, SC, TN, TX, VA and WA. Submissions from other states are labelled `manual-review` and passed to a person - never rejected, and never quietly approved.<br><br>### How to set up<br>1. Install the `@apify/n8n-nodes-apify` community node.<br>2. Add your Apify API key as an Apify credential on the **Look up the licence** node.<br>3. Activate the workflow to get the production form URL, and use that as your intake form. Swap the trigger for a Webhook node if you already collect submissions in Typeform, Jotform or your own site.<br>4. Replace **Write the audit row** with Google Sheets -> Append row, Airtable, or a database insert. It ships as a Code node so the template runs on import without a credential.<br>5. Add a Slack credential to **Post the verdict to Slack**, pick a channel, and enable the node.<br><br>### Requirements<br>- An Apify account and API token (about $0.004 per check)<br>- Self-hosted n8n (community node)<br>- Optional: Slack, and a sheet or table for the audit trail<br><br>### How to customize<br>- **Different intake form:** swap the Form Trigger for a Webhook node. The validator accepts several field spellings and a full state name as well as a 2-letter code.<br>- **Stricter matching:** `review` currently covers any name that doesn't match. Tighten or loosen the comparison in **Decide the verdict**.<br>- **Add a state:** one row in the `ACTORS` map in **Route by state**.<br>- **Route the verdicts:** add a Switch after **Decide the verdict** to send `pass` straight to your CRM and hold the rest.<br><br>## 4. Claimed vs found<br>The submission claims a licence number. This compares it against what the board actually holds.<br><br>`review` is the verdict that earns its keep: an **active** licence registered to a different<br>company is what a borrowed licence number looks like. Passing it because the licence is valid is<br>exactly the mistake this template exists to prevent.<br><br>If the lookup itself fails, the verdict is `manual-review` - never a pass. **The gate fails<br>closed.** |
| Write the audit row | `n8n-nodes-base.code` | Structures verification results into an audit log format. | Decide the verdict, Send it to a human | Post the verdict to Slack | ## Verify a subcontractor's licence the moment they submit your onboarding form<br><br>![Workflow overview](https://scrapebench.dev/img/n8n/subcontractor-onboarding-gate/canvas.png)<br><br>> **Self-hosted only.** This template uses the `@apify/n8n-nodes-apify` community node, which cannot be installed on n8n Cloud.<br><br>### Who's it for<br>General contractors onboarding subs, property managers approving vendors, marketplaces admitting trade accounts, franchise ops teams.<br><br>The problem it fixes is not that nobody checks licences - it is that the check is done by hand by whoever happens to process the form. It is inconsistent, it gets skipped under load, and it leaves no record of *when* it was checked or what the board said at the time. When it matters, months later, there is nothing to point at.<br><br>### What it does<br>A subcontractor submits the built-in n8n form. Before anyone opens the submission, the claimed licence number is checked against that state's licensing board and the submission comes back with one of four verdicts:<br><br>- `pass` - an active licence exists and it is held by the company that claimed it<br>- `review` - the licence is active but registered to a **different** name. Borrowed licence numbers are a real pattern, so this never auto-passes<br>- `fail` - the licence is expired, revoked, suspended, or there is no record of it at all<br>- `manual-review` - the form was incomplete, the state is not covered, or the lookup itself failed<br><br>**The gate fails closed.** Any error, timeout or ambiguity becomes `manual-review`, never a pass. A compliance tool that guesses in your favour when it breaks is a liability, not a control.<br><br>Every submission also produces an audit row - what was claimed, what the board held, the verdict, and the timestamp.<br><br>Covers AL, AR, CA, CT, FL, MA, MI, MN, NC, NM, NV, OR, SC, TN, TX, VA and WA. Submissions from other states are labelled `manual-review` and passed to a person - never rejected, and never quietly approved.<br><br>### How to set up<br>1. Install the `@apify/n8n-nodes-apify` community node.<br>2. Add your Apify API key as an Apify credential on the **Look up the licence** node.<br>3. Activate the workflow to get the production form URL, and use that as your intake form. Swap the trigger for a Webhook node if you already collect submissions in Typeform, Jotform or your own site.<br>4. Replace **Write the audit row** with Google Sheets -> Append row, Airtable, or a database insert. It ships as a Code node so the template runs on import without a credential.<br>5. Add a Slack credential to **Post the verdict to Slack**, pick a channel, and enable the node.<br><br>### Requirements<br>- An Apify account and API token (about $0.004 per check)<br>- Self-hosted n8n (community node)<br>- Optional: Slack, and a sheet or table for the audit trail<br><br>### How to customize<br>- **Different intake form:** swap the Form Trigger for a Webhook node. The validator accepts several field spellings and a full state name as well as a 2-letter code.<br>- **Stricter matching:** `review` currently covers any name that doesn't match. Tighten or loosen the comparison in **Decide the verdict**.<br>- **Add a state:** one row in the `ACTORS` map in **Route by state**.<br>- **Route the verdicts:** add a Switch after **Decide the verdict** to send `pass` straight to your CRM and hold the rest.<br><br>## 5. The audit trail<br>Claimed vs found vs verdict, with a timestamp and the board's own URL.<br><br>Swap for Google Sheets, Airtable or a database insert. Months later this is the thing you point<br>at, not the decision on its own. |
| Post the verdict to Slack | `n8n-nodes-base.slack` | Posts verification verdict summaries to Slack. | Write the audit row | None | ## Verify a subcontractor's licence the moment they submit your onboarding form<br><br>![Workflow overview](https://scrapebench.dev/img/n8n/subcontractor-onboarding-gate/canvas.png)<br><br>> **Self-hosted only.** This template uses the `@apify/n8n-nodes-apify` community node, which cannot be installed on n8n Cloud.<br><br>### Who's it for<br>General contractors onboarding subs, property managers approving vendors, marketplaces admitting trade accounts, franchise ops teams.<br><br>The problem it fixes is not that nobody checks licences - it is that the check is done by hand by whoever happens to process the form. It is inconsistent, it gets skipped under load, and it leaves no record of *when* it was checked or what the board said at the time. When it matters, months later, there is nothing to point at.<br><br>### What it does<br>A subcontractor submits the built-in n8n form. Before anyone opens the submission, the claimed licence number is checked against that state's licensing board and the submission comes back with one of four verdicts:<br><br>- `pass` - an active licence exists and it is held by the company that claimed it<br>- `review` - the licence is active but registered to a **different** name. Borrowed licence numbers are a real pattern, so this never auto-passes<br>- `fail` - the licence is expired, revoked, suspended, or there is no record of it at all<br>- `manual-review` - the form was incomplete, the state is not covered, or the lookup itself failed<br><br>**The gate fails closed.** Any error, timeout or ambiguity becomes `manual-review`, never a pass. A compliance tool that guesses in your favour when it breaks is a liability, not a control.<br><br>Every submission also produces an audit row - what was claimed, what the board held, the verdict, and the timestamp.<br><br>Covers AL, AR, CA, CT, FL, MA, MI, MN, NC, NM, NV, OR, SC, TN, TX, VA and WA. Submissions from other states are labelled `manual-review` and passed to a person - never rejected, and never quietly approved.<br><br>### How to set up<br>1. Install the `@apify/n8n-nodes-apify` community node.<br>2. Add your Apify API key as an Apify credential on the **Look up the licence** node.<br>3. Activate the workflow to get the production form URL, and use that as your intake form. Swap the trigger for a Webhook node if you already collect submissions in Typeform, Jotform or your own site.<br>4. Replace **Write the audit row** with Google Sheets -> Append row, Airtable, or a database insert. It ships as a Code node so the template runs on import without a credential.<br>5. Add a Slack credential to **Post the verdict to Slack**, pick a channel, and enable the node.<br><br>### Requirements<br>- An Apify account and API token (about $0.004 per check)<br>- Self-hosted n8n (community node)<br>- Optional: Slack, and a sheet or table for the audit trail<br><br>### How to customize<br>- **Different intake form:** swap the Form Trigger for a Webhook node. The validator accepts several field spellings and a full state name as well as a 2-letter code.<br>- **Stricter matching:** `review` currently covers any name that doesn't match. Tighten or loosen the comparison in **Decide the verdict**.<br>- **Add a state:** one row in the `ACTORS` map in **Route by state**.<br>- **Route the verdicts:** add a Switch after **Decide the verdict** to send `pass` straight to your CRM and hold the rest. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in a self-hosted n8n environment, follow these sequential steps:

1. **Install Prerequisites:** Ensure your self-hosted n8n instance has the community node `@apify/n8n-nodes-apify` installed via Community Nodes settings.
2. **Create Node 1: Form Trigger**
   - Type: `n8n-nodes-base.formTrigger`
   - Name: `Subcontractor submits the form`
   - Parameters: Set form title to "Subcontractor onboarding". Add form fields: Company (Required), Contact name (Required), Email (Type: Email, Required), State (Placeholder: "2-letter code, e.g. CA", Required), License number (Required). Set response mode to `lastNode`.
3. **Create Node 2: Validation Code**
   - Type: `n8n-nodes-base.code`
   - Name: `Validate the submission`
   - Parameters: Insert JavaScript code to normalize field names (company, contact, email, license number, state) and convert full US state names to 2-letter codes using a dictionary mapping. Output object containing `valid` boolean and `missing` fields string.
   - Connection: Connect `Subcontractor submits the form` output to `Validate the submission` input.
4. **Create Node 3: State Router Code**
   - Type: `n8n-nodes-base.code`
   - Name: `Route by state`
   - Parameters: Insert JavaScript code defining the `ACTORS` lookup table mapping US states (`AL`, `AR`, `CA`, `CT`, `FL`, `MA`, `MI`, `MN`, `NC`, `NM`, `NV`, `OR`, `SC`, `TN`, `TX`, `VA`, `WA`) to respective Apify actors and query fields. Output `covered` status, `actorId`, and stringified `actorInput`.
   - Connection: Connect `Validate the submission` output to `Route by state` input.
5. **Create Node 4: Feasibility IF Node**
   - Type: `n8n-nodes-base.if`
   - Name: `Can we check it?`
   - Parameters: Set condition to evaluate if `{{ $json.covered }}` is equal to `true`.
   - Connection: Connect `Route by state` output to `Can we check it?` input.
6. **Create Node 5: Apify License Lookup**
   - Type: `@apify/n8n-nodes-apify.apify`
   - Name: `Look up the licence`
   - Parameters: Set Resource to `Actors`, Operation to `Run actor and get dataset`. Set Actor ID expression to `={{ $json.actorId }}` and Custom Body expression to `={{ $json.actorInput }}`. Configure error handling (`onError`) to `continueRegularOutput`.
   - Credentials: Configure Apify API Token credentials.
   - Connection: Connect `Can we check it?` True output (Index 0) to `Look up the licence` input.
7. **Create Node 6: Manual Review Set Node**
   - Type: `n8n-nodes-base.set`
   - Name: `Send it to a human`
   - Parameters: Assign a string property named `summary` with value `=Needs a human: {{ $json.reason }}` and retain other fields.
   - Connection: Connect `Can we check it?` False output (Index 1) to `Send it to a human` input.
8. **Create Node 7: Verdict Decision Code**
   - Type: `n8n-nodes-base.code`
   - Name: `Decide the verdict`
   - Parameters: Insert JavaScript code comparing claimed license details against scraped board results, applying regex checks for active/inactive status and company name matching. Output verdict (`pass`, `review`, `fail`, `manual-review`) and summary string.
   - Connection: Connect `Look up the licence` output to `Decide the verdict` input.
9. **Create Node 8: Audit Trail Code**
   - Type: `n8n-nodes-base.code`
   - Name: `Write the audit row`
   - Parameters: Insert JavaScript code mapping verification outputs into a clean audit log schema.
   - Connections: Connect both `Decide the verdict` output and `Send it to a human` output into `Write the audit row`.
10. **Create Node 9: Slack Notification**
    - Type: `n8n-nodes-base.slack`
    - Name: `Post the verdict to Slack`
    - Parameters: Set Resource to `channel`, Select to `channel`, Channel ID to `#onboarding`, and Text to `={{ $json.summary }}`. Keep node disabled by default.
    - Credentials: Set Slack OAuth2 / Bot Token credentials.
    - Connection: Connect `Write the audit row` output to `Post the verdict to Slack` input.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Self-hosted requirement notice | This template utilizes the community node `@apify/n8n-nodes-apify`, which is incompatible with n8n Cloud environments. |
| Workflow overview canvas image | [Workflow overview canvas](https://scrapebench.dev/img/n8n/subcontractor-onboarding-gate/canvas.png) |