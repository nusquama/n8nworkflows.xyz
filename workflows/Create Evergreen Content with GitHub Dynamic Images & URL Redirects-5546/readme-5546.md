Create Evergreen Content with GitHub Dynamic Images & URL Redirects

https://n8nworkflows.xyz/workflows/create-evergreen-content-with-github-dynamic-images---url-redirects-5546


# Create Evergreen Content with GitHub Dynamic Images & URL Redirects

### 1. Workflow Overview

This workflow automates the creation and updating of evergreen digital content by combining dynamic images hosted on GitHub with URL redirects managed via a URL shortening service. Its primary use case is to maintain always-current promotional materials or informational assets that refresh automatically over time without changing their access URLs.

The workflow is logically divided into three main blocks:

- **1.1 Initialization and Setup (One-time execution)**: Creates initial GitHub image file and URL alias for redirection.
- **1.2 Periodic Updates (Scheduled triggers)**: Automatically updates the dynamic image hourly on GitHub and updates the URL redirect daily to point to a new Wikipedia page.
- **1.3 User Interaction and Notification (Manual and form triggers)**: Supports manual triggering to retrieve the latest image and sending it via email, as well as providing a form displaying the dynamic image.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Setup (One-time execution)

**Overview:**  
This block sets up the initial environment by creating the first GitHub file that will hold the dynamic image, creating the URL alias on the shortening service, and obtaining the downloadable URL of the image for further use.

**Nodes Involved:**  
- Create new URL redirection  
- Create a file  
- Get the download URL  

**Node Details:**

- **Create new URL redirection**  
  - Type: HTTP Request  
  - Role: Creates a new alias on the URL shortening service (`shorten.rest`) that redirects to a Wikipedia page for the current day of the year.  
  - Configuration: POST request to `https://api.shorten.rest/aliases` with JSON body specifying the destination URL based on the current day formatted and sanitized (`$now.format('DDD').split(',')[0].replaceAll(' ','_')`). Query parameters include domain and alias name ("short.fyi" and "today-in-history"). Uses HTTP header authentication.  
  - Inputs: None (triggered manually)  
  - Outputs: Alias creation response for further processing  
  - Potential failures: Authentication errors, API rate limits, malformed date format, network issues.

- **Create a file**  
  - Type: GitHub  
  - Role: Creates the initial image file (`dynamic_img.png`) in the GitHub repository under the specified path. This file serves as the base for subsequent dynamic image updates.  
  - Configuration: Uses OAuth2 authentication, specifies repository and owner, commits with message "initial creation". File content is a placeholder "some content".  
  - Inputs: None (triggered manually)  
  - Outputs: Confirmation of file creation  
  - Potential failures: Authentication issues, repository permission errors, API limits.

- **Get the download URL**  
  - Type: GitHub  
  - Role: Fetches the downloadable URL of the just-created image file from GitHub, which can be used in emails or forms to display the image.  
  - Configuration: OAuth2 authenticated GET request to GitHub API for the file path. Does not fetch binary data but metadata including the download URL.  
  - Inputs: None (triggered manually)  
  - Outputs: JSON containing download URL  
  - Potential failures: File not found, permission errors, rate limits.

---

#### 2.2 Periodic Updates (Scheduled triggers)

**Overview:**  
This block handles periodic updates to keep the content fresh. A scheduled hourly trigger updates the dynamic image on GitHub, and a daily trigger updates the URL redirection to point to a new Wikipedia page corresponding to the current day.

**Nodes Involved:**  
- Schedule Trigger (daily)  
- Schedule Trigger1 (hourly)  
- Update URL redirection  
- Create Image  
- Update GitHub file  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Fires once daily at minute 8 of every hour to update the URL redirection.  
  - Configuration: Interval with trigger at minute 8.  
  - Inputs: None  
  - Outputs: Triggers the "Update URL redirection" node.  

- **Update URL redirection**  
  - Type: HTTP Request  
  - Role: Updates the existing URL alias on `shorten.rest` to redirect to the current day's Wikipedia article, keeping the alias URL intact but changing the target.  
  - Configuration: PUT request with JSON body similar to the creation node, uses same authentication and query parameters. Configured to never error on response to ensure smooth scheduling.  
  - Inputs: Triggered by Schedule Trigger  
  - Outputs: API response confirming update  
  - Potential failures: Authentication, API limits, network errors.

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Fires hourly at minute 1 to update the GitHub image file with a new dynamic image representing the current time and date.  
  - Configuration: Interval with trigger at minute 1 of every hour.  
  - Inputs: None  
  - Outputs: Triggers "Create Image" node.

