Send Voice Calls in seconds: Automate Text-To-Speech using ClickSend API

https://n8nworkflows.xyz/workflows/send-voice-calls-in-seconds--automate-text-to-speech-using-clicksend-api-3072


# Send Voice Calls in seconds: Automate Text-To-Speech using ClickSend API

### 1. Workflow Overview

This workflow automates the sending of **text-to-speech (TTS) voice calls** via the ClickSend API. It is designed for scenarios such as notifications, reminders, or any use case requiring automated voice communication. The workflow accepts user input through a form, including the message text, recipient phone number, voice type, and language, then triggers a voice call that reads the message aloud.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Captures user input via a form submission.
- **1.2 Voice Call Dispatch**: Sends the TTS voice call request to the ClickSend API.
- **1.3 User Guidance and Setup**: Provides instructions and setup notes for users to configure and use the workflow properly.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures the necessary input parameters from the user through a web form. It collects the text message, recipient phone number, voice type, and language for the voice call.

- **Nodes Involved:**  
  - `On form submission`

- **Node Details:**  
  - **Node Name:** On form submission  
  - **Type:** Form Trigger  
  - **Technical Role:** Entry point that triggers the workflow when a user submits the form.  
  - **Configuration:**  
    - Form titled "Send Voice Message" with four required fields:  
      - **Body:** Textarea for the message (max 600 characters).  
      - **To:** Recipient phone number, including international prefix (e.g., +39xxxxxxxxxx).  
      - **Voice:** Dropdown with options "male" or "female".  
      - **Lang:** Dropdown with multiple language options (e.g., en-us, it-it, fr-fr, etc.).  
    - Webhook ID assigned for external access.  
  - **Key Expressions/Variables:**  
    - `$json.Body`, `$json.To`, `$json.Voice`, `$json.Lang` are extracted from form submission data.  
  - **Input/Output Connections:**  
    - No input connections (trigger node).  
    - Output connected to the `Send Voice` node.  
  - **Version-Specific Requirements:**  
    - Requires n8n version supporting Form Trigger node (v0.154.0+ recommended).  
  - **Potential Failures:**  
    - Missing or invalid form fields (e.g., empty Body or invalid phone number format).  
    - Webhook connectivity issues or unauthorized access.  
  - **Sub-workflow Reference:** None.

---

#### 1.2 Voice Call Dispatch

- **Overview:**  
  This block sends the voice call request to the ClickSend API using the data collected from the form. It constructs a JSON payload with the message, recipient, voice, language, and enables machine detection.

- **Nodes Involved:**  
  - `Send Voice`

- **Node Details:**  
  - **Node Name:** Send Voice  
  - **Type:** HTTP Request  
  - **Technical Role:** Sends a POST request to the ClickSend voice API endpoint to initiate the TTS voice call.  
  - **Configuration:**  
    - URL: `https://rest.clicksend.com/v3/voice/send`  
    - Method: POST  
    - Authentication: HTTP Basic Auth using ClickSend credentials (username and API key as password).  
    - Headers: `Content-Type: application/json`  
    - Body (JSON):  
      ```json
      {
        "messages": [
          {
            "source": "n8n",
            "body": "{{ $json.Body }}",
            "to": "{{ $json.To }}",
            "voice": "{{ $json.Voice }}",
            "lang": "{{ $json.Lang }}",
            "machine_detection": 1
          }
        ]
      }
      ```  
    - The expressions interpolate form data dynamically.  
  - **Key Expressions/Variables:**  
    - Uses `{{ $json.Body }}`, `{{ $json.To }}`, `{{ $json.Voice }}`, and `{{ $json.Lang }}` from the incoming data.  
  - **Input/Output Connections:**  
    - Input from `On form submission` node.  
    - No further output nodes connected (end of workflow).  
  - **Version-Specific Requirements:**  
    - Requires n8n version supporting HTTP Request node with HTTP Basic Auth (v0.100.0+).  
  - **Potential Failures:**  
    - Authentication errors due to invalid credentials.  
    - API rate limits or quota exceeded.  
    - Network timeouts or connectivity issues.  
    - Invalid phone number or unsupported language/voice parameters causing API errors.  
  - **Sub-workflow Reference:** None.

---

#### 1.3 User Guidance and Setup

- **Overview:**  
  This block contains sticky notes that provide users with setup instructions, usage guidance, and contextual information about the workflow.

- **Nodes Involved:**  
  - `Sticky Note`  
  - `Sticky Note1`  
  - `Sticky Note2`

