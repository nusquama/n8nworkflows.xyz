Extract TikTok Usernames from Any Video or Creator Link Format

https://n8nworkflows.xyz/workflows/extract-tiktok-usernames-from-any-video-or-creator-link-format-10249


# Extract TikTok Usernames from Any Video or Creator Link Format

### 1. Workflow Overview

This n8n workflow is designed to extract the TikTok creator’s username from any TikTok video or creator link format provided by a user. It addresses the complexity of TikTok’s multiple URL formats—including direct profile URLs and shortened mobile app links—that make straightforward username extraction unreliable. The workflow ensures robust username retrieval by resolving shortened links to their canonical forms and parsing the username accurately.

**Target Use Cases:**  
- Automations requiring TikTok creator identification from shared links  
- Data enrichment or social media monitoring where creator usernames are needed  
- User-facing forms or API endpoints to validate and transform TikTok URLs  

**Logical Blocks:**  
- **1.1 Input Reception:** Handles user input via a form trigger and validates the TikTok link.  
- **1.2 Link Resolution:** Detects if the link is shortened or indirect and follows HTTP redirects to get the canonical TikTok video URL.  
- **1.3 Username Extraction:** Parses the canonical URL to extract the TikTok username.  
- **1.4 Error Handling:** Manages invalid links and username extraction failures by informing the user.  
- **1.5 Output Delivery:** Displays or returns the extracted username and the corresponding profile URL.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures the TikTok link input from a user via a web form and performs initial validation to check if the input contains "tiktok".  

- **Nodes Involved:**  
  - *On form submission*  
  - *If invalid TikTok link*  

- **Node Details:**  

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point; receives user-submitted TikTok link through a web form with a single required field labeled "TikTok creator/video link:".  
    - Configuration: The form asks for any TikTok link format including profiles or videos.  
    - Inputs: External user input  
    - Outputs: Passes form data to validation node  
    - Edge Cases: Empty or malformed submissions are caught downstream  
    - Sticky Note: Suggests alternative input methods if a form is not desired  

  - **If invalid TikTok link**  
    - Type: If Condition  
    - Role: Validates that the submitted link contains the string "tiktok" to filter out irrelevant inputs.  
    - Configuration: Condition checks if input field string does *not* contain "tiktok" (case sensitive, strict validation).  
    - Inputs: From form submission  
    - Outputs:  
      - If TRUE (invalid link): routes to error inform node  
      - If FALSE (valid link): routes to next block for further processing  
    - Edge Cases: Links missing "tiktok" anywhere are considered invalid  

#### 1.2 Link Resolution

- **Overview:**  
  Resolves shortened TikTok URLs (e.g., vm.tiktok.com) to their canonical www.tiktok.com URLs by performing an HTTP request without automatically following redirects. Extracts the redirect target from the response headers or body.  

- **Nodes Involved:**  
  - *If final TikTok link*  
  - *Get final link of TikTok video*  
  - *Extract final TikTok video link*  

- **Node Details:**  

  - **If final TikTok link**  
    - Type: If Condition  
    - Role: Determines if the input link already contains an "@" character, which indicates it may be a canonical link with a username embedded.  
    - Configuration: Checks if the original submitted link contains "@"  
    - Inputs: From "If invalid TikTok link" node  
    - Outputs:  
      - TRUE: Directly proceeds to username extraction  
      - FALSE: Proceeds to HTTP request to resolve shortened link  

  - **Get final link of TikTok video**  
    - Type: HTTP Request  
    - Role: Sends a GET request to the provided TikTok link without following redirects, to obtain the actual final URL from the Location header or embedded link.  
    - Configuration:  
      - URL: Dynamic from the original input  
      - Redirect: Disabled (followRedirects = false)  
      - Response: Configured to never error on HTTP failure codes to capture redirect info  
    - Inputs: From "If final TikTok link" (FALSE branch)  
    - Outputs: Raw HTTP response data  

  - **Extract final TikTok video link**  
    - Type: Set Node  
    - Role: Uses regex to extract the redirected URL from the HTTP response body’s `href` attribute, setting it as the new "TikTok creator/video link:" for further processing.  
    - Configuration: Regex extracts the first href attribute value  
    - Inputs: From HTTP request node  
    - Outputs: Updated canonical TikTok link  

#### 1.3 Username Extraction

- **Overview:**  
  Parses the canonical TikTok URL to extract the username substring after "@" and before any query parameters or path separators.  

- **Nodes Involved:**  
  - *Extract username of TikTok creator/video link*  
  - *If the username is null*  

- **Node Details:**  

  - **Extract username of TikTok creator/video link**  
    - Type: Set Node  
    - Role: Applies a regex match to extract the username from the TikTok link’s substring following “@”.  
    - Configuration: Regex `/@([^/?]+)/` extracts the username string immediately following "@" until a “/” or “?”  
    - Inputs: From either directly submitted canonical link or resolved link node  
    - Outputs: Sets a new field `username` containing the extracted TikTok username string  

  - **If the username is null**  
    - Type: If Condition  
    - Role: Checks if the extracted username is missing, null, or empty, indicating failure to parse.  
    - Configuration: Checks for non-existence or empty string for `username` field  
    - Inputs: From username extraction node  
    - Outputs:  
      - TRUE: Routes to error handling nodes  
      - FALSE: Routes to output display node  

