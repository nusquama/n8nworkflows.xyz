Generate and approve UGC ad videos with OpenRouter, Fal, Replicate, and Discord

https://n8nworkflows.xyz/workflows/generate-and-approve-ugc-ad-videos-with-openrouter--fal--replicate--and-discord-18326


# Generate and approve UGC ad videos with OpenRouter, Fal, Replicate, and Discord

### 1. Workflow Overview

This workflow automates the generation, human review, and production of user-generated content (UGC) advertising videos. Users submit a product image, a textual creative description, and a chosen visual style via an entry form. The system extracts product details using OpenRouter, crafts optimized image and video prompts with an AI agent, and creates a visual preview using Fal. It then submits this preview to Discord for human evaluation. Depending on the reviewer's choice (Approve, Regenerate, or Reject), the workflow triggers video generation via Replicate, loops through AI prompt revision, or gracefully ends. Approved video tasks are polled until completed, logged into an n8n Data Table, and confirmed via Discord notifications.

The architecture groups logically into the following functional blocks:
- **1.1 Input Reception & Product Analysis:** Captures user form submissions and performs an initial visual analysis of the product image using OpenRouter.
- **1.2 Prompt Generation & Image Preview:** Drafts structured UGC image/video prompts using an AI Agent, parses the JSON payload, and calls the Fal API to generate a 9:16 aspect ratio preview image.
- **1.3 Initial Review Routing:** Posts the generated preview to Discord and collects stakeholder decisions (Approve, Regenerate, or Reject) via an interactive form.
- **1.4 Video Generation & Polling (Main Branch):** Initiates a Replicate video job upon approval, polls prediction statuses using a wait-and-check loop, and dispatches completion notifications.
- **1.5 Regeneration & Revision Loop:** Handles reviewer-requested alterations using a secondary AI Agent, regenerates preview assets via Fal, collects secondary approval, and submits revised jobs to Replicate.
- **1.6 Data Persistence & Finalization:** Finalizes job execution metadata, writes records to an n8n Data Table, and posts final completion logs to Discord.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Product Analysis
- **Overview:** This block collects the initial UGC parameters and uploaded image from an n8n Form trigger, then sends them to OpenRouter to inspect the visual properties of the product.
- **Nodes Involved:** `When UGC Form Submitted`, `Post Chat to OpenRouter API`
- **Node Details:**
  - **`When UGC Form Submitted`**
    - Type: `n8n-nodes-base.formTrigger`
    - Role: Entry point capturing product images, UGC text prompts, and video styles.
    - Config: Accepts JPEG, PNG, and WebP files; requires text description and style dropdown.
    - Connections: Output connects to `Post Chat to OpenRouter API`.
    - Edge Cases: Missing required fields or unsupported image file formats will halt execution at the trigger level.
  - **`Post Chat to OpenRouter API`**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Performs a raw HTTP POST request to OpenRouter to extract precise physical attributes and branding details from the uploaded image.
    - Config: Uses JSON body payload containing base64 encoded image data and custom extraction instructions.
    - Expressions: `={{ (() => { const image = $binary.Product_Image; ... })() }}`
    - Connections: Input from `When UGC Form Submitted`, output to `UGC Generation Agent`.
    - Edge Cases: API rate limits, oversized binary payloads, or authentication expiry on OpenRouter credentials.

---

