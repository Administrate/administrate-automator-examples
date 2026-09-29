# Review and Cancel Under-Subscribed Events

**Version:** 1.0.0
**Last Updated:** 2026-09-29

## Problem

Events that don't reach their minimum numbers need to be cancelled early enough for instructors and learners to change their plans. Finding them means someone checking upcoming events against registration counts every day. Cancelling each one then means updating the event, emailing instructors, emailing learners and cancelling every registration by hand.

## Automator Solution

Every weekday morning the workflow finds published events that are under-subscribed and emails the planning team a review table. Each event has its own **Cancel Event** button. Nothing is cancelled automatically: a person reviews the list and confirms each cancellation. The workflow then cancels the event, emails the instructors and learners with your Administrate email templates, and cancels each learner registration.

## Features

- Scheduled check (Mon–Fri 8:30am) with a manual trigger for previews
- Two configurable rules:
  - **4-week rule**: events 25–31 days out with fewer than `FOUR_WEEK_MIN_REGS` registrations
  - **2-week rule**: events 11–17 days out with fewer than 50% of their minimum places
- Pages through all matching events (cursor pagination), not just the first page
- One review email with a table of events and a Cancel button for each one
- Safe approval links:
  - Opening a link only shows a confirmation page. Email link scanners that pre-fetch links cannot cancel anything.
  - The cancellation happens only when someone presses **Confirm cancellation**, which sends a POST
  - Each link carries a random one-time token that expires after `APPROVAL_LINK_TTL_DAYS`
- The rules are checked again at confirmation time, so events that gained registrations after the email went out are left alone
- Instructor and participant notifications sent with Administrate email templates
- Every learner registration on the event is cancelled

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials configured
- An SMTP account for sending the review email
- Two Administrate email templates (one for instructors, one for participants) and a sending address
- Permission to update events and cancel learners

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

The workflow has two parts on one canvas: **Part 1 – Cancellation Checker** (schedule, top) and **Part 2 – Cancellation Executor** (Approval Webhook, bottom). They share the one-time tokens through the workflow's static data, so keep them in the same workflow.

### Configuration

#### 1. Set credentials

| Credential type | Nodes |
|---|---|
| Administrate OAuth2 | `ADM: Get Upcoming Events`, `ADM: Get Upcoming Events1`, `ADM: Cancel Event`, `ADM: Send Instructor Email`, `ADM: Send Participant Emails`, `ADM: Cancel Learner Registration` |
| SMTP | `Send Review Email1` |

#### 2. Edit the `Config` node (Part 1)

| Key | Default | Description |
|---|---|---|
| `ADM_API_URL` | `https://api.getadministrate.com/graphql` | Administrate GraphQL endpoint |
| `FOUR_WEEK_MIN_REGS` | `3` | 4-week rule: minimum registrations |
| `FOUR_WEEK_FROM_DAYS` / `FOUR_WEEK_TO_DAYS` | `25` / `31` | 4-week rule window (days from today) |
| `TWO_WEEK_FROM_DAYS` / `TWO_WEEK_TO_DAYS` | `11` / `17` | 2-week rule window (days from today) |
| `PLANNING_EMAIL` | `admin@example.com` | Who receives the review email |
| `FROM_EMAIL` | `admin@example.com` | Sender address for the review email (must be allowed by your SMTP account) |
| `EXECUTOR_WEBHOOK_URL` | `https://YOUR-N8N-HOST/webhook/<path>` | The **Production URL** of the `Approval Webhook` node. Replace `YOUR-N8N-HOST` with your n8n host; the path already matches the imported webhook. |
| `ADM_INSTANCE_SUBDOMAIN` | `REPLACE_WITH_INSTANCE_SUBDOMAIN` | The first part of your Administrate URL (`<subdomain>.administrateapp.com`). Used to link each event in the email. |
| `APPROVAL_LINK_TTL_DAYS` | `7` | How long a Cancel link stays valid |
| `PAGE_SIZE` / `MAX_PAGES` | `50` / `20` | Events fetched per API page, and the page limit |

#### 3. Edit the `Config1` node (Part 2)

| Key | Default | Description |
|---|---|---|
| `ADM_API_URL` | `https://api.getadministrate.com/graphql` | Administrate GraphQL endpoint |
| `FOUR_WEEK_*` / `TWO_WEEK_*` | as above | Keep these the same as in `Config` |
| `INSTRUCTOR_ROLE_NAME` | `Instructor` | Staff role name that counts as an instructor (not case-sensitive) |
| `INSTRUCTOR_TEMPLATE_ID` | `REPLACE_WITH_INSTRUCTOR_TEMPLATE_ID` | Administrate email template ID sent to instructors |
| `PARTICIPANT_TEMPLATE_ID` | `REPLACE_WITH_PARTICIPANT_TEMPLATE_ID` | Administrate email template ID sent to learners |
| `SENDING_ADDRESS_ID` | `REPLACE_WITH_SENDING_ADDRESS_ID` | Administrate sending email address ID |
| `PAGE_SIZE` / `MAX_PAGES` | `50` / `20` | API paging for the re-check |

