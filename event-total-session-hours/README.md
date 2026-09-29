# Sync Event Total Session Hours

**Version:** 1.0.0
**Last Updated:** 2026-09-29

## Problem

Many reports, certificates and compliance records need an Event's total training hours. Administrate stores the start and end of each Session, but not a running total for the Event. Teams end up adding up session lengths by hand and typing the result into a custom field, and that value goes stale as soon as a session is moved, extended or cancelled.

## Automator Solution

Whenever an Event is updated, this workflow adds up the length of all its **published** Sessions and writes the total into a number custom field on the Event. The field always matches what's actually scheduled.

## Features

- Runs automatically on every Event update
- Counts only published Sessions, so draft and cancelled Sessions are ignored
- Pages through Events with many Sessions (100 per page, up to 5,000 Sessions)
- Session times are handled as local times, so daylight saving changes don't skew durations
- Optional one-hour lunch deduction for long Sessions
- Writes only when the total has changed, so updating the field doesn't trigger another run
- Updates only the target custom field; other custom field values are left alone
- Produces success and error log items that you can route anywhere

## Setup Instructions

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials
- A custom field of type **Number** on Events to hold the total (for example "Total Hours")

### Installation

1. Download the `workflow.json` file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Set credentials

Select your Administrate OAuth2 credential on:
- `Administrate Trigger`
- `Get Event Sessions`
- `Write Total Hours`

#### 2. Fill in the Config node

| Field | Value |
|-------|-------|
| `eventId` | Leave as is. It's read from the webhook payload. |
| `customFieldDefinitionKey` | Replace `REPLACE_WITH_EVENT_CUSTOM_FIELD_KEY` with the `key` of your Event number custom field. |
| `deductLunchBreak` | `false` by default. Set it to `true` to take one hour off any Session of 4.5 hours or more that spans 12:00–13:00. |

To find the custom field key, run this query in the Administrate GraphQL explorer:

```graphql
query {
  customFieldTemplate(type: Event) {
    customFieldDefinitions { key label type }
  }
}
```

#### 3. Optional: send the log somewhere

`Build Success Log` and `Build Error Log` each produce one item containing `status`, `timestamp`, `eventId`, `eventTitle`, `totalHours` and `errors`. To keep a record, add a node between each builder and **Merge**, for example Slack, Google Sheets or an Administrate integration log.

### Testing

1. Activate the workflow. The Administrate Trigger registers an *Event Updated* webhook for you.
2. Open an Event that has several published Sessions and make any small edit.
3. Check that the custom field now shows the total hours of its published Sessions.
4. In n8n, open the executions list. You should see one run that wrote the value and a second run that stopped at **Value Changed?**.

## How It Works

1. **Trigger**: an *Event Updated* webhook fires in Administrate.
2. **Config**: reads the Event ID from the payload, along with the configured custom field key.
3. **Get Event Sessions**: fetches the Event's Sessions, 100 per page, and the current custom field value.
4. **Compute Duration**: works out each published Session's length in hours, applying the lunch deduction if it's switched on.
5. **Sum Hours / Format Total**: adds up the durations and rounds to two decimal places.
6. **Value Changed?**: compares the new total with the stored value. If they're the same, the run ends here.
7. **Write Total Hours**: updates the Event custom field with the `event.update` mutation.
8. **Has Errors?**: checks for GraphQL and validation errors and builds a success or error log item.

## Troubleshooting

**The field isn't updating:**
- Check the workflow is active and the *Event Updated* webhook exists in Administrate.
- Confirm `customFieldDefinitionKey` is the key of an **Event** custom field of type **Number**.
- Only published Sessions are counted, so an Event with only draft Sessions totals 0.

**The total looks wrong by an hour per day:**
- Check the `deductLunchBreak` setting in Config.

**The workflow runs twice for every edit:**
- This is expected. Writing the field triggers *Event Updated* again, and the second run stops at **Value Changed?** without writing anything.
