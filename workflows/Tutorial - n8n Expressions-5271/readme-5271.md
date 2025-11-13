Tutorial - n8n Expressions

https://n8nworkflows.xyz/workflows/tutorial---n8n-expressions-5271


# Tutorial - n8n Expressions

### 1. Workflow Overview

This workflow is a comprehensive tutorial designed to teach users how to utilize **n8n expressions** to dynamically access, manipulate, and combine data across multiple nodes within an n8n workflow. It demonstrates how to pull data from one node and use it in another, showing progressively advanced techniques for working with JSON, arrays, nested objects, JavaScript functions, and multiple items.

The workflow is structured into logical blocks representing lessons, each focusing on a specific expression concept or technique:

- **1.1 Input Reception:** Starting point and data setup with sample data.
- **1.2 Basic Data Access:** Accessing simple values and using selectors.
- **1.3 Working with Arrays and Nested Data:** Accessing array elements and nested object properties.
- **1.4 Combining Data and JavaScript Functions:** Complex expression usage including array of objects and JavaScript functions.
- **1.5 Object Inspection and Utility Functions:** Extracting keys, converting objects to strings.
- **1.6 Handling Multiple Items:** Working with multiple output items and aggregation.
- **1.7 Final Assembly:** Combining all learned concepts into a final summary.

Each block is also accompanied by **sticky notes** explaining the concepts in plain English.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block contains the initial manual trigger and a node that sets up all sample data used throughout the tutorial. It serves as the "source of truth" for all subsequent lessons.

- **Nodes Involved:**  
  - Start Tutorial  
  - Source Data  
  - Sticky Note (Tutorial Introduction)  
  - Sticky Note1 (Data Source Explanation)

- **Node Details:**

  - **Start Tutorial**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow  
    - Config: No parameters  
    - Connections: Outputs to "Source Data" node  
    - Edge Cases: None (manual trigger)

  - **Source Data**  
    - Type: Set  
    - Role: Provides sample JSON data with various types (string, number, boolean, array, object)  
    - Config: Sets fields:
      - `name` (string): "Alice"
      - `age` (number): 30
      - `is_active` (boolean): true
      - `skills` (array): ["JavaScript", "Python", "n8n"]
      - `projects` (array of objects): [{name: "Project A", status: "Done"}, {name: "Project B", status: "In Progress"}]
      - `contact` (object): {email: "alice@example.com", phone: null}
    - Connections: Outputs to "1. The Basics" node  
    - Edge Cases: None; static data

  - **Sticky Note (Tutorial Introduction)**  
    - Type: Sticky Note  
    - Role: Explains the overall tutorial purpose and how to use the workflow  
    - Content: Introduction to expressions and instructions for following the tutorial

  - **Sticky Note1 (Data Source Explanation)**  
    - Type: Sticky Note  
    - Role: Describes the sample data structure in "Source Data" node

---

#### 1.2 Basic Data Access

- **Overview:**  
  Introduces how to access simple values from the "Source Data" node using n8n expressions, including the use of selectors like `.item` and `.last()`.

- **Nodes Involved:**  
  - 1. The Basics  
  - 2. The n8n Selectors  
  - Sticky Note2 (Lesson 1 Explanation)  
  - Sticky Note9 (Lesson 2 Explanation)

- **Node Details:**

  - **1. The Basics**  
    - Type: Set  
    - Role: Demonstrates accessing a simple string value (`name`) from "Source Data" with expression `{{ $('Source Data').item.json.name }}`  
    - Config: Sets `user_name` field using the expression  
    - Input: "Source Data"  
    - Output: "2. The n8n Selectors"  
    - Edge Cases: Expression failure if "Source Data" missing or field renamed

  - **2. The n8n Selectors**  
    - Type: Set  
    - Role: Shows use of `.last()` selector to access the last item explicitly  
    - Config: Sets `user_name_from_first` with expression `{{ $('Source Data').last().json.name }}`  
    - Input: "1. The Basics"  
    - Output: "3. Working with Arrays"  
    - Edge Cases: Same as above; `.last()` safer if multiple items exist

  - **Sticky Note2 (Lesson 1 Explanation)**  
    - Explains the purpose of expressions and breakdown of accessing simple values

  - **Sticky Note9 (Lesson 2 Explanation)**  
    - Explains `.first()`, `.last()`, and `.all()` selectors and why `.last()` is often preferred

---

#### 1.3 Working with Arrays and Nested Data

- **Overview:**  
  Demonstrates how to access array elements and nested object properties using expressions.