#### 1.2 Prompt Generation & Image Preview
- **Overview:** This block converts the product breakdown and customer request into structured AI image and video prompts, parses the raw string response, and calls the Fal image editing API to generate a visual mockup.
- **Nodes Involved:** `UGC Generation Agent`, `OpenRouter Chat Interface`, `Extract UGC Prompt Data`, `Call UGC Image API Service`, `Prepare UGC Image Data`
- **Node Details:**
  - **`UGC Generation Agent`**
    - Type: `@n8n/n8n-nodes-langchain.agent`
    - Role: Orchestrates prompt generation ensuring strict adherence to product identity constraints.
    - Config: Defined prompt type mapping system constraints with dynamic user requests.
    - Connections: Input from `Post Chat to OpenRouter API` and `OpenRouter Chat Interface`, output to `Extract UGC Prompt Data`.
  - **`OpenRouter Chat Interface`**
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`
    - Role: Provides the underlying LLM backend (`dots-studio/dots-3-note-preview:free`) for the generation agent.
    - Connections: Connected as an AI language model to `UGC Generation Agent`.
  - **`Extract UGC Prompt Data`**
    - Type: `n8n-nodes-base.code`
    - Role: Cleans markdown formatting from the AI agent's response and parses it into strict JSON keys (`image_prompt`, `video_prompt`, `negative_constraints`).
    - Config: JavaScript snippet with fallback handling for malformed outputs.
    - Connections: Input from `UGC Generation Agent`, output to `Call UGC Image API Service`.
  - **`Call UGC Image API Service`**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Submits the image prompt and source image to the Fal editing endpoint to create a vertical preview.
    - Config: POST request to `https://fal.run/fal-ai/nano-banana-2/edit` using HTTP Header authentication (`Fal API`).
    - Connections: Input from `Extract UGC Prompt Data`, output to `Prepare UGC Image Data`.
    - Edge Cases: Fal API timeouts or invalid image URLs.
  - **`Prepare UGC Image Data`**
    - Type: `n8n-nodes-base.set`
    - Role: Standardizes generated image metadata, mode flags, and source URLs into clean execution properties.
    - Connections: Input from `Call UGC Image API Service`, output to `Send UGC Image to Discord`.

---

#### 1.3 Initial Review Routing
- **Overview:** This block sends the generated image preview and prompts to Discord for human oversight and evaluates reviewer decisions to determine the next operational path.
- **Nodes Involved:** `Send UGC Image to Discord`, `UGC Approval Form`, `Route by UGC Approval`
- **Node Details:**
  - **`Send UGC Image to Discord`**
    - Type: `n8n-nodes-base.discord`
    - Role: Posts a formatted preview summary containing truncated prompts, mode flags, and status metadata to a Discord webhook.
    - Connections: Input from `Prepare UGC Image Data`, output to `UGC Approval Form`.
  - **`UGC Approval Form`**
    - Type: `n8n-nodes-base.form`
    - Role: Collects review decisions (`Approve`, `Regenerate`, or `Reject`) and optional textual feedback from human reviewers via an interactive form.
    - Connections: Input from `Send UGC Image to Discord`, output to `Route by UGC Approval`.
  - **`Route by UGC Approval`**
    - Type: `n8n-nodes-base.switch`
    - Role: Directs workflow execution based on the reviewer's choice.
    - Config: Three rules evaluating `{{ $json.Decision }}` against `"Approve"`, `"Regenerate"`, and `"Reject"`.
    - Connections: Input from `UGC Approval Form`. Outputs route to `Request Replicate Video Creation` (Approve) or `Regenerate UGC Content Agent` (Regenerate). Note: Reject branch terminates.

---

#### 1.4 Video Generation & Polling (Main Branch)
- **Overview:** Initiates video generation on Replicate for approved concepts, polls the processing status iteratively using wait intervals, and posts the final video asset to Discord.
- **Nodes Involved:** `Request Replicate Video Creation`, `Wait 30 Seconds`, `Check Video Creation Status`, `Route by Video Status`, `Wait 80 Seconds`, `Notify Discord of Approved UGC`
- **Node Details:**
  - **`Request Replicate Video Creation`**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Submits the approved video prompt and reference image URL to Replicate's `minimax/video-01` prediction endpoint.
    - Config: POST request using HTTP Header authentication.
    - Connections: Input from `Route by UGC Approval`, output to `Wait 30 Seconds`.
  - **`Wait 30 Seconds`**
    - Type: `n8n-nodes-base.wait`
    - Role: Pauses execution to allow video rendering jobs to initialize before status polling begins.
    - Connections: Inputs from `Request Replicate Video Creation` or `Wait 80 Seconds`, output to `Check Video Creation Status`.
  - **`Check Video Creation Status`**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Queries the dynamic prediction status URL returned by the initial Replicate job creation request.
    - Connections: Input from `Wait 30 Seconds`, output to `Route by Video Status`.
  - **`Route by Video Status`**
    - Type: `n8n-nodes-base.switch`
    - Role: Inspects the job status field (`succeeded`, `starting`, or `processing`).
    - Connections: Input from `Check Video Creation Status`. Routes `Succeeded` to `Notify Discord of Approved UGC`, while `Starting` and `Processing` loop back to `Wait 80 Seconds`.
  - **`Wait 80 Seconds`**
    - Type: `n8n-nodes-base.wait`
    - Role: Secondary longer wait interval for heavy video rendering states during polling loops.
    - Connections: Inputs from `Route by Video Status` outputs, output to `Wait 30 Seconds`.
  - **`Notify Discord of Approved UGC`**
    - Type: `n8n-nodes-base.discord`
    - Role: Sends a completion alert containing the final video URL and model metadata to Discord.
    - Connections: Input from `Route by Video Status`, output to `Finalize Job Details`.

