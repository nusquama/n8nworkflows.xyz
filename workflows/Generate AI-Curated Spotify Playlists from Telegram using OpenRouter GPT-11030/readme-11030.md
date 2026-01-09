Generate AI-Curated Spotify Playlists from Telegram using OpenRouter GPT

https://n8nworkflows.xyz/workflows/generate-ai-curated-spotify-playlists-from-telegram-using-openrouter-gpt-11030


# Generate AI-Curated Spotify Playlists from Telegram using OpenRouter GPT

### 1. Workflow Overview

This workflow automates the creation of personalized Spotify playlists based on user requests sent via Telegram. The user sends a message describing the desired playlist style or theme. The workflow then uses an AI agent (powered by OpenRouter GPT) to generate a curated list of 25-40 songs matching the user’s query. Subsequently, it creates a new Spotify playlist under the user’s account, searches each suggested track on Spotify, and adds the found tracks to the playlist with respect for API rate limits.

**Target Use Cases:**  
- Users wanting quick, AI-curated Spotify playlists without manually searching and compiling songs.  
- DJs or music enthusiasts who want automated playlist generation with natural language descriptions.  

**Logical Blocks:**  
- **1.1 Receive & Initialize:** Trigger workflow on Telegram message, create an empty Spotify playlist, and notify the user.  
- **1.2 AI Curation:** Use an AI agent to generate a structured list of track suggestions based on the user’s Telegram message.  
- **1.3 Fulfillment Loop:** Process AI output by splitting the song list, searching each track on Spotify, and adding tracks to the playlist one by one with a delay to avoid rate limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Receive & Initialize

**Overview:**  
This block listens for incoming Telegram messages, creates a new Spotify playlist named after the user and the request, and sends back the playlist URL to the user as confirmation.

**Nodes Involved:**  
- Telegram Trigger  
- Create a playlist  
- Send a text message  

**Node Details:**  

- **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Role: Entry point; triggers on any Telegram message update.  
  - Configuration: Listens to "message" updates; uses Telegram bot credentials named "spotgen".  
  - Inputs: None (trigger node)  
  - Outputs: Telegram message JSON including user, chat, and text.  
  - Edge Cases: Missing/invalid Telegram credentials, webhook misconfiguration, or receiving unsupported message types.

- **Create a playlist**  
  - Type: Spotify node, operation "create playlist"  
  - Role: Creates a new empty Spotify playlist for the user.  
  - Configuration: Playlist name dynamically generated from Telegram username + message text + current time (minutes and hours).  
  - Inputs: Telegram Trigger output (user info and message text)  
  - Outputs: JSON including new playlist ID and external Spotify URL.  
  - Credentials: Spotify OAuth2 credentials ("Spotify account").  
  - Edge Cases: Spotify API rate limits, auth expiration, or invalid user scopes.

- **Send a text message**  
  - Type: Telegram node  
  - Role: Sends the Spotify playlist URL back to the Telegram chat as confirmation.  
  - Configuration: Message text set to the Spotify playlist URL extracted from "Create a playlist" node output. Chat ID taken from Telegram Trigger node message.  
  - Inputs: Playlist creation output and Telegram Trigger output.  
  - Outputs: Confirmation of message sent.  
  - Edge Cases: Telegram API errors, invalid chat ID, or sending failures.

---

#### 1.2 AI Curation

**Overview:**  
This block uses an AI agent to generate a playlist tracklist based on the user’s input. The agent acts as a DJ, outputting 25-40 track suggestions in a structured JSON format.

**Nodes Involved:**  
- AI Agent  
- OpenRouter Chat Model  
- Structured Output Parser  
- Split Out Array  

**Node Details:**  

- **AI Agent**  
  - Type: LangChain Agent node  
  - Role: Core AI processor that crafts the playlist track suggestions.  
  - Configuration: Prompt instructs to generate 25-40 playlists similar to the Telegram input message, return results in JSON, with a system message framing it as "you are greatest dj ever lived". Prompt type is "define". Output parser enabled.  
  - Inputs: Playlist request text from Telegram Trigger via Create a playlist node.  
  - Outputs: AI-generated JSON list of tracks.  
  - Sub-nodes: Uses OpenRouter Chat Model as language model and Structured Output Parser for output parsing.  
  - Edge Cases: API timeouts, malformed prompt, or unexpected AI output format.

