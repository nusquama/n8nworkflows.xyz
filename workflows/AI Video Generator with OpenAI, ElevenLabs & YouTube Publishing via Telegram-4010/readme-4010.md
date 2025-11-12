AI Video Generator with OpenAI, ElevenLabs & YouTube Publishing via Telegram

https://n8nworkflows.xyz/workflows/ai-video-generator-with-openai--elevenlabs---youtube-publishing-via-telegram-4010


# AI Video Generator with OpenAI, ElevenLabs & YouTube Publishing via Telegram

---

### 1. Workflow Overview

This workflow automates the end-to-end process of generating AI-powered videos from user input received via Telegram, leveraging OpenAI for ideation and script generation, ElevenLabs for audio synthesis, and Creatomate for video rendering. It further manages video uploads to YouTube and video publishing notifications back to Telegram.

**Target Use Cases:**  
- Content creators wanting automated video production from textual prompts  
- Entrepreneurs and developers building AI-based multimedia automation tools  
- Marketing teams automating video generation and distribution workflows  

**Logical Blocks:**

- **1.1 Telegram Input & Validation:** Receives user messages, verifies API keys, and initiates the ideation process.  
- **1.2 AI Ideation & Script Generation:** Uses OpenAI models to generate video ideas and scripts, with conversational memory and structured output parsing.  
- **1.3 Multimedia Asset Creation:** Generates images and audio based on the script using AI services, including chunking scripts for audio synthesis and image prompting.  
- **1.4 Video Assembly & Rendering:** Aggregates multimedia assets, merges audio and video, sends rendering requests to Creatomate, and waits for final video production.  
- **1.5 Approval & Publishing:** Handles Telegram-based approval for ideas and final videos, converts videos to YouTube-compatible formats, uploads to YouTube, and notifies users.  

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input & Validation

- **Overview:**  
  This block listens for messages from Telegram users, checks if the message is valid, and ensures all necessary API keys are set before proceeding.

- **Nodes Involved:**  
  Telegram Trigger, If Message From User, Set API Keys, If All API Keys Set, Telegram API Keys Missing, Missing API Keys (stop node)

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Trigger node listening for incoming Telegram messages.  
    - Configuration: Uses a unique webhook ID connected to a Telegram bot.  
    - Inputs: Incoming Telegram updates.  
    - Outputs: Passes data to 'If Message From User'.  
    - Potential Failures: Telegram webhook misconfiguration, network issues.  

  - **If Message From User**  
    - Type: Conditional node filtering messages from users (not channels or bots).  
    - Key expression: Checks message origin/type.  
    - Inputs: From Telegram Trigger.  
    - Outputs: Routes valid user messages to ideation flow.  

  - **Set API Keys**  
    - Type: Set node for assigning API keys (OpenAI, ElevenLabs, Creatomate, YouTube, etc.) from credentials or environment variables.  
    - Configuration: Must be set manually before workflow runs (noted in sticky).  
    - Inputs: Triggered after Telegram message validation.  
    - Outputs: Passes to condition checking API key presence.  
    - Edge Cases: Missing or invalid keys halt workflow downstream.  

  - **If All API Keys Set**  
    - Type: Conditional node verifying all required keys are present.  
    - Inputs: From Set API Keys.  
    - Outputs: Continues if keys present; otherwise, sends Telegram alert.  

  - **Telegram: API Keys Missing**  
    - Type: Telegram node sending notification to users about missing API keys.  
    - Inputs: From If All API Keys Set (false branch).  
    - Outputs: Ends with 'Missing API Keys' stop node.  

  - **Missing API Keys**  
    - Type: Stop and Error node that terminates workflow on missing keys.  
    - Inputs: From Telegram API Keys Missing node.  

---

#### 2.2 AI Ideation & Script Generation

- **Overview:**  
  This block handles idea generation, script creation, and conversational memory management using OpenAI models and LangChain integration.

- **Nodes Involved:**  
  Discuss Ideas 💡, OpenAI Chat Model, Track Conversation Memory, Structure Model Output, Ideator 🧠, Script, Chunk Script, If Idea Approved, If No Video Idea, Telegram: Conversational Response, Telegram: Approve Idea, Idea Denied

