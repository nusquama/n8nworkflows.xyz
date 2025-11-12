Call Center Analytics with Dual-AI Verification using DeepSeek Models

https://n8nworkflows.xyz/workflows/call-center-analytics-with-dual-ai-verification-using-deepseek-models-5658


# Call Center Analytics with Dual-AI Verification using DeepSeek Models

---

### 1. Workflow Overview

This workflow, titled **"Call Center Analytics with Dual-AI Verification using DeepSeek Models,"** is designed to analyze call center CRM data using a dual-layer AI verification approach. The primary purpose is to generate a detailed analytical report from call center agent metrics and then verify the quality and accuracy of this report through a secondary AI model before sending the data to a callback API.

**Target Use Cases:**
- Automated CRM data analysis for call centers.
- Quality assurance of AI-generated analytical reports.
- Integration with external systems via callbacks.
- Testing and verification of AI outputs using distinct AI models.

**Logical Blocks:**

- **1.1 Input Reception**
  - Handles data input from either a webhook or manual trigger with example data.
  
- **1.2 Primary AI Analysis**
  - Processes CRM data to generate an analytical report using a chain LLM node enhanced by DeepSeek reasoning.

- **1.3 Secondary AI Verification**
  - Evaluates the generated report for accuracy, completeness, and insight quality using a separate DeepSeek AI chat model.

- **1.4 Output Dispatch**
  - Sends the verified report data to an external callback API.

- **1.5 Workflow Control and Testing**
  - Manual trigger and example data nodes to facilitate workflow testing.

- **1.6 Documentation and User Guidance**
  - Sticky notes providing instructions, comments, and contact details.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block manages how data enters the workflow, either via a webhook for real-time integration or a manual trigger for testing with predefined example data.

**Nodes Involved:**  
- Webhook  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Example data  

**Node Details:**

- **Webhook**  
  - *Type*: Webhook (HTTP POST/GET)  
  - *Role*: Receives incoming HTTP requests containing CRM data.  
  - *Configuration*: Path set to a unique webhook ID, supports POST and GET methods, allowing flexible integration.  
  - *Inputs/Outputs*: Output connected to the "Report" node.  
  - *Edge Cases*: Possible failure if webhook URL is not publicly accessible, or if invalid/malformed data is sent.

- **When clicking ‘Test workflow’**  
  - *Type*: Manual Trigger  
  - *Role*: Allows manual initiation of the workflow for testing purposes.  
  - *Configuration*: No parameters; triggers the flow when clicked.  
  - *Inputs/Outputs*: Outputs to "Example data."  
  - *Edge Cases*: None significant; manual user action required.

- **Example data**  
  - *Type*: Code Node  
  - *Role*: Provides sample CRM data for testing without external input.  
  - *Configuration*: Returns an array of objects with call center agent metrics (productivity, leads, upsell, etc.).  
  - *Inputs/Outputs*: No inputs; outputs JSON data to "Report."  
  - *Edge Cases*: Static data; no dynamic errors expected.

---

#### 1.2 Primary AI Analysis

**Overview:**  
Generates a detailed analytical report on call center data using a chain LLM node with support from DeepSeek's reasoning model to enhance insight quality.

**Nodes Involved:**  
- Report  
- DeepSeek Reasonning  

**Node Details:**

- **Report**  
  - *Type*: Chain LLM (LangChain)  
  - *Role*: Analyzes CRM data and produces a Markdown-formatted report with actionable insights.  
  - *Configuration*:  
    - Prompt includes detailed instructions to analyze lead conversion rates, upsell, and agent rankings.  
    - Input data is stringified JSON from either webhook or example data node.  
    - Output is Markdown text.  
  - *Inputs/Outputs*: Receives JSON from "Webhook" or "Example data"; outputs to "Recheck" node.  
  - *Edge Cases*: Potential issues if input JSON is malformed or incomplete; prompt requires well-structured data.  
  - *Version Specifics*: Uses LangChain module v1.6.

- **DeepSeek Reasonning**  
  - *Type*: DeepSeek LLM Chat (Reasoner model)  
  - *Role*: Provides enhanced reasoning capabilities to the "Report" node’s chain LLM for improved analysis.  
  - *Configuration*: Model set to "deepseek-reasoner" with no additional options.  
  - *Inputs/Outputs*: Serves as language model backend for "Report."  
  - *Credentials*: Authenticates via DeepSeek API credentials.  
  - *Edge Cases*: API key invalid or expired, network failures, or DeepSeek service downtime.

---

#### 1.3 Secondary AI Verification

**Overview:**  
Validates the AI-generated report by cross-checking factual accuracy, completeness, insight quality, and format compliance using a separate DeepSeek chat model.

