# LinkedIn Image Posting — n8n Workflow Documentation

**Project Name:** LinkedIn Image Posting <br>
**Platform:** n8n (Self-Hosted) <br>
**AI Model:** Google Gemini (via PaLM API) <br>
**LinkedIn API:** n8n LinkedIn OAuth2 node <br>
**Version:** 1.0 <br> 
**Author:** Prasanth D <br>
**PPT:**  [Live here ](https://gamma.app/docs/id18r7r81ksil85)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Prerequisites](#3-prerequisites)
4. [Component Reference](#4-component-reference)
   - 4.1 [Receive Post Title (Webhook)](#41-receive-post-title-webhook-node)
   - 4.2 [AI Agent](#42-ai-agent-node)
   - 4.3 [Simple Memory](#43-simple-memory-node)
   - 4.4 [Google Gemini Chat Model](#44-google-gemini-chat-model-node)
   - 4.5 [Merge](#45-merge-node)
   - 4.6 [Post to LinkedIn](#46-post-to-linkedin-node)
   - 4.7 [Show Confirmation](#47-show-confirmation-node)
5. [HTML Upload Form](#5-html-upload-form)
6. [Credentials Setup](#6-credentials-setup)
7. [End-to-End Testing Guide](#7-end-to-end-testing-guide)
8. [Error Reference](#8-error-reference)
9. [Going Live (Production)](#9-going-live-production)
10. [Limitations & Notes](#10-limitations--notes)

---

## 1. Project Overview

The **LinkedIn Image Posting** workflow is an automated n8n pipeline that takes a post title and an image from a simple HTML form, generates a polished professional LinkedIn caption using Google Gemini AI, and publishes it directly to your LinkedIn profile — all in a single click.

### How It Works
```
User submits form (title + image)
            ↓
Webhook receives POST request with binary image data
            ↓
AI Agent (Gemini) generates professional LinkedIn caption
            ↓
Merge node combines AI output with original image
            ↓
Post to LinkedIn node publishes the post with image
            ↓
Show Confirmation returns HTTP 200 success response
```

### Key Features

- AI-generated captions using Google Gemini
- Accepts image uploads directly from a browser form
- Posts to LinkedIn with a real image (not just text)
- Returns confirmation when posting is complete
- Session memory enables multi-turn AI conversations
- Clean plain-text output — no Markdown, bullets, or asterisks in captions

---

## 2. System Architecture
```
[Receive Post Title — Webhook]
           ↓  (simultaneously fans out)
  ┌────────┴──────────────────────────┐
  │                                   │
[AI Agent]                    [Merge ← input 0]
  │  (Gemini generates caption)        │
  └──────────────► [Merge ← input 1]
                         │
                [Post to LinkedIn]
                         │
               [Show Confirmation]
```

The webhook fans out to both the AI Agent and the first input of the Merge node simultaneously. Once the AI Agent completes, it sends its output to the second Merge input. Merge combines the original binary image with the AI text and passes the complete package to the LinkedIn node.

---

## 3. Prerequisites

| Requirement | Details |
|-------------|---------|
| n8n instance | Self-hosted at `localhost:5678` |
| Google Gemini API key | PaLM API key from Google AI Studio |
| LinkedIn OAuth2 | Connected LinkedIn account in n8n credentials |
| LinkedIn Person URN | Your LinkedIn member ID (e.g. `WobOM-vX4K`) |
| HTML upload form | Static HTML to POST title + image |

---

## 4. Component Reference

### 4.1 Receive Post Title (Webhook Node)

**Purpose:** Acts as the entry point. Listens for an HTTP POST from the HTML form carrying both the post title (text field) and the image (binary file).

**Why it's needed:** Without this node there is no way to trigger the workflow or receive the file and title from an external source.

**Configuration:**

| Setting | Value |
|---------|-------|
| HTTP Method | `POST` |
| Path | `receive-post-title` |
| Respond | `Using 'Respond to Webhook' Node` |
| Binary Mode | `Separate` (set in workflow settings) |

**Test URL:**
```
http://localhost:5678/webhook-test/receive-post-title
```

**Production URL:**
```
http://localhost:5678/webhook/receive-post-title
```

> **Note:** Use the Test URL during development. Switch to the Production URL after clicking **Publish** in n8n.

---

### 4.2 AI Agent Node

**Purpose:** Generates a professional LinkedIn post caption based on the submitted title and image. Uses Google Gemini as the language model with passthrough binary image support.

**Why it's needed:** This is the intelligence layer — it converts a raw title into a full, polished LinkedIn post ready for publishing.

**Configuration:**

| Setting | Value |
|---------|-------|
| Agent Type | Conversational Agent |
| Model | Google Gemini Chat (PaLM API) |
| Memory | Simple Memory (Window Buffer, size 10) |
| Passthrough Binary Images | `ON` |
| Output Parser | Enabled |

**System Prompt:**
```
Your task is to create high-quality, professional LinkedIn posts based on the title provided by the user.

Write in clear and polished English that is easy for both technical and non-technical audiences to understand.

Use short paragraphs for readability.

Do not use bullet points.

Do not use asterisks.

Do not use bold, italics, or any Markdown formatting.

Do not use special characters for formatting.

Keep the tone professional, confident, and positive.

Briefly explain why the topic is important and its practical relevance.

End the post with 5–10 relevant hashtags.

Do not use emojis.

Do not include explanations, headings, labels, or meta commentary.

Do not mention AI, prompts, or instructions.

Output only plain text suitable for direct posting on LinkedIn.
```

**User Prompt Template:**
```
Write a LinkedIn post using this title:
{{$binary.data ? "Image attached" : "No image"}}

An image is attached to this request. Write a suitable caption for a professional LinkedIn post.
```

**Dynamic Variables:**

| Variable | Source | Example Output |
|----------|--------|----------------|
| `{{ $binary.data }}` | Webhook binary | Image file object |
| `{{ $json.output }}` | Agent output | Full LinkedIn caption text |

---

### 4.3 Simple Memory Node

**Purpose:** Provides a conversational context window to the AI Agent so it can maintain coherent sessions across multiple turns.

**Why it's needed:** Without memory, each request is stateless. Memory allows the agent to remember prior exchanges within a session.

**Configuration:**

| Setting | Value |
|---------|-------|
| Session ID Type | Custom Key |
| Session Key | `linkedin-post-session` |
| Context Window Length | `10` (last 10 exchanges remembered) |

> **Note:** All users share the same session key in this setup. For multi-user deployments, use a dynamic session key based on user ID or timestamp.

---

### 4.4 Google Gemini Chat Model Node

**Purpose:** Provides the large language model backbone for the AI Agent. Connects to Google's Gemini API using your PaLM API credentials.

**Why it's needed:** The AI Agent node requires a connected language model to generate text. This node supplies that model.

**Configuration:**

| Setting | Value |
|---------|-------|
| Node Type | `lmChatGoogleGemini` |
| Credential | Google Gemini (PaLM) API account |
| Default Options | Standard (no overrides needed) |

---

### 4.5 Merge Node

**Purpose:** Combines two separate data streams into one item — the original binary image from the Webhook and the generated text from the AI Agent.

**Why it's needed:** The LinkedIn node requires both the image binary and the caption text in a single item. The Webhook and the AI Agent output them separately, so Merge is essential.

**Configuration:**

| Setting | Value |
|---------|-------|
| Mode | `Combine` |
| Combine By | `Combine By Position` |
| Input 0 (index 0) | Data from Webhook (contains binary image) |
| Input 1 (index 1) | Data from AI Agent (contains generated text) |

**How it works:**
```
Webhook output  →  Merge Input 0  ─┐
                                    ├─► Combined Item → LinkedIn node
AI Agent output →  Merge Input 1  ─┘
```

---

### 4.6 Post to LinkedIn Node

**Purpose:** Publishes the AI-generated caption and uploaded image to your LinkedIn profile using the official LinkedIn API via n8n's built-in LinkedIn integration.

**Why it's needed:** This is the action node — it is responsible for actually delivering the post to LinkedIn.

**Configuration:**

| Setting | Value |
|---------|-------|
| Person URN | `WobOM-vX4K` (your LinkedIn member ID) |
| Text | `={{ $json.output }}` (AI-generated caption) |
| Share Media Category | `IMAGE` |
| Binary Property Name | `image_0` |
| Credential | LinkedIn OAuth2 account |

> **Note:** The binary property name `image_0` comes from the Webhook receiving the image under the field name `image`, with n8n appending `0` as the index.

**Example published post structure:**
```json
{
  "author": "urn:li:person:WobOM-vX4K",
  "commentary": "Launching a new automation project is one of the most rewarding...\n\n#n8n #Automation #LinkedIn",
  "content": {
    "media": {
      "image": "<binary image data>"
    }
  }
}
```

---

### 4.7 Show Confirmation Node (Respond to Webhook)

**Purpose:** Sends an HTTP response back to the browser form after the workflow completes, confirming that the post was published successfully.

**Why it's needed:** The Webhook node is configured to respond via a Respond to Webhook node. Without this node, the browser would hang waiting for a response.

**Configuration:**

| Setting | Value |
|---------|-------|
| Respond With | `Text` |
| Response Body | `your workflow is completed` |
| Response Code | `200` |

---

## 5. HTML Upload Form

This is the front-end interface used to trigger the workflow. It submits a post title and an image to the webhook endpoint.

### Code
```html
<!DOCTYPE html>
<html>
<head>
  <title>LinkedIn Image Poster</title>
</head>
<body>
  <h2>Post to LinkedIn with AI Caption</h2>

  <!-- For Testing: use webhook-test URL -->
  <form action="http://localhost:5678/webhook-test/receive-post-title"
        method="POST"
        enctype="multipart/form-data">

    <label>Post Title:</label><br><br>
    <input type="text" name="title" placeholder="Enter your post topic" required>

    <br><br>
    <label>Select Image:</label><br><br>
    <input type="file" name="image" accept="image/*" required>

    <br><br>
    <button type="submit">Generate & Post to LinkedIn</button>
  </form>
</body>
</html>
```

### Important Notes

| Item | Testing | Production |
|------|---------|------------|
| Form action URL | `webhook-test/receive-post-title` | `webhook/receive-post-title` |
| Title field name | Must be `title` | Must be `title` |
| Image field name | Must be `image` | Must be `image` |
| Encoding type | `multipart/form-data` | `multipart/form-data` |

> **Critical:** The `name="image"` attribute in the file input determines the binary field name in the Webhook output. n8n appends `0` to it, producing `image_0`. This must match the **Binary Property Name** in the Post to LinkedIn node.

---

## 6. Credentials Setup

### 6.1 Google Gemini (PaLM) API

1. Go to [aistudio.google.com](https://aistudio.google.com)
2. Sign in and click **Get API Key**
3. Copy the API key
4. In n8n, go to **Credentials → New → Google PaLM API**
5. Paste the key and save

### 6.2 LinkedIn OAuth2

1. In n8n, go to **Credentials → New → LinkedIn OAuth2 API**
2. Follow the OAuth flow to connect your LinkedIn account
3. Grant permissions for posting (`w_member_social` scope)
4. Save — it will appear as **LinkedIn account**

### 6.3 Finding Your LinkedIn Person URN

1. Log in to LinkedIn in your browser
2. Go to your profile page
3. Open Developer Tools (`F12`) → **Network** tab
4. Refresh the page and search for `me` in the requests
5. Your URN will appear as a short alphanumeric string (e.g., `WobOM-vX4K`)

---

## 7. End-to-End Testing Guide

### Step 1: Activate Test Mode

1. Open your n8n workflow
2. Click the **Receive Post Title** webhook node
3. Click **"Listen for test event"** — n8n is now waiting for input

### Step 2: Submit the Form

1. Open the HTML form in your browser
2. Type a post title (e.g., `Launched my new n8n automation project`)
3. Select an image file
4. Click **"Generate & Post to LinkedIn"**

### Step 3: Verify Webhook Received Data

Check the Webhook node OUTPUT panel — you should see:
```
Binary: image_0
File Name: your-image.jpg
Mime Type: image/jpeg
Text Field: title = "Launched my new n8n automation project"
```

### Step 4: Verify AI Agent Output

The AI Agent node output should contain a field called `output` with a full LinkedIn post. Example:
```
Launching a new automation project is one of the most rewarding experiences
in the world of software development. It brings together creativity,
technical skill, and a drive to solve real problems efficiently.

This project reflects months of learning and iteration, and marks a new
milestone in building scalable, intelligent workflows.

#n8n #Automation #WorkflowAutomation #LinkedIn #AITools #Productivity
```

### Step 5: Verify LinkedIn Post

1. Go to your LinkedIn profile
2. Check **Posts** — the image and AI-generated caption should be visible
3. Confirm hashtags are present and the image displays correctly

### Step 6: Confirm HTTP Response

The browser should display or receive:
```
your workflow is completed
```

---

## 8. Error Reference

### Error 1: 401 Unauthorized (LinkedIn)

**Message:**
```json
{ "status": 401, "message": "Unauthorized" }
```

**Cause:** LinkedIn OAuth2 token has expired or was not granted correct scopes.

**Fix:** Re-authenticate the LinkedIn credential in n8n. Ensure the `w_member_social` scope is enabled during OAuth.

---

### Error 2: Binary Image Not Found in LinkedIn Node

**Symptom:** Post to LinkedIn node throws a binary data error.

**Cause:** Binary property name mismatch. The form field is `image` but the LinkedIn node expects `image_0`.

**Fix:** Confirm that the LinkedIn node's **Binary Property Name** is set to `image_0` to match n8n's auto-indexed binary field naming.

---

### Error 3: AI Agent Output is Empty

**Symptom:** The `output` field from the AI Agent node is blank or missing.

**Cause:** Google Gemini API key is missing, invalid, or quota has been exceeded.

**Fix:** Verify the PaLM API credential in n8n. Check your API quota at [aistudio.google.com](https://aistudio.google.com).

---

### Error 4: Merge Node Has Mismatched Items

**Symptom:** Merged output is missing either the image or the caption text.

**Cause:** One of the Merge inputs completed significantly before the other, causing a position mismatch.

**Fix:** Ensure Merge mode is set to **Combine By Position**. Both inputs must complete before Merge proceeds.

---

### Error 5: No Respond to Webhook Node Found

**Message:**
```json
{ "code": 0, "message": "No Respond to Webhook node found in the workflow" }
```

**Cause:** The Webhook node's **Respond** setting is `Using 'Respond to Webhook' Node` but the Show Confirmation node is missing or disconnected.

**Fix:** Ensure the **Show Confirmation** (Respond to Webhook) node is properly connected at the end of the workflow.

---

## 9. Going Live (Production)

### Step 1: Publish the Workflow

1. Click the **Publish** button in the top right of n8n
2. The workflow is now active and runs automatically on every form submission

### Step 2: Update the HTML Form URL

Change the form action from the test URL to the production URL:
```html
<!-- Testing URL (before publish) -->
<form action="http://localhost:5678/webhook-test/receive-post-title" ...>

<!-- Production URL (after publish) -->
<form action="http://localhost:5678/webhook/receive-post-title" ...>
```

### Step 3: Customize the System Prompt (Optional)

In the AI Agent node, update the system prompt to match your personal brand voice, industry, or language preference.

### Step 4: Re-authenticate LinkedIn Periodically

LinkedIn OAuth2 tokens expire. Set a reminder to reconnect your LinkedIn credential in n8n every 60 days.

---

## 10. Limitations & Notes

| Limitation | Details |
|-----------|---------|
| Shared session memory | All users share one Gemini session key — use dynamic keys for multi-user setups |
| LinkedIn rate limits | LinkedIn API limits the number of posts per day on free accounts |
| Image formats | Accepts any browser-supported image type; LinkedIn recommends JPG or PNG |
| n8n must be running | Workflow only functions while n8n is active on localhost |
| LinkedIn token expiry | OAuth2 tokens expire; periodic re-authentication is required |
| Language | Gemini writes in English by default; modify the system prompt for other languages |
| Caption style | The system prompt enforces no Markdown — update it if you want emoji or formatting |

---

## Final Workflow Summary
```
Receive Post Title (Webhook)
  ├─► Merge (input 0) ────────────────────────────────────┐
  └─► AI Agent                                             │
        ├── Google Gemini Chat Model (language model)      │
        └── Simple Memory (session context)                │
              └─► Merge (input 1) ────────────────────────►┤
                                                           │
                                               Post to LinkedIn
                                                           │
                                              Show Confirmation ✅
```

| Node | Type | Purpose |
|------|------|---------|
| Receive Post Title | Webhook | Entry point — receives title + image via HTTP POST |
| AI Agent | LangChain Agent | Generates professional LinkedIn caption using Gemini |
| Simple Memory | Memory Buffer | Gives AI session context window |
| Google Gemini Chat Model | Language Model | Powers the AI Agent with Gemini LLM |
| Merge | Merge | Combines AI text output with original binary image |
| Post to LinkedIn | LinkedIn | Publishes the image and caption to LinkedIn |
| Show Confirmation | Respond to Webhook | Returns HTTP 200 success response to browser |

**Total Nodes:** 7
**External Services:** Google Gemini API, LinkedIn API
**Trigger:** HTTP POST with post title text and image file
**Output:** AI-written LinkedIn post published with your image ✅

---

*Documentation generated for LinkedIn Image Posting v1.0 — n8n Self-Hosted Workflow*
*Documented by Prasanth D*