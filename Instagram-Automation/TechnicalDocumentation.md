# Technical Documentation: My Workflow (n8n)

## Overview

**Workflow Name:** My workflow  
**Workflow ID:** `FVj6PniRaI0ORpSw`  
**Status:** Inactive  
**Execution Order:** v1  
**Version ID:** `6fc79f5a-829b-4498-a5d7-c56cd6f7008e`  

This workflow automates social media posting by taking a predefined image URL and caption, then publishing them to Instagram and Facebook via the Ayrshare API.

---

## Architecture

```
[Manual Trigger] ──► [Edit Fields] ──► [HTTP Request → Ayrshare API]
```

---

## Nodes

### 1. Manual Trigger
**Node Name:** `When clicking 'Execute workflow'`  
**Type:** `n8n-nodes-base.manualTrigger`  
**Type Version:** 1  
**Node ID:** `058c1c68-8d89-44dc-9adb-9136e310159f`  
**Position:** `[0, 0]`

**Description:**  
Serves as the entry point of the workflow. This node is triggered manually by the user clicking the "Execute workflow" button in the n8n interface. It passes control to the next node with no parameters.

---

### 2. Edit Fields (Set Node)
**Node Name:** `Edit Fields`  
**Type:** `n8n-nodes-base.set`  
**Type Version:** 3.4  
**Node ID:** `539d1e61-3e64-46f3-b956-95a7b941703e`  
**Position:** `[208, 0]`

**Description:**  
Defines and sets the two key data fields used in the API call downstream.

**Assignments:**

| Field ID | Variable Name | Value | Type |
|---|---|---|---|
| `adb91d5a-3f37-4500-b5bb-da71f458fa3d` | `imageUrl` | `https://th-i.thgim.com/public/entertainment/movies/co3f7l/article68358912.ece/alternates/FREE_1200/kalidas.jpg` | `string` |
| `1abc6cfb-f61a-4b51-9509-93f59a587ee2` | `caption` | `handsome` | `string` |

**Output Fields:**
- `imageUrl` — The publicly accessible URL of the image to be posted.
- `caption` — The text caption/post body to accompany the image.

---

### 3. HTTP Request (Ayrshare Post)
**Node Name:** `HTTP Request`  
**Type:** `n8n-nodes-base.httpRequest`  
**Type Version:** 4.4  
**Node ID:** `90aebb5c-2c0f-4a70-9543-46410bfdd0fa`  
**Position:** `[416, 0]`

**Description:**  
Sends a POST request to the Ayrshare social media API to publish the image and caption to Instagram and Facebook simultaneously.

**Request Configuration:**

| Parameter | Value |
|---|---|
| Method | `POST` |
| URL | `https://app.ayrshare.com/api/post` |
| Body Format | JSON |

**Request Headers:**

| Header | Value |
|---|---|
| `Authorization` | `Bearer DDCD3E14-4F82445B-B1B1593D-21CC12D9` |
| `Content-Type` | `application/json` |

> ⚠️ **Security Note:** The API key is hardcoded in the workflow. It is strongly recommended to store this in n8n's **Credentials** or **Environment Variables** to prevent accidental exposure.

**Request Body:**

```json
{
  "post": "{{ $json.caption }}",
  "platforms": ["instagram", "facebook"],
  "mediaUrls": ["{{ $json.imageUrl }}"]
}
```

**Dynamic Expressions:**
- `{{ $json.caption }}` — Resolved from the `caption` field set in the Edit Fields node.
- `{{ $json.imageUrl }}` — Resolved from the `imageUrl` field set in the Edit Fields node.

---

## Node Connections

| Source Node | Target Node | Connection Type |
|---|---|---|
| `When clicking 'Execute workflow'` | `Edit Fields` | `main[0] → main[0]` |
| `Edit Fields` | `HTTP Request` | `main[0] → main[0]` |

---

## Data Flow

1. **Trigger:** User manually initiates the workflow.
2. **Field Assignment:** The `Edit Fields` node sets `imageUrl` and `caption` as JSON fields in the pipeline.
3. **API Call:** The `HTTP Request` node constructs a JSON body using these fields and sends a POST request to Ayrshare's `/api/post` endpoint.
4. **Outcome:** The image is published with the caption to both Instagram and Facebook.

---

## External Dependencies

| Service | Endpoint | Purpose |
|---|---|---|
| Ayrshare API | `https://app.ayrshare.com/api/post` | Social media cross-posting (Instagram + Facebook) |

---

## Configuration & Settings

| Setting | Value |
|---|---|
| Execution Order | `v1` |
| Binary Mode | `separate` |
| Available in MCP | `false` |
| Active | `false` |

---

## Potential Improvements

- **Parameterize Inputs:** Replace hardcoded `imageUrl` and `caption` with dynamic inputs (e.g., from a webhook, form trigger, or database).
- **Secure the API Key:** Move the Ayrshare Bearer token to n8n's built-in credentials manager.
- **Add Error Handling:** Attach an error branch to the HTTP Request node to handle API failures gracefully.
- **Extend Platform Support:** Ayrshare supports additional platforms (Twitter/X, LinkedIn, TikTok, etc.) that could be added to the `platforms` array.
- **Activate Workflow:** Set `active: true` and connect to a scheduled or event-based trigger for automated posting.