- **Node Details:**  
  - **Node Name:** Sticky Note  
    - **Type:** Sticky Note  
    - **Role:** Provides step 1 instructions for registering on ClickSend and configuring credentials.  
    - **Content Highlights:**  
      - Link to ClickSend registration: https://clicksend.com/?u=586989  
      - Instructions to create Basic Auth credentials in the `Send Voice` node.  
  - **Node Name:** Sticky Note1  
    - **Type:** Sticky Note  
    - **Role:** Provides step 2 instructions on submitting the form and receiving the voice call.  
  - **Node Name:** Sticky Note2  
    - **Type:** Sticky Note  
    - **Role:** Describes the overall purpose of the workflow as an automation for TTS voice calls using ClickSend API.  
  - **Input/Output Connections:**  
    - None (informational only).  
  - **Potential Failures:**  
    - None (informational nodes).  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name          | Node Type       | Functional Role                      | Input Node(s)       | Output Node(s) | Sticky Note                                                                                              |
|--------------------|-----------------|------------------------------------|---------------------|----------------|--------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger    | Captures user input via form        | None                | Send Voice     |                                                                                                        |
| Send Voice         | HTTP Request    | Sends TTS voice call request to API | On form submission  | None           |                                                                                                        |
| Sticky Note        | Sticky Note     | Setup instructions for ClickSend   | None                | None           | ## STEP 1<br>[Register here to ClickSend](https://clicksend.com/?u=586989) and obtain your API Key and 2 € of free credits<br>In the node "Send Voice" create a "Basic Auth" with the username you registered and the API Key provided as your password |
| Sticky Note1       | Sticky Note     | Instructions on form submission use | None                | None           | ## STEP 2<br>Submit the form and you will receive a call to the phone number you entered where the selected voice will tell you the content of the text you wrote. |
| Sticky Note2       | Sticky Note     | Workflow purpose description        | None                | None           | ## Automate text-to-speech voice calls<br>This workflow is a simple yet powerful way to automate text-to-speech voice calls using the ClickSend API. It’s ideal for notifications, reminders, or any scenario where voice communication is needed. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Add a **Form Trigger** node named `On form submission`.  
   - Configure the form with the title "Send Voice Message".  
   - Add the following required fields:  
     - **Body**: Textarea, placeholder "Body (max. 600 chars)".  
     - **To**: Text input, placeholder "+39xxxxxxxxxx".  
     - **Voice**: Dropdown with options "male" and "female".  
     - **Lang**: Dropdown with options:  
       - en-us, it-it, en-au, en-gb, de-de, es-es, fr-fr, is-is, da-dk, nl-nl, pl-pl, pt-br, ru-ru.  
   - Save and note the webhook URL generated for this node.

2. **Create the HTTP Request Node**  
   - Add an **HTTP Request** node named `Send Voice`.  
   - Set the HTTP Method to POST.  
   - Set the URL to `https://rest.clicksend.com/v3/voice/send`.  
   - Under Authentication, select **HTTP Basic Auth**.  
   - Create or select credentials for ClickSend:  
     - Username: Your ClickSend username.  
     - Password: Your ClickSend API Key.  
   - Set Headers:  
     - `Content-Type: application/json`  
   - Set the Body Content Type to JSON.  
   - Use the following JSON body with expressions to map form data:  
     ```json
     {
       "messages": [
         {
           "source": "n8n",
           "body": "{{ $json.Body }}",
           "to": "{{ $json.To }}",
           "voice": "{{ $json.Voice }}",
           "lang": "{{ $json.Lang }}",
           "machine_detection": 1
         }
       ]
     }
     ```  
   - Enable sending the body and headers.

3. **Connect Nodes**  
   - Connect the output of `On form submission` to the input of `Send Voice`.

4. **Add Sticky Notes (Optional but Recommended)**  
   - Add sticky notes with the following content to guide users:  
     - Step 1: Registration and credential setup with ClickSend (include link https://clicksend.com/?u=586989).  
     - Step 2: Instructions on submitting the form and receiving the voice call.  
     - General description of the workflow purpose.

5. **Test the Workflow**  
   - Activate the workflow.  
   - Submit the form via the webhook URL with valid data.  
   - Verify that the recipient phone number receives a voice call reading the submitted text.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Register at ClickSend to obtain API Key and 2 € free credits for testing.                                  | https://clicksend.com/?u=586989                   |
| The workflow uses HTTP Basic Authentication with ClickSend credentials for secure API access.              | Credential setup in `Send Voice` node             |
| The maximum allowed length for the text message is 600 characters to comply with API limits.               | Form field constraint                              |
| Machine detection is enabled to identify if the call is answered by a machine, improving call handling.    | `"machine_detection": 1` in API request body      |
| Supported languages include English (US, AU, GB), Italian, German, Spanish, French, Icelandic, Danish, Dutch, Polish, Portuguese (Brazil), and Russian. | Language dropdown options in form                  |
| This workflow is ideal for automating notifications, reminders, or any scenario requiring voice communication. | Workflow purpose                                   |

---

This documentation provides a complete understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.