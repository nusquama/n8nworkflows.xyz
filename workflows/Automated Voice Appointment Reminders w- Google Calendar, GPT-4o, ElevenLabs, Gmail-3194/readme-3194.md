Automated Voice Appointment Reminders w/ Google Calendar, GPT-4o, ElevenLabs, Gmail

https://n8nworkflows.xyz/workflows/automated-voice-appointment-reminders-w--google-calendar--gpt-4o--elevenlabs--gmail-3194


# Automated Voice Appointment Reminders w/ Google Calendar, GPT-4o, ElevenLabs, Gmail

### 1. Workflow Overview

This workflow automates the generation and delivery of **voice appointment reminders** via email. It targets businesses that rely on scheduled appointments (e.g., medical offices, real estate agencies, service providers) to reduce no-shows and improve client engagement.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Appointment Retrieval:** Initiates the workflow manually or on a schedule and fetches upcoming appointments from Google Calendar.
- **1.2 Message Generation with AI:** Uses GPT-4o-mini via LangChain nodes to create a structured, professional voice reminder script and email subject based on appointment details.
- **1.3 Voice Message Synthesis:** Sends the AI-generated text to ElevenLabs API to produce a natural-sounding MP3 voice file.
- **1.4 Email Delivery:** Attaches the generated audio file and sends it to the client via Gmail.

Supporting nodes include code for file naming and sticky notes with documentation links and API key reminders.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Appointment Retrieval

- **Overview:** This block triggers the workflow either manually or on a schedule and retrieves upcoming appointments from a specified Google Calendar.
- **Nodes Involved:**  
  - When clicking 'Test workflow' (Manual Trigger)  
  - Schedule Trigger  
  - Get Appointments (Google Calendar)

- **Node Details:**

  - **When clicking 'Test workflow'**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of the workflow for testing or on-demand runs.  
    - Configuration: No parameters; triggers workflow immediately when clicked.  
    - Inputs: None  
    - Outputs: Connects to "Get Appointments" node.  
    - Edge Cases: None significant; manual trigger depends on user action.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow at defined intervals (default is every minute due to empty interval object).  
    - Configuration: Default interval (can be customized to daily, hourly, etc.).  
    - Inputs: None  
    - Outputs: Connects to "Get Appointments" node.  
    - Edge Cases: Misconfiguration of interval could cause too frequent or no triggers.

  - **Get Appointments**  
    - Type: Google Calendar  
    - Role: Fetches upcoming calendar events (appointments) from a specified Google Calendar.  
    - Configuration:  
      - Operation: getAll events  
      - Limit: 2 events per run  
      - TimeMax: 2 days from current time (`{{$now.plus({ day: 2 })}}`)  
      - Calendar: Specific calendar email (e.g., "mymail@gmail.com")  
    - Inputs: Trigger nodes ("When clicking 'Test workflow'" or "Schedule Trigger")  
    - Outputs: Sends event data to "create message" node.  
    - Version: 1.3  
    - Edge Cases:  
      - Authentication errors if Google credentials are invalid or expired.  
      - No events found returns empty data, downstream nodes must handle gracefully.  
      - API rate limits or quota exceeded errors.

---

#### 2.2 Message Generation with AI

- **Overview:** This block uses OpenAI GPT-4o-mini via LangChain nodes to generate a structured voice reminder script and a short email subject based on appointment details.
- **Nodes Involved:**  
  - create message (LangChain Chain LLM)  
  - OpenAI Chat Model (LangChain LLM Chat OpenAI)  
  - Structured Output Parser (LangChain Output Parser Structured)