#### 1.4 Error Handling

- **Overview:**  
  Manages error states by informing the user when the link is invalid or when username extraction fails. Stops workflow execution with error messages.  

- **Nodes Involved:**  
  - *Inform the link is invalid*  
  - *Invalid link*  
  - *Inform an error retrieving the username*  
  - *Username is null/empty*  

- **Node Details:**  

  - **Inform the link is invalid**  
    - Type: Form (completion)  
    - Role: Returns a user-friendly error message that the submitted link is invalid.  
    - Inputs: From "If invalid TikTok link" (TRUE branch)  
    - Outputs: Passes to stop and error node  

  - **Invalid link**  
    - Type: Stop and Error  
    - Role: Terminates workflow with error message “The link is invalid!”  
    - Inputs: From previous form completion node  
    - Outputs: Workflow halts  

  - **Inform an error retrieving the username**  
    - Type: Form (completion)  
    - Role: Returns an error message when username extraction fails.  
    - Inputs: From "If the username is null" (TRUE branch)  
    - Outputs: Passes to stop and error node  

  - **Username is null/empty**  
    - Type: Stop and Error  
    - Role: Terminates workflow with error message “An error ocurred while retrieving the creator's username!”  
    - Inputs: From previous form completion node  
    - Outputs: Workflow halts  

#### 1.5 Output Delivery

- **Overview:**  
  Presents the successfully extracted TikTok username to the user in a formatted completion form, including a direct profile URL using the username.  

- **Nodes Involved:**  
  - *Display the creator's username*  

- **Node Details:**  

  - **Display the creator's username**  
    - Type: Form (completion)  
    - Role: Displays a confirmation message with the extracted username and a clickable profile URL.  
    - Configuration: Uses expressions to embed the username dynamically in the message and profile URL.  
    - Inputs: From "If the username is null" (FALSE branch)  
    - Outputs: Final user-facing output  

---

### 3. Summary Table

| Node Name                           | Node Type           | Functional Role                                | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                           |
|-----------------------------------|---------------------|-----------------------------------------------|-------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------|
| On form submission                 | Form Trigger        | Entry point; receives TikTok link input       | -                             | If invalid TikTok link          | Suggests alternative input methods if not using form                                                 |
| If invalid TikTok link             | If Condition        | Validates input contains "tiktok"              | On form submission            | Inform the link is invalid, If final TikTok link | Suggests customizing invalid link handling                                                          |
| Inform the link is invalid         | Form (completion)   | User feedback on invalid link                   | If invalid TikTok link (TRUE) | Invalid link                   | Suggests alternative invalid link handling                                                          |
| Invalid link                      | Stop and Error      | Stops workflow with invalid link error          | Inform the link is invalid     | -                              |                                                                                                     |
| If final TikTok link              | If Condition        | Checks if link contains "@" character           | If invalid TikTok link (FALSE) | Extract username..., Get final link... |                                                                                                     |
| Get final link of TikTok video   | HTTP Request        | Resolves shortened TikTok link to canonical URL | If final TikTok link (FALSE)   | Extract final TikTok video link |                                                                                                     |
| Extract final TikTok video link  | Set Node            | Extracts canonical TikTok link from HTTP data   | Get final link of TikTok video | Extract username...             |                                                                                                     |
| Extract username of TikTok creator/video link | Set Node            | Extracts username substring from TikTok URL     | If final TikTok link (TRUE), Extract final TikTok video link | If the username is null         |                                                                                                     |
| If the username is null           | If Condition        | Checks if username extraction failed            | Extract username...            | Inform an error retrieving username, Display the creator's username | Suggests customizing username extraction failure handling                                           |
| Inform an error retrieving the username | Form (completion)   | User feedback on username extraction failure    | If the username is null (TRUE) | Username is null/empty          |                                                                                                     |
| Username is null/empty            | Stop and Error      | Stops workflow with username extraction error   | Inform an error retrieving the username | -                              |                                                                                                     |
| Display the creator's username    | Form (completion)   | Shows extracted username and profile URL        | If the username is null (FALSE) | -                              | Suggests alternative output handling                                                                |
| Sticky Note                      | Sticky Note         | Notes on input method                            | -                             | -                              | "If you don't want to use a form..."                                                                |
| Sticky Note1                     | Sticky Note         | Notes on invalid link handling                   | -                             | -                              | "If you don't want to inform the user through the form..."                                         |
| Sticky Note2                     | Sticky Note         | Blank / placeholder                              | -                             | -                              |                                                                                                     |
| Sticky Note3                     | Sticky Note         | Notes on username extraction failure handling   | -                             | -                              | "If you don't want to inform the user through the form..."                                         |
| Sticky Note4                     | Sticky Note         | Notes on output delivery                          | -                             | -                              | "If you don't want to inform the user through the form..."                                         |
| Sticky Note5                     | Sticky Note         | Workflow purpose explanation                      | -                             | -                              | Detailed description on TikTok link formats and extraction rationale                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: `On form submission`  
   - Purpose: To receive user input of TikTok link  
   - Configuration:  
     - Form Title: *TikTok creator/video link:*  
     - Form Fields: One required text field labeled “TikTok creator/video link:” with placeholder `https://[...]tiktok.com/[...]`  
     - Button Label: “Get creator's username”  
     - Response Mode: Last Node  
     - Form Description: Explain supported TikTok link formats for user guidance  