- **Nodes Involved:**  
  - 3. Working with Arrays  
  - 4. Going Deeper  
  - 5. The Combo Move  
  - Sticky Note3 (Lesson 3 Explanation)  
  - Sticky Note4 (Lesson 4 Explanation)  
  - Sticky Note5 (Lesson 5 Explanation)

- **Node Details:**

  - **3. Working with Arrays**  
    - Type: Set  
    - Role: Accesses second element in the `skills` array (`skills[1]`)  
    - Config: `second_skill` = `{{ $('Source Data').last().json.skills[1] }}`  
    - Input: "2. The n8n Selectors"  
    - Output: "4. Going Deeper"  
    - Edge Cases: Index out of range if skills array is empty or shorter than 2

  - **4. Going Deeper**  
    - Type: Set  
    - Role: Accesses nested property `contact.email`  
    - Config: `user_email` = `{{ $('Source Data').last().json.contact.email }}`  
    - Input: "3. Working with Arrays"  
    - Output: "5. The Combo Move"  
    - Edge Cases: Null or missing `contact` or `email` keys cause undefined value

  - **5. The Combo Move**  
    - Type: Set  
    - Role: Access status of first project in `projects` array of objects  
    - Config: `first_project_status` = `{{ $('Source Data').last().json.projects[0].status }}`  
    - Input: "4. Going Deeper"  
    - Output: "6. A Touch of Magic"  
    - Edge Cases: Empty `projects` array or missing `status` key

  - **Sticky Note3 (Lesson 3 Explanation)**  
    - Explains array indexing and zero-based indexing

  - **Sticky Note4 (Lesson 4 Explanation)**  
    - Describes accessing nested objects step-by-step

  - **Sticky Note5 (Lesson 5 Explanation)**  
    - Shows combining array and nested object access in one expression

---

#### 1.4 Combining Data and JavaScript Functions

- **Overview:**  
  Introduces JavaScript functions within expressions to manipulate and inspect data values.

- **Nodes Involved:**  
  - 6. A Touch of Magic  
  - 7. Inspecting Objects  
  - Sticky Note6 (Lesson 6 Explanation)  
  - Sticky Note10 (Lesson 7 Explanation)

- **Node Details:**

  - **6. A Touch of Magic**  
    - Type: Set  
    - Role: Applies JavaScript functions to data:
      - Converts `name` to uppercase
      - Converts `age` to dog years (rounded)
      - Checks data type of `age`
    - Config:
      - `name_in_caps` = `{{ $('Source Data').last().json.name.toUpperCase() }}`
      - `age_in_dog_years` = `{{ Math.round($('Source Data').last().json.age / 7) }}`
      - `age_data_type` = `{{ typeof $('Source Data').last().json.age }}`
    - Input: "5. The Combo Move"  
    - Output: "7. Inspecting Objects"  
    - Edge Cases: `name` missing or not string causes `.toUpperCase()` failure; `age` missing or not number affects math

  - **7. Inspecting Objects**  
    - Type: Set  
    - Role: Uses `Object.keys()` to get all keys in the `contact` object  
    - Config: `contact_keys` = `{{ Object.keys($('Source Data').last().json.contact) }}`  
    - Input: "6. A Touch of Magic"  
    - Output: "8. Utility Functions"  
    - Edge Cases: `contact` missing or not object causes error

  - **Sticky Note6 (Lesson 6 Explanation)**  
    - Explains usage of JavaScript string and Math functions and `typeof` operator within expressions

  - **Sticky Note10 (Lesson 7 Explanation)**  
    - Describes usefulness of `Object.keys()` to dynamically inspect objects

---

#### 1.5 Object Inspection and Utility Functions

- **Overview:**  
  Demonstrates converting JSON objects to formatted string and passing arrays unchanged for further processing.

- **Nodes Involved:**  
  - 8. Utility Functions  
  - Sticky Note11 (Lesson 8 Explanation)

- **Node Details:**

  - **8. Utility Functions**  
    - Type: Set  
    - Role:  
      - Converts `contact` object to pretty-printed JSON string  
      - Passes `skills` array as is for later splitting  
    - Config:  
      - `contact_as_string` = `{{ JSON.stringify($('Source Data').last().json.contact, null, 2) }}`  
      - `skills` = `{{ $('Source Data').last().json.skills }}`  
    - Input: "7. Inspecting Objects"  
    - Output: "Split Out Skills"  
    - Edge Cases: `contact` missing causes empty or error string; skills must be array

  - **Sticky Note11 (Lesson 8 Explanation)**  
    - Explains `JSON.stringify()` usage for converting objects to readable strings for external usage

