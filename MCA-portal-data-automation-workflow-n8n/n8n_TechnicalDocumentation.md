# Automated File Upload and Email Notification using n8n
## Overview

- The Excel Upload Tool is a lightweight, single-page web application that allows users to upload `.xlsx` or `.xls` files to an n8n automation workflow via a webhook. Users must also specify a course type (`PVT` or `LLP`) before submitting, which is sent alongside the file as form metadata.

- PPT : [Link here](https://gamma.app/docs/Automated-File-Upload-and-Email-Notification-using-n8n-apa9xjiew65ua5g)
---

## Features

- Upload Excel files (`.xlsx`, `.xls`) directly from the browser
- Select a course type classification (`PVT` or `LLP`) via radio buttons
- Client-side input validation before submission
- Real-time upload status feedback (uploading, success, error)
- Sends data to an n8n webhook endpoint using `multipart/form-data`

---

## File Structure

```
upload.html        # Self-contained single-file application (HTML + CSS + JS)
```

No external dependencies, build steps, or server-side code are required.

---

## Prerequisites

| Requirement | Details |
|---|---|
| n8n instance | Running locally or on a server |
| Webhook URL | Active n8n webhook at `/webhook-test/excel-file` |
| Browser | Any modern browser (Chrome, Firefox, Edge, Safari) |
| File types | `.xlsx` or `.xls` only |

---

## Configuration

### Webhook Endpoint

The upload target is hardcoded in the JavaScript section of `upload.html`:

```javascript
const response = await fetch(
    'http://localhost:5678/webhook-test/excel-file',
    {
        method: 'POST',
        body: formData,
    }
);
```

**To change the endpoint**, update the URL string in the `fetch()` call to point to your n8n instance host, port, and webhook path.

| Parameter | Default Value | Description |
|---|---|---|
| Host | `localhost` | n8n server hostname or IP |
| Port | `5678` | n8n default port |
| Path | `/webhook-test/excel-file` | n8n webhook route |

> **Note:** For production deployments, replace `http://localhost:5678` with your live n8n webhook URL and ensure HTTPS is used.

---

## Usage

1. **Open** `upload.html` in a web browser.
2. **Select a file** by clicking the file input area and choosing an `.xlsx` or `.xls` file.
3. **Choose a course type** by selecting either the `PVT` or `LLP` radio button.
4. **Click "Upload File"** to submit.
5. **Check the status message** displayed below the button for confirmation or error details.

---

## Form Payload

The form data is submitted as `multipart/form-data` with the following fields:

| Field | Type | Description |
|---|---|---|
| `file` | File (`.xlsx` / `.xls`) | The Excel file selected by the user |
| `courseType` | String (`"PVT"` or `"LLP"`) | The course classification radio selection |

---

## Validation Logic

Client-side validation runs before any network request is made. The following checks are enforced in order:

1. **Course type not selected** → Displays: `⚠️ Please select PVT or LLB!`
2. **No file selected** → Displays: `⚠️ Please select a file first!`

If both inputs are valid, the upload proceeds.

---

## Status Messages

| State | Message | Style |
|---|---|---|
| Uploading | `⏳ Uploading...` | Default (grey) |
| Success | `✅ File uploaded successfully!` | Green |
| HTTP Error | `❌ Upload failed.` | Red |
| Network / JS Error | `❌ Error: <error message>` | Red |

---

## n8n Integration

The tool is designed to trigger an **n8n webhook node**. Once the webhook receives the request, the `file` binary and `courseType` string are available in the n8n workflow for downstream processing (e.g., parsing the spreadsheet, routing by course type, writing to a database).

**Recommended n8n workflow structure:**

```
Webhook (POST) → Extract Binary Data → Switch (by courseType) → Process PVT / LLP
```

Ensure the n8n webhook node is set to **accept binary data** and that the workflow is **active** (not in test mode) for production use.

---

## Known Issues & Limitations

| Issue | Detail |
|---|---|
| Hardcoded endpoint | The webhook URL must be manually updated for non-local deployments |
| No file size limit | No client-side cap on file size; large files may time out |
| `LLB` typo in validation | The error message reads `"PVT or LLB"` but the radio value is `"LLP"` — these should be made consistent |
| HTTP only | The default endpoint uses `http://`, which is insecure over public networks |
| No CORS handling | CORS errors will occur if the page is served from a different origin than the n8n instance |

---

## Security Considerations

- **Do not expose** the n8n webhook URL publicly without authentication (use n8n's built-in header authentication or an API gateway).
- **Validate file contents server-side** in n8n — client-side `accept=".xlsx, .xls"` can be bypassed.
- **Use HTTPS** in all non-local deployments to protect file data in transit.

---

## Customisation Guide

### Changing Accepted File Types

Update the `accept` attribute on the file input:

```html
<input type="file" id="fileInput" accept=".xlsx, .xls" />
```

### Adding a New Course Type Option

Add a new radio button in the `.radio-group` div:

```html
<label>
    <input type="radio" name="courseType" value="NEW_TYPE">
    NEW_TYPE
</label>
```

No JavaScript changes are needed — the value is read dynamically.

### Styling

All styles are contained in the `<style>` block within `upload.html`. The layout uses flexbox centering with a card-style container. Primary button color is `#4CAF50` (green).

---

