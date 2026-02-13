Create lyric posters from Spotify tracks with Musixmatch and OpenAI

https://n8nworkflows.xyz/workflows/create-lyric-posters-from-spotify-tracks-with-musixmatch-and-openai-12736


# Create lyric posters from Spotify tracks with Musixmatch and OpenAI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Create lyric posters from Spotify tracks with Musixmatch and OpenAI  
**Purpose:** Collect a Spotify track URL, fetch track metadata from Spotify, retrieve lyrics from Musixmatch, use an AI agent to pick a single impactful lyric line and craft a detailed poster prompt, generate an A4-like portrait poster image via OpenAI image generation, then return the image to the form submitter.

### 1.1 Input Reception & Normalization
Accepts a Spotify track URL via an n8n Form trigger and extracts a canonical `spotify:track:<id>` URI.

### 1.2 Track Metadata Retrieval (Spotify)
Uses the extracted URI to fetch track name and artist from Spotify.

### 1.3 AI Processing (Matching + Lyrics + Prompt Authoring)
An AI Agent (powered by an OpenAI chat model) calls two Musixmatch “tool” nodes: one to match the track by metadata, then one to fetch lyrics. The agent selects a single-line quote and produces a structured image-generation prompt.

### 1.4 Image Generation & User Delivery
Generates a 1024×1536 poster image using OpenAI image generation and returns it to the user as a binary download via the Form completion node.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Track ID Extraction
**Overview:** Collects the Spotify URL from a form and converts it into a Spotify track URI suitable for the Spotify node.  
**Nodes involved:**  
- Sticky Note (How it works / setup)  
- Sticky Note1 (Input)  
- Collect Spotify URL from form  
- Extract Spotify track ID

#### Sticky Note (How it works / setup)
- **Type / role:** Sticky Note; documentation overlay for the workflow.
- **Configuration choices:** Explains overall behavior and setup requirements (Spotify, Musixmatch, OpenAI credentials; output size).
- **Connections:** None.
- **Failure modes:** None (non-executing).

#### Sticky Note1 (Input)
- **Type / role:** Sticky Note; labels the input block (“Collects Spotify URL and extracts track ID”).
- **Connections:** None.

#### Collect Spotify URL from form
- **Type / role:** `Form Trigger` (interactive entry point).
- **Key configuration:**
  - **Form title:** “Create your lyrics poster”
  - **Description:** “Paste a valid Spotify song URL to generate a lyrics poster”
  - **Field:** “Spotify track URL” (expects a full Spotify track URL)
  - **Response mode:** `lastNode` (the form waits and returns the output of the last node in the execution chain).
- **Input/Output connections:**
  - Output → **Extract Spotify track ID**
- **Version notes:** typeVersion `2.4` (Form nodes changed across n8n versions; ensure you’re on a version that supports Form Trigger + completion flow).
- **Edge cases / failures:**
  - User submits a non-track URL (album/playlist) → downstream extraction fails.
  - User submits `spoti.fi` short link → explicit error thrown later (by Code node).

#### Extract Spotify track ID
- **Type / role:** `Code` node; parses and validates Spotify track ID and emits `track_uri`.
- **Key configuration:**
  - JavaScript extracts a **22-character** base62 track ID from either:
    - `https://open.spotify.com/track/<id>?...`
    - `spotify:track:<id>`
  - Writes: `item.json.track_uri = "spotify:track:<trackId>"`
  - Input field expected: `Spotify track URL` with fallbacks: `url`, `spotifyUrl`, `spotify_track_url`.
- **Input/Output connections:**
  - Input ← **Collect Spotify URL from form**
  - Output → **Get track metadata from Spotify**
- **Edge cases / failures (explicitly handled):**
  - Missing field → throws error with expected field name.
  - `spoti.fi/` short URL → throws error instructing to add an HTTP Request node to expand redirects.
  - Any URL/URI without a 22-char track ID → throws “Invalid Spotify track URL/URI…”.

---

### Block 2 — Track Metadata Retrieval (Spotify)
**Overview:** Fetches track metadata (name, artist) required to locate lyrics and to populate poster credits.  
**Nodes involved:**  
- Get track metadata from Spotify

#### Get track metadata from Spotify
- **Type / role:** `Spotify` node; calls Spotify API to retrieve a track object.
- **Key configuration:**
  - **Resource:** Track
  - **Operation:** Get
  - **ID expression:** `={{ $json.track_uri }}`
  - Requires Spotify OAuth2 credentials.
- **Input/Output connections:**
  - Input ← **Extract Spotify track ID**
  - Output → **Select lyric and build image prompt**
