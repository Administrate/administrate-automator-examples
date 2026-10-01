# Certificate Renewal Reminders

## Problem

Many training certificates are only valid for a fixed number of years. Learners need to be told before their certificate runs out so they can book a refresher. Doing this by hand means running reports, working out who is due, emailing each learner and remembering who has already been reminded. People get missed or get the same email twice.

## Automator Solution

A daily scheduled workflow finds learners who passed a course that awards a renewable achievement, and whose certificate is coming up for renewal. It checks a sent log so nobody is reminded twice for the same event, then sends a templated reminder email per event using Administrate's bulk learner email. Every event email is logged once to Administrate's External Integration Logs, stating how many learners it went to. A separate error handler workflow records any run that fails outright in an n8n Data Table.

The courses in scope are not listed in the workflow. They come from each course template's **Achievements** tab. Add a renewable achievement to a course in Administrate and it is included on the next run.

## Features

- Runs daily at 07:00 (workflow timezone)
- Works out the due date from each achievement type's validity period and a configurable notice period
- Deduplicates against an n8n Data Table, so each learner is reminded once per event
- **Dry run by default**: produces a report of who would be emailed and sends nothing
- Throttle (`MAX_PER_RUN`) and safety net (`ABORT_ABOVE`) to stop runaway sends
- Catch-up mode for a one-time date range (`CATCHUP_FROM` / `CATCHUP_TO`)
- Refuses to send when the template or sending address is not set
- Checks page counts against totals, so a truncated read fails loudly instead of silently skipping learners
- Success and failure logging to External Integration Logs, one entry per event email; the run turns red if anything was held back or failed
- Error handler workflow that records failed runs, with a link to the execution, in a Data Table

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials, with permission to read course templates and learners and to send bulk learner emails
- One or more achievement types with a validity period, attached to course templates
- A communication template for the reminder email and a sending email address in Administrate

### Installation

This example has two workflows. Import them in this order:

| File | Workflow |
| --- | --- |
| [`workflows/10-error-handler.json`](workflows/10-error-handler.json) | Certificate Renewal Reminders - Error Handler |
| [`workflows/20-certificate-renewal-reminders.json`](workflows/20-certificate-renewal-reminders.json) | Certificate Renewal Reminders |

For each file:

1. Download the JSON file from the `workflows` directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Set OAuth Credentials

Select your **Administrate OAuth2** credential on every HTTP Request node:
- `Count Course Templates`
- `Read Register Page`
- `Count Due Learners`
- `Read Due Learners Page`
- `Send Reminder`
- `Log Success`
- `Log Failure`

#### 2. Create the Sent Log Data Table

Create an n8n Data Table (for example `certificate_renewal_sent_log`) with these columns:

| Column | Type |
| --- | --- |
| `learner_id` | string |
| `event_id` | string |
| `contact_id` | string |
| `course_template_id` | string |
| `achievement_type_id` | string |
| `event_end` | string |
| `validity_months` | number |
| `sent_at` | string |
| `run_mode` | string |

Then select this table in **both** Data Table nodes, `Read Sent Log` and `Record Sent`. They ship with the placeholder `REPLACE_WITH_SENT_LOG_DATA_TABLE_ID`.

#### 3. Fill in the Config node

| Setting | Placeholder / default | What to set |
| --- | --- | --- |
| `RENEWABLE_ACHIEVEMENT_TYPE_IDS` | `REPLACE_WITH_ACHIEVEMENT_TYPE_IDS_COMMA_SEPARATED` | Comma-separated Achievement Type IDs that should trigger reminders. The IDs differ between instances, so look them up in your own instance. |
| `TEMPLATE_ID` | `REPLACE_WITH_COMMUNICATION_TEMPLATE_ID` | The Communication Template ID for the reminder email |
| `SENDING_ADDRESS_ID` | `REPLACE_WITH_SENDING_EMAIL_ADDRESS_ID` | The Sending Email Address ID to send from |
| `NOTICE_PERIOD_MONTHS` | `6` | How many months before expiry to remind |
| `FLOOR_DATE` | `2021-01-01` | Ignore events that ended before this date |
| `WINDOW_DAYS` | `31` | Width of the due window. Use `7` in production so a missed run is picked up by the next one. |
| `DRY_RUN` | `true` | Set to `false` only after checking a dry run report |
| `MAX_PER_RUN` | `400` | Maximum learners emailed per run |
| `ABORT_ABOVE` | `2000` | Refuse the whole run if more learners than this are due |
| `SEND_AUDIENCE` | `passed` | Learner audience passed to the bulk email. It can only narrow the recipients. |
| `CATCHUP_FROM` / `CATCHUP_TO` | empty | Optional event end date range (`yyyy-MM-dd`) for a one-time catch-up. Clear afterwards. |
| `TIMEZONE` | `Europe/London` | Timezone for the due window. Keep it the same as the workflow timezone setting. |
| `ORIGIN_IDENTIFIER` | `Automator - Certificate Renewal Reminder` | Origin shown in External Integration Logs |
| `PAGE_SIZE` | `100` | Page size for API reads (100 is the maximum) |
| `GRAPHQL_URL` | `https://api.getadministrate.com/graphql` | Administrate GraphQL endpoint |