- **Node Details:**

  - **Discuss Ideas 💡**  
    - Type: LangChain Agent node managing conversational AI interaction.  
    - Configuration: Uses OpenAI Chat Model and tracks memory for context.  
    - Inputs: Incoming user messages (validated).  
    - Outputs: Sends results to conditional branches for idea presence.  
    - Edge Cases: Model timeouts, malformed responses.  

  - **OpenAI Chat Model**  
    - Type: Language model node using OpenAI’s conversational API.  
    - Configuration: Set with model parameters (e.g., temperature, max tokens).  
    - Inputs: Connected to Discuss Ideas for generating responses.  

  - **Track Conversation Memory**  
    - Type: Memory buffer window node saving conversation context.  
    - Configuration: Maintains limited window of recent conversation.  

  - **Structure Model Output**  
    - Type: Output parser node structuring AI-generated content into usable fields.  
    - Configuration: Parses JSON or structured text from AI response.  

  - **Ideator 🧠**  
    - Type: OpenAI node generating detailed scripts and ideas from inputs.  
    - Inputs: User prompts and parameters.  
    - Outputs: Produces script text for further processing.  

  - **Script**  
    - Type: Set node storing the generated script.  
    - Outputs: Feeds into the chunking process for audio synthesis.  

  - **Chunk Script**  
    - Type: HTTP Request node, presumably calls an external API/service to split the script text into manageable audio chunks.  
    - Inputs: Script text.  
    - Outputs: Splits to multiple chunks for parallel audio generation.  

  - **If Idea Approved**  
    - Type: Conditional node verifying user approval of generated idea.  
    - Branches: Approved ideas continue to processing start; rejected ideas flow to denial.  

  - **If No Video Idea**  
    - Type: Conditional node checking if AI generated no valid video idea.  
    - Branches: Sends a conversational response or approval prompt via Telegram.  

  - **Telegram: Conversational Response**  
    - Type: Telegram node sending AI conversational output to user.  

  - **Telegram: Approve Idea**  
    - Type: Telegram node sending prompt to approve generated video idea.  

  - **Idea Denied**  
    - Type: Set node preparing data for re-discussion or alternative actions.  

---

#### 2.3 Multimedia Asset Creation

- **Overview:**  
  Generates images and audio assets from the script, handling asynchronous waits and API interactions to create multimedia components.

- **Nodes Involved:**  
  Convert Script to Audio, Upload to Cloudinary, Image Prompter 📷, Aggregate Prompts, Request Images, Generating Images, Get Images, Request Videos, Generating Videos, Get Videos, Aggregate Videos

- **Node Details:**

  - **Convert Script to Audio**  
    - Type: HTTP Request node calling ElevenLabs or similar TTS API.  
    - Inputs: Chunked script parts.  
    - Outputs: Audio files or URLs.  
    - Configuration: API endpoints, voice model settings, authentication.  
    - Edge Cases: API rate limits, audio generation failures.  

  - **Upload to Cloudinary**  
    - Type: HTTP Request node uploading audio or image files to Cloudinary cloud storage.  
    - Inputs: Audio or image file/binary data.  
    - Outputs: URLs for media assets.  
    - Edge Cases: Upload failures, auth errors.  

  - **Image Prompter 📷**  
    - Type: OpenAI node generating descriptive prompts for image creation.  
    - Inputs: Script or idea text.  
    - Outputs: Prompts for image generation API.  

  - **Aggregate Prompts**  
    - Type: Aggregate node collecting generated image prompts into an array.  

  - **Request Images**  
    - Type: HTTP Request node calling external image generation API (e.g., DALL·E, Stable Diffusion).  
    - Inputs: Aggregated prompts.  
    - Outputs: Initiates image generation jobs.  

  - **Generating Images**  
    - Type: Wait node with webhook, pauses workflow until images are ready.  

  - **Get Images**  
    - Type: HTTP Request node retrieving generated images after completion.  
    - Outputs: Image URLs or data.  

  - **Request Videos**  
    - Type: HTTP Request node starting video generation jobs based on images and scripts.  

  - **Generating Videos**  
    - Type: Wait node for asynchronous video generation completion.  

  - **Get Videos**  
    - Type: HTTP Request node fetching generated video assets.  

  - **Aggregate Videos**  
    - Type: Aggregate node grouping multiple video fragments or assets for merging.  

---

#### 2.4 Video Assembly & Rendering

- **Overview:**  
  Combines audio with video, generates rendering JSON payloads, sends jobs to Creatomate, and awaits final video production.

- **Nodes Involved:**  
  Merge Videos and Audio, Generate Render JSON, Set JSON Variable, Send to Creatomate, Generating Final Video, Get Final Video, Merge Video Variables

