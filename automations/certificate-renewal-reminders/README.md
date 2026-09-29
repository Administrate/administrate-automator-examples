# Certificate Renewal Reminders

## Problem

Many training certificates are only valid for a fixed number of years. Learners need to be told before their certificate runs out so they can book a refresher. Doing this by hand means running reports, working out who is due, emailing each learner, and remembering who has already been reminded. People get missed or get the same email twice.

## Automator Solution

A daily scheduled workflow finds learners who passed a course that awards a renewable achievement, and whose certificate is coming up for renewal. It checks a sent log so nobody is reminded twice for the same event, then sends a templated reminder email per event using Administrate's bulk learner email. Every send is logged to Administrate's External Integration Logs.

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
- Success and failure logging to External Integration Logs; the run turns red if anything was held back or failed

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials, with permission to read course templates and learners and to send bulk learner emails
- One or more achievement types with a validity period, attached to course templates
- A communication template for the reminder email and a sending email address in Administrate

### Installation

1. Download the `workflow.json` file from this directory
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

### Testing

1. Leave `DRY_RUN` set to `true`
2. Run the workflow manually
3. Open the output of `Dry Run Report`. It lists the events and learners that would be emailed, and any intervals excluded by `FLOOR_DATE`.
4. When the list looks right, set `DRY_RUN` to `false` and run it again. Consider a low `MAX_PER_RUN` for the first live run.
5. Check that learners received the email, rows appeared in the sent log Data Table, and entries appear in External Integration Logs
6. Run it again and confirm nobody is emailed twice
7. Activate the workflow

## How It Works

1. **Trigger**: The schedule fires daily at 07:00
2. **Read the course register**: All course templates are read in pages, and those with a renewable achievement type (and its validity period) are kept
3. **Build due windows**: For each validity period, the event end date range that is now due is calculated (validity minus notice period, over `WINDOW_DAYS`), limited by `FLOOR_DATE`. Catch-up dates override this if set.
4. **Find due learners**: Passed, non-cancelled learners on events ending in each window are counted, then read in pages. Totals are checked against the count.
5. **Deduplicate**: The sent log is read for each due event, and learners already reminded for that event are dropped. `ABORT_ABOVE` and `MAX_PER_RUN` are applied.
6. **Group by event**: Learners are grouped into one bulk email per event
7. **Dry run or send**: In dry run, a report is produced. Otherwise `sendBulkLearnerAdhocEmail` is called per event with the template and sending address.
8. **Record and log**: Successful sends are written to the sent log and logged as successes. Failures are logged and the loop moves to the next event.
9. **Run summary**: The run fails at the end if any event failed or any learner was held back by the throttle

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

**Emails sent but not delivered:**
- Check the sending address and template in Administrate. Some test instances accept email but do not deliver it.
