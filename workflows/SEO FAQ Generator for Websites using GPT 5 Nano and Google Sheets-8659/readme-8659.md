SEO FAQ Generator for Websites using GPT 5 Nano and Google Sheets

https://n8nworkflows.xyz/workflows/seo-faq-generator-for-websites-using-gpt-5-nano-and-google-sheets-8659


# SEO FAQ Generator for Websites using GPT 5 Nano and Google Sheets

### 1. Workflow Overview

This workflow automates the generation of SEO-friendly FAQ content for websites by leveraging GPT-5 Nano and Google Sheets. It fetches website sitemap URLs, parses them, extracts page data, generates FAQs, meta descriptions, and titles using OpenAI, and logs the results in Google Sheets. The workflow supports both manual and scheduled triggers, handles batch processing for scalability, and sends email notifications with the generated content.

**Logical blocks:**  
- **1.1 Trigger and Initialization:** Manual and scheduled start points, user agent rotation setup, and initial options configuration.  
- **1.2 Sitemap Retrieval and Parsing:** Fetches sitemap from the target website, parses XML to extract URLs.  
- **1.3 URL and Batch Processing:** Splits URLs into manageable batches and iterates over them to process each page.  
- **1.4 Content Fetching and FAQ Generation:** HTTP requests to page URLs, then generate FAQ content with GPT-5 Nano.  
- **1.5 Meta Description and Title Generation:** Uses OpenAI to generate meta descriptions and titles for pages.  
- **1.6 Data Persistence and Notification:** Stores results in Google Sheets and sends an email with the generated FAQ content.  
- **1.7 Error Handling and Limits:** Stops execution on critical errors and limits batch sizes to avoid API or request overload.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Initialization

**Overview:**  
Handles workflow initiation via manual trigger or scheduled trigger. Sets a rotating user agent and options for subsequent processing.

**Nodes involved:**  
- MANUAL  
- Schedule Trigger  
- UA Rotativo  
- OPTIONS

**Node details:**

- **MANUAL**  
  - Type: Manual Trigger  
  - Role: Allows manual start of the workflow for on-demand execution.  
  - Config: Default, no parameters.  
  - Connections: Outputs to UA Rotativo.  
  - Edge cases: None typical; manual start depends on user interaction.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically initiates workflow on a schedule.  
  - Config: Default schedule (not specified here).  
  - Connections: Outputs to UA Rotativo.  
  - Edge cases: Scheduling misconfiguration could cause missed runs.

- **UA Rotativo**  
  - Type: Code  
  - Role: Generates or rotates user agents for HTTP requests to mimic different browsers/devices.  
  - Config: Custom JavaScript code (not detailed here).  
  - Connections: Outputs to OPTIONS.  
  - Edge cases: Incorrect rotation logic may cause request failures or blocking.

- **OPTIONS**  
  - Type: Set  
  - Role: Defines initial parameters or options for the workflow, possibly including sitemap URL, batch sizes, or API keys.  
  - Config: Static or expression-based key-value pairs.  
  - Connections: Outputs to Selected node.  
  - Edge cases: Missing or invalid options can halt downstream processes.

---

#### 1.2 Sitemap Retrieval and Parsing

**Overview:**  
Retrieves the sitemap XML from the website, parses it, and validates the response. Contains error handling for missing or malformed sitemaps.

**Nodes involved:**  
- Selected  
- Split Out  
- Method  
- Maping Sitemap2  
- XML  
- Sitemap Error3

**Node details:**

- **Selected**  
  - Type: Code  
  - Role: Selects or filters data from options, possibly extracting the sitemap URL.  
  - Config: Custom JavaScript to pick relevant data.  
  - Connections: Outputs to Split Out.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits an array or object to separate elements for processing.  
  - Config: Default; splits input items.  
  - Connections: Outputs to Method.

- **Method**  
  - Type: Split In Batches  
  - Role: Divides the workload into batches for scalable processing.  
  - Config: Batch size unspecified but critical for performance.  
  - Connections: Outputs to FAQ Results and Maping Sitemap2.