---

#### 1.6 Handling Multiple Items

- **Overview:**  
  Shows how to work with multiple items outputted from a node and aggregate data using `$items()` and array functions.

- **Nodes Involved:**  
  - Split Out Skills  
  - 9. The "All Items" View  
  - Sticky Note7 (Lesson 9 Explanation)

- **Node Details:**

  - **Split Out Skills**  
    - Type: Split Out  
    - Role: Splits the array field `skills` into multiple output items while preserving other fields  
    - Config:  
      - `fieldToSplitOut`: "skills"  
      - `include`: All other fields included  
    - Input: "8. Utility Functions"  
    - Output: "9. The \"All Items\" View"  
    - Edge Cases: Empty or missing `skills` array results in no output items

  - **9. The "All Items" View**  
    - Type: Set  
    - Role: Aggregates all output items from "Split Out Skills" into a single comma-separated string  
    - Config:  
      - `all_skills_string` = `{{ $('Split Out Skills').all().map(item => item.json.skills).join(', ') }}`  
      - `executeOnce`: true (runs once rather than per item)  
    - Input: "Split Out Skills"  
    - Output: "Final Exam"  
    - Edge Cases: No items output from previous node results in empty string

  - **Sticky Note7 (Lesson 9 Explanation)**  
    - Explains how `$items()` and arrow functions are used to process multiple items

---

#### 1.7 Final Assembly

- **Overview:**  
  Combines all previously accessed and processed data into a final summary string using multiple expressions.

- **Nodes Involved:**  
  - Final Exam  
  - Sticky Note8 (Final Exam Explanation)

- **Node Details:**

  - **Final Exam**  
    - Type: Set  
    - Role: Constructs a summary string combining data points from multiple nodes to demonstrate full workflow mastery  
    - Config:  
      - `final_summary` field includes:
        ```
        User {{ $('2. The n8n Selectors').last().json.user_name_from_first }} is {{ $('Source Data').last().json.age }}.

        Their best skill is {{ $('3. Working with Arrays').last().json.second_skill }}.

        Their first project was {{ $('Source Data').last().json.projects[0].name }}, which is now {{ $('5. The Combo Move').last().json.first_project_status }}.

        All skills: {{ $('9. The "All Items" View').last().json.all_skills_string }}.
        ```
    - Input: "9. The \"All Items\" View"  
    - Output: None (end of workflow)  
    - Edge Cases: Any missing intermediate nodes or fields cause incomplete summary or errors

  - **Sticky Note8 (Final Exam Explanation)**  
    - Encourages learners by showing how the final node ties all concepts together

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                                 | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                                                                    |
|------------------------|---------------------|------------------------------------------------|-------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Start Tutorial         | Manual Trigger      | Entry point to start workflow manually          |                         | Source Data                   |                                                                                                                                                |
| Source Data            | Set                 | Provides sample JSON data for tutorial          | Start Tutorial           | 1. The Basics                 | ## Our Data Source: Explains the sample data structure                                                                                         |
| Sticky Note            | Sticky Note         | Tutorial introduction and usage instructions    |                         |                               | # Tutorial - Mastering n8n Expressions: Introduction and overview                                                                             |
| Sticky Note1           | Sticky Note         | Explains the sample data in "Source Data"       |                         |                               | ## Our Data Source: Describes data types and fields                                                                                           |
| 1. The Basics          | Set                 | Access simple value (`name`) from source data   | Source Data              | 2. The n8n Selectors          | ## Lesson 1: Accessing a Simple Value: Expression breakdown                                                                                     |
| Sticky Note2           | Sticky Note         | Explains Lesson 1 expression                     |                         |                               | ## Lesson 1: Accessing a Simple Value                                                                                                         |
| 2. The n8n Selectors   | Set                 | Demonstrates `.last()` selector usage            | 1. The Basics            | 3. Working with Arrays        | ## Lesson 2: The n8n Selectors: Explains `.first()`, `.last()`, `.all()` selectors                                                             |
| Sticky Note9           | Sticky Note         | Explains Lesson 2 selectors                       |                         |                               | ## Lesson 2: The n8n Selectors                                                                                                                |
| 3. Working with Arrays | Set                 | Access second element of array (`skills[1]`)    | 2. The n8n Selectors     | 4. Going Deeper              | ## Lesson 3: Accessing an Array Element                                                                                                       |
| Sticky Note3           | Sticky Note         | Explains array indexing                           |                         |                               | ## Lesson 3: Accessing an Array Element                                                                                                       |
| 4. Going Deeper        | Set                 | Access nested object property (`contact.email`) | 3. Working with Arrays   | 5. The Combo Move            | ## Lesson 4: Accessing Nested Data                                                                                                            |
| Sticky Note4           | Sticky Note         | Explains nested data access                       |                         |                               | ## Lesson 4: Accessing Nested Data                                                                                                            |
| 5. The Combo Move      | Set                 | Access array of objects (`projects[0].status`)  | 4. Going Deeper          | 6. A Touch of Magic          | ## Lesson 5: Accessing Data in an Array of Objects                                                                                            |
| Sticky Note5           | Sticky Note         | Explains accessing array of objects              |                         |                               | ## Lesson 5: Accessing Data in an Array of Objects                                                                                            |
| 6. A Touch of Magic    | Set                 | Use JS functions to transform and inspect data  | 5. The Combo Move        | 7. Inspecting Objects         | ## Lesson 6: A Touch of Magic (JS Functions)                                                                                                 |
| Sticky Note6           | Sticky Note         | Explains JS functions usage                       |                         |                               | ## Lesson 6: A Touch of Magic (JS Functions)                                                                                                 |
| 7. Inspecting Objects  | Set                 | Use `Object.keys()` to get keys of `contact`    | 6. A Touch of Magic      | 8. Utility Functions          | ## Lesson 7: Inspecting Objects                                                                                                               |
| Sticky Note10          | Sticky Note         | Explains `Object.keys()` usage                    |                         |                               | ## Lesson 7: Inspecting Objects                                                                                                               |
| 8. Utility Functions   | Set                 | Convert object to string; pass array forward     | 7. Inspecting Objects    | Split Out Skills              | ## Lesson 8: Utility Functions (`JSON.stringify()`)                                                                                          |
| Sticky Note11          | Sticky Note         | Explains `JSON.stringify()`                       |                         |                               | ## Lesson 8: Utility Functions                                                                                                               |
| Split Out Skills       | Split Out           | Splits `skills` array into multiple output items| 8. Utility Functions     | 9. The "All Items" View       |                                                                                                                                                |
| 9. The "All Items" View| Set                 | Aggregate all skills into comma-separated string | Split Out Skills         | Final Exam                   | ## Lesson 9: Working with Multiple Items (`$items` & Arrow Functions)                                                                        |
| Final Exam             | Set                 | Combine all learned data into final summary      | 9. The "All Items" View  |                               | ## 🎓 FINAL EXAM: Putting It All Together                                                                                                    |
| Sticky Note8           | Sticky Note         | Explains final exam node and summary construction|                         |                               | ## 🎓 FINAL EXAM: Putting It All Together                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: *Start Tutorial*  
   - Purpose: Entry point to manually start the workflow

