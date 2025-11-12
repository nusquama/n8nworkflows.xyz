Generate Multi-Platform Social Media Posts with GPT-4.1 and PostPulse

https://n8nworkflows.xyz/workflows/generate-multi-platform-social-media-posts-with-gpt-4-1-and-postpulse-9195


# Generate Multi-Platform Social Media Posts with GPT-4.1 and PostPulse

### 1. Workflow Overview

This workflow automates the creation and scheduling of multi-platform social media posts using AI content generation and PostPulse social media management. It is designed to take a core post idea, adapt it intelligently for various social media platforms by respecting their unique content restrictions and audience expectations, and then schedule these posts as drafts via PostPulse for review or publishing.

**Target Use Cases:**  
- Social media managers who want to generate platform-tailored posts automatically from a single idea.  
- Marketing teams leveraging AI to produce and schedule posts across multiple platforms (Twitter, LinkedIn, TikTok, YouTube, Telegram, etc.).  
- Users of PostPulse seeking integration for draft scheduling and account management.

**Logical Blocks:**  
- **1.1 Input Reception:** Manual trigger and initial idea setting.  
- **1.2 Idea Adaptation for Platforms:** Code-based transformation of the base idea into platform-specific prompts with character limits and hashtags.  
- **1.3 AI Content Generation:** Using GPT-4.1 to produce tailored posts per platform.  
- **1.4 Post-Processing and Mapping:** Normalizing AI output and associating it with corresponding platforms.  
- **1.5 Account Retrieval:** Fetching connected social media accounts from PostPulse.  
- **1.6 Data Merging:** Combining AI-generated content with the social accounts for each platform.  
- **1.7 Publishing:** Scheduling the posts as drafts in PostPulse.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow manually and sets the initial post idea content.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- idea  
- Sticky Note1

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command.  
  - Config: No parameters; simple manual trigger.  
  - Input: None  
  - Output: Triggers two parallel paths to “Get connected accounts” and “idea” nodes.  
  - Failure Types: None typical; manual action required.

- **idea**  
  - Type: Set node  
  - Role: Holds the core initial social media post idea as a string variable.  
  - Config: Sets `"idea"` variable to a hardcoded string describing a restaurant menu update and discount.  
  - Expressions: Static text, no dynamic data used.  
  - Input: Trigger output from manual trigger.  
  - Output: JSON object with `idea` property.  
  - Failure Types: None expected unless node misconfigured.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Content: "## Idea  \nEnter your post idea here."  
  - Role: Documentation for user to input idea content.

---

#### 1.2 Idea Adaptation for Platforms

**Overview:**  
Transforms the single idea into multiple platform-specific prompts with character limits and hashtags, preparing for AI processing.

**Nodes Involved:**  
- Setting Restrictions and Hashtags  
- Sticky Note2

**Node Details:**  

- **Setting Restrictions and Hashtags**  
  - Type: Code node (JavaScript)  
  - Role: Generates an array of platform-specific prompt objects, each containing the platform name and an adapted text prompt including character limits and hashtag instructions.  
  - Key Logic: Defines character limits per platform (e.g., Twitter 280 chars), crafts prompt asking AI to create posts within these limits and add 3-4 relevant hashtags.  
  - Input: JSON with `idea` string from “idea” node.  
  - Output: Array of JSON objects, each with `platform` and `text` fields.  
  - Edge Cases: If `idea` is missing or empty, output prompts will be incomplete. Custom platform list or limits can be changed here.  
  - Sticky Note2 explains optional adjustments for limits and hashtags.

- **Sticky Note2**  
  - Content: Explains the purpose of the node in adapting ideas for platform limits and hashtags with optional customization.

---

#### 1.3 AI Content Generation

**Overview:**  
Uses GPT-4.1 to generate platform-tailored social media posts based on prompts created in the previous block.

**Nodes Involved:**  
- AI Content Adapter  
- Sticky Note3

**Node Details:**  

- **AI Content Adapter**  
  - Type: OpenAI (LangChain) node  
  - Role: Sends each platform-specific prompt to GPT-4.1 and receives content variations adapted for each platform’s audience and restrictions.  
  - Config:  
    - Model: GPT-4.1  
    - System prompt: Expert social media content creator, writes only in English.  
    - User message: Injected from input `text` field (platform-specific prompt).  
  - Credentials: OpenAI API key configured.  
  - Input: Array of prompts (platform + text) from the previous code node.  
  - Output: Array of AI-generated messages containing post content.  
  - Failure Risks: API authentication errors, rate limits, timeouts, malformed inputs.  
  - Sticky Note3 documents its role as the AI content generation engine.

