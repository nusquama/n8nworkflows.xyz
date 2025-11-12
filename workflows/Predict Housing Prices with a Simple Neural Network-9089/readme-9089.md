Predict Housing Prices with a Simple Neural Network

https://n8nworkflows.xyz/workflows/predict-housing-prices-with-a-simple-neural-network-9089


# Predict Housing Prices with a Simple Neural Network

### 1. Workflow Overview

This workflow implements a simple neural network to predict housing prices based on four input features: square feet, number of rooms, age of the house, and distance to the city. Using basic neural network concepts such as weighted sums, biases, and ReLU activation, the workflow calculates an estimated house price. The workflow is designed for regression use cases where users want to input housing characteristics via a webhook and receive a price prediction.

The workflow’s logic is organized into the following key blocks:

- **1.1 Input Reception:** Receives housing feature data from an external HTTP request via a webhook.
- **1.2 Input Processing and Feature Extraction:** Extracts and prepares individual input features for neural network processing.
- **1.3 Neural Network Layer 1 (Hidden Layer):** Computes weighted sums and applies biases and ReLU activation for two neurons.
- **1.4 Neural Network Output Layer:** Combines outputs from hidden neurons with weights and bias to produce the final price prediction.
- **1.5 Response:** Sends the predicted price back as a JSON response to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives HTTP POST requests containing housing data parameters and triggers the workflow.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook Trigger  
    - Configuration: Listens at path `regression/house/price` with default options, response mode set to "responseNode".  
    - Input: Receives JSON payload with fields: `sqaure_feet` (note the typo), `number_rooms`, `age_in_years`, `distance_to_city_in_miles`.  
    - Output: Passes the input JSON downstream for processing.  
    - Edge Cases: Missing or malformed JSON input; typo in `sqaure_feet` must be handled carefully to avoid errors.  
    - No sub-workflows invoked.

---

#### 1.2 Input Processing and Feature Extraction

- **Overview:**  
Extracts individual input features from the webhook payload and converts or renames them as needed for network input.

- **Nodes Involved:**  
  - Square Feet  
  - Number of Rooms  
  - Age  
  - Distance to City

- **Node Details:**  
  - **Square Feet**  
    - Type: Code Node (JavaScript)  
    - Extracts `sqaure_feet` from input query (typo preserved) and returns as object with key `sqaure_feet`.  
    - Input: Raw webhook JSON  
    - Output: `{ sqaure_feet: <number> }`  
    - Edge Cases: If property missing or non-numeric, may cause downstream issues.

  - **Number of Rooms**  
    - Type: Code Node  
    - Extracts `number_rooms` from webhook query JSON.  
    - Output: `{ number_rooms: <number> }`

  - **Age**  
    - Type: Code Node  
    - Extracts `age_in_years` from webhook query JSON.  
    - Output: `{ age_in_years: <number> }`

  - **Distance to City**  
    - Type: Code Node  
    - Extracts `distance_to_city_in_miles` from webhook query JSON.  
    - Output: `{ distance_to_city_in_miles: <number> }`

---

#### 1.3 Neural Network Layer 1 (Hidden Layer)

- **Overview:**  
Implements two neurons in the hidden layer. Each neuron calculates weighted sums of inputs plus bias, applies ReLU activation, and outputs a value.

- **Nodes Involved:**  
  - Neuron 1 - Input 1, 2, 3, 4  
  - Neuron 1 - Input 1 - Weight, Input 2- Weight, Input 3 - Weight, Input 4 - Weight  
  - Merge 1  
  - Neuron 1 - Weighted Sum  
  - Neuron 1 - Bias  
  - Neuron 1 - ReLU  
  - Neuron 2 - Input 1, 2, 3, 4  
  - Neuron 2 - Input 1 - Weight, Input 2- Weight, Input 3 - Weight, Input 4 - Weight  
  - Merge 2  
  - Neuron 2 - Weighted Sum  
  - Neuron 2 - Bias  
  - Neuron 2 - ReLU  
  - Sticky Notes: #Input Layer, #Neuron 1, #Neuron 2, #Hidden Layer

