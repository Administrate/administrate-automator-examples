# Daily Event Resourcing Gaps Digest

**Version:** 1.0.0
**Last Updated:** 2026-09-29

## Problem

Operations teams need to know which upcoming events are still short of instructors, other staff or resources such as rooms and equipment. In Administrate that means opening events one by one or running several reports. Gaps are often found too late to fix.

## Automator Solution

Every morning, the workflow reads the events starting in the next few weeks and Administrate's resourcing checks for them. It emails one HTML digest showing, per event, the personnel and resources that are required, booked and missing. Events that need action are listed first. You can choose to only show events that have gaps.

## Features

- Daily email at 07:00 with every upcoming event in a configurable window
- Personnel by staff role: required, booked and missing
- Resources by resource type: required, booked and missing
- Events are flagged **ACTION NEEDED** (something missing), **OVER-BOOKED** or **OK**, and sorted by severity then start date
- Fill rate and target fill rate per event
- Links from each event title to the event in Administrate
- Optional "only show gaps" mode and lifecycle state exclusions (for example draft and cancelled)
- Cursor pagination on both API queries, with a configurable safety cap
- No email is sent when there are no events to report

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials with permission to read events and event issues
- An SMTP credential that can send from the address you set in `FROM_EMAIL`

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Set Credentials

- **Administrate OAuth2**: select it on `Query Events - Resources` and `Query Event Issues - Staff`
- **SMTP**: select it on `Send Digest Email`

#### 2. Fill in the Config node

| Setting | Placeholder / default | What to set |
| --- | --- | --- |
| `ADMIN_API_URL` | `https://YOUR-INSTANCE.administrateapp.com/graphql` | Your Administrate GraphQL endpoint |
| `EVENT_URL_TEMPLATE` | `https://YOUR-INSTANCE.administrateapp.com/beta#/events/events/{id}` | Link for each event title. Keep `{id}`, which is replaced with the numeric event ID. Leave empty for no links. |
| `RECIPIENTS` | `admin@example.com` | Comma-separated list of people who receive the digest |
| `FROM_EMAIL` | `admin@example.com` | Sender address. Your SMTP server must allow it. |
| `WINDOW_DAYS` | `30` | How many days ahead to look |
| `MAX_EVENTS` | `1000` | Safety cap on records read per query, across all pages |
| `PAGE_SIZE` | `100` | Records per API page (100 is the Administrate maximum) |
| `ONLY_SHOW_GAPS` | `false` | `true` only lists events that are short or over-booked |
| `EXCLUDE_STATES` | `draft,cancelled` | Comma-separated event lifecycle states to leave out |
| `EMAIL_SUBJECT_PREFIX` | `Event Resourcing Digest` | Text at the start of the email subject |

#### 3. Set the send time (optional)

The digest is sent at 07:00 in the workflow's timezone. Change it on the `Daily Schedule` node.

### Testing

1. Set `RECIPIENTS` to your own address
2. Run the workflow manually
3. Check the `Build Digest` output and the email you receive
4. Compare one or two events against their resourcing in Administrate
5. Set the real recipients and activate the workflow

## How It Works

1. **Trigger**: The schedule fires daily at 07:00
2. **Config**: Loads the settings above
3. **Query Events - Resources**: Reads events whose start is between now and `WINDOW_DAYS` ahead, with their required resources (type, quantity, sessions). It follows `pageInfo.endCursor` until `hasNextPage` is false or `MAX_EVENTS` is reached.
4. **Query Event Issues - Staff**: Reads `eventIssues` from today onwards, page by page: staff role fulfilment and missing resource types per session, plus fill rate and location. This node runs once, however many pages the first query returned.
5. **Build Digest**: Joins both results by event ID and skips excluded states. It adds up the session figures per event (a requirement across N sessions counts as N slots), works out Booked = Required − Missing for resources, flags gaps and over-bookings, and builds the HTML email.
6. **Send Digest Email**: Sends the digest over SMTP

## Troubleshooting

**No email arrives:**
- If there are no events in the window (or none with gaps when `ONLY_SHOW_GAPS` is `true`), no email is sent
- Check the SMTP credential and that `FROM_EMAIL` is allowed to send

**"Administrate GraphQL returned errors":**
- Check `ADMIN_API_URL` and that the OAuth2 credential has access to events and event issues

**Booked and Missing show "—" for an event:**
- That event was not found in the `eventIssues` results. `eventIssues` is only filtered by start date from today, so it can include events after the window. Raise `MAX_EVENTS` so more pages are read.

**Event links go to the wrong place:**
- Check `EVENT_URL_TEMPLATE` uses your instance name and still contains `{id}`

**Digest is sent at the wrong time:**
- Check the workflow timezone (Workflow Settings) and the hour on `Daily Schedule`
