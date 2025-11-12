Generate Landing Page Layouts from Competitor Analysis with GPT-4

https://n8nworkflows.xyz/workflows/generate-landing-page-layouts-from-competitor-analysis-with-gpt-4-4717


# Generate Landing Page Layouts from Competitor Analysis with GPT-4

---

### 1. Workflow Overview

This workflow automates the generation of landing page layouts by analyzing competitor websites and synthesizing insights with AI. It is aimed at SEO specialists, web designers, and digital marketers who want to quickly draft effective, competitive landing page structures tailored to their unique services and target audience.

The workflow is logically divided into the following blocks:

- **1.1 Input Preparation:** Setting up user-specific data such as services offered, target audience, and competitor URLs.
- **1.2 Competitor Analysis:** Splitting the list of competitor URLs and analyzing each competitor’s landing page to extract main sections and content summaries.
- **1.3 Data Aggregation:** Aggregating the analyzed results from multiple competitor sites into a consolidated input.
- **1.4 Layout Generation:** Using GPT-4 powered AI to generate a recommended landing page outline tailored to the user’s services, target audience, and competitor research.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Preparation

- **Overview:**  
  This block collects and sets the initial input data: a description of the user’s services, target audience, and a list of competitor website URLs to analyze.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set input data

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution during testing  
    - Configuration: No parameters; triggers the workflow manually  
    - Connections: Outputs to "Set input data"  
    - Edge Cases: None (manual trigger)  

  - **Set input data**  
    - Type: Set  
    - Role: Initializes key input variables for the workflow  
    - Configuration:  
      - `my_services`: Text describing the user’s services ("We provide an all-in-one data solution for AI development...")  
      - `my_target_audience ` (note trailing space): Target audience string ("AI developers")  
      - `competitor_url`: Array of competitor URLs to analyze  
    - Key Expressions: Static values assigned directly  
    - Connections: Outputs to "Split competitor url"  
    - Edge Cases:  
      - Trailing space in variable name `my_target_audience ` may cause confusion or require consistent referencing  
      - Input data must be accurate and URLs valid to avoid downstream errors  

---

#### 2.2 Competitor Analysis

- **Overview:**  
  This block processes the competitor URLs by splitting the array into individual URLs, analyzing each website’s main sections via AI, and aggregating the results.

- **Nodes Involved:**  
  - Split competitor url  
  - Analyze competitor  
  - Aggregate analyzed result  
  - OpenAI GPT 4.1 (Language Model for Analyze competitor)

- **Node Details:**

  - **Split competitor url**  
    - Type: Split Out  
    - Role: Splits the array of competitor URLs into individual items for sequential processing  
    - Configuration: Splits on field `competitor_url`  
    - Connections: Outputs each URL item to "Analyze competitor"  
    - Edge Cases: Empty or malformed arrays will result in no processing  

  - **Analyze competitor**  
    - Type: LangChain Agent node (AI agent prompt)  
    - Role: Analyzes a single competitor website URL to outline its main sections and content  
    - Configuration:  
      - Prompt defines the AI persona as a web content analyst  
      - Instructs to visit the URL and summarize up to 10 primary sections with brief descriptions  
      - Output format: Bulleted or numbered list, professional and concise English  
      - Input variable: `{{ $json.competitor_url }}` - the current URL from split node  
    - Connections: Inputs from "Split competitor url"; outputs to "Aggregate analyzed result"  
    - Edge Cases:  
      - If the AI cannot access website content or URL is invalid, analysis may fail or return incomplete data  
      - Network or timeout issues with AI or URL fetching  
      - Requires a valid OpenAI GPT API key (provided via credential)  

  - **Aggregate analyzed result**  
    - Type: Aggregate  
    - Role: Collects all individual competitor analyses into one consolidated dataset  
    - Configuration: Aggregates the field `output` from multiple previous nodes  
    - Connections: Inputs from multiple "Analyze competitor" executions; outputs to "GenerateLayout"  
    - Edge Cases: If no analyses were successful, aggregation output will be empty  

  - **OpenAI GPT 4.1**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Underlying OpenAI GPT-4 variant used by "Analyze competitor" node to generate summaries  
    - Configuration: Model set to `gpt-4.1`  
    - Credentials: OpenAI API key required  
    - Edge Cases: API quota limits, connectivity or authentication failures  

