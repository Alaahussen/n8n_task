# Smart Email Assistant — Self-hostable n8n Automation

**Project:** Smart Email Assistant (Self-hostable)

**Goal:** Design and implement a self-hostable n8n automation that reads inbound emails, classifies their intent using Ollama (Llama 3.2:1b), sends template-based or AI-assisted responses, schedules meetings automatically via Google Calendar & Google Meet, sends mail via Gmail, logs audit records in Google Sheets, and posts traces to Slack. 

---

## Table of Contents

1. Overview
2. Architecture
3. Requirements
4. Credentials & Permissions Checklist
5. Install & Setup (Step-by-step)

   * Host & OS
   * n8n installation
   * Ollama installation & model
   * Google Workspace (Gmail, Calendar, Sheets) setup
   * Microsoft 365 / IMAP alternative
   * Slack setup
6. Environment variables (.env example)
7. n8n Workflow Design

   * Main workflow (Inbound Email -> Classification -> Actions)
   * Sub-workflows (Meeting creation, Logging, Manual review)
8. Prompt & Classification

   * Ollama classifier prompt (example)
   * Confidence threshold and manual review
9. Template examples and test emails
10. Logs & Auditing (Google Sheets schema)
11. Troubleshooting & FAQ
12. live demo

---

## 1. Overview

This project provides a reproducible, self-hostable automation that:

* Receives inbound email via Gmail (recommended) 
* Classifies each message into one of: `GENERAL_INQUIRY`, `SUPPORT_REQUEST`, `MEETING_REQUEST`, `SPAM`, `OUT_OF_OFFICE` using Ollama running a local Llama 3.2:1b model.
* Sends automated template-based responses for supported types (GENERAL_INQUIRY, SUPPORT_REQUEST, MEETING_REQUEST) or flags for manual review depending on confidence.
* Creates ICS invites and schedules meetings in Google Calendar (with Google Meet links) when `MEETING_REQUEST` is received and accepted.
* Logs every automated action and manual decision to a Google Sheet for audit.
* Posts a short trace to Slack channels.

---

## 2. Architecture

```
Inbound Email (Gmail webhook)
        |
        v
     n8n inbox workflow
        |
   fetch email content
        |
  -> Ollama classifier (local Llama 3.2:1b)
        |                \
        |                 -> If SPAM or OUT_OF_OFFICE: Log & stop (no reply)
        |
  Intent (with confidence)
   /    |    \
  /     |     \
Reply?  |   MEETING_REQUEST -> Create ICS + Google Calendar event + Meet link
(Templates or AI-assisted) |     |-> Send Gmail reply or Draft
  |     |                   |
  v     v                   v
Send Gmail   Log to Google Sheet   Post trace to Slack

Manual review queue if confidence < 0.7
```

---

## 3. Requirements

* A server (local machine) with Docker support or ability to run n8n and Ollama.
* n8n (self-hosted) — community edition is sufficient.
* Ollama installed locally (to host Llama 3.2:1b). Ensure the model fits RAM/cpu — 1B model is small and suitable for modest servers.
* Google Workspace account (recommended) or ability to create OAuth credentials for Gmail, Calendar, and Sheets. 
* Slack workspace with ability to create an app and grant scopes.

---

## 4. Credentials & Permissions Checklist

Create and store these credentials securely (use a secrets manager or n8n credentials):

### Google (OAuth 2.0)

* Project in Google Cloud Console
* OAuth consent screen (External or Internal) — add test users if `External`
* OAuth Client ID & Client Secret
* Scopes to request:

  * `https://www.googleapis.com/auth/gmail.send` (send messages)
  * `https://www.googleapis.com/auth/gmail.readonly` or `https://www.googleapis.com/auth/gmail.modify` (read & mark emails)
  * `https://www.googleapis.com/auth/calendar` (create/update events)
  * `https://www.googleapis.com/auth/spreadsheets` (write logs)

> **Note:** For production with many external users you must verify the OAuth consent screen which may take time. For small-scale testing, add accounts as test users.

### Slack

* Create Slack App in your workspace
* Scopes required (bot token):

  * `chat:write` (post messages)
  * `channels:read` or `conversations.list` (to list channels)
  * `channels:join` (if posting into channels your app must join)
  * `users:read` (optional)