- **OpenRouter Chat Model**  
  - Type: LangChain OpenRouter Chat Model node  
  - Role: Underlying language model backend providing GPT-5 nano responses.  
  - Configuration: Model set to "openai/gpt-5-nano", response format JSON object.  
  - Inputs: Receives prompt from AI Agent node.  
  - Outputs: Raw AI response.  
  - Credentials: OpenRouter API credentials ("OpenRouter account").  
  - Edge Cases: API limits, authentication failure, or network errors.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser node  
  - Role: Parses AI output into a predefined JSON schema for song lists (track title and artist).  
  - Configuration: Manual JSON schema defines an array of objects each with "track" and "artist" string properties.  
  - Inputs: AI raw output from OpenRouter Chat Model.  
  - Outputs: Parsed JSON array under field "output".  
  - Edge Cases: Parsing failures if AI output deviates from schema.

- **Split Out Array**  
  - Type: Split Out node  
  - Role: Splits the parsed array of songs into individual items for processing.  
  - Configuration: Splits on field "output".  
  - Inputs: Parsed structured output.  
  - Outputs: Single item objects each representing one track.  
  - Edge Cases: Empty arrays or missing "output" field.

---

#### 1.3 Fulfillment Loop

**Overview:**  
This block loops through each suggested track, searches Spotify for the exact song, and adds it to the playlist. It respects Spotify API rate limits by adding a 1-second wait between each addition.

**Nodes Involved:**  
- Loop Over Items (Split In Batches)  
- Search tracks by keyword  
- If (check track found)  
- Add an Item to a playlist  
- Wait 1 Sec  

**Node Details:**  

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes each track item individually (batch size default 1).  
  - Inputs: Output of "Split Out Array" node.  
  - Outputs: Single track items forwarded for search.  
  - Edge Cases: Empty input, batch size misconfiguration.

- **Search tracks by keyword**  
  - Type: Spotify node, operation "search track"  
  - Role: Searches Spotify catalog for the given track and artist combined keyword.  
  - Configuration: Query built as `"<artist> <track>"` from current JSON item. Limit set to 1 result.  
  - Inputs: One track item from AI output.  
  - Outputs: Search results with track metadata including URI.  
  - Credentials: Spotify OAuth2 credentials.  
  - Edge Cases: No results found, API errors.

- **If**  
  - Type: If node  
  - Role: Checks if the search result contains a valid track URI.  
  - Configuration: Condition checks if field `$json.uri` exists.  
  - Inputs: Search tracks by keyword output.  
  - Outputs:  
    - True branch: Proceed to add track to playlist.  
    - False branch: Skip adding track.  
  - Edge Cases: Missing or empty search result.

- **Add an Item to a playlist**  
  - Type: Spotify node, operation "add item to playlist"  
  - Role: Adds the found Spotify track to the created playlist.  
  - Configuration: Playlist ID dynamically taken from "Create a playlist" output, track ID from search node result.  
  - Inputs: Track URI from If node true branch.  
  - Outputs: Confirmation of addition.  
  - Credentials: Spotify OAuth2 credentials.  
  - Edge Cases: API rate limits, invalid IDs, or network failure.

- **Wait 1 Sec**  
  - Type: Wait node  
  - Role: Delays next iteration by 1 second to respect Spotify rate limits.  
  - Configuration: Delay unit set to seconds, duration 1.  
  - Inputs: After "Add an Item to a playlist".  
  - Outputs: Triggers next iteration of Loop Over Items.  
  - Edge Cases: Workflow execution timeout if too many iterations.

---

### 3. Summary Table