- **Node Details:**  

  - **Input Nodes for Neurons (Neuron 1 and 2)**  
    - Type: Set Nodes  
    - Assign feature values and fixed weights to each input.  
    - Example: For Neuron 1, Input 1 uses `sqaure_feet` with weight 3.5124223.  
    - Neuron 2 has different weights for the same inputs.  
    - Inputs connected from feature extraction nodes.  
    - Edge Cases: Input value missing or non-numeric can cause calculation issues.

  - **Weight Calculation Nodes (Neuron X - Input Y - Weight)**  
    - Type: Set Nodes  
    - Compute weighted value = value * weight.  
    - Input: Output from corresponding neuron input nodes.  
    - Output: JSON with `weighted_value` number.

  - **Merge Nodes (Merge 1 and Merge 2)**  
    - Type: Merge Nodes  
    - Collect weighted inputs for each neuron to a single stream.

  - **Neuron Weighted Sum (Neuron 1 - Weighted Sum, Neuron 2 - Weighted Sum)**  
    - Type: Code Nodes  
    - Sum all weighted values for the neuron inputs.  
    - Output: `{ weighted_sum: <number> }`

  - **Neuron Bias Nodes (Neuron 1 - Bias, Neuron 2 - Bias)**  
    - Type: Set Nodes  
    - Add fixed bias to weighted sum (e.g., 84.52918 for Neuron 1).  
    - Output key: `logit`.

  - **ReLU Activation (Neuron 1 - ReLU, Neuron 2 - ReLU)**  
    - Type: Set Nodes  
    - Apply ReLU function: output = max(0, logit).  
    - Output key: `output`.

- **Version-specific Notes:**  
  - Uses n8n version supporting advanced Set and Code nodes (typeVersion 3+ and 2+).
  - Code nodes use JavaScript expressions compatible with n8n's sandbox.

- **Potential Failures:**  
  - Missing inputs or weights lead to NaNs.  
  - Incorrect data types (e.g., string instead of number) in inputs cause expression evaluation failures.

---

#### 1.4 Neural Network Output Layer

- **Overview:**  
Takes the outputs from the two hidden neurons, applies output weights and bias, sums them, and produces the final house price estimate.

- **Nodes Involved:**  
  - Output - Neuron 1  
  - Output - Neuron 2  
  - Output - Neuron 1 - Weight  
  - Output - Neuron 2 - Weight  
  - Output Merge  
  - Output - Weighted Sum  
  - Output - Bias  
  - Sticky Notes: #Output Layer, #Output Neuron

- **Node Details:**  

  - **Output - Neuron 1 and 2**  
    - Type: Set Nodes  
    - Receive `output` from respective neuron ReLU node as `value`.  
    - Assign fixed output weights (Neuron 1 weight: 23.246952, Neuron 2 weight: 24.472496).

  - **Output Weight Nodes**  
    - Multiply neuron output value by weight to get weighted value.

  - **Output Merge**  
    - Merges weighted values from both neurons into one stream.

  - **Output - Weighted Sum**  
    - Code node sums weighted values from both outputs.

  - **Output - Bias**  
    - Adds fixed output bias (38.705364) to weighted sum.  
    - Outputs final `output` value representing predicted house price.

- **Potential Failures:**  
  - Missing or malformed neuron outputs can cause errors.  
  - Weight or bias misconfiguration leads to incorrect predictions.

---

#### 1.5 Response

