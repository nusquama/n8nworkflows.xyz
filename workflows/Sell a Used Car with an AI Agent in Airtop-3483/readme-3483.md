Sell a Used Car with an AI Agent in Airtop

https://n8nworkflows.xyz/workflows/sell-a-used-car-with-an-ai-agent-in-airtop-3483


# Sell a Used Car with an AI Agent in Airtop

### 1. Workflow Overview

This workflow automates the process of selling a used car by interacting with online car resale marketplaces through real browser sessions powered by Airtop. It streamlines data entry, navigates marketplace forms, and extracts competitive offers in structured JSON format, eliminating manual repetitive tasks.

**Target Use Cases:**  
- Car dealerships and individuals seeking quick, multi-platform price offers  
- Developers and automation engineers building automated car resale or valuation tools  
- IT teams optimizing inventory resale processes with automated data extraction  

**Logical Blocks:**  
- **1.1 Input Reception:** Manual trigger and variable setup for car details  
- **1.2 Browser Session Management:** Creating, loading, interacting with, and terminating Airtop browser sessions  
- **1.3 Interaction & Navigation:** Automated clicking and typing actions on marketplace forms  
- **1.4 AI-driven Decision Making:** Using Airtop’s AI extraction to determine next steps based on page content  
- **1.5 Response Parsing & Routing:** Parsing AI responses and switching logic to perform appropriate actions or finalize offers  
- **1.6 Output & Cleanup:** Capturing final offer data and terminating sessions  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Starts the workflow manually and sets the car details variables used throughout the process.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Variables (Set node)  

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates workflow execution on demand  
  - Configuration: Default manual trigger, no parameters  
  - Input: None  
  - Output: Triggers next node  
  - Failures: None expected  

- **Variables**  
  - Type: Set  
  - Role: Defines car description variable containing VIN, mileage, zip code, condition, ownership info  
  - Configuration: Single string variable `car_description` with detailed vehicle info  
  - Key Expression: Static string with car details, e.g. `"VIN: 1FTRF17253NB81140 Milage: 221081 Zip code: 01952 Condition: Perfect..."`  
  - Input: Trigger from manual node  
  - Output: Passes variable to session creation node  
  - Failures: None expected unless variable is empty or malformed  

---

#### 2.2 Browser Session Management

**Overview:**  
Manages Airtop browser sessions to load target websites, interact with forms, and terminate sessions after completion.

**Nodes Involved:**  
- Create session (Airtop)  
- Load website (Airtop)  
- Terminate session (Airtop)  

**Node Details:**  

- **Create session**  
  - Type: Airtop  
  - Role: Starts a new real browser session for automation  
  - Configuration: Default session creation, no parameters  
  - Input: Variables node output  
  - Output: Session ID and window ID for subsequent nodes  
  - Credentials: Airtop API key required  
  - Failures: API key invalid, session creation timeout, network issues  

- **Load website**  
  - Type: Airtop  
  - Role: Opens the target resale marketplace URL (`https://sell.peddle.com/instant-offer`) in the browser session  
  - Configuration: URL parameter set, waits for DOM content loaded event  
  - Input: Session creation output  
  - Output: Window loaded and ready for interaction  
  - Credentials: Airtop API key  
  - Failures: URL unreachable, page load timeout, network errors  

- **Terminate session**  
  - Type: Airtop  
  - Role: Ends the browser session to free resources after workflow completion  
  - Configuration: Uses session ID from Create session node  
  - Input: Offer received node output (final step)  
  - Output: Session termination confirmation  
  - Credentials: Airtop API key  
  - Failures: Session ID invalid, termination failure  

---

#### 2.3 Interaction & Navigation

**Overview:**  
Automates user interactions on the loaded webpage, such as clicking buttons and typing text, to progress through the car selling process.

**Nodes Involved:**  
- Click VIN button (Airtop)  
- Wait 7 secs (Wait)  
- Take screenshot (Airtop)  
- Type (Airtop)  
- Click (Airtop)  

**Node Details:**  

- **Click VIN button**  
  - Type: Airtop  
  - Role: Clicks the "Autofill with VIN" button to initiate VIN-based form filling  
  - Configuration: Element description set to "Rounded white button 'Autofill with VIN'"  
  - Input: Load website output  
  - Output: Triggers wait node  
  - Credentials: Airtop API key  
  - Failures: Element not found, click not registered  

- **Wait 7 secs**  
  - Type: Wait  
  - Role: Pauses workflow for 7 seconds to allow page updates after click  
  - Configuration: Fixed 7-second delay  
  - Input: Click VIN button output or Type/Click nodes (depending on path)  
  - Output: Next interaction or screenshot node  
  - Failures: None expected  

