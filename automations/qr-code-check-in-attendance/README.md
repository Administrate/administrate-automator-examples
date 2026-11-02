# QR Code Check-in and Attendance

## Problem

Taking attendance on paper registers or by clicking through each learner in Administrate is slow, error-prone and has to be re-keyed afterwards. Reception staff cannot see at a glance who has arrived. Walk-ins who are not yet booked on a private course have to be added by hand before their attendance can be marked.

## Automator Solution

A set of six linked workflows gives each Event a printable QR code page. Learners scan the code with their phone and enter their email address, and Automator:

- finds the matching Contact (or, on private events, creates one and registers them as a walk-in),
- checks that check-in for the session is open,
- records session attendance in Administrate,
- logs every scan to an n8n Data Table.

A live reception dashboard shows who has arrived, a daily email reconciles the day's scans, and a failure handler emails an alert if anything breaks.

## Features

- One-click QR link generation from the **Automator** menu on an Event
- Printable / downloadable QR page with one code per session (or one per event)
- Mobile-friendly n8n check-in form with no app to install
- Check-in window per session (opens before start, closes after), timezone-aware, including multi-day sessions
- Existing contacts matched by email. Unknown learners on **private** events can self-register as walk-ins. Public events send them to reception.
- Walk-in account matching by email domain or company name, falling back to an individual account. Free-mail domains such as gmail.com never match by domain, so a walk-in is never filed under a stranger's account.
- Attendance recorded with `learner.recordAttendance`; registration uses `event.registerNamedLearners`
- Every scan, successful or refused, logged to a Data Table and to Administrate's External Integration Log
- Clear result screens for every outcome (checked in, not open yet, closed, see reception, and so on)
- Reception dashboard: checked in / expected / walk-ins per session, auto-refreshing
- Daily reconciliation email: unmarked scans, contacts needing review, possible duplicates
- Error workflow that emails an actionable alert with a link to the failed execution

## The Workflows

| File | Workflow | Trigger | Purpose |
|---|---|---|---|
| `workflows/10-qr-url-generator.json` | 1.0 QR Check-in - URL Generator (on Event) | Administrate manual Event action | Writes the QR page link into an Event custom field |
| `workflows/20-qr-code-page.json` | 2.0 QR Check-in - QR Code Page | Webhook (GET) | Renders the printable QR page for an event |
| `workflows/30-check-in-form.json` | 3.0 QR Check-in - Check-in Form | n8n Form | The check-in process itself |
| `workflows/40-reconciliation-report.json` | 4.0 QR Check-in - Daily Reconciliation Report | Schedule (07:00 daily) | Emails a digest of the last 24 hours of scans |
| `workflows/50-check-in-dashboard.json` | 5.0 QR Check-in - Check-in Dashboard | Webhook (GET) | Live reception dashboard |
| `workflows/60-failure-handler.json` | 6.0 QR Check-in - Failure Handler | Error Trigger / manual | Emails an alert when any of the above fails |

The chain is: **1.0** writes a link to **2.0** → **2.0** prints codes that open **3.0** → **3.0** writes the check-in log → **4.0** and **5.0** read the log. **6.0** watches all of them.

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials, able to read events and contacts, create contacts and accounts, register learners, record attendance and write External Integration Logs
- The Administrate trigger node (`CUSTOM.administrateTrigger`) available in your Automator instance
- n8n Data Tables enabled
- An SMTP credential for the report and alert emails. The report can also send through Administrate, see step 7.
- An Event custom field (type text) to hold the QR page link, for example **QR Code Generator URL**

### Installation

Import all six files from the `workflows/` folder in this order. In your Automator instance, click the menu (⋮) and select "Import from File" for each.

1. `60-failure-handler.json`, first, so it exists when you set it as the Error Workflow on the others
2. `30-check-in-form.json`
3. `20-qr-code-page.json`
4. `10-qr-url-generator.json`
5. `40-reconciliation-report.json`
6. `50-check-in-dashboard.json`

**Keep the leading number in each workflow name** (1.0, 2.0 ...). The failure handler uses it to explain what a failure means.

### Configuration

#### 1. Create the check-in log Data Table

Create one n8n Data Table named `EventCheckInLog` with these 18 columns and copy its ID:

