# Log Automation Failures to Administrate

**Version:** 1.0.0
**Last Updated:** 2026-09-29

## Problem

When an automation fails, the failure usually lives only in the n8n execution list. Administrators working in Administrate never see it, executions are pruned after a while, and nobody is told. Failures that stop a run part-way (a timeout, an expired credential, a Stop and Error) are especially easy to miss, because nothing downstream runs to report them.

## Automator Solution

Three workflows that give every automation consistent failure visibility:

1. **Error Handler - Log Failures to Administrate** is a shared **Error Workflow**. Any workflow that points to it gets its failed production runs written, in plain language, to Administrate's External Integration Log, and in full detail to an n8n Data Table.
2. **Subflow - Log to Administrate** is a reusable sub-workflow that any workflow can call to write a success or failure entry to the External Integration Log against a record.
3. **Daily Log Digest Email** gathers the last 24 hours of log entries for one log origin and emails them as a table.

## Features

- One shared Error Workflow for all your automations, set per workflow in Workflow settings
- Fetches the failed execution back through the n8n API, so the log carries the real reason and failed node, not just the trigger's metadata
- Plain-language entry in Administrate (what failed, why, when); technical detail (node, run mode, stack) kept in a Data Table for n8n users
- Durable, queryable failure history in the `automation_failures` Data Table, with a direct link to each failed execution
- Designed not to throw: the external calls continue on error so a failure is never lost to a second failure
- Ready-built notification text (`notify_subject`, `notify_body`) for adding an email or chat alert
- Reusable log sub-workflow with a standard payload format
- Optional daily digest email of log activity

## Setup Instructions

### Prerequisites

- Access to Administrate Automator (n8n) with Data Tables available
- An **Administrate OAuth2** credential
- An **n8n API** credential (API key from n8n Settings > n8n API), used by the error handler to read failed executions
- For the digest only: an existing workflow that sends an email (see Configuration step 5)

### Installation

Import each file using the menu (⋮) and "Import from File". The workflows do not depend on each other, so install the ones you need:

| File | Workflow |
|---|---|
| `workflows/10-error-handler.json` | Error Handler - Log Failures to Administrate |
| `workflows/20-log-to-administrate-subflow.json` | Subflow - Log to Administrate |
| `workflows/30-daily-log-digest.json` | Daily Log Digest Email (optional) |

### Configuration

#### 1. Create the `automation_failures` Data Table

In the Data Tables screen of the project the error handler lives in, create a table named `automation_failures` with these **String** columns:

`failed_at`, `workflow_name`, `workflow_id`, `execution_id`, `execution_url`, `failed_node`, `message`, `stage`, `details`, `notified`

Copy the table's ID from its URL.

#### 2. Configure the Error Handler

Open the **Config** node and set:

| Key | Set to |
|---|---|
| `FAILURES_TABLE_ID` | `REPLACE_WITH_AUTOMATION_FAILURES_TABLE_ID` -> the ID of `automation_failures` |
| `N8N_API_BASE_URL` | `https://YOUR-N8N-HOST/api/v1` -> your n8n host |
| `N8N_UI_BASE_URL` | `https://YOUR-N8N-HOST` -> your n8n host, no trailing slash |
| `ADM_LOG_ORIGIN_IDENTIFIER` | Optional. The origin name failure entries appear under (default `Automator - Workflow Failures`) |
| `ADM_RUN_LEVEL_NODE_ID` | Leave as `T3JnYW5pc2F0aW9uOg==` (files entries under the Organisation type) |
| `ADM_GRAPHQL_URL` | Leave as is |

Attach credentials:
- `Fetch failed execution` -> **n8n API**
- `Log failure to Administrate` -> **Administrate OAuth2**

Publish the workflow.

#### 3. Set it as the Error Workflow on your other workflows

For **each** workflow you want covered:

1. Open the workflow
2. Open **Workflow settings** (the ⋮ menu > Settings)
3. Under **Error workflow**, select **Error Handler - Log Failures to Administrate**
4. Save (and republish if the workflow is published)

Do not also add an Error Trigger inside those workflows; the Error Workflow setting takes precedence and an inline one becomes dead code.

#### 4. Configure the log sub-workflow

In both Code nodes (`Build log data` and `Convert error to payload`), change `LOG_ORIGIN_IDENTIFIER` at the top to the origin name your entries should appear under (default `Automator Integration`). Attach the **Administrate OAuth2** credential to `Log to external log`.

Call it from another workflow with an **Execute Workflow** node, passing:

| Input | Meaning |
|---|---|
| `nodeId` | ID of the Administrate record to log against (e.g. an Event or Contact ID) |
| `status` | External log status, e.g. `success` or `failed` |
| `message` | Plain-language summary |
| `details` | Optional extra text shown in the entry |
| `workflowId` / `executionId` | Optional; default to the calling execution's own IDs |