- **Take screenshot**  
  - Type: Airtop  
  - Role: Captures current browser window screenshot for debugging or validation  
  - Configuration: Window resource, takeScreenshot operation  
  - Input: Wait 7 secs output  
  - Output: Image data passed to AI decision node  
  - Credentials: Airtop API key  
  - Failures: Screenshot failure due to session issues  

- **Type**  
  - Type: Airtop  
  - Role: Types text into specified input element on the webpage  
  - Configuration: Uses AI response to specify element description and text to type, presses Enter key  
  - Input: Switch node output (Type path)  
  - Output: Wait 7 secs node  
  - Credentials: Airtop API key  
  - Failures: Element not found, typing failure, expression errors in element/text  

- **Click**  
  - Type: Airtop  
  - Role: Clicks specified button or element on the webpage as directed by AI response  
  - Configuration: Uses AI response element description  
  - Input: Switch node output (Click path)  
  - Output: Wait 7 secs node  
  - Credentials: Airtop API key  
  - Failures: Element not found, click failure  

---

#### 2.4 AI-driven Decision Making

**Overview:**  
Analyzes the current webpage content and determines the next action (click, type, or finalize with price) using Airtop’s AI extraction capabilities.

**Nodes Involved:**  
- Think next action (Airtop)  
- Parse response (Code)  
- Switch (Switch)  

**Node Details:**  

- **Think next action**  
  - Type: Airtop  
  - Role: Uses AI to interpret the page, extract the main question, decide next action (click/type/price), and identify target element and text  
  - Configuration: Complex prompt instructing the AI to parse page content and car info, outputting structured JSON with fields: question, action, element, text  
  - Input: Take screenshot output  
  - Output: JSON response with action instructions  
  - Credentials: Airtop API key  
  - Failures: AI timeout, malformed response, prompt misinterpretation  

- **Parse response**  
  - Type: Code  
  - Role: Parses the JSON string from AI response into usable JSON object, extracts sessionId and windowId for context  
  - Configuration: JavaScript code parsing `modelResponse` field  
  - Input: Think next action output  
  - Output: Parsed JSON for switch node  
  - Failures: JSON parse errors, missing fields  

- **Switch**  
  - Type: Switch  
  - Role: Routes workflow based on AI action type: "TYPE", "CLICK", or "PRICE"  
  - Configuration: Checks if response.action contains "TYPE", "CLICK", or "PRICE" and routes accordingly  
  - Input: Parse response output  
  - Output: Routes to Type node, Click node, or Offer received node  
  - Failures: Unexpected action strings, case sensitivity issues  

---

#### 2.5 Response Parsing & Routing

**Overview:**  
Handles the final offer extraction and workflow termination once a price offer is received.

**Nodes Involved:**  
- Offer received (Set)  
- Terminate session (Airtop)  

**Node Details:**  

- **Offer received**  
  - Type: Set  
  - Role: Extracts and stores offer details from AI response: offer price, offer ID, and offer URL  
  - Configuration: Sets variables `Offer_price`, `Offer_id`, `Offer_URL` from response fields question, element, text respectively  
  - Input: Switch node (PRICE path)  
  - Output: Terminate session node  
  - Failures: Missing or malformed offer data  

- **Terminate session**  
  - See details in 2.2  

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                          | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                   |
|------------------------|---------------------|----------------------------------------|----------------------------|---------------------------|-----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Initiates workflow manually             | None                       | Variables                 |                                                                                               |
| Variables              | Set                 | Defines car description variables       | When clicking ‘Test workflow’ | Create session            |                                                                                               |
| Create session         | Airtop              | Starts browser session                   | Variables                  | Load website              | Requires Airtop API key credential                                                            |
| Load website           | Airtop              | Loads target resale marketplace URL     | Create session             | Click VIN button          |                                                                                               |
| Click VIN button       | Airtop              | Clicks "Autofill with VIN" button       | Load website               | Wait 7 secs               |                                                                                               |
| Wait 7 secs            | Wait                | Waits 7 seconds for page update          | Click VIN button, Type, Click | Take screenshot           |                                                                                               |
| Take screenshot        | Airtop              | Captures current browser window screenshot | Wait 7 secs                | Think next action         | Useful to validate the current screen the agent is on                                         |
| Think next action      | Airtop              | AI analyzes page, decides next action   | Take screenshot            | Parse response            |                                                                                               |
| Parse response         | Code                | Parses AI JSON response                   | Think next action          | Switch                    |                                                                                               |
| Switch                 | Switch              | Routes workflow based on AI action type | Parse response             | Type, Click, Offer received |                                                                                               |
| Type                   | Airtop              | Types text into webpage element          | Switch (Type path)          | Wait 7 secs               |                                                                                               |
| Click                  | Airtop              | Clicks webpage element                    | Switch (Click path)         | Wait 7 secs               |                                                                                               |
| Offer received         | Set                 | Extracts final offer details              | Switch (PRICE path)         | Terminate session         |                                                                                               |
| Terminate session      | Airtop              | Ends browser session                      | Offer received             | None                      |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually  

