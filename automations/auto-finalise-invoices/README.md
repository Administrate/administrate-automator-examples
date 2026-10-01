# Auto-Finalise Invoices Before Event Start

## 🧩 Problem

Many training providers raise a draft invoice when a booking is made but only want to finalise it shortly before the Event starts, once the booking is unlikely to change. Doing this by hand means checking every Opportunity each day, copying the comments and purchase order reference onto the invoice, finalising it and then moving the Opportunity on. It's repetitive and easy to miss, and a late invoice delays payment.

## 🤖 Automator Solution

This workflow runs daily. It finds every Opportunity in a chosen pipeline step whose soonest Event starts within a set number of days. For each one with a **Draft** invoice, it copies two Opportunity attributes onto the invoice, finalises the invoice and progresses the Opportunity to a target step. A Slack summary lists what was moved, what was skipped and anything that failed.

A dry run mode is switched on by default so you can check the results before anything changes.

## ✨ Features

- Runs on a daily schedule
- Pages through every Opportunity in the source step (`PAGE_SIZE` per page)
- Uses the soonest Event on each Opportunity to decide whether it's within the window
- Copies a comments attribute and a PO reference attribute onto the invoice
- Finalises the invoice and progresses the Opportunity to the target step
- Skips invoices that aren't in Draft (for example Final or Void) and lists them in the summary
- Checks the `errors` array on every mutation and stops the chain for that Opportunity at the first failure
- Posts a Slack summary with moved, skipped and failed Opportunities, including the failing stage and message
- `DRY_RUN` mode (on by default) lists what WOULD be finalised and moved without changing anything
- Passes all values as GraphQL variables, so quotes or line breaks in comments are handled safely

## 🛠️ Setup

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials with permission to read Opportunities and update and finalise invoices
- A Slack OAuth2 credential with permission to post to the summary channel
- Two Opportunity attributes holding the invoice comments and PO reference
- A source step where Opportunities wait for invoicing and a target step to move them to afterwards

### Installation

1. Download the [`workflow.json`](workflow.json) file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Set credentials

Select your Administrate OAuth2 credential on:
- `Get Opportunities`
- `Update Invoice`
- `Finalise Invoice`
- `Progress Opportunity`

Select your Slack OAuth2 credential on `Send Slack Summary`.

#### 2. Fill in the Config node

| Field | Value |
|-------|-------|
| `SOURCE_STEP_ID` | Replace `REPLACE_WITH_SOURCE_STEP_ID` with the ID of the Opportunity step to process. |
| `TARGET_STEP_ID` | Replace `REPLACE_WITH_TARGET_STEP_ID` with the ID of the step to move Opportunities to after finalising. |
| `COMMENTS_ATTRIBUTE_KEY` | Replace `REPLACE_WITH_COMMENTS_ATTRIBUTE_KEY` with the `definitionKey` of the Opportunity attribute to write to the invoice comments. |
| `PO_REFERENCE_ATTRIBUTE_KEY` | Replace `REPLACE_WITH_PO_REFERENCE_ATTRIBUTE_KEY` with the `definitionKey` of the Opportunity attribute to write to the invoice PO reference. |
| `DAYS_WINDOW` | `10` by default. An invoice is finalised when the soonest Event starts within this many days. |
| `PAGE_SIZE` | `100` by default. The number of Opportunities fetched per request. |
| `SLACK_CHANNEL` | Replace `REPLACE_WITH_SLACK_CHANNEL_ID` with the ID of the Slack channel for the summary. |
| `DRY_RUN` | `true` by default. While `true`, no mutations run and the summary lists what WOULD be finalised and moved. Set it to `false` to make changes. |

To find step IDs, run this query in the Administrate GraphQL explorer against Opportunities you know are in each step:

```graphql
query {
  opportunities(first: 20) {
    edges { node { name step { id name } } }
  }
}
```

To find attribute keys, run the `Get Opportunities` query against an Opportunity that has the attributes filled in and look at the `attributes` array. Each entry has a `definitionKey` and a `value`.

#### 3. Optional: change the schedule

`Schedule Trigger` runs once a day at midnight. Open it to pick a different time.

### Testing

1. Leave `DRY_RUN` set to `true`.
2. Put a test Opportunity with a Draft invoice in the source step, linked to an Event that starts within `DAYS_WINDOW` days. Fill in the comments and PO reference attributes, including a comment with quotes.
3. Run the workflow manually. The Slack summary should list the test Opportunity under "WOULD be updated, finalised and moved" and nothing in Administrate should change.
4. Set `DRY_RUN` to `false` and run it again. Check the invoice now shows the comments and PO reference, is finalised and the Opportunity is in the target step.
5. To test failures, point `TARGET_STEP_ID` at an invalid step and run it on another test Opportunity. The summary should list it under failures with the reason.
6. Run it again. The moved Opportunity is no longer in the source step, so it isn't processed twice.

## ⚙️ How It Works

1. **Schedule Trigger**: starts the run once a day.
2. **Config**: holds the step IDs, attribute keys, window, page size, Slack channel and dry run flag.
3. **Get Opportunities**: fetches every Opportunity in the source step, one page at a time, with its invoice state, attributes and Events.
4. **Prepare Items**: keeps Opportunities whose soonest Event starts between now and `DAYS_WINDOW` days ahead and picks out the two attribute values.
5. **Loop Over Items**: handles one Opportunity at a time.
6. **Is Draft?**: invoices that aren't in Draft go to **Tag Skipped**.
7. **Is Dry Run?**: in dry run mode, **Tag Dry Run** records what would happen and no mutations run.
8. **Update Invoice**: writes the comments and PO reference to the invoice. **Update Succeeded?** checks the `errors` array.
9. **Finalise Invoice**: finalises the invoice. **Finalise Succeeded?** checks the `errors` array.
10. **Progress Opportunity**: moves the Opportunity to the target step.
11. **Tag Mutation Result**: records the Opportunity as moved or as an error, with the stage that failed and the message.
12. **Format Slack Summary / Send Slack Summary**: posts the moved, would-move, failed and skipped lists to Slack.

## 🔧 Troubleshooting

**Nothing is processed:**
- Check `SOURCE_STEP_ID` is the right step and that the Opportunities have an Event starting within `DAYS_WINDOW` days. Events that have already started are ignored.
- Check `DRY_RUN`. While it's `true` the summary shows what would happen but nothing changes.

**No Slack message is posted:**
- Check the Slack credential and that the app has been added to the `SLACK_CHANNEL` channel.
- If no Opportunities in the source step fall within the window, the run stops before the summary.

**The invoice comments or PO reference are blank:**
- Confirm `COMMENTS_ATTRIBUTE_KEY` and `PO_REFERENCE_ATTRIBUTE_KEY` match the `definitionKey` values in the Opportunity's `attributes` array.

**An Opportunity is listed under failures:**
- The reason shows which stage failed (updating, finalising or moving) and the message from Administrate. If updating fails, the invoice isn't finalised. If finalising fails, the Opportunity isn't moved.
- "No response returned" means the request itself failed. Open the execution and look at the failing Administrate node for details.

**Get Opportunities fails with a pagination limit error:**
- The node stops after 50 pages. Raise `PAGE_SIZE` if the source step holds more than 50 pages of Opportunities.
