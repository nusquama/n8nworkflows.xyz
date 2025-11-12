Create an array of objects

https://n8nworkflows.xyz/workflows/create-an-array-of-objects-767


# Create an array of objects

### 1. Workflow Overview

This workflow demonstrates how to create and manipulate an array of objects within n8n using Function nodes. It is designed for scenarios where you need to construct a dataset programmatically and then aggregate or transform it into a specific structure for further processing or output.

Logical blocks included:

- **1.1 Data Creation:** Generates an initial array of individual JSON objects.
- **1.2 Array Aggregation:** Combines the individual objects into a single array encapsulated within a new JSON object.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Creation

- **Overview:**  
  This block programmatically creates mock data consisting of three individual JSON objects, each representing a person with an `id` and `name`.

- **Nodes Involved:**  
  - Mock Data (Function node)

- **Node Details:**  

  - **Mock Data**  
    - **Type and Role:** Function node; generates static mock data as an array of JSON objects.  
    - **Configuration:**  
      The function returns an array of three items, each item containing a `json` property with two fields: `id` (number) and `name` (string).  
      ```js
      return [
        { json: { id: 1, name: "Jim" } },
        { json: { id: 2, name: "Stefan" } },
        { json: { id: 3, name: "Hans" } }
      ];
      ```  
    - **Input/Output:**  
      No input data. Outputs an array of three JSON items.  
    - **Key expressions:** None beyond the static array construction.  
    - **Potential failures:**  
      Minimal risk since this is static code. Syntax errors could prevent workflow execution.  
    - **Version Requirements:** Compatible with n8n versions supporting Function nodes (generally all recent versions).  
    - **Sub-workflow:** None.

#### 1.2 Array Aggregation

- **Overview:**  
  This block aggregates the individual JSON objects from the previous node into a single JSON object containing one property, `data_object`, which holds an array of these objects.

- **Nodes Involved:**  
  - Create an array of objects (Function node)

- **Node Details:**  

  - **Create an array of objects**  
    - **Type and Role:** Function node; aggregates input items into a single JSON object with an array property.  
    - **Configuration:**  
      The function accesses the incoming `items` array, maps each item's `json` property into a new array, and returns a single-item array wrapping this result inside a new JSON object under the key `data_object`.  
      ```js
      return [
        {
          json: {
            data_object: items.map(item => item.json),
          },
        }
      ];
      ```  
    - **Input/Output:**  
      Input: array of JSON objects from "Mock Data" node.  
      Output: one JSON object containing a single key `data_object` with an array of the original objects.  
    - **Key expressions:** `items.map(item => item.json)` extracts the JSON part of each input item.  
    - **Potential failures:**  
      - If input items are empty or not structured as expected, the resulting array might be empty or malformed.  
      - Expression failures if `items` is undefined or not an array (unlikely in this controlled flow).  
    - **Version Requirements:** Standard Function node compatibility.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name               | Node Type       | Functional Role          | Input Node(s) | Output Node(s)              | Sticky Note |
|-------------------------|-----------------|-------------------------|---------------|-----------------------------|-------------|
| Mock Data               | Function        | Creates mock JSON data   | -             | Create an array of objects   |             |
| Create an array of objects | Function      | Aggregates objects into one JSON array | Mock Data   | -                           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node "Mock Data":**  
   - Type: Function  
   - Configure Function Code:  
     ```js
     return [
       { json: { id: 1, name: "Jim" } },
       { json: { id: 2, name: "Stefan" } },
       { json: { id: 3, name: "Hans" } }
     ];
     ```  
   - No credentials needed.  
   - Position on canvas as preferred (e.g., x=800, y=300).

2. **Create Node "Create an array of objects":**  
   - Type: Function  
   - Configure Function Code:  
     ```js
     return [
       {
         json: {
           data_object: items.map(item => item.json),
         },
       }
     ];
     ```  
   - No credentials needed.  
   - Position to the right of "Mock Data" node.

3. **Connect Nodes:**  
   - Connect output of "Mock Data" node to input of "Create an array of objects" node.

4. **Save and Execute:**  
   - Execute to verify the output JSON object contains a single property `data_object` with an array of the three original objects.

---

### 5. General Notes & Resources

| Note Content                                         | Context or Link                                   |
|-----------------------------------------------------|--------------------------------------------------|
| This workflow demonstrates the use of the Function node to manipulate arrays of JSON objects flexibly within n8n. | Useful for data transformation and preparation.  |