- **Edge cases / failures:**
  - OAuth expired/invalid scopes → authentication error.
  - Track not found / invalid ID → 404-like API error.
  - Rate limits → intermittent failures.
  - Metadata shape assumptions (later node uses `$json.album.artists[0].name`) may break if Spotify response differs (rare, but possible for unusual items).

---

### Block 3 — AI Processing (Musixmatch tools + Prompt authoring agent)
**Overview:** Uses an AI Agent with an OpenAI chat model. The agent must (1) match the track on Musixmatch using Spotify title/artist, (2) fetch lyrics, (3) select a single strong line under strict constraints, and (4) output a long structured image-generation prompt.  
**Nodes involved:**  
- Sticky Note2 (AI Processing)  
- OpenAI Chat Model  
- Match track by metadata in Musixmatch  
- Get lyrics from Musixmatch  
- Select lyric and build image prompt

#### Sticky Note2 (AI Processing)
- **Type / role:** Sticky Note; labels this block.
- **Connections:** None.

#### OpenAI Chat Model
- **Type / role:** LangChain Chat Model node (`lmChatOpenAi`); provides the LLM used by the agent.
- **Key configuration:**
  - **Model:** `gpt-5.2` (selected from list)
  - **Options:** default/empty
  - Connected as the agent’s `ai_languageModel`.
- **Input/Output connections:**
  - Output (ai_languageModel) → **Select lyric and build image prompt**
- **Edge cases / failures:**
  - OpenAI credential invalid, quota exceeded, or model unavailable in your account/region.
  - Model name/version mismatches if your n8n/OpenAI integration doesn’t support `gpt-5.2`.

#### Match track by metadata in Musixmatch
- **Type / role:** Musixmatch “Tool” node; used by the AI agent to identify the correct Musixmatch track.
- **Key configuration:**
  - **Operation:** `matcherTrackGet`
  - **qTrack:** `{{$fromAI('Track_Name', '', 'string')}}`
  - **qArtist:** `{{$fromAI('Artist_Name', '', 'string')}}`
  - This node is not called directly by the main flow; it is invoked by the agent as a tool.
- **Input/Output connections:**
  - Output (ai_tool) → **Select lyric and build image prompt**
- **Important behavior note (agent-tool binding):**
  - The agent is instructed to call a tool named `match_track_by_metadata`. In practice, n8n maps the tool-call to this Musixmatch tool node.
  - The `$fromAI(...)` expressions are placeholders populated from the agent’s tool call arguments.
- **Edge cases / failures:**
  - Musixmatch API key missing/invalid → auth error.
  - No match found / ambiguous results → agent may select wrong track or fail to proceed.
  - Rate limits.

#### Get lyrics from Musixmatch
- **Type / role:** Musixmatch “Tool” node; retrieves full lyrics for a matched track.
- **Key configuration:**
  - **Operation:** `trackLyricsGet`
  - **commonTrackId:** `{{$fromAI('Common_Track_ID', '', 'string')}}` (must be provided by the agent based on matching results)
- **Input/Output connections:**
  - Output (ai_tool) → **Select lyric and build image prompt**
- **Edge cases / failures:**
  - Track exists but lyrics not available in Musixmatch region/account tier.
  - API returns partial/limited lyrics depending on Musixmatch plan.
  - Agent may pass an invalid `Common_Track_ID` if matching step fails.

#### Select lyric and build image prompt
- **Type / role:** LangChain Agent node; orchestrates tool use + reasoning to produce the final image prompt.
- **Key configuration:**
  - **Input text:** `={{ $json.name }} by {{ $json.album.artists[0].name }}`
    - Derived from Spotify track metadata.
  - **System message:** A strict, multi-step specification requiring:
    1) Match track via Musixmatch tool  
    2) Get full lyrics  
    3) Select **exactly one** lyric line (≤ 90 chars, verbatim, no quotes, no line breaks)  
    4) Interpret meaning  
    5) Output a **250–450 word** structured prompt with required sections (Song Metadata, Concept, Style, Composition, Texture, Typography, Constraints)
  - **Prompt type:** “define”
  - **Output parser enabled:** `hasOutputParser: true` (expects clean structured output; increases risk of failure if model deviates).
- **Input/Output connections:**
  - Main input ← **Get track metadata from Spotify**
  - AI language model input ← **OpenAI Chat Model**
  - Tool inputs ← **Match track by metadata in Musixmatch**, **Get lyrics from Musixmatch**
  - Main output → **Generate poster with OpenAI**