- **Maping Sitemap2**  
  - Type: HTTP Request  
  - Role: Fetches the sitemap XML from the URL.  
  - Config: HTTP GET request; user agent rotation applied.  
  - Notes: May error if sitemap missing, multiple sitemaps found, or non-WordPress sites; advises contacting support or manual sitemap input.  
  - Connections: Outputs to XML and Sitemap Error3 (error branch).  
  - Edge cases: Network errors, 404 sitemap not found, invalid sitemap format.

- **XML**  
  - Type: XML  
  - Role: Parses the sitemap XML to extract URLs.  
  - Config: Parses HTTP response body.  
  - Connections: Outputs to Split Out1.  
  - Edge cases: Malformed XML or unexpected schema can cause parsing failure.

- **Sitemap Error3**  
  - Type: Stop and Error  
  - Role: Stops workflow on critical sitemap fetch/parsing errors.  
  - Config: Default error stop.  
  - Connections: None (terminal).  
  - Notes: Critical to prevent downstream processing on invalid sitemap.

---

#### 1.3 URL and Batch Processing

**Overview:**  
Splits extracted URLs into batches and limits the number of concurrent processes to avoid overload.

**Nodes involved:**  
- Split Out1  
- Limit  
- Loop page

**Node details:**

- **Split Out1**  
  - Type: Split Out  
  - Role: Further splits the sitemap URL list for controlled processing.  
  - Config: Defaults, with error branch continuing regular output.  
  - Connections: Outputs to Limit and empty fallback branch.

- **Limit**  
  - Type: Limit  
  - Role: Caps the number of processed items in parallel or overall.  
  - Config: Default or configured limit value.  
  - Connections: Outputs to Loop page.

- **Loop page**  
  - Type: Split In Batches  
  - Role: Iterates over page URLs in batches for sequential processing.  
  - Config: Batch size unspecified but important for request pacing.  
  - Connections: Outputs to Method and HTTP Req page nodes.

---

#### 1.4 Content Fetching and FAQ Generation

**Overview:**  
Fetches each page content via HTTP and generates FAQ content using GPT-5 Nano, handling errors and clearing temporary data.

**Nodes involved:**  
- HTTP Req page  
- Clear  
- FAQ Results  
- Add items DB

**Node details:**

- **HTTP Req page**  
  - Type: HTTP Request  
  - Role: Retrieves the content of each page URL.  
  - Config: GET requests with user agent rotation; retries enabled.  
  - Connections: Outputs to Clear and Stop and Error1 (error branch).  
  - Edge cases: Request timeouts, 404 errors, server errors.

- **Clear**  
  - Type: Code  
  - Role: Cleans or resets temporary data before generating FAQs.  
  - Config: Custom JavaScript.  
  - Connections: Outputs to meta descr and title gen.  
  - Edge cases: Errors in code execution may disrupt flow.

- **FAQ Results**  
  - Type: Code  
  - Role: Processes or formats the FAQ output from OpenAI.  
  - Config: Custom JavaScript.  
  - Connections: Outputs to Add items DB.

- **Add items DB**  
  - Type: Split In Batches  
  - Role: Prepares FAQ data for database insertion and email sending.  
  - Config: Batch processing parameters.  
  - Connections: Outputs to Text email and Append row in db.

---

#### 1.5 Meta Description and Title Generation

**Overview:**  
Generates SEO meta descriptions and titles for each processed page using OpenAI, with error handling and integration back into the batch loop.

**Nodes involved:**  
- meta descr and title gen  
- Stop and Error

**Node details:**

- **meta descr and title gen**  
  - Type: OpenAI (Langchain node)  
  - Role: Uses GPT-5 Nano to generate meta descriptions and titles.  
  - Config: Uses retry on failure; OpenAI credentials required.  
  - Connections: Outputs to Loop page (for next batch) and Stop and Error (error branch).  
  - Edge cases: API rate limits, network issues, malformed prompts.