- **Create Image**  
  - Type: Edit Image  
  - Role: Programmatically creates a new 640x480 image with layered graphical elements and text displaying the current date and time.  
  - Configuration: Multi-step image creation with transparent backgrounds, text blocks "TODAY", current date (localized English medium format), and current time in 24h format. Uses Courier New and Courier New Bold fonts for styling. Positions and sizes are dynamically calculated to center text. Outputs a binary image file named "dynamic_img".  
  - Inputs: Triggered by hourly Schedule Trigger1  
  - Outputs: Binary image data for the updated image  
  - Potential failures: Font file availability, image processing errors.

- **Update GitHub file**  
  - Type: GitHub  
  - Role: Updates the existing `dynamic_img.png` file in GitHub repository with the newly created image binary from the previous node. Commits with message "dynamic_img_update".  
  - Configuration: OAuth2 authentication, edit file operation, binary data input, repository and owner specified.  
  - Inputs: Receives binary image from "Create Image"  
  - Outputs: Confirmation of file update  
  - Potential failures: File lock or conflict, permission issues, file size limits.

---

#### 2.3 User Interaction and Notification (Manual and form triggers)

**Overview:**  
This block enables manual workflow execution to retrieve the latest dynamic image from GitHub and send it via email, and also provides a form trigger displaying the dynamic image with clickable URL for end-users or testers.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get a file  
- Send a message  
- Form with dynamic image  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual execution of the workflow to initiate fetching the latest image and sending a test email.  
  - Configuration: No parameters.  
  - Inputs: User manually triggers.  
  - Outputs: Triggers "Get a file" node.

- **Get a file**  
  - Type: GitHub  
  - Role: Retrieves the latest dynamic image file metadata (including download URL) from GitHub repository.  
  - Configuration: OAuth2 authenticated GET operation, specifying file path and repository. Does not fetch binary data but file metadata.  
  - Inputs: Triggered by manual trigger.  
  - Outputs: JSON with download URL.

- **Send a message**  
  - Type: Gmail  
  - Role: Sends an email with an HTML message containing the dynamic image linked to the short URL.  
  - Configuration: Sends to fixed recipient `teds.tech.talks@gmail.com`, subject "Test Image", HTML body containing an `<h2>` heading and an anchor wrapping an `<img>` tag with the GitHub download URL from the previous node. Uses Gmail OAuth2 credentials.  
  - Inputs: Receives JSON with download URL from "Get a file".  
  - Outputs: Email send confirmation.  
  - Potential failures: Gmail auth errors, quota exceeded, malformed HTML.

- **Form with dynamic image**  
  - Type: Form Trigger  
  - Role: Provides a web form displaying the dynamic clickable image linking to the short URL, used for testing or demonstrating the workflow.  
  - Configuration: Custom CSS for styling, form title "Test Form", a single HTML field embedding an `<h2>` heading and an anchor wrapping the dynamic image from GitHub URL (hardcoded). No attribution appended.  
  - Inputs: HTTP webhook trigger (URL exposed by n8n)  
  - Outputs: Form submission data (not used further in this workflow).  
  - Potential failures: Webhook misconfiguration, CSS or HTML rendering issues.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                              | Input Node(s)                   | Output Node(s)               | Sticky Note                                                                                                                     |
