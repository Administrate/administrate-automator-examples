# Waitlist Automation: Timed Place Offers with Accept/Decline

**Version:** 1.0.0
**Last Updated:** 2026-09-29

## Problem

When an Event is full, learners go on a waitlist as Opportunities at a "Waitlist" step. When a place opens up, because the Event's capacity changes or a learner cancels, someone has to notice, work out who is next in the queue, hold the place for them, email them, chase an answer and then book them or move on to the next person. Done by hand, places sit empty, people are offered seats out of order, and a declined or ignored offer holds up everyone behind it.

## Automator Solution

A set of five workflows runs the waitlist for you:

- When a place frees up, the next waitlisted people (in queue order) are offered it. Their place is **reserved** and they get an email with a link to a response page.
- The response page shows the offer with **Accept** and **Decline** buttons. Accepting registers the contact on the Event and moves the Opportunity to Won. Declining releases the hold and the place rolls straight on to the next person.
- Offers that are not answered within the hold period (48 hours by default) expire automatically, and the place is offered to the next person.
- Every outcome is written to the Event's External Integration Log, and unexpected failures are recorded by a shared error workflow.

## Features

- Three ways a place is noticed: Event Updated, Learner Cancelled (opt-in per Event), and an hourly expiry sweep
- Queue order by waitlist date, with capacity worked out from active (non-cancelled) learners and outstanding offers
- Places are reserved for the offered person and the reservation is read back before any email is sent
- Scanner-safe response flow: the email link opens a page that changes nothing (GET); answers are recorded only when a button is pressed (POST), so corporate mail scanners cannot accept or decline on someone's behalf
- Concurrency-safe: offers and answers claim their Data Table row inside the write, so overlapping runs cannot double-offer or double-book
- Reservation release is verified on both accept and decline
- Branded HTML email and response pages, with all styling in Config
- Audit logging to Administrate External Integration Logs, plus a shared error workflow that records failures to a Data Table and against the affected Event

## Setup Instructions

### Prerequisites

- Access to Administrate Automator (n8n) with Data Tables available
- An Administrate OAuth2 credential
- An n8n API credential (for the error handler only; see Configuration step 6)
- An Opportunity step for waitlisted people (default name `Waitlist`) and a Won step to move accepted Opportunities to
- A verified Administrate sending email address
- An Event checkbox custom field named `Waitlist On?` (used to opt Events in for the Learner Cancelled trigger)

### Installation

Create the two Data Tables first (see Configuration step 1), then import the workflows **in this order**, using the menu (⋮) and "Import from File":

| Order | File | Workflow |
|---|---|---|
| 1 | `workflows/10-error-handler.json` | Waitlist 03 - Error Handler (shared) |
| 2 | `workflows/20-response-page.json` | Waitlist 01 - Response Page |
| 3 | `workflows/30-response-handler.json` | Waitlist 02 - Response Handler |
| 4 | `workflows/40-waitlist-automation.json` | Waitlist Automation |
| 5 | `workflows/50-learner-cancelled.json` | Waitlist 04 - Learner Cancelled |

The error handler goes first so it can be selected as the Error Workflow on the others, and the core goes before Learner Cancelled because Learner Cancelled calls it by ID.

### Configuration

Every value you need to change lives in the **Config** node at the start of each workflow (plus one trigger filter, see step 5). Placeholders are named `REPLACE_WITH_...`, and `YOUR-N8N-HOST` must be replaced with your n8n host name.

#### 1. Create the Data Tables

Create both in the Data Tables screen of the project the workflows live in. All columns are **String**.

**`waitlist_offers`** - one row per waitlisted request (Opportunity):

| Column | Written by |
|---|---|
| `event_id` | Waitlist Automation |
| `event_title` | Waitlist Automation |
| `contact_id` | Waitlist Automation |
| `opportunity_id` | Waitlist Automation |
| `status` | both (`waiting`, `offered`, `accepted`, `declined`, `expired`, `register_failed`) |
| `token` | Waitlist Automation |
| `waitlisted_since` | Waitlist Automation |
| `offered_at` | Waitlist Automation |
| `expires_at` | Waitlist Automation |
| `updated_at` | both |
| `response_channel` | Waitlist 02 |

**`automation_failures`** - one row per failed execution: `occurred_at`, `workflow_name`, `workflow_id`, `execution_id`, `execution_url`, `failed_node`, `error_message`, `event_id`, `logged_to_administrate`.

Copy each table's ID (from its URL in the Data Tables screen) for the next steps.

#### 2. Waitlist 03 - Error Handler (Config)

| Key | Set to |
|---|---|
| `FAILURES_TABLE_ID` | `REPLACE_WITH_AUTOMATION_FAILURES_TABLE_ID` -> ID of `automation_failures` |
| `N8N_BASE_URL` | `https://YOUR-N8N-HOST` -> your n8n base URL, no trailing slash |
| `ORIGIN_IDENTIFIER` | Optional. Log origin name; keep identical across all five workflows |
| `EVENT_ID_KEYS` | Leave as `event_id,eventId` |