---

#### 2.3 Layout Generation

- **Overview:**  
  Using the aggregated competitor research along with user-specific service descriptions and target audience, this block instructs GPT-4 to generate a tailored landing page layout outline.

- **Nodes Involved:**  
  - GenerateLayout (LangChain Agent)  
  - OpenAI GPT 4. (Language Model for GenerateLayout)

- **Node Details:**

  - **GenerateLayout**  
    - Type: LangChain Agent node (AI agent prompt)  
    - Role: Synthesizes competitor outlines, user services, and target audience into a recommended landing page structure  
    - Configuration:  
      - Prompt includes:  
        - Research outlines from competitor sites (`{{ $json.output }}`)  
        - List of user services (`{{ $('Set input data').item.json.my_services }}`)  
        - Target audience description (`{{ $('Set input data').item.json['my_target_audience '] }}`)  
      - Defines AI persona as a senior website content strategist  
      - Instructions to produce a bulleted list of main page sections with brief descriptions  
      - Constraints: No subsections unless asked, professional English, relevant to business or professional websites  
    - Connections: Inputs from "Aggregate analyzed result" and "OpenAI GPT 4." (model node)  
    - Edge Cases:  
      - Dependencies on accurate input data and aggregated competitor analysis  
      - Potential for AI output variability or failure if inputs are malformed  
      - Requires valid OpenAI API credentials  

  - **OpenAI GPT 4.**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Provides the GPT-4.1 language model response for "GenerateLayout" node  
    - Configuration: Model set to `gpt-4.1`  
    - Credentials: OpenAI API key configured  
    - Edge Cases: Same as for the other OpenAI node, including API limits and authentication issues  

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                                      | Input Node(s)          | Output Node(s)      | Sticky Note                                                                                                                                                |
|-------------------------|---------------------------------|-----------------------------------------------------|------------------------|---------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                 | Entry point to manually start the workflow          | —                      | Set input data       |                                                                                                                                                            |
| Set input data           | Set                             | Initialize user services, target audience, competitor URLs | When clicking ‘Test workflow’ | Split competitor url | Sticky Note1: Setup Initial Data - Prepare your unique services and target audience profile for customization later. Gather competitor URLs to analyze.     |
| Split competitor url     | Split Out                      | Split array of competitor URLs into individual items | Set input data          | Analyze competitor   |                                                                                                                                                            |
| Analyze competitor       | LangChain Agent (AI prompt)     | Analyze competitor website main sections             | Split competitor url    | Aggregate analyzed result | Sticky Note2: Analyzed Competitor Site Layout - Fetches and analyzes your chosen competitor’s landing page.                                                |
| Aggregate analyzed result| Aggregate                      | Aggregate multiple competitor analysis outputs       | Analyze competitor      | GenerateLayout       |                                                                                                                                                            |
| GenerateLayout           | LangChain Agent (AI prompt)     | Generate landing page layout based on competitor research and user data | Aggregate analyzed result, OpenAI GPT 4. | —                   | Sticky Note3: Create Layouts Tailored to Your Services and Audience - Generate tailored layouts useful for wireframing or initial design.                 |
| OpenAI GPT 4.1           | LangChain LM Chat OpenAI        | GPT-4 model used by Analyze competitor node          | —                      | Analyze competitor   |                                                                                                                                                            |
| OpenAI GPT 4.            | LangChain LM Chat OpenAI        | GPT-4 model used by GenerateLayout node               | —                      | GenerateLayout       |                                                                                                                                                            |
| Sticky Note              | Sticky Note                    | Informational content about workflow purpose and use | —                      | —                   | Sticky Note: Overview explaining who the workflow is for, problems solved, and setup instructions.                                                         |
| Sticky Note1             | Sticky Note                    | Setup instructions                                   | —                      | —                   | See "Set input data" row                                                                                                                                    |
| Sticky Note2             | Sticky Note                    | Describes competitor site analysis                    | —                      | —                   | See "Analyze competitor" row                                                                                                                                |
| Sticky Note3             | Sticky Note                    | Describes layout generation and usage                 | —                      | —                   | See "GenerateLayout" row                                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: When clicking ‘Test workflow’  
   - Purpose: Start the workflow manually during testing  