- **Overview:**  
Sends the final predicted house price back to the requester.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**  
  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Configuration: Responds with JSON containing key `"price"` set to the final output value from Output - Bias node.  
    - Input: Receives final output from Output - Bias node.  
    - Edge Cases: If no output is calculated or node fails, response will be empty or error.

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                 | Input Node(s)                                | Output Node(s)                          | Sticky Note                          |
|-------------------------------|-------------------------|--------------------------------|----------------------------------------------|----------------------------------------|------------------------------------|
| Webhook                      | Webhook Trigger         | Receive input data             | -                                            | Distance to City, Age, Number of Rooms, Square Feet |                                    |
| Square Feet                  | Code                    | Extract square feet feature    | Webhook                                       | Neuron 1 - Input 1, Neuron 2 - Input 1 |                                    |
| Number of Rooms              | Code                    | Extract number of rooms feature| Webhook                                       | Neuron 1 - Input 2, Neuron 2 - Input 2 |                                    |
| Age                          | Code                    | Extract house age feature      | Webhook                                       | Neuron 1 - Input 3, Neuron 2 - Input 3 |                                    |
| Distance to City             | Code                    | Extract distance to city feature| Webhook                                       | Neuron 1 - Input 4, Neuron 2- Input 4  |                                    |
| Neuron 1 - Input 1           | Set                     | Assign value and weight for Neuron 1 Input 1 | Square Feet                                  | Neuron 1 - Input 1 - Weight           | #Input Layer                       |
| Neuron 1 - Input 1 - Weight  | Set                     | Calculate weighted value       | Neuron 1 - Input 1                            | Merge 1                               | #Input Layer                       |
| Neuron 1 - Input 2           | Set                     | Assign value and weight for Neuron 1 Input 2 | Number of Rooms                              | Neuron 1 - Input 2- Weight             | #Input Layer                       |
| Neuron 1 - Input 2- Weight   | Set                     | Calculate weighted value       | Neuron 1 - Input 2                           | Merge 1                               | #Input Layer                       |
| Neuron 1 - Input 3           | Set                     | Assign value and weight for Neuron 1 Input 3 | Age                                         | Neuron 1 - Input 3 - Weight            | #Input Layer                       |
| Neuron 1 - Input 3 - Weight  | Set                     | Calculate weighted value       | Neuron 1 - Input 3                           | Merge 1                               | #Input Layer                       |
| Neuron 1 - Input 4           | Set                     | Assign value and weight for Neuron 1 Input 4 | Distance to City                            | Neuron 1 - Input 4 - Weight            | #Input Layer                       |
| Neuron 1 - Input 4 - Weight  | Set                     | Calculate weighted value       | Neuron 1 - Input 4                           | Merge 1                               | #Input Layer                       |
| Merge 1                     | Merge                   | Combine weighted inputs for Neuron 1 | Neuron 1 - Input X - Weight nodes            | Neuron 1 - Weighted Sum                | #Input Layer                       |
| Neuron 1 - Weighted Sum      | Code                    | Sum weighted inputs            | Merge 1                                      | Neuron 1 - Bias                       | #Neuron 1                         |
| Neuron 1 - Bias              | Set                     | Add bias to weighted sum       | Neuron 1 - Weighted Sum                      | Neuron 1 - ReLU                      | #Neuron 1                         |
| Neuron 1 - ReLU              | Set                     | Apply ReLU activation          | Neuron 1 - Bias                             | Output - Neuron 1                    | #Neuron 1, #Hidden Layer          |
| Neuron 2 - Input 1           | Set                     | Assign value and weight for Neuron 2 Input 1 | Square Feet                                 | Neuron 2 - Input 1 - Weight           | #Input Layer                       |
| Neuron 2 - Input 1 - Weight  | Set                     | Calculate weighted value       | Neuron 2 - Input 1                           | Merge 2                               | #Input Layer                       |
| Neuron 2 - Input 2           | Set                     | Assign value and weight for Neuron 2 Input 2 | Number of Rooms                             | Neuron 2 - Input 2- Weight             | #Input Layer                       |
| Neuron 2 - Input 2- Weight   | Set                     | Calculate weighted value       | Neuron 2 - Input 2                           | Merge 2                               | #Input Layer                       |
| Neuron 2 - Input 3           | Set                     | Assign value and weight for Neuron 2 Input 3 | Age                                        | Neuron 2- Input 3 - Weight             | #Input Layer                       |
| Neuron 2- Input 3 - Weight   | Set                     | Calculate weighted value       | Neuron 2 - Input 3                           | Merge 2                               | #Input Layer                       |
| Neuron 2- Input 4            | Set                     | Assign value and weight for Neuron 2 Input 4 | Distance to City                           | Neuron 2 - Input 4 - Weight            | #Input Layer                       |
| Neuron 2 - Input 4 - Weight  | Set                     | Calculate weighted value       | Neuron 2- Input 4                           | Merge 2                               | #Input Layer                       |
| Merge 2                     | Merge                   | Combine weighted inputs for Neuron 2 | Neuron 2 - Input X - Weight nodes            | Neuron 2 - Weighted Sum                | #Input Layer                       |
| Neuron 2 - Weighted Sum      | Code                    | Sum weighted inputs            | Merge 2                                      | Neuron 2 - Bias                       | #Neuron 2                         |
| Neuron 2 - Bias              | Set                     | Add bias to weighted sum       | Neuron 2 - Weighted Sum                      | Neuron 2 - ReLU                      | #Neuron 2                         |
| Neuron 2 - ReLU              | Set                     | Apply ReLU activation          | Neuron 2 - Bias                             | Output - Neuron 2                    | #Neuron 2, #Hidden Layer          |
| Output - Neuron 1            | Set                     | Assign output value and weight | Neuron 1 - ReLU                             | Output - Neuron 1 - Weight            | #Output Layer, #Output Neuron     |
| Output - Neuron 1 - Weight   | Set                     | Calculate weighted output      | Output - Neuron 1                           | Output Merge                         | #Output Layer, #Output Neuron     |
| Output - Neuron 2            | Set                     | Assign output value and weight | Neuron 2 - ReLU                             | Output - Neuron 2 - Weight            | #Output Layer, #Output Neuron     |
| Output - Neuron 2 - Weight   | Set                     | Calculate weighted output      | Output - Neuron 2                           | Output Merge                         | #Output Layer, #Output Neuron     |
| Output Merge                | Merge                   | Combine weighted outputs       | Output - Neuron 1 - Weight, Output - Neuron 2 - Weight | Output - Weighted Sum                | #Output Layer, #Output Neuron     |
| Output - Weighted Sum        | Code                    | Sum weighted outputs           | Output Merge                                 | Output - Bias                        | #Output Layer, #Output Neuron     |
| Output - Bias                | Set                     | Add bias to output sum         | Output - Weighted Sum                        | Respond to Webhook                  | #Output Layer, #Output Neuron     |
| Respond to Webhook           | Respond to Webhook       | Return final price prediction  | Output - Bias                               | -                                    |                                    |
| Sticky Note, Sticky Note1, Sticky Note9, Sticky Note3, Sticky Note4, Sticky Note2 | Sticky Note            | Visual comments and grouping  | -                                            | -                                    | Various as per node clustering    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: `regression/house/price`  
   - Response Mode: `responseNode`  

