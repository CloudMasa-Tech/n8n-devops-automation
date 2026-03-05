# N8N Workflow Documentation: Gmail/Calendar Daily Events Digest

## Executive Summary

This document provides comprehensive technical documentation for the **Gmail/Calendar Daily Events Digest** n8n workflow. The workflow automates the process of fetching daily calendar events and sending a formatted email summary to the user each morning.

---

## Workflow Overview

### Purpose
Automatically retrieve all calendar events scheduled for the current day and deliver them via email in a user-friendly, formatted HTML message every morning at 9:00 AM.

### Workflow ID
- **Name:** gmail/calender
- **ID:** cUs1pvxVeA3HuhEF
- **Version ID:** 9c7d8ac9-2e97-40d5-bd29-43232ea315af
- **Status:** Inactive
- **Timezone:** Asia/Kolkata

---

## Architecture & Components

The workflow consists of four sequential nodes that work together to gather, process, and deliver calendar information:

### 1. Schedule Trigger
**Node ID:** b3d3992e-4f0f-4554-b211-9fdfbea3ad96  
**Type:** n8n-nodes-base.scheduleTrigger (v1.3)

**Configuration:**
- **Trigger Time:** Daily at 9:09 AM
- **Frequency:** Daily recurring interval

**Purpose:** Initiates the workflow execution at the specified time each day.

---

### 2. Get Many Events
**Node ID:** 17192a5a-8e10-4d5f-bdc8-9475587033d1  
**Type:** n8n-nodes-base.googleCalendar (v1.3)

**Configuration:**
- **Operation:** getAll
- **Calendar:** xxx@gmail.com
- **Return All Results:** Enabled
- **Time Range:** Next 24 hours from execution time
  - **Minimum Time:** Current time (ISO format)
  - **Maximum Time:** Current time + 24 hours (ISO format)
- **Sorting:** Ordered by start time
- **Timezone:** Asia/Kolkata
- **Credentials:** Google Calendar OAuth2 API (Google Calendar account 3)

**Purpose:** Fetches all events scheduled for the next 24 hours from the specified Google Calendar.

---

### 3. Code in JavaScript
**Node ID:** 5a5b8d11-f0c0-4864-8270-5c0aff7c59f8  
**Type:** n8n-nodes-base.code (v2)

**Functionality:**
Processes the calendar events data and transforms it into a professional HTML email body.

**Processing Logic:**
1. Extracts event data from the calendar API response
2. Formats event times from ISO datetime to readable 12-hour format (e.g., "2:30 PM")
3. Handles "All Day" events appropriately
4. Creates individual event cards with styling
5. Includes event details:
   - Event title
   - Time (or "All Day" if no specific time)
   - Location
   - Description
6. Generates comprehensive HTML with consistent styling
7. Displays "No events for today!" message if calendar is empty

**Output:** JSON object containing formatted HTML email body

---

### 4. Send a Message
**Node ID:** cd1e4340-fe72-41b9-a89f-44d3182cb221  
**Type:** n8n-nodes-base.gmail (v2.2)

**Configuration:**
- **Send To:** xxx@gmail.com
- **Subject:** "Good Morning! Your Events for Today - [Current Date]"
- **Message Body:** Dynamically generated HTML from JavaScript code node
- **Credentials:** Gmail OAuth2 (Gmail account 3)
- **Webhook ID:** b56447a5-5e1e-4ca0-92de-b0fa845d6dc4

**Purpose:** Sends the formatted email with today's events to the user's inbox.

---

## Workflow Connections

```
Schedule Trigger 
    ↓
Get Many Events 
    ↓
Code in JavaScript 
    ↓
Send a Message
```

Each node passes its output to the next node in sequential order, forming a linear processing pipeline.

---

## Email Format

### Structure
The generated email follows this HTML structure:

- **Header:** "📅 Good Morning !"
- **Introduction:** Greeting message
- **Separator:** Horizontal line
- **Event Cards:** One card per event containing:
  - Event title (📌 icon)
  - Time in 12-hour format (⏰ icon)
  - Location (📍 icon)
  - Description (📝 icon)
- **Empty State:** "No events for today!" (if calendar is empty)
- **Separator:** Horizontal line
- **Footer:** Automation notification with timestamp

### Styling
- **Font:** Arial, sans-serif
- **Max Width:** 600px
- **Primary Color:** #4285f4 (Google Blue)
- **Event Background:** #f0f8ff (Light Blue)
- **Event Border:** 4px solid left border in primary color

---

## Execution Configuration

| Setting | Value |
|---------|-------|
| Execution Order | v1 |
| Binary Mode | Separate |
| Time Saved Mode | Fixed |
| Timezone | Asia/Kolkata |
| Caller Policy | Workflows from Same Owner |
| Execution Timeout | Unlimited (-1) |
| Available in MCP | No |

---

## Data Flow

1. **Trigger:** Schedule trigger fires at 9:09 AM daily
2. **Retrieval:** Calendar node fetches events from xxx@gmail.com for the next 24 hours
3. **Transformation:** JavaScript code processes event data and generates HTML
4. **Delivery:** Gmail node sends formatted email to user
5. **Completion:** Workflow execution concludes

---

## Credentials & Integrations

### Google Calendar
- **Account:** Google Calendar account 3
- **Email:** xxx@gmail.com
- **Authentication:** OAuth2

### Gmail
- **Account:** Gmail account 3
- **Email:** xxx@gmail.com
- **Authentication:** OAuth2

---

## Activation Status

**Current Status:** Inactive  
To activate this workflow, toggle the active switch in the n8n interface. Once activated, the schedule trigger will execute daily at 9:09 AM.

---

## Usage Notes

- The workflow uses the user's local timezone (Asia/Kolkata) for all time conversions
- Events are sorted chronologically by start time for easy reading
- The email subject dynamically includes the current date
- The workflow handles edge cases such as missing event descriptions or locations
- All-day events are displayed with "All Day" notation instead of specific times

---

## Technical Specifications

- **Node Count:** 4
- **Total Connections:** 3
- **Webhook Support:** Yes (Gmail node)
- **Instance ID:** 63a97d52b8b699f13f18b2eb934b18652e3648610e19e0fd63930985bbeb587c
- **Template Setup:** Completed

---

## Maintenance & Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| No events received | Verify calendar permissions in Google Calendar OAuth2 settings |
| Email not sent | Check Gmail OAuth2 credentials and ensure account is not restricted |
| Incorrect time format | Verify timezone is set to Asia/Kolkata in code node |
| Missing event details | Ensure events have all fields populated in Google Calendar |

### Monitoring
Enable execution history logging to track workflow performance and troubleshoot failures. Review email delivery logs in the Send a Message node for delivery status.

---

## Version History

- **Last Updated:** March 5, 2026
- **Version ID:** 9c7d8ac9-2e97-40d5-bd29-43232ea315af
- **Template Credentials Setup:** Completed

---

**Document Prepared For:** Professional Workflow Documentation  
**Created:** March 2026  
**Classification:** Internal Use