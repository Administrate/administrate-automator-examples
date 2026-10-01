# Instructor Availability Conflicts Digest

## Problem

Instructors get double-booked: two sessions at the same time after a reschedule, a session on a day they are recorded as absent or a session added on top of another one. Administrate flags these conflicts, but only when someone opens the right screen. They are often found a few days before the session, when replacing the instructor is hardest.

## Automator Solution

Every weekday morning this workflow asks Administrate for its own conflict check over the coming weeks and emails one digest to the people who manage instructors. The digest lists, per instructor and by date, every session they are booked on while also booked on another session or recorded as absent. When there is no conflict, no email is sent. The workflow only reads data: it never changes events, sessions or bookings.

## Features

- Uses Administrate's own conflict check (`contactConflicts`), so overlaps and absences are worked out by Administrate, not guessed from text
- Covers every event, whether it was created by hand, imported or planned with Scheduler
- One email, grouped by instructor and sorted by date, with duplicates removed
- Shows what each session conflicts with: another session or an absence, with its times
- Optional links to each event
- No email when there is nothing to report
- Option to leave out conflicts with unconfirmed bookings
- Dry-run mode that builds the digest without sending it
- Read only: nothing in Administrate is changed

## Setup Instructions

### Prerequisites

- Access to Administrate Automator (n8n)
- An Administrate OAuth2 credential with permission to read contacts and sessions and to send emails to contacts
- A verified Administrate sending email address
- Each recipient must exist as a contact in Administrate, because the email is sent through Administrate

### Installation

1. Download `workflow.json` from this folder.
2. In Automator, create a workflow, open the menu (⋮) and select **Import from File**.
3. Follow the configuration steps below.

### Configuration

#### 1. Set OAuth credentials

Attach your Administrate OAuth2 credential to the three HTTP Request nodes: `Read Conflicts`, `Find Recipients` and `Send Digest`.

#### 2. Fill in the Config node

| Key | Set to |
|---|---|
| `WINDOW_DAYS` | How many days ahead to check (default `30`) |
| `INCLUDE_UNCONFIRMED` | `false` to leave out conflicts with unconfirmed bookings (default `true`) |
| `RECIPIENT_EMAILS` | `REPLACE_WITH_RECIPIENT_EMAIL` -> comma-separated addresses; each must be an Administrate contact |
| `SENDING_ADDRESS_ID` | `REPLACE_WITH_SENDING_ADDRESS_ID` -> ID of a verified sending address |
| `EVENT_URL_TEMPLATE` | Link to an event, where `{id}` is replaced by the event ID. Leave the `YOUR-INSTANCE` placeholder to show no links |
| `EMAIL_SUBJECT_PREFIX` | Text at the start of the email subject |
| `BRAND_NAME`, `BRAND_COLOUR` | Name and header colour shown in the email |
| `DRY_RUN` | `true` to build the digest without sending it (default `false`) |

To find the sending address ID:

```graphql
query { sendingEmailAddresses { edges { node { id name address verified } } } }
```

#### 3. Set the send time (optional)

The `Every Morning` trigger runs at 07:00, Monday to Friday. Change its cron expression to suit your team.

#### 4. Optional: set an error workflow

If you use a shared error workflow, such as [Log Automation Failures to Administrate](../error-logging-to-administrate/), select it under **Workflow settings > Error workflow**. The workflow stops with a clear error if Administrate cannot be read, if a recipient is not a contact or if an email cannot be sent.

### Testing

1. Set `DRY_RUN` to `true` and run the workflow with **Run Now**.
2. Check the `Dry Run Report` node: it shows the subject, the number of sessions with a conflict and the instructors concerned. Compare a few lines with the instructor calendars in Administrate.
3. Set `RECIPIENT_EMAILS` to your own address (it must be a contact), set `DRY_RUN` to `false` and run again. Check the email you receive.
4. Add an address that is not a contact to `RECIPIENT_EMAILS` and run again: the workflow should stop before sending anything and name the address.
5. Set the real recipients and activate the workflow.

## How It Works

1. **Every Morning** (or **Run Now**) reads the Config node.
2. **Build Query** and **Read Conflicts** ask Administrate for every contact with a conflict between today and today + `WINDOW_DAYS`. Administrate returns, for each contact, the sessions in conflict and what each one conflicts with.
3. **Build Digest** removes duplicates, optionally leaves out unconfirmed bookings, groups the sessions by instructor and sorts them by date, then builds the HTML email. If nothing is left, the workflow stops and no email is sent.
4. **Dry Run?** sends the result to **Dry Run Report** when `DRY_RUN` is on.
5. **Build Recipient Query**, **Find Recipients** and **Resolve Recipients** look up each address in `RECIPIENT_EMAILS` as an Administrate contact. If one is missing, the run stops before any email is sent.
6. **Send Digest** sends the email to each recipient through Administrate, and **Check Send** confirms each one was accepted.

### Related example

[Daily Event Resourcing Gaps Digest](../event-resourcing-digest/) reports events that are missing instructors or resources. This example reports instructors who are booked twice or while absent. The two work well together.

## Troubleshooting

**No email arrives:**
- If there is no conflict in the window, no email is sent. Run with `DRY_RUN` set to `true` and check the `Build Digest` output
- Check `SENDING_ADDRESS_ID` is a verified sending address

**The run stops with "No Administrate contact found for":**
- Create a contact for that address in Administrate, or remove it from `RECIPIENT_EMAILS`

**A conflict you expected is missing:**
- Check it falls within `WINDOW_DAYS`
- If `INCLUDE_UNCONFIRMED` is `false`, conflicts with unconfirmed bookings are left out

**Links to events do not work:**
- Set `EVENT_URL_TEMPLATE` to your instance address, keeping `{id}` where the event ID goes