* Install app to workspace and get Bot OAuth token.

### Ollama

* Local installation; no remote API key required. Ensure `ollama` CLI is accessible to the n8n host or run Ollama in a container and expose an API port if needed.
* 
---

## 5. Install & Setup (Step-by-step)

### 5.1 n8n Installation

### 5.3 Ollama Installation & Model

* Install Ollama using official instructions from the Ollama website for your OS.
* Pull the model (example):

```bash
# Example (replace with exact model name if different)
ollama pull llama-3.2-1b
```

* Test locally:

```bash
ollama run llama-3.2-1b --prompt "Hello" --json
```

**How n8n connects to Ollama:**

* Option A: Run Ollama locally and use `ollama serve` (if available) to expose a local HTTP endpoint that n8n can call.
* Option B: Execute the Ollama CLI from n8n via an `Execute Command` node (less recommended for concurrency).

> Important: Keep Ollama accessible only from the host or via secure network.

### 5.4 Google Workspace Setup (Gmail, Calendar, Sheets)

1. Create a Google Cloud Project.
2. Enable APIs: Gmail API, Google Calendar API, Google Sheets API, Google Drive API (optional).
3. Configure OAuth consent screen (Internal for same GSuite org or External for test users)
4. Create OAuth 2.0 Client ID (Type: Web application) and add authorized redirect URIs to include your n8n host, e.g. `https://your-n8n-host/auth/google/callback`.
5. In n8n, create Google OAuth credentials using the Client ID and Client Secret. Test authentication and select scopes.
6. For Gmail “send as” functionality, create the Gmail credential in n8n and test sending.

### 5.6 Slack Setup

* Create a Slack App in the workspace, add a bot user, and grant scopes (`chat:write`, `channels:read`).
* Install to workspace and copy Bot token to n8n credentials.

---

## 6. Environment variables (.env example)

Store secrets in n8n credentials or an env file that Docker uses. Example `.env` snippet:

```
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=changeme
OLLAMA_HOST=127.0.0.1
OLLAMA_PORT=11434
```
---

## 7. n8n Workflow Design

### 7.1 Main Workflow (Inbound Email -> Classification -> Action)

Nodes (high-level):

1. **Trigger**: Gmail Trigger (new email) 
2. **Get Email**: Gmail node to fetch full message (headers, body, attachments)
3. **Preprocess**: Transform text, strip signatures, extract sender, subject, dates
4. **Classifier**: HTTP Request to Ollama API or Execute Command to run local model with prompt
5. **Decision**: Switch node based on `intent` returned and `confidence`
6. **Actions**:

   * `SPAM` & `OUT_OF_OFFICE`: Log only. 
   * `GENERAL_INQUIRY` or `SUPPORT_REQUEST`: Use a Templates node or call an AI assistant to produce a reply. Then `Send Email (Gmail)`.
   * `MEETING_REQUEST`: Create ICS + Google Calendar event, attach ICS to response, send email reply.
7. **Logging**: Append a row to Google Sheet with event details, classification, confidence, timestamps, messageId, actionTaken, and actor.
8. **Slack Notification**: Post a short summary to a channel.
9. **Manual Review Queue**: If `confidence < 0.7`, create a task in a `manual_review` sheet or queue and optionally notify a human.

### 7.2 Meeting creation details

* Generate ICS as `text/calendar` binary with timezone-aware DTSTART/DTEND (use `TZID=Africa/Cairo`), include `UID`, `DTSTAMP`, `ORGANIZER`, `ATTENDEE`.
* Use Google Calendar node to create an event and request conferenceData for Google Meet.

**Important timezone note:** Always write DTSTART/DTEND with `TZID=Africa/Cairo` and include a `VTIMEZONE` block to prevent Google Calendar from interpreting UTC times incorrectly.

---

## 8. Prompt & Classification

### Ollama classifier prompt (example)

