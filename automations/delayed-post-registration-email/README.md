# Delayed Email After Registration

## 🧩 Problem

Some emails work best a little while after someone registers: joining instructions a few hours later, a pre-course questionnaire the next day or a welcome pack once the booking has settled. Administrate's triggered emails send when the registration happens, so teams end up sending these follow-ups by hand or not at all, and each Event may need a different email and a different delay.

## 🤖 Automator Solution

When a Learner is created on an Event, this workflow reads two custom fields on the Event: the name of a communication template and a delay in minutes. It records the planned send time on the Learner, waits for the delay, then sends the template to that Learner and records whether the send succeeded. Events without both fields filled in are ignored.

## ✨ Features

- Runs automatically when a Learner is created
- The template and delay are set per Event with custom fields, so one workflow covers every Event
- The webhook filter skips Learners on Events that don't use the feature, so those runs never start
- Stores the planned send time (UTC) on the Learner for reporting
- Stops with a clear error if no communication template matches the name on the Event
- Records success or failure on the Learner in a custom field
- All data values are passed to the API as GraphQL variables

## 🛠️ Setup Instructions

### 📋 Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials with permission to read Events and communication templates, update Learners and send ad hoc emails
- A verified sending email address in Administrate
- One or more communication templates to send
- These custom fields:

| Entity | Type | Purpose |
|--------|------|---------|
| Event | Text | The **name** of the communication template to send |
| Event | Number or Text | The delay in **minutes** after registration |
| Learner | Text | Stores the planned send time |
| Learner | Checkbox or Text | Set to `true` when the email was sent and `false` when it failed |

### 📥 Installation

1. Download the [`workflow.json`](workflow.json) file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### ⚙️ Configuration

#### 1. Set credentials

Select your Administrate OAuth2 credential on:
- `Filtered Learner Created Webhook`
- `Get Template ID`
- `Store Scheduled Time`
- `Send Email`
- `Mark Email Sent`
- `Mark Email Failed`

#### 2. Fill in the Config node

| Field | Placeholder | Value |
|-------|-------------|-------|
| `TEMPLATE_FIELD_KEY` | `REPLACE_WITH_EVENT_TEMPLATE_FIELD_KEY` | The `key` of the Event custom field holding the template name |
| `DELAY_FIELD_KEY` | `REPLACE_WITH_EVENT_DELAY_FIELD_KEY` | The `key` of the Event custom field holding the delay in minutes |
| `SCHEDULED_TIME_FIELD_KEY` | `REPLACE_WITH_LEARNER_SCHEDULED_TIME_FIELD_KEY` | The `key` of the Learner custom field that stores the planned send time |
| `EMAIL_SENT_FIELD_KEY` | `REPLACE_WITH_LEARNER_EMAIL_SENT_FIELD_KEY` | The `key` of the Learner custom field set to `true` or `false` after sending |
| `SENDING_ADDRESS_ID` | `REPLACE_WITH_SENDING_ADDRESS_ID` | The ID of the sending email address the emails come from |

To find the custom field keys, run these queries in the Administrate GraphQL explorer:

```graphql
query {
  event: customFieldTemplate(type: Event) {
    customFieldDefinitions { key label type }
  }
  learner: customFieldTemplate(type: Learner) {
    customFieldDefinitions { key label type }
  }
}
```

To find the sending address ID:

```graphql
query {
  sendingEmailAddresses {
    edges { node { id name address verified } }
  }
}
```

#### 3. Update the trigger filter

The filter on **Filtered Learner Created Webhook** runs inside Administrate before the workflow starts, so it can't read the Config node. Open the trigger and, in its filter expression, replace:
- `REPLACE_WITH_EVENT_TEMPLATE_FIELD_KEY` with the same value as `TEMPLATE_FIELD_KEY`
- `REPLACE_WITH_EVENT_DELAY_FIELD_KEY` with the same value as `DELAY_FIELD_KEY`

If you change the keys later, update both Config and the filter.

#### 4. Set up an Event

On each Event that should send a delayed email, fill in the template name field with the exact name of a communication template and the delay field with a number of minutes, for example `120` for two hours or `1440` for one day.

### 🧪 Testing

1. Activate the workflow. The Administrate Trigger registers a *Learner Created* webhook with the filter for you.
2. On a test Event, set the template name and a short delay such as `2`.
3. Register a test Learner on the Event.
4. Check that the Learner's scheduled time field is filled in straight away and the execution is waiting at **Wait**.
5. After the delay, check the Learner receives the email and the email sent field is `true`.
6. Set the template name on the Event to a name that doesn't exist and register another Learner. The run should stop at **Template Not Found** and no email should be sent.
7. Register a Learner on an Event without the custom fields. No execution should start.

## 🔍 How It Works

1. **Filtered Learner Created Webhook**: a *Learner Created* webhook fires, but only when the Learner's Event has both the template name and delay fields filled in.
2. **Config**: holds the custom field keys and sending address and passes the webhook payload on.
3. **Split Custom Fields / Isolate Template Custom Field**: pick out the Event's template name field.
4. **Get Template ID**: looks up the communication template by name.
5. **Template Found?**: stops the run at **Template Not Found** if there's no matching template.
6. **Collect Email Details**: gathers the template ID, Learner ID, Event ID and delay, reading the IDs and delay directly from the webhook payload.
7. **Set Scheduled Time / Store Scheduled Time**: works out when the email will be sent and saves it on the Learner.
8. **Wait**: pauses for the delay.
9. **Send Email**: sends the template to the Learner with `sendBulkLearnerAdhocEmail`.
10. **Email Sent Without Errors? / Mark Email Sent / Mark Email Failed**: records the result on the Learner.

The Administrate nodes stop the run if the API returns top-level GraphQL errors, so failures show up in the executions list.

### ⏳ Long delays

The **Wait** node keeps each execution open for the whole delay. That's fine for minutes or hours, but with long delays and busy Events you can end up with a large number of waiting executions. They take up resources on your Automator instance and are lost if waiting executions are cleared or the workflow is changed in a way that stops them resuming.

For long delays, consider a schedule-based alternative. This example doesn't include it, but the building blocks are already here:
1. Keep the first part of this workflow up to **Store Scheduled Time** and remove the **Wait** and send steps.
2. In a second workflow, use a Schedule Trigger that runs every few minutes.
3. Query Learners whose scheduled time field is in the past and whose email sent field is empty.
4. Send the email to each one and set the email sent field, exactly as this workflow does after **Wait**.

Because the planned send time is stored on the Learner, nothing is held in memory and a missed run is picked up by the next one.

## 🩺 Troubleshooting

**No executions start when a Learner is created:**
- Check the workflow is active and the *Learner Created* webhook exists in Administrate.
- Check the placeholders in the trigger filter have been replaced with your Event custom field keys.
- Check the Event has both the template name and delay fields filled in.

**The run stops at Template Not Found:**
- The template name on the Event must match a communication template name exactly, including capitals and spaces.

**The run stops at Get Template ID, Store Scheduled Time or a Mark node with an API error:**
- Check the custom field keys in Config belong to the right entities: Event for the template and delay fields, Learner for the scheduled time and email sent fields.

**The email sent field is `false`:**
- Open the **Send Email** output to see the error. Check `SENDING_ADDRESS_ID` is a verified sending address and the template is valid for Learner emails.

**The email is sent at the wrong time:**
- The delay is in minutes. The stored scheduled time is in UTC, so it may differ from local time by an hour or more.