2. **Create a Set node**  
   - Name: Set input data  
   - Purpose: Initialize input variables  
   - Parameters:  
     - `my_services` (string): "We provide an all-in-one data solution for AI development. It offers a SaaS platform for data collection, creation, and annotation on a monthly subscription model."  
     - `my_target_audience ` (string, note trailing space): "AI developers"  
     - `competitor_url` (array): ["https://competitor1.com/", "https://competitor2.com/", "https://competitor3.com/", "https://competitor4.com/"]  
   - Connect output from "When clicking ‘Test workflow’" to this node  

3. **Create a Split Out node**  
   - Name: Split competitor url  
   - Purpose: Split competitor URL array into individual URLs  
   - Parameter: Field to split out set to `competitor_url`  
   - Connect output from "Set input data" to this node  

4. **Create a LangChain Agent node (Analyze competitor)**  
   - Name: Analyze competitor  
   - Purpose: Analyze each competitor URL and summarize main site sections  
   - Prompt: Use the provided detailed prompt instructing the AI to visit the URL and list up to 10 main sections with brief descriptions, in professional English, formatted as a list  
   - Input expression: Use `{{ $json.competitor_url }}` to reference the current URL from the split node  
   - Connect output from "Split competitor url" to this node  

5. **Create an Aggregate node**  
   - Name: Aggregate analyzed result  
   - Purpose: Combine all competitor analyses into one dataset  
   - Configuration: Aggregate on the field `output`  
   - Connect output from "Analyze competitor" to this node  

6. **Create another LangChain LM Chat OpenAI node**  
   - Name: OpenAI GPT 4.1  
   - Purpose: Provide GPT-4.1 model for "Analyze competitor"  
   - Model: Set to `gpt-4.1`  
   - Credentials: Configure with your OpenAI API credentials  

7. **Connect OpenAI GPT 4.1 node output to "Analyze competitor" node’s AI language model input**

8. **Create a LangChain Agent node (GenerateLayout)**  
   - Name: GenerateLayout  
   - Purpose: Generate a landing page layout based on aggregated competitor data and user input  
   - Prompt: Use provided prompt that instructs the AI to synthesize competitor outlines, user services, and target audience to produce an outline of main page sections with descriptions in bulleted list format  
   - Input expressions:  
     - `{{ $json.output }}` for aggregated competitor research  
     - `{{ $('Set input data').item.json.my_services }}` for services description  
     - `{{ $('Set input data').item.json['my_target_audience '] }}` for target audience (note trailing space)  
   - Connect output from "Aggregate analyzed result" to this node  

9. **Create another LangChain LM Chat OpenAI node**  
   - Name: OpenAI GPT 4.  
   - Purpose: Provide GPT-4.1 model for "GenerateLayout"  
   - Model: Set to `gpt-4.1`  
   - Credentials: Use your OpenAI API key  

10. **Connect output of OpenAI GPT 4. node to "GenerateLayout" node’s AI language model input**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                                                                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| This workflow automates the design of landing pages by synthesizing competitor analysis and user data with GPT-4.1 AI models.                                                | Workflow description                                                                                              |
| To customize, update the services and target audience in the "Set input data" node and replace competitor URLs as needed.                                                   | Customization instructions                                                                                        |
| Useful for SEO specialists, web designers, and digital marketers looking for fast, competitive landing page drafts.                                                         | Target audience description                                                                                       |
| Additional enhancements could include keyword extraction or detecting design patterns for richer insights.                                                                   | Suggested workflow extensions                                                                                     |
| OpenAI GPT API credentials must be configured correctly to avoid authentication or quota errors.                                                                             | Credential setup requirement                                                                                      |
| Trailing space in variable name `my_target_audience ` requires consistent referencing or renaming for clarity.                                                                | Variable naming caution                                                                                           |
| Sticky notes provide contextual explanations for each stage and usage recommendations, visible in workflow UI for user guidance.                                            | Sticky notes content                                                                                              |
| Workflow tested with OpenAI GPT-4 models (`gpt-4.1`), ensure n8n version supports LangChain nodes and OpenAI integrations as configured.                                      | Version and compatibility note                                                                                     |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---