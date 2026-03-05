# Technical Documentation: Automated Weekly Service Report Workflow

**Workflow Name:** Automated Weekly Service Report Workflow <br>
**Workflow ID:** `Pm3LW5i7eNcL2BPX` <br>
**Status:** Inactive <br>
**Execution Order:** v1 <br>
**PPT**: [Link here ](https://gamma.app/docs/Automated-Weekly-Service-Report-System-n8n-u9765w9p0w2x2n4?mode=doc )

---

## Overview

This workflow automates the process of extracting service data from a MySQL database on a scheduled basis, transforming it, appending it to a Google Sheet, and notifying a recipient via Gmail with a link to the updated report.

---

## Architecture Diagram

```
[Schedule Trigger]
       ↓
[Execute SQL Query]  ← MySQL DB
       ↓
[Code in JavaScript]  ← Data transformation
       ↓
[Append Row in Sheet]  ← Google Sheets
       ↓
[Aggregate]
       ↓
[Send a Message]  ← Gmail
```

---

## Trigger

### Schedule Trigger
| Property | Value |
|---|---|
| Node Type | `n8n-nodes-base.scheduleTrigger` |
| Node ID | `f8449449-2338-4c28-be3f-5493c6555657` |
| Trigger Time | Daily at **1:03 PM** |

The workflow runs automatically once per day at 13:03 (1:03 PM) server time.

---

## Nodes

### 1. Execute a SQL Query

| Property | Value |
|---|---|
| Node Type | `n8n-nodes-base.mySql` |
| Node ID | `299bb064-01b3-439f-8162-46f00584890a` |
| Operation | `executeQuery` |
| Credential | MySQL account *(replace with your credential ID and name)* |

**SQL Query:**
```sql
SELECT *
FROM service
WHERE date >= DATE_SUB(CURDATE(), INTERVAL 6 DAY)
  AND date <= CURDATE();
```

**Purpose:** Fetches all records from the `service` table for the **last 7 days** (today inclusive), effectively a rolling weekly dataset.

---

### 2. Code in JavaScript

| Property | Value |
|---|---|
| Node Type | `n8n-nodes-base.code` |
| Node ID | `af99707e-0a94-466b-acdf-11deb9e1ac85` |
| Language | JavaScript |

**Purpose:** Transforms and sanitizes the raw SQL output. Maps each record to a clean object with only the required fields. Returns a fallback message (`"No Data Found"`) if the query returns no rows.

**Fields Extracted:**
- `id`
- `name`
- `service`
- `legal_fee_category`
- `legal_fee`
- `govt_fee_category`
- `govt_fee`
- `total`
- `date`

**Logic:**
```javascript
if (items.length > 0) {
  // Map and return all items with defined fields
} else {
  // Return { message: "No Data Found" }
}
```

---

### 3. Append Row in Sheet

| Property | Value |
|---|---|
| Node Type | `n8n-nodes-base.googleSheets` |
| Node ID | `e72f6adc-ffff-4b2e-8c06-757a09483198` |
| Operation | `append` |
| Credential | Google Sheets account 2 (ID: `rLlpU6gSGIeYratV`) |
| Document ID | `1421yI5dAJfQSKyA223RCKRYXDOatgg3OPJgd6edimjs` |
| Sheet | Sheet1 (`gid=0`) |
| Mapping Mode | Auto-map input data |
| Match Column | `id` |

**Sheet URL:**
```
https://docs.google.com/spreadsheets/d/1421yI5dAJfQSKyA223RCKRYXDOatgg3OPJgd6edimjs/edit#gid=0
```

**Column Schema:**

| Column | Type | Match Key |
|---|---|---|
| `id` | string | ✅ Yes (default) |
| `name` | string | No |
| `service` | string | No |
| `legal_fee_category` | string | No |
| `legal_fee` | string | No |
| `govt_fee_category` | string | No |
| `govt_fee` | string | No |
| `total` | string | No |
| `date` | string | No |

---

### 4. Aggregate

| Property | Value |
|---|---|
| Node Type | `n8n-nodes-base.aggregate` |
| Node ID | `029dbfaf-26ba-4ca0-8637-90ec8c9971b0` |
| Mode | Aggregate All Item Data |

**Purpose:** Combines all output items from the Google Sheets append step into a single item before passing to the Gmail node. This ensures the email is sent only once regardless of how many rows were appended.

---

### 5. Send a Message (Gmail)

| Property | Value |
|---|---|
| Node Type | `n8n-nodes-base.gmail` |
| Node ID | `30878c8c-c455-4d9c-b480-7dbcdf8d6655` |
| Credential | Gmail account *(replace with your credential ID)* |
| To | `example@gmail.com` *(replace with actual recipient)* |
| Subject | `Report Generated` |

**Email Body:**
```
Here the file is: https://docs.google.com/spreadsheets/d/1421yI5dAJfQSKyA223RCKRYXDOatgg3OPJgd6edimjs/edit?gid=0#gid=0
```

---

## Data Flow Summary

| Step | Input | Output |
|---|---|---|
| Schedule Trigger | Time event | Workflow start signal |
| Execute SQL Query | Date range filter | Raw DB rows (last 7 days) |
| Code in JavaScript | Raw rows | Cleaned, field-mapped objects |
| Append Row in Sheet | Cleaned objects | Rows written to Google Sheet |
| Aggregate | Multiple row outputs | Single combined output |
| Send a Message | Aggregated signal | Email with Google Sheet link |

---

## Credentials Required

| Service | Credential Name | Notes |
|---|---|---|
| MySQL | MySQL account | Replace ID and name with your own |
| Google Sheets | Google Sheets account 2 | OAuth2, ID: `rLlpU6gSGIeYratV` |
| Gmail | Gmail account | OAuth2, replace ID with your own |

---

## Configuration Notes

- **Recipient email** (`example@gmail.com`) must be updated to the actual target address.
- **MySQL credential** ID and name are placeholders — replace with valid n8n credential references.
- **Gmail credential** ID is a placeholder — replace with your actual Gmail OAuth2 credential ID.
- The workflow is currently **inactive** and must be activated in n8n before it runs on schedule.
- The SQL query uses `CURDATE()` which is based on the **MySQL server's timezone** — ensure the server timezone is correctly configured.

---

## Error Handling

The JavaScript node includes basic handling for empty datasets, returning a `"No Data Found"` message. However, there is **no explicit error branch** for SQL failures, Google Sheets write failures, or Gmail send failures. It is recommended to add error handling nodes in production deployments.



