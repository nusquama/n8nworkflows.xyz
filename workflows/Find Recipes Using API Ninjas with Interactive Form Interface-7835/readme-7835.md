Find Recipes Using API Ninjas with Interactive Form Interface

https://n8nworkflows.xyz/workflows/find-recipes-using-api-ninjas-with-interactive-form-interface-7835


# Find Recipes Using API Ninjas with Interactive Form Interface

### 1. Workflow Overview

This workflow serves as a simple interactive recipe finder application. It enables users to input a query—such as an ingredient or dish name—through a form interface. The workflow then calls the API Ninjas Recipe API to retrieve matching recipes and returns the formatted recipe details back to the user within the same interface.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Captures user input through a form trigger node.
- **1.2 API Integration:** Sends the user query to the API Ninjas Recipe API using an authenticated HTTP request.
- **1.3 Output Presentation:** Displays the retrieved recipe title, ingredients, and instructions back on a form completion page.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block waits for the user to submit a recipe query via a web form. It collects the input which will be used as the search parameter for the recipe API.

- **Nodes Involved:**  
  - Form Trigger - Recipe Finder

- **Node Details:**  

  **Form Trigger - Recipe Finder**  
  - Type: `formTrigger`  
  - Role: Entry point for user interaction; captures user input via a web form.  
  - Configuration:  
    - Form title set to "Find a recipe".  
    - A single form field named `query` where the user inputs the ingredient or dish.  
    - Response mode set to "lastNode", meaning the response shown to the user will come from the last executed node downstream.  
    - Webhook ID configured for external access to this form trigger.  
  - Input: None (trigger node)  
  - Output: Passes JSON containing `{ query: <user input> }` to the next node.  
  - Edge cases:  
    - User submits empty or invalid query (no explicit validation here).  
    - Network or webhook connection failure.  

#### 2.2 API Integration

- **Overview:**  
  This block takes the user query and makes an authenticated call to the API Ninjas Recipe API to fetch recipe data matching the query.

- **Nodes Involved:**  
  - Fetch Recipe from API Ninjas

- **Node Details:**  

  **Fetch Recipe from API Ninjas**  
  - Type: `httpRequest`  
  - Role: Calls the external API to retrieve recipe information.  
  - Configuration:  
    - HTTP Method: GET (default for query parameters).  
    - URL: `https://api.api-ninjas.com/v1/recipe`  
    - Query Parameter: `query` set dynamically using the expression `={{ $json.query }}` which pulls the user input from the previous node.  
    - Authentication: HTTP Header Auth using API key credential stored in "API Ninjas Credential". The API key is sent in request headers as required by API Ninjas.  
  - Input: JSON with field `query` from the Form Trigger node.  
  - Output: JSON response from API containing recipe data, expected to include fields like title, ingredients, instructions.  
  - Edge cases:  
    - Invalid or missing API key causing authentication failure.  
    - API rate limiting or downtime.  
    - No recipes found for query (response empty or error).  
    - Network timeout or HTTP errors.  
  - Version-specific requirements: Uses n8n HTTP Request node v4.2 features.  

#### 2.3 Output Presentation

- **Overview:**  
  This block formats the fetched recipe data and returns it as a completion message on the form page, presenting the user with the recipe title, ingredients, and instructions.

- **Nodes Involved:**  
  - Show Recipe Result

- **Node Details:**  

  **Show Recipe Result**  
  - Type: `form` node used in completion mode  
  - Role: Displays the final recipe information back to the user after the API call completes.  
  - Configuration:  
    - Operation set to "completion" to provide a final form completion page.  
    - Completion title dynamically set to `={{ $json.title }}`, showing the recipe title.  
    - Completion message formatted as HTML with headings for Ingredients and Instructions, using `{{ $json.ingredients }}` and `{{ $json.instructions }}` to inject API response fields.  
    - Webhook ID assigned for form response handling.  
  - Input: JSON recipe data from API call node.  
  - Output: HTTP response to the user-submitted form page displaying the recipe.  
  - Edge cases:  
    - Missing or malformed recipe fields causing incomplete display.  
    - HTML rendering issues if fields contain unexpected data.  
    - Network or webhook issues delivering the form response.  

