Classify Lead Sentiment with Google Gemini and Send WhatsApp Responses via Typeform & Supabase

https://n8nworkflows.xyz/workflows/classify-lead-sentiment-with-google-gemini-and-send-whatsapp-responses-via-typeform---supabase-4322


# Classify Lead Sentiment with Google Gemini and Send WhatsApp Responses via Typeform & Supabase

### 1. Workflow Overview

This workflow automates lead sentiment classification and response messaging using AI and cloud services. It targets sales, onboarding, and support teams who want to prioritize leads based on sentiment derived from their input messages.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception:** Captures new lead responses via a Typeform webhook.
- **1.2 Data Preparation:** Prepares and normalizes lead data for analysis.
- **1.3 Sentiment Classification:** Uses Google Gemini AI or other configured AI models to classify lead message sentiment as Positive, Neutral, or Negative.
- **1.4 Lead Storage:** Stores the lead data in Supabase according to sentiment category: Hot (Positive), Neutral, or Cold (Negative).
- **1.5 Response Messaging:** Combines stored lead data and sends a personalized WhatsApp message confirming receipt and providing feedback.
- **1.6 Supporting Elements:** Sticky notes with descriptions and prompt suggestions to guide users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures new lead data submitted via Typeform using a webhook.

- **Nodes Involved:**  
  - Receive New Lead (Typeform)

- **Node Details:**  
  - **Receive New Lead (Typeform)**  
    - Type: Webhook  
    - Role: Entry point to receive new lead submissions via HTTP POST  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `/lead-webhook`  
      - No additional options configured  
    - Input: External HTTP POST requests from Typeform on new lead submission  
    - Output: JSON containing lead data fields such as message text  
    - Potential Failures:  
      - Webhook not reachable or misconfigured path  
      - Missing expected fields in incoming payload

#### 2.2 Data Preparation

- **Overview:**  
  Normalizes and prepares lead data for AI sentiment classification.

- **Nodes Involved:**  
  - Prepare Lead Data

- **Node Details:**  
  - **Prepare Lead Data**  
    - Type: Set  
    - Role: Adjusts or enriches incoming JSON data before passing it to AI nodes  
    - Configuration: Empty options, likely used for future field mappings or default value settings  
    - Input: Output from webhook node (lead data)  
    - Output: Same data passed forward unaltered in current config  
    - Potential Failures: Expression errors if fields referenced later are missing

#### 2.3 Sentiment Classification

- **Overview:**  
  Classifies the sentiment of the lead message using AI models.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Classify Sentiment (Gemini or other ai model)

- **Node Details:**  
  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Provides AI language model capability for sentiment analysis  
    - Configuration: Default options, no prompt or input text configured here (likely used as a language model resource)  
    - Input: None directly; connected as ai_languageModel resource for next node  
    - Output: AI model instance passed to sentiment classification node  
    - Potential Failures: Authentication or quota issues with Google Gemini API  

  - **Classify Sentiment (Gemini or other ai model)**  
    - Type: LangChain Sentiment Analysis  
    - Role: Performs sentiment classification on lead message text  
    - Configuration:  
      - Input Text Expression: `{{$json["message"] || $json["mensagem"] || $json["resposta"]}}`  
      - Uses the AI model from Google Gemini Chat Model node  
    - Input: Lead message data from Prepare Lead Data node  
    - Output: Branches to three nodes based on sentiment: Hot, Neutral, Cold  
    - Potential Failures:  
      - Missing message fields in input JSON  
      - AI model errors or latency  
      - Unexpected sentiment output format

#### 2.4 Lead Storage

- **Overview:**  
  Stores leads into Supabase database categorized by sentiment.

- **Nodes Involved:**  
  - Store Hot Lead  
  - Store Neutral Lead  
  - Store Cold Lead

- **Node Details:**  
  - **Store Hot Lead**  
    - Type: Supabase  
    - Role: Stores leads classified as “hot” (positive sentiment)  
    - Configuration: Uses Supabase credentials (not exposed)  
    - Input: Leads classified positive from sentiment node  
    - Output: Passes data to Combine Lead Data node  
    - Potential Failures: Authentication errors, network issues, data validation errors in Supabase  

  - **Store Neutral Lead**  
    - Type: Supabase  
    - Role: Stores leads classified as “neutral”  
    - Configuration: Same as above  
    - Input: Leads classified neutral  
    - Output: Passes data to Combine Lead Data node  
    - Potential Failures: Same as Store Hot Lead  

  - **Store Cold Lead**  
    - Type: Supabase  
    - Role: Stores leads classified as “cold” (negative sentiment)  
    - Configuration: Same as above  
    - Input: Leads classified negative  
    - Output: Passes data to Combine Lead Data node  
    - Potential Failures: Same as Store Hot Lead  