2. **Add Set Node "Variables"**  
   - Type: Set  
   - Add string variable `car_description` with detailed car info (VIN, mileage, zip, condition, ownership)  
   - Connect Manual Trigger → Variables  

3. **Add Airtop Node "Create session"**  
   - Type: Airtop  
   - Operation: Create new browser session (default)  
   - Connect Variables → Create session  
   - Configure Airtop API credentials  

4. **Add Airtop Node "Load website"**  
   - Type: Airtop  
   - Operation: Open URL  
   - URL: `https://sell.peddle.com/instant-offer`  
   - Wait until: DOM content loaded  
   - Connect Create session → Load website  
   - Use same Airtop credentials  

5. **Add Airtop Node "Click VIN button"**  
   - Type: Airtop  
   - Operation: Interaction → Click  
   - Element description: `Rounded white button "Autofill with VIN"`  
   - Connect Load website → Click VIN button  

6. **Add Wait Node "Wait 7 secs"**  
   - Type: Wait  
   - Duration: 7 seconds  
   - Connect Click VIN button → Wait 7 secs  

7. **Add Airtop Node "Take screenshot"**  
   - Type: Airtop  
   - Operation: Take screenshot of window  
   - Connect Wait 7 secs → Take screenshot  

8. **Add Airtop Node "Think next action"**  
   - Type: Airtop  
   - Operation: Extraction → Query  
   - Prompt: Use detailed prompt to analyze page, extract question, action, element, text; includes car_description variable  
   - Output schema: JSON with fields question, action, element, text  
   - Connect Take screenshot → Think next action  

9. **Add Code Node "Parse response"**  
   - Type: Code  
   - JavaScript: Parse JSON from `modelResponse` field, extract sessionId, windowId, and response object  
   - Connect Think next action → Parse response  

10. **Add Switch Node "Switch"**  
    - Type: Switch  
    - Rules:  
      - If `response.action` contains "TYPE" → output "Type"  
      - If `response.action` contains "CLICK" → output "Click"  
      - If `response.action` contains "PRICE" → output "Got price"  
    - Connect Parse response → Switch  

11. **Add Airtop Node "Type"**  
    - Type: Airtop  
    - Operation: Interaction → Type  
    - Parameters:  
      - Text: `{{ $json.response.text }}`  
      - Element description: `{{ $json.response.element }}`  
      - Press Enter key: true  
    - Connect Switch (Type output) → Type  

12. **Add Airtop Node "Click"**  
    - Type: Airtop  
    - Operation: Interaction → Click  
    - Element description: `{{ $json.response.element }}`  
    - Connect Switch (Click output) → Click  

13. **Connect Type and Click Nodes → Wait 7 secs**  
    - To allow page update before next screenshot  

14. **Add Set Node "Offer received"**  
    - Type: Set  
    - Variables:  
      - Offer_price = `{{ $json.response.question }}`  
      - Offer_id = `{{ $json.response.element }}`  
      - Offer_URL = `{{ $json.response.text }}`  
    - Connect Switch (Got price output) → Offer received  

15. **Add Airtop Node "Terminate session"**  
    - Type: Airtop  
    - Operation: Terminate session  
    - Session ID: `{{ $('Create session').last().json.sessionId }}`  
    - Connect Offer received → Terminate session  

16. **Configure Airtop API Credentials**  
    - Create and assign Airtop API key credential to all Airtop nodes  

17. **Test workflow manually**  
    - Trigger manual start  
    - Verify each step executes correctly and outputs structured offers  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| A free Airtop API key is required to run this workflow.                                                      | https://portal.airtop.ai/?utm_campaign=n8n                                                     |
| The workflow uses real browser automation to interact with resale marketplaces like Peddle.com.               |                                                                                               |
| Screenshot node is useful for debugging and validating the current page state during automation.             |                                                                                               |
| The AI prompt in "Think next action" node is critical for guiding the automation through the selling process.| Prompt includes instructions for extracting questions, actions, elements, and price offers.    |
| Regularly update the car description and marketplace URLs to maintain accuracy and reliability.              |                                                                                               |
| This workflow is designed for n8n version supporting Airtop nodes and JSON parsing in Code nodes.            |                                                                                               |