No Data Tables are needed. The one-time tokens are kept in the workflow's static data.

#### 4. Activate the workflow

The schedule and the Approval Webhook only run in production once the workflow is active.

### Testing

1. Make sure at least one published event falls inside a rule window and is under its threshold
2. Activate the workflow
3. Links only work from **production runs**. n8n saves static data (where the tokens live) only for active, trigger-started executions. Emails from a manual "Execute workflow" run are fine for checking the content, but their Cancel links show "Link invalid or expired". To get working links quickly, change the schedule cron to a few minutes from now, wait for the run, then change it back.
4. Open the review email and click **Cancel Event** on a test event. A confirmation page appears and nothing has changed yet.
5. Click **Confirm cancellation**. The event should be cancelled, the instructor and participant emails sent, and every learner cancelled.
6. Click the same link again. It should show "Link invalid or expired".

## How It Works

**Part 1 – Cancellation Checker**

1. **Trigger**: runs Mon–Fri at 8:30am (or manually)
2. **Build ADM Query**: builds an `events` query for published events 11–31 days out
3. **ADM: Get Upcoming Events**: the HTTP Request node's built-in pagination follows `pageInfo.endCursor` until `hasNextPage` is false (up to `MAX_PAGES`). It returns one item per page.
4. **Apply Cancellation Rules**: combines the pages and applies the 4-week and 2-week rules
5. **Build Review Email**: creates a random one-time token for each qualifying event and saves it in workflow static data, then builds the HTML table with a Cancel link per event (`?eventId=…&token=…`)
6. **Send Review Email1**: sends the table to `PLANNING_EMAIL` over SMTP

**Part 2 – Cancellation Executor**

1. **Approval Webhook** accepts GET and POST
2. **GET (link opened)** → `Build Confirmation Page` checks the token (read-only) and returns a page with the event details and a **Confirm cancellation** button, or an "invalid or expired" page. No data is changed.
3. **POST (button pressed)** → `Verify Approval Token` checks that the token matches the event and hasn't expired, then deletes it so it can't be used again. Invalid requests get the `Invalid Link` page (HTTP 403).
4. **Build ADM Query1 / ADM: Get Upcoming Events1 / Apply Cancellation Rules1**: fetch all pages again and re-apply the rules to the verified event only. If it no longer qualifies, the `No Longer Qualifies` page is returned.
5. **ADM: Cancel Event**: sets the event's lifecycle state to `cancelled`
6. **Build Notification Data → Confirm Cancellation**: returns a confirmation page to the browser, then carries on in the background
7. **ADM: Send Instructor Email / ADM: Send Participant Emails**: send the templates with `bulkAdhocEmails.sendBulkContactAdhocEmail`
8. **ADM: Cancel Learner Registration**: runs `learner.cancel` once per learner

## Troubleshooting

**No review email arrives:**
- Check that the workflow is active and the schedule has run (Executions list)
- If no events qualify, the run ends at `No Events – Done` and no email is sent
- Check the SMTP credential and that `FROM_EMAIL` is allowed to send

**Clicking Cancel Event shows "Link invalid or expired":**
- The link has already been used, or is older than `APPROVAL_LINK_TTL_DAYS`
- The email came from a manual test run (tokens aren't saved for manual runs, see Testing)
- The workflow was re-imported or its static data was reset after the email was sent

**Clicking the link gives a 404:**
- `EXECUTOR_WEBHOOK_URL` must be the Approval Webhook's **Production URL** (`/webhook/…`, not `/webhook-test/…`), and the workflow must be active

**"No Longer Qualifies" page:**
- The event gained registrations, moved outside the rule windows, or is no longer published. Nothing was changed.

**Instructors or learners not emailed:**
- Check `INSTRUCTOR_TEMPLATE_ID`, `PARTICIPANT_TEMPLATE_ID` and `SENDING_ADDRESS_ID` in `Config1`
- Check that `INSTRUCTOR_ROLE_NAME` matches the staff role used on your events

**Some events are missing from the review email:**
- Increase `MAX_PAGES` (or `PAGE_SIZE`) if you have more than `PAGE_SIZE × MAX_PAGES` published events in the window
