WhatsApp Fact-Checking Bot with Perplexity AI and Twilio

https://n8nworkflows.xyz/workflows/whatsapp-fact-checking-bot-with-perplexity-ai-and-twilio-6842


# WhatsApp Fact-Checking Bot with Perplexity AI and Twilio

### 1. Workflow Overview

This workflow implements a WhatsApp-based fact-checking bot that utilizes Perplexity AI for verifying news or claims submitted by users via WhatsApp messages. It is designed for users seeking quick, reliable verification of information through a conversational interface. The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Listens for incoming WhatsApp messages via a webhook, capturing the user’s query and phone number.
- **1.2 AI Fact-Checking Processing:** Sends the received message text to Perplexity AI with a precise prompt to fact-check and summarize the claim, including citations.
- **1.3 Output Delivery:** Sends the AI-generated verdict, summary, and citations back to the user through WhatsApp using Twilio’s messaging service.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures incoming WhatsApp messages from users via Twilio’s webhook. It extracts the user’s message text and sender information to forward downstream.

**Nodes Involved:**  
- Receive Whatsapp Messages

**Node Details:**  

- **Receive Whatsapp Messages**  
  - *Type & Role:* Webhook node; entry point listening for POST requests at `/whatsapp-inbound`.  
  - *Configuration:* HTTP POST method on path `whatsapp-inbound`. No additional options enabled.  
  - *Key Expressions/Variables:* None explicitly used; raw incoming JSON is passed downstream.  
  - *Input/Output:* No input (trigger node); outputs the entire webhook payload including `Body` (user message), `From` (user phone), and `To` (bot phone).  
  - *Version Requirements:* Uses version 2 of the webhook node.  
  - *Edge Cases / Failures:* Potential failures include webhook security issues (e.g., Twilio signature validation absent here), malformed payloads, or network timeouts.  
  - *Sub-Workflow:* None.

---

#### 2.2 AI Fact-Checking Processing

**Overview:**  
This block takes the user’s query text and sends it to Perplexity AI with a strict system prompt that instructs the AI to analyze, verify with neutral sources, and produce a verdict, summary, and citations.

**Nodes Involved:**  
- Confirm news with Citations

**Node Details:**  

- **Confirm news with Citations**  
  - *Type & Role:* Perplexity AI node; performs AI-powered fact-checking.  
  - *Configuration:*  
    - Model used: `sonar` (Perplexity’s fact-checking model).  
    - Messages:  
      - System message defines role as a cautious, objective fact-checker with stepwise instructions.  
      - User message is dynamically set to the content of the incoming WhatsApp message (`{{$json.body.Body}}`).  
    - Simplify option enabled (likely to return a simplified response structure).  
  - *Key Expressions/Variables:* `{{$json.body.Body}}` extracts the user’s query text from the webhook payload.  
  - *Input/Output:* Input from webhook node; outputs structured JSON including `message` (verdict and summary) and `citations` (array of source URLs).  
  - *Version Requirements:* Version 1 of Perplexity node.  
  - *Edge Cases / Failures:* Possible API authentication errors, network issues, or AI service timeouts; malformed or ambiguous queries may reduce accuracy; missing citations in response are possible.  
  - *Sub-Workflow:* None.

---

#### 2.3 Output Delivery

**Overview:**  
This block sends the AI-generated fact-checking results back to the user via WhatsApp using Twilio’s messaging API, formatting the message with the verdict, summary, and numbered citations.

**Nodes Involved:**  
- Send Summary to whatsapp

**Node Details:**  

- **Send Summary to whatsapp**  
  - *Type & Role:* Twilio node; sends outbound WhatsApp messages.  
  - *Configuration:*  
    - `To` is dynamically extracted by removing the `whatsapp:` prefix from the original sender’s number.  
    - `From` is similarly extracted from the bot’s WhatsApp number.  
    - `Message` composes the text from the AI response:  
      ```
      {{$json.message}}

      ---

      *Sources:*
      {{ $json.citations.map((url, index) => `[${index + 1}] ${url}`).join('\n') }}
      ```  
    - `toWhatsapp` flag enabled to specify WhatsApp messaging.  
  - *Key Expressions/Variables:*  
    - `$('Receive Whatsapp Messages').item.json.body.From.replace('whatsapp:', '')` for recipient.  
    - `$('Receive Whatsapp Messages').item.json.body.To.replace('whatsapp:', '')` for sender.  
    - `$json.message` and `$json.citations` for the AI response content.  
  - *Input/Output:* Input from the Perplexity AI node; outputs Twilio message send status.  
  - *Version Requirements:* Version 1 of Twilio node.  
  - *Edge Cases / Failures:* Possible Twilio API errors (authentication, quota limits, invalid phone numbers), message formatting issues, or network failures.  
  - *Sub-Workflow:* None.

---

#### 2.4 Documentation Notes

