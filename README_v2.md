# Gym Automation v2 (n8n) - Full Suite

A complete gym management automation built in n8n. It handles member signup, payments, renewals, expiry tracking, WhatsApp messaging, weekly reports, backups and an owner dashboard. Every gym keeps its own separate data.

**File:** `gym-automation-n8n-v2.json` (import via **Workflows → Import from File**)

## What's New in v2

| # | Feature | How it works |
|---|---------|--------------|
| 1 | **Auto Expired status** | Every day at 9 AM, members whose expiry date has passed are marked `Expired` in the sheet. |
| 2 | **AI motivational messages** | Google Gemini writes a short Roman Urdu motivational line for welcome messages and renewal reminders. A default line is used if the AI call fails. |
| 3 | **Weekly report by email** | The Monday report goes to the owner on WhatsApp and as an HTML email via Gmail. |
| 4 | **Payment confirmation tracking** | Signup records `Paid` or `Pending`. A Payment Form marks pending payments as paid. Every payment is logged in the `Payments` tab. |
| 5 | **Duplicate member detection** | A signup with an already-registered phone number is blocked, and the person gets a WhatsApp notice. |
| 6 | **WhatsApp error handling** | Each message is retried 3 times. If it still fails, it is logged in `Failed Messages` and the owner is emailed. |
| 7 | **Renewal updates expiry** | A Renewal Form extends the expiry from the current expiry date (or from today if already expired), reactivates the member and logs the payment. |
| 8 | **Owner dashboard** | A browser page with member counts, monthly collection, expiring members and pending payments. |
| 9 | **Backup** | Every Sunday at 2 AM the Google Sheet is copied to a Drive backup folder. |
| 10 | **Separate data per gym** | One `Gym Config` node holds all gym-specific values. Duplicate the workflow for each gym. |

## Architecture

All 7 triggers pass through a single `Gym Config` node, then a `Route by Flow` switch sends the run to the right flow.

| Flow | Trigger | Main steps |
|------|---------|-----------|
| Signup | Signup Form | Get Members → Build Member Record → duplicate check → Save Member → Log Payment → AI welcome → WhatsApp |
| Renewal | Renewal Form | Get Members → find member → update expiry/plan/status → Log Payment → WhatsApp confirmation |
| Payment | Payment Form | Get Members → find member → set Payment Status `Paid` → Log Payment → WhatsApp confirmation |
| Daily | Daily 9AM | Mark expired members; find members expiring in 3 days or today → AI reminder → WhatsApp |
| Weekly report | Monday 9 AM | Members + Payments → stats → WhatsApp + Email |
| Backup | Sunday 2 AM | Copy sheet to Drive backup folder |
| Dashboard | Webhook (GET) | Key check → Members + Payments → HTML page |

All messages go through one shared `WhatsApp Send` node, so retry and error handling live in one place.

## Google Sheet Structure

Create one Google Sheet per gym with these 3 tabs. Column names must match exactly.

**Members**

| Member ID | Name | Phone | Plan | Fee | Join Date | Expiry | Status | Payment Status |
|---|---|---|---|---|---|---|---|---|

**Payments**

| Payment ID | Member ID | Name | Phone | Amount | Method | Type | Date | Status |
|---|---|---|---|---|---|---|---|---|

**Failed Messages**

| Date | Name | Phone | Flow | Message | Error |
|---|---|---|---|---|---|

Dates are stored as text in `YYYY-MM-DD` format. Do not reformat these columns in Google Sheets.

## Setup