**Nodes Involved:**  
- Recheck  
- DeepSeek Chat  

**Node Details:**

- **Recheck**  
  - *Type*: Chain LLM (LangChain)  
  - *Role*: Acts as a verification expert, evaluating the quality of the report against original data.  
  - *Configuration*:  
    - Prompt instructs to verify factual accuracy, comprehensiveness, formatting, and insight quality.  
    - Input includes original CRM data and AI-generated report in Markdown.  
    - Output is a JSON object rating report quality and listing any missing elements or inaccuracies.  
  - *Inputs/Outputs*: Input from "Report" node; outputs to "HTTP Request" node.  
  - *Edge Cases*: Complexity of prompt may cause timeout or partial evaluations; errors if inputs are missing or improperly formatted.  
  - *Version Specifics*: Uses LangChain module v1.6.

- **DeepSeek Chat**  
  - *Type*: DeepSeek LLM Chat  
  - *Role*: Provides the AI language model backend for the "Recheck" node.  
  - *Configuration*: Default model options; no special parameters.  
  - *Inputs/Outputs*: Linked as language model provider for "Recheck."  
  - *Credentials*: Uses DeepSeek API credentials.  
  - *Edge Cases*: Similar to "DeepSeek Reasonning," risks include API failures or credential issues.

---

#### 1.4 Output Dispatch

**Overview:**  
Sends the verified report and related data to an external system via HTTP POST callback.

**Nodes Involved:**  
- HTTP Request  

**Node Details:**

- **HTTP Request**  
  - *Type*: HTTP Request  
  - *Role*: Posts the verified report data to a configured external API endpoint.  
  - *Configuration*:  
    - URL placeholder: "YOUR_CALL_BACK_API" (requires user replacement).  
    - Method: POST.  
    - Body: JSON with `data` field containing the verified report text (`{{$node['Report'].json.text}}`).  
  - *Inputs/Outputs*: Input from "Recheck" node.  
  - *Edge Cases*:  
    - Failures if callback URL is incorrect, unreachable, or returns errors.  
    - Network timeouts or malformed payload issues.

---

#### 1.5 Workflow Control and Testing

**Overview:**  
Supports workflow execution control and testing with manual triggers and sample data.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Example data  

**Node Details:**  
- Details covered under block 1.1.

---

#### 1.6 Documentation and User Guidance

**Overview:**  
Contains sticky notes for user guidance, instructions, and contact information to assist workflow users or maintainers.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note6  
- Sticky Note7  
- Sticky Note8  
- Sticky Note9  

**Node Details:**

- **Sticky Note (Generate report - Using deepseek R1)**  
  - Provides a reminder that the report generation uses DeepSeek Reasoner model version R1.

- **Sticky Note1 (Double-check - Using deepseek V3)**  
  - Indicates the verification step uses DeepSeek version 3.

- **Sticky Note2 (Test Workflow)**  
  - Explains the manual trigger node purpose for testing with example data.

- **Sticky Note6 (Change here)**  
  - Advises users they can customize AI prompts to adjust workflow goals.

- **Sticky Note7 (Contact info)**  
  - Provides contact email for help or suggestions: mediaplus.ma@gmail.com.

- **Sticky Note8 (Just to test)**  
  - Simple note indicating test purpose.

- **Sticky Note9 (Production)**  
  - Marks the webhook node as intended for production use.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                      | Input Node(s)               | Output Node(s)               | Sticky Note                                                  |
