Perform unit-aware calculations and conversions with CalcsLive

https://n8nworkflows.xyz/workflows/perform-unit-aware-calculations-and-conversions-with-calcslive-13502


# Perform unit-aware calculations and conversions with CalcsLive

# Unit-aware Calculations and Conversions with CalcsLive

This document provides a technical analysis and reproduction guide for the **CalcsLive Demo Workflow**. This workflow demonstrates the power of unit-aware calculations, allowing for automatic conversions, composable logic, and flexible input/output unit management within n8n.

---

### 1. Workflow Overview

The primary purpose of this workflow is to demonstrate how the CalcsLive community node handles physical quantities (values + units) without requiring manual conversion math. It processes three distinct but related calculation paths:

*   **1.1 Linear Speed Calculation:** Computes speed from distance and time.
*   **1.2 Geometric Volume Calculation:** Computes the volume of a cylinder based on dimensions.
*   **1.3 Chained Mass Calculation:** Combines the result of the volume calculation with a density input to derive total mass, demonstrating "composable" calculations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Preparation
**Overview:** This block initializes the static data used for the demonstrations, including measurements like distance, diameter, and density, each paired with a unit string.
*   **Nodes Involved:** `When clicking 'Execute workflow'`, `Set Inputs for Speed`, `Set Inputs for Cylinder`, `Set Density Input`.
*   **Node Details:**
    *   **Manual Trigger:** Entry point for the workflow.
    *   **Set (Speed):** Defines `distance` (360), `unit` (km), `time` (2), and `timeUnit` (h).
    *   **Set (Cylinder):** Defines `Diameter` (2), `DiaUnit` (cm), `Height` (10), and `HeightUnit` (cm).
    *   **Set (Density):** Defines `Density` (1500) and `DensityUnit` (kg/m³).
*   **Edge Cases:** Missing unit strings or invalid unit symbols (e.g., "kms" instead of "km") will cause the subsequent CalcsLive nodes to fail.

#### 2.2 Speed Calculation
**Overview:** Calculates velocity by calling a specific CalcsLive article.
*   **Nodes Involved:** `Speed Calc (d,t) → v`.
*   **Node Details:**
    *   **Type:** `calcslive.calcsLive`
    *   **ArticleID:** `3M6P9TF5P-3XA`
    *   **Configuration:** Maps `d` to distance and `t` to time.
    *   **Output Choice:** Configured to return the result `v` specifically in `km/h`.
    *   **Failure Types:** API authentication errors or invalid `ArticleID`.

#### 2.3 Volume & Mass (Chained Logic)
**Overview:** A two-step process where a volume is calculated, merged with density data, and then used to calculate mass.
*   **Nodes Involved:** `Cylinder Volume (D,h) → V`, `Merge Volume + Density`, `Mass Calc (V,ρ) → m`.
*   **Node Details:**
    *   **Cylinder Volume Node:** Uses Article `3M6P9TF5P-3XA`. Takes diameter and height. Returns volume `V` in Liters (`L`).
    *   **Merge Node:** Combines the output of the Volume node with the static Density data defined in the input block.
    *   **Mass Calc Node:** Uses Article `3M6PBGU7S-3CA`. It maps the volume `V` (from the previous node's output) and density `rho` to calculate mass `m`.
    *   **Output Choice:** Explicitly requests the final mass in grams (`g`).
*   **Dependencies:** The Mass calculation cannot execute until both the Volume calculation and the Density Set node have completed.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking 'Execute workflow' | Manual Trigger | Entry Point | None | Set Inputs for Speed, Set Inputs for Cylinder, Set Density Input | |
| Set Inputs for Speed | Set | Data Initialization | Manual Trigger | Speed Calc (d,t) → v | |
| Set Inputs for Cylinder | Set | Data Initialization | Manual Trigger | Cylinder Volume (D,h) → V | |
| Set Density Input | Set | Data Initialization | Manual Trigger | Merge Volume + Density | |
| Speed Calc (d,t) → v | CalcsLive | Calculation | Set Inputs for Speed | None | Speed Calculation. ArticleID: 3M6P9TF5P-3XA. Calc link: https://calcslive.com/editor/3M6P9TF5P-3XA |
| Cylinder Volume (D,h) → V | CalcsLive | Calculation | Set Inputs for Cylinder | Merge Volume + Density | Cylinder Volume. ArticleID: 3M6P9TF5P-3XA. Calc link: https://calcslive.com/editor/3M6P9TF5P-3XA |
| Merge Volume + Density | Merge | Data Merging | Cylinder Volume, Set Density Input | Mass Calc (V,ρ) → m | Combines data from Cylinder Volume (V) and Density Input (ρ). |
| Mass Calc (V,ρ) → m | CalcsLive | Chained Calc | Merge Volume + Density | None | Chained Calculation. ArticleID: 3M6PBGU7S-3CA. Calc link: https://calcslive.com/editor/3M6PBGU7S-3CA |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    *   Install the community node `@calcslive/n8n-nodes-calcslive` via Settings > Community Nodes.
    *   Create a free account at [calcslive.com](https://www.calcslive.com) and generate an API Key.
    *   Configure the "CalcsLive API" credentials in n8n.

2.  **Define Inputs:**
    *   Add a **Manual Trigger** node.
    *   Add three **Set** nodes connected to the trigger:
        *   `Set Inputs for Speed`: Numbers `distance=360`, `time=2`; Strings `unit=km`, `timeUnit=h`.
        *   `Set Inputs for Cylinder`: Numbers `Diameter=2`, `Height=10`; Strings `DiaUnit=cm`, `HeightUnit=cm`.
        *   `Set Density Input`: Number `Density=1500`, String `DensityUnit=kg/m³`.

3.  **Speed Logic:**
    *   Create a **CalcsLive** node. Set Article ID to `3M6P9TF5P-3XA`.
    *   Under Inputs, map symbol `d` to `{{ $json.distance }}` with unit `{{ $json.unit }}`. Map symbol `t` to `{{ $json.time }}` with unit `{{ $json.timeUnit }}`.
    *   Under Outputs, set symbol `v` with unit `km/h`.

4.  **Volume & Mass Logic:**
    *   Create a **CalcsLive** node for Volume. Article ID: `3M6P9TF5P-3XA`. Map `D` and `h` using the Cylinder Set node variables. Set Output `V` to unit `L`.
    *   Create a **Merge** node. Set Mode to "Combine" and Method to "Combine All". Connect the Volume node to Input 1 and the Density Set node to Input 2.
    *   Create a final **CalcsLive** node for Mass. Article ID: `3M6PBGU7S-3CA`.
    *   Map symbol `V` to `{{ $json.data.calculation.outputs.V.value }}` with unit `{{ $json.data.calculation.outputs.V.unit }}`.
    *   Map symbol `rho` to the density variables from the merged data.
    *   Set Output `m` to unit `g`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Intro Video | [YouTube Demo](https://www.youtube.com/watch?v=xC4iFwNkIQs) |
| Units Reference | [CalcsLive Units Help](https://calcslive.com/help/units-reference) |
| Speed/Volume Calc Source | [Article 3M6P9TF5P-3XA](https://calcslive.com/editor/3M6P9TF5P-3XA) |
| Mass Calc Source | [Article 3M6PBGU7S-3CA](https://calcslive.com/editor/3M6PBGU7S-3CA) |