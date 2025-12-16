Automate Personalized Assignment Reminders for Students with Canvas

https://n8nworkflows.xyz/workflows/automate-personalized-assignment-reminders-for-students-with-canvas-11746


# Automate Personalized Assignment Reminders for Students with Canvas

### 1. Workflow Overview

This workflow automates personalized assignment reminder messages to students enrolled in a specific Canvas course. It targets instructors or educational administrators who want to notify students about their pending (unsubmitted) assignments via Canvas Conversations.

The workflow consists of the following logical blocks:

- **1.1 Initialization & Input Parameters:** Triggering the workflow manually and setting initial parameters such as the Canvas base URL and the target course name.
- **1.2 Course Identification:** Fetching all courses associated with the instructor, aggregating paginated results, and filtering to find the course ID matching the given course name.
- **1.3 Student Retrieval:** Retrieving all students enrolled in the identified course, handling pagination, and aggregating student data.
- **1.4 Pending Assignments Retrieval:** For each student, fetching unsubmitted assignments (pending submissions) with pagination, then aggregating and enriching these submissions with student details and due date conversions.
- **1.5 Data Aggregation & Formatting:** Grouping pending assignments per student into a human-readable summary message, converting due dates to local time (GMT-3 São Paulo).
- **1.6 Sending Reminders:** Sending a personalized Canvas conversation message to each student listing their pending assignments.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Input Parameters

- **Overview:** This block triggers the workflow execution manually and sets initial fixed parameters including the Canvas base URL and the target course name for which reminders will be sent.
- **Nodes Involved:** 
  - When clicking ‘Execute workflow’
  - Initial parameters
  - Sticky Note6 (overview note)