**Sticky Notes:**  
- **WhatsApp Gateway:** Explains the webhook as the entry point listening for incoming WhatsApp messages with outputs including message body and sender number.  
- **The Digital Detective:** Describes the Perplexity AI node’s role in analyzing and fact-checking the user query, producing a verdict, summary, and citations.  
- **WhatsApp Reply Service:** Details the Twilio node’s function of sending the AI’s response back to the user via WhatsApp.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                 | Input Node(s)             | Output Node(s)            | Sticky Note                                                                                       |
|---------------------------|---------------------|--------------------------------|---------------------------|---------------------------|-------------------------------------------------------------------------------------------------|
| Receive Whatsapp Messages | Webhook             | Receives incoming WhatsApp messages | None                      | Confirm news with Citations | 📬 **WhatsApp Gateway**: Entry point for incoming WhatsApp messages, outputs message and sender. |
| Confirm news with Citations | Perplexity AI       | Fact-checks user query; generates verdict, summary, citations | Receive Whatsapp Messages | Send Summary to whatsapp   | 🤖 **The Digital Detective**: Performs AI fact-checking with strict instructions and citations.  |
| Send Summary to whatsapp   | Twilio              | Sends AI response back via WhatsApp | Confirm news with Citations | None                      | 📲 **WhatsApp Reply Service**: Sends verdict, summary, and sources back to user via WhatsApp.    |
| Sticky Note               | Sticky Note         | Documentation note              | None                      | None                      | Explains the webhook node functionality.                                                       |
| Sticky Note1              | Sticky Note         | Documentation note              | None                      | None                      | Explains the Perplexity AI node role and output.                                               |
| Sticky Note2              | Sticky Note         | Documentation note              | None                      | None                      | Explains the Twilio node role in replying to users.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Receive Whatsapp Messages"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `whatsapp-inbound`  
   - Purpose: To listen for incoming WhatsApp messages forwarded by Twilio.  
   - No special options needed.  
   - Save and activate webhook to obtain URL for Twilio configuration.  

2. **Create Perplexity AI Node: "Confirm news with Citations"**  
   - Type: Perplexity AI  
   - Model: Set to `sonar` (fact-checking model)  
   - Messages:  
     - System message: Provide the exact prompt that instructs the AI to:  
       1) Analyze user query,  
       2) Search multiple neutral sources,  
       3) Provide a verdict from the set {✅ Likely True, ❌ Likely False, ⚠️ Misleading, ❓ Unverified},  
       4) Summarize findings neutrally in 2-3 sentences,  
       5) Include citations.  
     - User message: Use expression to insert the incoming WhatsApp message text: `{{$json.body.Body}}`  
   - Enable "Simplify" option if available.  
   - Add Perplexity API credentials (API key and ID).  

3. **Create Twilio Node: "Send Summary to whatsapp"**  
   - Type: Twilio  
   - Set `To` parameter with expression:  
     `={{ $('Receive Whatsapp Messages').item.json.body.From.replace('whatsapp:', '') }}`  
   - Set `From` parameter with expression:  
     `={{ $('Receive Whatsapp Messages').item.json.body.To.replace('whatsapp:', '') }}`  
   - Message body: Compose message using expressions:  
     ```
     {{$json.message}}

     ---

     *Sources:*
     {{ $json.citations.map((url, index) => `[${index + 1}] ${url}`).join('\n') }}
     ```  
   - Enable WhatsApp messaging (`toWhatsapp` flag).  
   - Add Twilio API credentials (Account SID, Auth Token, WhatsApp-enabled phone number).  

4. **Connect Nodes:**  
   - Connect output of "Receive Whatsapp Messages" to input of "Confirm news with Citations".  
   - Connect output of "Confirm news with Citations" to input of "Send Summary to whatsapp".  

5. **Set Credentials:**  
   - Configure and verify Perplexity API credentials in n8n credentials settings.  
   - Configure Twilio credentials with WhatsApp permissions and phone numbers.  

6. **Deploy and Test:**  
   - Deploy the workflow.  
   - Configure Twilio WhatsApp sandbox or phone number to forward incoming WhatsApp messages to the webhook URL provided by the "Receive Whatsapp Messages" node.  
   - Test by sending WhatsApp messages and verify the AI-generated fact-checking reply.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                           |
|------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| The system prompt for Perplexity AI is carefully designed to ensure objective, neutral fact-checking with source citations to build trust. | Core component of AI fact-checking logic.                                 |
| Twilio’s WhatsApp messaging service requires prior setup of a WhatsApp-enabled number or sandbox environment with proper credentials.    | Twilio account and WhatsApp sandbox configuration.                        |
| The workflow assumes the incoming webhook request structure matches Twilio’s standard WhatsApp message webhook payload.                 | Important for correct data extraction from webhook payload.              |
| Sticky notes in the workflow provide helpful documentation and can be referenced for understanding node roles and flow.                 | Internal workflow documentation.                                          |

---

**Disclaimer:** The text provided is exclusively generated from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.