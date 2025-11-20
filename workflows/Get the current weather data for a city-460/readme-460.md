Get the current weather data for a city

https://n8nworkflows.xyz/workflows/get-the-current-weather-data-for-a-city-460


# Get the current weather data for a city

### 1. Workflow Overview

This workflow retrieves the current weather data for a specified city using the OpenWeatherMap API. It is designed for quick, manual execution to obtain real-time weather information. The workflow consists of two logical blocks:

- **1.1 Input Reception:** Triggering the workflow manually.
- **1.2 Weather Data Retrieval:** Querying the OpenWeatherMap API to fetch current weather details for a given city.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually when the user clicks the execute button in n8n. It serves as the entry point to trigger the weather data retrieval process.

- **Nodes Involved:**  
  - *On clicking 'execute'*

- **Node Details:**  

  - **Node Name:** On clicking 'execute'  
  - **Type and Technical Role:** Manual Trigger — starts the workflow on user command.  
  - **Configuration Choices:** No parameters configured; default manual trigger settings.  
  - **Key Expressions or Variables:** None.  
  - **Input and Output Connections:** No input nodes; outputs to the OpenWeatherMap node.  
  - **Version-Specific Requirements:** Compatible with n8n version 0.120.0 and later.  
  - **Edge Cases or Potential Failures:** None specific; failure would occur if the workflow is not manually started.  
  - **Sub-workflow Reference:** None.

#### 1.2 Weather Data Retrieval

- **Overview:**  
  This block calls the OpenWeatherMap API to obtain current weather data for a predefined city ("berlin,de"). The node processes the API response and outputs the weather information to the workflow.

- **Nodes Involved:**  
  - *OpenWeatherMap*

- **Node Details:**  

  - **Node Name:** OpenWeatherMap  
  - **Type and Technical Role:** OpenWeatherMap node — interacts with the OpenWeatherMap API to fetch weather data.  
  - **Configuration Choices:**  
    - City Name set to "berlin,de" (Berlin, Germany).  
    - Uses OpenWeatherMap API credentials (API key expected but not included).  
  - **Key Expressions or Variables:** City name is statically set; no dynamic variables or expressions used.  
  - **Input and Output Connections:** Receives input from the Manual Trigger node; no further output nodes connected.  
  - **Version-Specific Requirements:** Requires valid OpenWeatherMap API credentials configured in n8n credentials manager.  
  - **Edge Cases or Potential Failures:**  
    - API authentication errors if credentials are missing or invalid.  
    - Network timeouts or connectivity issues.  
    - Invalid city name causing API to return errors or empty data.  
    - Rate limiting by OpenWeatherMap API.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role               | Input Node(s)          | Output Node(s)        | Sticky Note |
|---------------------|-------------------------|------------------------------|-----------------------|-----------------------|-------------|
| On clicking 'execute'| Manual Trigger          | Initiates workflow manually  | —                     | OpenWeatherMap        |             |
| OpenWeatherMap      | OpenWeatherMap API Node | Fetches current weather data | On clicking 'execute' | —                     |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Get the current weather data for a city."

2. **Add a Manual Trigger node:**  
   - Drag "Manual Trigger" node onto the canvas.  
   - Leave default settings (no parameters needed).  
   - Rename it to "On clicking 'execute'."

3. **Add an OpenWeatherMap node:**  
   - Drag "OpenWeatherMap" node onto the canvas.  
   - Configure parameters:  
     - Set **City Name** to `berlin,de`.  
   - Configure credentials:  
     - Create or select existing OpenWeatherMap API credentials with a valid API key.  
   - Rename this node to "OpenWeatherMap."

4. **Connect nodes:**  
   - Connect the output of "On clicking 'execute'" node to the input of "OpenWeatherMap" node.

5. **Save the workflow.**

6. **Execute manually:**  
   - Click the "Execute Workflow" button to trigger the Manual Trigger node, which will run the OpenWeatherMap node and retrieve current weather data for Berlin.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                            |
|-----------------------------------------------------------------------------------------------|-------------------------------------------|
| Ensure you have a valid OpenWeatherMap API key before running this workflow.                   | https://openweathermap.org/api            |
| The city name must follow the "city,country code" format for accurate API queries.            | OpenWeatherMap API documentation          |
| For real-time applications, consider adding error handling nodes (e.g., IF or Error Trigger). | n8n documentation on error handling       |

---

This document fully describes the workflow, enabling users and automation agents to understand, reproduce, and maintain it effectively.