2. **Create a Set Node**  
   - Name: *Source Data*  
   - Purpose: Define sample data for tutorial  
   - Configuration:
     - Add fields with exact types and values:  
       - `name` (string): "Alice"  
       - `age` (number): 30  
       - `is_active` (boolean): true  
       - `skills` (array): `["JavaScript","Python","n8n"]`  
       - `projects` (array of objects): `[{"name":"Project A","status":"Done"},{"name":"Project B","status":"In Progress"}]`  
       - `contact` (object): `{"email":"alice@example.com","phone":null}`  
   - Connect *Start Tutorial* → *Source Data*

3. **Create a Set Node**  
   - Name: *1. The Basics*  
   - Purpose: Access simple field `name` from *Source Data*  
   - Add field:  
     - `user_name` (string)  
     - Value: Expression `{{ $('Source Data').item.json.name }}`  
   - Connect *Source Data* → *1. The Basics*

4. **Create a Set Node**  
   - Name: *2. The n8n Selectors*  
   - Purpose: Use `.last()` selector to get `name`  
   - Add field:  
     - `user_name_from_first` (string)  
     - Value: `{{ $('Source Data').last().json.name }}`  
   - Connect *1. The Basics* → *2. The n8n Selectors*

5. **Create a Set Node**  
   - Name: *3. Working with Arrays*  
   - Purpose: Access second skill (`skills[1]`)  
   - Add field:  
     - `second_skill` (string)  
     - Value: `{{ $('Source Data').last().json.skills[1] }}`  
   - Connect *2. The n8n Selectors* → *3. Working with Arrays*