---

#### 1.5 Regeneration & Revision Loop
- **Overview:** Handles reviewer rejection feedback by utilizing a secondary AI agent to rewrite prompts, generating new visual previews via Fal, collecting secondary approval, and submitting revised jobs to Replicate.
- **Nodes Involved:** `Regenerate UGC Content Agent`, `Secondary OpenRouter Chat`, `Parse Regenerated AI Output`, `Generate Revised UGC Image`, `Prepare Revised UGC Preview`, `Share Regenerated UGC Preview`, `Approve Revised UGC Form`, `Route by Revised UGC Approval`, `Discord Rejection Notice`, `Discord Approval Confirmation`, `Create Revised Replicate Video`
- **Node Details:**
  - **`Regenerate UGC Content Agent`**
    - Type: `@n8n/n8n-nodes-langchain.agent`
    - Role: Updates the image and video prompt plan based on reviewer critique.
    - Connections: Input from `Route by UGC Approval` and `Secondary OpenRouter Chat`, output to `Parse Regenerated AI Output`.
  - **`Secondary OpenRouter Chat`**
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`
    - Role: LLM backend (`dots-studio/dots-3-note-preview:free`) for prompt regeneration.
    - Connections: Connected as AI language model to `Regenerate UGC Content Agent`.
  - **`Parse Regenerated AI Output`**
    - Type: `n8n-nodes-base.code`
    - Role: Cleans and parses the revised AI text output into structured JSON fields.
    - Connections: Input from `Regenerate UGC Content Agent`, output to `Generate Revised UGC Image`.
  - **`Generate Revised UGC Image`**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Calls the Fal editing endpoint using the revised image prompt.
    - Connections: Input from `Parse Regenerated AI Output`, output to `Prepare Revised UGC Preview`.
  - **`Prepare Revised UGC Preview`**
    - Type: `n8n-nodes-base.set`
    - Role: Bundles revised parameters and sets the mode to `GENERATED REGENERATED`.
    - Connections: Input from `Generate Revised UGC Image`, output to `Share Regenerated UGC Preview`.
  - **`Share Regenerated UGC Preview`**
    - Type: `n8n-nodes-base.discord`
    - Role: Posts the updated preview prompts to Discord for secondary review.
    - Connections: Input from `Prepare Revised UGC Preview`, output to `Approve Revised UGC Form`.
  - **`Approve Revised UGC Form`**
    - Type: `n8n-nodes-base.form`
    - Role: Collects the final human decision (`Approve` or `Reject`) on the revised concept.
    - Connections: Input from `Share Regenerated UGC Preview`, output to `Route by Revised UGC Approval`.
  - **`Route by Revised UGC Approval`**
    - Type: `n8n-nodes-base.switch`
    - Role: Routes execution based on the revised decision.
    - Connections: Input from `Approve Revised UGC Form`. Routes to `Create Revised Replicate Video` (Approve) or `Discord Rejection Notice` (Reject).
  - **`Discord Rejection Notice`**
    - Type: `n8n-nodes-base.discord`
    - Role: Notifies stakeholders that the revised UGC concept was declined and terminated.
    - Connections: Input from `Route by Revised UGC Approval`.
  - **`Discord Approval Confirmation`**
    - Type: `n8n-nodes-base.discord`
    - Role: Confirms approval of the regenerated video concept.
    - Connections: Input from `Create Revised Replicate Video`, output to `Finalize Job Details`.
  - **`Create Revised Replicate Video`**
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Submits the revised prompt and image payload to Replicate.
    - Connections: Input from `Route by Revised UGC Approval`, output to `Discord Approval Confirmation` (which feeds into finalization).

---

#### 1.6 Data Persistence & Finalization
- **Overview:** Consolidates final job execution records, records metadata into an n8n Data Table, and dispatches a closing status notification to Discord.
- **Nodes Involved:** `Finalize Job Details`, `Save Job Record to Database`, `Notify Discord of Completion`
- **Node Details:**
  - **`Finalize Job Details`**
    - Type: `n8n-nodes-base.set`
    - Role: Generates unique job IDs, assigns completion timestamps, and sets final job status variables.
    - Config: Assigns ISO timestamps and unique randomized job strings.
    - Connections: Inputs from `Notify Discord of Approved UGC` or `Discord Approval Confirmation`, output to `Save Job Record to Database`.
  - **`Save Job Record to Database`**
    - Type: `n8n-nodes-base.dataTable`
    - Role: Persists structured job metrics into the `ugc_jobs` n8n Data Table.
    - Config: Maps table schema fields (`job_id`, `job_status`, `mode`, `completed_at`, `final_video_status`, `final_video_prompt`) to incoming JSON data.
    - Connections: Input from `Finalize Job Details`, output to `Notify Discord of Completion`.
    - Edge Cases: Data table misconfigurations or missing schema columns.
  - **`Notify Discord of Completion`**
    - Type: `n8n-nodes-base.discord`
    - Role: Sends the final workflow completion confirmation message to Discord.
    - Connections: Input from `Save Job Record to Database`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When UGC Form Submitted` | `formTrigger` | Capture product request | None | `Post Chat to OpenRouter API` | Capture product request |