- **Node Details:**

  - **Merge Videos and Audio**  
    - Type: Merge node combining audio and video streams.  
    - Inputs: Aggregated videos and audio URLs.  
    - Outputs: Combined media ready for rendering.  

  - **Generate Render JSON**  
    - Type: HTTP Request node generating structured JSON to define Creatomate rendering project.  
    - Inputs: Merged media data.  

  - **Set JSON Variable**  
    - Type: Set node preparing the final JSON payload for Creatomate.  

  - **Send to Creatomate**  
    - Type: HTTP Request node sending render JSON to Creatomate API to start video rendering job.  

  - **Generating Final Video**  
    - Type: Wait node with webhook, pauses workflow until Creatomate signals video render completion.  

  - **Get Final Video**  
    - Type: HTTP Request node retrieving the completed video file URL or data.  

  - **Merge Video Variables**  
    - Type: Merge node collecting final video data and metadata for approval step.  

---

#### 2.5 Approval & Publishing

- **Overview:**  
  Manages user approval for the final video, converts video for YouTube upload, uploads the video, and sends notifications via Telegram.

- **Nodes Involved:**  
  Telegram: Approve Final Video, If Final Video Approved, Convert Video to Base64, Decode Base64 to File, Upload to YouTube, Telegram: Video Uploaded, Telegram: Video Declined

- **Node Details:**

  - **Telegram: Approve Final Video**  
    - Type: Telegram node requesting user approval on final video.  

  - **If Final Video Approved**  
    - Type: Conditional node branching on user approval.  
    - Yes: Proceed to conversion and upload.  
    - No: Notify user of decline.  

  - **Convert Video to Base64**  
    - Type: HTTP Request node encoding video file to base64 format suitable for n8n file handling.  

  - **Decode Base64 to File**  
    - Type: Convert to File node decoding base64 string back to file binary for upload.  

  - **Upload to YouTube**  
    - Type: YouTube node uploading the video file to the channel.  
    - Configuration: OAuth2 credentials, title, description, tags.  
    - Edge Cases: Upload failures, quota limits.  

  - **Telegram: Video Uploaded**  
    - Type: Telegram node notifying user of successful upload.  

  - **Telegram: Video Declined**  
    - Type: Telegram node notifying user of video rejection.  

---

### 3. Summary Table

