Create Personalized City Trip Itineraries with Telegram Bot & GPT-4o

https://n8nworkflows.xyz/workflows/create-personalized-city-trip-itineraries-with-telegram-bot---gpt-4o-9708


# Create Personalized City Trip Itineraries with Telegram Bot & GPT-4o

### 1. Workflow Overview

This workflow, titled **Create Personalized City Trip Itineraries with Telegram Bot & GPT-4o**, is designed to provide customized multi-day travel itineraries for any city through conversational interaction on Telegram. It leverages the GPT-4o language model to generate personalized travel plans based on user input, including optional preset travel styles such as cozy, extreme, family-friendly, budget, luxury, cultural, nature, romantic, and nightlife themes.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception:** Listens for incoming Telegram messages from users.
- **1.2 User Feedback Simulation:** Displays a “typing…” indicator in the Telegram chat to simulate real-time conversation.
- **1.3 Input Routing & Classification:** Determines if the user input contains a preset travel style command or is a generic city request.
- **1.4 Prompt Construction:** Builds the appropriate AI prompt tailored to the detected travel style or default handling.
- **1.5 AI Processing & Response Generation:** Uses GPT-4o to generate the travel itinerary based on the constructed prompt.
- **1.6 Output Delivery:** Sends the generated itinerary message back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures messages sent by users to the Telegram bot.
- **Nodes Involved:**  
  - `Start: Receive Message on Telegram`
- **Node Details:**  
  - Type: **Telegram Trigger**  
  - Role: Listens for new messages from Telegram users to start the workflow.  
  - Configuration: Set to listen for "message" updates only. Requires Telegram API credentials with bot token configured.  
  - Inputs: None (trigger node)  
  - Outputs: Passes incoming message JSON to next node.  
  - Edge Cases: Telegram API connection failures, invalid tokens, webhook issues, malformed messages.  
  - Version Requirements: n8n Telegram Trigger node v1.1 or higher recommended.

#### 2.2 User Feedback Simulation

- **Overview:** Simulates a human-like typing indicator in Telegram chat during processing.
- **Nodes Involved:**  
  - `Show Typing Indicator`
- **Node Details:**  
  - Type: **Telegram Node (sendChatAction)**  
  - Role: Sends “typing...” chat action to Telegram chat to improve UX.  
  - Configuration: Uses chat ID extracted from the incoming Telegram message to send the typing indicator.  
  - Inputs: Receives message from Telegram Trigger node.  
  - Outputs: Passes message forward for routing.  
  - Edge Cases: Telegram API rate limits, failure to fetch chat ID, invalid chat ID.  
  - Version Requirements: Telegram node v1.2 or higher.

#### 2.3 Input Routing & Classification

- **Overview:** Analyzes user input text to detect if it starts with a preset command (e.g., `/cozy`, `/extreme`) and routes accordingly.
- **Nodes Involved:**  
  - `Route by Input Type`
- **Node Details:**  
  - Type: **Switch**  
  - Role: Checks user input text against multiple `startsWith` conditions for preset commands. Routes the flow to corresponding prompt builder nodes.  
  - Configuration: Contains 10 output branches for each preset (`Cozy`, `Extreme`, `Family`, `Budget`, `Luxury`, `Cultural`, `Nature`, `Romantic`, `Nightlife`) plus a default `Any other` branch.  
  - Inputs: Receives message from "Show Typing Indicator" node.  
  - Outputs: Routes to the matching preset prompt node or to a generic prompt node if no match found.  
  - Edge Cases: Unrecognized commands, case sensitivity handled strictly (case-sensitive). Empty or malformed messages.  
  - Version Requirements: Switch node v3.2 or higher.

#### 2.4 Prompt Construction

- **Overview:** Defines the AI prompt content tailored to the requested travel style, or a generic prompt if no preset matched.
- **Nodes Involved:**  
  - `cozy promt`  
  - `extreme promt`  
  - `family promt`  
  - `budget promt`  
  - `luxury promt`  
  - `cultural promt`  
  - `nature promt`  
  - `romantic promt`  
  - `nightlife promt`  
  - `any other promt`
