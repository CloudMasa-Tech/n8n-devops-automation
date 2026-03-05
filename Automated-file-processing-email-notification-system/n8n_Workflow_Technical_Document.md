# Venkat_Workflow — Technical Documentation

**Workflow ID:** `SaOIIp9vQjjFQJfb`  
**Version ID:** `b352fe85-557a-4c28-b197-6012248b684e`  
**Platform:** n8n  
**Status:** Inactive (`active: false`)  
**Execution Order:** v1  
**Binary Mode:** Separate  

---

## 1. Overview

`Venkat_Workflow` is an n8n automation that performs a secure file distribution pipeline. When triggered manually, it downloads a file from Google Drive, compresses it into a ZIP archive, sends it via Gmail to a recipient, waits one minute, permanently moves the original file to Trash on Google Drive, and finally sends a confirmation email notifying the recipient that the file has been cleaned up.

---

## 2. Architecture Diagram

```
[Manual Trigger]
       │
       ▼
[Download File] ──── Google Drive (OAuth2)
       │
       ▼
[Compression] ──── Compress to ZIP (RecentlyAddedFile)
       │
       ▼
[Send a message] ──── Gmail (ZIP attached)
       │
       ▼
[Wait 1 minute]
       │
       ▼
[Delete a file] ──── Google Drive (Move to Trash)
       │
       ▼
[Send a message1] ──── Gmail (Deletion confirmation)
```

---

## 3. Node Reference

### 3.1 Manual Trigger

| Property | Value |
|----------|-------|
| **Node ID** | `5dc4dd78-f254-4664-a2dd-0dde287fa41d` |
| **Type** | `n8n-nodes-base.manualTrigger` |
| **Version** | 1 |
| **Position** | `[144, -112]` |

**Description:** Entry point of the workflow. The workflow executes only when the user manually clicks the "Execute workflow" button in the n8n interface. There is no scheduled or webhook-based trigger.

---

### 3.2 Download File

| Property | Value |
|----------|-------|
| **Node ID** | `f8366b9e-2355-4d13-a3a8-e1d5ff5eb73b` |
| **Type** | `n8n-nodes-base.googleDrive` |
| **Version** | 3 |
| **Position** | `[384, -112]` |
| **Operation** | `download` |
| **Credential** | Google Drive OAuth2 (`mOli3M6Dtuw514K9`) |

**File Target:**
```
https://drive.google.com/file/d/1jsYfaYk17jv3sjbtBoyeQzI_9WdI-g9z/view?usp=drive_link
```

**Description:** Downloads the specified file from Google Drive using a URL-mode file ID resolution. The file is loaded into n8n's binary data pipeline for downstream processing.

---

### 3.3 Compression

| Property | Value |
|----------|-------|
| **Node ID** | `3ebc8e39-61dd-4d77-96dd-2205c4c72c04` |
| **Type** | `n8n-nodes-base.compression` |
| **Version** | 1.1 |
| **Position** | `[656, -112]` |
| **Operation** | `compress` |
| **Output Filename** | `RecentlyAddedFile` |

**Description:** Takes the downloaded binary file and compresses it into a ZIP archive named `RecentlyAddedFile`. The output is a single compressed binary passed to the Gmail node as an attachment.

---

### 3.4 Send a message (Primary Email)

| Property | Value |
|----------|-------|
| **Node ID** | `f5744f10-64ce-4135-9d0c-409bd0f2a01d` |
| **Type** | `n8n-nodes-base.gmail` |
| **Version** | 2.2 |
| **Position** | `[928, -112]` |
| **Credential** | Gmail OAuth2 (`yTzBagIeecGnaXP2`) |
| **Webhook ID** | `ff4296f5-c9be-4922-8011-c97db76b9a86` |

**Email Details:**

| Field | Value |
|-------|-------|
| **To** | `kaaaran28@gmail.com` |
| **Subject** | `N8n Automation` |
| **Body** | *"This is the zipped file from the folder and after you receive this message the file will be automatically deleted from the automation workflow"* |
| **Attachment** | Binary data from Compression node (ZIP file) |

**Description:** Sends the compressed ZIP file to the designated recipient. Informs the recipient that the source file will be deleted after delivery.

---

### 3.5 Wait

| Property | Value |
|----------|-------|
| **Node ID** | `76f06310-2146-4532-8e5d-bb4953f6568f` |
| **Type** | `n8n-nodes-base.wait` |
| **Version** | 1.1 |
| **Position** | `[288, 144]` |
| **Webhook ID** | `667ff7f0-f8e9-44b2-aedb-f8a736532144` |
| **Duration** | 1 minute |