- **Sticky Note3**  
  - Content: Describes the AI Content Adapter as the node generating platform-specific content variations.

---

#### 1.4 Post-Processing and Mapping

**Overview:**  
Maps AI output messages back to their corresponding platforms, preparing for merging with social media accounts.

**Nodes Involved:**  
- Unification of Platforms and Text  
- Sticky Note4

**Node Details:**  

- **Unification of Platforms and Text**  
  - Type: Code node (JavaScript)  
  - Role: Takes AI-generated posts and annotates each with its platform, preserving order and extracting the text content.  
  - Logic: Uses a predefined platform array to assign platform names to AI outputs; extracts `message.content` from AI node JSON.  
  - Input: AI Content Adapter output items.  
  - Output: Array of objects with `platform` and `text`.  
  - Edge Cases: Handles mismatch in item counts by defaulting platform to “unknown.”  
  - Sticky Note4 explains this node’s role in combining platform info with AI text.

- **Sticky Note4**  
  - Content: Describes the unification of AI-generated text with platform identifiers.

---

#### 1.5 Account Retrieval

**Overview:**  
Fetches connected social media accounts from PostPulse for use in scheduling posts.

**Nodes Involved:**  
- Get connected accounts  
- Sticky Note8

**Node Details:**  

- **Get connected accounts**  
  - Type: PostPulse node (resource: account)  
  - Role: Retrieves the user’s linked social media accounts from PostPulse API.  
  - Credentials: OAuth2 credentials for PostPulse account required.  
  - Input: Triggered in parallel from manual trigger.  
  - Output: List of connected accounts including platform ID and account identifiers.  
  - Failure Risks: OAuth token expiration, API errors, network failures.  
  - Sticky Note8 documents this node’s purpose.

- **Sticky Note8**  
  - Content: Describes retrieval of linked social media accounts.

---

#### 1.6 Data Merging

**Overview:**  
Merges AI-generated post content with corresponding social media accounts to prepare complete publishing data.

**Nodes Involved:**  
- Merge  
- Sticky Note5

**Node Details:**  

- **Merge**  
  - Type: Merge node  
  - Role: Combines two data streams — AI-generated posts and connected accounts — matching items by platform.  
  - Mode: Combine (fieldsToMatchString = “platform”) ensures posts are matched with correct accounts by platform name.  
  - Input:  
    - Main input: AI content with platform info  
    - Secondary input: Connected accounts with platform info  
  - Output: Combined JSON objects containing post text and account details.  
  - Edge Cases: If no matching platform found in accounts or AI data, merge fails to join properly; partial data may remain unmatched.  
  - Sticky Note5 explains the merging purpose.

- **Sticky Note5**  
  - Content: Describes merging platform info from accounts with AI posts for publishing.

---

#### 1.7 Publishing

**Overview:**  
Schedules the merged social media posts as drafts in PostPulse, ready for review or publishing.

**Nodes Involved:**  
- Publish Post  
- Sticky Note6

**Node Details:**  

- **Publish Post**  
  - Type: PostPulse node (resource: post)  
  - Role: Sends the prepared post content to PostPulse API to create drafts for each connected account/platform.  
  - Config:  
    - `isDraft`: true (posts are drafts, not immediately published)  
    - `publications`: JSON array specifying posts content, platform settings, and account IDs.  
    - Content truncation logic: JavaScript snippet truncates posts to platform-specific max lengths (different for Twitter, Blue Sky, etc.) with sentence boundary consideration, or truncates by space and appends ellipsis.  
    - Platform mapping: Maps internal platform codes (e.g., X_TWITTER) to PostPulse platform types (e.g., TWITTER).  
    - Scheduled time: Current UTC time.  
  - Credentials: PostPulse OAuth2 credentials.  
  - Input: Merged array of post text and account info from “Merge” node.  
  - Output: PostPulse API response confirming draft creation.  
  - Failure Risks: API errors, invalid content length, platform mismatches, auth token expiration.  
  - Sticky Note6 documents the publishing function.

- **Sticky Note6**  
  - Content: Describes sending prepared posts to PostPulse as drafts.

---

### 3. Summary Table