6. **Create a Set Node**  
   - Name: *4. Going Deeper*  
   - Purpose: Access nested `contact.email`  
   - Add field:  
     - `user_email` (string)  
     - Value: `{{ $('Source Data').last().json.contact.email }}`  
   - Connect *3. Working with Arrays* → *4. Going Deeper*

7. **Create a Set Node**  
   - Name: *5. The Combo Move*  
   - Purpose: Access first project status (`projects[0].status`)  
   - Add field:  
     - `first_project_status` (string)  
     - Value: `{{ $('Source Data').last().json.projects[0].status }}`  
   - Connect *4. Going Deeper* → *5. The Combo Move*

8. **Create a Set Node**  
   - Name: *6. A Touch of Magic*  
   - Purpose: Use JS functions to transform and inspect data  
   - Add fields:  
     - `name_in_caps` (string): `{{ $('Source Data').last().json.name.toUpperCase() }}`  
     - `age_in_dog_years` (number): `{{ Math.round($('Source Data').last().json.age / 7) }}`  
     - `age_data_type` (string): `{{ typeof $('Source Data').last().json.age }}`  
   - Connect *5. The Combo Move* → *6. A Touch of Magic*

9. **Create a Set Node**  
   - Name: *7. Inspecting Objects*  
   - Purpose: List keys of `contact` object  
   - Add field:  
     - `contact_keys` (array): `{{ Object.keys($('Source Data').last().json.contact) }}`  
   - Connect *6. A Touch of Magic* → *7. Inspecting Objects*

10. **Create a Set Node**  
    - Name: *8. Utility Functions*  
    - Purpose: Convert `contact` to string and pass `skills` array  
    - Add fields:  
      - `contact_as_string` (string): `{{ JSON.stringify($('Source Data').last().json.contact, null, 2) }}`  
      - `skills` (array): `{{ $('Source Data').last().json.skills }}`  
    - Connect *7. Inspecting Objects* → *8. Utility Functions*

11. **Create a Split Out Node**  
    - Name: *Split Out Skills*  
    - Purpose: Split `skills` array into multiple items  
    - Configuration:  
      - Field to split out: `skills`  
      - Include all other fields: true  
    - Connect *8. Utility Functions* → *Split Out Skills*

12. **Create a Set Node**  
    - Name: *9. The "All Items" View*  
    - Purpose: Aggregate all split skills into a single string  
    - Add field:  
      - `all_skills_string` (string): `{{ $('Split Out Skills').all().map(item => item.json.skills).join(', ') }}`  
    - Set `Execute Once` to true  
    - Connect *Split Out Skills* → *9. The "All Items" View*

13. **Create a Set Node**  
    - Name: *Final Exam*  
    - Purpose: Combine all learned data into a final summary string  
    - Add field:  
      - `final_summary` (string) with value (multiline expression):
        ```
        User {{ $('2. The n8n Selectors').last().json.user_name_from_first }} is {{ $('Source Data').last().json.age }}.

        Their best skill is {{ $('3. Working with Arrays').last().json.second_skill }}.

        Their first project was {{ $('Source Data').last().json.projects[0].name }}, which is now {{ $('5. The Combo Move').last().json.first_project_status }}.

        All skills: {{ $('9. The "All Items" View').last().json.all_skills_string }}.
        ```
    - Connect *9. The "All Items" View* → *Final Exam*

14. **Add Sticky Notes**  
    - Add all sticky notes as per content and position described in the original workflow, associating each note with the relevant lesson node(s).

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is a step-by-step guide to mastering n8n expressions using sample data and progressive lessons. | Tutorial Introduction Sticky Note                                                              |
| Expressions are written inside double curly braces `{{ }}` and allow dynamic data access and manipulation.  | General explanation throughout sticky notes                                                    |
| Use `.last()` selector to safely access the last output item to avoid errors when nodes output multiple items.| Lesson 2 Sticky Note                                                                           |
| JavaScript functions can be used inside expressions to transform data (e.g., `.toUpperCase()`, `Math.round()`).| Lesson 6 Sticky Note                                                                           |
| Use `Object.keys()` to dynamically inspect object keys in JSON data.                                         | Lesson 7 Sticky Note                                                                           |
| Use `JSON.stringify()` to convert objects to JSON strings, useful for sending data to external services.     | Lesson 8 Sticky Note                                                                           |
| `$items()` and arrow functions allow processing multiple items and aggregation.                              | Lesson 9 Sticky Note                                                                           |
| Final summary node demonstrates how to combine data from multiple sources using complex expressions.         | Final Exam Sticky Note                                                                         |

---

**Disclaimer:**  
The provided text comes exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.