2. **Create Code Nodes to Extract Inputs** (each connected from Webhook)  
   - **Square Feet**: JS code: `return { sqaure_feet: $input.first().json.query.sqaure_feet };`  
   - **Number of Rooms**: JS code: `return { number_rooms: $input.first().json.query.number_rooms };`  
   - **Age**: JS code: `return { age_in_years: $input.first().json.query.age_in_years };`  
   - **Distance to City**: JS code: `return { distance_to_city_in_miles: $input.first().json.query.distance_to_city_in_miles };`

3. **Create Neuron 1 Input Set Nodes** (connect from feature extraction nodes)  
   - For each input (Square Feet, Number of Rooms, Age, Distance to City):  
     - Set node assigning `value` from respective input field.  
     - Assign corresponding weight (Neuron 1 weights):  
       - Square Feet: 3.5124223  
       - Number of Rooms: 395.781  
       - Age: -23.359684  
       - Distance to City (converted to km): -106.583916  
     - Note: Distance converted from miles to km? The workflow uses `distance_to_city_in_km` in some nodes; if conversion needed, add it before.

4. **Create Neuron 1 Input Weight Calculation Set Nodes**  
   - For each Neuron 1 input node, create a node calculating `weighted_value = value * weight`.

5. **Create Merge Node (Merge 1)**  
   - Number of Inputs: 4  
   - Connect from all Neuron 1 Input Weight nodes.

6. **Create Neuron 1 Weighted Sum Code Node**  
   - JS code sums all `weighted_value`s from Merge 1.

7. **Create Neuron 1 Bias Set Node**  
   - Adds bias 84.52918 to weighted sum as `logit`.

8. **Create Neuron 1 ReLU Set Node**  
   - Implements ReLU: `output = logit > 0 ? logit : 0`.

9. **Repeat Steps 3-8 for Neuron 2**  
   - Use Neuron 2 weights:  
     - Square Feet: 2.7669308  
     - Number of Rooms: 425.43658  
     - Age: -18.90951  
     - Distance to City: -105.73043  
   - Bias: 94.53608  

10. **Create Output Layer Set Nodes**  
    - **Output - Neuron 1**:  
      - `value` = Neuron 1 ReLU output  
      - `weight` = 23.246952  
    - **Output - Neuron 2**:  
      - `value` = Neuron 2 ReLU output  
      - `weight` = 24.472496  

11. **Create Output Weight Calculation Set Nodes**  
    - Multiply `value * weight` for Output - Neuron 1 and Neuron 2.

12. **Create Output Merge Node**  
    - Merge weighted outputs from Output - Neuron 1 - Weight and Output - Neuron 2 - Weight.

13. **Create Output Weighted Sum Code Node**  
    - Sum all `weighted_value`s from Output Merge.

14. **Create Output Bias Set Node**  
    - Add bias 38.705364 to weighted sum, output as final predicted price.

15. **Create Respond to Webhook Node**  
    - Respond with JSON: `{ "price": {{ $json.output }} }`  
    - Connect from Output Bias node.

16. **Connect all nodes as per the logical flow above.**

17. **Add Sticky Notes for clarity** (optional but recommended)  
    - Label blocks as Input Layer, Neuron 1, Neuron 2, Hidden Layer, Output Layer, Output Neuron.

18. **Credentials**  
    - No external credentials required; uses standard n8n nodes only.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                      |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The workflow uses a typo in the input property `sqaure_feet` instead of `square_feet`. Ensure clients send the correct field or adjust the workflow accordingly. | Input data format                                   |
| Visual grouping with sticky notes helps understand neural network layers and neurons.          | Workflow visualization                              |
| This workflow is a demonstration of a simple feedforward neural network implemented purely with n8n's nodes without external ML services. | Educational and proof-of-concept use case           |
| The ReLU activation function is implemented as a simple conditional expression in Set nodes.  | Neural network basics                               |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.