|--------------------------|---------------------|----------------------------------------------|--------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Manual start to fetch latest image and email it | None                           | Get a file                   |                                                                                                                                 |
| Create new URL redirection | HTTP Request        | Create initial URL alias pointing to today's Wikipedia page | None                           |                              | Run these nodes only once to create initial URL alias                                                                         |
| Update URL redirection     | HTTP Request        | Update URL alias daily to point to new Wikipedia page | Schedule Trigger               |                              | These triggers periodically update image and URL                                                                             |
| Schedule Trigger           | Schedule Trigger    | Daily trigger at minute 8 to update URL redirect | None                           | Update URL redirection       | These triggers periodically update image and URL                                                                             |
| Schedule Trigger1          | Schedule Trigger    | Hourly trigger at minute 1 to update dynamic image | None                           | Create Image                 | These triggers periodically update image and URL                                                                             |
| Create Image               | Edit Image          | Create new dynamic image with current date/time | Schedule Trigger1              | Update GitHub file           | These triggers periodically update image and URL                                                                             |
| Update GitHub file         | GitHub              | Update image file in GitHub repository         | Create Image                   |                              | These triggers periodically update image and URL                                                                             |
| Get a file                 | GitHub              | Retrieve latest image file metadata (download URL) | When clicking ‘Execute workflow’ | Send a message               |                                                                                                                                 |
| Send a message             | Gmail               | Send test email containing dynamic image       | Get a file                    |                              | Via E-mail: Prepare HTML message with image link                                                                              |
| Create a file              | GitHub              | Create initial placeholder image file          | None                           | Get the download URL         | Run these nodes only once to create initial GitHub file                                                                       |
| Get the download URL       | GitHub              | Get download URL of image file                   | Create a file                  |                              | Run these nodes only once to get downloadable image URL                                                                       |
| Form with dynamic image    | Form Trigger        | Display dynamic clickable image in a web form   | None                          |                              | Test the form and note the dynamic clickable image. Custom CSS recommended                                                    |
| Sticky Note                | Sticky Note         | Informational content with example links        | None                          |                              | Content of both image and URL are dynamic. Try: https://short.fyi/today-in-history and GitHub image URL                        |
| Sticky Note1               | Sticky Note         | Instruction to run initial setup nodes once     | None                          |                              | Run these nodes only once to create initial URL alias, initial GitHub file, get image URL                                      |
| Sticky Note2               | Sticky Note         | Explanation of scheduled triggers                | None                          |                              | Scheduled triggers update image hourly and URL daily                                                                          |
| Sticky Note3               | Sticky Note         | Notes on HTML form styling for dynamic image     | None                          |                              | Configure form CSS for correct image size                                                                                     |
| Sticky Note4               | Sticky Note         | Email usage instructions for dynamic image       | None                          |                              | Example HTML snippet for sending dynamic image in emails                                                                      |
| Sticky Note5               | Sticky Note         | Use-cases summary                                  | None                          |                              | Three ideas for using dynamic image and URL in promo materials                                                                |
| Sticky Note6               | Sticky Note         | Workflow purpose and requirements                  | None                          |                              | Explains evergreen content concept and service requirements (shorten.rest, GitHub)                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create new URL redirection node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.shorten.rest/aliases`  
   - Authentication: HTTP Header Auth with API key (shorten.rest account)  
   - Query Parameters:  
     - domainName = `short.fyi`  
     - aliasName = `today-in-history`  
   - Body (JSON):  
     ```json
     {
       "destinations": [{"url": "https://en.wikipedia.org/wiki/{{ $now.format('DDD').split(',')[0].replaceAll(' ','_') }}"}]
     }
     ```  
   - Set to send body as JSON.

2. **Create a file on GitHub**  
   - Type: GitHub node  
   - Operation: Create file  
   - Repository: `public-tests` owned by `ed-parsadanyan`  
   - File Path: `n8n-examples/dynamic-images/dynamic_img.png`  
   - Content: "some content" (placeholder)  
   - Commit message: "initial creation"  
   - Authentication: OAuth2 with GitHub credentials.

3. **Get the download URL of the GitHub file**  
   - Type: GitHub node  
   - Operation: Get file  
   - File Path: same as above  
   - Authentication: OAuth2  
   - Output: JSON metadata including `download_url`.

4. **Schedule daily URL redirect update trigger**  
   - Type: Schedule Trigger  
   - Set to trigger daily at minute 8.

5. **Update URL redirection node**  
   - Type: HTTP Request  
   - Method: PUT  
   - URL: `https://api.shorten.rest/aliases`  
   - Same authentication and query parameters as creation node.  
   - Body (JSON): same as creation node, with dynamic Wikipedia URL.  
   - Configure to never error on response.

6. **Schedule hourly image update trigger**  
   - Type: Schedule Trigger  
   - Set to trigger every hour at minute 1.

