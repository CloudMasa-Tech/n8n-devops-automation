# WhatsApp Bulk Sender — n8n Workflow Documentation

**Project Name:** WhatsApp Bulk Sender  
**Platform:** n8n (Self-Hosted / Cloud)  
**WhatsApp API:** UltraMsg  
**Version:** 1.1  
**Date:** March 5, 2026  
**Author:** Monesh  
**Developer Portfolio:** [moneshgomon8n.netlify.app](https://moneshgomon8n.netlify.app)  
**PPT:** [View here](https://app.presentations.ai/view/K2qkFWvDfp)

---

<img width="1342" height="589" alt="image" src="https://github.com/user-attachments/assets/c775fa41-f5f5-4f6c-84f1-37e0e8bc7b60" />


## Table of Contents

1. [Project Overview](#1-project-overview)
2. [System Architecture](#2-system-architecture)
3. [Prerequisites](#3-prerequisites)
4. [Component Reference](#4-component-reference)
   - 4.1 [Webhook Node](#41-webhook-node)
   - 4.2 [Extract from File Node](#42-extract-from-file-node)
   - 4.3 [Loop Over Items Node](#43-loop-over-items-node)
   - 4.4 [HTTP Request Node](#44-http-request-node)
5. [Excel File Format](#5-excel-file-format)
6. [HTML Upload Form](#6-html-upload-form)
7. [UltraMsg API Setup](#7-ultramsg-api-setup)
8. [End-to-End Testing Guide](#8-end-to-end-testing-guide)
9. [Sample Test Data](#9-sample-test-data)
10. [Error Reference](#10-error-reference)
11. [Going Live (Production)](#11-going-live-production)
12. [Limitations & Notes](#12-limitations--notes)

---

## 1. Project Overview

The **WhatsApp Bulk Sender** is an automated workflow built in n8n that allows users to send personalized WhatsApp messages to multiple contacts by simply uploading an Excel file.

### How It Works

```
User uploads Excel file (.xlsx)
            ↓
Webhook receives the file as binary data
            ↓
Extract from File parses the Excel into rows
            ↓
Loop Over Items processes one contact at a time
            ↓
HTTP Request sends WhatsApp message via UltraMsg API
            ↓
Each contact receives a personalized WhatsApp message
```

### Key Features

- Upload an Excel file with names and phone numbers
- Automatically sends a personalized WhatsApp message to every contact
- Processes contacts one at a time to avoid API rate limits
- No manual intervention required after upload
- Supports any number of contacts in the Excel file

### Message Template (Current)

```
Hello {Name}, congratulations! Based on your profile evaluation, you have been 
selected for the next round of our Cloud & DevOps hiring process. Our team will 
contact you soon with further details. Thank you for your interest.
```

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        USER                                  │
│              Opens HTML Upload Form                          │
│              Selects Excel File (.xlsx)                      │
│              Clicks Upload Button                            │
└───────────────────────┬─────────────────────────────────────┘
                        │  HTTP POST (multipart/form-data)
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                   n8n WORKFLOW                               │
│                                                             │
│  [Webhook] → [Extract from File] → [Loop Over Items]        │
│                                           ↓                 │
│                                   [HTTP Request]            │
│                                           ↓                 │
│                              (loops back for each row)      │
└───────────────────────┬─────────────────────────────────────┘
                        │  REST API call
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                   UltraMsg API                               │
│           https://api.ultramsg.com                          │
│         (Connected to your WhatsApp number)                 │
└───────────────────────┬─────────────────────────────────────┘
                        │  WhatsApp message delivery
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              RECIPIENTS' WHATSAPP                            │
│   Each contact receives a personalized message              │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Prerequisites

| Requirement | Details |
|-------------|---------|
| n8n instance | Self-hosted or n8n Cloud |
| UltraMsg account | Free trial at ultramsg.com |
| WhatsApp number | Connected to UltraMsg via QR scan |
| Excel file | `.xlsx` format with `Name` and `Contact` columns |
| HTML upload form | Static HTML file to upload Excel via browser |

### UltraMsg Credentials Required

> ⚠️ **After importing the workflow, you must update these two values in the HTTP Request node before running.**

| Field | Placeholder in Workflow | Where to Find |
|-------|------------------------|---------------|
| Instance ID | `YOUR_INSTANCE_ID` | UltraMsg Dashboard → Instance page |
| API Token | `YOUR_API_TOKEN` | UltraMsg Dashboard → Instance page |
| Full API URL | `https://api.ultramsg.com/YOUR_INSTANCE_ID/messages/chat?token=YOUR_API_TOKEN` | Constructed from above |

---

## 4. Component Reference

### 4.1 Webhook Node

**Purpose:** Acts as the entry point of the workflow. It listens for incoming HTTP POST requests carrying the Excel file.

**Why it's needed:** Without this node, there is no way to trigger the workflow or receive the file from an external source (like the HTML form).

**Configuration:**

| Setting | Value |
|---------|-------|
| HTTP Method | `POST` |
| Path | `upload-excel` |
| Respond | `Immediately` |
| Field Name for Binary Data | `data` |
| Raw Body | `OFF` |

**Test URL:**
```
http://localhost:5678/webhook-test/upload-excel
```

**Production URL:**
```
http://localhost:5678/webhook/upload-excel
```

> **Note:** Use the Test URL during development. Switch to the Production URL after clicking **Publish** in n8n.

---

### 4.2 Extract from File Node

**Purpose:** Converts the raw binary Excel file received from the Webhook into structured JSON rows that n8n can process.

**Why it's needed:** The Webhook receives the Excel file as raw binary data (unreadable by n8n). This node acts as a parser that translates every row in the Excel sheet into a JSON object.

**Configuration:**

| Setting | Value |
|---------|-------|
| Operation | `Extract From XLSX` |
| Input Binary Field | `data0` |

**Example Input (Binary):**
```
File: ContactData.xlsx
Mime Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
```

**Example Output (JSON):**
```json
[
  { "Name": "Monesh", "Contact": 9845726130 },
  { "Name": "Gomo",   "Contact": 9173648250 }
]
```

> **Note:** The binary field name is `data0` (not `data`) because the HTML form sends the file under the key `data`, and n8n automatically appends `0` as the index.

---

### 4.3 Loop Over Items Node

**Purpose:** Iterates over each row from the Excel file one by one, sending each contact individually to the HTTP Request node.

**Why it's needed:** Without this node, all contacts would be sent to the HTTP Request node simultaneously, which can cause API rate limit errors or missed messages. The loop ensures contacts are processed sequentially.

**Configuration:**

| Setting | Value |
|---------|-------|
| Node Type | `splitInBatches` (v3) |
| Batch Size | `1` (default) |

**How the loop works:**

```
Excel has 3 contacts → Loop iterates 3 times

Iteration 1:  Sends { Name: "Monesh", Contact: 9845726130 } → HTTP Request
Iteration 2:  Sends { Name: "Gomo",   Contact: 9173648250 } → HTTP Request
Iteration 3:  Done → workflow ends
```

**Loop Outputs:**

| Output | Behaviour |
|--------|-----------|
| Output 0 (loop) | Sends current batch to HTTP Request node |
| Output 1 (done) | Loops back to itself until all items are processed |

> **Best Practice:** Keep Batch Size at `1` to avoid WhatsApp rate limiting.

**Scaling:**

| Contacts in Excel | Loop Iterations |
|-------------------|----------------|
| 3 | 3 |
| 50 | 50 |
| 100 | 100 |
| 1000 | 1000 |

---

### 4.4 HTTP Request Node

**Purpose:** Sends a personalized WhatsApp message to each contact by calling the UltraMsg API.

**Why it's needed:** This is the action node — it is responsible for actually delivering the WhatsApp message to each phone number using the UltraMsg REST API.

**Configuration:**

| Setting | Value |
|---------|-------|
| Method | `POST` |
| URL | `https://api.ultramsg.com/YOUR_INSTANCE_ID/messages/chat?token=YOUR_API_TOKEN` |
| Body Content Type | `JSON` |
| Body Spec | `Specify Body: JSON (raw)` |

**Request Body (current message template):**
```json
{
  "to": "91{{ $json.Contact }}",
  "body": "Hello {{ $json.Name }}, congratulations! Based on your profile evaluation, you have been selected for the next round of our Cloud & DevOps hiring process. Our team will contact you soon with further details. Thank you for your interest."
}
```

**Dynamic Variables:**

| Variable | Source | Example Output |
|----------|--------|----------------|
| `{{ $json.Name }}` | Excel `Name` column | `Monesh` |
| `{{ $json.Contact }}` | Excel `Contact` column | `9845726130` |

**Example API call:**
```
POST https://api.ultramsg.com/YOUR_INSTANCE_ID/messages/chat?token=YOUR_API_TOKEN

Body:
{
  "to": "919845726130",
  "body": "Hello Monesh, congratulations! Based on your profile evaluation..."
}
```

**Successful Response:**
```json
{
  "sent": "true",
  "message": "ok",
  "id": 2
}
```

> **Note:** The `91` prefix is the India country code. If your contacts are from a different country, replace `91` with the appropriate country code.

---

## 5. Excel File Format

The Excel file must follow this exact format:

### Required Columns

| Column Name | Data Type | Example | Notes |
|-------------|-----------|---------|-------|
| `Name` | Text | `Monesh` | Contact's full name |
| `Contact` | Number | `9845726130` | 10-digit mobile number (without country code) |

### Sample Excel Data

| Name       | Contact    |
|------------|------------|
| Monesh     | 9845726130 |
| Gomo       | 9173648250 |
| NewPerson  | 9081746523 |

### Rules

- File must be saved as `.xlsx` format
- Column headers must be exactly `Name` and `Contact` (case-sensitive)
- Contact numbers must be 10 digits without country code or spaces
- Do not include any merged cells or extra formatting
- First row must be the header row

---

## 6. HTML Upload Form

This is the front-end interface used to upload the Excel file and trigger the workflow.

### Code

```html
<!DOCTYPE html>
<html>
<head>
  <title>WhatsApp Bulk Sender</title>
</head>
<body>
  <h2>Upload Excel File to Send WhatsApp Messages</h2>

  <!-- For Testing: use webhook-test URL -->
  <form action="http://localhost:5678/webhook-test/upload-excel"
        method="POST"
        enctype="multipart/form-data">

    <label>Select Excel File (.xlsx):</label><br><br>
    <input type="file" name="data" accept=".xlsx,.xls" required>

    <br><br>
    <button type="submit">Send WhatsApp Messages</button>
  </form>
</body>
</html>
```

### Important Notes

| Item | Testing | Production |
|------|---------|------------|
| Form action URL | `webhook-test/upload-excel` | `webhook/upload-excel` |
| Input `name` attribute | Must be `data` | Must be `data` |
| File type accepted | `.xlsx, .xls` | `.xlsx, .xls` |

> **Critical:** The `name="data"` attribute in the file input must match the **Field Name for Binary Data** setting in the Webhook node (`data`). If these don't match, the file will not be received. The Extract from File node then reads it as `data0` (n8n appends the index automatically).

---

## 7. UltraMsg API Setup

### Step 1: Create Account
1. Go to **ultramsg.com**
2. Click **Sign Up Free**
3. Register with your email address

### Step 2: Create an Instance
1. After login, click **"Add Instance"**
2. Name it (e.g., `BulkSender`)
3. A QR code will appear on screen

### Step 3: Connect Your WhatsApp
1. Open WhatsApp on your mobile phone
2. Go to **Settings → Linked Devices → Link a Device**
3. Scan the QR code shown on UltraMsg
4. Status will change to **"Connected"** ✅

### Step 4: Retrieve Credentials

From the Instance Manage page, note down:

| Field | Location |
|-------|----------|
| Instance ID | Shown on manage page (e.g., `instance163934`) |
| Token | Shown on manage page (e.g., `xxxxxxxxxxxx`) |
| API URL | `https://api.ultramsg.com/{your-instance-id}` |

Update the HTTP Request node URL in the workflow with your real values:
```
https://api.ultramsg.com/{YOUR_INSTANCE_ID}/messages/chat?token={YOUR_TOKEN}
```

### Free Trial Limits

| Limit | Value |
|-------|-------|
| Messages per day | 100 |
| Trial duration | 2 days |
| Contacts | Unlimited |

---

## 8. End-to-End Testing Guide

### Step 1: Prepare Test Excel File

Create an Excel file named `TestContacts.xlsx`:

| Name      | Contact    |
|-----------|------------|
| TestUser1 | 9XXXXXXXXX |
| TestUser2 | 9XXXXXXXXX |

> Replace `9XXXXXXXXX` with real phone numbers you can verify.

### Step 2: Start Listening in n8n

1. Open your n8n workflow
2. Click the **Webhook** node
3. Click **"Listen for test event"**

### Step 3: Upload the File

1. Open your HTML upload form in a browser
2. Select `TestContacts.xlsx`
3. Click **"Send WhatsApp Messages"**

### Step 4: Verify Webhook Received File

In the Webhook node OUTPUT, you should see:

```
Binary: data0
File Name: TestContacts.xlsx
File Extension: xlsx
Mime Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
```

### Step 5: Execute Extract from File

Click **"Execute step"**. You should see:

```json
[
  { "Name": "TestUser1", "Contact": 9XXXXXXXXX },
  { "Name": "TestUser2", "Contact": 9XXXXXXXXX }
]
```

### Step 6: Verify HTTP Request Response

After the HTTP Request node executes:

```json
[
  { "sent": "true", "message": "ok", "id": 1 },
  { "sent": "true", "message": "ok", "id": 2 }
]
```

### Step 7: Confirm on WhatsApp

Recipients should have received:
```
Hello TestUser1, congratulations! Based on your profile evaluation, you have been 
selected for the next round of our Cloud & DevOps hiring process...
```

---

## 9. Sample Test Data

### Test Case 1: Single Contact

**Excel file:**

| Name   | Contact    |
|--------|------------|
| Monesh | 9845726130 |

**Expected WhatsApp message:**
```
Hello Monesh, congratulations! Based on your profile evaluation, you have been 
selected for the next round of our Cloud & DevOps hiring process. Our team will 
contact you soon with further details. Thank you for your interest.
```

---

### Test Case 2: Multiple Contacts

**Excel file:**

| Name       | Contact    |
|------------|------------|
| Monesh     | 9845726130 |
| Gomo       | 9173648250 |
| NewPerson  | 9081746523 |

**Expected n8n output:**
```json
[
  { "sent": "true", "message": "ok", "id": 2 },
  { "sent": "true", "message": "ok", "id": 3 },
  { "sent": "true", "message": "ok", "id": 4 }
]
```

**Loop iterations:** 3 | **Messages sent:** 3

> ⚠️ Contact numbers above are for documentation purposes only. Do not use them in production.

---

## 10. Error Reference

### Error 1: Wrong Token

**Error message:**
```json
{ "error": "Wrong token. Please provide token as a GET parameter." }
```

**Cause:** Token is missing from the URL or incorrect.

**Fix:** Update the HTTP Request URL with your real token:
```
https://api.ultramsg.com/YOUR_INSTANCE_ID/messages/chat?token=YOUR_TOKEN
```

---

### Error 2: `to` and `body` are required

**Error message:**
```json
{ "error": [{ "to": "is required" }, { "body": "is required" }] }
```

**Cause:** Body Content Type is set to `Form-Data` instead of `JSON`.

**Fix:** In the HTTP Request node, set **Specify Body** to `JSON` and use:
```json
{
  "to": "91{{ $json.Contact }}",
  "body": "Hello {{ $json.Name }}, congratulations!..."
}
```

---

### Error 3: No Respond to Webhook Node

**Error message:**
```json
{ "code": 0, "message": "No Respond to Webhook node found in the workflow" }
```

**Cause:** Webhook node's **Respond** setting is set to `Using 'Respond to Webhook' Node` but no such node exists.

**Fix:** Change the Webhook node's **Respond** setting to `Immediately`.

---

### Error 4: File Not Received

**Symptom:** Webhook fires but no binary data appears in output.

**Cause:** The HTML form's input `name` attribute does not match the Webhook's **Field Name for Binary Data**.

**Fix:** Ensure the form input uses `name="data"`:
```html
<input type="file" name="data" accept=".xlsx,.xls">
```

---

### Error 5: Empty Output from Extract from File

**Symptom:** Extract from File node shows no rows.

**Cause:** Input Binary Field is set to `data` instead of `data0`.

**Fix:** Change **Input Binary Field** to `data0` in the Extract from File node.

---

## 11. Going Live (Production)

### Step 1: Update UltraMsg Credentials
Replace the placeholders in the HTTP Request node URL:
- `YOUR_INSTANCE_ID` → your real UltraMsg instance ID
- `YOUR_API_TOKEN` → your real UltraMsg token

### Step 2: Publish the Workflow
1. Click the **"Publish"** button in the top right of n8n
2. The workflow is now active and will run automatically on new uploads

### Step 3: Update the HTML Form URL

```html
<!-- Testing (before publish) -->
<form action="http://localhost:5678/webhook-test/upload-excel" ...>

<!-- Production (after publish) -->
<form action="http://localhost:5678/webhook/upload-excel" ...>
```

### Step 4: Customise Your Message

In the HTTP Request node, update the `body` field as needed:

```json
{
  "to": "91{{ $json.Contact }}",
  "body": "Dear {{ $json.Name }}, we are pleased to inform you about our latest offer. Reply YES to know more!"
}
```

---

## 12. Limitations & Notes

| Limitation | Details |
|-----------|---------|
| UltraMsg free trial | Expires after 2 days, 100 messages/day limit |
| Country code | Currently hardcoded to India (`91`). Change for other countries |
| Column names | Must be exactly `Name` and `Contact` (case-sensitive) |
| File format | Only `.xlsx` and `.xls` supported |
| n8n must be running | Workflow only works while n8n is active |
| WhatsApp connection | Your WhatsApp must remain linked to UltraMsg instance |
| Credentials | `YOUR_INSTANCE_ID` and `YOUR_API_TOKEN` must be replaced before first run |

---

## Workflow JSON Summary

```
Webhook (POST /upload-excel)
  └─► Extract from File (operation: xlsx, field: data0)
          └─► Loop Over Items (splitInBatches v3, batch: 1)
                  ├─► [loop output] HTTP Request (UltraMsg API POST)
                  └─► [done output] → back to Loop Over Items
```

**Total nodes:** 4  
**External services:** UltraMsg API  
**Trigger:** HTTP POST with Excel file upload  
**Output:** WhatsApp messages sent to all contacts in Excel ✅

---

<div align="center">

**Documentation v1.1 — WhatsApp Bulk Sender — n8n Workflow**

Documented by **Monesh D** | [moneshgomo.netlify.app](https://moneshgomo.netlify.app)

</div>