- **Node Details:**  
  - Type: **Set**  
  - Role: Each node sets a JSON object with a `prompt` string customized to the travel style. The prompt instructs the AI to create an n-day city trip plan (default 3 days if unspecified).  
  - Configuration: Raw JSON mode; prompt texts are detailed, including tone, key itinerary components (what to see, where to eat, tips), formatting instructions (bold, italics, line breaks), and use of emojis. Language is matched to user input language.  
  - Inputs: Routed from the Switch node based on detected preset.  
  - Outputs: Forward prompt JSON to the AI node.  
  - Edge Cases: Expression errors if input message text is missing or improperly referenced.  
  - Version Requirements: Set node v3.4 or higher.

#### 2.5 AI Processing & Response Generation

- **Overview:** Sends the constructed prompt to the GPT-4o model and obtains a personalized travel itinerary text.
- **Nodes Involved:**  
  - `Generate personalised answer`
- **Node Details:**  
  - Type: **AI/ML API (OpenAI GPT-4o)**  
  - Role: Makes an API call to OpenAI GPT-4o with the assembled prompt plus the original user message appended.  
  - Configuration: Uses credentials for the AI/ML account with API key. Model set to "openai/gpt-4o". Prompt is dynamically constructed from the previous Set node output combined with the user's original message text from Telegram.  
  - Inputs: Receives prompt JSON from prompt builder nodes.  
  - Outputs: JSON including AI-generated content forwarded to Telegram send message node.  
  - Edge Cases: API rate limits, authentication errors, prompt length or formatting issues, timeouts.  
  - Version Requirements: n8n AI/ML API node supporting GPT-4o, configured with valid API credentials.

#### 2.6 Output Delivery

- **Overview:** Sends the AI-generated trip itinerary back to the user in Telegram chat.
- **Nodes Involved:**  
  - `Send message to Telegram`
- **Node Details:**  
  - Type: **Telegram Node (sendMessage)**  
  - Role: Delivers the final travel itinerary message to the user's Telegram chat.  
  - Configuration: Uses the chat ID of the original message to send the response. Text content is taken from the AI node output (`$json.content`). Attribution is disabled.  
  - Inputs: Receives AI response JSON from the Generate personalised answer node.  
  - Outputs: None (end of flow).  
  - Edge Cases: Telegram API limits, invalid chat ID, message size limits.  
  - Version Requirements: Telegram node v1.2 or higher.

---

### 3. Summary Table

