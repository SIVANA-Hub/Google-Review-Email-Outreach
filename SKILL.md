---
name: google-review-outreach
description: >
  Sends personalized Google review request emails to customers after their rental or service
  is complete. Customer data can come from a CSV, Excel file, Google Sheet, or simply typed
  in the chat — no spreadsheet required. Use this skill whenever the user says things like:
  "send review request", "ask clients for a review", "run google review outreach", "request
  a review from today's customers", "follow up with rentals", or "send the review link to
  [customer name]". Always use this skill for any Google review or post-booking follow-up
  task at Potomac Boat Rentals.
---

# Google Review Outreach

Sends a warm, personalized email to customers after their rental asking them to leave a
Google review.

**Before doing anything else, read `assets/client.json`** — it contains the business name,
owner signature, phone, website, Google review link, and email template. Use these values
everywhere instead of hardcoding anything. This makes the skill reusable across different
clients by simply swapping the client.json file.

---

## Prerequisites

Only one thing is required to run this skill:

- **Gmail is connected** in Cowork

A spreadsheet or CSV is helpful for bulk sends but not required — customer data can also come from a Google Sheet or typed directly in chat.

---

## Workflow

### Step 1 — Get the customer data

The skill accepts customer data in three ways — use whichever the user provides:

- **Uploaded file** (CSV or Excel): read it directly
- **Google Sheet**: use the Google Drive MCP to read it
- **Chat prompt**: if the user types a name, email, and rental type directly (e.g. "send a
  review to Maria Lopez, kayak rental, maria@email.com"), treat that as a single-row dataset

If working from a file, the expected columns are:

| Column | Required | Description |
|--------|----------|-------------|
| `Name` | Yes | Customer first and last name |
| `Email` | Yes | Customer email address |
| `Rental Type` | No | e.g. "Pontoon Boat", "Kayak", "Paddleboard" |
| `Orders` | No | If present, skip rows where Orders = 0 |
| `Review Sent` | No | Skip rows already marked "Yes" |

### Step 2 — Filter eligible customers

Skip any row where:
- Email is blank
- `Review Sent` is already "Yes"
- `Orders` column exists and equals 0

### Step 3 — Send email via Gmail Compose + Chrome

Use Gmail Compose in Chrome with DOM injection. This is the confirmed working method that
correctly renders the "Leave a Review" CTA as a hyperlink.

**Step 3a — Open Gmail Compose:**
Click the Compose button in Gmail (tab must be open and signed in).

**Step 3b — Fill fields and inject HTML body via JavaScript:**

```javascript
// Fill To field
const toInput = document.querySelector('input[name="to"], [aria-label="To recipients"]');
toInput.focus();
toInput.value = 'customer@email.com';
toInput.dispatchEvent(new Event('input', {bubbles: true}));
toInput.dispatchEvent(new KeyboardEvent('keydown', {key: 'Tab', keyCode: 9, bubbles: true}));

// Fill Subject
const subjectInput = document.querySelector('input[name="subjectbox"], [aria-label="Subject"]');
subjectInput.focus();
subjectInput.value = client.email.subject;
subjectInput.dispatchEvent(new Event('input', {bubbles: true}));

// Build body using DOM manipulation (required — Gmail blocks innerHTML via Trusted Types)
const body = document.querySelector('[contenteditable="true"][g_editable="true"]');
while (body.firstChild) body.removeChild(body.firstChild);

const mk = (tag, text) => {
    const el = document.createElement(tag);
    if (text) el.appendChild(document.createTextNode(text));
    return el;
};

body.appendChild(mk('p', 'Hi [first name],'));
body.appendChild(mk('p', 'Thanks so much for choosing [client.name]! We hope your [rental type] rental was a blast and that you got to enjoy everything the Potomac has to offer.'));
body.appendChild(mk('p', "If you have a spare moment, we'd really appreciate it if you could leave us a quick Google review. It means the world to a small business like ours and helps other boaters find us:"));

const ctaP = document.createElement('p');
ctaP.appendChild(document.createTextNode('👉 '));
const a = document.createElement('a');
a.href = '[review.google_review_link]';
a.appendChild(document.createTextNode('Leave a Review'));
ctaP.appendChild(a);
body.appendChild(ctaP);

body.appendChild(mk('p', 'Thanks again, hope to see you back on the water soon!'));

const sigP = document.createElement('p');
sigP.appendChild(document.createTextNode('[client.founder.signature]'));
sigP.appendChild(document.createElement('br'));
sigP.appendChild(document.createTextNode('📞 [client.contact_phone]'));
sigP.appendChild(document.createElement('br'));
sigP.appendChild(document.createTextNode('🌐 [client.website]'));
body.appendChild(sigP);

body.dispatchEvent(new InputEvent('input', {bubbles: true}));
```

**Step 3c — Click Send:**

```javascript
const sendBtn = Array.from(document.querySelectorAll('button,[role="button"]'))
    .find(b => b.getAttribute('data-tooltip')?.includes('Ctrl-Enter') || b.textContent.trim() === 'Send');
if (sendBtn) sendBtn.click();
```

**Step 3d — Verify:**
After clicking Send, the compose window closes and the Drafts count returns to 0.
Check the Sent folder to confirm delivery.

**Email rules (always enforce):**
- No em dashes
- "Leave a Review" must always be a hyperlink — never a raw URL
- The CTA is its own prominent line, not buried in a paragraph
- No "good or bad feedback", no "takes less than a minute", no opt-out framing
- If rental type is unknown, use "recent rental"

### Step 4 — Mark as sent

If working from a spreadsheet, update the `Review Sent` column to `"Yes"` after sending.
If it's a local file, save the updated version to the outputs folder.
If it's a Google Sheet, write back via the Drive MCP.

If the user typed the customer directly in chat, skip this step.

### Step 5 — Report back

Tell the user:
- How many emails were sent
- Any rows skipped and why (already sent, missing email, 0 orders)

---

## Handling edge cases

- **Missing email**: skip the row, flag it to the user
- **Missing rental type**: use "recent rental" in the email body
- **Single customer from chat**: no spreadsheet needed, send directly, no marking required
- **Already marked "Yes"**: skip silently

---

## Tone notes

Read `client.tone` from client.json and match it. Always use first names, keep it genuine,
and never let the message feel like a mass marketing blast.
