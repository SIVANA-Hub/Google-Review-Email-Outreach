---
name: google-review-outreach
description: >
  Sends personalized Google review requests to Potomac Boat Rentals customers after their rental
  is complete — via both email (Gmail) and SMS (Twilio). Reads client data from a Google Sheet or
  uploaded Excel/CSV file. Use this skill whenever the user says things like: "send review request",
  "ask clients for a review", "run google review outreach", "request a review from today's customers",
  "follow up with rentals", or "send the review link to [customer name]". Also triggers automatically
  when a booking completion is detected. Always use this skill for any Google review or post-booking
  follow-up task at Potomac Boat Rentals.
---

# Google Review Outreach

Sends a warm, personalized email + SMS to customers after their rental asking them to leave a
Google review. Marks each client as "Review Sent" in the spreadsheet so no one gets contacted twice.

**Before doing anything else, read `assets/client.json`** — it contains the business name, owner
signature, phone, website, Google review link, and SMS number. Use these values everywhere in the
email and SMS instead of hardcoding anything. This makes the skill reusable across different clients
by simply swapping the client.json file.

---

## Prerequisites

Before running for the first time, make sure:

1. **Twilio is set up** — see `references/twilio_setup.md` for step-by-step instructions.
2. **Twilio credentials are saved** to `twilio_config.json` in this skill's directory:
   ```json
   {
     "account_sid": "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
     "auth_token": "your_auth_token_here",
     "from_number": "+1XXXXXXXXXX"
   }
   ```
3. **Gmail is connected** in Cowork (used to send/draft emails).
4. **Spreadsheet is ready** — Google Sheet or Excel/CSV with these columns (exact names):

   | Column | Description |
   |--------|-------------|
   | `Name` | Customer first and last name |
   | `Email` | Customer email address |
   | `Phone` | Customer phone number (include country code, e.g. +17031234567) |
   | `Rental Type` | e.g. "Pontoon Boat", "Kayak", "Paddleboard" |
   | `Rental Date` | Date of rental (any readable format) |
   | `Review Sent` | Leave blank — the skill fills in "Yes" after sending |

---

## Workflow

### Step 1 — Get the spreadsheet data

If the user provides an uploaded file (CSV or Excel): read it directly.
If the user mentions a Google Sheet: use the Google Drive MCP to read it.
If the user just says a customer name/phone/email verbally: treat that as a single-row dataset.

### Step 2 — Filter eligible customers

Only process rows where `Review Sent` is blank or "No". Skip anyone already marked "Yes".

### Step 3 — Send email (Gmail compose URL → Chrome send)

Use this two-step approach — it is confirmed working:

**Step 3a — Build the Gmail compose URL:**
Construct a pre-filled Gmail compose URL using the customer's info and email template:
```
https://mail.google.com/mail/?view=cm&fs=1&to={EMAIL}&su={SUBJECT_URL_ENCODED}&body={BODY_URL_ENCODED}
```
URL-encode the subject and body (spaces → +, special chars → %XX). Navigate to this URL in Chrome.

**Step 3b — Fill the To field and send:**
The compose window opens pre-filled with subject and body, but the To field needs to be set manually:
1. Use `mcp__claude-in-chrome__find` to locate the "To recipients" input field
2. Use `mcp__claude-in-chrome__form_input` to set the recipient email address
3. Use JavaScript to press Tab (to confirm the address), then click the Send button:
```javascript
// Confirm address with Tab
const toInput = document.querySelector('[aria-label="To recipients"]');
toInput.dispatchEvent(new KeyboardEvent('keydown', {key: 'Tab', keyCode: 9, bubbles: true}));
await new Promise(r => setTimeout(r, 800));
// Click Send
const sendBtn = Array.from(document.querySelectorAll('button,[role="button"]'))
  .find(b => b.getAttribute('data-tooltip')?.includes('Ctrl-Enter'));
if (sendBtn) sendBtn.click();
```

**Step 3c — Verify by checking Sent folder:**
After clicking Send, navigate to `https://mail.google.com/mail/u/0/#sent` and confirm the
email appears at the top. The compose page does NOT redirect after sending — always verify via
Sent. Do not re-send if it's already in Sent.

Use the email template below, personalizing with values from client.json.

**Subject:** Use `client.email.subject` from client.json

**Always use `htmlBody` (not plain `body`) when creating the draft** so the review link renders
as a proper hyperlink. Do NOT paste the raw URL — link it to clean anchor text instead.

Build the email using these client.json values:
- `client.name` → business name
- `client.founder.signature` → sign-off
- `review.google_review_link` → the review URL (hyperlinked, never pasted raw)
- `client.contact_phone` → phone number
- `client.website` → website

**Rules for this email:**
- No em dashes (use commas or periods instead)
- The review CTA must be the most prominent thing in the email, not buried in a paragraph
- Make the customer feel genuinely excited and valued, not obligated
- Never say "good or bad", "honest feedback", "takes less than a minute", or anything that frames it as optional or low-stakes
- Never give them an out

Template (fill in bracketed values from client.json):
```
Hi [first name],

Thanks so much for choosing [client.name]! We hope your [Rental Type] rental was a blast
and that you got to enjoy everything the Potomac has to offer.

If you have a spare moment, we'd really appreciate it if you could leave us a quick Google
review. It means the world to a small business like ours and helps other boaters find us:

Leave a Review: [review.google_review_link]

Thanks again, hope to see you back on the water soon!

[client.founder.signature]
```

Use plain `body` (not htmlBody) so the email renders cleanly in all clients including Outlook.
The review link should be on its own line, clearly labeled "Leave a Review:" followed by the URL.

### Step 4 — Send SMS via Twilio

Run the SMS script:
```bash
python scripts/send_sms.py \
  --name "[Name]" \
  --phone "[Phone]" \
  --rental "[Rental Type]"
```

The script reads credentials from `twilio_config.json` automatically.

**SMS message sent** (build from client.json):
```
Hi [customer first name]! Thanks for renting with [client.name]. We'd love to hear how your
[Rental Type] rental went — a quick Google review goes a long way! [review.google_review_link]
```

### Step 5 — Mark as sent

After successfully sending both email and SMS, update the `Review Sent` column to `"Yes"` for
that row in the spreadsheet. If the spreadsheet is a Google Sheet, write back via the Drive MCP.
If it's a local file, save the updated version to the outputs folder.

### Step 6 — Report back

Tell the user:
- How many review requests were sent
- Any rows skipped (already sent, missing email/phone)
- Confirmation that emails were sent (not drafted)

---

## Handling edge cases

- **Missing phone number**: skip SMS, send email only. Note it in the report.
- **Missing email**: skip email, send SMS only. Note it in the report.
- **Both missing**: skip the row entirely, flag it to the user.
- **Twilio not configured**: skip SMS for all, send emails only, remind the user to set up Twilio
  (point them to `references/twilio_setup.md`).
- **Single customer named verbally** (e.g. "send review to John, 703-555-1234"): treat as a
  one-row dataset, no spreadsheet needed. Still mark it as sent if possible.

---

## Tone notes

Read `client.tone` from client.json and match it. The tone description in the config tells you
exactly how this business communicates — follow it closely. Always use first names, keep it
genuine, and never let the message feel like a mass marketing blast.