- **Node Details:**

  - **create message**  
    - Type: LangChain Chain LLM  
    - Role: Constructs a prompt with appointment details and requests a structured JSON response containing a voice message script and email subject.  
    - Configuration:  
      - Prompt includes variables: recipient name, appointment time, address, current date.  
      - Message instructs the AI to generate a professional, cordial voice reminder script including sender info and confirmation invitation.  
      - Output parser enabled with a manual JSON schema expecting "message" and "mail_object" strings.  
    - Inputs: Appointment data from "Get Appointments"  
    - Outputs: Structured JSON output to "Generate Voice Reminder"  
    - Version: 1.5  
    - Edge Cases:  
      - AI model may return malformed JSON or unexpected output; parser errors possible.  
      - Network or API errors with OpenAI.  
      - Input data missing or incomplete (e.g., missing location) may affect output quality.

  - **OpenAI Chat Model**  
    - Type: LangChain LLM Chat OpenAI  
    - Role: Provides the GPT-4o-mini model for the "create message" node to generate text.  
    - Configuration: Model set to "gpt-4o-mini" with default options.  
    - Inputs: Connected internally by LangChain nodes; no direct external inputs.  
    - Outputs: Text response to "create message" node.  
    - Version: 1.2  
    - Edge Cases: API key or quota issues; model unavailability.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser Structured  
    - Role: Parses the AI text output into a structured JSON object with defined schema.  
    - Configuration: Manual schema with "message" and "mail_object" string properties.  
    - Inputs: AI text output from "OpenAI Chat Model"  
    - Outputs: Parsed JSON to "create message" node for further processing.  
    - Version: 1.2  
    - Edge Cases: Parsing failures if AI output is malformed or incomplete.

---

#### 2.3 Voice Message Synthesis

- **Overview:** Converts the AI-generated text message into a natural-sounding voice MP3 file using ElevenLabs Text-to-Speech API.
- **Nodes Involved:**  
  - Generate Voice Reminder (HTTP Request)  
  - Change filename (Code)

- **Node Details:**

  - **Generate Voice Reminder**  
    - Type: HTTP Request  
    - Role: Calls ElevenLabs API to synthesize speech from the AI-generated text message.  
    - Configuration:  
      - URL: ElevenLabs TTS endpoint with voice ID "JBFqnCBsd6RMkjVDRZzb"  
      - Method: POST  
      - Authentication: Generic HTTP Custom Auth (ElevenLabs API key)  
      - Body Parameters:  
        - text: AI-generated message (`{{$json.output.message}}`)  
        - model_id: "eleven_multilingual_v2"  
      - Query Parameters: output_format = "mp3_22050_32" (MP3 audio, 22050 Hz, 32 kbps)  
      - Retry on failure enabled  
    - Inputs: Structured JSON from "create message"  
    - Outputs: Binary audio data to "Change filename"  
    - Version: 4.2  
    - Edge Cases:  
      - API key invalid or expired  
      - Network timeouts or API rate limits  
      - Text too long or malformed causing synthesis failure

  - **Change filename**  
    - Type: Code (JavaScript)  
    - Role: Adds a meaningful filename to the binary audio data using the email subject from AI output.  
    - Configuration:  
      - Extracts `mail_object` from input JSON and appends ".mp3" extension  
      - Sets `fileName` property on binary data for downstream email attachment  
    - Inputs: Binary audio from "Generate Voice Reminder"  
    - Outputs: Binary with filename to "Send Voice Reminder"  
    - Version: 2  
    - Edge Cases:  
      - Missing or empty `mail_object` causes filename issues  
      - Binary data missing or corrupted

---

#### 2.4 Email Delivery

- **Overview:** Sends the generated voice reminder MP3 file as an email attachment to the client using Gmail API.
- **Nodes Involved:**  
  - Send Voice Reminder (Gmail)

- **Node Details:**

  - **Send Voice Reminder**  
    - Type: Gmail  
    - Role: Sends an email with the voice reminder attached to the first attendee's email from the appointment.  
    - Configuration:  
      - Recipient: `{{$('Get Appointments').item.json.attendees[0].email}}` (first attendee email)  
      - Subject: AI-generated short email subject (`{{$('create message').item.json.output.mail_object}}`)  
      - Message body: Fixed text "👇 Information for tomorrow 🗣️"  
      - Sender name: "John Carpenter"  
      - Attachments: Binary audio file from "Change filename"  
      - Append Attribution: Disabled  
    - Inputs: Binary audio with filename from "Change filename"  
    - Outputs: None (end node)  
    - Version: 2.1  
    - Edge Cases:  
      - Missing or invalid recipient email  
      - Gmail API authentication errors or quota limits  
      - Attachment size limits exceeded  
      - Email delivery failures (spam filters, invalid addresses)