#### 2.5 Response Messaging

- **Overview:**  
  Combines stored lead data and sends a WhatsApp message through official API.

- **Nodes Involved:**  
  - Combine Lead Data  
  - Send WhatsApp Message

- **Node Details:**  
  - **Combine Lead Data**  
    - Type: Merge  
    - Role: Merges outputs from the three Supabase storage nodes into one unified payload  
    - Configuration: Number of inputs set to 3 (hot, neutral, cold leads)  
    - Input: Data from Store Hot Lead, Store Neutral Lead, Store Cold Lead  
    - Output: Combined data forwarded to WhatsApp message node  
    - Potential Failures: Data mismatch or missing inputs if one storage node fails  

  - **Send WhatsApp Message**  
    - Type: WhatsApp  
    - Role: Sends personalized WhatsApp confirmation messages to leads  
    - Configuration:  
      - Operation: "send"  
      - Uses official WhatsApp Cloud API (credentials not shown)  
    - Input: Combined lead data from Merge node  
    - Output: None further; end of workflow for a lead  
    - Potential Failures:  
      - API authentication or quota issues  
      - Invalid phone number formatting  
      - Message content errors  

#### 2.6 Supporting Elements

- **Overview:**  
  Provides user guidance and prompt suggestions via sticky notes.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  
  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Describes the workflow purpose and detailed explanation  
    - Content Highlights:  
      - Automates lead classification and WhatsApp response  
      - Explains each main step of the workflow  
      - Target user scenarios (sales, onboarding, support)  

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Suggests an AI prompt to classify sentiment  
    - Content:  
      - Prompt: Classify sentiment of message as Positive, Neutral, or Negative  
      - Example usage: `"{{$json["message"]}}"`

---

### 3. Summary Table

| Node Name                    | Node Type                             | Functional Role                          | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                                      |
|------------------------------|-------------------------------------|----------------------------------------|----------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Receive New Lead (Typeform)   | Webhook                             | Capture incoming leads via webhook     | -                                | Prepare Lead Data                  |                                                                                                                                 |
| Prepare Lead Data             | Set                                 | Prepare and normalize lead data        | Receive New Lead (Typeform)       | Classify Sentiment (Gemini or other ai model) |                                                                                                                                 |
| Google Gemini Chat Model      | LangChain Google Gemini Chat Model  | Provide AI model resource for sentiment| -                                | Classify Sentiment (Gemini or other ai model) |                                                                                                                                 |
| Classify Sentiment (Gemini or other ai model) | LangChain Sentiment Analysis         | Classify lead sentiment                 | Prepare Lead Data, Google Gemini Chat Model | Store Hot Lead, Store Neutral Lead, Store Cold Lead                    |                                                                                                                                 |
| Store Hot Lead               | Supabase                            | Store leads with positive sentiment    | Classify Sentiment                | Combine Lead Data                 |                                                                                                                                 |
| Store Neutral Lead           | Supabase                            | Store leads with neutral sentiment     | Classify Sentiment                | Combine Lead Data                 |                                                                                                                                 |
| Store Cold Lead              | Supabase                            | Store leads with negative sentiment    | Classify Sentiment                | Combine Lead Data                 |                                                                                                                                 |
| Combine Lead Data            | Merge                              | Combine stored lead data from categories| Store Hot Lead, Store Neutral Lead, Store Cold Lead | Send WhatsApp Message             |                                                                                                                                 |
| Send WhatsApp Message        | WhatsApp                           | Send personalized WhatsApp message     | Combine Lead Data                | -                                 |                                                                                                                                 |
| Sticky Note                  | Sticky Note                       | Describes workflow purpose and details | -                                | -                               | ## Lead Sentiment Qualifier – Classify incoming leads using AI and reply via WhatsApp  Short Description: Automatically classify leads from a Typeform based on sentiment using Google Gemini. Store them in Supabase by category (hot, neutral, cold) and send personalized WhatsApp responses using the official WhatsApp Cloud API. Full Description: This workflow helps you qualify leads instantly by analyzing the sentiment of their message. New leads are captured through a Typeform webhook The message is processed and analyzed using Google Gemini (sentiment classification: Positive, Neutral or Negative) Depending on the result, the lead is stored in Supabase under the appropriate label (hot, neutral, or cold) A personalized WhatsApp message is sent using the official WhatsApp Cloud API to confirm receipt and provide feedback Ideal for sales teams, onboarding funnels, and support flows that want to prioritize leads based on tone, urgency, or engagement level. |
| Sticky Note1                 | Sticky Note                       | Suggests AI prompt for sentiment classification| -                                | -                               | ## Prompt sugestion  Classify the sentiment of the message below as Positive, Neutral or Negative: "{{$json["message"]}}"      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: Receive New Lead (Typeform)**
   - Node Type: Webhook  
   - HTTP Method: POST  
   - Path: `lead-webhook`  
   - Purpose: Receive new leads from Typeform submissions