---

### 3. Summary Table

| Node Name                 | Node Type         | Functional Role          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                         |
|---------------------------|-------------------|-------------------------|-----------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------|
| Workflow description      | stickyNote        | Documentation overview  |                             |                            | ## Workflow overview\n\nSimple recipe finder app. User enters an ingredient or dish into a form. Workflow calls API Ninjas Recipe API and shows the recipe result back on the form page.\n\n### Setup\n- **Form Trigger - Recipe Finder**: the entry form with a `query` field.\n- **Fetch Recipe from API Ninjas**: API call node using header auth.\n- **Show Recipe Result**: returns title, ingredients, instructions in formatted text.\n\n### Flow\n1. Form trigger waits for input.\n2. Passes query to API Ninjas.\n3. Fetches recipe.\n4. Shows formatted recipe back to user.\n\n### Notes\nQuick demo of how to build a recipe finder workflow in n8n. Expand easily with more features like multiple recipes, emails, or saving results. |
| Form Trigger - Recipe Finder | formTrigger       | Input Reception          |                             | Fetch Recipe from API Ninjas | See Workflow description node for overall workflow explanation.                                                     |
| Fetch Recipe from API Ninjas | httpRequest       | API Integration          | Form Trigger - Recipe Finder | Show Recipe Result          | See Workflow description node for overall workflow explanation.                                                     |
| Show Recipe Result         | form              | Output Presentation      | Fetch Recipe from API Ninjas |                            | See Workflow description node for overall workflow explanation.                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node**  
   - Add a *Form Trigger* node and name it "Form Trigger - Recipe Finder".  
   - Set the form title to "Find a recipe".  
   - Add a single form field named `query` (text input).  
   - Set the response mode to "lastNode" to ensure the final node's output is returned.  
   - Save webhook ID or leave n8n to auto-assign.  

2. **Create the HTTP Request node**  
   - Add an *HTTP Request* node named "Fetch Recipe from API Ninjas".  
   - Set the request URL to `https://api.api-ninjas.com/v1/recipe`.  
   - Set the HTTP method to GET (default).  
   - Under Query Parameters, add a parameter named `query` with the value `={{ $json.query }}` to dynamically use the form input.  
   - Configure authentication:  
     - Select "HTTP Header Auth" credential type.  
     - Create or select credentials for API Ninjas API key; add the key as required by the API Ninjas documentation (usually `X-Api-Key` header).  
   - Connect the output of the Form Trigger node to this HTTP Request node.  

3. **Create the Form node for displaying results**  
   - Add a *Form* node named "Show Recipe Result".  
   - Set operation to "completion".  
   - Set the completion title to `={{ $json.title }}` to display the recipe name dynamically.  
   - Set the completion message to an HTML template:  
     ```html
     <h3>Ingredients</h3>
     {{ $json.ingredients }}

     <h3>Instructions</h3>
     {{ $json.instructions }}
     ```  
   - Connect the output of the HTTP Request node to this Form node.  

4. **Finalize connections and deploy**  
   - Ensure the connection chain is: Form Trigger → HTTP Request → Form.  
   - Activate the workflow.  
   - Test by accessing the form URL provided by the Form Trigger node and submitting a query.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| This workflow is a quick demo showing how to build an interactive recipe finder using n8n and API Ninjas Recipe API. Expandable to include multiple recipes, email, or saving features. | Workflow description sticky note content                  |
| API Ninjas documentation: https://api-ninjas.com/api/recipe                                                                                                             | Reference for API parameters and authentication details   |
| Use secure storage for API keys in n8n credentials to avoid exposing sensitive information.                                                                                | Best practice for API key management                       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.