| Column | Type | Contents |
|---|---|---|
| `timestamp` | string | ISO time of the scan |
| `eventId` | string | Administrate Event ID |
| `eventTitle` | string | Event title |
| `email` | string | Email entered |
| `domain` | string | Email domain |
| `firstName` | string | First name |
| `lastName` | string | Last name |
| `company` | string | Company or account name |
| `contactId` | string | Administrate Contact ID |
| `learnerId` | string | Administrate Learner ID |
| `outcome` | string | `already-registered`, `registered-now`, `new-created-registered` or `not-registered` |
| `accountId` | string | Account the walk-in was filed under |
| `status` | string | Match status, e.g. `found-existing`, `reconciled-domain`, `needs-review-*` |
| `matchMethod` | string | How the contact/account was matched |
| `suggestedAccountName` | string | Name of the matched account |
| `sessionId` | string | Administrate Session ID the attendance was recorded against |
| `attendanceMark` | string | `Present`, or `blocked-<reason>` when attendance was not recorded |
| `hasPassed` | boolean | Always `false` (reserved) |

Set the table ID as `CHECKIN_LOG_TABLE_ID` in the **Config** node of **3.0**, **4.0** and **5.0**.

#### 2. Set credentials

- **Administrate OAuth2**:
  - 1.0: `Generate QR Code Webhook`, `Write QR URL`, `Log Failure`, `Log Success`
  - 2.0: `Fetch Event`
  - 3.0: the twelve HTTP Request nodes listed in its Credentials sticky
  - 4.0: `Send Report`
  - 5.0: `Fetch Todays Events`, `Fetch More Learners`
- **SMTP**: 4.0 `Send Report SMTP`, 6.0 `Send Alert Email`

#### 3. Link the workflows (URLs)

The workflows call each other by URL. The imported files already contain matching paths, so in most cases you only need to replace `YOUR-N8N-HOST` with your n8n host name. n8n can change a webhook ID or path on import if it clashes with an existing one, so **after importing, open each trigger node, copy its Production URL and compare**:

| Where | Config field | Must equal |
|---|---|---|
| 2.0 Config | `BASE_URL` | 3.0 `Class Check-In (QR)` Form Trigger production URL (`https://YOUR-N8N-HOST/form/<webhook id>`) |
| 1.0 Config | `BASE_URL` | 2.0 `QR Page (Webhook)` production URL (`https://YOUR-N8N-HOST/webhook/<path>`) |
| 5.0 Config | `PAGE_URL` | 5.0 `Dashboard Page` production URL (its own address, used for links) |
| 6.0 Config | `N8N_BASE_URL` | `https://YOUR-N8N-HOST` (used to build execution links) |

Once QR codes are printed, do not recreate the 3.0 Form Trigger. Printed codes carry its URL.

#### 4. Placeholders to replace

| Workflow | Config field | Replace with |
|---|---|---|
| 1.0 | `QR_FIELD_KEY` = `REPLACE_WITH_QR_URL_CUSTOM_FIELD_KEY` | The `definitionKey` of your Event text custom field that stores the QR link. Read it from `customFieldTemplate(type: Event)` in the GraphQL API. |
| 3.0, 4.0, 5.0 | `CHECKIN_LOG_TABLE_ID` = `REPLACE_WITH_CHECKIN_LOG_TABLE_ID` | ID of the Data Table from step 1 |
| 4.0 | `REPORT_EMAILS` = `admin@example.com` | Comma separated recipients (SMTP route) |
| 4.0 | `SMTP_FROM` = `Automator <admin@example.com>` | Sender (SMTP route) |
| 4.0 | `SENDING_ADDRESS_ID` = `REPLACE_WITH_SENDING_ADDRESS_ID` | Administrate verified sending address ID (Administrate route only) |
| 4.0 | `REPORT_CONTACT_IDS` = `REPLACE_WITH_REPORT_CONTACT_IDS` | Comma separated Administrate Contact IDs to receive the report (Administrate route only) |
| 6.0 | `ALERT_TO` = `admin@example.com` | Who receives failure alerts |
| 6.0 | `ALERT_FROM` = `Automator <admin@example.com>` | Sender for alerts |

#### 5. Error workflow

After importing, open **Workflow Settings** on 1.0, 2.0, 3.0, 4.0 and 5.0 and set **Error Workflow** to `6.0 QR Check-in - Failure Handler`. The imported files do not carry this setting, because workflow IDs differ between instances.