**Description:** Introduces a deliberate 1-minute delay between the email send and the file deletion step. This provides a buffer to ensure the email is fully delivered before the source file is removed from Google Drive.

---

### 3.6 Delete a file

| Property | Value |
|----------|-------|
| **Node ID** | `f0ef0b63-8936-4f61-9f8a-0067e4443227` |
| **Type** | `n8n-nodes-base.googleDrive` |
| **Version** | 3 |
| **Position** | `[544, 144]` |
| **Operation** | `deleteFile` |
| **Credential** | Google Drive OAuth2 (`mOli3M6Dtuw514K9`) |
| **Delete Permanently** | `false` (moves to Trash) |

**File Target:**
```
https://drive.google.com/file/d/1jsYfaYk17jv3sjbtBoyeQzI_9WdI-g9z/view?usp=drive_link
```

**Description:** Deletes the original source file from Google Drive. With `deletePermanently: false`, the file is moved to Google Drive Trash (recoverable) rather than being permanently destroyed.

---

### 3.7 Send a message1 (Confirmation Email)

| Property | Value |
|----------|-------|
| **Node ID** | `59dd798b-507c-448b-a74d-a7f696688bd0` |
| **Type** | `n8n-nodes-base.gmail` |
| **Version** | 2.2 |
| **Position** | `[800, 144]` |
| **Credential** | Gmail OAuth2 (`yTzBagIeecGnaXP2`) |
| **Webhook ID** | `ff4296f5-c9be-4922-8011-c97db76b9a86` |

**Email Details:**

| Field | Value |
|-------|-------|
| **To** | `kaaaran28@gmail.com` |
| **Subject** | `N8n Automation Done` |
| **Body** | *"The sended zip file deleted successfully"* |
| **Attachment** | None |

**Description:** Sends a final confirmation email notifying the recipient that the file deletion has completed successfully. Terminal node of the workflow.

---

## 4. Connection Map

| Source Node | Target Node | Connection Type |
|-------------|-------------|-----------------|
| When clicking 'Execute workflow' | Download file | main → main[0] |
| Download file | Compression | main → main[0] |
| Compression | Send a message | main → main[0] |
| Send a message | Wait | main → main[0] |
| Wait | Delete a file | main → main[0] |
| Delete a file | Send a message1 | main → main[0] |

All connections are sequential `main`-type with no branching or error paths.

---

## 5. Credentials Summary

| Credential Name | Type | ID | Used By |
|-----------------|------|----|---------|
| Google Drive account | `googleDriveOAuth2Api` | `mOli3M6Dtuw514K9` | Download file, Delete a file |
| Gmail account | `gmailOAuth2` | `yTzBagIeecGnaXP2` | Send a message, Send a message1 |

---

## 6. Execution Flow Summary

```
Step 1  →  Manual trigger fires
Step 2  →  File downloaded from Google Drive (binary output)
Step 3  →  File compressed into ZIP archive ("RecentlyAddedFile")
Step 4  →  ZIP sent as email attachment to kaaaran28@gmail.com
Step 5  →  Workflow pauses for 1 minute
Step 6  →  Original file moved to Google Drive Trash
Step 7  →  Confirmation email sent to kaaaran28@gmail.com
```

---

## 7. Known Limitations & Notes

- **Static File ID:** The Google Drive file URL is hardcoded. To reuse this workflow for different files, the file ID must be updated in both the `Download file` and `Delete a file` nodes.
- **Single Recipient:** The email recipient (`kaaaran28@gmail.com`) is hardcoded. Dynamic recipients would require expression-based configuration.
- **Soft Delete Only:** The file is moved to Trash, not permanently deleted. It can be restored within Google Drive's 30-day Trash retention window.
- **No Error Handling:** The workflow has no error branch or retry logic. A failure at any step will halt execution without notification.
- **Workflow is Inactive:** The `active: false` flag means this workflow will not run automatically. It must be manually triggered from the n8n UI.
- **MCP Disabled:** `availableInMCP: false` — this workflow is not exposed via the Model Context Protocol interface.

---

## 8. Metadata

| Property | Value |
|----------|-------|
| **Workflow Name** | `Venkat_Workflow` |
| **Workflow ID** | `SaOIIp9vQjjFQJfb` |
| **Version ID** | `b352fe85-557a-4c28-b197-6012248b684e` |
| **Instance ID** | `43d140dd61c3945a1199d37c1fc724ad261c1ceaab2b95f2209fc2db179975f8` |
| **Tags** | None |
| **Pin Data** | None |