| `Post Chat to OpenRouter API` | `httpRequest` | Capture product request | `When UGC Form Submitted` | `UGC Generation Agent` | Capture product request |
| `UGC Generation Agent` | `agent` | Draft UGC prompts | `Post Chat to OpenRouter API`, `OpenRouter Chat Interface` | `Extract UGC Prompt Data` | Draft UGC prompts |
| `OpenRouter Chat Interface` | `lmChatOpenRouter` | Draft UGC prompts | None | `UGC Generation Agent` | Draft UGC prompts |
| `Extract UGC Prompt Data` | `code` | Draft UGC prompts | `UGC Generation Agent` | `Call UGC Image API Service` | Draft UGC prompts |
| `Call UGC Image API Service` | `httpRequest` | Generate image preview | `Extract UGC Prompt Data` | `Prepare UGC Image Data` | Generate image preview |
| `Prepare UGC Image Data` | `set` | Generate image preview | `Call UGC Image API Service` | `Send UGC Image to Discord` | Generate image preview |
| `Send UGC Image to Discord` | `discord` | Initial preview approval | `Prepare UGC Image Data` | `UGC Approval Form` | Initial preview approval |
| `UGC Approval Form` | `form` | Initial preview approval | `Send UGC Image to Discord` | `Route by UGC Approval` | Initial preview approval |
| `Route by UGC Approval` | `switch` | Initial preview approval | `UGC Approval Form` | `Request Replicate Video Creation`, `Regenerate UGC Content Agent` | Initial preview approval |
| `Request Replicate Video Creation` | `httpRequest` | Start approved video | `Route by UGC Approval` | `Wait 30 Seconds` | Start approved video |
| `Regenerate UGC Content Agent` | `agent` | Regenerate UGC concept | `Route by UGC Approval`, `Secondary OpenRouter Chat` | `Parse Regenerated AI Output` | Regenerate UGC concept |
| `Secondary OpenRouter Chat` | `lmChatOpenRouter` | Regenerate UGC concept | None | `Regenerate UGC Content Agent` | Regenerate UGC concept |
| `Parse Regenerated AI Output` | `code` | Regenerate UGC concept | `Regenerate UGC Content Agent` | `Generate Revised UGC Image` | Regenerate UGC concept |
| `Generate Revised UGC Image` | `httpRequest` | Regenerate UGC concept | `Parse Regenerated AI Output` | `Prepare Revised UGC Preview` | Regenerate UGC concept |
| `Prepare Revised UGC Preview` | `set` | Regenerate UGC concept | `Generate Revised UGC Image` | `Share Regenerated UGC Preview` | Regenerate UGC concept |
| `Share Regenerated UGC Preview` | `discord` | Review regenerated concept | `Prepare Revised UGC Preview` | `Approve Revised UGC Form` | Review regenerated concept |
| `Approve Revised UGC Form` | `form` | Review regenerated concept | `Share Regenerated UGC Preview` | `Route by Revised UGC Approval` | Review regenerated concept |
| `Route by Revised UGC Approval` | `switch` | Review regenerated concept | `Approve Revised UGC Form` | `Create Revised Replicate Video`, `Discord Rejection Notice` | Review regenerated concept |
| `Discord Rejection Notice` | `discord` | Review regenerated concept | `Route by Revised UGC Approval` | None | Review regenerated concept |
| `Discord Approval Confirmation` | `discord` | Review regenerated concept | `Create Revised Replicate Video` | `Finalize Job Details` | Review regenerated concept |
| `Create Revised Replicate Video` | `httpRequest` | Review regenerated concept | `Route by Revised UGC Approval` | `Discord Approval Confirmation` | Review regenerated concept |
| `Wait 30 Seconds` | `wait` | Poll video status | `Request Replicate Video Creation`, `Wait 80 Seconds` | `Check Video Creation Status` | Poll video status |
| `Check Video Creation Status` | `httpRequest` | Poll video status | `Wait 30 Seconds` | `Route by Video Status` | Poll video status |
| `Route by Video Status` | `switch` | Poll video status | `Check Video Creation Status` | `Notify Discord of Approved UGC`, `Wait 80 Seconds`, `Wait 80 Seconds` | Poll video status |
| `Wait 80 Seconds` | `wait` | Poll video status | `Route by Video Status` | `Wait 30 Seconds` | Poll video status |
| `Notify Discord of Approved UGC` | `discord` | Poll video status | `Route by Video Status` | `Finalize Job Details` | Poll video status |
| `Finalize Job Details` | `set` | Finalize and notify | `Notify Discord of Approved UGC`, `Discord Approval Confirmation` | `Save Job Record to Database` | Finalize and notify |
| `Save Job Record to Database` | `dataTable` | Finalize and notify | `Finalize Job Details` | `Notify Discord of Completion` | Finalize and notify |
| `Notify Discord of Completion` | `discord` | Finalize and notify | `Save Job Record to Database` | None | Finalize and notify |
| Sticky Note | `stickyNote` | Overview Documentation | None | None | UGC AI Generator |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to manually build and configure the workflow inside n8n:

1. **Create the Entry Form Trigger (`When UGC Form Submitted`)**:
   - Add a **Form Trigger** node.
   - Configure form title: `UGC AI Video Generator`.
   - Add Form Fields:
     1. Field type `File`, label `Product Image`, required, accept types: `image/jpeg,image/png,image/webp`.
     2. Field type `Textarea`, label `UGC Prompt`, required.
     3. Field type `Dropdown`, label `Video Style`, options: `Luxury`, `Lifestyle`, `Streetwear`, `Product Showcase`, `Social Media UGC`, `Cinematic`, required.

2. **Add Product Analysis (`Post Chat to OpenRouter API`)**:
   - Add an **HTTP Request** node connected to the Form Trigger.
   - Method: `POST`, URL: `https://openrouter.ai/api/v1/chat/completions`.
   - Set Authentication to `Predefined Credential Type` -> `OpenRouter account` (or configure HTTP Header Auth).
   - Set Header `Content-Type: application/json`.
   - Configure JSON Body to extract base64 image data and query OpenRouter for product attribute parsing.

3. **Configure AI Generation Agent & Model (`UGC Generation Agent` & `OpenRouter Chat Interface`)**:
   - Add an **AI Agent** node connected to the HTTP Request.
   - Add an **OpenRouter Chat Model** node and connect its output to the Agent's AI Language Model input.
   - Configure the Chat Model to use `dots-studio/dots-3-note-preview:free` with OpenRouter credentials.
   - Configure the Agent prompt to act as a professional UGC advertising prompt engineer, parsing the product analysis and outputting structured JSON (`image_prompt`, `video_prompt`, `negative_constraints`).