- **Stop and Error**  
  - Type: Stop and Error  
  - Role: Halts workflow on unrecoverable generation errors.  
  - Connections: None (terminal).

---

#### 1.6 Data Persistence and Notification

**Overview:**  
Stores generated FAQ data into Google Sheets and sends notification emails with the results via Gmail.

**Nodes involved:**  
- Append row in db  
- Error insert row  
- Text email  
- Send a message  
- Stop and Error1  
- Stop and Error (for insert errors)

**Node details:**

- **Append row in db**  
  - Type: Google Sheets  
  - Role: Adds rows with generated FAQ data to a Google Sheet.  
  - Config: Retry enabled; error output continues.  
  - Connections: Outputs to Add items DB and Error insert row (error branch).  
  - Edge cases: Google Sheets API quota, auth issues, malformed data.

- **Error insert row**  
  - Type: Stop and Error  
  - Role: Stops workflow if Google Sheets row insertion fails irrecoverably.  
  - Notes: Contact email provided for support.

- **Text email**  
  - Type: OpenAI (Langchain node)  
  - Role: Generates the email text content using OpenAI.  
  - Config: Retry enabled; no explicit error node because previous nodes handle failures.  
  - Connections: Outputs to Send a message.

- **Send a message**  
  - Type: Gmail  
  - Role: Sends the generated email with FAQ content.  
  - Config: Uses OAuth2 Gmail credentials; no parameters detailed here.  
  - Connections: None (terminal).

- **Stop and Error1**  
  - Type: Stop and Error  
  - Role: Stops workflow on HTTP request page errors.  
  - Notes: Contact email provided.

---

#### 1.7 Error Handling and Miscellaneous Nodes

**Overview:**  
Includes various error nodes to halt workflow on critical failures and several sticky notes for documentation or reminders.

**Nodes involved:**  
- Stop and Error (multiple)  
- Sticky Note nodes (various)

**Node details:**

- **Stop and Error nodes**  
  - Type: Stop and Error  
  - Role: Various points in workflow stop execution on errors; some include contact details for support.  
  - Placement: Sitemap errors, HTTP request failures, Google Sheets insert failures, and meta generation failures.  
  - Edge cases: Critical to prevent faulty data propagation.

- **Sticky Note nodes**  
  - Type: Sticky Note  
  - Role: Visual notes inside the editor for explanations or warnings.  
  - Content: Mostly empty or brief notes; one notes sitemap retrieval issues and support contact.  
  - Usage: Helpful for human maintainers to understand context or known issues.

---

### 3. Summary Table

