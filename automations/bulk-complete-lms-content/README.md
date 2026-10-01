# Bulk Complete LMS Content

## 🧩 Problem

Sometimes a Learner needs their online content marked as complete without working through it, for example when they've already covered the material elsewhere or a technical problem stopped their progress being recorded. Doing this in Administrate means opening each Learner and completing each piece of content one at a time, which is slow for an Event with several Learners and several SCORM packages.

## 🤖 Automator Solution

This workflow adds an **LMS Completion Override** action to the Automator dropdown on an Event. When you run it, it finds every Learner on the Event whose override checkbox custom field is ticked and marks every SCORM content item on the Event as 100% complete for each of them.

**This workflow handles SCORM content only.** Other content types (such as videos, documents or quizzes) are not changed. It also doesn't mark the Learner as having passed the Event; only the SCORM content progress is completed.

## ✨ Features

- Runs on demand from the Automator dropdown on an Event
- Only processes Learners whose override custom field is ticked
- Matches the custom field by its unique API name, set in the Config node
- Completes every visible, non-archived SCORM content item on the Event
- Pages through Events with many Learners and many content items
- Checks the `errors` array on both the `attemptScorm` and `adminCompleteScorm` mutations
- Carries on with the remaining Learners if one fails and reports every failure in a summary
- Passes all values as GraphQL variables

## 🛠️ Setup

### Prerequisites

- Access to Administrate Automator
- Administrate OAuth2 credentials with permission to read Learners and Event content and to complete Learner content
- A Learner custom field of type **Checkbox**, for example named "LMS Content Override" with the unique API name `lms_content_override`
- An Event with SCORM content attached

### Installation

1. Download the [`workflow.json`](workflow.json) file from this directory
2. In your Automator instance, click the menu (⋮) and select "Import from File"
3. Upload the workflow JSON file

### Configuration

#### 1. Set credentials

Select your Administrate OAuth2 credential on:
- `Administrate Trigger`
- `Get Learners`
- `Get Event Content`
- `Attempt SCORM`
- `Complete SCORM`

#### 2. Fill in the Config node

| Field | Value |
|-------|-------|
| `EVENT_ID` | Leave as is. It's read from the trigger payload. |
| `OVERRIDE_FIELD_LOCATOR` | The unique API name of your Learner checkbox custom field. `lms_content_override` by default. Change it if your field uses a different name. |

To check the custom field's API name, run this query in the Administrate GraphQL explorer and use its `definitionLocator`:

```graphql
query {
  customFieldTemplate(type: Learner) {
    customFieldDefinitions { definitionLocator label type }
  }
}
```

#### 3. Optional: send the summary somewhere

`Summarise Results` produces one item containing `eventId`, `completedCount`, `warningCount`, `errorCount`, the `warnings` and `failures` lists and a plain text `summary`. To be notified, add a node after it, for example Slack or email.

### Testing

1. Activate the workflow. The Administrate Trigger registers the **LMS Completion Override** action on Events.
2. On a test Event with SCORM content, open the Students tab, click a student and select Edit Student.
3. Tick "Yes" for LMS Content Override and save.
4. Open the Automator dropdown on the Event and select **LMS Completion Override**.
5. Open the Event's Students tab, click Options and choose "Record Results". Expand the student's entry with the blue Progress button and check that all SCORM content is complete.
6. Check that students without the box ticked are unchanged.
7. In n8n, open the execution and check **Summarise Results**. Run it a second time on the same Event and check the summary for any warnings or failures.

## ⚙️ How It Works

1. **Administrate Trigger**: runs when you select **LMS Completion Override** from the Event's Automator dropdown.
2. **Config**: reads the Event ID from the payload, along with the custom field API name.
3. **Get Learners**: fetches the Event's Learners, 100 per page, with the value of the override custom field.
4. **Filter For Override**: keeps Learners whose override field is `true`.
5. **Get Event Content**: fetches the Event's visible, non-archived SCORM content.
6. **Create All Possible Combinations**: pairs every selected Learner with every SCORM content item.
7. **Loop Over Items**: handles one Learner and content pair at a time.
8. **Attempt SCORM / Complete SCORM**: starts a SCORM attempt and then completes it as an administrator.
9. **Tag SCORM Result**: checks both mutations' `errors` arrays and records the pair as completed, completed with a warning or failed.
10. **Summarise Results**: counts the results and lists every warning and failure.

## 🔧 Troubleshooting

**The action doesn't appear in the Automator dropdown:**
- Check the workflow is active and the Administrate credential on the trigger is for the right instance.

**Nothing is completed:**
- Check the Learner's override box is ticked and saved.
- Confirm `OVERRIDE_FIELD_LOCATOR` matches the custom field's unique API name exactly.
- Only SCORM content is processed. Hidden and archived content is also skipped.

**A pair is listed under failures:**
- The reason shows the message from Administrate for the `attemptScorm` or `adminCompleteScorm` mutation.
- "No response returned" means the request itself failed. Open the execution and look at the failing Administrate node for details.

**A pair is listed under warnings:**
- The content was completed but the attempt step returned an error, for example because the Learner already had an attempt. Check the Learner's progress to confirm.
