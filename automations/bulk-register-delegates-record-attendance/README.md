# Bulk Register Delegates and Record Attendance

## Problem

After a private course, the office often gets the delegate list afterwards. Each delegate then has to be found or created under the client's account, registered on the event, marked as attended on every session, and given a Pass or Fail result. Doing this one screen at a time for a whole group is slow and easy to get wrong.

## Automator Solution

Staff fill in a simple web form (`form.html`, included): the event ID once, then one row per delegate. The workflow checks everything before it writes anything. It then reuses or creates each contact under the event's booking account, registers them, records attendance on every non-cancelled session, and optionally records Pass/Fail. It returns a results page showing the outcome for every delegate.

## Features

- Standalone HTML form with add/remove rows and paste-from-spreadsheet
- Webhook protected by header authentication
- Up-front validation of the whole submission (required fields, email format, duplicates, limits)
- Checks that the event is **private**, published and has a booking account
- Reuses existing contacts matched by email; detects ambiguous duplicate contacts and existing enrolments
- Creates new contacts under the event's booking account (never an account sent by the browser)
- Registers delegates and records attendance on **every non-cancelled session**
- Optional Pass/Fail per delegate ("No result" leaves the existing result unchanged)
- Batched GraphQL requests with thorough error checking; a clear HTML results page
- Limits: 50 delegates, 100 sessions and 1,000 attendance records per submission

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials configured
- Somewhere staff can open `form.html` (a local file, or an internal web page)
- Permission to create contacts, register learners, and record attendance and results

### Installation

1. Download `workflow.json` and `form.html` from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Set credentials

| Credential type | Name | Nodes |
|---|---|---|
| Administrate OAuth2 | any | `Check Contacts and Event1`, `Create Contacts1`, `Register Contacts1`, `Record Attendance1`, `Record Learner Results` |
| Header Auth (`httpHeaderAuth`) | `Webhook Header Auth` | `Delegate Webhook1` |

Create the **Webhook Header Auth** credential in n8n (Credentials → New → Header Auth):
- **Name**: the header name, e.g. `X-Webhook-Key`
- **Value**: a long random secret

Requests without this header and value are rejected by n8n before the workflow runs.

#### 2. Activate the workflow

Activate it and copy the **Production URL** from `Delegate Webhook1` (`https://YOUR-N8N-HOST/webhook/<path>`).

#### 3. Configure `form.html`

Open `form.html` in a text editor and edit the clearly marked **CONFIG** block at the top of the `<script>`:

| Constant | Set to |
|---|---|
| `WEBHOOK_URL` | The Production URL from step 2 (replace `https://YOUR-N8N-HOST/webhook/REPLACE_WITH_WEBHOOK_PATH`) |
| `AUTH_HEADER_NAME` | The header name from the Webhook Header Auth credential (default `X-Webhook-Key`) |
| `AUTH_HEADER_VALUE` | The header value from that credential (replace `REPLACE_WITH_WEBHOOK_SECRET`) |

**Keep the configured form private.** Anyone with the file can call the webhook. Share it only with staff (for example, host it on an internal site) and change the credential value if it leaks.

#### 4. CORS

The form calls the webhook from the browser, so the webhook must allow the page's origin. `Delegate Webhook1` → Options → **Allowed Origins (CORS)** is set to `*`. For production, change it to the origin that serves the form (e.g. `https://intranet.example.com`).

#### 5. Optional settings

Defaults live at the top of the `Validate Submission1` Code node: the API URL (`https://api.getadministrate.com/graphql`), `displayTimeZone` (`Europe/London`, used only when the event has no time zone), and the limits.

There are no Data Tables in this workflow.

### Payload

The form sends `POST` JSON in this shape (the workflow also accepts `delegates` as a JSON string, or a single delegate as top-level `firstName`/`lastName`/`email`/`result` fields):

```json
{
  "eventID": "<encoded Administrate Event ID>",
  "delegates": [
    { "firstName": "Alex", "lastName": "Smith", "email": "alex.smith@example.com", "result": "pass" },
    { "firstName": "Sam", "lastName": "Jones", "email": "sam.jones@example.com", "result": "none" }
  ]
}
```

`result` must be `none`, `pass` or `fail`.

### Testing

1. Create or pick a **published private** event with a booking account and at least one session
2. Open `form.html`, enter the event's encoded API ID and two test delegates (one existing contact, one new)
3. Submit. The results page should show both as "Registered · Attended" with the chosen results
4. In Administrate, check the learners, their session attendance and their results
5. Change `AUTH_HEADER_VALUE` to a wrong value and submit again. The form should report that the webhook rejected the request.

## How It Works

1. **Delegate Webhook1**: receives the POST (header auth checked by n8n)
2. **Validate Submission1**: validates the event ID and delegate rows, then builds one batched query
3. **Check Contacts and Event1 / Check Results1**: loads the event (type, state, booking account, sessions) and looks up each email's contact and existing active enrolment. Any problem stops the run before anything is written.
4. **Build Contact Routes1**: splits delegates into existing and new contacts. Both branches always run, so the Merge always receives two inputs.
5. **Create Contacts1**: creates new contacts under the event's booking account (a read-only no-op if there are none)
6. **Register Contacts1**: registers contacts who aren't already enrolled
7. **Record Attendance1**: marks each learner as attended on every non-cancelled session
8. **Record Learner Results**: records Pass/Fail where selected and attendance was confirmed
9. **Build HTML Response1 / Respond with HTML1**: returns a results page (HTTP 200 on full success, 207 for partial success, 4xx for validation problems)

## Troubleshooting

**The form says the webhook rejected the request:**
- `AUTH_HEADER_NAME` / `AUTH_HEADER_VALUE` in `form.html` must match the Webhook Header Auth credential exactly

**The form says it could not reach the webhook:**
- Check `WEBHOOK_URL` (Production URL, `/webhook/…`) and that the workflow is active
- Check **Allowed Origins (CORS)** on `Delegate Webhook1` includes the form's origin. Opening the form as a local file sends `Origin: null`, which `*` allows.

**"This form is for private courses only":**
- The account for new contacts comes from the event's booking account, so only private events are supported

**"This email matches multiple contact records":**
- Merge or fix the duplicate contacts in Administrate, then resubmit

**Partial success (some delegates need attention):**
- Changes that succeeded are kept. Fix the listed rows and resubmit; existing enrolments are reused.