| Node Name                   | Node Type                            | Functional Role                                 | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                  |
|-----------------------------|------------------------------------|------------------------------------------------|-------------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                    | Starts workflow manually                        | None                          | Get connected accounts, idea     |                                                                                              |
| idea                        | Set                                | Holds the initial post idea                     | When clicking ‘Execute workflow’| Setting Restrictions and Hashtags | ## Idea  Enter your post idea here.                                                          |
| Setting Restrictions and Hashtags | Code (JavaScript)               | Adapts idea into platform-specific prompts     | idea                          | AI Content Adapter               | ## Setting Restrictions and Hashtags  Here the idea is adapted for each platform with limits and hashtags. Optional changes allowed. |
| AI Content Adapter          | OpenAI (LangChain)                  | Generates AI posts tailored for each platform  | Setting Restrictions and Hashtags | Unification of Platforms and Text | ## AI Content Adapter  This node generates platform-specific content variations.              |
| Unification of Platforms and Text | Code (JavaScript)               | Maps AI output back to platforms                 | AI Content Adapter            | Merge                           | ## Unification of Platforms and Text  Combines AI-generated text with platform info.          |
| Get connected accounts      | PostPulse (account)                 | Retrieves connected social media accounts       | When clicking ‘Execute workflow’| Merge                           | ## PostPulse Get Connected Accounts  Retrieves linked social media accounts for publishing.  |
| Merge                      | Merge                              | Combines AI posts with account info              | Unification of Platforms and Text, Get connected accounts | Publish Post                  | ## Merge  Merges platform info from accounts with AI posts, preparing for publishing.         |
| Publish Post                | PostPulse (post)                   | Schedules posts as drafts in PostPulse          | Merge                         | None                           | ## Publish Post  Sends prepared posts to PostPulse as drafts for scheduling or publishing.    |
| Sticky Note1                | Sticky Note                       | Documentation for idea input                      | None                          | None                           | ## Idea  Enter your post idea here.                                                          |
| Sticky Note2                | Sticky Note                       | Documentation for prompt adaptation              | None                          | None                           | ## Setting Restrictions and Hashtags  Explains platform limits and hashtags.                 |
| Sticky Note3                | Sticky Note                       | Documentation for AI content generation           | None                          | None                           | ## AI Content Adapter  Describes AI content generation node.                                  |
| Sticky Note4                | Sticky Note                       | Documentation for post-processing                  | None                          | None                           | ## Unification of Platforms and Text  Describes mapping AI output to platforms.               |
| Sticky Note5                | Sticky Note                       | Documentation for data merging                      | None                          | None                           | ## Merge  Describes merging AI posts with account data.                                      |
| Sticky Note6                | Sticky Note                       | Documentation for publishing                        | None                          | None                           | ## Publish Post  Describes sending posts as drafts to PostPulse.                             |
| Sticky Note8                | Sticky Note                       | Documentation for account retrieval                 | None                          | None                           | ## PostPulse Get Connected Accounts  Explains account retrieval node.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters needed.

2. **Create Set Node for Idea**  
   - Node Type: Set  
   - Name: `idea`  
   - Add a string field named `idea` with the text:  
     `"Write a post for my restaurant called \"West Coast\", write that oysters and squid have appeared on our menu and many new items from Asian cuisine, in honor of this, a discount on Asian cuisine and seafood dishes minus 40 percent!"`  
   - Connect output of Manual Trigger to this node.

3. **Create Code Node for Platform-Specific Prompt Generation**  
   - Node Type: Code (JavaScript)  
   - Name: `Setting Restrictions and Hashtags`  
   - Paste the following JS code:

     ```javascript
     let idea = items[0].json.idea;
     const platforms = {
       TELEGRAM: 800,
       TIKTOK: 800,
       LINKEDIN: 800,
       X_TWITTER: 280,
       YOUTUBE: 400
     };
     return Object.entries(platforms).map(([platform, maxLength]) => {
       return {
         json: {
           platform: platform,
           text: `${idea} \nCreate a post for ${platform} within ${maxLength} characters and add 3-4 relevant hashtags based on the topic. Write in English only.`
         }
       };
     });
     ```
   - Connect output of `idea` node to this node.

4. **Create OpenAI Node for AI Content Generation**  
   - Node Type: OpenAI (LangChain)  
   - Name: `AI Content Adapter`  
   - Credentials: Configure with your OpenAI API credentials.  
   - Parameters:  
     - Model: `gpt-4.1`  
     - System prompt: `"You are an expert in creating content for social media, adapting content for different platforms while maintaining the core message and maximizing engagement for each platform's unique audience. Write in English only."`  
     - User message: Expression usage of the incoming item’s `text` field (use expression: `{{$json["text"]}}`)  
   - Connect output of `Setting Restrictions and Hashtags` node here.

