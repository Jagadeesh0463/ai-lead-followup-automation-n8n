# AI Lead Follow-up Automation using n8n

An AI-powered lead management workflow built with **n8n**, **Groq LLM**, **Gmail**, and **Google Sheets**. It automatically captures leads, stores them in a lightweight CRM, sends AI-generated acknowledgment emails, performs timed follow-ups, checks lead status dynamically, and escalates inactive leads.

```text
Trigger → AI → CRM → Email → Wait → Status Check → Follow-up → Escalation
```

## Table of Contents

- [Problem Statement](#problem-statement)
- [Features](#features)
- [Workflow Architecture](#workflow-architecture)
- [Technologies Used](#technologies-used)
- [How It Works](#how-it-works)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Running the Workflow](#running-the-workflow)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Known Limitations](#known-limitations)
- [Security](#security)
- [Roadmap](#roadmap)
- [Author](#author)

## Problem Statement

Leads are often not followed up quickly enough. Delayed responses reduce meeting bookings, conversion rates, sales opportunities, and customer engagement. This workflow solves that by automating the full sequence: **lead capture → acknowledgment → follow-up → escalation**.

Suitable for sales teams, agencies, freelancers, consultants, real estate businesses, coaching businesses, and customer onboarding systems.

## Features

- Lead capture via webhook (POST endpoint)
- AI-generated acknowledgment and follow-up emails (Groq LLM)
- Lead tracking in Google Sheets (lightweight CRM)
- Timed follow-up sequences with dynamic status checks
- Escalation alerts for inactive leads
- End-to-end automation with no manual steps after lead capture

> **Demo configuration:** The exported workflow uses **1-minute** wait times for faster testing. For production, change the Wait nodes to **1 hour** (follow-up) and **24 hours** (escalation).

## Workflow Architecture

```text
Lead Capture Webhook
        ↓
Set Lead Fields
        ↓
Store Lead in CRM
        ↓
Generate Acknowledgment Email
        ↓
Send Acknowledgment Email
        ↓
Wait (1 Minute Demo / 1 Hour Production)
        ↓
Check Lead Status
        ↓
Did Customer Respond?
     ├── YES → END
     └── NO
            ↓
     Generate Follow-up Email
            ↓
     Send Follow-up Reminder
            ↓
     Wait (1 Minute Demo / 24 Hours Production)
            ↓
     Check Lead Status After 24 Hours
            ↓
     Did Customer Respond?
          ├── YES → END
          └── NO
                 ↓
          Send Escalation Alert
```

![Workflow](03-ai-lead-followup-automation-n8n-workflow.png)

## Technologies Used

| Tool | Purpose |
|------|---------|
| n8n | Workflow automation |
| Groq LLM (`groq/compound-mini`) | AI email generation |
| Gmail API | Sending emails |
| Google Sheets | Lead tracking CRM |
| Webhook | Lead capture endpoint |

## How It Works

**Lead capture.** The webhook receives a POST request with the lead's `name`, `email`, and `service`. The lead is stored in Google Sheets with `status = new` and a timestamp.

**Acknowledgment.** Groq LLM generates a short (under 80 words), professional acknowledgment email, which is sent immediately via Gmail.

**Follow-up.** After the wait period, the workflow re-reads the lead's row from Google Sheets. If `status != responded`, an AI-generated follow-up reminder is sent.

**Escalation.** After a second wait period, the status is checked again. If the lead still hasn't responded, an internal escalation alert is emailed with the lead's name, email, requested service, and submission timestamp.

**Lead states.** The `status` column supports `new`, `responded`, and `closed`. Reminders and escalations only fire while `status != responded`.

## Prerequisites

- Docker Desktop (or Node.js 18+)
- n8n (Docker image or npm install)
- Google account
- Groq API key
- Google Sheets OAuth credentials
- Gmail OAuth credentials

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/Jagadeesh0463/03-ai-lead-followup-automation-n8n.git
cd 03-ai-lead-followup-automation-n8n
```

### 2. Create the Google Sheet

Create a sheet named **Lead Tracker** with these columns: `name`, `email`, `service`, `status`, `created_at`.

Optional additional columns: `follow_up_sent`, `escalated`, `last_contacted`.

### 3. Start n8n

With Docker (recommended):

```bash
docker start n8n
# or with Docker Compose:
docker compose up -d
```

With npm:

```bash
n8n start
```

Then open n8n at `http://localhost:5678`.

### 4. Import the workflow

In n8n, go to **Workflows → Import** and select `03-ai-lead-followup-automation-n8n.json`.

### 5. Configure credentials

This repository does **not** include OAuth credentials or API keys. Connect your own **Gmail OAuth**, **Google Sheets OAuth**, and **Groq API** credentials after import.

### 6. Replace placeholders

Edit the imported nodes (or the JSON before import) and replace:

| Placeholder | Location | Replace with |
|-------------|----------|--------------|
| `{{YOUR_EMAIL}}` | Send Escalation Alert node | Your escalation recipient address |
| `{{DOCUMENTID}}` | All three Google Sheets nodes | Your Google Sheet document ID |
| `{{YOUR_NAME}}` | Both email generation prompts | Your sign-off name |

The webhook path is already configured as `lead-capture` — no changes needed. Credential references are re-linked automatically when you connect your own accounts.

### 7. Activate the workflow

Turn ON the **Active** toggle at the top-right of the workflow. The Production Webhook URL is generated automatically by n8n once the workflow is activated — it does not exist while the workflow is inactive.

## Running the Workflow

Send a **POST** request to the Production Webhook URL:

```text
http://localhost:5678/webhook/lead-capture
```

For testing **before** activation, use the Test URL (only active while **Execute Workflow** is waiting in the editor):

```text
http://localhost:5678/webhook-test/lead-capture
```

> **The webhook accepts POST only.** Opening the URL in a browser sends a GET request and will not trigger the workflow — use curl, Postman, or a form that submits via POST.

To expose the webhook publicly (e.g., for external forms), use a tunnel like ngrok:

```bash
ngrok http 5678
```

Then use `https://YOUR-NGROK-URL/webhook/lead-capture`. If you restart ngrok, a new public URL is generated — update any webhook integrations, forms, or shared links, as old URLs will return a 404.

### Example request

```bash
curl -X POST http://localhost:5678/webhook/lead-capture \
  -H "Content-Type: application/json" \
  -d '{"name":"Bhagya","email":"example@gmail.com","service":"AI Automation"}'
```

Verify success by checking:

- **Executions** page in n8n — a new execution appears and runs green
- **Google Sheets** — a new row is added with the lead data
- **Inbox** — the acknowledgment email is delivered to the lead's address
- **Workflow** — execution enters the Wait node before the follow-up check

## Testing

**Scenario 1 — customer does not respond.** Send a test lead and wait through both Wait periods. Expected: acknowledgment email, follow-up reminder, and escalation alert are all sent.

**Scenario 2 — customer responds.** The workflow checks the `status` column in Google Sheets to decide whether to continue. To simulate a customer reply, manually change the lead's row to `status = responded`. The workflow then stops — no reminder or escalation emails are sent.

## Troubleshooting

### 404 — Webhook not registered

Possible causes:

- Workflow is not **Active**
- Using the Test URL without clicking **Execute Workflow** first
- Using an outdated ngrok URL (free ngrok URLs change on every restart)
- Sending a GET request (e.g., opening the URL in a browser) instead of POST

Solutions: activate the workflow, use the Production URL (`/webhook/lead-capture`) for normal operation, use the Test URL only while Execute Workflow is waiting, update ngrok URLs after every restart, and test with curl or Postman using POST.

### Workflow runs but no email is sent

Verify Gmail OAuth credentials are connected and authorized, check the Groq API key is valid and has quota, and inspect the failing node's error output under **Executions**.

### Lead not appearing in Google Sheets

Confirm the document ID placeholder was replaced and the sheet has the required columns: `name`, `email`, `service`, `status`, `created_at`.

## Known Limitations

The workflow currently requires manually updating `status = responded` in Google Sheets. Without this update, all leads eventually escalate. Automatic Gmail reply detection is planned (see [Roadmap](#roadmap)).

## Security

Never commit Gmail credentials, Google Sheets credentials, Groq API keys, or personal email addresses. This repository uses placeholders for all personal values, and a `.gitignore` is included to protect secrets.

## Roadmap

- Gmail reply detection (automatic `responded` status)
- Slack alerts
- Airtable / HubSpot CRM integration
- Analytics dashboard and lead scoring
- WhatsApp notifications
- Retry mechanisms and customer segmentation

## Author

**Jagadeesh S**

Built with n8n + Groq + Gmail + Google Sheets.