#### 5. Configure the Daily Log Digest (optional)

Create two Data Tables (String columns):

- **Log URL table** with columns `endpointName`, `url`. Add a row with `endpointName` = `automationLogs` and `url` = the address of the External Integration Logs page in your Administrate instance (e.g. `https://YOUR-INSTANCE.administrateapp.com/...`).
- **Recipients table** with columns `description`, `emailValues`. Add a row with `description` = `automationDigest` and `emailValues` = the recipient address(es), e.g. `admin@example.com`.

Then set the **Config** node:

| Key | Set to |
|---|---|
| `LOG_ORIGIN_NAME` | `REPLACE_WITH_LOG_ORIGIN_NAME` -> text contained in the name of the log origin to report on |
| `LOG_URL_TABLE_ID` | `REPLACE_WITH_LOG_URL_TABLE_ID` -> ID of the Log URL table |
| `RECIPIENTS_TABLE_ID` | `REPLACE_WITH_RECIPIENTS_TABLE_ID` -> ID of the Recipients table |
| `RECIPIENT_GROUP` | The `description` value of the recipients row (default `automationDigest`) |
| `EMAIL_WORKFLOW_ID` | `REPLACE_WITH_ID_OF_EMAIL_NOTIFICATION_WORKFLOW` -> ID of your email-sending workflow (from its URL, `/workflow/<id>`). It must accept the inputs `subject`, `recipients` and `bodyHTML` |
| `TIMEZONE` | IANA time zone for timestamps in the email (default `America/New_York`) |

Attach the **Administrate OAuth2** credential to `Get Log Origin ID` and `Get Logs`.

The digest starts from an **Execute Workflow** trigger. To run it daily, create a small workflow with a **Schedule Trigger** (e.g. every day at 07:00) followed by an **Execute Workflow** node that calls the digest.

### Testing

**Error handler** (Error Workflows fire on **production runs only**; manual executions never reach them):

1. Create a throwaway workflow with a Schedule Trigger (every minute) followed by a **Stop and Error** node
2. Set its Error workflow to **Error Handler - Log Failures to Administrate** and publish it
3. After it fails, deactivate it
4. Confirm a new row in `automation_failures` with a working `execution_url`
5. Confirm a `failed` entry in Administrate's External Integration Logs under the Organisation type, with the reason and time

**Log sub-workflow:** call it from a test workflow with an Event ID as `nodeId`, `status` = `success` and a message, then check the Event's integration log.

**Digest:** run the scheduling workflow manually and confirm the email arrives with the last 24 hours of entries.

## How It Works

**Error Handler**

1. **A workflow failed** (Error Trigger) fires when a workflow that uses this as its Error Workflow fails in production
2. **Config** holds the URLs, table ID and log origin
3. **Extract failure** normalises the trigger payload (workflow, execution, node, message, timestamp)
4. **Fetch failed execution** reads the failed execution back through the n8n API, because the trigger payload carries metadata only (continues on error)
5. **Build failure record** works out the real reason and failed node, builds the execution link, a plain-language log message, a structured log payload and notification text
6. **Record the failure** inserts the full record into `automation_failures`
7. **Log failure to Administrate** writes a `failed` entry to the External Integration Log under the Organisation type (continues on error)

**Log sub-workflow:** builds a payload containing the execution link and optional details, then calls `createExternalIntegrationLog` against the given `nodeId`.

**Daily digest:** looks up the logs page URL and recipients from Data Tables, finds the log origin by name, fetches every entry from the last 24 hours (paging through results), sorts them, builds an HTML table with timestamps in `TIMEZONE`, and passes it to your email workflow.

## Troubleshooting

**Nothing is logged when a workflow fails:**
- Error Workflows only fire on production executions, never manual runs
- Check the failing workflow's **Workflow settings > Error workflow** points at this handler, and that the handler is published
- Check the Administrate OAuth2 credential on `Log failure to Administrate`

**Rows appear but the reason is vague, or the execution link is wrong:**
- `Fetch failed execution` could not read the execution. Check the n8n API credential and `N8N_API_BASE_URL`
- Check `N8N_UI_BASE_URL` is your n8n host

**No row in `automation_failures`:**
- Check `FAILURES_TABLE_ID` and that the column names match exactly

**Sub-workflow entry does not appear:**
- `nodeId` must be a valid Administrate record ID; the Administrate response's `errors` array on `Log to external log` explains refusals

**Digest email is empty or not sent:**
- `LOG_ORIGIN_NAME` must match part of an existing log origin's name
- Check the Data Table rows (`automationLogs` and your `RECIPIENT_GROUP`) exist
- Check `EMAIL_WORKFLOW_ID` and that the email workflow accepts `subject`, `recipients` and `bodyHTML`
