Create AI video ads from product descriptions using Anthropic and deAPI

https://n8nworkflows.xyz/workflows/create-ai-video-ads-from-product-descriptions-using-anthropic-and-deapi-13893


# Create AI video ads from product descriptions using Anthropic and deAPI

# 1. Workflow Overview

This workflow, titled **“AI Video Ad Creator”**, turns a simple product description into a short AI-generated video advertisement. A user submits product information through an n8n form, an AI agent creates optimized prompts for image and video generation, deAPI generates a hero image and then animates it into a video, and Gmail sends the resulting video link to the provided email address.

Its primary use cases include:
- Rapid ad mockup generation for e-commerce products
- Internal creative concept prototyping
- Automated marketing asset creation from structured product inputs
- AI-assisted product showcase generation without manual prompt engineering

The workflow is organized into the following logical blocks:

## 1.1 Input Reception

A form trigger collects the product name, product description, preferred visual style, and destination email address. This is the workflow’s entry point.

## 1.2 AI Prompt Engineering

An AI Agent, powered by Anthropic, analyzes the submitted product details. It uses two deAPI tool nodes to improve prompts:
- one for the still product image
- one for the motion/video prompt

A structured output parser forces the AI response into a predictable JSON shape.

## 1.3 Product Image Generation

The optimized image prompt is sent to deAPI to generate a landscape-format hero product image. This image acts as the visual foundation for the video step.

## 1.4 Video Ad Generation

deAPI uses the generated image as the source for image-to-video generation. The AI-generated video prompt controls animation style, motion, and atmosphere.

## 1.5 Delivery by Email

The final video URL returned by deAPI is inserted into an HTML email and sent via Gmail to the email address entered in the form.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception

### Overview
This block collects the user’s product details through an n8n-hosted web form. The submitted fields become the core variables used by all downstream AI, image, video, and email steps.

### Nodes Involved
- Product Details Form

### Node Details

#### Product Details Form
- **Type and technical role:**  
  `n8n-nodes-base.formTrigger`  
  This is the workflow trigger node. It creates a hosted form and starts a new execution whenever a user submits it.

- **Configuration choices:**  
  The form is titled **“AI Video Ad Creator”** and includes:
  - **Product Name** — required text field
  - **Product Description** — required textarea
  - **Visual Style** — optional text field
  - **Email** — required text field

  It also includes the description:  
  *“Fill in your product details to automatically generate a professional video ad.”*

- **Key expressions or variables used:**  
  This node outputs the following fields referenced later:
  - `$json['Product Name']`
  - `$json['Product Description']`
  - `$json['Visual Style']`
  - `$json['Email']`

- **Input and output connections:**  
  - **Input:** none; it is the entry point
  - **Output:** AI Agent

- **Version-specific requirements:**  
  Uses **typeVersion 2.2** of the Form Trigger node. Ensure your n8n version supports the modern form builder and hosted form execution behavior.

- **Edge cases or potential failure types:**  
  - Form may reject incomplete required fields
  - Email is collected as plain text; no explicit validation pattern is configured
  - Optional visual style may be blank, which can affect AI output richness
  - If n8n is not reachable publicly, form submission may not be usable externally
  - If the instance is not on HTTPS, downstream integrations or form exposure may be problematic; the workflow notes explicitly state HTTPS is required

- **Sub-workflow reference:**  
  None

---

## 2.2 AI Prompt Engineering

### Overview
This block transforms raw product input into production-ready prompts for media generation. The AI Agent uses an Anthropic chat model, calls deAPI prompt booster tools, and returns a structured JSON object containing both the boosted image prompt and the boosted video prompt.

### Nodes Involved
- AI Agent
- Anthropic Chat Model
- Image prompt booster in deAPI
- Video prompt booster in deAPI
- Structured Output Parser

### Node Details