|-------------------------|----------------------------------|------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Initiates workflow manually         | —                           | Example data                | ## Test Workflow<br>Click this button to test the workflow with example data |
| Example data            | Code                             | Provides sample CRM data             | When clicking ‘Test workflow’ | Report                     |                                                              |
| Webhook                 | Webhook                          | Receives external CRM data          | —                           | Report                      | ## Production                                                |
| Report                  | Chain LLM (LangChain)            | Generates analytical report          | Webhook, Example data        | Recheck                     | ## Generate report<br>Using deepseek R1                      |
| DeepSeek Reasonning     | DeepSeek LLM Chat (Reasoner)     | Provides reasoning for report        | — (attached to Report)       | Report                      |                                                              |
| Recheck                 | Chain LLM (LangChain)            | Verifies report quality              | Report                      | HTTP Request                | ## Double-check<br>Using deepseek V3                         |
| DeepSeek Chat           | DeepSeek LLM Chat                | Provides AI verification model       | — (attached to Recheck)      | Recheck                     |                                                              |
| HTTP Request            | HTTP Request                    | Sends verified report to callback API | Recheck                     | —                           |                                                              |
| Sticky Note             | Sticky Note                     | User guidance and comments           | —                           | —                           | ## Generate report<br>Using deepseek R1                      |
| Sticky Note1            | Sticky Note                     | User guidance and comments           | —                           | —                           | ## Double-check<br>Using deepseek V3                         |
| Sticky Note2            | Sticky Note                     | User guidance and comments           | —                           | —                           | ## Test Workflow<br>Click this button to test the workflow with example data |
| Sticky Note6            | Sticky Note                     | User guidance for prompt customization | —                           | —                           | ## Change here<br>You can edit/add details about your goal by changing the AI promps. |
| Sticky Note7            | Sticky Note                     | Contact information                  | —                           | —                           | ## Do you need more help or have any suggestions?<br>Contact me at mediaplus.ma@gmail.com |
| Sticky Note8            | Sticky Note                     | Simple test note                    | —                           | —                           | ## Just to test                                              |
| Sticky Note9            | Sticky Note                     | Indicates production webhook node    | —                           | —                           | ## Production                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "When clicking ‘Test workflow’"  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Code Node for Example Data**  
   - Name: "Example data"  
   - Type: Code  
   - JavaScript code to return sample CRM agent metrics data structured as an array of JSON objects.  
   - Connect output from "When clicking ‘Test workflow’" to this node.

3. **Create Webhook Node**  
   - Name: "Webhook"  
   - Type: Webhook  
   - HTTP Methods: POST and GET  
   - Path: Use a unique identifier (e.g., "b408defb-315d-4676-b4c4-1dcebe81ffc0")  
   - Connect no input; this is an entry node for external data.

4. **Create Chain LLM Node for Report Generation**  
   - Name: "Report"  
   - Type: Chain LLM (LangChain)  
   - Text (Prompt):  
     - Provide instructions to analyze CRM data focusing on lead conversion, upsell, and agent ranking.  
     - Input data included as JSON stringified from the webhook or example data.  
     - Output format: Markdown.  
   - Connect inputs from both "Webhook" and "Example data" nodes to this node.  
   - Configure to use DeepSeek Reasoner as the LLM backend (see next step).

5. **Create DeepSeek Reasoner Node**  
   - Name: "DeepSeek Reasonning"  
   - Type: DeepSeek LLM Chat Node  
   - Model: "deepseek-reasoner"  
   - Credentials: Set DeepSeek API credentials (must create or import first).  
   - Connect as AI language model backend for "Report" node.

6. **Create Chain LLM Node for Report Verification**  
   - Name: "Recheck"  
   - Type: Chain LLM (LangChain)  
   - Prompt: Evaluate the AI-generated report for factual accuracy, coverage, insight quality, and formatting compliance against original input data.  
   - Input includes original CRM JSON and the Markdown report text from "Report" node.  
   - Output: JSON with verification results, quality score, and suggestions.  
   - Connect input from "Report" node.  
   - Configure to use DeepSeek Chat as AI backend (next step).

7. **Create DeepSeek Chat Node**  
   - Name: "DeepSeek Chat"  
   - Type: DeepSeek LLM Chat Node  
   - Credentials: Use same DeepSeek API credentials as above.  
   - Connect as AI language model backend for "Recheck" node.

8. **Create HTTP Request Node to Dispatch Output**  
   - Name: "HTTP Request"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: Set to your callback API endpoint (replace "YOUR_CALL_BACK_API").  
   - Body: JSON format with key "data" containing verified report text from the "Report" node.  
   - Connect input from "Recheck" node.

9. **Add Sticky Notes for Documentation and Guidance**  
   - Create sticky notes with content describing each block such as:  
     - "Generate report using DeepSeek R1" near "Report" node.  
     - "Double-check using DeepSeek V3" near "Recheck" node.  
     - Testing instructions near manual trigger node.  
     - Contact information as footer note.  
     - Production note near webhook.

10. **Validate Connections and Credentials**  
    - Ensure all nodes are properly linked as per connections described.  
    - Verify DeepSeek API credentials are active and authorized.  
    - Test webhook URL accessibility if deploying in production.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                              |
|----------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Workflow uses DeepSeek AI models: Reasoner for report generation (R1), and Chat for verification (V3). | Branding and versioning information.                          |
| Contact for support or suggestions: mediaplus.ma@gmail.com                                   | Maintainer contact information.                              |
| To test the workflow, use the manual trigger node which feeds example CRM data for local debugging. | Workflow testing instructions.                               |
| Callback API URL must be replaced by the user to integrate with the intended external system. | Integration setup requirement.                               |

---

**Disclaimer:** The provided text is extracted exclusively from an automated n8n workflow. It adheres strictly to content policies, contains no illegal or offensive material, and works solely with legal and public data.