| Node Name              | Node Type                        | Functional Role                          | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                 |
|------------------------|---------------------------------|----------------------------------------|------------------------|-----------------------------|---------------------------------------------------------------------------------------------|
| Telegram Trigger        | Telegram Trigger                 | Entry point, receive Telegram message  | -                      | Create a playlist            | ## 1. Receive & Initialize<br>Triggers on Telegram message → creates empty Spotify playlist → sends "wait for it" reply |
| Create a playlist       | Spotify                         | Create new Spotify playlist             | Telegram Trigger       | AI Agent, Send a text message | ## 1 -1  Create play list first<br>and send back play list url to usr                       |
| Send a text message     | Telegram                        | Send playlist URL back to Telegram user | Create a playlist      | AI Agent                    |                                                                                             |
| AI Agent               | LangChain Agent                  | Generate AI-curated playlist tracks     | Create a playlist, Send a text message | Split Out Array            | ## 2. AI Curation<br>AI agent acts as a pro DJ and outputs 30–50 track suggestions in structured format |
| OpenRouter Chat Model   | LangChain OpenRouter Chat Model | Provides GPT-5 nano model for AI Agent  | AI Agent (ai_languageModel) | AI Agent (ai_languageModel) |                                                                                             |
| Structured Output Parser| LangChain Output Parser          | Parses AI output into structured JSON   | OpenRouter Chat Model  | AI Agent (ai_outputParser)  |                                                                                             |
| Split Out Array         | Split Out                      | Splits AI JSON array into individual tracks | AI Agent              | Loop Over Items             | Splits the 'output' list into individual items                                              |
| Loop Over Items         | Split In Batches               | Processes each track item individually  | Split Out Array        | Search tracks by keyword     | ## 3. Fulfillment Loop<br>Splits the list → searches each track on Spotify → adds found tracks with 1-second delay |
| Search tracks by keyword| Spotify                        | Search Spotify catalog for tracks       | Loop Over Items        | If                         |                                                                                             |
| If                     | If                            | Checks if Spotify search found a track  | Search tracks by keyword| Add an Item to a playlist / Loop Over Items |                                                                                             |
| Add an Item to a playlist| Spotify                       | Adds found track to Spotify playlist    | If (true branch)       | Wait 1 Sec                  |                                                                                             |
| Wait 1 Sec             | Wait                           | Delays next addition by 1 second        | Add an Item to a playlist | Loop Over Items            |                                                                                             |
| Sticky Note            | Sticky Note                    | Workflow overview and instructions      | -                      | -                           | ## How it works<br>This workflow lets users create a brand-new Spotify playlist directly from Telegram in one message.<br>1. User sends a message in Telegram (e.g. "Create a chill house playlist")<br>2. workflow create play list for user (e.g "<username> chill house")<br>3. The AI (acting as a DJ) generates 30–50 track suggestions with perfect 3-column CSV format<br>4. Workflow creates an empty playlist, searches every track on Spotify, and adds the ones it finds<br>Built to respect Spotify API rate limits (1-second delay between adds).<br>## Setup steps<br>1. Connect your Telegram bot (use Telegram Trigger node credentials)<br>2. Connect your Spotify account (use Spotify node credentials)<br>3. (Optional) Change the AI prompt or model in the "AI Agent" node if you want a different music style<br>4. Activate the workflow<br>That’s it — send a message to your bot and get a playlist in <60 seconds! |
| Sticky Note1           | Sticky Note                    | AI Curation summary                     | -                      | -                           | ## 2. AI Curation<br>AI agent acts as a pro DJ and outputs 30–50 track suggestions in structured format |
| Sticky Note2           | Sticky Note                    | Fulfillment loop summary                 | -                      | -                           | ## 3. Fulfillment Loop<br>Splits the list → searches each track on Spotify → adds found tracks with 1-second delay |
| Sticky Note3           | Sticky Note                    | Receive & Initialize summary             | -                      | -                           | ## 1. Receive & Initialize<br>Triggers on Telegram message → creates empty Spotify playlist → sends "wait for it" reply |
| Sticky Note4           | Sticky Note                    | Playlist creation and user notification | -                      | -                           | ## 1 -1  Create play list first<br>and send back play list url to usr                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates.  
   - Use Telegram API credentials connected to your bot.  
   - Set webhook if needed.  

2. **Create Spotify "Create a playlist" node**  
   - Type: Spotify node, resource "playlist", operation "create".  
   - Set playlist name expression:  
     `={{ $('Telegram Trigger').item.json.message.from.username }} {{ $('Telegram Trigger').item.json.message.text }} gen {{ (new Date()).getMinutes().toString().padStart(2, '0') + ':' + (new Date()).getHours().toString().padStart(2, '0') }}`  
   - Use Spotify OAuth2 credentials.  
   - Connect input from Telegram Trigger.  

3. **Create Telegram "Send a text message" node**  
   - Type: Telegram node  
   - Text: `={{ $('Create a playlist').item.json.external_urls.spotify }}`  
   - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
   - Use Telegram API credentials.  
   - Connect input from "Create a playlist".  

