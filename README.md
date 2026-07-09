# Google Review Outreach Skill

A Claude skill that sends personalized Google review request emails to customers after their rental or service is complete. Built for Potomac Boat Rentals — reusable for any client by swapping `client.json`.

---

## What It Does

- Reads customer data from a CSV, Excel file, or Google Sheet
- Filters to customers who have completed an order and haven't been contacted yet
- Sends a personalized HTML email to each customer via Gmail with a hyperlinked "Leave a Review" CTA
- Reports how many were sent and flags any skipped rows
- SMS support via Twilio (optional — see setup below)

---

## File Structure

```
google-review-outreach/
├── SKILL.md                  # Claude reads this to know how to run the skill
├── assets/
│   └── client.json           # Business config — swap this per client
├── scripts/
│   └── send_sms.py           # Twilio SMS script (optional)
└── references/
    └── twilio_setup.md       # Step-by-step Twilio setup guide
```

---

## Setup

### 1. Connect Gmail in Cowork
The skill sends emails through your connected Gmail account. Make sure Gmail is connected under Cowork settings before running.

### 2. Fill out `client.json`
All business identity, tone, review link, and email template live in `assets/client.json`. The skill reads this file before doing anything — nothing is hardcoded.

Minimum required fields:

```json
{
  "client": {
    "name": "Your Business Name",
    "contact_phone": "703-000-0000",
    "website": "yourbusiness.com",
    "founder": {
      "signature": "Owner Name & the Team Name"
    }
  },
  "review": {
    "google_review_link": "https://search.google.com/local/writereview?placeid=YOUR_PLACE_ID"
  },
  "email": {
    "subject": "Your subject line here",
    "sender_signature": "Name\n📞 Phone\n🌐 website.com",
    "html_template": "<p>Hi {{first_name}},</p>..."
  }
}
```

To find your Google Place ID: go to [Google Place ID Finder](https://developers.google.com/maps/documentation/places/web-service/place-id) and search your business name.

### 3. Prepare your customer spreadsheet
The skill accepts a CSV, Excel file, or Google Sheet with these columns:

| Column | Required | Description |
|--------|----------|-------------|
| `Name` | Yes | Customer first and last name |
| `Email` | Yes | Customer email address |
| `Rental Type` | No | e.g. Pontoon Boat, Kayak, Paddleboard |
| `Orders` | No | If present, rows with 0 orders are skipped |
| `Review Sent` | No | Skill marks this "Yes" after sending |

WooCommerce customer exports work out of the box.

---

## Usage

Trigger the skill by saying things like:

- `"send review requests to today's customers"` + attach a CSV
- `"send a review request to John Smith, pontoon rental, john@email.com"`
- `"run google review outreach"` (if a spreadsheet is already referenced)

The skill accepts customer data in three ways:

1. **Uploaded file** — attach a CSV or Excel export directly in the chat (WooCommerce exports work out of the box)
2. **Google Sheet** — paste or reference a Sheet URL and the skill reads it via Google Drive
3. **Chat prompt** — type the customer details directly, e.g. `"send a review to Maria Lopez, kayak rental, maria@email.com"` — no file needed

Claude will read the file, filter eligible customers, and send each one the personalized HTML email via Gmail.

---

## Email Template

The approved template (stored in `client.json`) looks like this:

> Hi [Name],
>
> Thanks so much for choosing Potomac Boat Rentals! We hope your recent rental was a blast and that you got to enjoy everything the Potomac has to offer.
>
> If you have a spare moment, we'd really appreciate it if you could leave us a quick Google review. It means the world to a small business like ours and helps other boaters find us:
>
> 👉 [Leave a Review](https://search.google.com/local/writereview?placeid=...)
>
> Thanks again, hope to see you back on the water soon!
>
> Patrick & the Potomac Boat Rentals Team
> 📞 703-677-1357
> 🌐 potomacboatrentals.com

**Rules enforced:**
- No em dashes
- "Leave a Review" is always a hyperlink — never a raw URL
- The CTA is never buried — it's its own prominent line
- No "good or bad feedback," no "takes less than a minute," no opt-outs

---

## SMS (Optional)

SMS support is built in but requires Twilio setup. The client's phone number (+1 571-837-2574 for Potomac Boat Rentals) is already purchased.

To activate:
1. Get the client's EIN for A2P 10DLC registration
2. Follow `references/twilio_setup.md`
3. Add `twilio_config.json` to the skill root with your Account SID and Auth Token

Until SMS is set up, the skill sends emails only and notes the gap in its report.

---

## Multi-Client Use

To use this skill for a different client:
1. Replace `assets/client.json` with the new client's config
2. Update the Google review link (`review.google_review_link`)
3. Update the email template (`email.html_template`)
4. Everything else — tone, signature, subject, CTA — pulls from the config automatically

---

## Built By

[Sivana Systems](https://sivanasystems.com) — AI automation for small businesses.