#### 4. Check the workflow timezone

The Schedule Trigger uses the workflow's timezone setting (Workflow Settings > Timezone), which ships as `Europe/London`. Change it to match `TIMEZONE`.

#### 5. 🚨 Set up the error handler

The error handler records any run of the main workflow that fails outright, for example a configuration guard refusing to send or a register read failing. It writes to an n8n Data Table rather than to Administrate, because the error payload carries no event ID to attach an External Integration Log to.

1. Create an n8n Data Table (for example `certificate_renewal_failures`) with these columns:

   | Column | Type |
   | --- | --- |
   | `failed_at` | string |
   | `workflow_name` | string |
   | `workflow_id` | string |
   | `execution_id` | string |
   | `execution_url` | string |
   | `execution_mode` | string |
   | `last_node` | string |
   | `error_message` | string |

2. Open **Certificate Renewal Reminders - Error Handler** and select this table in `Record Failure`. It ships with the placeholder `REPLACE_WITH_FAILURES_DATA_TABLE_ID`.
3. In `Shape Failure Row`, replace `https://YOUR-N8N-HOST` with your n8n host. The execution link is built from it because `execution.url` in the error payload can point at `localhost` when no editor base URL is configured.
4. Save and publish the error handler. Some instances refuse an unpublished workflow as an error workflow.
5. Open **Certificate Renewal Reminders**, go to Workflow Settings > Error Workflow and select **Certificate Renewal Reminders - Error Handler**. Save.

The error handler only records failures. If you want an alert as well, add an email or chat node after `Record Failure`.

### Testing

1. Leave `DRY_RUN` set to `true`
2. Run the workflow manually
3. Open the output of `Dry Run Report`. It lists the events and learners that would be emailed, and any intervals excluded by `FLOOR_DATE`.
4. When the list looks right, set `DRY_RUN` to `false` and run it again. Consider a low `MAX_PER_RUN` for the first live run.
5. Open a learner in Administrate and confirm the communication is in their history. A green execution proves nothing on its own.
6. Check that rows appeared in the sent log Data Table and that each event has one success entry in External Integration Logs, stating how many learners it went to
7. Run it again and confirm nobody is emailed twice
8. Read the `Run Summary` output for `suppressedByCap` and `droppedGroups`. Either being non-zero means work was left undone.
9. Test the error handler: an error workflow only fires on production executions, never manual ones. Publish a throwaway workflow that throws deliberately, set the error handler as its Error Workflow and confirm a row appears in the failures Data Table.
10. Activate the workflow

## How It Works

1. **Trigger**: The schedule fires daily at 07:00
2. **Read the course register**: All course templates are read in pages, and those with a renewable achievement type (and its validity period) are kept
3. **Build due windows**: For each validity period, the event end date range that is now due is calculated (validity minus notice period, over `WINDOW_DAYS`), limited by `FLOOR_DATE`. Catch-up dates override this if set.
4. **Find due learners**: Passed, non-cancelled learners on events ending in each window are counted, then read in pages. Totals are checked against the count.
5. **Deduplicate**: The sent log is read for each due event, and learners already reminded for that event are dropped. `ABORT_ABOVE` and `MAX_PER_RUN` are applied.
6. **Group by event**: Learners are grouped into one bulk email per event
7. **Dry run or send**: In dry run, a report is produced. Otherwise `sendBulkLearnerAdhocEmail` is called per event with the template and sending address.
8. **Record and log**: Successful sends are written to the sent log and logged once per event as a success, with the learner count. Failures are logged and the loop moves to the next event.
9. **Run summary**: The run fails at the end if any event failed or any learner was held back by the throttle
10. **Error handler**: If the run fails, the error handler records the workflow, node, error message and an execution link in the failures Data Table

## Troubleshooting

**"No course template carries a renewable achievement type":**
- Check `RENEWABLE_ACHIEVEMENT_TYPE_IDS` holds IDs from this instance
- Confirm the achievement types are attached to course templates and have a validity period

**"No reminder can ever be sent while FLOOR_DATE is ...":**
- `FLOOR_DATE` is later than every interval's due window. Move it earlier.

**"Refusing to send: TEMPLATE_ID and SENDING_ADDRESS_ID empty in Config":**
- Fill in both values, or set `DRY_RUN` back to `true`

**"Refusing to send: ... above ABORT_ABOVE":**
- The selection is much larger than expected. Check the Config dates and achievement types before raising `ABORT_ABOVE`.

**Run turns red with learners held back:**
- `MAX_PER_RUN` capped the run. The remaining learners are picked up on the next run while they are still inside the window.

**Learners emailed twice:**
- Make sure both Data Table nodes point at the same table

**One success log per learner instead of per event:**
- Make sure `Log Success` has Settings > Execute Once switched on. `Record Sent` passes on one item per learner.

**Failed runs not recorded in the failures Data Table:**
- Check the error handler is published and set as the Error Workflow in the main workflow's settings
- Error workflows do not fire for manual executions

**Execution link in the failures Data Table points at localhost:**
- Set `instanceBase_str` in `Shape Failure Row` to your n8n host

**Emails sent but not delivered:**
- Check the sending address and template in Administrate. Some test instances accept email but do not deliver it.