1. **Import** `gym-automation-n8n-v2.json` into n8n.
2. **Create the sheet** with the 3 tabs above (headers only).
3. **Edit the `Gym Config` node:**

   | Field | Meaning |
   |-------|---------|
   | `gymName` | Gym name used in messages and reports |
   | `sheetId` | Google Sheet ID (the long part of the sheet URL) |
   | `ownerPhone` | Owner WhatsApp number, e.g. `923001234567` |
   | `ownerEmail` | Owner email for reports and failure alerts |
   | `backupFolderId` | Google Drive folder ID for backups |
   | `dashboardKey` | Secret key for the dashboard URL |
   | `currency` | Currency label, e.g. `Rs` |
   | `reminderDays` | Days before expiry to send the reminder (default 3) |
   | `tzOffsetHours` | Timezone offset from UTC (Pakistan = 5) |

4. **Select credentials** on the nodes that need them:
   - Google Sheets (all Google Sheets nodes)
   - Google Drive (`Backup Spreadsheet`)
   - Gmail (`Email Weekly Report`, `Email Owner: Failed Message`)
   - WhatsApp Business Cloud (`WhatsApp Send`), and set its **Phone Number ID**
   - Google Gemini API key (`Gemini (Welcome)`, `Gemini (Reminder)`), free from Google AI Studio
5. **Test each flow** (see below), then **activate** the workflow.

## Using the Forms and Dashboard

- **Signup Form:** share with new members. Fields: name, WhatsApp number, plan, fee, payment status, payment method.
- **Renewal Form:** used by members or reception to renew. Needs the registered WhatsApp number.
- **Payment Form:** used by reception when a pending payment is received.
- **Dashboard:** open `https://<your-n8n-domain>/webhook/gym-dashboard?key=<dashboardKey>` in a browser. A wrong or missing key returns `401 Unauthorized`.

Form URLs are shown in each Form Trigger node (use the Production URL after activating).

## Multiple Gyms (Separate Data)

1. Duplicate the workflow in n8n and rename it (e.g. `Gym Automation - Gym B`).
2. Edit only `Gym Config` (new Sheet ID, owner details, backup folder, dashboard key).
3. Select that gym's credentials and WhatsApp Phone Number ID.

Each gym then has its own Sheet, its own backups, its own dashboard key and (optionally) its own WhatsApp number. Nothing is shared between gyms.

## Error Handling

- `WhatsApp Send` retries 3 times with a 3 second wait.
- After the final failure, the item goes to the error output: `Build Failure Record` → row in `Failed Messages` + email to the owner.
- If Gemini is unavailable, the AI nodes continue and a default motivational line is used.

## Notes and Limitations

- **WhatsApp templates:** Meta requires approved message templates for business-initiated messages outside the 24-hour window. In test mode, messages only go to verified numbers. Plan for templates before going live.
- **Gemini model:** the model name (`models/gemini-2.0-flash`) can be changed in the Gemini nodes. You can also swap in an OpenAI or other chat model node.
- **Dashboard security:** the key in the URL is a simple protection, not full authentication. Do not share the link publicly.
- **Phone format:** numbers are normalized automatically (`03001234567` → `923001234567`).
- **Form responses:** forms show the default "submitted" page. Duplicate or not-found cases are reported to the person via WhatsApp, not on the form page.
- **Import check:** the JSON structure and the JavaScript in Code nodes were validated, but the workflow has not been run against live Google, WhatsApp or Gemini accounts. Test with sample data first.

## Testing Checklist

1. Submit the Signup Form → row in `Members` and `Payments`, welcome WhatsApp arrives.
2. Submit the same number again → duplicate message, no new row.
3. Submit the Payment Form for a pending member → status becomes `Paid`.
4. Submit the Renewal Form → expiry extended, status `Active`.
5. Set an old expiry date manually and run `Daily 9AM` manually → status becomes `Expired`.
6. Run `Weekly Report Monday` manually → WhatsApp and email report arrive.
7. Open the dashboard URL with the correct key.
8. Use a wrong WhatsApp Phone Number ID temporarily → row appears in `Failed Messages` and an email arrives.

## Tech Stack

n8n (Form, Schedule, Webhook, Switch, IF, Code, Google Sheets, Google Drive, Gmail, WhatsApp Business Cloud, Google Gemini via LangChain nodes).

## License

MIT