#### 6. Check-in rules (3.0 Config)

| Field | Default | Meaning |
|---|---|---|
| `OPEN_MINUTES_BEFORE` | `120` | Check-in opens this many minutes before each session starts |
| `CLOSE_HOURS_AFTER` | `12` | Check-in closes this many hours after each session starts |
| `SELF_REGISTER_EVENT_TYPES` | `private` | Event types on which unknown or unbooked learners may register themselves. Empty = nobody. |
| `EMAIL_USAGE` | `primary` | Email usage for walk-in contacts |
| `ATTENDED` | `true` | Leave as `true` (see the caveat in 4.0's sticky) |
| `CHECKIN_WINDOW_OVERRIDE` | `off` | **Testing only.** `any` ignores the window. Must be `off` in production. |
| `ORIGIN_IDENTIFIER` | `Automator - QR Check-in` | Label on External Integration Log entries |
| `FREE_MAIL_DOMAINS` | `gmail.com,googlemail.com,outlook.com,...` | Comma separated public email domains (lower case, no @). A walk-in from one of these is never matched to an account by email domain, only by company name, otherwise an individual account. Add any other public domains your learners use. Empty = none. |

If you change `OPEN_MINUTES_BEFORE` / `CLOSE_HOURS_AFTER`, make the same change in 5.0's Config. 5.0 uses them only for labels.

#### 7. Other optional settings

- **2.0 Config**: `TITLE`, `HEADER`, `BODY`, `FOOTER`, `ALT_TEXT`, `LOGO` (optional `data:` image URL drawn in the centre of each code), `QR_MODE` (`session` or `event`), `TIME_24H`, `SAVE_LABEL`, `PRINT_LABEL`, `SHOW_SAVE_BUTTON`, `SHOW_PRINT_BUTTON`, `FILE_PREFIX` (PNG file name prefix). Colours and QR size are constants at the top of the `Build QR Page` Code node.
- **4.0 Config**: `EMAIL_TRANSPORT` (`smtp`, or `administrate` to send via Administrate's bulk ad hoc email), `SEND_ENABLED` (`false` builds the report without sending), `LOOKBACK_HOURS`, `DUP_THRESHOLD`, `REPORT_TITLE`.
- **5.0 Config**: `TIME_ZONE`, `REFRESH_SECONDS`, `PAGE_TITLE`, `LEARNER_PAGE_SIZE` (delegates read per request, default and maximum `100`), `LEARNER_MAX_PAGES` (most pages of delegates read per event, default `5`, so up to 500).
- **Timezone**: 1.0 to 5.0 have Workflow Settings timezone `Europe/London`, and 4.0 runs at 07:00 in that zone. Change it to yours.

#### 8. Secure the dashboard

The 5.0 `Dashboard Page` webhook is **open by default** and lists learner names. Before real use, set **Authentication** on that node (for example Basic Auth with an n8n Basic Auth credential), and do not share the URL publicly.

### Testing

1. Activate 6.0, then run it by hand from **Test Alert By Hand**. You should receive a "TEST alert" email.
2. Activate 3.0, 2.0 and 1.0. Activating 1.0 registers the **QR Code for Attendance** action on Events.
3. In Administrate, open an Event with a session today (or set `CHECKIN_WINDOW_OVERRIDE` to `any` temporarily). From the **Automator** menu choose **QR Code for Attendance**.
4. Refresh the Event and open the link in the custom field. The QR page shows one code per session.
5. Scan a code with your phone and enter the email of a learner booked on the event. You should see "You're checked in!", the learner's session attendance should show Present in Administrate, and a row should appear in the Data Table.
6. Try an unknown email on a private event. You should get the name page, then check-in as a walk-in.
7. Activate 5.0 and open its URL. The session card should show the check-in.
8. Run 4.0 by hand and check the digest email.
9. Set `CHECKIN_WINDOW_OVERRIDE` back to `off` and clear test rows from the Data Table (delete rows, not the table).

## How It Works

1. **Generate the link (1.0)**: Staff choose **QR Code for Attendance** on an Event. The workflow builds `BASE_URL?eventId=<id>`, writes it to the Event custom field `QR_FIELD_KEY`, and logs success or failure to the External Integration Log.
2. **Render the codes (2.0)**: Opening that link fetches the Event and its sessions and returns a self-contained HTML page (the QR library is inlined) with one code per session. Each code encodes the 3.0 form URL plus `eventId` and `sessionId`. URL parameters (`&mode=`, `&sessionId=`, `&date=`, `&from=`, `&to=`) filter the codes. Missing or non-matching events refuse to produce a code rather than print a useless one.
3. **Check in (3.0)**:
   - The form reads the email and hidden `eventId` / `sessionId`.
   - **Fetch Event** and **Resolve Session** pick the session whose check-in window is open. Nothing is written if none is open.
   - The contact is found by email. If they have no active learner on the event, they are registered on private events (`registerNamedLearners`) or sent to reception on public ones.
   - Unknown emails on private events get a second page (name, email, company, mobile). A contact is created under an account matched by email domain or company name, otherwise under a new individual account. Domains listed in `FREE_MAIL_DOMAINS` skip the email domain match.
   - `learner.recordAttendance` marks the session.
   - Every path ends in **Build Log Row**, which writes the Data Table row and an External Integration Log entry and shows the result screen.
4. **Dashboard (5.0)**: Reads today's published events with sessions and learners from Administrate and the walk-ins from the log. Shows one card per session with Checked in / Expected / Walk-ins counts and drill-down name lists. Events with more than 100 learners are read page by page (up to `LEARNER_MAX_PAGES`). If an event still has more, or a page fails, the dashboard names that event and warns that its numbers may be too low.
5. **Daily report (4.0)**: At 07:00 reads the last `LOOKBACK_HOURS` of log rows and emails counts by status and result, every scan whose attendance was not recorded (with the reason), contacts needing review, and possible duplicate people.
6. **Failure handler (6.0)**: When any other workflow fails in production, emails the workflow name, failing node, message, what it means in practice, and a link to the execution.

## Troubleshooting

**"QR Code for Attendance" not in the Automator menu:**
- Check that 1.0 is active and its trigger node has the Administrate OAuth2 credential
- Check the webhook registration is active in Administrate

**The custom field is not filled in:**
- Check `QR_FIELD_KEY` is the field's `definitionKey` in *your* instance, not its label
- Check 1.0's executions. `IF Write Errors?` stops the run and logs a failure if Administrate refused the update.

**The QR page shows "No event selected" or "No matching sessions":**
- The link needs `?eventId=`. Use the link written by 1.0.
- Remove or correct the `sessionId` / `date` / `from` / `to` filter

**Scanning opens a "Problem loading form" / 404 page:**
- 2.0 `BASE_URL` does not match 3.0's Form Trigger production URL, or 3.0 is not active

**"Check-in isn't open yet" / "Check-in has closed":**
- Working as designed. Adjust `OPEN_MINUTES_BEFORE` / `CLOSE_HOURS_AFTER`, or use `CHECKIN_WINDOW_OVERRIDE = any` for testing only.

**"Please see reception":**
- The email is unknown or not booked, and the event type is not in `SELF_REGISTER_EVENT_TYPES`

**"Check-in not completed" (attendance-refused):**
- Commonly "Learners missing from session": the session was added after the learner was booked. Add the learner to the session in Administrate and mark attendance by hand.

**Data Table errors:**
- Check `CHECKIN_LOG_TABLE_ID` in 3.0, 4.0 and 5.0, and that the table has all 18 columns. 3.0 stops with "missing Config" before writing anything if it is blank.

**A walk-in was filed under the wrong account:**
- If their email is on a public domain, add that domain to `FREE_MAIL_DOMAINS` in 3.0's Config

**The dashboard says "Not every delegate could be read":**
- The event has more learners than `LEARNER_MAX_PAGES` x `LEARNER_PAGE_SIZE`, or a page request failed. Raise `LEARNER_MAX_PAGES` or check 5.0's executions.

**Saved PNGs are blank:**
- Firefox with anti-fingerprinting enabled blocks canvas export. Use Chrome or Edge.

**No failure alerts:**
- The Error Workflow setting must be set on each workflow, and alerts fire only for production (not manual) runs. Use **Test Alert By Hand** to test the email path.
- If the alert email itself cannot be sent, 6.0's run fails. Check 6.0's own executions list and its SMTP credential.