2. **Create Set Node: Prepare Lead Data**
   - Node Type: Set  
   - Leave options empty or prepare to map/normalize incoming fields if needed  
   - Connect Webhook output to this node

3. **Create LangChain Google Gemini Chat Model Node**
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Use default options  
   - No inputs; acts as AI model resource node

4. **Create LangChain Sentiment Analysis Node: Classify Sentiment**
   - Node Type: `@n8n/n8n-nodes-langchain.sentimentAnalysis`  
   - Parameters:  
     - Input Text: Expression `{{$json["message"] || $json["mensagem"] || $json["resposta"]}}`  
   - Connect "Prepare Lead Data" main output and "Google Gemini Chat Model" ai_languageModel output as inputs  
   - This node outputs 3 branches for sentiment classification

5. **Create Three Supabase Nodes for Lead Storage**
   - Node Type: Supabase  
   - Credentials: Configure with your Supabase API credentials  
   - Create nodes named:  
     - Store Hot Lead (for positive sentiment)  
     - Store Neutral Lead (for neutral sentiment)  
     - Store Cold Lead (for negative sentiment)  
   - Connect each classification branch from sentiment node to the appropriate Supabase node

6. **Create Merge Node: Combine Lead Data**
   - Node Type: Merge  
   - Parameters: Number of Inputs = 3  
   - Connect outputs from all three Supabase nodes into this merge node

7. **Create WhatsApp Node: Send WhatsApp Message**
   - Node Type: WhatsApp  
   - Operation: send  
   - Configure official WhatsApp Cloud API credentials (OAuth2 or API key as per your setup)  
   - Connect output of Merge node to this node  
   - Define message template for personalized confirmation feedback (not detailed in workflow but recommended)

8. **Add Sticky Notes for Documentation**
   - Add one large sticky note describing workflow purpose, usage, and steps  
   - Add a sticky note with the example prompt for sentiment classification:
     - `"Classify the sentiment of the message below as Positive, Neutral or Negative:\n\n\"{{$json["message"]}}\""`

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                            | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow uses Google Gemini AI for sentiment classification, leveraging n8n’s LangChain integration. Ensure you have valid credentials and API access to Google Gemini.                                                                                                                                 | Google Gemini API documentation: https://developers.google.com/ai/gemini                             |
| Supabase nodes require API key and URL credentials configured in n8n credentials manager. The Supabase project should have tables to store leads with fields matching the input data schema.                                                                                                               | Supabase documentation: https://supabase.com/docs                                                    |
| WhatsApp node uses official WhatsApp Cloud API. OAuth2 or token credentials must be configured. Phone numbers must be in E.164 format for message delivery.                                                                                                                                               | WhatsApp Cloud API docs: https://developers.facebook.com/docs/whatsapp/cloud-api                      |
| The workflow expects lead messages under keys like “message”, “mensagem” (Portuguese), or “resposta” (Portuguese for response). Modify expressions if your form uses different field names.                                                                                                               |                                                                                                         |
| Consider handling errors on API calls (Google Gemini, Supabase, WhatsApp) by adding error workflows or retry logic for production usage.                                                                                                                                                                |                                                                                                         |
| For performance, monitor API rate limits and quotas on Google Gemini and WhatsApp Cloud API, especially in high lead volume scenarios.                                                                                                                                                                  |                                                                                                         |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or copyrighted elements. All data handled is legal and public.