| Node Name                   | Node Type                 | Functional Role                          | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                                  |
|-----------------------------|---------------------------|----------------------------------------|----------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------|
| Start: Receive Message on Telegram | Telegram Trigger          | Receives incoming Telegram messages    | None                             | Show Typing Indicator           | ## 🌍 Travel Idea Generator — Getting Started in 4 Steps<br>1) 🔐 Set Up Access<br>2) Activate Workflow<br>3) Interact via Telegram with presets.            |
| Show Typing Indicator        | Telegram (sendChatAction)  | Shows typing indicator in chat          | Start: Receive Message on Telegram | Route by Input Type             | ## 🔍 Node Overview<br>- Telegram Trigger: listens for user messages<br>- Show Typing Indicator: UX improvement<br>- Route by Input Type: preset detection.    |
| Route by Input Type          | Switch                    | Routes based on user input prefix       | Show Typing Indicator             | Cozy promt, Extreme promt, Family promt, Budget promt, Luxury promt, Cultural promt, Nature promt, Romantic promt, Nightlife promt, Any other promt |                                                                                                              |
| cozy promt                  | Set                       | Builds prompt for cozy travel style     | Route by Input Type              | Generate personalised answer    | ### Different prompts for different cases                                                                     |
| extreme promt               | Set                       | Builds prompt for extreme travel style  | Route by Input Type              | Generate personalised answer    | ### Different prompts for different cases                                                                     |
| family promt                | Set                       | Builds prompt for family-friendly style | Route by Input Type              | Generate personalised answer    | ### Different prompts for different cases                                                                     |
| budget promt                | Set                       | Builds prompt for budget travel style   | Route by Input Type              | Generate personalised answer    | ### Different prompts for different cases                                                                     |
| luxury promt                | Set                       | Builds prompt for luxury travel style   | Route by Input Type              | Generate personalised answer    | ### Different prompts for different cases                                                                     |
| cultural promt              | Set                       | Builds prompt for cultural travel style | Route by Input Type              | Generate personalised answer    | ### Different prompts for different cases                                                                     |
| nature promt                | Set                       | Builds prompt for nature travel style   | Route by Input Type              | Generate personalised answer    | ### Different prompts for different cases                                                                     |
| romantic promt              | Set                       | Builds prompt for romantic travel style | Route by Input Type              | Generate personalised answer    | ### Different prompts for different cases                                                                     |
| nightlife promt             | Set                       | Builds prompt for nightlife travel style| Route by Input Type              | Generate personalised answer    | ### Different prompts for different cases                                                                     |
| any other promt             | Set                       | Builds prompt for generic travel requests| Route by Input Type              | Generate personalised answer    | ### Different prompts for different cases                                                                     |
| Generate personalised answer| AI/ML API (OpenAI GPT-4o) | Calls GPT-4o to generate itinerary text | All prompt Set nodes             | Send message to Telegram        | ## Generates and sends message  to Telegram                                                                   |
| Send message to Telegram    | Telegram (sendMessage)     | Sends AI-generated itinerary to user    | Generate personalised answer     | None                           | ## Generates and sends message  to Telegram                                                                   |
| Sticky Note6                | Sticky Note               | Notes on prompt types                    | None                            | None                           | ### Different prompts for different cases                                                                      |
| Sticky Note7                | Sticky Note               | Highlights message sending function      | None                            | None                           | ## Generates and sends message  to Telegram                                                                   |
| Sticky Note                 | Sticky Note               | Notes on prompt templates                 | None                            | None                           | ## Prompts for different request templates                                                                     |
| Sticky Note - Getting Started | Sticky Note               | Workflow setup and usage instructions    | None                            | None                           | ## 🌍 Travel Idea Generator — Getting Started in 4 Steps<br>Includes Telegram and AI key setup, activation, and interaction commands.                           |
| Sticky Note - Node Overview | Sticky Note               | Summary of nodes and their roles          | None                            | None                           | ## 🔍 Node Overview<br>Summary of main nodes and their functions                                               |
| Sticky Note - Example       | Sticky Note               | Sample input/output demonstration          | None                            | None                           | ## 🟡 Example:<br>Shows example Telegram input and expected output format                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**
   - Name: `Start: Receive Message on Telegram`
   - Type: Telegram Trigger
   - Configure to listen for: `message` updates
   - Set credentials with your Telegram bot token (from BotFather).
   - This node starts the workflow when a user sends a message.

2. **Add Telegram Node for Typing Indicator:**
   - Name: `Show Typing Indicator`
   - Type: Telegram (sendChatAction)
   - Operation: `sendChatAction`
   - Chat ID: `={{ $json["message"]["chat"]["id"] }}`
   - Credentials: Use the same Telegram API credentials
   - Connect input from `Start: Receive Message on Telegram`.

3. **Add Switch Node for Input Routing:**
   - Name: `Route by Input Type`
   - Type: Switch
   - Add 10 outputs with rules checking if the input text starts with the following commands (case-sensitive):
     - `/cozy`
     - `/extreme`
     - `/family`
     - `/budget`
     - `/luxury`
     - `/cultural`
     - `/nature`
     - `/romantic`
     - `/nightlife`
     - A default `Any other` output for unmatched inputs.
   - Use expression: `{{$json["message"]["text"]}}` for the left value in conditions.
   - Connect input from `Show Typing Indicator`.