4. **Create LangChain AI Agent node**  
   - Type: LangChain Agent node  
   - Text prompt:  
     `=geneaate 25-40 playlist similer to  :{{ $('Telegram Trigger').item.json.message.text }} and return in json`  
   - System message: "you are greatest dj ever lived, return json"  
   - Enable output parser.  
   - Prompt type: "define"  
   - Connect input from "Create a playlist" and "Send a text message".  

5. **Create LangChain OpenRouter Chat Model node**  
   - Type: LangChain OpenRouter Chat Model  
   - Model: "openai/gpt-5-nano"  
   - Response format: JSON object  
   - Use OpenRouter API credentials.  
   - Set as language model for AI Agent node.  

6. **Create LangChain Structured Output Parser node**  
   - Type: Structured Output Parser  
   - Schema (manual): JSON schema defining array of objects with "track" and "artist" string fields.  
   - Connect AI Agent node output parser to this node.  

7. **Create Split Out Array node**  
   - Type: Split Out  
   - Field to split: "output" (the array of tracks from AI output)  
   - Connect input from AI Agent output parser.  

8. **Create Split In Batches node ("Loop Over Items")**  
   - Type: Split In Batches  
   - Default batch size 1 (process one track at a time)  
   - Connect input from Split Out Array node.  

9. **Create Spotify "Search tracks by keyword" node**  
   - Type: Spotify node, resource "track", operation "search"  
   - Query expression: `={{ $json.artist }} {{ $json.track }}`  
   - Limit: 1  
   - Use Spotify OAuth2 credentials.  
   - Connect input from Loop Over Items node (second output).  

10. **Create If node**  
    - Type: If node  
    - Condition: Check if `$json.uri` exists (track found)  
    - Connect input from "Search tracks by keyword".  

11. **Create Spotify "Add an Item to a playlist" node**  
    - Type: Spotify node, resource "playlist", operation "add item"  
    - Playlist ID: `=spotify:playlist:{{ $('Create a playlist').item.json.id }}`  
    - Track ID: `=spotify:track:{{ $json.id }}`  
    - Use Spotify OAuth2 credentials.  
    - Connect input from If node (true output).  

12. **Create Wait node ("Wait 1 Sec")**  
    - Type: Wait node  
    - Wait time: 1 second  
    - Connect input from "Add an Item to a playlist".  

13. **Loop back Wait node output to Loop Over Items node**  
    - Connect Wait 1 Sec node’s output back to Loop Over Items node (second output) to continue processing next batch.  

14. **Connect If node false output**  
    - Connect false output of If node back to Loop Over Items node (first output) to skip adding and continue.  

15. **Finalize connections**  
    - Telegram Trigger → Create a playlist  
    - Create a playlist → Send a text message  
    - Send a text message → AI Agent  
    - AI Agent → Split Out Array → Loop Over Items → Search tracks by keyword → If → Add an Item to a playlist → Wait 1 Sec → Loop Over Items (continue looping)  

16. **Activate workflow**  
    - Ensure all credentials are valid and nodes configured as above.  
    - Activate to listen for Telegram messages and generate playlists on demand.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow was designed to respect Spotify API rate limits by adding a 1-second delay between adding tracks to the playlist. It handles missing tracks gracefully by skipping them. The AI prompt and model can be customized if you want a different music style. Workflow activation and credential setup are required before use.                                                                                                                                                                                                                                           | See Sticky Note in workflow for setup and usage instructions.                                                       |
| AI agent prompt uses a system message to simulate a professional DJ and outputs 25-40 track suggestions formatted as JSON, which is parsed into structured data for further processing.                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Node: AI Agent, OpenRouter Chat Model, Structured Output Parser                                                     |
| The playlist naming convention includes the Telegram username, user’s playlist request text, and current time to ensure uniqueness and clarity.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Node: Create a playlist                                                                                              |
| Spotify and Telegram credentials require OAuth2 authentication with appropriate scopes: Spotify needs playlist modification permissions; Telegram bot needs message and chat access.                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Credential setup is essential for workflow operation.                                                                |
| The workflow is suited for users familiar with n8n automation, Telegram bot usage, and Spotify API integrations. It provides a seamless AI-curated music experience triggered from messaging.                                                                                                                                                                                                                                                                                                                                                                                                                                             | Overall workflow context                                                                                             |

---

**Disclaimer:** The provided text and workflow derive exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data are legal and publicly accessible.