| Node Name                 | Node Type                               | Functional Role                          | Input Node(s)                   | Output Node(s)                      | Sticky Note                   |
|---------------------------|---------------------------------------|----------------------------------------|--------------------------------|------------------------------------|------------------------------|
| Telegram Trigger          | telegramTrigger                       | Receive Telegram user messages          | -                              | If Message From User                |                              |
| If Message From User      | if                                   | Filter only messages from users         | Telegram Trigger               | Discuss Ideas 💡                   |                              |
| Set API Keys             | set                                  | Set API keys before processing          | Telegram: Processing Started   | If All API Keys Set                 | SET BEFORE STARTING           |
| If All API Keys Set       | if                                   | Check presence of all required API keys | Set API Keys                  | Input Variables, Telegram API Keys Missing |                              |
| Telegram: API Keys Missing| telegram                             | Notify user about missing API keys      | If All API Keys Set            | Missing API Keys                   |                              |
| Missing API Keys          | stopAndError                        | Stop workflow due to missing keys       | Telegram: API Keys Missing     | -                                  |                              |
| Discuss Ideas 💡          | langchain.agent                    | Manage AI conversation & ideation       | If Message From User           | If No Video Idea                   |                              |
| OpenAI Chat Model         | langchain.lmChatOpenAi             | OpenAI conversational AI model          | Discuss Ideas 💡                | Discuss Ideas 💡                   |                              |
| Track Conversation Memory | langchain.memoryBufferWindow       | Maintain conversation context            | Discuss Ideas 💡                | Discuss Ideas 💡                   |                              |
| Structure Model Output    | langchain.outputParserStructured    | Parse AI output into structured format  | Discuss Ideas 💡                | Discuss Ideas 💡                   |                              |
| Ideator 🧠                | langchain.openAi                   | Generate video ideas and scripts         | Input Variables                | Script                            |                              |
| Script                   | set                                  | Store generated script                   | Ideator 🧠                    | Convert Script to Audio, Upload to Cloudinary |                              |
| Chunk Script             | httpRequest                         | Split script into audio-friendly chunks | Script                        | Split Out                        |                              |
| Split Out                | splitOut                           | Split chunked script output              | Chunk Script                  | Image Prompter 📷                 |                              |
| Image Prompter 📷          | langchain.openAi                   | Generate prompts for image generation    | Split Out                    | Aggregate Prompts, Request Images |                              |
| Aggregate Prompts         | aggregate                         | Collect image prompts                    | Image Prompter 📷             | Merge Video Variables (via connections) |                              |
| Request Images            | httpRequest                       | Request image generation                 | Aggregate Prompts             | Generating Images                 |                              |
| Generating Images         | wait                              | Wait for image generation completion     | Request Images                | Get Images                      |                              |
| Get Images                | httpRequest                       | Retrieve generated images                 | Generating Images             | Request Videos                  |                              |
| Request Videos            | httpRequest                       | Request video generation                  | Get Images                   | Generating Videos               |                              |
| Generating Videos         | wait                              | Wait for video generation completion      | Request Videos               | Get Videos                     |                              |
| Get Videos                | httpRequest                       | Retrieve generated videos                 | Generating Videos            | Aggregate Videos               |                              |
| Aggregate Videos          | aggregate                         | Collect video fragments for merging      | Get Videos                  | Merge Videos and Audio          |                              |
| Convert Script to Audio   | httpRequest                       | Generate audio from text script           | Script                      | Chunk Script, Upload to Cloudinary |                              |
| Upload to Cloudinary      | httpRequest                       | Upload audio/images to Cloudinary         | Convert Script to Audio, others | Merge Videos and Audio          |                              |
| Merge Videos and Audio    | merge                            | Combine audio and video streams           | Aggregate Videos, Upload to Cloudinary | Generate Render JSON          |                              |
| Generate Render JSON      | httpRequest                      | Generate JSON for Creatomate rendering    | Merge Videos and Audio        | Set JSON Variable               |                              |
| Set JSON Variable         | set                              | Prepare JSON payload                      | Generate Render JSON          | Send to Creatomate             |                              |
| Send to Creatomate        | httpRequest                      | Send video rendering job to Creatomate   | Set JSON Variable             | Generating Final Video          |                              |
| Generating Final Video    | wait                             | Wait for final video rendering            | Send to Creatomate            | Get Final Video                |                              |
| Get Final Video           | httpRequest                      | Retrieve final rendered video             | Generating Final Video        | Merge Video Variables          |                              |
| Merge Video Variables     | merge                            | Aggregate final video variables           | Get Final Video, Aggregate Prompts | Telegram: Approve Final Video |                              |
| Telegram: Approve Final Video | telegram                      | Request user approval on final video      | Merge Video Variables         | If Final Video Approved         |                              |
| If Final Video Approved   | if                               | Branch on user video approval              | Telegram: Approve Final Video | Convert Video to Base64, Telegram: Video Declined |                              |
| Convert Video to Base64   | httpRequest                      | Convert video file to base64 string        | If Final Video Approved       | Decode Base64 to File          |                              |
| Decode Base64 to File     | convertToFile                   | Decode base64 string to file binary        | Convert Video to Base64       | Upload to YouTube              |                              |
| Upload to YouTube         | youTube                         | Upload video to YouTube channel             | Decode Base64 to File        | Telegram: Video Uploaded       |                              |
| Telegram: Video Uploaded  | telegram                        | Notify user of successful upload            | Upload to YouTube             | -                              |                              |
| Telegram: Video Declined  | telegram                        | Notify user of video rejection               | If Final Video Approved (no)  | -                              |                              |
| Telegram: Conversational Response | telegram                 | Send conversational AI responses            | If No Video Idea              | -                              |                              |
| Telegram: Approve Idea    | telegram                        | Ask user to approve generated idea           | If No Video Idea (yes)        | If Idea Approved               |                              |
| If Idea Approved          | if                             | Branch based on user approval of idea        | Telegram: Approve Idea        | Telegram: Processing Started, Idea Denied |                              |
| Telegram: Processing Started | telegram                    | Notify user processing has started            | If Idea Approved             | Set API Keys                  |                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: telegramTrigger  
   - Configure with Telegram bot credentials and webhook setup.  
   - Position: Start node to receive user messages.

2. **Add If node "If Message From User"**  
   - Condition: Check message type to ensure it’s from a user, not a bot/channel.  
   - Connect from Telegram Trigger.

3. **Add Set node "Set API Keys"**  
   - Manually set environment variables or credentials for OpenAI, ElevenLabs, Creatomate, YouTube, Cloudinary, Telegram.  
   - Connect from "Telegram: Processing Started" node (you need this to notify before setting keys).

4. **Add If node "If All API Keys Set"**  
   - Check all required API key variables are present and non-empty.  
   - Connect from "Set API Keys".