5. **Create Code Node to Map AI Outputs to Platforms**  
   - Node Type: Code (JavaScript)  
   - Name: `Unification of Platforms and Text`  
   - Paste the following JS code:

     ```javascript
     const platforms = ["TELEGRAM", "TIKTOK", "LINKEDIN", "X_TWITTER", "YOUTUBE"];
     return items.map((item, index) => {
       return {
         json: {
           platform: platforms[index] || "unknown",
           text: item.json?.message?.content || ""
         }
       };
     });
     ```
   - Connect output of `AI Content Adapter` here.

6. **Create PostPulse Node to Retrieve Connected Accounts**  
   - Node Type: PostPulse  
   - Name: `Get connected accounts`  
   - Resource: `account`  
   - Credentials: Configure OAuth2 credentials for your PostPulse account.  
   - Connect output of Manual Trigger to this node in parallel with the `idea` node (i.e., both triggered by the manual trigger).

7. **Create Merge Node to Combine AI Posts with Accounts**  
   - Node Type: Merge  
   - Name: `Merge`  
   - Mode: Combine  
   - Fields to match string: `platform`  
   - Connect main input from `Unification of Platforms and Text` node output.  
   - Connect second input from `Get connected accounts` node output.

8. **Create PostPulse Node to Publish Posts as Drafts**  
   - Node Type: PostPulse  
   - Name: `Publish Post`  
   - Resource: `post`  
   - Credentials: Use the same PostPulse OAuth2 credentials.  
   - Parameters:  
     - `isDraft`: `true`  
     - `publications`: Configure JSON to create posts with content truncated appropriately per platform, mapping platform codes to PostPulse platform types. Use the JavaScript expression for truncation logic as in the original workflow (provided below).  
     - `scheduledTime`: Use expression `{{$now.toUTC()}}`  
   - Connect output of `Merge` node here.

**Content truncation and publishing JSON snippet for `Publish Post`:**

Use a JavaScript expression in the `content` field similar to:

```javascript
(function(){
  const fullText = $json.text || '';
  const p = $json.platform;
  let max = 500;
  if(p==='X_TWITTER') max = 280;
  if(p==='BLUE_SKY') max = 300;
  if(fullText.length <= max) return fullText;
  const d = fullText.lastIndexOf('.', max),
        e = fullText.lastIndexOf('!', max),
        q = fullText.lastIndexOf('?', max),
        s = Math.max(d,e,q),
        l = max*0.6;
  if(s>0 && s>=l) return fullText.substring(0,s+1);
  const w = fullText.lastIndexOf(' ', max);
  return fullText.substring(0, w>0?w:max) + '...';
})()
```

And in `platformSettings` field, map platform codes:

```javascript
(function() {
  const platformMapping = {
    'X_TWITTER': 'TWITTER',
    'YOUTUBE': 'YOUTUBE',
    'THREADS': 'THREADS',
    'TIKTOK': 'TIK_TOK',
    'INSTAGRAM': 'INSTAGRAM',
    'FACEBOOK': 'FACEBOOK',
    'LINKEDIN': 'LINKEDIN',
    'BLUE_SKY': 'BLUE_SKY',
    'TELEGRAM': 'TELEGRAM'
  };
  return JSON.stringify({ "type": platformMapping[$json.platform] });
})()
```

9. **Add Sticky Notes for Documentation (Optional)**  
   - Add sticky notes at corresponding workflow positions explaining each block as per the original.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow automatically adapts your ideas into social media posts with AI, adjusts content for different platforms (Twitter, LinkedIn, TikTok, Telegram, YouTube, etc.), and schedules them as drafts in PostPulse for further review or publishing. | Branding, workflow description.                                                                             |
| Linked platforms include Twitter (X), LinkedIn, TikTok, Telegram, YouTube, Threads, Instagram, Facebook, Blue Sky.      | Social media coverage scope.                                                                                 |
| OpenAI GPT-4.1 is used with a system prompt to ensure posts are tailored for platform audiences and written in English. | AI content generation detail.                                                                                 |
| PostPulse OAuth2 credentials required for API access to social accounts and post scheduling.                            | Integration requirements.                                                                                     |
| Platform-specific character limits are respected with custom truncation logic to retain sentence boundaries where possible. | Content length management.                                                                                     |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.