| Node Name             | Node Type                   | Functional Role                      | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                             |
|-----------------------|-----------------------------|------------------------------------|--------------------------|-------------------------|-------------------------------------------------------------------------------------------------------|
| MANUAL                | Manual Trigger              | Manual workflow start               | -                        | UA Rotativo             |                                                                                                       |
| Schedule Trigger      | Schedule Trigger            | Scheduled workflow start            | -                        | UA Rotativo             |                                                                                                       |
| UA Rotativo           | Code                        | Rotate user agents                  | MANUAL, Schedule Trigger | OPTIONS                 |                                                                                                       |
| OPTIONS               | Set                         | Set initial options                 | UA Rotativo               | Selected                |                                                                                                       |
| Selected              | Code                        | Select sitemap URL or params       | OPTIONS                   | Split Out               |                                                                                                       |
| Split Out             | Split Out                   | Split data for processing          | Selected                  | Method                  |                                                                                                       |
| Method                | Split In Batches            | Batch processing of URLs            | Split Out, Loop page      | FAQ Results, Maping Sitemap2 |                                                                                                   |
| Maping Sitemap2       | HTTP Request                | Fetch sitemap XML                  | Method                    | XML, Sitemap Error3      | It may give an error if the sitemap is not found or if there is more than one sitemap at a time or if it is not WordPress, contact support in case of error or manually enter the sitemap URL. |
| Sitemap Error3        | Stop and Error              | Stop on sitemap fetch/parsing error | Maping Sitemap2 (error)   | -                       |                                                                                                       |
| XML                   | XML                         | Parse sitemap XML                  | Maping Sitemap2           | Split Out1               |                                                                                                       |
| Split Out1            | Split Out                   | Split URLs for batch processing   | XML                       | Limit                   |                                                                                                       |
| Limit                 | Limit                       | Limit concurrent processing       | Split Out1                | Loop page                |                                                                                                       |
| Loop page             | Split In Batches            | Iterate over page URLs             | Limit                     | Method, HTTP Req page    |                                                                                                       |
| HTTP Req page         | HTTP Request                | Fetch page content                 | Loop page                 | Clear, Stop and Error1   |                                                                                                       |
| Stop and Error1       | Stop and Error              | Stop on HTTP request error         | HTTP Req page (error)      | -                       | Contact oriolrotllant3@gmail.com if you encounter this error.                                         |
| Clear                 | Code                        | Clean/reset before FAQ generation  | HTTP Req page             | meta descr and title gen |                                                                                                       |
| meta descr and title gen | OpenAI (Langchain node)    | Generate meta description & title | Clear                     | Loop page, Stop and Error |                                                                                                       |
| Stop and Error        | Stop and Error              | Stop on meta generation error      | meta descr and title gen (error) | -                |                                                                                                       |
| FAQ Results           | Code                        | Process FAQ output                 | Method                    | Add items DB             |                                                                                                       |
| Add items DB          | Split In Batches            | Batch data for DB and email       | FAQ Results, Append row in db | Text email, Append row in db |                                                                                                   |
| Append row in db      | Google Sheets               | Append FAQ data to Google Sheets  | Add items DB              | Add items DB, Error insert row |                                                                                                   |
| Error insert row      | Stop and Error              | Stop on DB insert error            | Append row in db (error)   | -                       | Contact oriolrotllant3@gmail.com if you encounter this error.                                         |
| Text email            | OpenAI (Langchain node)     | Generate email body text           | Add items DB              | Send a message           | I'm not including an error node here because if the API is incorrect, it would have already failed.   |
| Send a message        | Gmail                       | Send email with FAQ content        | Text email                | -                       |                                                                                                       |
| Sticky Note*          | Sticky Note                 | Explanations and warnings          | -                        | -                       | Various notes, including sitemap error handling advice and contact info.                              |

*Multiple sticky notes exist but mostly contain empty content or minor remarks.

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers:**  
   - Add a **Manual Trigger** node named `MANUAL`. Default settings.  
   - Add a **Schedule Trigger** node named `Schedule Trigger`. Configure desired schedule.

2. **User Agent Rotation:**  
   - Add a **Code** node named `UA Rotativo`. Implement JavaScript logic to generate or rotate user agent strings dynamically.

3. **Set Initial Options:**  
   - Add a **Set** node named `OPTIONS`. Define key workflow parameters such as the sitemap URL, batch sizes, or API keys.

4. **Select Sitemap URL:**  
   - Add a **Code** node named `Selected`. Write JavaScript to extract or validate the sitemap URL from the options.

5. **Split Sitemap URLs:**  
   - Add a **Split Out** node named `Split Out` to split incoming sitemap URLs or data.

6. **Batch URL Processing:**  
   - Add a **Split In Batches** node named `Method`. Set batch size accordingly (e.g., 10). This will feed the sitemap fetch and FAQ generation process.

7. **Fetch Sitemap XML:**  
   - Add an **HTTP Request** node named `Maping Sitemap2`. Configure as GET request to fetch sitemap XML. Use the user agent from `UA Rotativo`.

8. **Handle Sitemap Errors:**  
   - Add a **Stop and Error** node named `Sitemap Error3`. Connect as error branch from `Maping Sitemap2`.