4. **Add Set Nodes for Each Prompt Type:**
   - Create 10 Set nodes named as per travel styles:
     - `cozy promt`, `extreme promt`, `family promt`, `budget promt`, `luxury promt`, `cultural promt`, `nature promt`, `romantic promt`, `nightlife promt`, `any other promt`.
   - Each node should:
     - Use Raw mode.
     - Output JSON with a single key: `"prompt"`.
     - The prompt string must instruct the AI to create a multi-day travel plan matching the style, including:
       - Language matching user input.
       - Default to 3 days if unspecified.
       - Sections for what to see, where to eat, tips.
       - Use formatting suitable for Telegram (bold, italics, line breaks).
       - Include emojis as per style.
     - Example for `cozy promt`:
       ```json
       {
         "prompt": "You are a travel expert and personal guide. Based on the user's request, create a n-day trip plan for the specified city. Write in the language in which the request was made If the user has not specified the number of days, set it to 3 days, focusing on a cozy, relaxing experience 🛋️. Include charming cafes, small museums, scenic streets, and relaxed activities. Presentation style: warm, friendly, calm, with emojis where appropriate. Format for Telegram with bold, italics, line breaks.\nDay 1:\nWhat to see: …\nWhere to eat: …\nTips: …\nDay 2: …\nDay 3: …"
       }
       ```
   - Connect each output of `Route by Input Type` to the matching Set node.

5. **Add AI/ML API Node for Text Generation:**
   - Name: `Generate personalised answer`
   - Type: AI/ML API (OpenAI GPT-4o)
   - Credentials: Configure with your OpenAI API key credentials.
   - Model: `openai/gpt-4o`
   - Prompt parameter: Use expression to combine prompt from Set node and original message text:
     ```
     ={{ $json.prompt }}

     Message:
     {{ $('Start: Receive Message on Telegram').item.json.message.text }}
     ```
   - Connect all Set nodes to this node.

6. **Add Telegram Send Message Node:**
   - Name: `Send message to Telegram`
   - Type: Telegram (sendMessage)
   - Text: `={{ $json.content }}`
   - Chat ID: `={{ $('Start: Receive Message on Telegram').item.json.message.chat.id }}`
   - Additional fields: Disable append attribution.
   - Credentials: Use Telegram API credentials.
   - Connect input from `Generate personalised answer`.

7. **Connect Workflow:**
   - Start: `Start: Receive Message on Telegram` → `Show Typing Indicator`
   - `Show Typing Indicator` → `Route by Input Type`
   - `Route by Input Type` → respective prompt Set nodes
   - Each prompt Set node → `Generate personalised answer`
   - `Generate personalised answer` → `Send message to Telegram`

8. **Activate Workflow:**
   - Ensure all credentials are properly set.
   - Turn on workflow via the editor's active switch.
   - Test interaction by sending commands to your Telegram bot as per presets or free text.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Telegram bot token must be obtained from BotFather and inserted into Telegram Trigger and Send Message nodes credentials.                                                                                                           | Telegram Bot API (https://core.telegram.org/bots/api)                                           |
| OpenAI API key required for GPT-4o usage; enter it into AI/ML API node credentials.                                                                                                                                                  | OpenAI API (https://platform.openai.com/account/api-keys)                                      |
| User can specify presets by prefixing messages with `/cozy`, `/extreme`, etc., to tailor travel plans. Without prefix, the bot uses a general prompt.                                                                              | Workflow Sticky Note - Getting Started                                                          |
| The prompts include explicit formatting instructions for Telegram using markdown-like syntax (bold, italics, line breaks) to enhance message presentation.                                                                          | See prompt texts in Set nodes                                                                    |
| The workflow simulates typing to improve user experience and avoid perceived delays.                                                                                                                                               | Node: Show Typing Indicator                                                                     |
| Example input and output provided in sticky notes illustrate typical usage and expected responses.                                                                                                                                  | Sticky Note - Example                                                                            |
| Strict case-sensitive matching for preset commands means inputs like `/Cozy` will not match; users must use lowercase as specified.                                                                                                | Switch node configuration                                                                       |
| Ensure the n8n instance has internet access and correct webhook URL setup for Telegram triggers to function properly.                                                                                                              | n8n webhook and Telegram bot setup guides                                                       |
| The workflow assumes 3 days as a default trip length if not specified by the user in their message.                                                                                                                                 | Included in prompt instructions                                                                 |

---

**Disclaimer:** The text analyzed and documented here originates exclusively from an automated workflow created with n8n, a no-code automation platform. It strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.