---

#### 2.5 Supporting Nodes (Sticky Notes)

- **Sticky Note (ElevenLabs API key)**  
  - Content: Reminder and link to obtain ElevenLabs API key: [Elevenlabs](https://try.elevenlabs.io/text-audio)  
  - Positioned near "Generate Voice Reminder" node.

- **Sticky Note1 (Gmail API Credentials)**  
  - Content: Link to Gmail API credential setup documentation: [n8n Gmail Credentials](https://docs.n8n.io/integrations/builtin/credentials/google/)  
  - Positioned near "Send Voice Reminder" node.

- **Sticky Note2 (Calendar API Credentials)**  
  - Content: Link to Google Calendar API credential setup documentation: [n8n Google Credentials](https://docs.n8n.io/integrations/builtin/credentials/google/)  
  - Positioned near "Get Appointments" node.

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                         | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                      |
|-------------------------|-------------------------------------|---------------------------------------|-------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| When clicking 'Test workflow' | Manual Trigger                      | Manual workflow start                  | None                          | Get Appointments             |                                                                                                 |
| Schedule Trigger         | Schedule Trigger                    | Scheduled workflow start               | None                          | Get Appointments             |                                                                                                 |
| Get Appointments         | Google Calendar                    | Fetch upcoming appointments            | When clicking 'Test workflow', Schedule Trigger | create message              | ## Calendar API Credentials  \n**Click here** to view the [documentation](https://docs.n8n.io/integrations/builtin/credentials/google/) and configure your access permissions for the Google Calendar API. |
| create message           | LangChain Chain LLM                | Generate structured voice reminder text and email subject | Get Appointments              | Generate Voice Reminder      |                                                                                                 |
| OpenAI Chat Model        | LangChain LLM Chat OpenAI          | Provides GPT-4o-mini model for text generation | create message (ai_languageModel) | Structured Output Parser     |                                                                                                 |
| Structured Output Parser | LangChain Output Parser Structured | Parses AI output into JSON             | OpenAI Chat Model             | create message               |                                                                                                 |
| Generate Voice Reminder  | HTTP Request                      | Convert text to speech via ElevenLabs | create message                | Change filename              | ## ElevenlabsAPI key\n**Click** to get your Elevenlabs  API key. [Elevenlabs](https://try.elevenlabs.io/text-audio) |
| Change filename          | Code                              | Add filename to binary audio data      | Generate Voice Reminder       | Send Voice Reminder          |                                                                                                 |
| Send Voice Reminder      | Gmail                             | Send email with voice reminder attached | Change filename              | None                        | ## Gmail API Credentials  \n**Click here** to view the [documentation](https://docs.n8n.io/integrations/builtin/credentials/google/) and configure your access permissions for the Google Gmail API. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node named "When clicking 'Test workflow'".
   - Add a **Schedule Trigger** node named "Schedule Trigger" with default or customized interval (e.g., daily at 9 AM).

2. **Add Google Calendar Node:**
   - Add a **Google Calendar** node named "Get Appointments".
   - Set operation to "getAll".
   - Set limit to 2.
   - Set `timeMax` to `{{$now.plus({ day: 2 })}}` to fetch events up to 2 days ahead.
   - Select the calendar email (e.g., "mymail@gmail.com").
   - Configure Google Calendar OAuth2 credentials (see [Google Calendar API docs](https://docs.n8n.io/integrations/builtin/credentials/google/)).
   - Connect both trigger nodes to "Get Appointments".

3. **Add LangChain Chain LLM Node:**
   - Add a **Chain LLM** node named "create message".
   - Configure prompt text with variables:  
     ```
     name: {{ $json.summary }}
     time: {{ $json.start.dateTime }}
     address: {{ $json.location }}
     Today's date: {{ $now }}
     ```
   - Add a message instructing the AI to generate a structured JSON with "message" (voice script) and "mail_object" (email subject), including recipient name, appointment time, address, sender info, and confirmation sentence.
   - Enable output parser with manual JSON schema:  
     ```
     {
       "type": "object",
       "properties": {
         "message": { "type": "string" },
         "mail_object": { "type": "string" }
       }
     }
     ```
   - Connect "Get Appointments" output to this node.

4. **Add LangChain OpenAI Chat Model Node:**
   - Add a **LangChain LLM Chat OpenAI** node named "OpenAI Chat Model".
   - Set model to "gpt-4o-mini".
   - Connect this node to the "create message" node’s AI language model input.

5. **Add LangChain Structured Output Parser Node:**
   - Add a **LangChain Output Parser Structured** node named "Structured Output Parser".
   - Use the same manual JSON schema as above.
   - Connect "OpenAI Chat Model" output to this node.
   - Connect this parser output back to "create message" node’s AI output parser input.

6. **Add HTTP Request Node for ElevenLabs:**
   - Add an **HTTP Request** node named "Generate Voice Reminder".
   - Set method to POST.
   - URL: `https://api.elevenlabs.io/v1/text-to-speech/JBFqnCBsd6RMkjVDRZzb`
   - Authentication: Generic HTTP Custom Auth with ElevenLabs API key.
   - Body parameters:  
     - `text`: `={{ $json.output.message }}`  
     - `model_id`: "eleven_multilingual_v2"  
   - Query parameters:  
     - `output_format`: "mp3_22050_32"  
   - Enable "Retry on Fail".
   - Connect "create message" output to this node.

7. **Add Code Node to Rename File:**
   - Add a **Code** node named "Change filename".
   - Use JavaScript code to set the binary data filename based on `mail_object` from AI output:  
     ```js
     const mailObject = $input.first().json.output.mail_object;
     const fileName = `${mailObject}.mp3`;

     return items.map(item => {
       if (item.binary && item.binary.data) {
         item.binary.data.fileName = fileName;
       }
       return item;
     });
     ```
   - Connect "Generate Voice Reminder" output to this node.

8. **Add Gmail Node to Send Email:**
   - Add a **Gmail** node named "Send Voice Reminder".
   - Set "Send To" to: `={{ $('Get Appointments').item.json.attendees[0].email }}` (first attendee email)  
   - Subject: `={{ $('create message').item.json.output.mail_object }}`  
   - Message: Fixed text "👇 Information for tomorrow 🗣️"  
   - Sender Name: "John Carpenter"  
   - Attachments: Attach binary data from "Change filename" node.  
   - Disable append attribution.  
   - Configure Gmail OAuth2 credentials (see [Gmail API docs](https://docs.n8n.io/integrations/builtin/credentials/google/)).  
   - Connect "Change filename" output to this node.

9. **Add Sticky Notes (Optional):**
   - Add sticky notes near relevant nodes with links and reminders for API key setup:
     - ElevenLabs API key link near "Generate Voice Reminder".
     - Google Calendar API credentials link near "Get Appointments".
     - Gmail API credentials link near "Send Voice Reminder".

10. **Test Workflow:**
    - Run manually via "When clicking 'Test workflow'" or wait for scheduled trigger.
    - Verify appointments are fetched, messages generated, voice files created, and emails sent.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The LangChain Community node used here only works on self-hosted n8n instances, not n8n Cloud.   | Important deployment note for users planning to use this workflow.                               |
| ElevenLabs API key required for voice synthesis. Sign up and get your key here:                   | [https://try.elevenlabs.io/text-audio](https://try.elevenlabs.io/text-audio)                     |
| Google Calendar and Gmail API credentials require OAuth2 setup with appropriate scopes.          | See n8n documentation: [Google Credentials](https://docs.n8n.io/integrations/builtin/credentials/google/) |
| Workflow designed for businesses needing automated voice reminders to reduce no-shows and improve client engagement. | Use cases include medical offices, real estate, service providers, legal and financial services. |
| Example voice reminder script format provided in the AI prompt for professional and cordial tone.| Ensures consistent, clear, and engaging voice messages.                                         |
| Video or image preview included in original workflow description (not embedded here).             | Visual aid for users importing or building the workflow.                                         |
| Workflow author and credits: Phil | Inforeole                                                      | [https://inforeole.fr](https://inforeole.fr)                                                    |

---

This documentation provides a complete, structured understanding of the Automated Voice Appointment Reminders workflow, enabling reproduction, modification, and troubleshooting for advanced users and automation agents alike.