#### 3. Waitlist 01 - Response Page and Waitlist 02 - Response Handler (Config)

Waitlist 01:

| Key | Set to |
|---|---|
| `OFFERS_TABLE_ID` | `REPLACE_WITH_WAITLIST_OFFERS_TABLE_ID` -> ID of `waitlist_offers` |
| `SUBMIT_URL` | `https://YOUR-N8N-HOST/webhook/<path>` -> the **production URL** of Waitlist 02's `Response Submitted` webhook. The path in the file already matches that webhook; only the host needs changing |
| `BRAND_NAME` | Your organisation's name (default `Your Organisation`) |
| `BRAND_CSS` | Optional. Page styling; the colour variables are at the top |
| `ACCEPT_LABEL` / `DECLINE_LABEL` | Optional. Button wording |

Waitlist 02:

| Key | Set to |
|---|---|
| `OFFERS_TABLE_ID` | `REPLACE_WITH_WAITLIST_OFFERS_TABLE_ID` -> ID of `waitlist_offers` |
| `WON_STEP_ID` | `REPLACE_WITH_WON_OPPORTUNITY_STEP_ID` -> ID of the Opportunity step accepted Opportunities move to |
| `BRAND_NAME` / `BRAND_CSS` | Same as Waitlist 01 |

`ACCEPT_ACTION` and `DECLINE_ACTION` must be identical in Waitlist 01 and Waitlist 02 (they are by default).

To find the Won step ID:

```graphql
query { opportunitySteps { edges { node { id name } } } }
```

#### 4. Waitlist Automation (Config)

| Key | Set to |
|---|---|
| `SENDING_ADDRESS_ID` | `REPLACE_WITH_SENDING_ADDRESS_ID` -> ID of a **verified** sending email address |
| `RESPONSE_PAGE_URL` | `https://YOUR-N8N-HOST/webhook/<path>` -> the **production URL** of Waitlist 01's `Offer Link` webhook. The path already matches; only the host needs changing |
| `OFFERS_TABLE_ID` | `REPLACE_WITH_WAITLIST_OFFERS_TABLE_ID` -> ID of `waitlist_offers` |
| `WAITLIST_STEP_NAME` | Name of your waitlist Opportunity step, exactly (default `Waitlist`) |
| `OFFER_RESPONSE_TIMEOUT_HOURS` | Hold period in hours (default `48`; `0` = hold indefinitely) |
| `BRAND_NAME` | Your organisation's name, shown in the email masthead |
| `EMAIL_SUBJECT_TEMPLATE`, `EMAIL_PALETTE`, `EMAIL_BUTTON_LABEL` | Optional. Email wording and colours |

To find the sending address ID:

```graphql
query { sendingEmailAddresses { edges { node { id name address verified } } } }
```

#### 5. Waitlist 04 - Learner Cancelled (Config and trigger)

| Key | Set to |
|---|---|
| `WAITLIST_FLAG_KEY` | `REPLACE_WITH_WAITLIST_FLAG_DEFINITION_KEY` -> definition key of the `Waitlist On?` Event custom field |
| `CORE_WORKFLOW_ID` | `REPLACE_WITH_ID_OF_WAITLIST_AUTOMATION` -> the ID of your imported Waitlist Automation workflow (from its URL, `/workflow/<id>`) |

The **Learner Cancelled Trigger** node's `filterExpression` also contains `REPLACE_WITH_WAITLIST_FLAG_DEFINITION_KEY`; replace it with the same key (the trigger cannot read Config).

To find the key:

```graphql
query { customFieldTemplate(type: Event) { customFieldDefinitions { key label type } } }
```

#### 6. Credentials

- **Administrate OAuth2** - attach to every Administrate node, Administrate Trigger and HTTP Request node in all five workflows (Waitlist 02 alone has ten HTTP Request nodes; each is listed in its Overview sticky note).
- **n8n API** - attach to `Fetch Failed Execution` in Waitlist 03. Create an API key in n8n under Settings > n8n API. This is only needed for the error handler.

#### 7. Set the Error Workflow

On each of Waitlist Automation, Waitlist 01, Waitlist 02 and Waitlist 04, open **Workflow settings > Error workflow** and select **Waitlist 03 - Error Handler (shared)**. The error handler is optional but recommended.

#### 8. Publish and activate the webhooks

1. Publish Waitlist 01 and Waitlist 02 first, so their webhooks are live, and confirm the URLs in steps 3 and 4.
2. Publish Waitlist Automation and Waitlist 04. Each registers an Administrate webhook.
3. Administrate registers webhooks as **inactive**. Activate each one:

```graphql
mutation { webhooks { update(input: { webhookId: "...", lifecycleState: active }) { errors { label message value } } } }
```

List registrations to find the IDs and check their state:

```graphql
query { webhooks { edges { node { id name url lifecycleState filterExpression } } } }
```

### Testing

1. Tick `Waitlist On?` on a test Event with a small capacity and fill it.
2. Add a test contact as an Opportunity Interest on the Event at the `Waitlist` step.
3. Cancel one learner. Within a few seconds Waitlist 04 runs and calls Waitlist Automation.
4. Check the Event: `reserved` should go up by one, and the test contact should receive the offer email. Open it in a real mail client and confirm the button renders as a filled button.
5. Click the link **in a browser** (the page returns 403 to non-browser clients) and press Accept.
6. Confirm the contact is now a learner on the Event, the Opportunity is at the Won step, `reserved` has dropped back, and `remainingPlaces` dropped by exactly one.
7. Check the Event's External Integration Logs for the offer and accept entries.
8. Repeat with Decline and confirm the place is offered to the next person. To test expiry, set `OFFER_RESPONSE_TIMEOUT_HOURS` to `1` and wait for the hourly sweep.

## How It Works

1. **Waitlist Automation** starts from one of three entry points: Event Updated (trigger), Learner Cancelled (called by Waitlist 04), or the hourly sweep, which marks offers past their `expires_at` as `expired` and re-runs the engine for those Events.
2. **Fetch Event Pages / Fetch Event** read the Event, its active learner count and every Interest (paging through all of them).
3. **Plan Offers** works out free places (`maxPlaces` minus active learners minus outstanding offers), picks the longest-waiting candidates at the waitlist step who have no open or accepted offer, and builds the email.
4. **Upsert Rows** records every candidate as `waiting`; **Claim Offer Rows** flips the chosen rows to `offered` only if they are still `waiting`, so overlapping runs cannot both offer the same place.
5. **Reserve Place / Set Reservation Hold / Verify Reservation** reserve the place until the offer expires and read it back. If the hold did not take, the row goes back to `waiting` and no email is sent.
6. **Send Offer Email** sends the offer from your sending address, with one link to the response page, and the outcome is logged against the Event.
7. **Waitlist 01 - Response Page** (GET) looks up the offer by token and shows the Accept/Decline form, or an already-answered/expired/full page. It changes nothing.
8. **Waitlist 02 - Response Handler** (POST) re-reads the row and the Event, re-checks capacity, claims the row, then either registers the contact, moves the Opportunity to Won and releases the hold (accept), or releases the hold so the place rolls on (decline). Each release is read back and the result logged.
9. **Waitlist 04 - Learner Cancelled** exists because cancelling a learner does not fire Event Updated. It resolves the learner to its Event, checks `Waitlist On?`, and hands the Event ID to Waitlist Automation.
10. **Waitlist 03 - Error Handler** catches any run that throws, fetches the failed execution through the n8n API to find the Event ID, writes a row to `automation_failures`, and logs against the Event when it can.

## Troubleshooting

**Nothing happens when a place opens:**
- Check the webhook registrations are **active**, not just registered
- For cancellations, confirm the Event has `Waitlist On?` ticked and the flag key is set in both Waitlist 04's Config and its trigger `filterExpression`
- Confirm `WAITLIST_STEP_NAME` matches the Opportunity step name exactly (a mismatch finds nobody and logs nothing)
- Keep `alwaysOutputData` switched on for `Load Offer Rows`, or an empty `waitlist_offers` table stops the engine silently

**Offer email not sent:**
- Confirm `SENDING_ADDRESS_ID` is a verified sending address
- Check the Event's External Integration Logs: a "Could not reserve" entry means the reservation did not take, so no email was sent (the row is set back to `waiting` and retried)

**Email button shows as a plain link:**
- Do not put quotes around font names in `EMAIL_PALETTE`

**The link in the email says the page cannot be reached, or the form does nothing:**
- Check `RESPONSE_PAGE_URL` (Waitlist Automation) and `SUBMIT_URL` (Waitlist 01) point at the **production** webhook URLs of the published Waitlist 01 and 02
- Testing with curl returns 403 because bots are ignored; use a browser

**Every answer comes back as not recognised:**
- `ACCEPT_ACTION` / `DECLINE_ACTION` differ between Waitlist 01 and Waitlist 02

**Accepted but not booked (`register_failed`):**
- Administrate refused the registration; the log entry on the Event explains why. The hold is kept for the person so an administrator can finish the booking by hand

**Accepted Opportunity lands on the wrong step:**
- Update `WON_STEP_ID` in Waitlist 02

**Error handler rows say "could not fetch the failed execution":**
- Check the n8n API credential and `N8N_BASE_URL` in Waitlist 03. The error handler only fires on production runs, never manual executions
