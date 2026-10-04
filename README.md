# Gmail AI Summarizer → Google Sheets (n8n Workflow)

An n8n automation that watches your Gmail inbox, summarizes every new email with **Google Gemini**, and logs a clean, readable row to **Google Sheets** — complete with a one-click link straight back into the original email in Gmail.

No Google Drive, no attachment uploads, no storage bloat — just a fast, lightweight email log.

---

## What it does

Whenever a new email lands in your inbox, the workflow:

1. Fetches the **full email** (headers + body) directly via the Gmail API
2. Extracts the **sender, subject, date, and readable body**
3. Pulls out **genuine content links** the sender actually wrote — filtering out tracking pixels, unsubscribe links, social-share icons, CSS/image assets, and platform navigation chrome (e.g. LinkedIn's header badge links)
4. Sends the email to **Google Gemini** for a concise, factual summary (no invented details, deadlines and action items flagged when present)
5. Notes whether the email has **attachments** and lists their filenames (without downloading or storing them anywhere)
6. Appends **one row per email** to a Google Sheet, with a working **"Open Gmail"** link that deep-links directly into that exact message

Each email is processed independently, one at a time, so a burst of new mail doesn't get mixed together or dropped.

---

## Sheet columns

| Column | Content |
|---|---|
| **Date** | Date the email was received |
| **Sender** | Sender's name/address |
| **Subject** | Email subject (wraps every 8 words) |
| **Summary** | AI-generated summary (wraps every 8 words) |
| **Links** | Genuine content links found in the email body |
| **Attachments** | `None`, or `Yes: filename1, filename2...` |
| **Open Gmail** | Clickable link that opens the exact email in Gmail |
| **Message ID** | Gmail's internal message ID (used to prevent duplicate rows) |
| **Processed At** | Timestamp the row was written |

---

## Architecture

```
Gmail Trigger (polls inbox every minute)
  ↓
Loop Over Items (processes one email at a time)
  ↓
Get Full Email (direct Gmail REST API call — guarantees full headers/body)
  ↓
Extract Email Data (parses sender, subject, date, body, attachment names)
  ↓
Extract Links (regex + filtering for genuine sender-written links)
  ↓
Build AI Prompt
  ↓
AI Summarize (Google Gemini)
  ↓
Parse AI Response (safe JSON parsing, graceful fallback on failure)
  ↓
Build Sheet Row (formats date, wraps text, builds the Gmail deep link)
  ↓
Append to Google Sheet
  ↓
back to Loop Over Items (next email)
```

---

## Requirements

- An [n8n](https://n8n.io/) instance (self-hosted or cloud)
- A Gmail account
- A Google Sheet to log into
- A free [Google Gemini API key](https://aistudio.google.com/apikey)

---

## Setup

### 1. Import the workflow

In n8n: **Workflows → Import from File** → select `gmail_ai_google_sheets.json`.

### 2. Create credentials

| Credential | Type | Used by |
|---|---|---|
| Gmail OAuth2 | Gmail OAuth2 API | `Gmail Trigger` |
| Gmail OAuth2 (same one) | Predefined Credential Type → Gmail | `Get Full Email` |
| Google Sheets OAuth2 | Google Sheets OAuth2 API | `Append to Google Sheet` |
| Gemini API Key | **Query Auth** — Name: `key`, Value: your Gemini API key | `AI Summarize` |

No credentials or API keys are embedded in the workflow file — attach each one after import.

### 3. Configure the Google Sheet

- Open the `Append to Google Sheet` node
- Set **Document ID** to your spreadsheet's ID
- Set **Sheet Name** to match your tab (default: `Sheet1`)
- Make sure your sheet's header row matches the columns listed above

### 4. (Optional) Make the sheet easier to read

This is a one-time formatting pass, not something the workflow re-applies on every run:

- Freeze the header row: **View → Freeze → 1 row**
- Bold + color the header row
- Turn on text wrapping for Subject, Summary, Links, Attachments columns: **Format → Wrapping → Wrap**
- Add alternating row colors: **Format → Alternating colors**

### 5. Activate the workflow

Toggle the workflow to **Active** in the top-right of the n8n canvas. Note: manually clicking "Test workflow" only previews the single most recent email — it does *not* reflect real polling behavior. Real continuous checking (every 1 minute, catching every genuinely new email) only happens once the workflow is active.

---

## Customization

- **Poll frequency**: edit `Gmail Trigger` → Poll Times
- **Gemini model**: edit the `url` in `AI Summarize` (default: `gemini-2.5-flash`)
- **Summary/subject line-wrap length**: edit the `8` in `wrapEveryNWords(text, 8)` inside `Build Sheet Row`
- **Link filtering rules**: edit the `noisePatterns` array in `Extract Links`
- **Gmail account slot**: if you're signed into multiple Google accounts and `/u/0/` isn't this account, change the index in the Gmail URL inside `Build Sheet Row`

---

## Error handling

- Empty bodies, missing links, and missing attachments never break the workflow — those fields are simply left blank/`None`
- If the Gemini call fails, the actual error message is written into the Summary column instead of silently failing, so you can see why
- If the full-email fetch fails for a single message, the row still gets written (using trigger data as a fallback) instead of losing that email entirely
- `Message ID` is used as a de-duplication key on append, so retries don't create duplicate rows

---

## License

MIT — use, modify, and share freely.