```
You are an email intent classifier. Analyze the email content and classify it into one of these categories:

1. GENERAL_INQUIRY - General questions or information requests
2. SUPPORT_REQUEST - Technical support or help needed
3. MEETING_REQUEST - Request to schedule a meeting or appointment
4. SPAM - Marketing, promotional, or spam content (NO automated response will be sent)
5. OUT_OF_OFFICE - Auto-reply or out-of-office messages (NO automated response will be sent)

Provide your response in JSON format with the following structure:
{
  "intent": "CATEGORY_NAME",
  "confidence": 0.95,
  "reasoning": "Brief explanation of why this classification was chosen"
}

The confidence score should be between 0 and 1. If confidence is below 0.7, it should be flagged for manual review.

IMPORTANT: SPAM and OUT_OF_OFFICE emails should be classified but will NOT receive automated responses.
```

**Calling Ollama:** send the prompt + email text as input. Parse the model's JSON output and validate. If the model returns non-JSON, fall back to regex extraction or flag for manual review.

### Confidence logic

* `confidence >= 0.85`: automated action allowed
* `0.7 <= confidence < 0.85`: automated action allowed but log with `reviewRecommended: true`
* `confidence < 0.7`: do not act automatically — add to manual review queue

---

## 9. Template examples and test emails

Create a folder called `/templates` containing response templates for each intent. Example templates:

* `general_inquiry.md` — polite acknowledgement and next steps
* `support_request.md` — request more info and provide troubleshooting steps
* `meeting_request.md` — propose times / accept + create calendar event
* `spam.txt` — optionally automated label/archival rule but no reply
* `out_of_office.txt` — label and log only

### Example templates (short)

**templates/general_inquiry.md**:

```
Subject: Re: {{subject}}

Hello {{sender_name}},

Thanks for reaching out about {{topic}}. I'd be happy to help. Could you please provide:
- More detail about X
- Any files or screenshots

Once I have that, I'll get back with recommended next steps.

Best,
Alaa Hussien
```

**templates/meeting_request.md**:

```
Subject: Re: Meeting Request — {{subject}}

Hello {{sender_name}},

Thanks for your request. I can confirm the meeting on {{start_time}} (Cairo time). I have scheduled a Google Meet and attached the invite.

See you then,
Alaa
```

### Test emails (examples)

Include a `/test-emails` folder with these samples (use for training & QA):

* `meeting-request-1.eml` — from Sara Ahmed proposing Monday Nov 17 12:00-13:00 Cairo
* `spam-marketing-1.eml` — promotional email promising sales boost
* `general-inquiry-1.eml` — Omar Khaled asking about image classification services
* `support-request-1.eml` — a user reporting an error and logs attached
* `out_of_office-1.eml` — automatic OOO reply

(You can use the sample emails we prepared earlier.)

---

## 10. Logs & Auditing (Google Sheets schema)

Create a Google Sheet with a tab `audit_logs` and columns:

* `recevid date` (ISO 8601)
* `message_id`
* `from_email`
* `subject`
* `intent`
* `confidence`
* `action_taken` (e.g., `replied`, `scheduled_event`, `logged_only`)
* `slack_ts` (if posted)
* `notes`

Use `Append` operation in n8n to add rows after each main action.

---

## 11. Troubleshooting & FAQ

**Q: Classifier returns invalid JSON**

* A: Implement a safe JSON extraction step: strip non-JSON before parsing, or use a regex to find the first `{ ... }`. If still invalid, send the email to manual review and log the failure.

**Q: Google OAuth consent prevents automated use**

* A: For testing, add the sending account as a test user. For production, you may need to submit the app for verification (may take weeks). Consider using domain-wide delegation in Workspace for internal apps.

**Q: Calendars showing wrong timezones (UTC vs Cairo)**

* A: Ensure ICS uses `DTSTART;TZID=Africa/Cairo:YYYYMMDDTHHMMSS` and include a `VTIMEZONE` block. When creating Google Calendar events through the API, set `start.timeZone` and `end.timeZone` to `Africa/Cairo`.

**Q: Ollama is slow or memory-heavy**

* A: Use the 1B Llama model as planned. Ensure your host has enough RAM. Consider running Ollama and n8n on separate machines.

**Q: Spam emails still replied to**

* A: Verify your classifier prompt and thresholds. Ensure `SPAM` and `OUT_OF_OFFICE` branches are explicitly configured to `logOnly` and skip `Send Email` node.

---

## 12. Live Demo


---