7. **Create Image node**  
   - Type: Edit Image  
   - Operation: Multi-step image creation  
   - Image size: 640x480  
   - Transparent background layers with two rectangles in different colors  
   - Add text "TODAY" at fixed position with Courier New font, size 72  
   - Add current date (localized medium format in English) centered horizontally, size 72, Courier New Bold  
   - Add current time (HH:mm 24h) centered similarly, size 72, Courier New Bold  
   - Output file name: "dynamic_img" (png).

8. **Update GitHub file node**  
   - Type: GitHub  
   - Operation: Edit file  
   - File Path: same as above  
   - Input: binary image from Create Image node  
   - Commit message: "dynamic_img_update"  
   - Authentication: OAuth2.

9. **Manual trigger for testing**  
   - Type: Manual Trigger node.

10. **Get a file node**  
    - Type: GitHub  
    - Operation: Get file metadata  
    - File Path: same as above  
    - Authentication: OAuth2.

11. **Send a message node (Gmail)**  
    - Type: Gmail node  
    - To: `teds.tech.talks@gmail.com`  
    - Subject: "Test Image"  
    - Body (HTML):  
      ```html
      <h2>Dynamic image</h2>
      <a href="https://short.fyi/today-in-history" target="_blank">
        <img src="{{ $json.download_url }}" />
      </a>
      ```  
    - Authentication: Gmail OAuth2.

12. **Form with dynamic image node**  
    - Type: Form Trigger  
    - Webhook ID auto-generated  
    - Form title: "Test Form"  
    - Form Description: "This form contains a dynamic image"  
    - Custom CSS: includes styling to ensure image max-width 100%, height auto, block display, margin bottom 8px (see sticky note content).  
    - Form field: Custom HTML with clickable image linking to `https://short.fyi/today-in-history` and image source from GitHub raw URL.

13. **Connections**  
    - Connect Schedule Trigger to Update URL redirection  
    - Connect Schedule Trigger1 to Create Image  
    - Connect Create Image to Update GitHub file  
    - Connect Manual Trigger to Get a file  
    - Connect Get a file to Send a message  

14. **Credentials Setup**  
    - HTTP Header Auth: API key for shorten.rest service  
    - GitHub OAuth2 credentials with repo write/read access  
    - Gmail OAuth2 credentials for sending email

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Evergreen digital assets remain fresh indefinitely by updating dynamic images and redirect URLs without changing their access paths. This workflow demonstrates this concept using GitHub for image hosting and shorten.rest for URL redirects.                                                                                                                               | Workflow purpose (Sticky Note6)                                                                    |
| Requires free accounts on [shorten.rest](https://shorten.rest/) for URL aliases and [GitHub](https://github.com/) for image hosting. These services can be swapped with alternatives like other shortening services or S3 buckets.                                                                                                                                            | Requirements (Sticky Note6)                                                                        |
| Example links to test dynamic content: [https://short.fyi/today-in-history](https://short.fyi/today-in-history) and the GitHub image [dynamic_img.png](https://github.com/ed-parsadanyan/public-tests/blob/main/n8n-examples/dynamic-images/dynamic_img.png)                                                                                                                 | Content examples (Sticky Note)                                                                     |
| When building HTML forms, apply CSS to ensure dynamic images scale correctly: `div.html img { max-width: 100%; height: auto; display: block; margin-bottom: 8px; }`                                                                                                                                                                                                         | Form styling note (Sticky Note3)                                                                   |
| Email usage example with clickable dynamic image and link included in HTML body. Good for newsletters or promotional emails.                                                                                                                                                                                                                                             | Email usage instructions (Sticky Note4)                                                           |
| The workflow includes three main usage ideas: 1) evergreen promotional links, 2) dynamic images in emails or web forms, 3) automated content updates without manual intervention.                                                                                                                                                                                          | Use-cases summary (Sticky Note5)                                                                   |
| Video or blog materials related to this workflow may be found at the GitHub repository or the n8n community examples from the author `ed-parsadanyan`.                                                                                                                                                                                                                      | Author and resources (implied from node metadata and links)                                       |

---

This documentation enables understanding, maintenance, and reproduction of the workflow fully within n8n without needing the original JSON export. It also highlights potential failure points and integration prerequisites.