5. **Add Telegram node "Telegram: API Keys Missing"**  
   - Sends message to user if API keys are missing.  
   - Connect from "If All API Keys Set" (false branch).

6. **Add Stop node "Missing API Keys"**  
   - Ends workflow in error if API keys are missing.  
   - Connect from "Telegram: API Keys Missing".

7. **Add LangChain Agent node "Discuss Ideas 💡"**  
   - Configure with OpenAI Chat Model, with conversation memory.  
   - Connect from "If Message From User" (true branch).

8. **Add OpenAI Chat Model node**  
   - Set model parameters (temperature, max tokens, etc.).

9. **Add Memory Buffer node "Track Conversation Memory"**  
   - Configure context window size.

10. **Add Output Parser node "Structure Model Output"**  
    - Configure expected structured JSON output.

11. **Add If node "If No Video Idea"**  
    - Branch based on presence of video idea in AI output.

12. **Add Telegram node "Telegram: Conversational Response"**  
    - Sends AI response if no video idea found.

13. **Add Telegram node "Telegram: Approve Idea"**  
    - Sends approval prompt if idea found.

14. **Add If node "If Idea Approved"**  
    - Branches on user approval of idea.

15. **Add Set node "Idea Denied"**  
    - Handles denial flow.

16. **Add Set node "Input Variables"**  
    - Prepare variables from approved idea for script generation.

17. **Add OpenAI node "Ideator 🧠"**  
    - Generate detailed script based on input variables.

18. **Add Set node "Script"**  
    - Store generated script output.

19. **Add HTTP Request node "Convert Script to Audio"**  
    - Call ElevenLabs TTS API with script chunks.

20. **Add HTTP Request node "Chunk Script"**  
    - Split script into audio-friendly chunks.

21. **Add SplitOut node "Split Out"**  
    - Split chunk data for parallel audio processing.

22. **Add OpenAI node "Image Prompter 📷"**  
    - Generate image prompts from script chunks.

23. **Add Aggregate node "Aggregate Prompts"**  
    - Collect image prompts.

24. **Add HTTP Request nodes "Request Images", "Request Videos"**  
    - Request image and video generation from external APIs.

25. **Add Wait nodes "Generating Images", "Generating Videos"**  
    - Wait for external job completion.

26. **Add HTTP Request nodes "Get Images", "Get Videos"**  
    - Retrieve generated images and videos.

27. **Add Aggregate node "Aggregate Videos"**  
    - Collect video fragments.

28. **Add HTTP Request node "Upload to Cloudinary"**  
    - Upload audio and images for hosting.

29. **Add Merge node "Merge Videos and Audio"**  
    - Combine audio and video streams.

30. **Add HTTP Request node "Generate Render JSON"**  
    - Generate Creatomate payload.

31. **Add Set node "Set JSON Variable"**  
    - Prepare JSON for Creatomate.

32. **Add HTTP Request node "Send to Creatomate"**  
    - Send render job.

33. **Add Wait node "Generating Final Video"**  
    - Wait for render completion.

34. **Add HTTP Request node "Get Final Video"**  
    - Retrieve final video.

35. **Add Merge node "Merge Video Variables"**  
    - Aggregate final video data.

36. **Add Telegram node "Telegram: Approve Final Video"**  
    - Request user approval.

37. **Add If node "If Final Video Approved"**  
    - Branch based on approval.

38. **Add HTTP Request node "Convert Video to Base64"**  
    - Convert video for n8n handling.

39. **Add Convert node "Decode Base64 to File"**  
    - Decode base64 to file.

40. **Add YouTube node "Upload to YouTube"**  
    - Configure with OAuth2 and upload video.

41. **Add Telegram nodes "Telegram: Video Uploaded", "Telegram: Video Declined"**  
    - Notify user of upload status.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| API keys must be set manually before starting the workflow to avoid execution errors. | Sticky note on "Set API Keys" node. |
| Workflow uses LangChain nodes for managing AI conversation and memory context. | General workflow design. |
| Video rendering is performed by Creatomate API, which requires a valid account and API key. | Video assembly block. |
| ElevenLabs or equivalent TTS API is used for converting script text to audio. | Multimedia asset creation. |
| YouTube node requires OAuth2 credentials with video upload permission. | Publishing block. |
| Telegram nodes use webhooks; proper bot setup and webhook registration are required. | Input reception block. |

---

This comprehensive reference covers all technical aspects, node configurations, logical flow, and integration points necessary for understanding, reproducing, or modifying the "AI Video Generator with OpenAI, ElevenLabs & YouTube Publishing via Telegram" workflow.