- **Node Details:**

  - **When clicking ‘Execute workflow’**
    - Type: Manual Trigger
    - Role: Entry point to start the workflow manually.
    - Configuration: No parameters.
    - Inputs: None
    - Outputs: Connects to "Initial parameters".
    - Edge cases: None (manual trigger).
  
  - **Initial parameters**
    - Type: Set Node
    - Role: Defines static inputs for the workflow.
    - Configuration: Sets two variables:
      - `canvasLink`: Canvas base URL (e.g., https://pucminas.instructure.com)
      - `courseName`: Exact course name to find (e.g., "25_2 - 05 - [IEC_EEO_O5_T2_Online]Int.Art.Orc.")
    - Inputs: Receives from Manual Trigger.
    - Outputs: Connects to "Get all teacher courses".
    - Edge cases: Course name must exactly match Canvas course name (case-sensitive).
  
  - **Sticky Note6**
    - Provides a detailed textual overview of the workflow purpose and setup instructions.
    - Contextual information only, no input/output.

#### 2.2 Course Identification

- **Overview:** Retrieves all courses for the instructor, handles pagination, and filters the list to find the course matching the target course name. Then extracts and sets the course ID.
- **Nodes Involved:** 
  - Get all teacher courses
  - Agregate courses pagination
  - Get course by name
  - Get course ID
  - Sticky Note1
- **Node Details:**

  - **Get all teacher courses**
    - Type: HTTP Request (GET)
    - Role: Calls Canvas API `/api/v1/courses` to get all courses for the authenticated instructor.
    - Configuration:
      - URL: `${canvasLink}/api/v1/courses`
      - Authentication: HTTP Bearer token (Canvas API token)
      - Pagination: Follows `Link` header for next page URLs automatically.
    - Inputs: From "Initial parameters".
    - Outputs: Connects to "Agregate courses pagination".
    - Edge cases: API rate limits, auth token expiry, network issues.
  
  - **Agregate courses pagination**
    - Type: Code (Python)
    - Role: Flattens paginated course API responses into a single list of course objects.
    - Configuration: Iterates over all pages, extracts course list per page, emits one item per course.
    - Inputs: Multiple paginated HTTP responses from "Get all teacher courses".
    - Outputs: Connects to "Get course by name".
    - Edge cases: Malformed data or empty pages.
  
  - **Get course by name**
    - Type: Filter
    - Role: Filters the list of courses to find the one matching `courseName`.
    - Configuration: String equality match on course `name` field vs. input parameter `courseName` (case-sensitive).
    - Inputs: Flattened course list.
    - Outputs: Connects to "Get course ID".
    - Edge cases: No match found (empty output), multiple matches (only first used).
  
  - **Get course ID**
    - Type: Set Node
    - Role: Extracts and sets `idCurso` variable with the course ID from filtered course.
    - Inputs: Single course JSON from "Get course by name".
    - Outputs: Connects to "Get course students".
  
  - **Sticky Note1**
    - Explains the purpose: "Get course ID by course name".

#### 2.3 Student Retrieval

- **Overview:** Retrieves all students enrolled in the identified course, handling pagination, and emits a flattened list of students.
- **Nodes Involved:** 
  - Get course students
  - Agregate students pagination
  - Sticky Note2
- **Node Details:**

  - **Get course students**
    - Type: HTTP Request (GET)
    - Role: Calls Canvas API `/api/v1/courses/{idCurso}/search_users` with `enrollment_type[]=student` to get all students.
    - Configuration:
      - Dynamic URL using course ID.
      - Pagination follows `Link` headers.
      - Auth: Canvas API Bearer token.
    - Inputs: Receives course ID from "Get course ID".
    - Outputs: Connects to "Agregate students pagination".
    - Edge cases: API errors, empty student list, pagination failures.
  
  - **Agregate students pagination**
    - Type: Code (Python)
    - Role: Flattens paginated student API responses into a single list of student objects.
    - Inputs: Multiple paginated HTTP responses.
    - Outputs: Connects to "Get course sumbissions".
    - Edge cases: Empty pages, malformed data.
  
  - **Sticky Note2**
    - Explains: "Get students by course ID".

#### 2.4 Pending Assignments Retrieval

- **Overview:** For each student, fetches their unsubmitted (pending) assignments using Canvas API, aggregates paginated results, and enriches these submissions with student details and formatted due dates.
- **Nodes Involved:** 
  - Get course sumbissions
  - Agregate submits pagination
  - Enrich Submissions with Student Data
  - Sticky Note3, Sticky Note4
- **Node Details:**

  - **Get course sumbissions**
    - Type: HTTP Request (GET)
    - Role: Calls Canvas API `/api/v1/courses/{idCurso}/students/submissions` filtering by student IDs and unsubmitted workflow state.
    - Configuration:
      - URL dynamically built with course ID.
      - Query parameters: `student_ids[]` dynamic per student, `include[]=assignment`, `workflow_state=unsubmitted`.
      - Pagination following `Link` header.
      - Auth: Canvas API Bearer token.
    - Inputs: Receives student list from "Agregate students pagination".
    - Outputs: Connects to "Agregate submits pagination".
    - Edge cases: Large student lists may cause API limits, pagination errors, no unsubmitted assignments.
  
  - **Agregate submits pagination**
    - Type: Code (Python)
    - Role: Flattens paginated responses of assignment submissions into a single list.
    - Inputs: Multiple HTTP responses from "Get course sumbissions".
    - Outputs: Connects to "Enrich Submissions with Student Data".
    - Edge cases: Missing assignment data, empty pages.
  
  - **Enrich Submissions with Student Data**
    - Type: Code (Python)
    - Role: Joins submission data with student data by student ID, extracts relevant fields, converts assignment due dates from UTC to GMT-3 formatted strings.
    - Inputs: Flattened submissions and student data (from previous nodes).
    - Outputs: Connects to "Group Pending Assignments per Student".
    - Key expressions:
      - Converts ISO 8601 UTC due dates to `DD/MM/YYYY HH:mm:ss` in GMT-3 (São Paulo time).
    - Edge cases: Missing due dates, parsing errors, missing student info.
  
  - **Sticky Note3** and **Sticky Note4**
    - Provide explanations for "Get pending submissions by course ID and student ID" and "Organize and format pending submissions information".

#### 2.5 Data Aggregation & Formatting

- **Overview:** Groups all pending assignments per student, builds a consolidated human-readable summary message listing assignments with titles, due dates, and links.
- **Nodes Involved:** 
  - Group Pending Assignments per Student
  - Sticky Note4 (shared with prior block)
- **Node Details:**

  - **Group Pending Assignments per Student**
    - Type: Code (Python)
    - Role: Aggregates submissions by student, builds a multi-line text summary listing each pending assignment with formatted due date and link.
    - Inputs: Enriched submission data from "Enrich Submissions with Student Data".
    - Outputs: Connects to "Send message in Canvas".
    - Key logic:
      - Groups by `student_id`.
      - Formats due dates to São Paulo local time (Brazilian format `DD/MM/YYYY HH:mm`).
      - Builds a message starting with a greeting, then enumerates assignments or states no pending assignments.
    - Edge cases: Students with no pending assignments, missing assignment names or URLs.

#### 2.6 Sending Reminders

- **Overview:** Sends a Canvas conversation message to each student listing their personalized pending assignments summary.
- **Nodes Involved:** 
  - Send message in Canvas
  - Sticky Note5
- **Node Details:**

  - **Send message in Canvas**
    - Type: HTTP Request (POST)
    - Role: Calls Canvas API `/api/v1/conversations` to send a conversation message.
    - Configuration:
      - Body parameters:
        - `recipients[]`: student ID
        - `subject`: course original name + " - Atividades pendentes"
        - `body`: pending assignments summary text
        - `force_new`: 1 (forces new conversation)
        - `context_code`: "course_{course_id}"
      - Content-Type: `application/x-www-form-urlencoded`
      - Auth: Canvas API Bearer token
    - Inputs: Aggregated messages per student from "Group Pending Assignments per Student".
    - Outputs: None (end node).
    - Edge cases: Message failing to send, invalid recipient IDs, API limits.
  
  - **Sticky Note5**
    - Explains: "Send pending submissions alert to students".

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                                   | Input Node(s)                 | Output Node(s)                | Sticky Note                                      |
|----------------------------|---------------------|-------------------------------------------------|------------------------------|------------------------------|-------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Workflow entry point                             | None                         | Initial parameters            |                                                 |
| Initial parameters          | Set                 | Set Canvas URL and target course name            | When clicking ‘Execute workflow’ | Get all teacher courses       | Sticky Note6 covers initialization explanation  |
| Get all teacher courses     | HTTP Request        | Fetch all instructor courses                      | Initial parameters            | Agregate courses pagination   |                                                 |
| Agregate courses pagination | Code (Python)       | Flatten paginated course results                  | Get all teacher courses       | Get course by name            |                                                 |
| Get course by name          | Filter              | Filter courses by exact course name               | Agregate courses pagination   | Get course ID                 | Sticky Note1 explains course ID lookup          |
| Get course ID               | Set                 | Extract and set course ID                          | Get course by name            | Get course students           |                                                 |
| Get course students         | HTTP Request        | Get all enrolled students (pagination)            | Get course ID                 | Agregate students pagination  | Sticky Note2 covers student retrieval            |
| Agregate students pagination| Code (Python)       | Flatten paginated student results                  | Get course students           | Get course sumbissions        |                                                 |
| Get course sumbissions      | HTTP Request        | Fetch unsubmitted assignments per student          | Agregate students pagination  | Agregate submits pagination   | Sticky Note3 covers submission retrieval        |
| Agregate submits pagination | Code (Python)       | Flatten paginated submission results               | Get course sumbissions        | Enrich Submissions with Student Data |                                                 |
| Enrich Submissions with Student Data | Code (Python)       | Join submissions with student info and format due dates | Agregate submits pagination   | Group Pending Assignments per Student | Sticky Note4 covers data enrichment and formatting |
| Group Pending Assignments per Student | Code (Python)       | Aggregate assignments per student and build summary message | Enrich Submissions with Student Data | Send message in Canvas        | Sticky Note4 covers data aggregation             |
| Send message in Canvas      | HTTP Request        | Send Canvas conversation with pending tasks       | Group Pending Assignments per Student | None                        | Sticky Note5 covers message sending              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**
   - Name: When clicking ‘Execute workflow’
   - No parameters.
   - This will manually start the workflow.

2. **Create Set node**
   - Name: Initial parameters
   - Set two static fields:
     - `canvasLink` (string): e.g. `https://pucminas.instructure.com`
     - `courseName` (string): exact Canvas course name, e.g. `"25_2 - 05 - [IEC_EEO_O5_T2_Online]Int.Art.Orc."`
   - Connect from Manual Trigger.

3. **Create HTTP Request node**
   - Name: Get all teacher courses
   - Method: GET
   - URL: `={{ $json.canvasLink }}/api/v1/courses`
   - Authentication: Use HTTP Bearer Auth credential with Canvas API token.
   - Pagination:
     - Enable pagination mode: `responseContainsNextURL`
     - Next URL expression: parse `Link` header for `rel="next"`
   - Connect from Initial parameters.

4. **Create Code node**
   - Name: Agregate courses pagination
   - Language: Python
   - Paste the provided Python code to flatten paginated course lists.
   - Connect from "Get all teacher courses".

5. **Create Filter node**
   - Name: Get course by name
   - Configure filter:
     - Condition: `$json.name === $('Initial parameters').first().json.courseName`
     - Case sensitive, strict type validation.
   - Connect from "Agregate courses pagination".

6. **Create Set node**
   - Name: Get course ID
   - Set `idCurso` to `={{ $json.id }}`
   - Connect from "Get course by name".

7. **Create HTTP Request node**
   - Name: Get course students
   - Method: GET
   - URL: `={{ $('Initial parameters').first().json.canvasLink }}/api/v1/courses/{{ $('Get course ID').first().json.idCurso }}/search_users`
   - Query Parameters: `enrollment_type[]=student`
   - Authentication: Canvas Bearer token
   - Pagination: Enable as above using `Link` header.
   - Connect from "Get course ID".

8. **Create Code node**
   - Name: Agregate students pagination
   - Language: Python
   - Paste provided Python code to flatten paginated students.
   - Connect from "Get course students".

9. **Create HTTP Request node**
   - Name: Get course sumbissions
   - Method: GET
   - URL: `={{ $('Initial parameters').first().json.canvasLink }}/api/v1/courses/{{ $('Get course ID').first().json.idCurso }}/students/submissions`
   - Query Parameters (repeat for each student):
     - `student_ids[]` = `={{ $json.id }}`
     - `include[]` = `assignment`
     - `workflow_state` = `unsubmitted`
   - Authentication: Canvas Bearer token
   - Pagination: Enable as above.
   - Connect from "Agregate students pagination".

10. **Create Code node**
    - Name: Agregate submits pagination
    - Language: Python
    - Paste provided Python code to flatten paginated submissions.
    - Connect from "Get course sumbissions".

11. **Create Code node**
    - Name: Enrich Submissions with Student Data
    - Language: Python
    - Paste provided Python code that:
      - Joins submissions with students by user ID.
      - Extracts student and assignment details.
      - Converts due dates from UTC ISO8601 to `DD/MM/YYYY HH:mm:ss` GMT-3.
    - Connect from "Agregate submits pagination".

12. **Create Code node**
    - Name: Group Pending Assignments per Student
    - Language: Python
    - Paste provided Python code that:
      - Groups all pending assignments by student ID.
      - Builds a multi-line text message listing assignments per student.
      - Formats due dates to São Paulo local time.
    - Connect from "Enrich Submissions with Student Data".

13. **Create HTTP Request node**
    - Name: Send message in Canvas
    - Method: POST
    - URL: `={{ $('Initial parameters').first().json.canvasLink }}/api/v1/conversations`
    - Content-Type: `application/x-www-form-urlencoded`
    - Body Parameters (form-urlencoded):
      - `recipients[]`: `={{ $json.student_id }}`
      - `subject`: `={{ $('Get course by name').first().json.original_name }} - Atividades pendentes`
      - `body`: `={{ $json.pending_summary }}`
      - `force_new`: `1`
      - `context_code`: `={{ 'course_' + $('Get course by name').first().json.id }}`
    - Authentication: Canvas Bearer token
    - Connect from "Group Pending Assignments per Student".

14. **Credentials Setup**
    - Create HTTP Bearer Auth credential in n8n with your Canvas API token.
    - Assign this credential to all HTTP Request nodes that access Canvas.

15. **Run and Test**
    - Manually trigger the workflow.
    - Verify that it fetches the correct course, retrieves students, gathers pending assignments, and sends personalized Canvas messages.
    - Adjust course name or base URL parameters as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow purpose: Automate sending personalized reminders to students with unsubmitted assignments in Canvas courses.                                                                                                        | Workflow description                                                                                   |
| Setup instructions and workflow logic overview are included in Sticky Note6 within the workflow.                                                                                                                            | Sticky Note content                                                                                     |
| Canvas API Documentation for Conversations: https://canvas.instructure.com/doc/api/conversations.html                                                                                                                      | For understanding message sending node                                                                |
| Due date conversion uses fixed GMT-3 offset and also attempts timezone conversion to America/Sao_Paulo for formatting, requiring Python 3.9+ (zoneinfo module).                                                               | Code nodes "Enrich Submissions with Student Data" and "Group Pending Assignments per Student"          |
| Pagination handling uses standard Canvas API `Link` header parsing to fetch all pages from the API endpoints.                                                                                                                | HTTP Request node pagination configuration                                                            |
| Ensure the Canvas API token has sufficient permissions to read courses, enrollments, submissions, and send conversations.                                                                                                    | Credential and permission setup                                                                        |
| Manual trigger node allows testing on demand; production could use scheduled triggers or event-based triggers as needed.                                                                                                     | Workflow trigger setup                                                                                  |

---

**Disclaimer:** The provided text and workflow structure are based solely on the given n8n workflow JSON and Canvas API usage. No illegal, offensive, or protected data is processed. This workflow respects all applicable content and usage policies.