#### AI Agent
- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.agent`  
  This is the orchestration node that receives product details, reasons about the creative task, invokes AI tools, and emits structured output.

- **Configuration choices:**  
  The prompt text is defined explicitly and includes:
  - Product Name
  - Product Description
  - Visual Style

  The system message instructs the model to:
  1. Create a compelling product image prompt and enhance it with the image prompt booster tool
  2. Create a short motion/camera prompt and enhance it with the video prompt booster tool
  3. Return both boosted prompts in the required JSON format

  The node is configured with:
  - **Prompt type:** define
  - **Output parser enabled:** true

- **Key expressions or variables used:**  
  Main prompt:
  - `{{ $json['Product Name'] }}`
  - `{{ $json['Product Description'] }}`
  - `{{ $json['Visual Style'] }}`

  Output is later referenced as:
  - `$json.output.boosted_image_prompt`
  - `$json.output.boosted_video_prompt`

- **Input and output connections:**  
  - **Main input:** Product Details Form
  - **AI language model input:** Anthropic Chat Model
  - **AI tool inputs:** Image prompt booster in deAPI, Video prompt booster in deAPI
  - **AI output parser input:** Structured Output Parser
  - **Main output:** deAPI Generate Image

- **Version-specific requirements:**  
  Uses **typeVersion 1.7** of the LangChain Agent node. Requires a compatible n8n version with AI Agent/tool-call support.

- **Edge cases or potential failure types:**  
  - LLM may fail due to credential, quota, model availability, or API rate limit issues
  - Tool calls may fail if deAPI credentials are invalid
  - If the model does not follow instructions cleanly, output parsing can fail
  - Missing or poor product descriptions may yield weak prompts
  - Blank visual style may still work, but output may be more generic

- **Sub-workflow reference:**  
  None

#### Anthropic Chat Model
- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  Supplies the underlying large language model used by the AI Agent.

- **Configuration choices:**  
  The selected model is **`claude-opus-4-6`**. No additional advanced options are configured.

- **Key expressions or variables used:**  
  No expressions are configured in this node directly.

- **Input and output connections:**  
  - **Output:** connected to AI Agent as `ai_languageModel`

- **Version-specific requirements:**  
  Uses **typeVersion 1.3**. Requires Anthropic credentials configured in n8n and support for this model identifier in the installed node version.

- **Edge cases or potential failure types:**  
  - Invalid Anthropic API key
  - Model access not enabled on the Anthropic account
  - API timeout or rate limiting
  - Future model name deprecation/change

- **Sub-workflow reference:**  
  None

#### Image prompt booster in deAPI
- **Type and technical role:**  
  `n8n-nodes-deapi.deapiTool`  
  Acts as an AI tool callable by the Agent to improve an image-generation prompt.

- **Configuration choices:**  
  - **Resource:** `prompt`
  - Default prompt boosting behavior for image prompts
  - The prompt value is supplied dynamically from the AI tool call

- **Key expressions or variables used:**  
  - `{{ $fromAI('Prompt', '', 'string') }}`  
  This expression lets the AI Agent pass a prompt string into the tool.

- **Input and output connections:**  
  - **AI tool output:** connected to AI Agent

- **Version-specific requirements:**  
  Uses **typeVersion 1** of the deAPI tool node. Requires the deAPI community/integration node installed and compatible.

- **Edge cases or potential failure types:**  
  - Missing or invalid deAPI credentials
  - Tool call payload malformed by the model
  - deAPI prompt service unavailable or rate-limited
  - Unexpected response shape from deAPI

- **Sub-workflow reference:**  
  None

#### Video prompt booster in deAPI
- **Type and technical role:**  
  `n8n-nodes-deapi.deapiTool`  
  AI tool used by the Agent to optimize a motion/video prompt intended for video generation.

- **Configuration choices:**  
  - **Resource:** `prompt`
  - **Operation:** `boostVideo`
  - Prompt is dynamically supplied by the agent

- **Key expressions or variables used:**  
  - `{{ $fromAI('VideoPrompt', '', 'string') }}`

- **Input and output connections:**  
  - **AI tool output:** connected to AI Agent

- **Version-specific requirements:**  
  Uses **typeVersion 1**. Requires deAPI node support for prompt boosting and specifically the `boostVideo` operation.

- **Edge cases or potential failure types:**  
  - Same deAPI credential and rate-limit risks as the image booster
  - The LLM might send a poor or overly descriptive motion prompt despite instructions
  - If deAPI changes the supported operation name, this node may break

- **Sub-workflow reference:**  
  None

#### Structured Output Parser
- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces a specific JSON structure for the AI Agent’s output.

- **Configuration choices:**  
  The schema example expects:
  - `boosted_image_prompt`
  - `boosted_video_prompt`

  This helps downstream nodes reliably access these values without free-form parsing.

- **Key expressions or variables used:**  
  No runtime expressions. The schema example defines the expected object structure.

- **Input and output connections:**  
  - **Output:** connected to AI Agent as `ai_outputParser`

- **Version-specific requirements:**  
  Uses **typeVersion 1.2**. Requires AI parser support in the installed n8n AI nodes.

- **Edge cases or potential failure types:**  
  - If the AI output is not compatible with the schema, parsing may fail
  - If the model returns additional nesting or text wrappers, execution may stop before image generation
  - If downstream assumptions about field names change, this parser and later expressions must be updated together

- **Sub-workflow reference:**  
  None

---

## 2.3 Product Image Generation

### Overview
This block takes the AI-generated image prompt and creates a landscape hero image suitable for use as the first frame of the final ad. It relies on deAPI and is configured to wait for the generation result.

### Nodes Involved
- deAPI Generate Image

### Node Details

#### deAPI Generate Image
- **Type and technical role:**  
  `n8n-nodes-deapi.deapi`  
  Generates a product image from the structured prompt created by the AI block.

- **Configuration choices:**  
  - **Ratio:** `landscape`
  - **Prompt:** uses the AI Agent’s `boosted_image_prompt`
  - **Wait timeout:** `120` seconds
  - **Landscape size option:** `1280x720`

  This produces a hero shot intended to serve as the basis for the video generation step.

- **Key expressions or variables used:**  
  - `{{ $json.output.boosted_image_prompt }}`

- **Input and output connections:**  
  - **Input:** AI Agent
  - **Output:** deAPI Generate Video

- **Version-specific requirements:**  
  Uses **typeVersion 1** of the deAPI node. The selected sizing option `Flux2LandscapeSize` should exist in your installed deAPI node version.

- **Edge cases or potential failure types:**  
  - Invalid deAPI credentials
  - Image generation timeout beyond 120 seconds
  - Prompt moderation or service rejection depending on deAPI policies
  - Output may not match expected composition if the prompt is too vague
  - If the AI output parser fails earlier, this node will receive no usable prompt

- **Sub-workflow reference:**  
  None

---

## 2.4 Video Ad Generation

### Overview
This block turns the generated image into a short product video ad using image-to-video generation. It applies the boosted motion prompt from the AI Agent and waits for the generated video to become available.

### Nodes Involved
- deAPI Generate Video

### Node Details

#### deAPI Generate Video
- **Type and technical role:**  
  `n8n-nodes-deapi.deapi`  
  Generates a video from an existing image source using deAPI’s video resource.

- **Configuration choices:**  
  - **Resource:** `video`
  - **Source:** `image`
  - **Ratio:** `landscape`
  - **Prompt:** uses the AI Agent’s `boosted_video_prompt`
  - **Wait timeout:** `240` seconds

  The node appears intended for image-to-video generation, using the upstream generated image as source material.

- **Key expressions or variables used:**  
  - `{{ $('AI Agent').item.json.output.boosted_video_prompt }}`

- **Input and output connections:**  
  - **Input:** deAPI Generate Image
  - **Output:** Gmail

- **Version-specific requirements:**  
  Uses **typeVersion 1**. Your installed deAPI node version must support:
  - `resource: video`
  - `source: image`
  - polling/wait behavior

- **Edge cases or potential failure types:**  
  - Invalid deAPI credentials
  - Video generation can exceed the 240-second timeout
  - Image-to-video may fail if the previous node’s image output is missing or not in the expected format
  - The node references the AI Agent directly for the prompt instead of the immediate input item, so multi-item runs could produce mismatched data if the workflow is later expanded for batching
  - deAPI response fields may differ by node version, affecting downstream access to `result_url`

- **Sub-workflow reference:**  
  None

---

## 2.5 Delivery by Email

### Overview
This block sends the final generated video URL to the email address provided in the original form submission. It uses a formatted HTML message and includes the product name in both the subject and body.

### Nodes Involved
- Gmail

### Node Details

#### Gmail
- **Type and technical role:**  
  `n8n-nodes-base.gmail`  
  Sends the final notification email containing the video ad download link.

- **Configuration choices:**  
  - **Recipient:** the email provided in the form
  - **Subject:** `Your AI Video Ad — <Product Name>`
  - **HTML body:** includes a heading, product name, direct link to the video, and a small attribution line
  - **Append attribution:** disabled

- **Key expressions or variables used:**  
  - Recipient: `{{ $('Product Details Form').item.json['Email'] }}`
  - Subject: `{{ $('Product Details Form').item.json['Product Name'] }}`
  - Body product name: `{{ $('Product Details Form').item.json['Product Name'] }}`
  - Video URL: `{{ $json.result_url }}`

- **Input and output connections:**  
  - **Input:** deAPI Generate Video
  - **Output:** none

- **Version-specific requirements:**  
  Uses **typeVersion 2.1**. Requires Gmail OAuth2 credentials set up in n8n.

- **Edge cases or potential failure types:**  
  - Gmail OAuth token expired or not properly configured
  - Recipient email malformed
  - `result_url` missing from deAPI video output
  - Gmail sending quota or API restrictions
  - HTML rendering is generally safe, but broken URLs will still result in unusable emails

- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Product Details Form | n8n-nodes-base.formTrigger | Entry-point form collecting product info and recipient email |  | AI Agent | ## 1. Product Details Form<br>Collects product info through a web form:<br><br>- **Product Name** (required)<br>- **Product Description** (required)<br>- **Visual Style** (optional)<br>- **Email** — where to send the result<br><br>Submit the form to kick off the pipeline. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Orchestrates prompt generation and tool usage | Product Details Form | deAPI Generate Image | ## 2. AI-Powered Prompt Creation<br>[Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br><br>The AI Agent analyzes the product and uses two deAPI tools:<br>- **Image Prompt Booster** — optimizes the product image prompt for FLUX<br>- **Video Prompt Booster** — optimizes the motion prompt for LTX-2<br><br>The **Structured Output Parser** ensures consistent JSON output with both boosted prompts. |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Provides the LLM backing the AI Agent |  | AI Agent | ## 2. AI-Powered Prompt Creation<br>[Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br><br>The AI Agent analyzes the product and uses two deAPI tools:<br>- **Image Prompt Booster** — optimizes the product image prompt for FLUX<br>- **Video Prompt Booster** — optimizes the motion prompt for LTX-2<br><br>The **Structured Output Parser** ensures consistent JSON output with both boosted prompts. |
| Image prompt booster in deAPI | n8n-nodes-deapi.deapiTool | AI tool for boosting the image-generation prompt |  | AI Agent | ## 2. AI-Powered Prompt Creation<br>[Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br><br>The AI Agent analyzes the product and uses two deAPI tools:<br>- **Image Prompt Booster** — optimizes the product image prompt for FLUX<br>- **Video Prompt Booster** — optimizes the motion prompt for LTX-2<br><br>The **Structured Output Parser** ensures consistent JSON output with both boosted prompts. |
| Video prompt booster in deAPI | n8n-nodes-deapi.deapiTool | AI tool for boosting the video motion prompt |  | AI Agent | ## 2. AI-Powered Prompt Creation<br>[Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br><br>The AI Agent analyzes the product and uses two deAPI tools:<br>- **Image Prompt Booster** — optimizes the product image prompt for FLUX<br>- **Video Prompt Booster** — optimizes the motion prompt for LTX-2<br><br>The **Structured Output Parser** ensures consistent JSON output with both boosted prompts. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Forces structured JSON output from the AI Agent |  | AI Agent | ## 2. AI-Powered Prompt Creation<br>[Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br><br>The AI Agent analyzes the product and uses two deAPI tools:<br>- **Image Prompt Booster** — optimizes the product image prompt for FLUX<br>- **Video Prompt Booster** — optimizes the motion prompt for LTX-2<br><br>The **Structured Output Parser** ensures consistent JSON output with both boosted prompts. |
| deAPI Generate Image | n8n-nodes-deapi.deapi | Generates the hero product image | AI Agent | deAPI Generate Video | ## 3. Generate Product Image<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>Generates a **1280x720** landscape hero shot of the product.<br><br>This image becomes the **first frame** of the video ad in the next step. |
| deAPI Generate Video | n8n-nodes-deapi.deapi | Converts the generated image into a short video ad | deAPI Generate Image | Gmail | ## 4. Generate Video Ad<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>Uses **image-to-video** generation to animate the product hero shot into a short video ad.<br><br>The AI-crafted video prompt controls the camera movement and animation style. |
| Gmail | n8n-nodes-base.gmail | Sends the final video link by email | deAPI Generate Video |  | ## 5. Email Delivery<br>[Read more about Gmail node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail)<br><br>Sends the video ad link to the email address from the form.<br><br>You can swap this for Slack, Microsoft Teams, or any other notification node. |
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Canvas documentation and usage overview |  |  | ## Try It Out!<br>### Generate a professional video ad from just a product description.<br><br>This workflow takes product details from a simple form, uses AI to create optimized visual prompts, generates a hero product image, animates it into a short video ad, and delivers the result by email.<br><br>### How it works<br>1. **Form Trigger** collects product name, description, style, and recipient email<br>2. **AI Agent** analyzes the product and uses both deAPI **Image Prompt Booster** and **Video Prompt Booster** tools to craft optimized prompts<br>3. **deAPI Generate Image** creates a professional product hero shot<br>4. **deAPI Generate Video** animates the hero image into a short video ad (image-to-video)<br>5. **Gmail** sends the video ad link to the specified email<br><br>### Requirements<br>- [deAPI](https://deapi.ai) account for image generation, video generation, and prompt boosting<br>- Anthropic account for AI Agent<br>- Gmail account for email delivery<br>- n8n instance must be on **HTTPS**<br><br>### Need Help?<br>Join the [n8n Discord](https://discord.gg/n8n) or ask in the [Forum](https://community.n8n.io/)!<br><br>Happy Automating! |
| Sticky Note - Generate Image | n8n-nodes-base.stickyNote | Canvas documentation for image generation stage |  |  | ## 3. Generate Product Image<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>Generates a **1280x720** landscape hero shot of the product.<br><br>This image becomes the **first frame** of the video ad in the next step. |
| Sticky Note - Generate Video | n8n-nodes-base.stickyNote | Canvas documentation for video generation stage |  |  | ## 4. Generate Video Ad<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>Uses **image-to-video** generation to animate the product hero shot into a short video ad.<br><br>The AI-crafted video prompt controls the camera movement and animation style. |
| Sticky Note - Gmail | n8n-nodes-base.stickyNote | Canvas documentation for email delivery stage |  |  | ## 5. Email Delivery<br>[Read more about Gmail node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail)<br><br>Sends the video ad link to the email address from the form.<br><br>You can swap this for Slack, Microsoft Teams, or any other notification node. |
| Sticky Note - Example | n8n-nodes-base.stickyNote | Example input shown on canvas |  |  | ### Example Input<br><br>**Product Name:**<br>Sony WH-1000XM6<br><br>**Product Description:**<br>Premium wireless noise-cancelling headphones with 40-hour battery life, multipoint connection, and adaptive sound control. Ultralight carbon fiber headband with soft-fit leather ear cushions.<br><br>**Visual Style:**<br>modern, minimalist, dark background<br><br>**Email:**<br>your@gmail.com |
| Sticky Note - AI Agent | n8n-nodes-base.stickyNote | Canvas documentation for AI block |  |  | ## 2. AI-Powered Prompt Creation<br>[Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br><br>The AI Agent analyzes the product and uses two deAPI tools:<br>- **Image Prompt Booster** — optimizes the product image prompt for FLUX<br>- **Video Prompt Booster** — optimizes the motion prompt for LTX-2<br><br>The **Structured Output Parser** ensures consistent JSON output with both boosted prompts. |
| Sticky Note - Example1 | n8n-nodes-base.stickyNote | Example output shown on canvas |  |  | ### Example Output<br><br>Link to the video ad: https://res.cloudinary.com/dsmonpqk2/video/upload/v1772714207/bgbzrPpWuZ1vV2u3lPUOQ0QoNQWGTGFxnnlkBSJw_rbrw2j.mp4 |
| Sticky Note - Form1 | n8n-nodes-base.stickyNote | Canvas documentation for form block |  |  | ## 1. Product Details Form<br>Collects product info through a web form:<br><br>- **Product Name** (required)<br>- **Product Description** (required)<br>- **Visual Style** (optional)<br>- **Email** — where to send the result<br><br>Submit the form to kick off the pipeline. |

---

# 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n.

## 4.1 Create the trigger form
1. Create a new workflow named **AI Video Ad Creator**.
2. Add a **Form Trigger** node.
3. Rename it to **Product Details Form**.
4. Set:
   - **Form Title:** `AI Video Ad Creator`
   - **Form Description:** `Fill in your product details to automatically generate a professional video ad.`
5. Add these form fields:
   1. **Product Name**
      - Type: text
      - Required: yes
      - Placeholder: `e.g. AirPods Pro 3`
   2. **Product Description**
      - Type: textarea
      - Required: yes
      - Placeholder: `Briefly describe your product and its key features`
   3. **Visual Style**
      - Type: text
      - Required: no
      - Placeholder: `e.g. modern, minimalist, luxury, bold`
   4. **Email**
      - Type: text
      - Required: yes
      - Placeholder: `you@example.com`

## 4.2 Add the AI Agent
6. Add an **AI Agent** node.
7. Rename it to **AI Agent**.
8. Connect **Product Details Form → AI Agent**.
9. Configure the AI Agent prompt text as:

   ```text
   Create a video ad for the following product:

   Product Name: {{ $json['Product Name'] }}
   Product Description: {{ $json['Product Description'] }}
   Visual Style: {{ $json['Visual Style'] }}
   ```

10. Set the prompt mode to a defined/manual prompt.
11. Enable structured output parsing.
12. In the **System Message**, use:

   ```text
   You are a creative director specializing in product video ads.

   Your tasks:
   1. Create a compelling product image prompt that shows the product in a professional, ad-worthy setting. Use the imagePromptBooster tool to enhance it.
   2. Write a short video motion prompt (1-2 sentences) describing how the camera should move and how the product should be animated. Think cinematic — smooth orbits, slow zooms, dramatic lighting shifts. Do NOT describe the product itself in the video prompt, only the motion and atmosphere. Use the videoPromptBooster tool to enhance it.

   Return both boosted prompts in the required JSON format.
   ```

## 4.3 Add the Anthropic model
13. Add an **Anthropic Chat Model** node.
14. Rename it to **Anthropic Chat Model**.
15. Choose model **claude-opus-4-6**.
16. Create or attach **Anthropic API credentials**.
17. Connect **Anthropic Chat Model → AI Agent** using the **AI Language Model** connection.

## 4.4 Add the structured output parser
18. Add a **Structured Output Parser** node.
19. Rename it to **Structured Output Parser**.
20. Configure the schema example so the AI is expected to return:
   - `boosted_image_prompt`
   - `boosted_video_prompt`

   Example schema content:

   ```json
   {
     "boosted_image_prompt": "professional product photograph of sleek wireless earbuds on a dark reflective surface...",
     "boosted_video_prompt": "slow cinematic camera orbit around the product with soft volumetric lighting gradually intensifying"
   }
   ```

21. Connect **Structured Output Parser → AI Agent** using the **AI Output Parser** connection.

## 4.5 Add the deAPI prompt booster tools
22. Add a **deAPI Tool** node.
23. Rename it to **Image prompt booster in deAPI**.
24. Configure:
   - **Resource:** `prompt`
   - **Prompt:** `{{ $fromAI('Prompt', '', 'string') }}`
25. Create or attach **deAPI credentials**.
26. Connect **Image prompt booster in deAPI → AI Agent** as an **AI Tool** connection.

27. Add another **deAPI Tool** node.
28. Rename it to **Video prompt booster in deAPI**.
29. Configure:
   - **Resource:** `prompt`
   - **Operation:** `boostVideo`
   - **Prompt:** `{{ $fromAI('VideoPrompt', '', 'string') }}`
30. Reuse the same **deAPI credentials**.
31. Connect **Video prompt booster in deAPI → AI Agent** as an **AI Tool** connection.

## 4.6 Add image generation
32. Add a **deAPI** node.
33. Rename it to **deAPI Generate Image**.
34. Connect **AI Agent → deAPI Generate Image**.
35. Configure:
   - **Prompt:** `{{ $json.output.boosted_image_prompt }}`
   - **Ratio:** `landscape`
   - In options:
     - **Wait Timeout:** `120`
     - **Landscape Size / Flux2LandscapeSize:** `1280x720`
36. Reuse the configured **deAPI credentials**.

## 4.7 Add video generation
37. Add another **deAPI** node.
38. Rename it to **deAPI Generate Video**.
39. Connect **deAPI Generate Image → deAPI Generate Video**.
40. Configure:
   - **Resource:** `video`
   - **Source:** `image`
   - **Ratio:** `landscape`
   - **Prompt:** `{{ $('AI Agent').item.json.output.boosted_video_prompt }}`
   - In options:
     - **Wait Timeout:** `240`
41. Reuse the same **deAPI credentials**.

### Important implementation note
42. Confirm that your deAPI video node automatically uses the incoming image from the previous node as the source image. If your installed deAPI node version requires an explicit image URL or file field mapping, configure that mapping based on the image output field returned by **deAPI Generate Image**. This is the most likely area where version-specific adjustment may be required.

## 4.8 Add email delivery
43. Add a **Gmail** node.
44. Rename it to **Gmail**.
45. Connect **deAPI Generate Video → Gmail**.
46. Create or attach **Gmail OAuth2 credentials**.
47. Configure:
   - **To:** `{{ $('Product Details Form').item.json['Email'] }}`
   - **Subject:** `Your AI Video Ad — {{ $('Product Details Form').item.json['Product Name'] }}`
   - **Message type:** HTML
   - **Body:**
     ```html
     <h2>Your video ad is ready!</h2>
     <p>Here is the AI-generated video ad for <strong>{{ $('Product Details Form').item.json['Product Name'] }}</strong>.</p>
     <p><a href="{{ $json.result_url }}">Download your video ad</a></p>
     <p style="color:#888;font-size:12px;">Generated with deAPI and n8n</p>
     ```
   - Disable **Append Attribution**

## 4.9 Credential requirements
48. Configure these credentials before activation:
   - **Anthropic account**
   - **deAPI account**
   - **Gmail OAuth2 account**
49. Ensure your n8n instance is available over **HTTPS**, especially for form usage and reliable OAuth/web integrations.

## 4.10 Test the full flow
50. Open the form URL generated by **Product Details Form**.
51. Submit sample data such as:
   - Product Name: `Sony WH-1000XM6`
   - Product Description: `Premium wireless noise-cancelling headphones with 40-hour battery life, multipoint connection, and adaptive sound control. Ultralight carbon fiber headband with soft-fit leather ear cushions.`
   - Visual Style: `modern, minimalist, dark background`
   - Email: your destination email
52. Verify:
   - The AI Agent returns both boosted prompts in structured JSON
   - The image node returns a generated hero image
   - The video node returns a `result_url`
   - Gmail sends the email correctly

## 4.11 Recommended hardening improvements
53. Add email validation before Gmail if you need stricter input control.
54. Add error branches or an Error Trigger workflow for:
   - AI parsing failures
   - deAPI timeout failures
   - Gmail send failures
55. If you plan to process multiple submissions in parallel or in batches, avoid cross-node item lookups like `$('AI Agent').item...` unless you are certain item pairing remains stable.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does **not** require a Call Workflow / Execute Workflow node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Try It Out! Generate a professional video ad from just a product description. The workflow takes product details from a simple form, uses AI to create optimized visual prompts, generates a hero product image, animates it into a short video ad, and delivers the result by email. | Canvas overview |
| deAPI account is required for image generation, video generation, and prompt boosting. | https://deapi.ai |
| AI Agent documentation | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent |
| deAPI documentation | https://docs.deapi.ai |
| Gmail node documentation | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail |
| Need help with n8n? Join the Discord. | https://discord.gg/n8n |
| Need help with n8n? Ask in the Forum. | https://community.n8n.io/ |
| Example input shown in the workflow: Sony WH-1000XM6, premium wireless noise-cancelling headphones, visual style “modern, minimalist, dark background”, and an email recipient. | Canvas example |
| Example output shown in the workflow: a hosted MP4 link returned by deAPI/cloud storage. | Canvas example output |