2. **Add an If node to validate the TikTok link**  
   - Name: `If invalid TikTok link`  
   - Condition:  
     - Check if the submitted link does *not* contain "tiktok" (case sensitive, strict)  
   - Connect from `On form submission` node  

3. **Add a Form node to inform invalid link**  
   - Name: `Inform the link is invalid`  
   - Operation: Completion  
   - Completion Title: “Error!”  
   - Completion Message: “The link is invalid!”  
   - Connect from `If invalid TikTok link` node’s TRUE branch  

4. **Add a Stop and Error node to halt on invalid link**  
   - Name: `Invalid link`  
   - Error Message: “The link is invalid!”  
   - Connect from `Inform the link is invalid` node  

5. **Add an If node to check if the TikTok link contains '@'**  
   - Name: `If final TikTok link`  
   - Condition:  
     - Check if original input link contains '@' character  
   - Connect from `If invalid TikTok link` node’s FALSE branch  

6. **Add an HTTP Request node to get the final canonical TikTok link**  
   - Name: `Get final link of TikTok video`  
   - Method: GET  
   - URL: Use expression to take the original TikTok link from form input  
   - Options: Disable automatic redirects (followRedirects = false)  
   - Response: Configure to never error on HTTP failure  
   - Connect from `If final TikTok link` node’s FALSE branch  

7. **Add a Set node to extract the redirected TikTok link from HTTP response**  
   - Name: `Extract final TikTok video link`  
   - Assignment: Use regex to extract the first href URL from the HTTP response data, set as new `TikTok creator/video link:`  
   - Connect from `Get final link of TikTok video` node  

8. **Add a Set node to extract username from TikTok link**  
   - Name: `Extract username of TikTok creator/video link`  
   - Assignment: Use regex `/@([^/?]+)/` on `TikTok creator/video link:` to extract username substring after '@'  
   - Connect from both `If final TikTok link` node’s TRUE branch and `Extract final TikTok video link` node  

9. **Add an If node to check if username extraction failed**  
   - Name: `If the username is null`  
   - Condition: Check if `username` field is missing or empty string  
   - Connect from `Extract username of TikTok creator/video link` node  

10. **Add a Form node to inform user of extraction failure**  
    - Name: `Inform an error retrieving the username`  
    - Operation: Completion  
    - Completion Title: “Error!”  
    - Completion Message: “An error ocurred while retrieving the creator's username.”  
    - Connect from `If the username is null` node’s TRUE branch  

11. **Add a Stop and Error node to halt on username extraction failure**  
    - Name: `Username is null/empty`  
    - Error Message: “An error ocurred while retrieving the creator's username!”  
    - Connect from `Inform an error retrieving the username` node  

12. **Add a Form node to display the extracted username**  
    - Name: `Display the creator's username`  
    - Operation: Completion  
    - Completion Title: “Creator's username found!”  
    - Completion Message:  
      ```
      <br>
      The creator's username is: <strong>{{ $json.username }}</strong>
      <br><br>
      This means their profile link is <em>https://www.tiktok.com/@{{ $json.username }}</em>
      ```  
    - Connect from `If the username is null` node’s FALSE branch  

13. **Add Sticky Notes as needed for documentation and guidance**  
    - Add notes describing input method alternatives, error handling customization, and workflow purpose for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow handles multiple TikTok link formats including mobile app short links (vm.tiktok.com) and full profile/video URLs. | Sticky Note5: Detailed explanation of the workflow’s purpose and TikTok link format complexities.     |
| You can replace the form trigger with other input methods if desired, as long as the input field is named "TikTok creator/video link:". | Sticky Note: Input method alternative suggestion.                                                     |
| Error handling nodes can be customized to change how users are informed about invalid links or extraction failures.             | Sticky Notes1 and 3: Guidance on customizing error notifications.                                     |
| The username extracted is always without the "@" symbol, suitable for building profile URLs or further data processing.         | Workflow logic and output messages.                                                                   |
| Workflow uses regex extraction; ensure input format variations do not break regex assumptions.                                   | Regex patterns: `/@([^/?]+)/` for username, `/href="([^"]+)"/` for link extraction.                    |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.