4. **Parse Agent Output (`Extract UGC Prompt Data`)**:
   - Add a **Code** node connected to the AI Agent.
   - Insert JavaScript to strip Markdown code fences (` ```json `), parse the resulting JSON string, and output clean property fields.

5. **Generate Initial Preview Image (`Call UGC Image API Service` & `Prepare UGC Image Data`)**:
   - Add an **HTTP Request** node connected to the Code node.
   - Method: `POST`, URL: `https://fal.run/fal-ai/nano-banana-2/edit`.
   - Configure HTTP Header Auth credentials (`Fal API`).
   - Pass JSON body mapping the image prompt, source product image URL, aspect ratio (`9:16`), and output format (`png`).
   - Add a **Set** node (`Prepare UGC Image Data`) to assign metadata parameters (`mode: GENERATED`, `status: completed`, `image_url`).

6. **Post Preview and Collect Approval (`Send UGC Image to Discord` & `UGC Approval Form`)**:
   - Add a **Discord** node configured with Webhook authentication to send the formatted preview summary.
   - Add a **Form** node (`UGC Approval Form`) with a dropdown field `Decision` (`Approve`, `Regenerate`, `Reject`) and a textarea field `Feedback`.

7. **Route Initial Approval (`Route by UGC Approval`)**:
   - Add a **Switch** node connected to the Approval Form.
   - Create 3 rules matching `{{ $json.Decision }}` equal to `"Approve"`, `"Regenerate"`, and `"Reject"`.

8. **Handle Approved Path & Replicate Video Generation (`Request Replicate Video Creation`)**:
   - From the `Approve` branch of the switch, add an **HTTP Request** node.
   - Method: `POST`, URL: `https://api.replicate.com/v1/models/minimax/video-01/predictions`.
   - Configure HTTP Header Auth (`Fal API` / Replicate endpoint credentials).
   - Pass JSON payload containing the video prompt, first-frame image URL, and `prompt_optimizer: true`.

9. **Configure Polling Loop (`Wait 30 Seconds`, `Check Video Creation Status`, `Route by Video Status`, `Wait 80 Seconds`)**:
   - Connect the Replicate request to a **Wait** node set to `30` seconds.
   - Connect the wait node to an **HTTP Request** node (`Check Video Creation Status`) pointing to the dynamic prediction status URL (`{{ $('Request Replicate Video Creation').first().json.urls.get }}`).
   - Connect to a **Switch** node (`Route by Video Status`) checking if `{{ $json.status }}` equals `succeeded`, `starting`, or `processing`.
   - Route `starting` and `processing` outputs to a second **Wait** node set to `80` seconds, looping back into the initial 30-second wait node.
   - Route `succeeded` to a **Discord** node (`Notify Discord of Approved UGC`) to broadcast the final video URL.

10. **Build Regeneration Branch (`Regenerate UGC Content Agent`, `Secondary OpenRouter Chat`, `Parse Regenerated AI Output`, `Generate Revised UGC Image`, `Prepare Revised UGC Preview`, `Share Regenerated UGC Preview`, `Approve Revised UGC Form`, `Route by Revised UGC Approval`, `Create Revised Replicate Video`)**:
    - From the `Regenerate` branch of the initial switch, add a second **AI Agent** and **OpenRouter Chat Model** pair to revise prompts based on reviewer feedback.
    - Add a **Code** node to parse the regenerated output.
    - Add an **HTTP Request** node to call the Fal API (`https://fal.run/fal-ai/nano-banana-2/edit`) for the revised image.
    - Add a **Set** node (`Prepare Revised UGC Preview`) and a **Discord** node to share the new preview.
    - Add an **Approval Form** (`Approve Revised UGC Form`) and a **Switch** node (`Route by Revised UGC Approval`) with `Approve` and `Reject` outputs.
    - Route `Reject` to a Discord notification node.
    - Route `Approve` to an **HTTP Request** node (`Create Revised Replicate Video`) pointing to the Replicate prediction endpoint, followed by a Discord confirmation node.

11. **Finalize and Save (`Finalize Job Details`, `Save Job Record to Database`, `Notify Discord of Completion`)**:
    - Connect successful video notifications from both main and regenerated branches to a **Set** node (`Finalize Job Details`) generating unique job IDs and timestamps (`{{ $now.toISO() }}`).
    - Add a **Data Table** node (`Save Job Record to Database`) targeting the `ugc_jobs` table, mapping schema columns (`job_id`, `job_status`, `mode`, `completed_at`, `final_video_status`, `final_video_prompt`).
    - Add a final **Discord** node to send the completion summary message.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| UGC AI Generator Workflow Overview | This template turns submitted UGC requests into AI-generated creative assets by orchestrating OpenRouter, Fal, Replicate, and Discord integrations. |
| Data Table Target | The final workflow step writes job logs into an n8n Data Table with ID `UflKhOI8xcdeioI0` (named `ugc_jobs`). |