- **Edge cases / failures:**
  - If Spotify returns unexpected structure (missing `album.artists[0]`) the expression may fail.
  - Tool call mismatch: if the agent fails to call the tools in order, output may be poor or invalid.
  - Output constraint violations (e.g., lyric too long, includes quotes, missing required sections) can break downstream expectations (though downstream only uses `$json.output` as prompt).

---

### Block 4 — Image Generation & Form Completion Output
**Overview:** Converts the agent’s final prompt into a high-quality portrait image and returns it as a downloadable file in the form completion step.  
**Nodes involved:**  
- Sticky Note3 (Output)  
- Generate poster with OpenAI  
- Return poster to user

#### Sticky Note3 (Output)
- **Type / role:** Sticky Note; labels output block.
- **Connections:** None.

#### Generate poster with OpenAI
- **Type / role:** OpenAI node (LangChain OpenAI resource) for **image generation**.
- **Key configuration:**
  - **Resource:** `image`
  - **Model:** `gpt-image-1`
  - **Prompt expression:** `={{ $json.output }}` (expects the agent node to output the prompt in `output`)
  - **Options:**
    - Size: `1024x1536` (portrait)
    - Quality: `high`
- **Input/Output connections:**
  - Input ← **Select lyric and build image prompt**
  - Output → **Return poster to user**
- **Edge cases / failures:**
  - OpenAI image model not enabled for the account, or policy refusal based on prompt content.
  - Prompt too long or malformed for the image endpoint.
  - Binary data handling: ensure the node outputs binary in a format compatible with the Form completion node.

#### Return poster to user
- **Type / role:** `Form` node (completion responder); returns the generated image to the submitter.
- **Key configuration:**
  - **Operation:** `completion`
  - **Respond with:** `returnBinary` (sends binary file back to browser)
  - **Title:** “Your poster is ready”
  - **Message:** “Find the poster in your download folder!”
- **Input/Output connections:**
  - Input ← **Generate poster with OpenAI**
- **Edge cases / failures:**
  - If the prior node didn’t produce binary output as expected, the response may fail or return empty.
  - Browser download behavior varies; users may not see the download depending on browser settings.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / setup notes | — | — | ## How it works… (includes setup steps and output size description) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation label for input block | — | — | ## Input \nCollects Spotify URL and extracts track ID |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation label for AI block | — | — | ## AI Processing \nMatches track, retrieves lyrics, selects best line, builds image prompt |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation label for output block | — | — | ## Output \nGenerates image and returns to user |
| Collect Spotify URL from form | n8n-nodes-base.formTrigger | Entry point: collect Spotify track URL | — | Extract Spotify track ID | ## Input \nCollects Spotify URL and extracts track ID |
| Extract Spotify track ID | n8n-nodes-base.code | Parse/validate track URL and create `track_uri` | Collect Spotify URL from form | Get track metadata from Spotify | ## Input \nCollects Spotify URL and extracts track ID |
| Get track metadata from Spotify | n8n-nodes-base.spotify | Fetch track name/artist from Spotify | Extract Spotify track ID | Select lyric and build image prompt |  |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backing the agent | — | Select lyric and build image prompt (ai_languageModel) | ## AI Processing \nMatches track, retrieves lyrics, selects best line, builds image prompt |
| Match track by metadata in Musixmatch | @musixmatch/n8n-nodes-musixmatch.musixmatchTool | Agent tool: find Musixmatch track by title/artist | — (agent tool call) | Select lyric and build image prompt (ai_tool) | ## AI Processing \nMatches track, retrieves lyrics, selects best line, builds image prompt |
| Get lyrics from Musixmatch | @musixmatch/n8n-nodes-musixmatch.musixmatchTool | Agent tool: fetch lyrics by Common Track ID | — (agent tool call) | Select lyric and build image prompt (ai_tool) | ## AI Processing \nMatches track, retrieves lyrics, selects best line, builds image prompt |
| Select lyric and build image prompt | @n8n/n8n-nodes-langchain.agent | Orchestrate tools + produce final image prompt | Get track metadata from Spotify (+ chat model + tools) | Generate poster with OpenAI | ## AI Processing \nMatches track, retrieves lyrics, selects best line, builds image prompt |
| Generate poster with OpenAI | @n8n/n8n-nodes-langchain.openAi | Generate poster image from prompt | Select lyric and build image prompt | Return poster to user | ## Output \nGenerates image and returns to user |
| Return poster to user | n8n-nodes-base.form | Return binary image to form submitter | Generate poster with OpenAI | — | ## Output \nGenerates image and returns to user |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: *Create lyric posters from Spotify tracks with Musixmatch and OpenAI* (or your preferred name).