9. **Parse Sitemap XML:**  
   - Add an **XML** node named `XML`. Configure to parse the response body of the sitemap request.

10. **Split Parsed URLs:**  
    - Add a **Split Out** node named `Split Out1` to split XML parsed URLs.

11. **Limit Concurrent Processing:**  
    - Add a **Limit** node named `Limit` to cap the number of URLs processed concurrently.

12. **Batch Loop Over Pages:**  
    - Add a **Split In Batches** node named `Loop page` to iterate over page URLs in batches.

13. **Fetch Page Content:**  
    - Add an **HTTP Request** node named `HTTP Req page`. Configure GET requests with retry enabled. Use rotating user agent.

14. **Handle Page Fetch Errors:**  
    - Add a **Stop and Error** node named `Stop and Error1`. Connect error output from `HTTP Req page`, with contact info in node notes.

15. **Clear Temporary Data:**  
    - Add a **Code** node named `Clear` to clean/reset any temporary data before FAQ generation.

16. **Generate Meta Description and Title:**  
    - Add an **OpenAI (Langchain)** node named `meta descr and title gen`. Configure with GPT-5 Nano model, proper prompt for meta generation. Enable retry on failure.

17. **Handle Meta Generation Errors:**  
    - Add a **Stop and Error** node named `Stop and Error`. Connect error output from meta generation node.

18. **Process FAQ Results:**  
    - Add a **Code** node named `FAQ Results`. Write code to transform OpenAI output into structured FAQ data.

19. **Batch FAQ for DB and Email:**  
    - Add a **Split In Batches** node named `Add items DB`. Prepare batches for database insertion and email content generation.

20. **Append Data to Google Sheets:**  
    - Add a **Google Sheets** node named `Append row in db`. Configure with proper credentials and target spreadsheet. Enable retries.

21. **Handle DB Insert Errors:**  
    - Add a **Stop and Error** node named `Error insert row`. Connect error output from Google Sheets node.

22. **Generate Email Text:**  
    - Add an **OpenAI (Langchain)** node named `Text email`. Use GPT-5 Nano to generate email body from FAQ data.

23. **Send Email Notification:**  
    - Add a **Gmail** node named `Send a message`. Configure with OAuth2 credentials, set recipient and message content from `Text email`.

24. **Connect Workflow:**  
    - Connect nodes following the structure: triggers → UA Rotativo → OPTIONS → Selected → Split Out → Method → Maping Sitemap2 → XML → Split Out1 → Limit → Loop page → HTTP Req page → Clear → meta descr and title gen → Loop page (loop)  
    - Additionally, Method connects to FAQ Results → Add items DB → Text email → Send a message  
    - Attach error nodes accordingly.

25. **Add Sticky Notes:**  
    - Optionally add sticky notes to document known issues, such as sitemap errors and support contact info.

26. **Configure Credentials:**  
    - Set up OpenAI credentials with GPT-5 Nano access.  
    - Set up Google Sheets credentials with appropriate access.  
    - Set up Gmail OAuth2 credentials for sending emails.

27. **Test Workflow:**  
    - Run manual trigger and verify each stage works, monitor for errors and API limits.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                             |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Sitemap retrieval may error if sitemap is missing, multiple sitemaps exist, or non-WordPress sites. Contact support or enter sitemap manually. | Sticky note on `Maping Sitemap2` node.                       |
| For critical errors (sitemap fetch, HTTP request, Google Sheets insert), contact oriolrotllant3@gmail.com. | Stop and Error1, Error insert row nodes notes.               |
| This workflow uses GPT-5 Nano via the n8n Langchain OpenAI node with retry enabled for robustness.          | OpenAI nodes configuration notes.                            |
| Google Sheets node uses retry with wait between tries set to 5000ms to handle transient API errors.         | Append row in db node configuration.                         |
| Email is sent using Gmail node with OAuth2 credentials; ensure proper permission scopes are granted.        | Send a message node.                                          |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.