2) **Add “Form Trigger” node**
   - Node type: **Form Trigger**
   - Form title: `Create your lyrics poster`
   - Form description: `Paste a valid Spotify song URL to generate a lyrics poster`
   - Add a field:
     - Field label/name: `Spotify track URL`
     - Placeholder: `https://open.spotify.com/track/...`
   - Response mode: **Last node** (`lastNode`)

3) **Add “Code” node: Extract Spotify track ID**
   - Node type: **Code**
   - Paste logic that:
     - Reads `item.json["Spotify track URL"]`
     - Extracts the 22-char ID from `open.spotify.com/track/<id>` or `spotify:track:<id>`
     - Outputs `item.json.track_uri = "spotify:track:<id>"`
     - Throws an error for `spoti.fi` short links (unless you add redirect expansion)
   - Connect: **Form Trigger → Code**

4) **Add “Spotify” node: Get track metadata**
   - Node type: **Spotify**
   - Resource: **Track**
   - Operation: **Get**
   - ID: set expression to `{{$json.track_uri}}`
   - Credentials:
     - Configure **Spotify OAuth2** in n8n (Spotify Developer app → client ID/secret; authorize).
   - Connect: **Code → Spotify**

5) **Add “OpenAI Chat Model” node**
   - Node type: **OpenAI Chat Model** (LangChain chat model node)
   - Model: choose `gpt-5.2` (or the closest available equivalent in your environment)
   - Credentials:
     - Configure **OpenAI API** credentials in n8n.
   - No main connections needed; it will connect to the agent via the AI port.

6) **Add Musixmatch tool node #1: Match track**
   - Node type: **Musixmatch Tool**
   - Operation: **matcherTrackGet**
   - qTrack: set expression to `{{$fromAI('Track_Name', '', 'string')}}`
   - qArtist: set expression to `{{$fromAI('Artist_Name', '', 'string')}}`
   - Credentials:
     - Configure **Musixmatch API** credentials (API key).
   - This node will connect to the agent via the **ai_tool** connection.

7) **Add Musixmatch tool node #2: Get lyrics**
   - Node type: **Musixmatch Tool**
   - Operation: **trackLyricsGet**
   - commonTrackId: `{{$fromAI('Common_Track_ID', '', 'string')}}`
   - Credentials: same Musixmatch API key
   - Connect to agent via **ai_tool**.

8) **Add “Agent” node: Select lyric and build image prompt**
   - Node type: **Agent** (LangChain)
   - Input text expression: `{{$json.name}} by {{$json.album.artists[0].name}}`
   - System message: paste the full strict specification (track match → lyrics → single-line selection constraints → structured poster prompt sections).
   - Ensure the agent:
     - Uses **OpenAI Chat Model** as its language model (connect `OpenAI Chat Model` → agent `ai_languageModel` input).
     - Has both Musixmatch tools connected to its `ai_tool` input.
   - Main connection: **Spotify → Agent**

9) **Add “OpenAI Image” node: Generate poster**
   - Node type: **OpenAI** (image resource node)
   - Resource: **Image**
   - Model: `gpt-image-1`
   - Prompt: expression `{{$json.output}}`
   - Options:
     - Size: `1024x1536`
     - Quality: `high`
   - Credentials: OpenAI API
   - Connect: **Agent → OpenAI Image**

10) **Add “Form” node: Return poster to user**
   - Node type: **Form**
   - Operation: **Completion**
   - Respond with: **Return Binary**
   - Completion title: `Your poster is ready`
   - Completion message: `Find the poster in your download folder!`
   - Connect: **OpenAI Image → Form Completion**

11) **(Optional but recommended) Handle Spotify short links**
   - Insert an **HTTP Request** node between Form Trigger and Code to expand `spoti.fi` redirects:
     - Follow redirects enabled
     - Replace incoming URL with final URL before extraction.

12) **Test**
   - Click **Test workflow**
   - Submit a valid Spotify track URL (`open.spotify.com/track/...`)
   - Confirm you receive a downloaded image.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow generates a printable lyrics poster from any Spotify song; fetches metadata, retrieves lyrics (Musixmatch), selects the most impactful line, and generates a gallery-ready A4 poster using OpenAI image generation. Output is a 1024×1536 portrait image with selected lyric and song credits. | From Sticky Note (“How it works”) |
| Setup steps: connect Spotify credentials in “Get track metadata from Spotify”; add Musixmatch API key to both Musixmatch tool nodes; connect OpenAI account in “Generate poster with OpenAI”; activate/test workflow. | From Sticky Note (“Setup steps”) |